# OpenMidgard Architecture Overview

This document orients a new contributor to the OpenMidgard codebase: what the
project is, how the source is organized, how the pieces link together at
build time, and where to look for the subsystems that matter most. It is a
map, not a spec — follow the file references into the code for details.

## What This Project Is

OpenMidgard is a from-source recreation of the **Ragnarok Online game
client**, intentionally targeting the classic **pre-Renewal 2008-era**
client (`packet_ver 23`, `2008-09-10aSakexe`). Much of the code originates
from decompiling/reverse-engineering the original `HighPriest.exe` client
binary and rewriting it as clean C++17 (see the comment banners like
`Clean C++17 rewrite of the HighPriest 2008 client` at the top of many
files, and comments referencing original memory offsets, e.g.
`// Memory layout from HighPriest.exe.h:30948` in `src/gamemode/Mode.h`).

Key implications of that origin for reading the code:
- Naming and structure sometimes mirror decompiler output (Hungarian-ish
  prefixes, `CXxx` class names, packed structs) rather than idiomatic
  modern C++.
- Packet formats and struct layouts are frequently pinned to exact byte
  sizes because they must match a specific game server's wire protocol
  (see `PACKET_VERSION_ALIGNMENT.md` and `rAthena_OpenMidgard_patch.md`).
- The project connects to a **separate, external game server** (e.g. an
  rAthena-based server) over TCP — this repository is client-only.

The client talks to a Ragnarok-style server stack (login server → character
server → map/zone server) and renders the classic 2D-sprites-over-3D-terrain
world using a pluggable modern rendering backend.

## Repository Layout

```
CMakeLists.txt          Top-level build: options, subdirectories, final exe
CMakePresets.json        Named build presets (VS2022, MinGW+Qt, Linux+Vulkan)
cmake/                   CMake helper modules (Qt setup, deploy helpers)
src/                     All client source, one subdirectory per subsystem
third_party/             Third-party license/notice bundles (currently Qt)
docs/                    Focused design/implementation notes (this file included)
.cursor/rules/           Editing/agent conventions for this repo (see below)
TODO.md                  Per-module implementation status / roadmap
PACKET_VERSION_ALIGNMENT.md   The packet-version decision record
rAthena_OpenMidgard_patch.md  Server-side patch needed to match this client
X64_MILESTONE.md         Notes on the x64 build milestone and legacy-DLL policy
ANTI_ALIASING.md         Design notes for the FXAA/SMAA post-process pipeline
```

## Build System

- CMake ≥ 3.20, C++17, single top-level `CMakeLists.txt` that
  `add_subdirectory()`s each module under `src/` and links them all into one
  executable target, `open_midgard` (output binary `open-midgard[.exe]`).
- Each subsystem is its own static library target (`ro_core`, `ro_network`,
  `ro_render3d`, …), declared in a per-directory `CMakeLists.txt`. This keeps
  include/link dependencies explicit — see "Module Dependency Graph" below.
- Platform behavior is switched largely through CMake options defined at the
  top of `CMakeLists.txt`:
  - `RO_ENABLE_NATIVE_D3D11` / `RO_ENABLE_NATIVE_D3D12` — Windows/MSVC only.
  - `RO_ENABLE_QT6_UI` (alias `RO_ENABLE_QT6_UI_SPIKE`) — forced ON for all
    non-Windows targets; optional on Windows.
  - `RO_ENABLE_MILES_AUDIO`, `RO_ENABLE_BINK_VIDEO`, `RO_ENABLE_GRANNY`,
    `RO_ENABLE_IJL` — legacy 32-bit-only middleware, auto-disabled on x64
    builds (see `X64_MILESTONE.md`).
  - `RO_ENABLE_DEV_DEPLOY` — copies the built exe to a hardcoded local test
    folder (`D:/Spel/OldRO`) for the maintainer's dev loop; safe to disable.
- Named presets in `CMakePresets.json` capture known-good combinations:
  `vs2022-win32`, `vs2022-x64` (native D3D7/11/12), `mingw-qt-x64[-debug]`
  (MinGW + Qt6, Windows), and `linux-qt-vulkan` (Qt6 + Vulkan, Linux).
