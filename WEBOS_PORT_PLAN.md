# Hostile Takeover → webOS (HP TouchPad) Port Plan

Single-player port of Hostile Takeover to webOS 3.0 on the HP TouchPad, distributed via sideloading and the webOS Ports community catalog (Preware).

This plan is sized to be executed by Claude Code in roughly the order presented. Each phase has concrete files, deliverables, and an acceptance check. **Do not skip ahead** — later phases depend on earlier ones building cleanly.

---

## 1. Scope

### In scope
- Single-player campaign and skirmish on HP TouchPad (webOS 3.0.x, 1024×768, ARMv7 Cortex-A9)
- Touch input (single-finger MVP, multi-touch in stretch phase)
- Audio
- Save games, preferences
- Sideload + community-catalog distribution (.ipk)

### Out of scope (deferred / cut)
- Multiplayer: TCP transport, lobby, chat, server browser
- Mission-pack downloads (uses libcurl)
- Stats/leaderboard HTTP service
- HP Pre / Veer (smaller screens) — TouchPad-only for now
- LuneOS / modern webOS-OSE — TouchPad webOS 3.0.5 only

The "no multiplayer" decision means **libcurl is not required**, which removes a significant cross-compile dependency.

---

## 2. Target Platform Reference

| Attribute | Value |
|---|---|
| Device | HP TouchPad |
| OS | webOS 3.0.5 |
| CPU | Qualcomm APQ8060 dual-core Cortex-A9 @ 1.2–1.5 GHz |
| ABI | `armv7-a`, `vfpv3`, hard-float (`-mfloat-abi=softfp` per PDK convention) |
| Screen | 1024×768 IPS, ~132 DPI |
| Toolchain | HP webOS PDK 3.0.5 (`arm-none-linux-gnueabi-` GCC 4.x) |
| Display lib | **SDL 1.2** (bundled with PDK) + PDL (Palm Development Library) |
| Touch | via PDL multitouch API (PDL_GetNumFingers / PDL_GetFinger) |
| Distribution | `.ipk` via `palm-package` / `palm-install`; community catalog via Preware |

References (for Claude Code to consult during implementation):
- https://developer.archive.palm.com/documentation/pdk/
- webOS PDK SDL example projects under `${PDK}/share/samples/`
- webOS Ports project (LunaCE) for community-catalog packaging conventions

---

## 3. Architectural Strategy

The existing SDL2 port is the closest sibling and the right starting point. Two architectural calls:

### 3.1 SDL2 → SDL 1.2 backport (display + input)

**Why not just use SDL2 on webOS?** As of this writing there is **no working SDL2 build for legacy HP webOS 3.x on the TouchPad**. SDL2 packages that show up in searches target *modern* LG webOS (smart TVs) — a different OS lineage with a different toolchain, libc, and graphics stack. The community effort to bring SDL2 to TouchPad webOS exists but is not in a usable state. **Do not spend time trying to make SDL2 work** — go straight to the backport.

SDL2's `SDL_Renderer`/`SDL_Texture` are not available in SDL 1.2. The port keeps the game's internal rendering model (paletted 8-bit `m_gamePixels` → 32-bit `m_gamePixels32`) and replaces only the **presentation layer**: SDL2 texture upload becomes an SDL 1.2 `SDL_Surface` lock/memcpy/unlock/flip. This is a self-contained rewrite of `game/sdl/display.cpp` plus surgery on touch handling in `game/sdl/host.cpp`.

Audio (`game/sdl/sdlsounddev.cpp`) already uses SDL 1.2-compatible APIs and ships unchanged.

### 3.2 Multiplayer removal: file-level exclusion + form stubs
Instead of `#ifdef`-ing through the codebase, the multiplayer subsystem is removed by:
1. Excluding multiplayer-only `.cpp` files from the build's source list.
2. Adding a small `mp_stubs.cpp` that defines the few symbols core code still references (e.g. `Transport *gptra = NULL;`, stubbed `IChatController` factory).
3. Stripping the "Multiplayer" entry from the main menu in `Shell.cpp` (compile-time, behind a new `-DNO_MULTIPLAYER` define).

This is cleaner than scattering ifdefs and leaves the multiplayer codepath salvageable if a future maintainer wants to re-enable it.

---

## 4. Directory Layout

New code lives under `game/sdl/webos/`, mirroring the convention of `game/sdl/linux/` and `game/sdl/android/`:

