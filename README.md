# visionos-angle-kit

A kit to reproduce our ANGLE build (OpenGL ES over Metal, with foveated rendering) on a
fresh machine. It contains everything that cannot be fetched from public sources: the
patches and the build configuration. The procedure is in [BUILD.md](BUILD.md).

## Contents

| Path | What | Origin |
|---|---|---|
| `patches/klepton.patch` | Rasterization-rate-map registry for ANGLE Metal (foveation through GLES) | [shinyquagsire23/Klepton](https://github.com/shinyquagsire23/Klepton), MIT |
| `patches/angle-metal-fixes.patch` | Deferred buffer GC (flicker fix under a rate map), non-triangle skip, diagnostic probes, Werror pragmas | own code, details in the patch header |
| `patches/multiview-stage2.patch` | Advertise GL_OVR_multiview/2 (stage 2, no effect by itself; only with `KL_GL_MULTIVIEW=1`) | own code |
| `patches/multiview-stage3.patch` | gl_ViewID_OVR/gl_Layer in the MSL translator (instanced emulation complete; takes effect at runtime only with stage 4) | own code, details in the patch header |
| `patches/multiview-stage4.patch` | Draw wiring: instances × numViews, layered pass, base-layer uniform, memoryless MSAA arrays (4a+4b) | own code, details in the patch header |
| `patches/multiview-stage5.patch` | Generic building blocks for multiview on system render targets: `GL_EXT_EGL_image_array` (a 2D-array MTLTexture imported without the slice attribute becomes a `GL_TEXTURE_2D_ARRAY` on the same MTLTexture), pass-break counter `passBreaks` and load-action probe `passLoads/s` under `KL_ANGLE_VRR_TRACE`, frame GPU time without a GL query (`KL_MTL_FRAME_GPU_TIME`, `ANGLEMetalPopFrameGpuTimeMs`) | own code, details in the patch header |
| `gn-args/device.gn` | gn arguments for the device build (iOS route, then retarget) | — |
| `gn-args/simulator.gn` | gn arguments for the simulator build | — |

## Reference points

- ANGLE base: `e4499e6b2835a6996507f1b99920bc56f0122573`
  (https://chromium.googlesource.com/angle/angle.git)
- Patch order: `klepton.patch` → `angle-metal-fixes.patch` →
  `multiview-stage2.patch` → `multiview-stage3.patch` →
  `multiview-stage4.patch` → `multiview-stage5.patch` (the multiview patches are
  optional, in exactly this order; only needed for multiview work. They contain no
  game-specific assumptions — another port uses them unchanged).
- Used by: [revc-visionos-app](https://github.com/LowRiderXR/revc-visionos-app)
  (Xcode project `AvpViceCity`; it links the prebuilt xcframeworks from
  `ThirdParty/ANGLE/`, which are attached to its GitHub releases and downloaded with
  checksums by its `setup.sh`).
- Every release of the app carries the same tag in this repository (`v1.0` …), so the
  patch state that produced the shipped frameworks can always be found.

## What the patches require from the guest

Rules that follow from the patches and from ANGLE's own validation; each one cost a device
cycle or a host debugging session to find.

| Patch | Rule for the GL client |
|---|---|
| `klepton.patch` | Only triangles are drawn under a rasterization rate map; lines and points are skipped (the Metal validation layer reports "only triangles may be drawn when using a rasterization rate map", `MTLDebugRenderCommandEncoder`; without validation the behaviour is undefined). A rate map registered by texture identity also applies to a 2D-array texture imported through `multiview-stage5.patch` (same `MTLTexture`, no view). |
| `angle-metal-fixes.patch` | Never start a pass on an implicit-MSAA (memoryless) attachment with `loadAction=Load` while a rate map is active: ANGLE reconstructs the content with an unresolve blit that is **not** rate-map aware, so the result is warped twice. That includes forced pass breaks (texture upload, `glGenerateMipmap`, queries in the middle of a pass) — count them with `passBreaks` under `KL_ANGLE_VRR_TRACE=1`, target 0 per frame in the game pass. |
| `multiview-stage2.patch` | Nothing changes without `KL_GL_MULTIVIEW=1`. With it, a multiview attachment on a 2D-array texture is accepted as is. |
| `multiview-stage3.patch` | Multiview vertex shaders must not write `gl_PointSize` (Metal forbids `[[point_size]]` once the pipeline carries a topology class, and layered rendering needs that class); points and lines into a multiview FBO are skipped. |
| `multiview-stage4.patch` | ANGLE checks `layout(num_views = N)` of the program against the views of the bound framebuffer for **equality**: a `num_views = 2` program cannot draw into a single-view FBO (HUD, shadow cameras, menus) and a mono program cannot draw into the multiview FBO — keep a mono and a multiview variant of every shader that runs on the multiview FBO and select at the actual framebuffer bind. Clears must be real load actions: before `glClear` set full color/depth/stencil write masks and disable the scissor, otherwise the clear becomes a draw and memoryless attachments start with garbage per layer. The explicit MSAA path (shared multisample renderbuffer + blit resolve per eye) is incompatible with a two-layer target — use `OVR_multiview_multisampled_render_to_texture`. |
| `multiview-stage5.patch` | No `GL_TIME_ELAPSED` query may be active while drawing into a multiview FBO (spec rule, enforced: every draw returns `INVALID_OPERATION`); measure GPU time with `KL_MTL_FRAME_GPU_TIME=1` / `ANGLEMetalPopFrameGpuTimeMs` instead, and not together with GL timer queries. Never `glReadPixels` from an imported private texture — read back with your own Metal blit into a shared texture after `glFinish`. |

Project-specific measurements and decision records are kept outside this repository, so it
can be published and given back to Klepton.
