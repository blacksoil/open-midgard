# Developer Onboarding

This document gives a seasoned C++ developer enough context to start making changes to OpenMidgard without having to read the source tree cold. It covers what the project is, how it's put together, how the pieces fit, and the non-obvious conventions that will otherwise cost you an afternoon each.

Read this once, end to end, before touching code. It is long on purpose — the codebase has a lot of implicit context (decompiler origin, dual UI paths, a runtime-selected packet protocol, multiple render backends) that isn't visible from any single file.

---

## 1. What this project is

OpenMidgard is a from-scratch C++17 **reimplementation of the 2008-era Ragnarok Online client**, targeting the pre-Renewal client generation specifically (packet family `2008-09-10aSakexe`, `packet_ver 23`), not the later Renewal client. It is not a fork of the original client's source — Gravity never released client source. Instead, this codebase was built by **decompiling/analyzing the original client binary** (internally called `HighPriest.exe` in comments and commit history) and reconstructing equivalent C++ source, then modernizing the platform/render layer around that reconstruction.

This provenance matters more than it would in a normal project — see §2.

Current direction (from `README.md` / `TODO.md`):
- Ship a playable classic-2008-feel client with a modern, cross-platform renderer.
- Support multiple graphics backends simultaneously (legacy Direct3D7/DirectDraw, Direct3D11, Direct3D12, Vulkan), selectable at runtime.
- Windows is the most mature build target; Linux (Qt6 + Vulkan) is the actively-developed second platform; macOS has a manual, less-exercised CMake path.
- The project is still under active construction — `TODO.md` tracks module-by-module completion (~33/145 originally-scoped modules have real implementations; many areas are intentionally partial). Don't assume a file exists just because the original client had an equivalent; check `TODO.md` and the actual `src/` tree.

## 2. Read this first: provenance and codebase conventions

**This is a decompiler-origin codebase.** You will see comments like `// Memory layout from HighPriest.exe.h:30992` and `static_assert(sizeof(SomeStruct) == 108, "...")` throughout `src/world`, `src/res`, `src/session`, `src/network`. These are not decoration — they document that a struct's field order, size, or wire layout is pinned to match either:
- the original client binary's in-memory layout (for structs the decompiler reconstructed), or
- the RO network protocol's on-the-wire byte layout (`#pragma pack(push,1)` structs in `src/network/Packet.h`), or
- a GRF/map asset file format (`src/res/WorldRes.h` and friends).

**Do not reorder, resize, or "clean up" these structs** without first confirming (via the `static_assert`, a size comment, or the packet alignment docs — §9) that nothing depends on the exact layout. A refactor that looks like a harmless tidy-up can silently break wire compatibility or asset parsing.

Consequences of the decompiler origin you should expect and not be alarmed by:
- MFC-flavored naming: `CGameActor`, `CFile`, `CGPak`, Hungarian-ish prefixes (`m_dwXxx`, `m_isSitting`).
- Public data members with no encapsulation in many "reconstructed" classes.
- Global singletons everywhere (`g_world`, `g_session`, `g_renderer`, `g_fileMgr`, `g_modeMgr`, `g_windowMgr`, `g_resMgr`, `g_pathFinder`, `g_skillMgr`, ...) rather than dependency injection. This is the dominant architecture pattern in the whole codebase — don't fight it locally, follow it.
- Two real global-variable typos that are permanent and load-bearing — grep will fail if you assume the "correct" spelling:
  - `g_ttemmgr` (item manager, not `g_itemMgr`)
  - `g_buabridge` (Lua bridge, not `g_luaBridge`)
- Compiler warnings are deliberately suppressed on MSVC for this reason (`CMakeLists.txt`: `/wd4100 /wd4189 /wd4996` with the comment *"Disable warnings that are unavoidable in a decompiler-origin codebase"*).

