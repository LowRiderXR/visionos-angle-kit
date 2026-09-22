# ANGLE für visionOS bauen — Reproduktion auf frischer Maschine

Ziel: `ANGLE_libEGL.xcframework` und `ANGLE_libGLESv2.xcframework` mit
xros-Slices, Klepton-Foveation und unseren Metal-Fixes.

Verifizierter Stand: gebaut mit Xcode 26.5 (Build `17F113`), läuft auf
visionOS 26.6. ANGLE hat kein xros-Target — gebaut wird für iOS, dann
retargetet (unten).

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
```

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

## Laufzeit-Schalter der Patches

| Variable | Wirkung |
|---|---|
| `KL_ANGLE_VRR_TRACE=1` | Diagnose-Sonden an (Banner, encode/s, cmds/s, bufferGC, …) |
| `KL_MTL_GC_DEFER=0` | GC-Aufschub aus (Referenzverhalten für A/B) |
| `KL_MTL_GC_DEFER_CAP_MB` | Deckel des Aufschubs, Default 256 |
| `KL_MTL_BUFFER_GC_MB` | GC-Speicherboden, Default 1 (nur Notausgang) |
