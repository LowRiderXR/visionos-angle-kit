# ANGLE für visionOS bauen — Reproduktion auf frischer Maschine

Ziel: `ANGLE_libEGL.xcframework` und `ANGLE_libGLESv2.xcframework` mit
xros-Slices, Klepton-Foveation und unseren Metal-Fixes.

Verifizierter Stand: gebaut mit Xcode 26.5 (Build `17F113`) UND mit
Xcode 27.0 (Build `27A266a`, iOS-27.0-SDK — siehe Abschnitt 7 für die dabei
nötigen Fixes); läuft auf visionOS 26.6. ANGLE hat kein xros-Target —
gebaut wird für iOS, dann retargetet (unten).

## 1. Quellen holen

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PWD/depot_tools:$PATH" DEPOT_TOOLS_UPDATE=0

mkdir angle && cd angle
fetch angle                       # oder: git clone + gclient sync
git checkout e4499e6b2835a6996507f1b99920bc56f0122573
gclient sync
```

## 2. Patches anwenden

Reihenfolge ist Pflicht:

```bash
git apply <kit>/patches/klepton.patch
git apply <kit>/patches/angle-metal-fixes.patch
git apply <kit>/patches/multiview-stage2.patch   # optional ab hier: Multiview,
git apply <kit>/patches/multiview-stage3.patch   #   in genau dieser Reihenfolge
git apply <kit>/patches/multiview-stage4.patch
git apply <kit>/patches/multiview-stage5.patch
```

Prüfung der Kette auf frischem Baum (`git worktree add … e4499e6b28`, dann
alle sechs nacheinander mit `git apply --check`) ist Teil jedes Exports.

## 3. Bauen

```bash
gn gen out/ios     --args="$(cat <kit>/gn-args/device.gn)"
gn gen out/ios-sim --args="$(cat <kit>/gn-args/simulator.gn)"
autoninja -C out/ios libEGL libGLESv2
autoninja -C out/ios-sim libEGL libGLESv2
```

**Achtung SDK-Bindung:** gn backt absolute SDK-Pfade (`-isysroot …/iPhoneOS<ver>.sdk`,
`CR_XCODE_BUILD`) in jede Kommandozeile. Ein Xcode-Wechsel erzwingt `gn gen`
neu und damit einen Komplett-Rebuild (~2100 Objekte). ANGLE@e4499e6b ist nie
gegen ein 27er-SDK gebaut worden — Kompilierfehler einplanen. Für kleine
Änderungen ohne `gn gen` gibt es den chirurgischen Weg (Abschnitt 6).

## 4. Retarget iOS → visionOS

Pro Slice (Gerät: `visionos`, Simulator: `visionossim`):

```bash
vtool -set-build-version visionos 1.0 26.0 -replace \
      -output libGLESv2 libGLESv2
# Info.plist (binär, mit plutil):
#   CFBundleSupportedPlatforms -> XROS bzw. XRSimulator
#   MinimumOSVersion           -> 1.0
#   UIDeviceFamily             -> entfernen
install_name_tool -id @rpath/libGLESv2.framework/libGLESv2 libGLESv2
```

Dann beide Slices bündeln:

```bash
xcodebuild -create-xcframework \
  -framework <device>/libGLESv2.framework \
  -framework <sim>/libGLESv2.framework \
  -output ANGLE_libGLESv2.xcframework
```

Analog für libEGL. Ergebnis-Identifier: `xros-arm64` und `xros-arm64-simulator`.

## 5. Verifizieren

```bash
# Patches wirklich einkompiliert? (angewandt und kompiliert sind zwei Dinge)
nm -gU ANGLE_libGLESv2.xcframework/xros-arm64/libGLESv2.framework/libGLESv2 \
  | grep ANGLEMetalSetRasterizationRateMap
