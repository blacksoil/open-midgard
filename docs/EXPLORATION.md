# Exploration Notes — macOS "3D device initialization failed" Debugging

This is a working notes file from debugging the built `open-midgard` binary
on macOS failing at startup with:

```
3D device initialization failed. The game will exit.
```

It exists to avoid re-doing the same repo exploration in a future session.
It is **not** a finished root-cause report — see "Investigation Status"
at the bottom for what's confirmed vs. still open.

## Confirmed: Where The Error Message Comes From

The exact string is emitted from two places, one per platform entry point:

- `src/main/WinMain.cpp:807` — Windows path:
  `ErrorMsg("3D device initialization failed. The game will exit.");`
- `src/main/AppMain_stub.cpp:302` — non-Windows path (this is what runs on
  macOS):
  `std::fputs("3D device initialization failed. The game will exit.\n", stderr);`

Which file is compiled as the app entry point is decided in the top-level
`CMakeLists.txt`:

```cmake
set(RO_APP_MAIN_SOURCE src/main/WinMain.cpp)
if(WIN32)
    add_executable(open_midgard WIN32 ${RO_APP_MAIN_SOURCE} src/main/OpenMidgard.rc)
else()
    set(RO_APP_MAIN_SOURCE src/main/AppMain_stub.cpp)
    add_executable(open_midgard ${RO_APP_MAIN_SOURCE})
endif()
```

So on macOS, **`src/main/AppMain_stub.cpp` is the real entry point**, not
`WinMain.cpp`. Any fix for the macOS init failure has to be traced through
`AppMain_stub.cpp`, not the Windows main.

**Not yet read**: the actual body of `AppMain_stub.cpp` around line 302 —
i.e., what function call immediately precedes the `fputs`, and what that
function's failure conditions are. This is the single highest-value next
read.

## Relevant Architecture Context (from earlier full exploration / `docs/ARCHITECTURE.md`)

This is prior knowledge from documenting the codebase overall (see
`docs/ARCHITECTURE.md` "Rendering Backend Strategy" and `render3d/`
sections), relevant to this bug:

- The rendering backend abstraction lives in `src/render3d/RenderBackend.h`
  / `RenderBackend.cpp`. Key API surface (from the header, already read in
  full):
  ```cpp
  enum class RenderBackendType {
      LegacyDirect3D7 = 0,
      Direct3D11,
      Direct3D12,
      Vulkan,
  };
  struct RenderBackendBootstrapResult { RenderBackendType backend; int initHr; };
  const char* GetRenderBackendName(RenderBackendType backend);
  bool IsRenderBackendImplemented(RenderBackendType backend);
  bool IsRenderBackendSupported(RenderBackendType backend);
  RenderBackendType GetConfiguredRenderBackend();
  bool SetConfiguredRenderBackend(RenderBackendType backend);
  RenderBackendType GetRequestedRenderBackend();
  bool InitializeRenderBackend(RoNativeWindowHandle hwnd, RenderBackendBootstrapResult* outResult);
  ```
  `InitializeRenderBackend(...)` is almost certainly the function
  `AppMain_stub.cpp` calls right before the failure print — **not yet
  confirmed by reading `AppMain_stub.cpp` itself**, but consistent with the
  API shape and with the identical pattern implied in `WinMain.cpp`.

- `RoNativeWindowHandle` is `HWND` on Windows and `void*` on non-Windows
  (from `RenderBackend.h`), so the handle passed in on macOS is whatever
  native window pointer the Qt/AppMain_stub path produces — **not yet
  confirmed what that value actually is on macOS** (could plausibly be
  null/stub, which would be a strong candidate root cause if Vulkan surface
  creation needs a real native view).

