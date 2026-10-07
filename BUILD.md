# Building ANGLE for visionOS — reproduction on a fresh machine

Goal: `ANGLE_libEGL.xcframework` and `ANGLE_libGLESv2.xcframework` with xros slices,
Klepton foveation and our Metal fixes.

Verified state: the shipped frameworks were built with Xcode 27.0 (build `27A266a`,
iOS 27.0 SDK — see section 7 for the fixes that needed) and are tested on visionOS 27.
An earlier build with Xcode 26.5 (build `17F113`) ran on visionOS 26.6; the current one
has not been tested there. ANGLE has no xros target — it is built for iOS and then
retargeted (below). The shipped slices carry `platform VISIONOS, minos 1.0, sdk 26.0`
(check with `vtool -show-build`).

## 1. Get the sources

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PWD/depot_tools:$PATH" DEPOT_TOOLS_UPDATE=0

mkdir angle && cd angle
fetch angle                       # or: git clone + gclient sync
git checkout e4499e6b2835a6996507f1b99920bc56f0122573
gclient sync
```

## 2. Apply the patches

The order is mandatory:

```bash
git apply <kit>/patches/klepton.patch
git apply <kit>/patches/angle-metal-fixes.patch
git apply <kit>/patches/multiview-stage2.patch   # optional from here on: multiview,
git apply <kit>/patches/multiview-stage3.patch   #   in exactly this order
git apply <kit>/patches/multiview-stage4.patch
git apply <kit>/patches/multiview-stage5.patch
```

Checking the chain on a clean tree (`git worktree add … e4499e6b28`, then all six in
sequence with `git apply --check`) is part of every export. The chain was last verified
2026-10-07: applied to clean e4499e6b sources it reproduces the tree that built the
shipped frameworks byte for byte.

## 3. Build

```bash
gn gen out/ios     --args="$(cat <kit>/gn-args/device.gn)"
gn gen out/ios-sim --args="$(cat <kit>/gn-args/simulator.gn)"
autoninja -C out/ios libEGL libGLESv2
autoninja -C out/ios-sim libEGL libGLESv2
```

**SDK binding:** gn bakes absolute SDK paths (`-isysroot …/iPhoneOS<ver>.sdk`,
`CR_XCODE_BUILD`) into every command line. Switching Xcode forces a new `gn gen` and
with it a full rebuild (~2100 objects). ANGLE@e4499e6b was never built upstream against a
27 SDK — expect compile errors (section 7). For small changes without `gn gen` there is
the surgical route (section 6).

## 4. Retarget iOS → visionOS

Per slice (device: `visionos`, simulator: `visionossim`):

```bash
vtool -set-build-version visionos 1.0 26.0 -replace \
      -output libGLESv2 libGLESv2
# Info.plist (binary, edit with plutil):
#   CFBundleSupportedPlatforms -> XROS or XRSimulator
#   MinimumOSVersion           -> 1.0
#   UIDeviceFamily             -> remove
install_name_tool -id @rpath/libGLESv2.framework/libGLESv2 libGLESv2
```

Then bundle both slices:

```bash
xcodebuild -create-xcframework \
  -framework <device>/libGLESv2.framework \
  -framework <sim>/libGLESv2.framework \
  -output ANGLE_libGLESv2.xcframework
```

Same for libEGL. Resulting identifiers: `xros-arm64` and `xros-arm64-simulator`.

## 5. Verify

```bash
# Are the patches really compiled in? (applied and compiled are two different things)
nm -gU ANGLE_libGLESv2.xcframework/xros-arm64/libGLESv2.framework/libGLESv2 \
  | grep ANGLEMetalSetRasterizationRateMap