**A `Ref/` folder is referenced constantly in commit messages, docs, and `.cursor`/`.github` instruction files, but it is not part of this repository** (it's `.gitignore`d). It's local reference material the original author keeps beside the repo:
- `Ref/` (decompiled original client, `HighPriest.exe.*`)
- `Ref/eAthena_src_2011` — an eAthena server emulator source tree, used to cross-check protocol/opcode behavior
- `Ref/RunningServer` — an actual running server tree including `packet_db.txt`, the ground truth for opcode-to-packet-version mapping
- `Ref/GRF-Content` — unpacked GRF asset reference

**You will not have `Ref/` by default.** If you need to verify packet layouts, opcode mappings, or original client behavior and don't have this folder, say so explicitly rather than guessing — the project's own workspace rules (`.github/instructions/*.md`, mirrored in `.cursor/rules/*.mdc`) explicitly say to check `Ref/` before guessing on packet/protocol/asset questions. Those rule files are worth skimming once; they encode the team's actual working conventions:
- `implementation_rules` — prefer complete, production-ready implementations over stubs/placeholders.
- `reference_lookup` — check `Ref/` before guessing on client/server/protocol/asset behavior.
- `packet_alignment` — always check `PACKET_VERSION_ALIGNMENT.md` before making packet decisions; don't assume the decompiled `Ref` client matches the intended `packet_ver 23` target.
- `media_file_handling` — never directly render/view media assets; inspect them byte-level.
- (`.cursor` only) `build-mode` — default build/verify flow is `cmake --preset vs2022-x64 -DRO_ENABLE_DEV_DEPLOY=ON` then `cmake --build --preset build-release-x64`.

## 3. Repository layout

```
CMakeLists.txt / CMakePresets.json   top-level build config
docs/                                 focused implementation notes (this file lives here)
cmake/                                CMake helper modules (Qt deploy, best-effort copy)
third_party/qt/                       Qt LGPL compliance notices (no vendored Qt source)
src/
  main/          application entry points (Win32 + POSIX/Qt), window/message loop
  core/           foundational utilities: file I/O, GRF archives, hashing, ini settings, timers, XML, dll loading
  platform/       Windows-API compatibility shim for non-Windows builds
  session/        CSession — all client-side "who am I / what do I have" player state
  network/        socket layer, packet framing/dispatch, packet-version profile abstraction
  cipher/         GRF archive decryption (DES-variant) + MD5
  security/       stub — packet encryption/checksum layer, not implemented
  gamemode/       top-level state machine: login flow, in-map gameplay state, packet routing
  world/          in-world entities: actors, players, items, skills-as-effects, map/ground data, pathing state
  pathfinder/     A*-based pathfinding over map attribute/collision data
  item/           static item metadata tables (names, descriptions, resources) — not in-world item instances
  skill/          static skill metadata tables — not in-world skill-effect instances
  lua/            Lua 5.1 VM bridge — reads original game's Lua data tables (skill effect IDs etc.)
  render/         backend-agnostic scene renderer + GDI-based 2D/sprite compositing
  render3d/       graphics backend abstraction + implementations (D3D7 legacy, D3D11, D3D12, Vulkan) + shaders
  res/            typed game-asset loaders (sprites, models, ground meshes, textures, world descriptors, ...)
  ui/             native retained-mode widget-tree UI (~87 files) — the UI system that actually ships
  qtui/            experimental Qt6/QML overlay compositor — not a UI replacement yet, see §11
  audio/          audio engine (pluggable backend: Miles legacy / miniaudio modern) + Bink video
  input/          stub — not the real input path today, see §14
```

Each `src/<subsystem>/` directory is built as its own static library (`add_subdirectory` + `add_library(ro_<name> STATIC ...)` in that directory's own `CMakeLists.txt`), all linked into one final executable target `open_midgard` (output binary name `open-midgard`). There is no glob-based source discovery — **to add a new source file, you must add it explicitly to that subsystem's own `CMakeLists.txt`.**

## 4. Build system

Requirements: CMake ≥ 3.20, a C++17 compiler, Git, and (for actually running the client) a legally obtained RO client data/runtime setup — the client looks for `data.grf` / `data/clientinfo.xml` / `data/` near the executable, current working directory, or a path set via `OPEN_MIDGARD_DATA_DIR`.

### Presets (`CMakePresets.json`)

| Preset | Platform | Notes |
|---|---|---|
| `vs2022-win32` / `vs2022-x64` | Windows, MSVC | native D3D11/D3D12 backends available; build dirs `build` / `build64` |
| `mingw-qt-x64` / `-debug` | Windows, MinGW | Qt6/QML spike enabled; hardcoded to a specific local `C:/Qt/...` layout — adjust `RO_QT_ROOT` if yours differs |
| `mingw-qt-x64-ci` | Windows, MinGW | same as above but `RO_ENABLE_DEV_DEPLOY=OFF`, no hardcoded compiler paths — the CI-oriented preset |
| `linux-qt-vulkan` | Linux | Ninja, Release, Qt6 UI forced on, Vulkan-only 3D backend, dev-deploy off — the relevant preset for a Linux dev box |

Configure + build (Linux example):
```bash
cmake --preset linux-qt-vulkan
cmake --build --preset build-linux-qt-vulkan -j
```
macOS has no checked-in preset; configure manually with `-DCMAKE_PREFIX_PATH=/path/to/Qt/6.x/macos` (see `README.md`).

The project's own documented default "build and verify" flow (from `.cursor/rules/build-mode-rules.mdc`) is the x64 MSVC Release preset with dev-deploy on:
```powershell
cmake --preset vs2022-x64 -DRO_ENABLE_DEV_DEPLOY=ON
cmake --build --preset build-release-x64
```

### Key CMake options

| Option | Default | Effect |
|---|---|---|
| `RO_ENABLE_DEV_DEPLOY` | ON (Win), forced OFF (non-Win) | Post-build copies the exe (and Qt runtime) to a hardcoded local path `D:/Spel/OldRO` — a specific developer's local RO install folder for quick manual testing. Turn this off if you don't have that path. |
| `RO_ENABLE_QT6_UI` (+ deprecated alias `RO_ENABLE_QT6_UI_SPIKE`) | OFF (Win default), forced ON (non-Win) | Builds `src/qtui/` and links Qt6. Non-Windows platforms force this on because Qt provides the base windowing layer there, not just the overlay spike — see §11. |
| `RO_QT_UI_DEFAULT_ENABLED` | OFF | Whether the Qt UI **runtime path** is on by default at process start, unless overridden by the `OPEN_MIDGARD_QT_UI` env var. Distinct from the build-time flag above. |
| `RO_ENABLE_NATIVE_D3D11` / `RO_ENABLE_NATIVE_D3D12` | ON | Only actually active on `WIN32 AND MSVC`; auto-disabled elsewhere, where Vulkan is the modern 3D backend instead. |
| `RO_ENABLE_MILES_AUDIO` / `RO_ENABLE_BINK_VIDEO` / `RO_ENABLE_GRANNY` / `RO_ENABLE_IJL` | ON/ON/ON/OFF, all forced OFF on x64 | Optional legacy 32-bit-only Windows middleware DLLs (Miles audio, Bink video, Granny models, Intel JPEG). Loaded best-effort at runtime via `CDllMgr`; their absence is non-fatal. Dead weight on the x64-only future direction (see `X64_MILESTONE.md`). |
| `RO_QT_ROOT`, `RO_QT_WINDEPLOYQT_EXECUTABLE` | — | Point at your Qt6 kit if it's not in the hardcoded default location. |

All of the above become `#if`-testable compile definitions (`RO_PLATFORM_WINDOWS`, `RO_HAS_NATIVE_D3D11`, `RO_ENABLE_QT6_UI`, etc.) — grep for these when you need to know what code path is live under a given configuration.

**Legacy DirectX 7 SDK**: the original client used DirectDraw4/Direct3D7. Set `DX7_SDK_DIR` env var if you need the real DX7 SDK headers; otherwise the build falls back to system Windows SDK headers for that legacy backend.

**x64 milestone note** (`X64_MILESTONE.md`): the first x64 milestone deliberately drops legacy middleware (Miles/Bink/Granny/IJL — all 32-bit only) and GRF `0x200`-compressed-index decompression now goes through linked `zlib` instead of `cps.dll`. Use Win32 only when you specifically need to validate old-middleware behavior.

## 5. Application lifecycle

Entry point selection happens in `src/main/CMakeLists.txt`: `WinMain.cpp` (Windows, real `WinMain`) or `AppMain_stub.cpp` (POSIX/Qt, real `main(argc, argv)`). Both implement the same contract declared in `src/main/WinMain.h`.

Non-Windows startup sequence (`AppMain_stub.cpp::main`), the one you'll be debugging on Linux:

1. **Resolve the runtime data root** (`ResolveRuntimeRoot()`): checks, in order, the `RO_DEV_DEPLOY_DIR` compile define, the `OPEN_MIDGARD_DATA_DIR` env var, the current working directory, and the executable's own directory — walking parent directories looking for `data.grf`/`clientinfo.xml`/`data/`/etc. First match wins and the process `chdir`s there. This is how the client finds game assets with no installer.
2. Init timer/math tables, read `OpenMidgard.ini` settings (`ReadRegistry()` — the name is legacy but it now reads an ini file, not the Windows registry, via `SettingsIni.h`).
3. `InitApp()` creates the main window (via the Qt-backed `RoQtCreateMainWindow` on non-Windows).
4. `CDllMgr::LoadAll()` — best-effort optional-DLL loading (no-op-ish on non-Windows).
5. Mount GRF archives: `g_fileMgr.AddPak("data.grf")` (+ a few more).
6. `CConnection::Startup()` — network subsystem init.
7. **`InitClientSystems()`** — the real engine boot: registers every resource-type loader on `g_resMgr` (`.gat`/`.gnd`/`.bmp`/`.rsm`/`.spr`/`.act`/`.pal`/`.rsw`/`.wav` → concrete `CXxxRes` classes), inits `CAudio`, inits `g_windowMgr` (native UI), inits Lua (`g_buabridge.Initialize()` + preloads bootstrap `.lub` scripts), initializes the active render backend (`GetRenderDevice().Initialize(...)`), inits `g_renderer`, and finally boots the Qt overlay runtime (`InitializeQtUiRuntime`).
8. `InitTimer(60)` — 60 FPS target.
9. **`g_modeMgr.Run(0, "")`** — hands control to `CModeMgr` (`src/gamemode/`), which owns the actual frame/message loop from here on. Everything above this line is one-time boot; everything below is per-frame.
10. On quit: teardown roughly mirrors init in reverse (`CConnection::Cleanup()`, `g_windowMgr.Reset()`, `g_buabridge.Shutdown()`, render device shutdown, Qt runtime shutdown, audio shutdown).

**Input plumbing**: even on non-Windows, `WindowProc` handles Win32-style messages (`WM_MOUSEMOVE`, `WM_LBUTTONDOWN`, `WM_CHAR`, etc.) — Qt's platform layer synthesizes these. A message first goes to the Qt UI runtime, then to `g_windowMgr` for native-UI hit testing, and only if unconsumed does it become a `g_modeMgr.SendMsg(CGameMode::GameMsg_*, ...)` game input event.

**Self-relaunch**: `RelaunchCurrentApplication()` (used for settings changes that require a restart, e.g. renderer/AA changes) does `fork()+execv()` of the same binary with the original launch args on POSIX.

**Crash handling** (Windows only): `WinMain.cpp` has a SEH-based minidump writer (`WriteUnhandledCrashDump`/`TryWriteCrashDumpFile`) using `MiniDumpWriteDump` loaded dynamically.

## 6. Core utilities (`src/core/`)

Built as `ro_core`, depended on by nearly everything.

- **`File.h/.cpp`** — `CFile` (buffered file I/O), `CMemFile`/`CMemMapFile` (memory-mapped views, back GRF access), **`CFileMgr`** (`g_fileMgr`) — the front door for all asset loading: `AddPak(...)` mounts a GRF, `GetData(fileName, &size)` returns a malloc'd buffer read from a mounted archive or loose disk file (loose-file-first vs archive-first controlled by `g_readFolderFirst`).
- **`GPak.h/.cpp`** — `CGPak`, the GRF archive reader. Handles Gravity's format versions 0x100–0x103, both DES-encrypted-index (`OpenPak01`) and zlib-compressed-index (`OpenPak02`) variants; per-file decryption via `MakeSeed`/nibble-swap; binary-search lookup by hashed filename. If you touch asset loading, read this file fully — it's a self-contained reimplementation of a real (obfuscated) archive format.
- **`Hash.h/.cpp`** — `CHash`, the filename-hash type used as the GRF index key.
- **`SettingsIni.h/.cpp`** — `OpenMidgard.ini` read/write (the project's own settings store, separate from the original client's registry-based config).
- **`Timer.h/.cpp`** — frame timing, frame-skip support.
- **`Xml.h/.cpp`** — a small hand-rolled XML parser built specifically for `clientinfo.xml`-style files, not general-purpose.
- **`Locale.cpp`** (decl in `ClientInfoLocale.h`) — parses `clientinfo.xml`, exposes the list of selectable server "worlds" at login, GM/dev account name-coloring lists.
- **`DllMgr.h/.cpp`** — `CDllMgr`, Windows-only best-effort loader for the optional legacy middleware DLLs (Miles/Bink/Granny/IJL). Failure to load is reported (`GetLastLoadReport()`) but non-fatal.
- **`Globals.h/.cpp`** — misc extern global state: service/server type enums, language/codepage, login server address, GM account-ID lists, and the `extern CSession g_session;` declaration (session lives in `src/session/`, declared here).

**MSVC build quirk**: `ro_core`'s `CMakeLists.txt` force-includes `Types.h` (`/FI`) as the very first header to resolve an ordering conflict between `<windows.h>`'s `lconv` and newer MSVC `<clocale>`. If you see mysterious `lconv`/`DWORD` redefinition errors, this is the mechanism to check.

## 7. Platform abstraction (`src/platform/`)

There is **no runtime platform-abstraction interface** (no `IPlatform` class) — the strategy is a **compile-time compatibility shim** that lets originally-Win32-only decompiled code compile mostly unmodified on Linux/macOS:

- **`WindowsCompat.h`** — on Windows, a passthrough to real `<windows.h>`. On other platforms, a large hand-written shim providing Win32 types/macros the rest of the codebase expects: `POINT`, `RECT`, `MSG`, `BITMAPINFO`, `CRITICAL_SECTION`, `RGB`/`GetRValue` macros, `ZeroMemory`, `HKEY_CURRENT_USER`, etc. It pulls in `qtui/QtPlatformWindow.h` and POSIX equivalents (`dlfcn.h`, `mach-o/dyld.h`) to implement the semantics.
- **`src/platform/win32/`** — a *separate*, tiny directory containing forwarding stubs (`windows.h`, `winsock2.h`, `mmsystem.h`, etc.). The top-level `CMakeLists.txt` does `include_directories(BEFORE .../src/platform/win32)` when `NOT WIN32`, so any file anywhere in the codebase doing `#include <windows.h>` transparently picks up this stub, which forwards to the real shim in `WindowsCompat.h`.

**Gotcha**: there are two "windows.h"-like things in this repo. The actual compat implementation is `src/platform/WindowsCompat.h`; `src/platform/win32/windows.h` is just the tiny forwarding stub injected ahead of the system include path. If you're debugging a missing-symbol issue on non-Windows related to Win32 types, look in `WindowsCompat.h`, not the stub.

## 8. Session state (`src/session/`)

**`CSession`** (`Session.h`, global `g_session`) is a single large class holding essentially all client-side "who am I, what do I look like, what do I own" state. Despite the name, this is **not** an auth/network session object — it's the live in-memory model of the currently-connected player, read and written directly by UI code, world code, gamemode code, and network packet handlers alike. It's the most likely file to need touching for any "expose new player data to the UI" feature, and correspondingly the place most prone to merge conflicts.

Contents, grouped:
- **Account/auth**: user id/password, account ID, auth code, char-server address, list of selectable servers.
- **Appearance/stats**: position/map, job/look/palettes/equip-slot visuals, full stat block, derived combat stats.
- **Inventory/storage**: `std::list<ITEM_INFO>` with add/remove/wear-location helpers.
- **Skills**: three separate lists — player, homunculus, mercenary.
- **Social**: party and friend lists.
- **NPC shop interaction**: buy/sell UI state machine.
- **Shortcuts**: 3 pages × 9 slots of skill-or-item hotbar bindings.
- **Status icons**: active buffs/debuffs with expiry pruning.
- **Job/sprite name resolution**: translates job IDs into sprite resource filenames for rendering (`GetJobActName`, `GetJobSprName`, etc.) — a key integration point between session state and the resource/render layer. Backed by a large generated table, `session/JobNameTable.generated.inc` — treat as generated, find its source-of-truth generator before hand-editing it (not present in this pass's scope; ask/search before editing).

## 9. Networking and packet protocol

This is the subsystem most likely to bite you if you skip context, because the wire protocol is **runtime-version-selectable** and the send/receive sides are maintained somewhat independently.

### Layering

- **`src/network/Connection.h/.cpp`** — raw socket layer. `CConnection` (non-blocking TCP wrapper) and `CRagConnection : public CConnection` (protocol framing, singleton `CRagConnection::instance()`). Reconnects to a new host/port at each phase (login → char → map/zone), driven by handoff packets like `0x0071`.
- **`src/network/PacketQueue.h/.cpp`** — `CPacketQueue`, a growable byte ring buffer (starts 64KB, doubles on overflow) for send/recv buffering.
- **`src/network/GronPacket.h/.cpp`** — the receive-side **packet-ID → size lookup table** (`ro::net::GetPacketSize`), a flat `std::array<s16, 0x10000>` indexed by packet ID. This is the single source of truth for how many bytes to read for a given opcode.
- **`src/network/Packet.h`** — raw wire-format structs, `#pragma pack(push,1)`, plus the `PacketProfile` namespace holding per-packet-version opcode/size constant sets.
- **`src/network/MapSendProfile.h/.cpp`** — the runtime packet-version abstraction: `PacketVersionId` enum (`PacketVer23`, `PacketVer22`, ...), profile structs, and `BuildActiveXxxPacket(...)` helpers that serialize a version-correct packet without the caller branching on raw opcodes.
- **`src/gamemode/LoginMode.cpp`**, **`GameMode.cpp`**, **`GameModePacket.h/.cpp`** — the actual login/char/map state machine and packet dispatch live here, *not* in `src/network/`. `src/network/` only provides socket/framing primitives.

### Receive framing (`CRagConnection::RecvPacket`)

1. Drain available bytes into the recv queue (non-blocking).
2. Peek the 2-byte little-endian packet ID.
3. Look up size via `GronPacket`'s table:
   - `0` → unknown ID: **drop 2 bytes and keep scanning** (so one unknown packet doesn't stall the whole stream); logged once per unique unknown ID.
   - `-1` (variable-length sentinel) → read a 2-byte length field at offset 2.
   - otherwise → fixed size.
4. Wait for the full packet to be buffered, then pop it.

There's a deliberate, non-dead debug aid built into this path: `Connection.cpp` logs the first 24 received packets on every fresh connection to `debug_hp_<pid>.log`, plus an 8-entry ring buffer of recent packet id/size pairs. Use this when diagnosing "server sent something unexpected."

### Dispatch

`CGameModePacketRouter` (`GameModePacket.h`) is a `std::unordered_map<u16, std::function<void(CGameMode&, const PacketView&)>>`. `RegisterDefaultGameModePacketHandlers(router)` is the single place all default handlers get wired up at startup. **To add a new packet handler**: register it there, and if it's a genuinely new packet ID, add its size to `FillPacketSizeTable` in `GronPacket.cpp` first — otherwise it's silently dropped as unknown.

### Sending

Version-insensitive packets are built as plain `#pragma pack(1)` structs from `Packet.h` and sent via `CRagConnection::instance()->SendPacket(...)`. **Version-sensitive gameplay packets should go through the `BuildActiveXxxPacket(...)` helpers in `MapSendProfile.cpp`** rather than hand-rolling a struct, since those already resolve the opcode/layout for whichever profile is active.

### Packet versioning — the single most important thing to understand here

Target profile: **`packet_ver 23`**, client date **`2008-09-10aSakexe`** (`PACKET_VERSION_ALIGNMENT.md`). "Sakexe" is an eAthena/rAthena packet-db naming convention for a specific pre-Renewal client build family; it is a **different, overlapping-in-date sibling family** to the early-Renewal "RagexeRE" family, with different opcodes for some of the same logical requests. This distinction is the source of the most confusing class of bug in this codebase: a server that isn't guarded correctly can silently start speaking the wrong opcode family for a client of roughly the same era, causing dropped packets or apparent random behavior with no obvious error (`rAthena_OpenMidgard_patch.md` documents the server-side guard needed if you're standing up a compatible rAthena test server).

Example of how much opcodes differ between adjacent "versions" for the *same* logical request:

| Function | packet_ver 22 | packet_ver 23 (target) |
|---|---|---|
| `wanttoconnection` | `0x009B / 26` | `0x0436 / 19` |
| `actionrequest` | `0x0190 / 19` | `0x0437 / 7` |
| `useskilltoid` | `0x0072 / 25` | `0x0438 / 10` |
| `useitem` | `0x009F / 14` | `0x0439 / 8` |
| `walktoxy`, `changedir`, `ticksend`, `getcharnamerequest`, `globalmessage` | unchanged between the two | unchanged |

**Send-side and receive-side knowledge are maintained separately and can drift.** The explicit project rule (`PACKET_VERSION_ALIGNMENT.md`): don't force the receive table to a "pure packet_ver 23 fantasy" — keep receive parsing aligned with what the server *actually* sends (validated from live `debug_hp_<pid>.log` captures), which as of the last audit was still a mixed older eAthena-style actor stream in places (e.g. `0x0078=55`, `0x02EE=60`) even while the client sends packet_ver 23 on the wanttoconnection/action/skill/item family. When something breaks, check the live receive trace before assuming the send-side code is wrong, and vice versa.

`TODO.md`'s "Packet Coverage Audit" section has the current up-to-date bucket list of which opcodes are implemented, safe-ignored, or still unregistered — check it before assuming a packet needs new code versus already having a handler.

**Files to touch first when doing packet work** (per the alignment doc): `src/network/Packet.h`, `src/gamemode/LoginMode.cpp` / `GameMode.cpp`, and `src/network/GronPacket.cpp` (only if receive-side validation shows a missing/wrong size).

### Cipher / security

- **`src/cipher/CDec.h/.cpp`** — a DES-derived cipher used **for GRF archive decryption**, not network packets (custom single-S-box variant matching Gravity's original GRF obfuscation).
- **`src/cipher/Md5.h/.cpp`** — standard MD5, used for login password hashing / exe-hash-check style packets.
- **`src/security/Security.h/.cpp`** is an explicit stub (`CSecurity`) — **there is no live network packet encryption/checksum layer implemented today.** Comment says "implement from Ref/Security.cpp." If you're asked to add packet obfuscation, this is the gap.

## 10. Rendering architecture

Two layers, not a 2D/3D split:

- **`src/render3d/`** — the **device abstraction layer**, talking to the actual graphics API.
  - `RenderBackend.h/.cpp` — `enum class RenderBackendType { LegacyDirect3D7, Direct3D11, Direct3D12, Vulkan }` plus selection/bootstrap (`GetConfiguredRenderBackend`, `SetConfiguredRenderBackend`, `InitializeRenderBackend`). Single source of truth for "which backend is active."
  - `RenderDevice.h` — `IRenderDevice`, a pure-virtual interface deliberately modeled on the **legacy Direct3D7 fixed-function pipeline** (`D3DTRANSFORMSTATETYPE`, `D3DRENDERSTATETYPE`, `D3DPRIMITIVETYPE` types are reused even by the modern backends). Every backend implements this same interface, meaning D3D11/D3D12/Vulkan implementations have to translate fixed-function state into their own pipelines rather than exposing anything more modern. `GetRenderDevice()` returns the process-wide singleton.
  - `Device.h/.cpp` — `C3dDevice` (`g_3dDevice`), the legacy DirectDraw7 + Direct3D7 device wrapper — the original 2008-era path, still live as `RenderBackendType::LegacyDirect3D7`. `Device_nonwin.cpp` is the non-Windows no-op fallback; `Dx7Compat.h` shims the D3D7/DirectDraw types on non-Windows so the same interface code compiles everywhere.
  - `shaders/` — only post-process/AA shaders are checked in as loose source (`vulkan_post_fxaa.*`, `vulkan_post_smaa.hlsl`, `vulkan_world.hlsl`); the D3D11/D3D12 world/post-process HLSL is embedded directly in `RenderDevice.cpp`. Vulkan SPIR-V is **generated** into `*.generated.h` headers by a build-time script — edit the `.hlsl`/`.frag`/`.vert` source and regenerate; don't hand-edit the generated headers.
- **`src/render/`** — the **scene/primitive layer built on top of `IRenderDevice`**, backend-agnostic:
  - `Renderer.h/.cpp` — `CRenderer` (`g_renderer`), the actual world/model renderer: owns per-frame primitive lists bucketed by pass (alpha, emissive, lightmap ground/light, raw), camera/lighting/fog state, and a name-keyed texture cache (`CTexMgr`/`g_texMgr`) with LRU eviction.
  - `DC.h/.cpp` — **GDI-based 2D drawing**: an off-screen ARGB DIB surface you get an `HDC` for, plus blit/stretch/alpha-blend helpers. **This is still load-bearing on every backend, including the modern ones** — 2D sprite/actor billboards and the entire native UI (§11) are rasterized via GDI into a CPU buffer, then uploaded as a texture and drawn as a screen quad by `CRenderer`. Changes here affect all render backends simultaneously.

**Backend selection**: build-time (CMake forces Qt6 + disables native D3D11/D3D12 on non-Windows, where Vulkan is the 3D path) combined with a runtime user setting (`RenderBackendType` persisted, read in `InitializeRenderBackend`). Renderer/AA changes require an app restart (`GraphicsSettingsRequireRestart`, see `ANTI_ALIASING.md`) — don't expect these to hot-swap.

Anti-aliasing (FXAA/SMAA, per `ANTI_ALIASING.md`) runs only on the offscreen 3D scene target, **before** the GDI-composited UI/cursor is blitted on top — UI text and cursor are intentionally never anti-aliased. The legacy D3D7 backend has no AA option at all (hidden, not just disabled, in the option window).

## 11. UI system

There are **two** UI mechanisms in this repo; only one of them is the real, shipping UI today.

### The real UI: `src/ui/` (native retained-mode widget tree)

~87 files. Pattern: classic C++ retained-mode widget tree, explicitly reconstructed to match the original client's binary layout (headers cite exact `HighPriest.exe.h` IDA line offsets — struct field order and window-ID enum values are pinned, not idiomatic-first).

- **`UIWindow`** (`UIWindow.h/.cpp`) — base class for every widget: parent/child tree, dirty flag, a big virtual interface (`OnDraw`, `OnProcess`, `OnLBtnDown/Up`, `OnMouseMove`, `OnChar`, `OnKeyDown`, `DragAndDrop`), and a Win32-style `SendMsg(msg, wparam, lparam, extra)` message-passing method with recursive hit testing (`HitTestDeep`). Every concrete widget/window subclasses this.
- **`UIFrameWnd`** — common "framed window with titlebar/close button" base most top-level dialogs build on.
- **`UIWindowMgr`** (`g_windowMgr`) — the top-level controller: owns every top-level window instance by name, a binary-faithful `WindowId` enum, `MakeWindow`/`ToggleWindow` factory, input routing from the platform layer, modal/capture/edit-window tracking, dirty-state tracking for redraw skipping, and the `ArgbDibSurface` the whole native UI composites into (tying back into `render/DC.h`'s GDI path).
- Roughly categorized by filename: shop/trade, inventory/items, login/char-select flow, social, NPC interaction, HUD/chrome, skills, settings — plus `UiScale.h/.cpp` for DPI/UI-scale handling.

### The experimental path: `src/qtui/` (Qt6/QML overlay — not a replacement)

Gated by `RO_ENABLE_QT6_UI_SPIKE` (build) and `RO_ENABLE_QT6_UI` / `OPEN_MIDGARD_QT_UI` (runtime). **This does not replace `src/ui/` today** — it currently produces only a composited overlay layer (`CompositeQtUiMenuOverlay`, `CompositeQtUiGameplayOverlay`) that draws over the native UI, with a `QtUiStateAdapter` bridging `CGameMode`/menu state into QML-bound Qt objects (`qml/GameOverlay.qml`, `qml/SpikeOverlay.qml`). All entry points are no-ops when Qt support isn't compiled in — verify the flags before assuming Qt-side changes have any effect.

`docs/qt6_qml_migration_spike.md` is the authoritative design doc for where this is headed: the intended end-state replaces only the GDI *overlay* stage (damage numbers, hover labels, vitals, chat preview, login/char-select panels) with a GPU-composited Qt Quick scene, via a documented 6-phase cutover plan — **none of those phases are complete as of writing**. Inventory/equip/storage/shop-style windows (the bulk of `src/ui/`) are explicitly out of scope for the current migration plan.

Don't confuse this Qt *overlay spike* with the fact that **Qt6 is also the base windowing/event-loop layer on non-Windows** (forced on via CMake regardless of the spike flag) — those are two different uses of Qt in this codebase.

## 12. World, actors, and the game state machine

### Entity hierarchy (`src/world/GameActor.h`, `World.h/.cpp`)

`CGameObject` (base: `OnProcess`/`OnActorDeleted`/`SendMsg`/`Render`) → `CRenderObject` (sprite/animation/position state) → `CAbleToMakeEffect` (can attach `CRagEffect` visual effects) → `CGameActor` (stats, `CPathInfo m_path`, `m_gid`). Concrete leaves: `CPc` (player character, with 2D name/portrait billboard caching), `CPlayer : CPc` (the locally-controlled player), `CGrannyPc` (3D-model-rendered variant), `CSkill : CGameActor` (an *active skill effect instance* in the world — not the static skill table, see below), `CItem : CRenderObject` (a dropped ground item, not an actor).

`CWorld` (`g_world`) is the per-map container: actor/game-object/item/skill lists, ground/terrain (`C3dAttr*` for collision/height, `C3dGround*` for the visual mesh), background/sky, and a scene-graph node used to spatially bucket actors for position-based lookup.

### Despawn is deferred, not immediate — this matters

`CWorld::RegisterActor`/`UnregisterActor` and `NotifyActorDeleted` broadcast `OnActorDeleted` to dependents (effects, UI, targeting) before destruction. But the actual scheduling lives in `CGameMode::m_deferredActorDeletes` (`src/gamemode/GameMode.h`): packet handlers push actors onto this vector instead of deleting synchronously, and the per-frame update loop drains and deletes at a safe point (plus `CleanupPendingActorDespawns` for delayed-despawn timers). **Never `delete` a `CGameActor*` synchronously from inside packet-handling code — queue it.** (This is exactly the pattern the recent "Defer despawned actor deletion" commit reinforced.)

### Map data and coordinates

`.gat`-derived collision/height data lives in `C3dAttr` — a grid of `CAttrCell{h1..h4, flag}` (per-corner heights for slope interpolation, `flag` encoding walkability/warp/no-walk zones). Visual ground mesh comes from `.gnd`-derived data (`C3dGround`/`CGndRes`). Static placements (actors, particle/effect sources, sound sources, lighting, water) come from an `.rsw`-equivalent world descriptor (`C3dWorldRes`, `src/res/WorldRes.h`). Coordinates are integer cell-based for gameplay/pathing, float world-space for rendering; `CPathInfo`/`PathCell` carries a sequence of cells with expected arrival ticks driving movement interpolation.

### Two layers named similarly — don't conflate them

- **In-world instances**: `CItem`, `CSkill` (§ above) — live in `src/world/`.
- **Static metadata tables**: `CItemMgr` (`src/item/Item.h`, global `g_ttemmgr` — note the typo) and `CSkillMgr` (`src/skill/Skill.h`, global `g_skillMgr`) load name/description/resource/cost tables from data files and answer display/lookup queries. Inventory slot data itself (`ITEM_INFO`, card slots, refine level, identified flag) is a separate flat struct used for UI/network, distinct again from the `CItemMgr` static tables.

### State machine (`src/gamemode/`)

`CModeMgr` (`g_modeMgr`) owns exactly one active `CMode*`, switched via `Switch(newMode, worldName)`:
- **`CLoginMode`** — login screen through character select/creation/deletion/server-select, itself a sub-state-machine (`LoginSubMode`: Notice → Licence → AccountList → Login → ConnectAccount → ConnectChar → ServerSelect → CharSelect → MakeChar/SelectChar/DeleteChar → ZoneConnect). Handles login-server packets directly (`OnAcceptLogin` `0x0069`, `OnAcceptChar` `0x006B`, etc.) via `PollNetwork()`.
- **`CGameMode`** — in-map gameplay: owns `CWorld* m_world`, `CView* m_view` (orbit camera), input state, actor name/position caches, ground-item tracking, party/guild/compass overlay data, and a large `GameInputMessage` enum for UI-originated commands. Also owns `MapLoadingStage` (None → PresentScreen → BootstrapAssets → CreateView → SendLoadAck → AwaitActors) driving map transitions.

Packet dispatch for gameplay is `GameModePacket.h/.cpp` (`TryReadPacket` + `CGameModePacketRouter`, see §9). `CView` (`View.h`) is the orbit camera with screen↔world cell hit-testing and a billboard-frame cache to avoid re-rendering 2D nameplates every frame.

## 13. Pathfinding (`src/pathfinder/`)

`CPathFinder` (`g_pathFinder`) operates directly on a `C3dAttr*`. Two-phase: try a direct line-walk first (Bresenham-like), fall back to A* (priority-queue open set, octile-distance weighting — orthogonal cost 10, diagonal cost 14, corner-cutting disallowed unless both flanking cells are walkable). Output is a `CPathInfo` of timestamped cells. This is one of the few genuinely clean (non-decompiled) files in the tree — no binary-layout comments, self-contained.

## 14. Lua scripting (`src/lua/`)

`CLuaBridge` (`g_buabridge` — note the typo) wraps a Lua 5.1 VM with bytecode-compat shims for loading the **original game's Lua data scripts**. Its role is narrow: read-only extraction of values from Lua global tables (e.g. per-skill visual-effect-ID lookups via `GetSkillEffectInfoBySkillId`, cached). This is a **data/config reader**, not a general scripting or modding layer — there's no evidence of C++ gameplay logic being driven by Lua callbacks.

## 15. Audio, video, input

- **`src/audio/`** — `CAudio` (`GetInstance()`) is a thin facade over a pluggable `IAudioBackend`: `MilesAudioBackend.cpp` (original client's proprietary backend, structs reconstructed from the decompiler) or `MiniaudioBackend.cpp` (modern replacement, header-only `miniaudio.h`). Handles map-BGM resolution and 3D positional SFX. `Video.cpp/h` handles Bink cutscene video (also decompiler-reconstructed, unrelated to miniaudio).
- **`src/input/`** — `Input.h` is an explicit stub today ("implement from Ref/Input.cpp"). **Don't assume this is the real input path** — actual input currently flows through the Qt platform event layer feeding `CMode::SendMsg` (see §5), not through `CInput`.

## 16. Resource loading and GRF archives

Two layers:
- **Archive layer** — `CGPak` (§6) reads `.grf` files (versions 0x100–0x103, DES-encrypted or zlib-compressed index), individually decrypting/decompressing entries on demand, backed by `CMemFile`/`CMemMapFile`.
- **Typed resource layer** (`src/res/`) — `CRes` is the base loadable-resource interface (`Load`, `LoadFromBuffer`, `Clone`, ref-counted). `CResMgr` (`g_resMgr`) is the cache/factory: register a file extension with a prototype object (`RegisterType`), then `Get("foo.spr")` finds-or-loads-and-caches by hashed filename, with LRU-style eviction (`UnloadRarelyUsedRes`). Concrete types: `Sprite`/`ActRes` (2D sprite + animation tables), `GndRes` (ground mesh), `ModelRes` (3D models), `ImfRes`, `EzEffectRes` (particles), `PaletteRes`, `Bitmap`/`Texture`, `WorldRes` (the `.rsw`-equivalent map descriptor, with `static_assert`-pinned struct sizes for byte-for-byte format fidelity).

`clientinfo.xml`/GRF discovery and mounting is orchestrated from `src/main/` (`ResolveRuntimeRoot`, `AddPak` calls) and `src/gamemode/LoginMode.cpp` (`LoadClientInfoCandidates`) — that's where to look for "how does the client find its data" end to end.

## 17. Pitfalls checklist

Before you start editing, keep these in your head:

1. **Don't reorder/resize a struct with a `HighPriest.exe.h` comment or a `static_assert(sizeof(...) == N)`** without confirming nothing depends on the exact layout (binary compat, wire compat, or file-format compat).
2. **Actor deletion goes through `CGameMode::m_deferredActorDeletes`**, never a synchronous `delete` from packet-handling code.
3. **`g_ttemmgr`** and **`g_buabridge`** are the real (misspelled) global names — don't "fix" them in isolation; a rename needs to be repo-wide and deliberate.
4. **Send-side and receive-side packet knowledge can drift.** Check `TODO.md`'s packet coverage audit and `PACKET_VERSION_ALIGNMENT.md` before assuming a packet needs new code, and check the live `debug_hp_<pid>.log` receive trace before assuming code is wrong when the server "misbehaves."
5. **New packet IDs need an entry in `GronPacket.cpp`'s size table**, or they're silently dropped as unknown even if you register a handler.
6. **`src/input/` is a stub.** Real input flows through the platform/Qt event layer into `CMode::SendMsg`.
7. **The Qt overlay path (`src/qtui/`) is inert unless its build+runtime flags are both on.** Don't assume changes there have any visible effect in a default build.
8. **GDI (`render/DC.h`) is load-bearing on every render backend**, not legacy-only — changes there affect the native UI and all sprite/billboard rendering regardless of which 3D backend is active.
9. **Don't hand-edit `*.generated.h` shader headers** in `src/render3d/` — edit the `.hlsl`/`.vert`/`.frag` source and regenerate.
10. **Renderer/AA settings changes require an app restart** — there's no hot-swap path; see `GraphicsSettingsRequireRestart`.
11. **`RO_ENABLE_DEV_DEPLOY` defaults to copying the build output to a specific developer's local Windows path (`D:/Spel/OldRO`)** — irrelevant/auto-off on non-Windows, but turn it off explicitly on Windows if you don't have that path.
12. **You probably don't have the `Ref/` folder.** If a task requires checking original-client/server/protocol behavior against it, say so explicitly rather than guessing.
13. **Job/sprite name tables (`session/JobNameTable.generated.inc`) are generated** — find the generator before hand-editing.
14. Distinguish **in-world instances** (`CItem`, `CSkill` in `src/world/`) from **static metadata managers** (`CItemMgr`/`g_ttemmgr` in `src/item/`, `CSkillMgr`/`g_skillMgr` in `src/skill/`) — similarly named, different files, different responsibilities.

## 18. Where to look first for common tasks

| Task | Start here |
|---|---|
| Add/modify a network packet | `PACKET_VERSION_ALIGNMENT.md`, then `src/network/Packet.h`, `src/network/GronPacket.cpp`, `src/gamemode/GameModePacket.cpp` / `LoginMode.cpp` / `GameMode.cpp` |
| Add a new UI window/dialog | `src/ui/UIWindow.h`, `src/ui/UIFrameWnd.h`, `src/ui/UIWindowMgr.h/.cpp` for registration |
| Change player state exposed to UI | `src/session/Session.h/.cpp` (`CSession`, `g_session`) |
| Change actor/entity behavior | `src/world/GameActor.h`, `src/world/World.h/.cpp` |
| Change map loading / transition flow | `src/gamemode/GameMode.h/.cpp` (`MapLoadingStage`) |
| Add a graphics backend feature or fix a render bug | `src/render3d/RenderDevice.h` (interface), then the specific backend implementation; `src/render/Renderer.h/.cpp` for scene-level logic |
| Touch anti-aliasing / post-process | `ANTI_ALIASING.md` first, then `src/render3d/shaders/`, `RenderDevice.cpp` |
| Add a new asset/resource type | `src/res/Res.h` (`CRes` base), `CResMgr::RegisterType` call site in `InitClientSystems()` |
| Add a build source file | The relevant subsystem's own `src/<subsystem>/CMakeLists.txt` (no glob — must be explicit) |
| Understand build options | §4 above, or `CMakeLists.txt` top section |

## 19. Where the workspace's own conventions live

- `README.md` — project pitch, build instructions per platform.
- `TODO.md` — module-by-module completion status and the current packet-coverage audit; check before assuming a feature is unimplemented or a packet is missing.
- `X64_MILESTONE.md` — x64 vs Win32 scope differences.
- `PACKET_VERSION_ALIGNMENT.md` — the packet-version decision record; read before any packet work.
- `rAthena_OpenMidgard_patch.md` — server-side patches needed to make an rAthena test server speak this client's protocol correctly.
- `ANTI_ALIASING.md` — AA implementation notes and roadmap.
- `docs/qt6_qml_migration_spike.md` — the Qt UI migration design/status doc.
- `docs/qt_build_and_compliance.md` — Qt kit discovery, deployment, LGPL compliance mechanics.
- `.github/instructions/*.instructions.md` (mirrored in `.cursor/rules/*.mdc`) — the team's standing working-conventions: prefer complete implementations over stubs, check `Ref/` before guessing, always check the packet alignment doc for protocol work, never directly render media assets when inspecting them.
