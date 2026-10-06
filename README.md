# visionos-angle-kit

Bausatz, um unseren ANGLE-Stand (OpenGL ES über Metal, mit Foveation) auf
einer frischen Maschine zu reproduzieren. Enthält alles, was nicht aus
öffentlichen Quellen nachladbar ist: die Patches und die Baukonfiguration.
Das Verfahren steht in [BUILD.md](BUILD.md).

## Inhalt

| Pfad | Was | Herkunft |
|---|---|---|
| `patches/klepton.patch` | Rasterization-Rate-Map-Registry für ANGLE-Metal (Foveation über GLES) | [shinyquagsire23/Klepton](https://github.com/shinyquagsire23/Klepton), MIT |
| `patches/angle-metal-fixes.patch` | GC-Aufschub (Flacker-Fix unter Rate Map), Nicht-Dreieck-Skip, Diagnose-Sonden, Werror-Pragmas | eigener Code, Details im Patch-Kopf |
| `patches/multiview-stage2.patch` | GL_OVR_multiview/2 melden (Stufe 2, ohne Wirkung; nur mit `KL_GL_MULTIVIEW=1`) | eigener Code |
| `patches/multiview-stage3.patch` | gl_ViewID_OVR/gl_Layer im MSL-Übersetzer (Instanz-Emulation vollständig; Laufzeitwirkung erst mit Stufe 4) | eigener Code, Details im Patch-Kopf |
| `patches/multiview-stage4.patch` | Draw-Verdrahtung: Instanzen ×numViews, layered Pass, Layer-Basis-Uniform, memoryless MSAA-Arrays (4a+4b) | eigener Code, Details im Patch-Kopf |
| `patches/multiview-stage5.patch` | Generische Bausteine für Multiview auf System-Targets: `GL_EXT_EGL_image_array` (2D-Array-MTLTexture ohne Slice-Attribut → `GL_TEXTURE_2D_ARRAY`, dieselbe MTLTexture), Pass-Abbruch-Zähler `passBreaks` und Load-Action-Sonde `passLoads/s` unter `KL_ANGLE_VRR_TRACE`, Frame-GPU-Zeit ohne GL-Query (`KL_MTL_FRAME_GPU_TIME`, `ANGLEMetalPopFrameGpuTimeMs`) | eigener Code, Details im Patch-Kopf |
| `gn-args/device.gn` | gn-Argumente für den Gerätebau (iOS-Route, dann Retarget) | — |
| `gn-args/simulator.gn` | gn-Argumente für den Simulatorbau | — |

## Bezugspunkte

- ANGLE-Basis: `e4499e6b2835a6996507f1b99920bc56f0122573`
  (https://chromium.googlesource.com/angle/angle.git)
- Patch-Reihenfolge: `klepton.patch` → `angle-metal-fixes.patch` →
  `multiview-stage2.patch` → `multiview-stage3.patch` →
  `multiview-stage4.patch` → `multiview-stage5.patch` (die multiview-Patches
  optional, in dieser Reihenfolge; nur für Multiview-Arbeit. Sie enthalten
  keine Spielannahmen — ein anderer Port nutzt sie unverändert).
- Verwendet von: [revc-visionos-app](https://github.com/LowRiderXR/revc-visionos-app)
  (Xcode-Projekt `AvpViceCity`, bindet die fertigen xcframeworks unter
  `ThirdParty/ANGLE/` ein; sie liegen als Release-Assets dort und werden von
  dessen `setup.sh` mit Prüfsummen geladen). Entwicklungsseitig ist
  `Prototypes/angle-patches/` ein Symlink auf `patches/` hier.

Projektspezifische Hintergründe (Messungen, Entscheidungswege) liegen im
privaten Docs-Repo `visionos-ports-docs` (`vicecity/angle-build.md`,
`common/klepton-foveation-reference.md`) — dieses Repo bleibt bewusst frei
davon, damit es später veröffentlicht oder an Klepton zurückgegeben werden
kann.