- On non-Windows, native D3D11/D3D12 are force-disabled and Qt6 UI is
  force-enabled at the top of `CMakeLists.txt`:
  ```cmake
  if(NOT WIN32)
      set(RO_ENABLE_NATIVE_D3D11 OFF CACHE BOOL ... FORCE)
      set(RO_ENABLE_NATIVE_D3D12 OFF CACHE BOOL ... FORCE)
      set(RO_ENABLE_QT6_UI ON CACHE BOOL ... FORCE)
      set(RO_ENABLE_DEV_DEPLOY OFF CACHE BOOL ... FORCE)
  endif()
  ```
  So on macOS, **Vulkan is the only implementable/supported 3D backend**
  (`LegacyDirect3D7`/`Direct3D11`/`Direct3D12` should all report
  unsupported via `IsRenderBackendSupported`, assuming that function is
  platform-aware — not yet confirmed by reading `RenderBackend.cpp`'s
  implementation, only inferred from the header + CMake gating). If backend
  selection logic has any bug where it still *attempts* a
  Windows-only backend on macOS instead of routing to Vulkan, that alone
  would produce this exact failure — this is one hypothesis, not confirmed.

- `src/render3d/CMakeLists.txt` sets a platform flag relevant to Vulkan
  surface creation:
  ```cmake
  set(RO_VULKAN_USE_WIN32_SURFACE 0)   # non-Windows, from top-level CMakeLists.txt
  ```
  (compiled in as `RO_VULKAN_USE_WIN32_SURFACE=${RO_VULKAN_USE_WIN32_SURFACE}`
  via `add_compile_definitions` in the top-level `CMakeLists.txt`). This
  implies there's a code branch in the Vulkan device/surface creation path
  keyed on this macro — **the non-Win32 branch of that code has not been
  read yet**, so it's unconfirmed whether real macOS/Metal surface creation
  (`VK_EXT_metal_surface` or MoltenVK equivalent) is actually implemented
  there, or whether it's a stub that always fails. Given `TODO.md`'s Vulkan
  milestones (0-8) mark Vulkan bring-up done, but that roadmap was written
  with **Windows Vulkan** validation in mind (the milestone text doesn't
  distinguish macOS/MoltenVK specifically) — macOS-specific surface
  creation could plausibly be a gap even though "Vulkan" as a backend is
  marked complete for Windows.

- `RO_HAS_VULKAN` is only set to 1 conditionally in
  `src/render3d/CMakeLists.txt`:
  - Windows: only if `VULKAN_SDK` env var is set **and**
    `$VULKAN_SDK/Include/vulkan/vulkan.h` exists.
  - Linux: via `find_package(Vulkan REQUIRED)`.
  - Darwin (macOS): via `find_package(Vulkan REQUIRED COMPONENTS MoltenVK)`.
  If this `find_package` step silently found stale/partial Vulkan config at
  build time (e.g., headers but no working loader, or a `MoltenVK`
  component resolved to something CMake accepted but that isn't actually
  functional at runtime), `RO_HAS_VULKAN` could be `1` at compile time while
  the actual runtime environment still can't create a working device —
  worth checking `cmake` configure log output for the Vulkan detection
  message (`message(STATUS "Vulkan SDK detected ...")` / package status)
  from the build that was already done, to confirm what was actually found.

## File/Symbol Map For Continuing This Investigation

These are the exact next files to read, in priority order, to actually
pin down root cause (this list is the output of scoping the problem, not
of reading these files yet):

1. **`src/main/AppMain_stub.cpp`**, especially the ~50-100 lines before
   line 302 — confirms what's actually called, with what arguments, and
   how the return value maps to the failure branch.
2. **`src/render3d/RenderBackend.cpp`** — the implementation behind the
   header already read; need to see `InitializeRenderBackend`'s body,
   backend selection/fallback order, and how `IsRenderBackendSupported`
   is actually implemented per platform.
3. **`src/render3d/RenderDevice.cpp`** (and check whether
   `RenderDevice_stub.cpp` is compiled instead/also on non-Windows —
   `render3d/CMakeLists.txt` conditionally sources `Device.cpp` vs.
   `Device_nonwin.cpp`, but `RenderDevice.cpp` itself is unconditional in
   that file; `RenderDevice_stub.cpp` exists but its build-condition was
   not confirmed).
4. Whatever Vulkan-specific device source implements
   `VulkanRenderDevice`-style init (instance/physical device/logical
   device/surface/swapchain creation) — filename not yet confirmed; grep
   `render3d/` for `Vulkan` to locate it, likely inside `RenderDevice.cpp`
   or a dedicated file not yet enumerated in this session's file listing.