```

At runtime: set `KL_ANGLE_VRR_TRACE=1` — the banner `[angle-vrr] TRACE ACTIVE rev=10 …`
confirms that the fresh build was loaded (compare the rev number in the banner with the
patch state).

Integration in Xcode: the frameworks must be **embedded and signed** (Copy Files →
Frameworks, Code Sign On Copy), because the app loads them at runtime via `dlopen` and
libEGL loads libGLESv2 itself through `@rpath` — both must be in the bundle. Linking them
in addition under "Link Binary With Libraries" is harmless (revc-visionos-app does both).

## 6. Surgical rebuild (single files, without gn gen)

When the baked-in SDK no longer exists but only a few translation units changed (ANGLE
ships its own clang in `third_party/llvm-build`; only the SDK headers vary):

1. Pull the command line: `ninja -t commands -s obj/<path>/<file>.o`, replace
   `-isysroot` with an existing iOS SDK (an Xcode beta carries older SDKs), append
   `-Wno-error`, run it.
2. Rebuild the link response file (`obj/<target>.rsp`, contents per `toolchain.ninja`:
   `${in} ${frameworks} ${swiftmodules} ${solibs} ${libs}`; the edge is in
   `obj/<target>_framework_shared_library.ninja`). Implicit (`|`) and order-only (`||`)
   deps do **not** belong in `${in}`.
3. Link with the same path substitution, then retarget as in section 4. Only `libGLESv2`
   links the Metal backend — `libEGL` usually stays untouched.

## 7. Rebuilding after an Xcode change — Xcode 27 log (2026-09-23)

The first build of ANGLE@e4499e6b28 against the iOS 27.0 SDK produced exactly three
errors; all three are gn arguments, not source changes (already contained in `gn-args/*.gn`):

| Error | Cause | Fix |
|---|---|---|
| `ld64.lld: could not load TAPI file … MacOSX27.0.sdk/….tbd: malformed file / unknown target` for the host tool `protoc` (~object 455) | ANGLE's bundled old lld cannot parse the tbd stubs of newer SDKs; `protoc` only comes in through the Perfetto dependency | `angle_enable_perfetto = false` (tracing is off anyway) |
| `error: function 'fprintf' is unsafe [-Werror,-Wunsafe-buffer-usage-in-libc-call]` in our trace probes (ContextMtl.mm and others) | the unsafe-buffers clang plugin lint plus `-Werror`; the surgical build had `-Wno-error` appended, ninja does not | **file-local pragmas** (`#pragma clang diagnostic ignored "-Wunsafe-buffer-usage[-in-libc-call]"`) in the six probe files, part of `angle-metal-fixes.patch`. Deliberately NOT `treat_warnings_as_errors = false`: that would be global and lower the warning threshold for the whole tree; verified 2026-09-23 that the tree builds cleanly with `-Werror` and only the pragmas |
| `ld64.lld: could not load TAPI file … iPhoneOS27.0.sdk/….tbd` when linking libEGL/libGLESv2 (after ~1272 objects) | the same lld weakness, now at the target link — unavoidable with the bundled lld | `use_lld = false` (Apple's linker from the active Xcode) |

Procedure that worked:

1. **New out directory** (`out/ios27`), leave the old one alone — the old ninja state is
   the only fallback and was built against an SDK that no longer exists. Likewise keep a
   copy of the last shipped xcframeworks before installing new ones.
2. `gn gen` with the gn args from this kit, build, on errors check the table above first.
3. Retarget/bundle as in section 4 (unchanged under Xcode 27; the build already sets
   `-install_name @rpath/…`, so `install_name_tool` can be skipped).
4. Verification: `nm` check (section 5), rev banner, then on the device image correctness
   AND performance against known values — a build that runs but is slower is not a success.

Time frame of the logged run: gn gen + 2 × ~1300 objects + 3 error rounds ≈ 45 minutes.
Accepted on the device 2026-09-23: rev banner, rate-map spike 18/18 with measurements
identical to the 26.5 build, image correct, eye GPU times unchanged.

## 8. Host build for desk tests (macOS)

The same tree builds a macOS Metal ANGLE as `libEGL.dylib`/`libGLESv2.dylib` plus the
host tool `angle_shader_translator` with `target_os = "mac"` (otherwise the same gn args).
This reproduces GL runs (link, draw, readback) entirely without a device — force the
backend via `eglGetPlatformDisplayEXT(EGL_PLATFORM_ANGLE_ANGLE, {TYPE_METAL_ANGLE})` (the
macOS default may be the native GL backend, which contains none of our patches). Error
texts via `GL_KHR_debug`, the runtime MSL via `glGetTranslatedShaderSourceANGLE`. Note:
`angle_shader_translator` from an iOS out directory is an iOS binary — macOS kills it
silently with SIGKILL. During stage 4a this loop found four defects that would otherwise
have cost one device cycle each.

## Runtime switches of the patches

| Variable | Effect |
|---|---|
| `KL_ANGLE_VRR_TRACE=1` | Diagnostic probes on (banner, encode/s, cmds/s, bufferGC, `passBreaks=N` = rate-mapped/layered passes that ended with Store inside a frame and continue with Load — target 0; `passLoads/s` = load actions color/depth/stencil of those passes per second, expected Clear/Clear/Clear) |
| `KL_GL_MULTIVIEW=1` | Advertise `GL_OVR_multiview/2` (+ `multisampled_render_to_texture`); without the variable the guest sees no multiview |
| `KL_MTL_FRAME_GPU_TIME=1` | Collect frame GPU time per glFlush frame (Σ command-buffer durations); the guest reads it via `ANGLEMetalPopFrameGpuTimeMs`; replacement for `GL_TIME_ELAPSED`, which is forbidden under multiview; not together with GL timer queries |
| `KL_MTL_GC_DEFER=0` | Deferred GC off (reference behaviour for A/B) |
| `KL_MTL_GC_DEFER_CAP_MB` | Cap of the deferral, default 256 |
| `KL_MTL_BUFFER_GC_MB` | GC memory floor, default 1 (emergency exit only) |