```
game/sdl/webos/
├── makefile                 # cross-compile makefile, modeled on linux/makefile
├── hosthelpers.cpp          # HostHelpers impl for webOS (mirrors linux/hosthelpers.cpp)
├── webos_lifecycle.cpp      # PDL_Quit / focus / pause handlers
├── mp_stubs.cpp             # stubs for symbols left dangling by MP removal
├── README.md                # build & install instructions
└── package/
    ├── appinfo.json         # webOS app manifest
    ├── icon.png             # 64×64 launcher icon
    └── package.sh           # wraps palm-package
```

Modified files (in-place, behind `__WEBOS__` or `NO_MULTIPLAYER` defines):
- `game/sdl/display.cpp` — SDL 1.2 backport path
- `game/sdl/host.cpp` — single-finger touch via SDL 1.2 mouse events; PDL multi-touch in stretch phase
- `game/sdl/sdleventserver.h` — SDL_PeepEvents is SDL 1.2-compatible, but verify; small tweaks expected
- `game/sdl/main.cpp` — call `PDL_Init` / `PDL_Quit` around `wi::GameMain`
- `game/Shell.cpp` — hide multiplayer menu entries under `NO_MULTIPLAYER`
- `game/sdl/sdlhttprequest.cpp` / `sdlhttpservice.cpp` — exclude from build, not modified

---

## 5. Implementation Phases

### Phase 0 — Toolchain setup (manual, no code)
Done outside the repo; Claude Code records the steps in `game/sdl/webos/README.md`.

1. Install HP webOS PDK 3.0.5 on a Linux dev box. Archive mirrors exist; the SDK installer drops a sysroot at `/opt/PalmPDK/` (or `~/PalmPDK/`).
2. Confirm `arm-none-linux-gnueabi-gcc --version` resolves (PDK adds it to PATH via `pdk-env.sh`).
3. Confirm SDL 1.2 headers + `libSDL.so` are present under `$PDK/device/lib/` and `$PDK/include/SDL/`.
4. Confirm PDL headers + lib (`PDL.h`, `libpdl.so`).
5. Install `novacom` + `palm-package` / `palm-install` host tools.

**Acceptance:** `arm-none-linux-gnueabi-g++ -dumpmachine` prints `arm-none-linux-gnueabi`, and `pkg-config --cflags sdl` (or PDK equivalent) finds SDL 1.2.

---

### Phase 1 — Build skeleton + multiplayer exclusion

**Goal:** A webOS makefile that links against PDK SDL 1.2 and produces an `arm-none-linux-gnueabi` ELF — even if the ELF segfaults at runtime.

#### 1.1 Create `game/sdl/webos/makefile`
Model on `game/sdl/linux/makefile`. Key differences:
- `CC := $(PDK_TOOLCHAIN)/arm-none-linux-gnueabi-gcc`
- `CXX := $(PDK_TOOLCHAIN)/arm-none-linux-gnueabi-g++`
- `CPPFLAGS += -D__LINUX__ -DSDL -DNO_MULTIPLAYER -D__WEBOS__ -DTRACKSTATE`
- `CPPFLAGS += -I$(PDK)/include -I$(PDK)/include/SDL`
- `LDFLAGS  += -L$(PDK)/device/lib -lSDL -lpdl -lpthread -lm`
- **No** `-lcurl`.
- C++ standard: `-std=c++0x` (matches Android NDK 4.8 build).
- Drop ASan (`-fsanitize=address` is unavailable on PDK GCC 4.x).
- Output: `webos-build/HostileTakeover`.

#### 1.2 Source-file exclusion
In the makefile's source-discovery wildcards, **exclude**:
```
game/Multiplayer.cpp
game/chatter.cpp
game/comm.cpp
game/SocTransport.cpp
game/xtransport.cpp                 # if present; verify
game/dlmissionpack.cpp
game/creategameform.cpp
game/createroomform.cpp
game/createroomform.cpp
game/chooseserverform.cpp
game/lobby.cpp
game/sdl/sdlhttprequest.cpp
game/sdl/sdlhttpservice.cpp
game/sdl/transportmgr.cpp
```
Use `filter-out` exactly as `game/sdl/linux/makefile` does for `SocTransport` et al. Verify each file exists before adding to the exclusion list — some may be header-only or already conditional.

