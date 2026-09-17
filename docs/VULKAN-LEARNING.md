# Vulkan Crash Course for OpenMidgard

This is a from-zero Vulkan primer for a seasoned software engineer who has never worked on a game or graphics engine before. It assumes you can read C++ comfortably and understand things like memory allocators, ring buffers, and state machines — but assumes **nothing** about GPUs, shaders, or real-time rendering.

The goal isn't to teach you Vulkan in the abstract. It's to teach you *this codebase's* Vulkan backend (`VulkanRenderDevice`, `src/render3d/RenderDevice.cpp`, roughly lines 4927–9092) well enough that you can read it, debug it, and extend it. Every concept below is immediately grounded in a real file/line/function in this repo. Read [`docs/DEVELOPER-ONBOARDING.md`](./DEVELOPER-ONBOARDING.md) first if you haven't — this document assumes you know the project's decompiler-origin context and the `IRenderDevice` abstraction it describes in §10.

---

## 1. What Vulkan actually is (and why this codebase needs it)

Skip the marketing description. Practically: **Vulkan is a C API for telling a GPU driver exactly what to do, with almost no implicit behavior.** Older APIs (Direct3D 7, OpenGL) had a *fixed-function pipeline* — you called functions like "enable alpha blending" or "set this texture," and the driver quietly built and managed GPU programs and state behind your back. Vulkan removes nearly all of that: **you** build the exact GPU program ("pipeline") ahead of time, **you** manage which memory belongs to which resource, and **you** insert every synchronization point by hand. Nothing happens implicitly.

This matters enormously for reading this codebase, because **OpenMidgard's client was originally written against Direct3D 7's fixed-function pipeline** (the game literally calls things like `SetRenderState(D3DRS_ALPHABLENDENABLE, TRUE)`), and that interface — `IRenderDevice` (`src/render3d/RenderDevice.h`) — was kept unchanged when D3D11, D3D12, and Vulkan backends were added later. So a huge fraction of "Vulkan code" in this repo isn't really about rendering — it's a **translation layer** converting decades-old fixed-function *state-setting calls* into modern *pre-built GPU pipeline objects*. That translation layer (§8 below) is the single most important thing to understand about this backend, more so than Vulkan itself.

---

## 2. Core vocabulary, each tied to this codebase

Read this section once, referring back to it as you read the rest of the document and the code.

