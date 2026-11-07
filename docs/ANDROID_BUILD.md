# Compilar y probar Android

La variante inicial es **devDebug**: se instala como **BitChord Dev**
(`com.dev.bitchord`) junto a la app normal. No requiere las claves del autor.
No es una versión de producción.

## Entorno

Instala JDK 17 (recomendado Temurin), Android Studio/SDK y las herramientas:

```sh
sdkmanager "platforms;android-37" "platforms;android-36" "build-tools;35.0.0" "ndk;27.0.12077973" "cmake;3.22.1"
```

Se necesita conexión para Maven Central, Google Maven, JitPack y las fuentes
Oboe/libusb de CMake. El flujo falla si un paquete requerido no está disponible;
no baja el SDK ni retira dependencias para forzar un APK.

Android 37 es el compileSdk de teléfono/TV; 36 se usa en bibliotecas/reloj.
El teléfono admite Android 8/API 26 o posterior; Wear OS requiere API 30.
`targetSdk` del teléfono sigue en 36.
Gradle 8.11.1 viene mediante el wrapper; AGP es 8.10.1, Kotlin 2.3.20.
El NDK indicado es el predeterminado de AGP 8.10 y se instala explícitamente en CI.
No se modifican versiones de dependencias en este PR.

Define `ANDROID_HOME` con la carpeta del SDK o crea `local.properties` con
`sdk.dir` (Android Studio lo escribe por ti). Sin SDK válido, settings omite
app/tv/wearos; eso no demuestra que los targets JVM puedan compilar sin SDK,
porque shared y sharedUi también aplican el plugin Android.

## Comandos desde la raíz

En Linux/macOS:

```sh
chmod +x gradlew
./gradlew --version
./gradlew :app:testDevDebugUnitTest :tv:testDevDebugUnitTest :shared:jvmTest :sharedUi:jvmTest
./gradlew :app:lintDevDebug
./gradlew :app:assembleDevDebug
```

En Windows (PowerShell):

```powershell
.\gradlew.bat --version
.\gradlew.bat :app:testDevDebugUnitTest :tv:testDevDebugUnitTest :shared:jvmTest :sharedUi:jvmTest
.\gradlew.bat :app:lintDevDebug
.\gradlew.bat :app:assembleDevDebug
```

El último comando compila teléfono y reloj con la misma clave debug e incluye
el APK del reloj como asset del teléfono. No usar
`-Pbitchord.bundleWear=false` para el APK de prueba completo.
TV se compila por separado: `./gradlew :tv:assembleDevDebug`.

Resultados:
- Teléfono: `app/build/outputs/apk/dev/debug/*.apk`.
- Reloj: `wearos/build/outputs/apk/dev/debug/*.apk`.
- TV: `tv/build/outputs/apk/dev/debug/*.apk`.
- Usar el APK universal del teléfono para evitar elegir arquitectura.
- Pruebas: `<módulo>/build/reports/tests/`.
- Lint: `app/build/reports/lint-results-devDebug.html`.

## Obtener el APK desde GitHub

1. Abrir **Actions → Check** en este fork.
2. Elegir la ejecución del Pull Request/rama. También admite **Run workflow**
   cuando la definición esté disponible en la rama predeterminada.
3. Revisar cada job: pruebas, lint, servidor y USB.
4. Descargar el artifact `bitchord-dev-debug-<número>` y extraer el ZIP.
5. Instalar el APK universal del teléfono. Android pedirá permiso para instalar
   desde el gestor de archivos. Los artifacts caducan a los 14 días.

Se intenta compilar y guardar APK incluso si fallan las pruebas, para facilitar
el diagnóstico; la ejecución sigue marcada como fallida. Un artifact por sí
solo no significa que todas las verificaciones aprobaron.
El reloj se firma con la misma clave que el teléfono en esa ejecución.
Cada ejecución usa clave desechable: una posterior puede no actualizar la
anterior. No desinstalar sin respaldar datos que se quieran conservar.

Si Actions está desactivado en el fork, el propietario debe habilitarlo en
GitHub. Una ejecución que necesita aprobación debe ser aprobada desde GitHub.
El flujo no utiliza secretos ni publica releases ni despliega el servidor.

## Integraciones y firma

Sin credenciales Last.fm, sus funciones autenticadas no se pueden comprobar.
Los servidores y el actualizador conservan los destinos del proyecto original.
Los addons con autenticación oficial pueden rechazar la firma debug/del fork.
No se intenta eludir esa comprobación.

Una versión prodRelease utiliza la identidad de producción y, sin un keystore
válido, sale sin firmar. No se debe distribuir como si fuera la versión oficial.
No incluir claves de producción en workflows de PR.

## Pruebas en dispositivo pendientes

Con teléfono/emulador conectado:
`./gradlew :app:connectedDevDebugAndroidTest`.
Existen pruebas instrumentadas de Android Auto; no se ejecutan en el flujo inicial.
Comprobar manualmente inicio, reproducción local/en línea, descarga y modo sin
conexión, colas, letras, cuentas, enlaces, notificaciones y reproducción con
pantalla apagada. Cast, USB/DAC, Bluetooth, TV y Wear requieren entornos
compatibles. Una prueba de host no certifica audio bit-perfect ni atestación.

Referencias:
- [AGP 8.10: compatibilidad de herramientas](https://developer.android.com/build/releases/agp-8-10-0-release-notes).
- [Configuración y permisos de Actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository).