#### 1.3 Create `game/sdl/webos/mp_stubs.cpp`
Define the bare minimum symbols that core (non-MP) code references. From the agent's report:
- `Transport *gptra = NULL;`
- A no-op `IChatController` if `Shell.cpp` instantiates one unconditionally.

The exact list is **discovered iteratively** from linker errors. Claude Code's task: keep adding stubs until the link succeeds. Each stub must be a no-op that returns a safe value (NULL, 0, false).

#### 1.4 Strip multiplayer menu entries
In `game/Shell.cpp` (and any `mainform.cpp` / similar), wrap menu entries that lead to `creategameform`/`chooseserverform`/`createroomform` in `#ifndef NO_MULTIPLAYER`. Goal: when the user reaches the main menu, "Multiplayer" is either hidden or grayed out — there is no codepath into the excluded forms.

**Acceptance for Phase 1:** `make -C game/sdl/webos` produces an ELF binary. `file webos-build/HostileTakeover` reports `ELF 32-bit LSB executable, ARM, EABI5`. The binary need not run yet.

---

### Phase 2 — Display: SDL2 → SDL 1.2 backport

**Goal:** The game launches on the TouchPad and renders the title screen.

#### 2.1 Add `__WEBOS__` branches to `game/sdl/display.cpp`
The existing code (per agent report, display.cpp lines 71–180) uses:
- `SDL_CreateWindow` → SDL 1.2: `SDL_SetVideoMode(1024, 768, 32, SDL_HWSURFACE | SDL_DOUBLEBUF | SDL_FULLSCREEN)`
- `SDL_CreateRenderer` / `SDL_CreateTexture` → **delete entirely** in the `__WEBOS__` branch; the returned `SDL_Surface*` from `SetVideoMode` is the presentation target.
- `m_gamePixels` (8-bit paletted, internal) and `m_gamePixels32` (RGB888 staging) **stay** — they're the game's own buffers, independent of SDL.

#### 2.2 Rewrite `Display::RenderGameSurface` for SDL 1.2
Replace (display.cpp lines 306–316) the SDL2 texture-update sequence with:
```cpp
#ifdef __WEBOS__
    SDL_Surface *screen = m_screen;
    if (SDL_MUSTLOCK(screen)) SDL_LockSurface(screen);
    // m_gamePixels32 is RGB888; screen->format is the display format.
    // For an MVP, assume screen->format matches and memcpy row-by-row
    // honoring screen->pitch vs. m_pitch32. If formats differ, fall back
    // to a per-pixel convert using SDL_MapRGB.
    uint8_t *dst = (uint8_t *)screen->pixels;
    uint8_t *src = (uint8_t *)m_gamePixels32;
    for (int y = 0; y < m_cy; y++) {
        memcpy(dst + y * screen->pitch, src + y * m_pitch32, m_cx * 4);
    }
    if (SDL_MUSTLOCK(screen)) SDL_UnlockSurface(screen);
    SDL_Flip(screen);
#else
    // existing SDL2 path
#endif
```

If the pixel formats don't match (likely: SDL 1.2 may give back 16bpp `RGB565` on this hardware), implement a one-time `SDL_PixelFormat` check in `Display::Init` and either:
- (preferred) request 32bpp explicitly in `SDL_SetVideoMode` and accept whatever the driver gives,
- or write a `convert565` row routine. **Do not over-engineer** — pick one path based on what `Display::Init` logs.

#### 2.3 Resolution
The game's internal resolution is 800×600 (display.cpp:132–139 per agent report). Target two options, behind a CLI flag or compile define:
- **A (default):** Render at native 1024×768 — bumps the internal mode. Requires verifying the game's UI layout isn't hard-coded to 800×600. From `RawBitmap.cpp` / form code, this risks breaking HUD placement.
- **B (safe):** Render at 800×600 to an offscreen surface, blit centered to a 1024×768 black screen. No layout risk. Slightly smaller play area (~78% of screen).

**Recommendation: start with B**, ship the MVP, then evaluate A. The blit uses `SDL_BlitSurface` with a centered dest rect.

#### 2.4 Logging
Add a `Log()` impl in webos/hosthelpers.cpp that writes to `/tmp/hostiletakeover.log` (visible via novacom). Critical for debugging Phase 2.

**Acceptance:** Game launches on device, shows the splash screen, advances to the main menu. Menu may not be interactive yet (Phase 3).

---

### Phase 3 — Input: single-finger touch via SDL 1.2 mouse events

**Goal:** User can tap the menu, start a mission, control units.