5. **`src/qtui/QtPlatformWindow.cpp`** — since Qt6 UI is forced on for
   macOS, this is the most likely place that's supposed to hand a native
   window/view handle to the Vulkan surface-creation call; worth checking
   whether that hookup exists for macOS or only for Windows.
6. **`ANTI_ALIASING.md`** and **`TODO.md`**'s Vulkan milestone sections —
   already read once for the architecture doc; re-skim specifically for
   any macOS/MoltenVK caveats (none were noticed on the first pass, but
   that pass wasn't looking for this).

## Useful General Repo Facts (carried over from earlier full exploration)

These were already established while writing `docs/ARCHITECTURE.md` /
`docs/BUILDING.md` and remain valid reference points for any further
debugging in this area:

- Build system: CMake ≥3.20, C++17, one static library target per
  `src/<subsystem>` directory, linked into a single `open_midgard`
  executable (see `docs/ARCHITECTURE.md` "Module Dependency Graph").
- `ro_render3d` depends on `ro_render`, `ro_core`, and conditionally on
  `Qt6::Gui` (non-Windows) and `Vulkan::Vulkan` (Linux/Darwin) /
  `VULKAN_SDK` headers (Windows) — see `src/render3d/CMakeLists.txt`.
- No CMake preset exists yet for macOS (`docs/BUILDING.md` documents a
  manual `cmake -S . -B build-macos -G Ninja ...` flow using
  `-DCMAKE_PREFIX_PATH="$(brew --prefix qt)"`).
- On this machine specifically, before the successful build that produced
  the failing binary, none of `cmake`, `ninja`, Qt, `vulkan-headers`,
  `vulkan-loader`, or `molten-vk` were installed — they were expected to
  be installed via Homebrew per `docs/BUILDING.md`. (Whether they were
  actually installed before this build succeeded, and which exact
  versions, was not re-verified in this session — worth confirming with
  `brew list --versions qt vulkan-headers vulkan-loader molten-vk` if
  version skew becomes a suspect.)
- Packet/networking, UI windows, world/actor rendering, etc. are unrelated
  to this bug and are documented separately in `docs/ARCHITECTURE.md` —
  not relevant to re-explore for this issue.

## Investigation Status

- A background research agent was dispatched to trace the full call chain
  (`AppMain_stub.cpp` → `RenderBackend`/`InitializeRenderBackend` →
  Vulkan device/surface creation → `QtPlatformWindow`) and rank concrete
  root causes. **It was stopped before returning results** (partway
  through, it had reached the point of setting breakpoints on
  `VulkanRenderDevice::Initialize`, `CreateInstance`, `CreateSurface`,
  `PickPhysicalDevice`, `CreateLogicalDevice`, and `RefreshRenderSize` —
  implying those are the real function/method names in the Vulkan device
  init path, which is itself a useful confirmed fact even though the trace
  output wasn't captured).
- **Nothing in this file about the Vulkan/surface code itself has been
  confirmed by direct reading in this session** — it is scoped and
  hypothesized from the header/CMake evidence above, not verified. Treat
  the "File/Symbol Map" section as the todo list, not as findings.
- Two leading hypotheses to test first when investigation resumes:
  1. macOS/non-Win32 surface creation in the Vulkan device path is
     incomplete/stubbed (never wired to a Metal-backed `VkSurfaceKHR` via
     `VK_EXT_metal_surface` or MoltenVK's surface extension), so it fails
     immediately regardless of environment setup.
  2. Environment/runtime issue rather than a code gap — e.g. the MoltenVK
     ICD (`libMoltenVK.dylib` / Vulkan loader `icd.json`) isn't discoverable
     at runtime (missing `VK_ICD_FILENAMES` or the Homebrew Vulkan loader
     not finding the Homebrew MoltenVK install), so a correctly-implemented
     Vulkan path still fails at `vkCreateInstance`/`vkCreatePhysicalDevice`
     time. Confirmed function names from the killed agent
     (`VulkanRenderDevice::Initialize`, `CreateInstance`, `CreateSurface`,
     `PickPhysicalDevice`, `CreateLogicalDevice`) suggest a debugger-based
     trace was the intended way to distinguish these — that's still the
     fastest way to get a definitive answer when resuming.
