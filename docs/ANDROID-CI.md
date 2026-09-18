# Android CI

El workflow `.github/workflows/android.yml` compila un APK de prueba para
`aarch64-linux-android` (ARM64 / arm64-v8a) en Ubuntu. Se ejecuta en pull
requests y pushes a `main` que cambien entradas del build Android; también
admite ejecución manual desde Actions → Android CI → Run workflow.

El build usa Java 17, Android SDK 36, NDK 27.0.12077973, Node 22 y Tauri CLI
2.11.4. Gradle conserva la versión del wrapper del repositorio. Las dependencias
JavaScript se instalan con `npm ci`; Rust usa `Cargo.lock`.
Las cachés de Cargo, npm y Gradle reducen el tiempo de ejecuciones posteriores.
El job tiene un límite de 60 minutos y cancela ejecuciones anteriores de la
misma rama.

Tauri carga automáticamente `tauri.android.conf.json` y ejecuta el build de
`ui-mobile` (TypeScript + Vite). Se instalan también las dependencias de
`ui` para los módulos compartidos. Se reutiliza `gen/android`, incluidas
las Activities y el reproductor nativo; no se regenera con `android init`.
El target Android usa los fallbacks Rust de los componentes Tetsuo según
`build.rs`, por lo que no necesita provisionar el compilador Tetsuo.

## Descargar e instalar

En una ejecución correcta, descarga el artefacto
`videodrome-android-arm64-debug-<run_number>` y descomprime el APK. Se conserva
durante siete días. Puedes instalarlo en un dispositivo ARM64:

```sh
adb install -r ruta/al/archivo.apk
```

Es un APK debug firmado automáticamente con la clave de depuración del runner,
con los assets web empaquetados; no necesita Vite en ejecución ni secretos de
firma. La clave puede cambiar entre runners: si Android rechaza una actualización
por firma incompatible, desinstala la versión anterior antes de instalarlo
(esto borra sus datos). No es un artefacto de publicación en Google Play.

El job comprueba que el APK contiene `lib/arm64-v8a/libvideodrome_lib.so` y
que su firma es válida. No ejecuta pruebas en emulador ni verifica reproducción,
red o comportamiento en dispositivo. La firma de producción, un AAB de release
y otros ABIs quedan fuera de este workflow.