SDL 1.2 on webOS delivers the first finger as `SDL_MOUSEBUTTONDOWN` / `SDL_MOUSEMOTION` / `SDL_MOUSEBUTTONUP` (the PDK SDL port does this translation automatically). The existing macOS/Linux branch in `game/sdl/host.cpp` (lines 296–314 per agent report) **already handles mouse events** and maps them to `penDownEvent` / `penMoveEvent` / `penUpEvent`.

#### 3.1 Extend the desktop branch to `__WEBOS__`
In `game/sdl/host.cpp`, the existing `#if defined(__MACOSX__) || defined(__LINUX__)` style guard around the mouse branch — extend to include `__WEBOS__`. Verify the touch (`SDL_FINGERDOWN`) branch is fully `#ifndef __WEBOS__`-excluded so SDL2-only types like `SDL_FingerID` don't get compiled.

#### 3.2 Coordinate scaling
If Phase 2.3 used option B (centered 800×600), translate mouse coordinates: subtract the letterbox offset; events outside the 800×600 active area are dropped. Add this in the host.cpp mouse branch.

#### 3.3 Right-click / context actions
The game uses right-click on desktop for context menu. On webOS, the simplest MVP is **no right-click equivalent** for now — single-tap is select/move, drag is box-select. Long-press for context menu is a stretch enhancement (Phase 9).

**Acceptance:** Start a mission, select a unit, move it, build a structure, lose/win the mission.

---

### Phase 4 — Audio

The existing `game/sdl/sdlsounddev.cpp` uses SDL 1.2-compatible APIs (per agent report: `SDL_OpenAudio`, `SDL_LockAudio`, `SDL_PauseAudio`, etc.). It should compile and run unchanged.