- `src/lua` and `src/core` use `FetchContent` to pull Lua 5.1.5 and zlib at
  configure time — these are the only external source dependencies fetched
  automatically; Qt6 and the Vulkan SDK must be present on the system.
- Vulkan is auto-detected via `VULKAN_SDK` env var (Windows) or
  `find_package(Vulkan)` (Linux/macOS), gated behind `RO_HAS_VULKAN`.

## Module Dependency Graph

Static library targets and their `target_link_libraries` relationships
(from each module's `CMakeLists.txt`); arrows read "depends on":

```
ro_main ──> ro_gamemode, ro_session, ro_network, ro_core, ro_cipher

ro_gamemode ──> ro_core, ro_cipher, ro_network, ro_world, ro_audio, ro_ui
                (+ Qt6::Core/Gui when RO_ENABLE_QT6_UI)

ro_qtui ──> Qt6::Core/Gui/Qml/Quick, ro_core, ro_gamemode, ro_session, ro_ui, ro_world

ro_world ──> ro_core, ro_res, ro_ui
ro_ui    ──> ro_core, ro_item, ro_session, ro_skill (+ Qt6::Core/Gui optionally)
ro_session ──> ro_core, ro_network, ro_skill
ro_render  ──> ro_core, ro_res, ro_render3d (+ Qt6 optionally)
ro_render3d ──> ro_render, ro_core (+ Vulkan / Qt6::Gui conditionally)
ro_res     ──> ro_core (+ Qt6::Gui when Qt UI enabled)
ro_network ──> ro_core
ro_audio   ──> ro_core
ro_input   ──> ro_core
ro_cipher  ──> ro_core
ro_security──> ro_cipher, ro_core
ro_skill   ──> ro_core
ro_item    ──> ro_core
ro_pathfinder ──> ro_core
ro_lua     ──> ro_core, lua51 (vendored via FetchContent)
ro_core    ──> zlibstatic (vendored), winmm (Windows)
```

`ro_core` is the common foundation every other module ultimately depends on.
There's a mild circular-looking dependency between rendering and world/UI
(`ro_render` → `ro_render3d`; `ro_world`/`ro_ui` sit above rendering in the
executable's link list but below `ro_gamemode`), but there are no `#include`
cycles across library boundaries — dependencies flow one direction via
`target_link_libraries(... PUBLIC ...)`.

## Subsystems (`src/`)

### `core/` — Foundation layer
Shared utilities every other module depends on:
- `Types.h` (at `src/Types.h`, one level up) — the fixed-width type aliases
  (`u8/u16/.../s64`), common math structs (`vector2d/3d`, `matrix`), and
  wire-format structs like `CHARACTER_INFO`/`SERVER_ADDR` used across
  networking and UI. Force-included via `/FI` on MSVC so `windows.h`/locale
  ordering is consistent project-wide (see comment in `src/core/CMakeLists.txt`).
- `GPak.{h,cpp}` — reader for Gravity's `.grf` archive format (the game's
  asset pak), including DES-based obfuscation (`OpenPak01`) and the newer
  zlib-compressed index variant (`OpenPak02`, GRF `0x200`).
- `File.{h,cpp}` — file/memory-file abstractions used to feed `CGPak` and
  resource loaders.
- `Hash.{h,cpp}` — filename hashing used as the GRF index key.
- `Timer.{h,cpp}`, `Globals.{h,cpp}`, `SettingsIni.{h,cpp}`, `Xml.{h,cpp}`,
  `Locale.cpp` — timing, global state, INI-based settings, XML parsing
  (used for session/client-info XML), locale handling.
- `DllMgr.{h,cpp}` / `DllProtos.h` — dynamic loading of legacy middleware
  DLLs (Miles audio, Bink video, Granny, IJL) with graceful absence on x64.
- Pulls in `zlib` via `FetchContent`.

### `cipher/` — Cryptography
- `CDec.{h,cpp}` — DES decryption (used by the GRF reader and login flow).
- `Md5.{h,cpp}` — MD5 hashing.

### `network/` — Transport and packet plumbing
- `Connection.{h,cpp}` — `CConnection` (raw cross-platform TCP socket wrapper
  over WinSock2/BSD sockets) and `CRagConnection` (adds the
  Ragnarok-specific packet framing: `SendPacket`/`RecvPacket` with a 2-byte
  packet-type header).
- `PacketQueue.{h,cpp}` — send/recv byte-queue buffering used by `CConnection`.
- `GronPacket.{h,cpp}` — the receive-side packet size/dispatch table (what
  the client expects the server to send and how large each packet is).
- `MapSendProfile.{h,cpp}` — runtime-selectable packet **profiles** so the
  client's outgoing packet shapes can be switched between packet-version
  eras (see `Packets.MapSendProfile` / `Packets.MapGameplaySendProfile` in
  `PACKET_VERSION_ALIGNMENT.md`) without hardcoding one opcode set.
- `Packet.h` — packet id/opcode and struct definitions.

This is the subsystem most tightly coupled to the **exact server** this
client talks to — see the "Networking & Packet Versioning" section below.

### `session/` — Client-side game/account state
- `Session.{h,cpp}` (`CSession`, global `g_session`) — the single large
  in-memory model of "what the logged-in player currently knows": account
  info, character appearance/stats, inventory/storage/equipment, party and
  friend lists, NPC shop state, shortcut bar slots, buffs/status icons, exp
  tracking, and a large set of name/sprite-lookup helpers (job/head/weapon
  name resolution used to build sprite/asset filenames).
- `SkillActionInfo{,2}.{h,cpp}` — per-skill action/animation metadata.

`CSession` is effectively the client's client-side "world model" for the
local player and is read/written from the UI, gamemode, and world layers.

### `gamemode/` — Top-level game state machine
- `Mode.{h,cpp}` — `CMode` abstract base class for major client states, and
  `CModeMgr` (global `g_modeMgr`), which owns the current mode instance and
  switches between them (`CModeMgr::Switch`). This is the outermost state
  machine: e.g. login flow vs. in-game flow.
- `LoginMode.{h,cpp}` — login server → character server → character
  select/creation flow (`CLoginMode`).
- `GameMode.{h,cpp}` — the main in-game loop once a character has entered
  the world (`CGameMode`); by far the largest single file in the repo
  (~10k lines) — owns per-frame update/render orchestration, gameplay
  packet sending, and overlay composition (`DrawGameplayOverlayToHdc`,
  `QueueModernOverlayQuad`, etc., referenced in
  `docs/qt6_qml_migration_spike.md`).
- `GameModePacket.{h,cpp}` — map-server receive-packet routing/handlers
  invoked from within game mode (actor lifecycle, chat, party, etc.).
- `View.{h,cpp}` — camera/viewport.
- `CursorRenderer.{h,cpp}` — mouse cursor rendering, including
  action-context cursor icons (`CursorAction` enum in `Mode.h`).

### `world/` — Game world, actors, and effects
- `World.{h,cpp}` — `C3dAttr`/ground-attribute loading (walkability/height
  maps), and a quadtree-like `SceneGraphNode` spatial index used to cull and
  query actors by position for rendering and interaction.
- `GameActor.{h,cpp}` / `3dActor.{h,cpp}` — game actor (player/NPC/monster)
  representation and its 3D rendering; player (`Pc`) logic is currently
  folded into this rather than split into separate `Player`/`Pc` classes
  (per `TODO.md`).
- `RagEffect.{h,cpp}`, `MsgEffect.{h,cpp}` — visual/skill effects and
  floating combat-text/damage-number effects.
- `Granny.{h,cpp}` — stub integration point for the Granny 3D animation
  middleware (disabled on x64; see `X64_MILESTONE.md`).

### `render/` and `render3d/` — Rendering
- `render/` — 2D drawing primitives and the higher-level renderer/texture
  manager: `Renderer.{h,cpp}`, `DC.{h,cpp}` (device-context-style drawing),
  `Prim.{h,cpp}` (primitives), `DrawUtil.{h,cpp}`.
- `render3d/` — the pluggable 3D backend layer:
  - `RenderBackend.{h,cpp}` — the backend enum
    (`LegacyDirect3D7 / Direct3D11 / Direct3D12 / Vulkan`) and the
    selection/bootstrap API (`InitializeRenderBackend`,
    `GetConfiguredRenderBackend`, `SetConfiguredRenderBackend`, support/
    implemented queries). This is the seam the option-window renderer
    picker and the relaunch-on-change flow (see `TODO.md` Milestones 0-8)
    are built around.
  - `Device.{h,cpp}` / `Device_nonwin.cpp` — legacy DirectDraw/Direct3D7
    device init (Windows) vs. a non-Windows stand-in.
  - `RenderDevice.{h,cpp}` / `RenderDevice_stub.cpp` — the routed
    render-device abstraction that D3D11/D3D12/Vulkan implementations sit
    behind.
  - `ModernRenderState.{h,cpp}` — shared fixed-function-state translation
    used across the modern backends (extracted from the D3D11 path per the
    "Backend Abstraction Hardening" milestone in `TODO.md`).
  - `GraphicsSettings.{h,cpp}` — persisted renderer/AA settings.
  - `VulkanShaders.generated.h`, `VulkanFxaaShaders.generated.h`,
    `VulkanSmaaShaders.generated.h` — generated SPIR-V/shader byte arrays
    for the Vulkan FXAA/SMAA post-process pipeline (see `ANTI_ALIASING.md`).
  - `shaders/` — shader sources these are generated from.

The FXAA/SMAA anti-aliasing pipeline runs as a post-process pass on the 3D
scene target only, explicitly kept separate from UI/cursor composition so
2D overlays stay pixel-sharp (`ANTI_ALIASING.md`, `TODO.md` Milestone 11).

### `res/` — Asset/resource loading
Typed resource loaders sitting on top of `core/GPak`:
`Bitmap`, `Sprite`, `Texture`, `PaletteRes`, `ImfRes` (character sprite
composition), `GndRes` (ground/terrain), `ModelRes` (3D models),
`WorldRes` (`.rsw` world files), `EzEffectRes` (effect definitions),
`Ijl` (legacy Intel JPEG loader, optional). `Res.{h,cpp}` provides the
common resource base (`CRes`, referenced as the base of `C3dAttr` in
`world/World.h`).

### `ui/` — 2D UI window system
`UIWindow.{h,cpp}` is the base class; `UIWindowMgr.{h,cpp}` owns
composition/z-order and top-level draw dispatch. Everything else is one
`UIXxxWnd` per screen/panel: login, server/char select, char creation,
inventory/equipment/storage, NPC shop/dialog/menu, party, skills, minimap,
shortcut bar, options, loading/wait screens, etc. Currently drawn via
classic GDI-style immediate drawing (`OnDraw`), which is exactly what the Qt
migration spike (below) is working to replace incrementally.

### `qtui/` — Qt 6 / QML UI bridge (optional, `RO_ENABLE_QT6_UI`)
An additive, currently CPU-image-bridged integration that lets a Qt Quick
scene coexist with the existing game loop without giving Qt ownership of the
main loop or window message pump:
- `QtUiSpikeBridge.{h,cpp}` — owns the Qt Quick scene/event loop
  coexistence.
- `QtUiStateAdapter.{h,cpp}` — pushes per-frame gameplay data (screen
  anchors for labels, chat preview, login status, routed input) into QML
  models.
- `QtPlatformWindow.{h,cpp}` — Qt-side platform window integration.
- Non-Windows builds force this module on (`RO_ENABLE_QT6_UI` forced ON in
  the top-level `CMakeLists.txt` when `NOT WIN32`), making Qt the only UI
  path outside Windows.
- See `docs/qt6_qml_migration_spike.md` for the full architecture decision
  (target: Qt Quick renders offscreen into a D3D11-owned texture that's
  composited as the existing "modern overlay" quad — GDI ownership is being
  migrated screen-by-screen, not swapped wholesale) and
  `docs/qt_build_and_compliance.md` for Qt kit detection, `windeployqt`
  packaging, and LGPL compliance scaffolding (`third_party/qt`).

### `skill/`, `item/` — Game data
`Skill.{h,cpp}` and `Item.{h,cpp}` load skill/item metadata and
descriptions (data-driven from game data files, not hardcoded), consumed by
`session/`, `ui/`, and `world/`.

### `pathfinder/` — Movement
`PathFinder.{h,cpp}` — grid-based pathfinding over the ground/attribute map
produced by `world/World.cpp`'s `C3dAttr`.

### `lua/` — Scripting VM
Vendors **Lua 5.1.5** source directly via `FetchContent` (built as the
`lua51` target) rather than porting the original client's Lua 5.0 VM
(`TODO.md` notes this was Option B: "link against an external Lua library"
instead of hand-porting ~26 VM source files). `LuaBridge.{h,cpp}` is the
client-side integration point; `lua51_lundump_compat.c` is a compatibility
shim for bytecode loading.

### `security/` — Anti-tamper / packet security stub
`Security.{h,cpp}` — currently a stub for the original client's packet
signing/anti-cheat layer (see `TODO.md` Phase 11); depends on `ro_cipher`.

### `input/` — Raw input
`Input.{h,cpp}` — keyboard/mouse input handling (currently a thin/stub
layer per `TODO.md`).

### `audio/` — Sound and video playback
`Audio.{h,cpp}` with a small backend interface (`AudioBackend.h`) and two
implementations: `MilesAudioBackend.cpp` (legacy Miles Sound System,
32-bit-only DLL) and `MiniaudioBackend.cpp` (modern, cross-platform,
enabled where Miles is not, e.g. x64 builds). `Video.{h,cpp}` handles Bink
video playback (also legacy/32-bit-only).

### `main/` — Entry point
- `WinMain.{h,cpp}` — Windows entry point: window creation, registry/INI
  bootstrap, and the main message/game loop that drives `CModeMgr::Run`.
- `AppMain_stub.cpp` — non-Windows entry point stand-in (the project is
  Windows-first; non-Windows platforms get a reduced/stub main).
- `OpenMidgard.rc` — Windows resource file (icon, version info).

### `platform/`
`WindowsCompat.h` (and `platform/win32/`) — a compatibility shim so
Windows-specific types/macros used throughout the decompiler-origin code
(`HWND`, `HINSTANCE`, etc.) resolve to something sane on non-Windows
builds; included **before** the Windows platform dir on non-Windows targets
via `include_directories(BEFORE ...)` in the top-level `CMakeLists.txt`.

## Runtime Flow (High Level)

1. `main/WinMain.cpp` initializes the window, settings, and render backend
   (`render3d/RenderBackend.cpp` picks D3D7/D3D11/D3D12/Vulkan based on
   persisted settings, an env var override, or default fallback), then
   hands control to `CModeMgr` (`gamemode/Mode.cpp`).
2. `CModeMgr` starts in `CLoginMode` (`gamemode/LoginMode.cpp`): connects
   via `network/Connection.cpp` to the login/char servers, authenticates,
   lists/creates characters, and on character selection triggers
   `CModeMgr::Switch` into `CGameMode`.
3. `CGameMode` (`gamemode/GameMode.cpp`) drives the per-frame loop: pump
   network packets (`network/GronPacket.cpp` for receive parsing,
   `gamemode/GameModePacket.cpp` for handling), update world/actor state
   (`world/World.cpp`, `world/GameActor.cpp`), update `session::CSession`,
   render the 3D scene (`render3d/` backend) plus 2D UI
   (`ui/UIWindowMgr.cpp`) and effects (`world/RagEffect.cpp`,
   `world/MsgEffect.cpp`) on top, and route input (`input/Input.cpp`).
4. Assets (sprites, maps, models) are pulled on demand from `.grf` archives
   via `core/GPak.cpp`, decoded by the typed loaders in `res/`.
5. If Qt UI is enabled, `qtui/QtUiStateAdapter.cpp` mirrors relevant
   per-frame state into QML models each frame, composited as described
   above.

## Networking & Packet Versioning

This is one of the most idiosyncratic — and most important to understand —
parts of the codebase:

- The client targets one specific historical server packet family:
  `packet_ver 23` / `2008-09-10aSakexe`. This choice is deliberate (README
  and `PACKET_VERSION_ALIGNMENT.md`), not a placeholder — later "Renewal"-era
  opcodes/layouts are explicitly out of scope and, where nearby, actively
  guarded against.
- `PACKET_VERSION_ALIGNMENT.md` is the living decision record: it tracks,
  per outgoing packet, the opcode/size chosen and why, plus known gaps
  between receive-side parsing and what the target server actually emits.
- `rAthena_OpenMidgard_patch.md` documents the **server-side** source patch
  required to make an rAthena server correctly speak this packet family
  instead of drifting into a nearby early-Renewal opcode table — useful
  context if you're standing up a test server for this client.
- `network/MapSendProfile.{h,cpp}` exists specifically so the *outgoing*
  packet family is a runtime-selectable profile rather than a compile-time
  constant, since the client intentionally mixes an old "legacy0072" compact
  zone-handshake mode with newer opcode choices for individual packets.
- `TODO.md`'s "Packet Version Alignment Plan" section is the working
  backlog for this effort — a big practical resource when touching anything
  packet-related, including a "still missing" / "safe-ignore" packet id
  inventory.
- `.cursor/rules/packet-alignment-rules.mdc` — repo convention: always
  check `PACKET_VERSION_ALIGNMENT.md` before packet-related changes, and do
  not assume the original decompiled client (`Ref/`) used the same packet
  version as this rebuild.

## Rendering Backend Strategy

Four backends share one abstraction (`render3d/RenderBackend.h`):
`LegacyDirect3D7` (original, Windows/MSVC only), `Direct3D11`,
`Direct3D12`, and `Vulkan` (the only options on non-Windows, alongside the
forced Qt6 UI path). The option window can switch the *selected* backend
independently from the *active* one, with a relaunch flow
(`Mode`/`WinMain`-level "restart now" support) since backends can't be
hot-swapped mid-session. `TODO.md`'s "Renderer Backend Roadmap" is the
authoritative status/history of this effort (D3D12 and Vulkan have both
reached rendering parity with D3D11 as of the milestones marked done).
Anti-aliasing (FXAA/SMAA) is layered on top per backend as a post-process
pass on the 3D scene target only — see `ANTI_ALIASING.md`.

## Legacy Middleware and the x64 Milestone

The original client depended on several 32-bit-only proprietary DLLs:
Miles Sound System (audio), Bink (video), Granny (3D animation), Intel JPEG
Library. `core/DllMgr.cpp` loads these dynamically and tolerates their
absence. On x64 builds these are force-disabled (`X64_MILESTONE.md`), with
modern replacements substituted where needed (e.g. `miniaudio` for audio).
Win32 builds remain the way to validate original middleware-dependent
behavior; x64 is the forward-looking, middleware-light target.

## Project Conventions Worth Knowing

- `.cursor/rules/implementation-rules.mdc`: prefer complete, production-
  ready implementations over placeholders/stubs unless scaffolding is
  explicitly requested.
- `.cursor/rules/packet-alignment-rules.mdc`: packet-version discipline, see
  above.
- Other rule files (`build-mode-rules.mdc`, `media-file-handling-rules.mdc`,
  `reference-lookup-rules.mdc`) cover build-mode expectations, handling of
  binary/media assets, and how to use decompiled reference material — worth
  a skim (`.cursor/rules/`) before large changes.
- `TODO.md` is the single best "what's implemented vs. stubbed" reference —
  it tracks per-file status (`x` done, blank not started) across every
  planned module, not just aspirational milestones.

## Where To Start Reading

- New to the project? Read `README.md`, then this file, then skim
  `TODO.md`'s summary table to calibrate how complete each subsystem is.
- Touching networking/packets? Read `PACKET_VERSION_ALIGNMENT.md` and
  `network/{Connection,GronPacket,MapSendProfile}.*` together.
- Touching rendering? Read `render3d/RenderBackend.h` first, then the
  backend-specific `RenderDevice*` files and `ANTI_ALIASING.md`.
- Touching UI? Read `ui/UIWindowMgr.cpp` and, if Qt-relevant,
  `docs/qt6_qml_migration_spike.md` for the migration plan before adding new
  GDI-drawn UI.
- Touching gameplay flow? Start at `gamemode/Mode.cpp` (state machine),
  then `gamemode/LoginMode.cpp` or `gamemode/GameMode.cpp` depending on
  which state you're changing, and `session/Session.h` for the client-side
  data model those modes read/write.