```

Zur Laufzeit: `KL_ANGLE_VRR_TRACE=1` setzen — das Banner
`[angle-vrr] TRACE ACTIVE rev=10 …` bestätigt, dass der frische Build
geladen wurde (rev-Nummer im Banner gegen den Patch-Stand prüfen).

Einbinden in Xcode: **Embed & Sign** (Copy Files → Frameworks, Code Sign On
Copy), **nicht** unter "Link Binary With Libraries" — geladen wird per
`dlopen`; libEGL lädt libGLESv2 selbst über `@rpath`, beide müssen ins Bundle.

## 6. Chirurgischer Rebuild (einzelne Dateien, ohne gn gen)

Wenn das eingebackene SDK nicht mehr existiert, aber nur einzelne
Übersetzungseinheiten geändert sind (ANGLE bringt sein clang in
`third_party/llvm-build` mit, es variieren nur die SDK-Header):

1. Kommandozeile ziehen: `ninja -t commands -s obj/<pfad>/<datei>.o`,
   darin `-isysroot` auf ein vorhandenes iOS-SDK tauschen (Xcode-beta trägt
   ältere SDKs), `-Wno-error` anhängen, ausführen.
2. Link-Response-Datei nachbauen (`obj/<target>.rsp`, Inhalt laut
   `toolchain.ninja`: `${in} ${frameworks} ${swiftmodules} ${solibs} ${libs}`;
   die Edge steht in `obj/<target>_framework_shared_library.ninja`).
   Implizite (`|`) und order-only (`||`) Deps gehören **nicht** in `${in}`.
3. Linken mit derselben Pfad-Ersetzung, dann Retarget wie in Abschnitt 4.
   Nur `libGLESv2` linkt das Metal-Backend — `libEGL` bleibt meist unberührt.

## 7. Neubau bei Xcode-Wechsel — Protokoll Xcode 27 (2026-09-23)

Beim ersten Bau von ANGLE@e4499e6b28 gegen das iOS-27.0-SDK traten genau
drei Fehler auf; alle drei sind gn-Argumente, keine Quelländerungen (in
`gn-args/*.gn` bereits enthalten):

| Fehler | Ursache | Fix |
|---|---|---|
| `ld64.lld: could not load TAPI file … MacOSX27.0.sdk/….tbd: malformed file / unknown target` beim Host-Tool `protoc` (~Objekt 455) | ANGLEs gebündeltes altes lld kann die tbd-Stubs neuer SDKs nicht parsen; `protoc` kommt nur über die Perfetto-Abhängigkeit herein | `angle_enable_perfetto = false` (Tracing ist ohnehin aus) |
| `error: function 'fprintf' is unsafe [-Werror,-Wunsafe-buffer-usage-in-libc-call]` in unseren Trace-Sonden (ContextMtl.mm u. a.) | Der unsafe-buffers-Clang-Plugin-Lint + `-Werror`; beim chirurgischen Bauen war `-Wno-error` angehängt, unter ninja nicht | **Datei-lokale Pragmas** (`#pragma clang diagnostic ignored "-Wunsafe-buffer-usage[-in-libc-call]"`) in den sechs Sonden-Dateien, Teil von `angle-metal-fixes.patch`. BEWUSST NICHT `treat_warnings_as_errors = false`: das wäre global und senkte die Warnschwelle im ganzen Baum; verifiziert 2026-09-23, dass der Baum mit `-Werror` und nur den Pragmas fehlerfrei baut |
| `ld64.lld: could not load TAPI file … iPhoneOS27.0.sdk/….tbd` beim Link von libEGL/libGLESv2 (nach ~1272 Objekten) | dieselbe lld-Schwäche, jetzt am Ziel-Link — unumgehbar mit gebündeltem lld | `use_lld = false` (Apples Linker aus dem aktiven Xcode) |

Vorgehen, das sich bewährt hat:

1. **Neues out-Verzeichnis** (`out/ios27`), das alte nicht anfassen — der
   alte ninja-Zustand ist die einzige Rückfallebene und gegen ein SDK
   gebaut, das es nicht mehr gibt. Ebenso `Frameworks-pre27/` als Kopie der
   zuletzt ausgelieferten xcframeworks anlegen, bevor installiert wird.
2. `gn gen` mit den gn-args aus diesem Kit, Build, bei Fehlern die Tabelle
   oben zuerst prüfen.
3. Retarget/Bündeln wie in Abschnitt 4 (unter Xcode 27 unverändert gültig;
   `-install_name @rpath/…` setzt der Build bereits, `install_name_tool`
   entfällt).
4. Verifikation: `nm`-Check (Abschnitt 5), rev-Banner, dann auf Gerät
   Bildkorrektheit UND Performance gegen bekannte Werte — ein Build, der
   läuft, aber langsamer ist, ist kein Erfolg.

Zeitrahmen des protokollierten Laufs: gn gen + 2×~1300 Objekte + 3
Fehlerrunden ≈ 45 Minuten. Auf Gerät abgenommen 2026-09-23: rev-Banner,
Rate-Map-Spike 18/18 mit identischen Messwerten zum 26.5-Build, Bild
korrekt, eye-GPU-Zeiten unverändert.

## 8. Host-Build für Schreibtisch-Tests (macOS)

Derselbe Baum baut mit `target_os = "mac"` (sonst gleiche gn-args) einen
macOS-Metal-ANGLE als `libEGL.dylib`/`libGLESv2.dylib` plus das Host-Tool
`angle_shader_translator`. Damit lassen sich GL-Läufe (Link, Draw, Readback)
komplett ohne Gerät reproduzieren — Backend via
`eglGetPlatformDisplayEXT(EGL_PLATFORM_ANGLE_ANGLE, {TYPE_METAL_ANGLE})`
erzwingen (der macOS-Default kann der native-GL-Backend sein, in dem keine
unserer Patches leben). Fehlertexte über `GL_KHR_debug`, das Laufzeit-MSL
über `glGetTranslatedShaderSourceANGLE`. Achtung: `angle_shader_translator`
aus einem iOS-out ist ein iOS-Binary — macOS beendet es kommentarlos mit
SIGKILL. Bei Stufe 4a fand diese Schleife vier Defekte, die sonst je einen
Gerätezyklus gekostet hätten.

## Laufzeit-Schalter der Patches

| Variable | Wirkung |
|---|---|
| `KL_ANGLE_VRR_TRACE=1` | Diagnose-Sonden an (Banner, encode/s, cmds/s, bufferGC, `passBreaks=N` = rate-gemappte/geschichtete Pässe, die im Frame mit Store endeten und mit Load weitergehen — Ziel 0) |
| `KL_GL_MULTIVIEW=1` | `GL_OVR_multiview/2` (+ `multisampled_render_to_texture`) melden; ohne die Variable sieht der Gast kein Multiview |
| `KL_MTL_GC_DEFER=0` | GC-Aufschub aus (Referenzverhalten für A/B) |
| `KL_MTL_GC_DEFER_CAP_MB` | Deckel des Aufschubs, Default 256 |
| `KL_MTL_BUFFER_GC_MB` | GC-Speicherboden, Default 1 (nur Notausgang) |