**Tasks:**
1. Confirm SDL 1.2 on webOS supports `AUDIO_U8` @ 8kHz mono. If not (likely it'll want at least 22kHz), add a resampler or change the requested format and let SDL convert.
2. Verify audio doesn't block the main thread on webOS (the PDK has occasionally finicky audio backend behavior).

**Acceptance:** Music, unit acknowledgments, and weapon SFX play during gameplay without crackling or dropouts.

---

### Phase 5 — HostHelpers impl

#### 5.1 Create `game/sdl/webos/hosthelpers.cpp`
Model on `game/sdl/linux/hosthelpers.cpp`. webOS-specific paths:

| Method | webOS path |
|---|---|
| `GetMainDataDir` | `/media/internal/.HostileTakeover/` or the app's data dir under `/var/luna/data/` |
| `GetSaveGamesDir` | `${MainDataDir}/SaveGames/` |
| `GetPrefsFilename` | `${MainDataDir}/prefs.bin` |
| `GetTempDir` | `/tmp/` |
| `GetMissionPacksDir` | unused (MP off) — return `""` |
| `GetCompletesDir` | `${MainDataDir}/Completes/` |
| `GetUdid` | Hash of `/proc/sys/kernel/random/boot_id` or the device nduid (PDL_GetDeviceInfo) |
| `GetPlatformString` | `"webos-touchpad"` |
| `GetChatController` | `NULL` (multiplayer off) |

The bundled `htdata832.pdb` ships **inside the .ipk** at a known install path (e.g. `/media/cryptofs/apps/usr/palm/applications/com.example.hostiletakeover/htdata832.pdb`). On first launch, `HostHelpers::Init` copies it to `GetMainDataDir()` if not already there — matching the Android lifecycle.

#### 5.2 `GetSurfaceProperties`
Return `cxWidth = 800, cyHeight = 600` (matches the internal render resolution, option B). `density = 1.0`. Pixel format flags: match what `Display::Init` settled on.

#### 5.3 No-op stubs
`InitiateAsk`, `GetAskString`, `InitiateWebView`, `OpenUrl` — implement as logging no-ops for the MVP. The game shouldn't reach these without multiplayer; if it does, log loudly.

**Acceptance:** `HostHelpers::Init` succeeds, all directories exist after first launch, save games persist across app restarts.

---

### Phase 6 — webOS lifecycle (PDL)

#### 6.1 PDL init/quit in `main.cpp`
```cpp
#ifdef __WEBOS__
    PDL_Init(0);
#endif
    int rc = wi::GameMain("");
#ifdef __WEBOS__
    PDL_Quit();
#endif
    return rc;
```

#### 6.2 Background / foreground
webOS sends `SDL_ACTIVEEVENT` (SDL 1.2) when the app loses focus (user swiped to card view). Map to the existing `kidmAppKillFocus` / `kidmAppSetFocus` message flow in `game/sdl/host.cpp` (the iOS branch already does this for SDL_APP_DIDENTERBACKGROUND / FOREGROUND — replicate that logic).

Since multiplayer is gone, the transport close/reconnect dance (host.cpp lines 414–449 per agent report) is unreachable. Leave it `#ifndef NO_MULTIPLAYER`'d out.

#### 6.3 Suspend & save
On `kidmAppKillFocus`: trigger an autosave to `${SaveGamesDir}/htsave_autosuspend`. On `kidmAppSetFocus`: if that autosave exists and the game is at the main menu (i.e. webOS killed the app), offer to resume.

**Acceptance:** Swipe-up to card view, switch apps, come back — game resumes. Force-close the card mid-mission, relaunch — autosave offers to resume.

---

### Phase 7 — Packaging

#### 7.1 `game/sdl/webos/package/appinfo.json`
```json
{
  "id": "com.example.hostiletakeover",
  "version": "1.0.0",
  "vendor": "Hostile Takeover Community",
  "type": "pdk",
  "main": "HostileTakeover",
  "title": "Hostile Takeover",
  "icon": "icon.png",
  "uiRevision": 2
}
```
`type: pdk` tells webOS this is a native (PDK) app, not a Mojo/Enyo HTML5 app.

#### 7.2 Package layout
```
hostiletakeover-pkg/
├── appinfo.json
├── icon.png
├── HostileTakeover            # the ELF
├── htdata832.pdb              # game data
└── htsfx.pdb                  # sound data
```

#### 7.3 `package/package.sh`
```bash
#!/bin/sh
set -e
STAGE=$(mktemp -d)
cp appinfo.json icon.png "$STAGE/"
cp ../../../../webos-build/HostileTakeover "$STAGE/"
cp ../../../../htdata832.pdb ../../../../htsfx.pdb "$STAGE/"
palm-package "$STAGE" -o "$PWD"
rm -rf "$STAGE"
```

#### 7.4 Install
```
palm-install com.example.hostiletakeover_1.0.0_all.ipk
```

#### 7.5 Community catalog submission
Stretch goal. The webOS-Ports / Preware catalog accepts `.ipk` PRs with a feed entry. Document the process in `game/sdl/webos/README.md`; don't automate.

**Acceptance:** A single `make && ./package.sh` produces an `.ipk` that `palm-install` accepts and that launches from the TouchPad launcher.

---

### Phase 8 — Stretch: multi-touch via PDL

The MVP works with single-finger taps. For drag-box selection + pinch-zoom on the minimap, add real multi-touch:

1. In `game/sdl/host.cpp`'s SDL event loop, after handling mouse events, poll `PDL_GetNumFingers` / `PDL_GetFinger` each tick.
2. Map finger #2 to the second-finger pen events (`penDownEvent2`, `penMoveEvent2`, `penUpEvent2`) — these already exist (host.cpp lines 122–145, the coalescing logic).
3. Reuse the existing `gtouches[]` tracking from the SDL2 branch where possible — port the data structures, swap the event source.

Defer until Phase 1–7 are stable.

---

### Phase 9 — Stretch: long-press for right-click context

Detect a stationary `penDownEvent` held >500ms; synthesize the game's "secondary action" event. Implement in `game/fingerhandler.cpp` (the natural home — it already does drag-vs-tap discrimination).

---

## 6. File Manifest

### New files
- `game/sdl/webos/makefile`
- `game/sdl/webos/hosthelpers.cpp`
- `game/sdl/webos/webos_lifecycle.cpp` (or fold into hosthelpers.cpp if small)
- `game/sdl/webos/mp_stubs.cpp`
- `game/sdl/webos/README.md`
- `game/sdl/webos/package/appinfo.json`
- `game/sdl/webos/package/icon.png`
- `game/sdl/webos/package/package.sh`

### Modified files (behind `__WEBOS__` / `NO_MULTIPLAYER` defines)
- `game/sdl/display.cpp`
- `game/sdl/host.cpp`
- `game/sdl/main.cpp`
- `game/sdl/sdleventserver.h` (small fixes if SDL 1.2 PeepEvents semantics differ)
- `game/Shell.cpp` (hide MP menu entries)
- Possibly `game/mainform.cpp` or similar (TBD by inspection)

### Untouched
- `game/sdl/sdlsounddev.cpp` — already SDL 1.2-compatible
- `game/sdl/sdlpackfile.cpp`, `game/filepdbreader.cpp` — pure POSIX
- `game/sdl/savegame.cpp` — pure POSIX
- All gameplay code under `game/*.cpp` (Tank, Miner, Simulation, etc.)
- `base/`, `mpshared/`, `inc/`, `yajl/`

---

## 7. Risk Register

| Risk | Likelihood | Mitigation |
|---|---|---|
| PDK SDL 1.2 gives only 16bpp video mode | Medium | Phase 2.2 implements 565 conversion as fallback |
| GCC 4.x rejects C++11 features used in newer code | Low | Code already builds under NDK 4.8 with `-std=c++0x`; same flag here |
| `base::Thread` (custom thread wrapper) uses Linux-only ABI quirks | Low | webOS is Linux underneath; should work — verify on first link |
| Linker errors from MP exclusion (dangling symbols) | High | Expected — iterative stubs in `mp_stubs.cpp` resolve them |
| HUD breaks at non-800×600 | Medium | Option B (letterboxed 800×600) sidesteps entirely; only re-evaluate if Phase 7 ships |
| SDL_PeepEvents semantics differ between 1.2 and 2.0 | Medium | sdleventserver.h is small; rewrite event poll if needed |
| Audio underruns on PDK's audio backend | Medium | Increase buffer count from 4 to 8; profile if it persists |
| Toolchain unavailable / PDK install broken | Medium | Phase 0 is gated; if it fails, stop and report — don't try workarounds |

---

## 8. Per-Phase Acceptance Checklist

Claude Code must explicitly confirm each acceptance before moving on:

- [ ] **P0** `arm-none-linux-gnueabi-g++ --version` works; SDL 1.2 + PDL headers present
- [ ] **P1** ARM ELF builds; `file` confirms architecture; binary not yet run
- [ ] **P2** Game launches on device, shows splash + main menu (touch may not work)
- [ ] **P3** Main menu navigable; start a mission; select & move units; complete a mission
- [ ] **P4** Audio plays cleanly during a full mission
- [ ] **P5** Save game from menu, force-quit, relaunch, load save — works
- [ ] **P6** Card-view swipe + return resumes mid-mission state; force-close restores via autosave
- [ ] **P7** `.ipk` installs via `palm-install`; app icon appears in launcher; app launches from launcher (not just CLI)
- [ ] **P8 (stretch)** Two-finger drag works for box-select
- [ ] **P9 (stretch)** Long-press triggers context menu / secondary action

---

## 9. What Claude Code Should NOT Do

- **Don't** edit gameplay code (anything under `game/*.cpp` other than `Shell.cpp` / form menu hookups) without explicit reason — the goal is a port, not a refactor.
- **Don't** add `#ifdef __WEBOS__` outside the SDL platform layer + Shell menu. Per-platform branches in game logic are a smell here.
- **Don't** add libcurl back. If something seems to need it, it's a multiplayer codepath that should be excluded instead.
- **Don't** rewrite the build system globally. The webos makefile is self-contained; don't touch `game/sdl/linux/makefile` or the Xcode projects.
- **Don't** "modernize" code while passing through it. Pre-existing memory-management patterns stay (recent commit history shows the maintainer is actively doing this work; don't conflict).
- **Don't** create a PR — the user will review locally and decide.
- **Don't** push to `main` or any branch other than the assigned feature branch.

---

## 10. Open Questions to Flag Back to User

The following are decisions Claude Code should **ask** the user about rather than guess:

1. **App ID**: `com.example.hostiletakeover` is a placeholder. Real submission to webOS Ports needs a stable, owned reverse-DNS ID. Confirm before Phase 7.
2. **Icon**: `package/icon.png` — does the user have official 64×64 art, or should Claude Code reuse `assets/iphone_icon.psd` flattened?
3. **PDK location**: Path to the installed PDK on the dev machine. Affects the makefile's `PDK` variable.
4. **Resolution decision**: After Phase 2 works, run option B and decide whether to attempt option A (native 1024×768) in a follow-up.
5. **Single-player only forever, or eventually re-enable MP?** If the latter, Phase 1's file-exclusion approach is correct (reversible). If never, the multiplayer files can be deleted in a cleanup pass after Phase 7.
