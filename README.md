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
| `gn-args/device.gn` | gn-Argumente für den Gerätebau (iOS-Route, dann Retarget) | — |
| `gn-args/simulator.gn` | gn-Argumente für den Simulatorbau | — |

## Bezugspunkte

- ANGLE-Basis: `e4499e6b2835a6996507f1b99920bc56f0122573`
  (https://chromium.googlesource.com/angle/angle.git)
- Patch-Reihenfolge: `klepton.patch` → `angle-metal-fixes.patch` →
  `multiview-stage2.patch` → `multiview-stage3.patch` (die letzten beiden
  optional; nur für Multiview-Arbeit).
- Verwendet von: AvpViceCity (Xcode-Projekt referenziert
  `Prototypes/angle-src/Frameworks/`; `Prototypes/angle-patches/` ist ein
  Symlink auf `patches/` hier).

Projektspezifische Hintergründe (Messungen, Entscheidungswege) liegen im
privaten Docs-Repo `visionos-ports-docs` (`vicecity/angle-build.md`,
`common/klepton-foveation-reference.md`) — dieses Repo bleibt bewusst frei
davon, damit es später veröffentlicht oder an Klepton zurückgegeben werden
kann.