| Term | Plain-English meaning | Where it lives in this repo |
|---|---|---|
| **Instance** (`VkInstance`) | The handle representing "this process is talking to the Vulkan API at all." Created once. | `VulkanRenderDevice::CreateInstance()`, `RenderDevice.cpp:7579` |
| **Physical device** (`VkPhysicalDevice`) | A handle to an actual GPU in the machine (there can be several — integrated + discrete). You don't own it, you just query and select one. | `PickPhysicalDevice()`, `:7707` |
| **Logical device** (`VkDevice`) | Your process's *own* connection to a chosen GPU — this is the handle almost every other Vulkan call takes. | `CreateLogicalDevice()`, `:7853` |
| **Queue** (`VkQueue`) | A GPU work submission channel. GPUs have separate queues for graphics, compute, and present (showing an image on screen); this backend uses one graphics queue and one present queue (often the same physical queue). | `m_graphicsQueue`/`m_presentQueue`, set up in `CreateLogicalDevice()` |
| **Swapchain** (`VkSwapchainKHR`) | The set of images (usually 2–3) that get cycled between "the GPU is drawing into it" and "the screen is displaying it" — this *is* double/triple buffering. | `CreateSwapChainResources()`, `:8027` |
| **Command buffer** (`VkCommandBuffer`) | A recorded list of GPU commands ("draw this," "copy that") that you build up on the CPU, then submit as a batch. Nothing executes until you submit it. | one per frame, recorded in `EnsureFrameStarted`/`BeginFrame`/draw calls/`Present` |
| **Command pool** (`VkCommandPool`) | The allocator that command buffers are carved out of. | `CreateCommandPool()`, `:7906` |
| **Shader module** (`VkShaderModule`) | A GPU program, pre-compiled to SPIR-V bytecode (Vulkan's shader IR — think "JVM bytecode, but for GPUs"). You author it as HLSL/GLSL text, a build step compiles it offline, and the app only ever loads the compiled bytes. | `CreateShaderModuleFromBytes()`, `:6286`; source at `src/render3d/shaders/*.hlsl` |
| **Pipeline** (`VkPipeline`) | The single most important Vulkan object: a **fully baked, immutable snapshot of the entire GPU draw configuration** — which shaders to run, vertex layout, blend mode, depth test, cull mode, etc. Unlike D3D7, you cannot "set blend mode" at draw time; you must have a whole pipeline object pre-built for every state combination you'll ever draw with. | `GetPipelineState()`, `:6320` — this repo's PSO cache, see §8 |
| **Pipeline layout** (`VkPipelineLayout`) | Describes *what kind of external data* a pipeline's shaders expect (descriptor set layouts + push-constant ranges) without saying what the data actually is yet. | `m_pipelineLayout` / `m_postPipelineLayout`, built in `CreatePipelineResources()`, `:5893` |
| **Descriptor set** (`VkDescriptorSet`) | The actual bound data for one draw call — "here are the specific textures/buffers to read from this time." Allocated from a **descriptor pool**, a fixed-capacity arena. | `GetOrCreateTextureDescriptorSet()`, `:7129` |
| **Push constants** | A tiny (tens of bytes), extremely cheap way to hand a shader some small per-draw values (screen size, a bitfield of feature flags, fog parameters) without the overhead of a full descriptor/buffer. | `ModernDrawConstants`, pushed via `vkCmdPushConstants` in every draw |
| **Render pass** (`VkRenderPass`) + **framebuffer** (`VkFramebuffer`) | A render pass describes *what kind of image* you're about to draw into and what should happen to it before/after (clear it? keep old contents? what layout should it end up in?). A framebuffer is the *specific image(s)* plugged into that description. | `m_renderPass` / `m_overlayRenderPass`, `CreateSwapChainResources()`, `:8118–8181` |
| **Image / image view** (`VkImage`/`VkImageView`) | A GPU-resident 2D/3D pixel buffer (a texture, a render target, a depth buffer — all the same underlying concept), plus a "view" describing how to interpret it (format, which mip levels, etc.) | textures: `CreateTextureHandle()`, `:6650`; depth buffer: `CreateDepthResources` |
| **Image layout** | Vulkan requires you to explicitly track and transition what an image is "for" right now (e.g. `COLOR_ATTACHMENT_OPTIMAL` while being drawn into, vs. `SHADER_READ_ONLY_OPTIMAL` while being sampled from, vs. `PRESENT_SRC_KHR` while on screen). Using an image in the wrong layout is a correctness bug the GPU won't necessarily catch. | transitions scattered through `PrepareOverlayPass()`, `:5479`, and `Present()` |
| **Fence** | A GPU→CPU signal: "the GPU is done with this batch of work," which your CPU code can wait on. | `m_inFlightFence`, waited on at the top of every `BeginFrame()` |
| **Semaphore** | A GPU→GPU signal: "this queue operation is done," used to order two pieces of *GPU* work relative to each other (the CPU never touches it directly). | `m_imageAvailableSemaphore` / `m_renderFinishedSemaphore`, created in `CreateSyncObjects()`, `:7930` |
| **Memory type / memory heap** | Vulkan exposes the GPU's actual memory pools directly — some fast but only GPU-readable (`DEVICE_LOCAL`), some slower but CPU-writable (`HOST_VISIBLE`/`HOST_COHERENT`). You must pick the right one per resource and allocate/bind memory yourself; there's no automatic "just make a buffer" call. | `FindMemoryType()`, `:8894` |

If you remember one sentence from this table: **Vulkan makes you build the GPU's "recipe" (pipeline) in advance and only hand it fresh "ingredients" (descriptor sets, push constants) at draw time** — almost everything below is elaborating on that one idea.

---

## 3. The bootstrap sequence (device creation)

This is one-time setup, run once at `VulkanRenderDevice::Initialize(HWND hwnd, ...)` (`:5034`), each step now logged on failure (see the commit that added `DbgLog` calls to each of these — you may have written that yourself):

```
LoadVulkanLoader()          :346   — dlopen/LoadLibrary the platform's Vulkan runtime, resolve the 2 bootstrap function pointers
CreateInstance()            :7579  — VkInstance: pick required surface extensions, call vkCreateInstance
CreateSurface()              :7659  — VkSurfaceKHR: bind the instance to the actual OS window (Win32 direct; Linux/macOS via Qt)
PickPhysicalDevice()          :7707  — enumerate GPUs, score them (discrete > integrated > virtual > CPU), pick the best
CreateLogicalDevice()         :7853  — VkDevice + graphics/present queues
CreateCommandPool()           :7906  — the allocator command buffers come from
CreateSyncObjects()           :7930  — 2 semaphores + 2 fences
CreatePipelineResources()      :5893  — descriptor layouts, samplers, shader modules, pipeline layouts, a 1×1 default texture
CreateSwapChainResources(...)  :8027  — the swapchain itself, depth buffer, render passes, AA offscreen targets/pipelines
```

Full walkthrough of what each step means and why it can fail is in the answer given earlier in this conversation about the Vulkan-logging commit — re-read that if you want the failure-mode detail for each step. The important structural fact for this section: **everything above only runs once.** Resize does *not* re-run this whole chain — see §7.

---

## 4. The `IRenderDevice` contract this class fulfills

Read `src/render3d/RenderDevice.h` in full (it's short, 81 lines) — this is the *entire* interface every backend (legacy D3D7, D3D11, D3D12, Vulkan) implements identically. Group the 24 pure-virtual methods mentally like this:

- **Lifecycle**: `Initialize`, `Shutdown`, `RefreshRenderSize`, `GetBackendType`
- **Frame boundaries**: `ClearColor`, `ClearDepth`, `BeginScene`, `PrepareOverlayPass`, `EndScene`, `Present`, `UpdateBackBufferFromMemory`
- **Fixed-function state** (the D3D7 legacy): `SetTransform`, `SetRenderState`, `SetTextureStageState`, `BindTexture`
- **Immediate-mode draw**: `DrawPrimitive`, `DrawIndexedPrimitive` — note these take **raw CPU vertex arrays as parameters**, exactly like D3D's old `DrawPrimitiveUP` ("user pointer") calls. There is no "create a vertex buffer once, draw it many times" object model here — every draw call hands over fresh vertex bytes. §6 explains how the Vulkan backend makes that cheap.
- **Textures**: `CreateTextureResource`, `UpdateTextureResource`, `ReleaseTextureResource`, `AdjustTextureSize`
- **Qt interop**: `GetQtUiRenderTargetInfo`/`GetQtUiTextureTargetInfo` — hand the Qt overlay compositor (see onboarding doc §11) a native handle into this backend's render target

Nowhere in this interface is there a concept of "pipeline" or "descriptor set" — those are pure Vulkan-backend implementation details, invisible to the calling game code. That's the whole point of the abstraction: the 2008-era game logic still thinks it's talking to a D3D7 fixed-function device.

---

## 5. The one shader that stands in for the fixed-function pipeline

Old D3D7 code could combine textures and colors in dozens of ways via "texture stage states" (`SetTextureStageState`). Vulkan has no such mechanism — every combination would need its own tiny shader program. Instead of generating dozens of shader permutations, this codebase uses **one flexible fragment shader with a runtime feature-flags bitfield**, passed in via push constants. Read `src/render3d/shaders/vulkan_world.hlsl` in full — it's under 110 lines and is the shader every piece of world geometry (sprites, ground, models) runs through:

```hlsl
struct DrawConstants {
    float screenWidth; float screenHeight;
    float alphaRef; float fogStart; float fogEnd;
    float fogColorR; float fogColorG; float fogColorB;
    uint flags;              // <- the bitfield
    float3 padding;
};
[[vk::push_constant]] DrawConstants g_drawConstants;
```

The fragment shader (`PSMain`) then does plain `if ((g_drawConstants.flags & 1u) != 0u) { ... sample texture0 ... }`, `& 64u` for "modulate by lightmap alpha," `& 128u` for "apply linear fog," `& 4u`/`& 8u` for alpha-test/discard. `BuildModernDrawFlags` (`src/render3d/ModernRenderState.cpp:90`) is the function that inspects the game's currently-set texture-stage-state combination and produces this bitfield.

**Why this matters for you**: if you're asked to add support for a new blending/texturing behavior the game uses, the fix is almost never "write a new shader" — it's **adding a new bit to this flags struct**, handling it in both `BuildModernDrawFlags` (C++) and `PSMain` (HLSL), and re-running the shader-generation step (see §9) so the `.generated.h` byte array picks up the change.

The **vertex shader** side (`VSMainTL`/`VSMainLM`) has its own quirk worth understanding: the incoming vertex positions are already **screen-space, pre-transformed by the CPU** (the D3D7-era `D3DFVF_XYZRHW` convention — "transformed and lit," i.e. "don't do a 3D transform, I already did it"). The vertex shader's whole job is just converting those screen pixel coordinates into Vulkan's normalized device coordinates (the `-1..1` cube every GPU expects):

```hlsl
float ndcX = (input.pos.x / screenWidth) * 2.0f - 1.0f;
float ndcY = 1.0f - (input.pos.y / screenHeight) * 2.0f;   // Y flip: screen-space Y grows down, NDC Y grows up
```

There's no camera matrix, no model/view/projection here — the game engine (`src/render/Renderer.h/.cpp`, per the onboarding doc) does all real 3D math on the CPU, the same way the original 2008 client did with D3D7's fixed-function transform pipeline.

---

## 6. Making "draw with raw CPU vertex data every call" cheap: the upload-page allocator

Real GPU APIs want you to build a vertex buffer once and reuse it. This game calls `DrawPrimitive`/`DrawIndexedPrimitive` with a fresh CPU array every single call (thousands of times per frame). Re-creating a `VkBuffer` per draw would be disastrously slow, so this codebase implements a **bump allocator over a small number of large, persistently-mapped host-visible buffers** ("upload pages," `UploadPage` struct at `:7297`):

- Each page (4MB for vertices, 512KB for indices by default) is a plain host-visible/host-coherent `VkBuffer`, permanently `vkMapMemory`'d — no map/unmap overhead per draw.
- `AllocateVertexBufferSlice`/`AllocateIndexBufferSlice` (`:7400–7474`) just `memcpy` the caller's data into the next free, alignment-rounded byte range of the current page, growing a new page only if the current one is full.
- The resulting byte offset is handed straight to `vkCmdBindVertexBuffers`/`vkCmdBindIndexBuffer` for that one draw call.
- Every page's write cursor **resets to zero at the start of every frame** (`ResetUploadPageCursors`, called from `BeginFrame()`) — this only works because the fence-wait at the top of `BeginFrame` (§7) guarantees the *previous* frame's GPU reads of that memory have already finished before you start overwriting it.

This is a very common real-time-rendering pattern (often called a "ring buffer" or "linear/frame allocator" for transient GPU data) — recognize it, because you'll see the same shape reused for the **descriptor set cache** (§8) and the **deferred-resource-destruction** buckets (§7) later in this file.

---

## 7. The per-frame loop, end to end

This is the sequence that runs every single frame. Trace it in this order in the source:

1. **`BeginScene()` → `EnsureFrameStarted()`** (`:8695`) — the game engine calls this once per frame before issuing any draws.
2. **`BeginFrame()`** (`:8818`), only on the *first* call this frame:
   - `vkWaitForFences(m_inFlightFence)` — **stall the CPU until the previous frame's GPU work is fully done.** This backend deliberately uses only **one frame in flight** (no double/triple-buffered CPU/GPU overlap) — simpler to reason about, at some performance cost. If you're ever asked "why can't we pipeline 2 frames ahead," this is the line that would need to change, along with duplicating every per-frame resource (upload pages, descriptor pool, command buffer) per frame-in-flight — a real, non-trivial project.
   - Drain deferred-destruction queues (`ReleasePendingTransferResources`) — now-safe to actually `vkDestroyImage`/`vkFreeMemory` textures that were released last frame (see §8's note on deferred texture destruction).
   - Reset the fence, the descriptor pool (`vkResetDescriptorPool`), and the upload-page cursors (§6).
   - `vkAcquireNextImageKHR` — ask the swapchain "which of your images can I draw into right now," blocking until one's free, and arranging for `m_imageAvailableSemaphore` to be signaled when it truly is.
   - `vkBeginCommandBuffer` on this frame's one command buffer.
   - `vkCmdBeginRenderPass` against either the offscreen scene target (if anti-aliasing is on) or directly against the swapchain image (if off).
3. **Draw calls** — `DrawPrimitive`/`DrawIndexedPrimitive` → `DrawTransformedPrimitive` (`:7476`): resolve/build the right `VkPipeline` for the current fixed-function state (§8), fetch/build the right descriptor set for the currently-bound textures (§8), allocate an upload-page slice for the vertex data (§6), then `vkCmdBindPipeline` → `vkCmdBindDescriptorSets` → `vkCmdPushConstants` → `vkCmdBindVertexBuffers` → `vkCmdDraw`/`vkCmdDrawIndexed`.
4. **`PrepareOverlayPass()`** (`:5479`) — the anti-aliasing resolve step, see §8. Ends the scene render pass, runs FXAA or SMAA, leaves a render pass open against the *actual swapchain image* for anything drawn after this point (the GDI-composited UI overlay, notably).
5. **`Present(bool vertSync)`** (`:5289`) — runs `PrepareOverlayPass()` if it hasn't already fired this frame, ends whatever render pass is still open, `vkEndCommandBuffer`, then `vkQueueSubmit` (wait on `m_imageAvailableSemaphore`, signal `m_renderFinishedSemaphore` + `m_inFlightFence` on completion), then `vkQueuePresentKHR` (wait on `m_renderFinishedSemaphore` before actually flipping to screen). If the driver reports `VK_ERROR_OUT_OF_DATE_KHR`/`VK_SUBOPTIMAL_KHR` (typically: the window was resized), this triggers `ResizeSwapChain()`.

**Note**: `EndScene()` is a no-op for this backend — don't be misled by the name symmetry with `BeginScene()`. All the real per-frame teardown work happens inside `Present()`.

There's a separate, render-pass-independent path — **`UpdateBackBufferFromMemory`** (`:5389`) — used to push a fully CPU-composited bitmap (the GDI-rendered native UI, per the onboarding doc's rendering section) straight onto the swapchain image via a staging buffer + `vkCmdCopyBufferToImage`, bypassing the shader/pipeline draw path entirely.

---

## 8. Two resource-management patterns worth internalizing

**Per-frame descriptor set cache.** `GetOrCreateTextureDescriptorSet(tex0, tex1)` (`:7129`) caches descriptor sets keyed by which textures are bound, but the entire cache and the descriptor pool it allocates from are wiped (`vkResetDescriptorPool`) at the start of *every* frame (`BeginFrame`). This sidesteps a whole class of "when can I safely free this descriptor set" bugs, at the cost of re-writing descriptor bindings for every unique texture combination every single frame — a deliberate simplicity-over-throughput tradeoff, reasonable given how few unique textures are bound per frame in a 2008-era game.

**Deferred GPU resource destruction.** `ReleaseTextureResource` (`:5622`) does **not** immediately call `vkDestroyImage`. It can't — the GPU might still be reading that texture from a command buffer that hasn't finished executing yet. Instead it queues the resource into `m_pendingReleaseTextures`, and the actual destruction happens inside `ReleasePendingTransferResources()`, called at the top of the *next* `BeginFrame()` — by which point the fence wait already proved the prior frame's GPU work is complete, so it's provably safe. This "defer until the next fence signal" pattern is the general Vulkan solution to "don't destroy something the GPU might still be using," and it recurs for buffers too (`m_pendingReleaseBuffers`/`m_pendingReleaseMemory`).

**The pipeline (PSO) cache and the fixed-function translation.** This is the answer to "how does `SetRenderState(D3DRS_ALPHABLENDENABLE, TRUE)` actually become a Vulkan draw":

1. `SetRenderState`/`SetTextureStageState` calls do **no GPU work at all** — they just write into a plain C++ struct, `ModernFixedFunctionState m_pipelineState` (`src/render3d/ModernRenderState.h`), via `ApplyModernRenderState`/`ApplyModernTextureStageState`.
2. At actual draw time, `GetPipelineState(isLightmap, topology)` (`:6320`) converts the *current* `m_pipelineState` into concrete Vulkan enums and uses it as a cache key against `m_pipelines` — a flat vector, linearly scanned (fine, because the number of distinct state combinations this game actually uses is small). A cache hit reuses an existing `VkPipeline`; a miss builds a brand-new one on the spot (`vkCreateGraphicsPipelines`) and caches it forever.
3. Texture-stage color/alpha *combining logic* (as opposed to blend/depth/cull state) is **not** baked into separate pipelines — it's collapsed into the push-constant flags bitfield from §5 instead, and branched on at runtime inside the one shared fragment shader. So: blend mode / depth test / cull mode → **real pipeline permutations**; texture combine modes → **runtime shader branches**. Recognize this as a deliberate hybrid, not an accident.

**A specific thing worth double-checking if you're touching resize or pipeline code**: `m_pipelines` does not appear to be cleared during `ResizeSwapChain()`/`CreateSwapChainResources()`, even though those functions *do* recreate `m_renderPass` (which every cached `VkPipeline` was built against). Cached pipelines built against a destroyed render pass being reused after a resize would be a real correctness bug in Vulkan (the validation layer should catch it loudly if so — see §10). This wasn't confirmed as an active bug, just flagged as something the research pass couldn't rule out — if you're ever debugging weird visual corruption specifically after a window resize, this is one of the first places to look.

---

## 9. Shaders: how source becomes code, and the anti-aliasing pipeline as a worked example

Shader source lives in `src/render3d/shaders/*.hlsl` (plus `.vert`/`.frag` GLSL variants for the fullscreen post-process passes). These are **not compiled as part of the normal C++ build** — they're compiled offline (HLSL/GLSL → SPIR-V, Vulkan's shader bytecode format) by a build-time tool into `src/render3d/VulkanShaders.generated.h`, `VulkanFxaaShaders.generated.h`, `VulkanSmaaShaders.generated.h` — plain headers containing nothing but `static const uint8_t kXxxSpirv[] = { 0x03, 0x02, 0x23, 0x07, ... }` byte arrays. **Edit the `.hlsl`/`.vert`/`.frag` source and regenerate; never hand-edit a `.generated.h` file.** `CreateShaderModuleFromBytes` (`:6286`) is the one function that turns these byte arrays into a real `VkShaderModule` at runtime (copying into a 4-byte-aligned buffer first, since SPIR-V requires 4-byte alignment and an incoming `const uint8_t*` isn't guaranteed to have it).

The anti-aliasing pass (see `ANTI_ALIASING.md` for the product-level picture) is the best worked example in this codebase of a **fullscreen post-process pass** — a pattern you'll see in essentially every real-time renderer, worth understanding cold:

- **No vertex buffer at all.** The vertex shader for every post pass (`VSMainPost`) generates a triangle *big enough to cover the whole screen* purely from the built-in `SV_VertexID`/`gl_VertexIndex` (0, 1, 2) — no vertices are bound, no vertex buffer exists:
  ```hlsl
  clipPos.x = (vertexId == 2u) ? 3.0f : -1.0f;
  clipPos.y = (vertexId == 1u) ? 3.0f : -1.0f;
  ```
  This produces one triangle whose corners lie *outside* the normal `-1..1` screen area — the GPU clips it down to exactly the visible rectangle for you. This "attributeless fullscreen triangle" trick is standard practice specifically because a triangle avoids the diagonal seam a screen-covering *quad* (2 triangles) would need.
- **FXAA** (`vulkan_post_fxaa.hlsl`/`.frag`) is one pass: sample the rendered scene at the current pixel and its 4 diagonal neighbors, compute perceptual brightness ("luma") for each, and if the local contrast exceeds a threshold (i.e., you're near an edge), blend along the estimated edge direction to soften it. One texture in, one texture out, one pipeline (`m_postPipeline`).
- **SMAA** (`vulkan_post_smaa.hlsl`) is a classic **3-pass chain**, each pass a separate pipeline, each writing to its own small offscreen image so the next pass can read it:
  1. `PSMainSMAAEdge` — detect which pixel edges are "real" edges worth anti-aliasing.
  2. `PSMainSMAABlendWeight` — reads the edge image, computes how much to blend across each detected edge.
  3. `PSMainSMAANeighborhood` — the only pass that writes to the *actual swapchain image*; reads both the original scene and the blend-weight image and produces the final anti-aliased pixel.
- Both FXAA and SMAA only ever touch an **offscreen scene color image** (`m_sceneImage`, separate from the swapchain), never the GDI-composited UI/cursor layer drawn after — this is why AA never affects text or the mouse cursor (again, see `ANTI_ALIASING.md`).
- Two `VkRenderPass` objects exist to express this shape: `m_renderPass` (used for the offscreen scene draw, and reused for SMAA's intermediate edge/blend-weight passes) and `m_overlayRenderPass` (used only for whichever pass's output must land in the actual presentable swapchain image — FXAA's one pass, or SMAA's final neighborhood-blend pass).

If you're ever asked to add a new post-process effect (bloom, color grading, a new AA technique), this FXAA/SMAA pair is the reference implementation to copy the shape of: one or more offscreen images, one `CreatePostPipeline` call per pass, one fullscreen-triangle draw per pass, wired into `PrepareOverlayPass()`.

---

## 10. Debugging: what to reach for when Vulkan misbehaves

- **`DbgLog(...)` calls** are scattered through the bootstrap chain (§3) and several draw/resize paths — these write to both the debugger output and a per-process log file (`debug_hp_<pid>.log`, per the onboarding doc's networking section — the same log file used for packet tracing). Start here for "why won't it even start" issues.
- **Validation layers** are Vulkan's equivalent of a sanitizer — they catch the large majority of "you used the API wrong" bugs (wrong image layout, unmatched pipeline/render-pass, resource used after being destroyed, missing synchronization) at the point of the mistake, with a specific, actionable message, rather than corrupting the frame silently or crashing the driver later. If you're debugging a Vulkan-specific issue and don't see validation output, check whether the Vulkan SDK's validation layer is actually enabled/available in your build/run configuration before assuming the code is correct.
- **`RO_LOG_VULKAN_STATS`** (a `CMakeLists.txt` option, off by default) enables extra performance-stats logging — check `g_vulkanPerfStats` usages in `RenderDevice.cpp` if you enable it.
- **`VK_ERROR_OUT_OF_DATE_KHR`/`VK_SUBOPTIMAL_KHR`** from `Present()` are not really errors — they're Vulkan's normal "the window changed size, please rebuild the swapchain" signal, handled by `ResizeSwapChain()` (§7). Don't treat these as bugs to fix; they're part of the intended resize flow.
- **The most common class of real Vulkan bug** you'll encounter working on this backend is an **image layout mismatch or missing synchronization** — e.g. sampling an image that's still in `COLOR_ATTACHMENT_OPTIMAL` layout instead of `SHADER_READ_ONLY_OPTIMAL`, or reading a buffer the GPU hasn't finished writing to. The validation layer will name the exact resource and the exact expected-vs-actual state when this happens — read that message carefully before guessing.
- **Cross-backend comparison is a real debugging tool here.** Because D3D11, D3D12, and Vulkan all sit behind the identical `IRenderDevice` interface and (per §5/§8) share the same `ModernFixedFunctionState`/push-constant-flags design, if something looks wrong only on Vulkan, diffing the equivalent D3D11/D3D12 code path in the same file is often the fastest way to spot what Vulkan is doing differently.

---

## 11. Where to make changes for common tasks

| Task | Start here |
|---|---|
| Add a new fixed-function render-state / texture-combine behavior | `src/render3d/ModernRenderState.h/.cpp` (`ApplyModernRenderState`, `BuildModernDrawFlags`) + `PSMain` in `vulkan_world.hlsl` (and the equivalent D3D11/D3D12 shader code) |
| Add a new blend mode / depth mode / cull mode | `ConvertBlendFactorVk`/`ConvertCullModeVk` + `GetPipelineState()`, `:6320` |
| Add a new post-process effect (bloom, color grading, another AA mode) | Model it on FXAA/SMAA per §9: new offscreen image(s), `CreatePostPipeline` call(s), wire into `PrepareOverlayPass()` |
| Debug "GPU init failed" | Bootstrap chain in §3, `DbgLog` output in `debug_hp_<pid>.log` |
| Debug "resize looks broken" | `ResizeSwapChain()`/`CreateSwapChainResources()`, and double-check the pipeline-cache staleness question flagged in §8 |
| Debug "texture looks wrong / corrupted" | Upload path: `UploadTextureRegion`, `:6904`; check image layout transitions and whether the upload happened via the immediate or batched path |
| Understand how a legacy `DrawPrimitive` call becomes GPU work | §6 (upload-page allocator) + §8 (pipeline cache) together |
| Change how many frames can be "in flight" (a real perf project) | `BeginFrame()`'s single-fence-wait design, §7 — requires duplicating per-frame resources |

---

## 12. Glossary (quick lookup)

- **SPIR-V** — Vulkan's portable shader bytecode format (analogous to JVM bytecode). Shaders are authored in HLSL/GLSL text and compiled to SPIR-V offline.
- **PSO** — "Pipeline State Object," industry shorthand for what Vulkan calls a `VkPipeline` — a fully baked GPU draw configuration.
- **Uber-shader** — one shader flexible enough (via runtime branches on a flags bitfield) to stand in for many separate specialized shaders. This codebase's `PSMain` is a textbook example.
- **Fullscreen triangle/quad pass** — a post-process technique: draw one triangle (or quad) covering the entire screen, with a fragment shader that samples a previously-rendered image and produces a new one. Used for FXAA/SMAA here.
- **Descriptor set** — the Vulkan mechanism for binding actual resources (textures, buffers) to a shader for one draw call.
- **Push constants** — a tiny, very fast alternative to descriptor sets/buffers for small per-draw values.
- **Fence vs. semaphore** — a fence is CPU-visible ("has the GPU finished?"); a semaphore orders GPU work relative to other GPU work and is not directly observable from the CPU.
- **Frames in flight** — how many frames' worth of GPU work the CPU is allowed to have queued up before it must wait. This backend uses exactly one.
- **Image layout transition** — Vulkan requires explicitly telling the driver when an image switches from "being rendered into" to "being sampled from" to "being presented," etc.
- **Swapchain** — the pool of images that get handed to the display in rotation; this *is* what people mean by "double/triple buffering."
