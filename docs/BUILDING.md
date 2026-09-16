# Building OpenMidgard

This document collects build instructions for all supported platforms. See
`docs/ARCHITECTURE.md` for how the build is organized (CMake options,
per-module static libraries, presets).

## Prerequisites (all platforms)

- CMake 3.20 or newer
- A C++17 compiler
- Git
- A legally obtained Ragnarok Online data/runtime setup for testing

At runtime, the client looks for `data.grf`, `data/clientinfo.xml`, and
`data/` near the executable or current working directory. You can also
point it at a custom runtime root with:

```
OPEN_MIDGARD_DATA_DIR=/path/to/runtime
```

## Windows

Windows currently has the most complete build flow, with two options.

### Option 1: Visual Studio 2022 (native D3D backends)

Available presets: `vs2022-win32`, `vs2022-x64`.

```powershell
cmake --preset vs2022-win32
cmake --build --preset build-release --config Release
```

Or for x64:

```powershell
cmake --preset vs2022-x64
cmake --build --preset build-release-x64 --config Release
```

Notes:
- Native Direct3D 11 and Direct3D 12 backends are only enabled on Windows
  MSVC builds.
- If you do not want the executable copied to `D:/Spel/OldRO`, configure
  with `-DRO_ENABLE_DEV_DEPLOY=OFF`.

### Option 2: MinGW + Qt 6

Preset: `mingw-qt-x64` (`mingw-qt-x64-debug` for a debug build).

```powershell
cmake --preset mingw-qt-x64
cmake --build --preset build-mingw-qt-x64
```

Debug build:

```powershell
cmake --preset mingw-qt-x64-debug
cmake --build --preset build-mingw-qt-x64-debug
```

Expected environment:
- Qt 6 desktop kit compatible with the chosen compiler.
- The included preset expects a Qt/MinGW layout similar to:
  - `C:/Qt/6.11.0/mingw_64`
  - `C:/Qt/Tools/mingw1310_64`
- If Qt is installed elsewhere, set `RO_QT_ROOT` to the matching kit root.

Notes:
- This is one of the main day-to-day workflows on Windows.
- Qt deployment (`windeployqt`) is handled automatically for Windows Qt
  builds — see `docs/qt_build_and_compliance.md`.

## Linux

Preset: `linux-qt-vulkan` (Qt6 + Vulkan; this is the only UI/render path on
Linux — native D3D backends are Windows-only).

```bash
cmake --preset linux-qt-vulkan
cmake --build --preset build-linux-qt-vulkan -j
```

Recommended dependencies:
- `cmake`
- `ninja`
- `g++` or `clang++` with C++17 support
- Qt 6 development packages, including QML/Quick
- Vulkan development packages / SDK

Practical note: on Ubuntu-like systems, the Qt runtime may also need the
`QtQml.WorkerScript` module package installed separately for the QML UI to
start correctly.

## macOS

macOS does not have a checked-in preset, but the project has a non-Windows
CMake path (same forced Qt6 + Vulkan/MoltenVK path as Linux) that can be
configured manually. This path is less exercised than Windows/Linux, so
expect some rough edges.

### 1. Install prerequisites (Homebrew)

```bash
brew install cmake ninja qt vulkan-headers vulkan-loader molten-vk
```

- `qt` provides the Qt 6 desktop kit (Core/Gui/Qml/Quick).
- `vulkan-headers` + `vulkan-loader` + `molten-vk` provide the Vulkan API
  on top of Metal, which `src/render3d/CMakeLists.txt` requires via
  `find_package(Vulkan REQUIRED COMPONENTS MoltenVK)` on Darwin.
- Xcode Command Line Tools (`xcode-select --install`) must already be
  present for Apple Clang.

### 2. Configure

```bash
cmake -S . -B build-macos \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DRO_ENABLE_DEV_DEPLOY=OFF \
  -DCMAKE_PREFIX_PATH="$(brew --prefix qt)"
```

`RO_ENABLE_DEV_DEPLOY=OFF` disables copying the built binary to the
maintainer's local Windows test folder (`D:/Spel/OldRO`), which is
meaningless on macOS. On non-Windows platforms, `RO_ENABLE_QT6_UI` is
forced on and the native Direct3D backends are forced off automatically by
the top-level `CMakeLists.txt` — no extra flags needed for that.

### 3. Build

```bash
cmake --build build-macos -j
```

### 4. Run

```bash
./build-macos/open-midgard
```

Point it at your game data via `OPEN_MIDGARD_DATA_DIR` if it's not next to
the binary.

## Build Options Reference

Set with `-D<OPTION>=ON|OFF` at configure time (see top of `CMakeLists.txt`
for the authoritative list and platform-forced overrides):

| Option | Default | Notes |
|---|---|---|
| `RO_ENABLE_MILES_AUDIO` | ON (x86), OFF (x64) | Legacy Miles Sound System, 32-bit only |
| `RO_ENABLE_BINK_VIDEO` | ON (x86), OFF (x64) | Legacy Bink video, 32-bit only |
| `RO_ENABLE_GRANNY` | ON (x86), OFF (x64) | Legacy Granny animation, 32-bit only |
| `RO_ENABLE_IJL` | OFF | Legacy Intel JPEG Library |
| `RO_ENABLE_DEV_DEPLOY` | ON (OFF on non-Windows) | Copies build output to `D:/Spel/OldRO` |
| `RO_ENABLE_QT6_UI` | OFF (forced ON on non-Windows) | Qt 6 / QML UI runtime |
| `RO_ENABLE_NATIVE_D3D11` / `RO_ENABLE_NATIVE_D3D12` | ON (forced OFF on non-Windows) | Native Direct3D backends, MSVC only |
| `RO_QT_ROOT` | (empty) | Path to a compiler-matching Qt 6 kit, e.g. `C:/Qt/6.11.0/msvc2022_64` |
| `RO_LOG_VULKAN_STATS` | OFF | Log Vulkan performance stats |

See `X64_MILESTONE.md` for why legacy middleware is disabled by default on
x64, and `docs/qt_build_and_compliance.md` for Qt-specific build/deploy/
licensing details.
