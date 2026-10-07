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
  checksums by its `setup.sh`). On the development machine `Prototypes/angle-patches/`
  is a symlink to `patches/` here.
- Every release of the app carries the same tag in this repository (`v1.0-rc1` …), so the
  patch state that produced the shipped frameworks can always be found.

Project-specific background (measurements, decision records) lives in the private docs
repository `visionos-ports-docs`; this repository is deliberately kept free of it so it can
be published and given back to Klepton.
