# Revisión inicial de BitchordV2

Fecha: 7 de noviembre de 2026.
Repositorio: clxnazepunk/BitchordV2.
Base examinada: acf219a7e4a7f52f026d4336d5c7ac5cf7bfafcc (1.9.1, código 39).
Sólo existía la rama main; el mantenimiento se propone en chore/android-ci-project-audit.

## Alcance real

Se inventarió el árbol completo del repositorio y se revisaron las configuraciones
de compilación, dependencias declaradas, workflows, puntos de entrada y áreas
sensibles de autenticación, actualización, backups y servidor. Se inspeccionaron
pruebas representativas y la prueba C++ de USB.
Esto es un mapa de arquitectura y una revisión inicial de riesgos, no una
auditoría exhaustiva línea por línea ni una certificación de seguridad.

El PR modifica CI y documentación. No cambia código de aplicación, recursos,
dependencias, permisos, modelos de audio ni protocolos. La conservación del
código se puede comprobar en el diff; la equivalencia en ejecución necesita
pruebas y dispositivos.

## Arquitectura y recorrido principal

| Área | Responsabilidad y relaciones |
| --- | --- |
| app | Aplicación Android Kotlin/Compose. BitChordApplication instala hooks de streaming, adaptadores de UI/plataforma, fuentes, settings y servicios. MainActivity aloja la interfaz. |
| app/playback | PlaybackService con Media3/ExoPlayer y sesión multimedia; PlayerConnection une UI y servicio. QueueCoordinator, historial, autoplay, caché, selección de stream, crossfade, DSP y PartySync. |
| app/data, auth, download | Música local, repositorios, sesiones/cookies, fuentes/addons, descargas y sincronización con WorkManager; adaptadores SMB/WebDAV y actualización del APK. |
| shared | Kotlin Multiplatform: modelos comunes, lógica de colas/automix y capa JVM compartida (YouTube/InnerTube, letras, Spotify, redes, sincronización). Android y JVM consumen commonMain y jvmSharedMain. |
| sharedUi | Pantallas/componentes Compose y player compartidos; implementaciones Android/JVM de plataforma. Compose de escritorio se selecciona por sistema operativo. |
| tv | Interfaz Android TV que reutiliza fuentes, recursos, assets y JNI de app; sustituye clases de forma explícita. Compilar el teléfono no prueba estas sustituciones. |
| wearos | App de reloj, controles y reproducción desde el teléfono; protocolo compartido en wearos/protocol. Su APK se genera e incluye dentro del teléfono. |
| native + app/src/main/cpp | Análisis de ritmo/espectrogramas, mezcla, JNI, DSD/DST, AAudio/Oboe y transporte USB/libusb. CMake compila para Android. |
| backend | Go + Gorilla WebSocket: REST, grupos Listen Together, reloj, códigos, presencia y límites. No usa base de datos; el estado vive en memoria. |
| web | Monorepo pnpm: app React/Vite, extensión WXT Chromium/Firefox y contrato TypeScript entre ambas. Tiene compilación independiente del APK. |

Recorrido simplificado: acción en Compose → repositorios/lógica compartida →
resolución de fuente/stream → PlaybackService/Media3 → DSP/salida Android o USB.
Las descargas y caché permiten uso sin conexión. La UI recibe estado de la
sesión; Listen Together sincroniza estado/cola, no distribuye audio.

settings.gradle.kts incluye shared y sharedUi siempre y app/tv/wearos cuando
detecta SDK. Aunque sus comentarios hablan de desktop, no incluye ningún módulo
de aplicación desktop. No se debe prometer un ejecutable de escritorio a partir
de este árbol.

## Funcionalidades que se preservan

El inventario del código y README comprende reproducción local y YouTube Music,
búsqueda/exploración, biblioteca y listas, cuentas/perfiles, recomendaciones,
colas e historial, descargas y reproducción offline, letras sincronizadas y
traducción, calidad por red, velocidad, silencio, temporizador, ecualización,
crossfade/automix y análisis ONNX. También fuentes/addons, SMB/WebDAV,
integraciones Spotify/Discord/Last.fm, Listen Together, Cast, Android Auto,
Bluetooth, USB/DAC, TV, Wear OS, temas dinámicos e interfaz multilingüe.
El cliente web y su extensión tienen su propio conjunto de adaptadores; no se
asume paridad completa con Android. Las funciones dependientes de servicios
externos no están garantizadas por conservar el código.

## Compilación y dependencias

- Wrapper Gradle 8.11.1, AGP 8.10.1 y JDK/JVM 17.
- Kotlin y plugin Compose Kotlin 2.3.20; Compose Multiplatform 1.10.3.
- Teléfono/TV compileSdk 37, targetSdk 36, minSdk 26; shared/sharedUi/reloj
  compileSdk 36; reloj minSdk 30.
- CMake 3.22.1, NDK predeterminado AGP 27.0.12077973. Splits Android:
  armeabi-v7a, arm64-v8a, x86_64 y universal.
- Google Maven, Maven Central y JitPack. Media3 1.11.0, Ktor 3.5.2,
  serialization 1.11.0, InnerTubeX v0.7.0, NewPipe v0.26.3, QuickJS 1.0.14,
  SMBJ 0.15.0, ONNX Android 1.28.0, Coil 3.0.4 y Haze 1.3.1, entre otras.
- Los módulos shared/sharedUi declaran algunas versiones inferiores de
  coroutines/serialization/Compose; la versión efectiva debe confirmarse con
  :app:dependencies y dependencyInsight. No se cambian en esta preparación.
- Override explícito R8 8.13.23 para un problema documentado de dexing/debug;
  quitarlo puede romper el reproductor. NewPipe se reempaqueta sin Utils.class
  porque la app contiene una implementación parcheada.
- CMake descarga Oboe 1.11.0 y libusb 1.0.27 con hashes SHA-256.
  Signalsmith está vendorizado. FFmpeg/librempeg llega como tres .so
  versionados; native/ffmpeg/build.sh contiene fuente/commit para reconstruir.
- Dos modelos ONNX en assets aumentan peso y memoria.
- backend/go.mod declara Go 1.27.0 y Gorilla WebSocket 1.5.3.
- web requiere Node >=22 y pnpm 12.11.2, con pnpm-lock.yaml;
  React 19, TypeScript 7, Vite 8, Vitest 5 y WXT para la extensión.

Guía paso a paso: [ANDROID_BUILD.md](ANDROID_BUILD.md).
Las versiones son las declaradas en esta base, no recomendaciones de actualización.

## Pruebas y automatización

Inventario de archivos de pruebas Kotlin (no número de casos):
- app/src/test/: 112
- app/src/androidTest/: 2
- shared/src/commonTest/: 9
- shared/src/jvmTest/: 8
- sharedUi/src/jvmTest/: 8
- tv/src/test/: 2
Además hay tests Go, tests TypeScript/Vitest y usb_stream_test.cpp.

Antes ya existían check.yml (pruebas Android/backend y APK), android.yml
(release firmado), tv.yml y web.yml. La preparación amplía los existentes:
- Check: teléfono, TV y tests JVM de shared/sharedUi, lint Android separado,
  APK devDebug completo con reloj, informes incluso con fallos, Go build/vet/race
  y prueba C++ USB con AddressSanitizer/UndefinedBehaviorSanitizer.
- TV: sigue generando su APK propio, sin secretos y con SDK explícito.
- Web: conserva typecheck/test/build e incluye cambios a check.yml como disparador.
- Flujos editados con permisos de lectura, checkout sin credenciales persistidas,
  Actions fijadas por commit y tiempos máximos. No se pasan claves de cuentas ni
  keystores a los PRs.
- Se mantienen los flujos release/despliegue originales sin ejecutarlos ni
  configurarlos. No hay publicación ni despliegue en este PR.
- No se añaden pruebas que sólo confirmen texto de configuración ni se suprimen
  pruebas existentes. Los fallos se deben investigar, no convertir en éxito.

No se configura emulador inicialmente: pruebas instrumentadas, rendimiento,
salida de audio y autenticación de hardware requieren una segunda validación.
Lint puede revelar deuda previa y está configurado para fallar, sin nuevo baseline.

## Riesgos y seguridad

| Prioridad | Evidencia y consecuencia | Próximo paso propuesto (sin implementarlo aquí) |
| --- | --- | --- |
| Alta | AuthStore conserva cookies Google, token de usuario Discord y contraseñas de red; EncryptedPrefs vuelve a preferencias sin cifrar si no funciona Keystore. Hay exclusiones de backup para ambas copias, pero el fallback pierde protección en reposo. | Revisar política de fallback y datos de sesión con dispositivos reales; usar cuentas de prueba. |
| Alta | AddonAuth atesta paquete/certificado y los addons pueden exigir tokens del servicio oficial. Cambiar servidor no otorga confianza al fork. | Acordar soporte con proveedores; probar con firma propia, sin eludir controles. |
| Alta para distribución | AppUpdateChecker consulta releases de kushagrasinghx/BitChord; applicationId prod conserva com.music.bitchord. Las firmas del fork no son las oficiales. | Antes de publicar definir identidad, claves y canal de actualización propios como trabajo separado. |
| Media/alta | Servicios externos originales siguen como defaults; Last.fm se compila en BuildConfig cuando se suministran claves. Un secreto dentro de un APK es recuperable. | No suministrar credenciales reales a builds públicos; revisar modelo de integraciones antes de distribución. |
| Media/alta | Fuentes/addons y código JavaScript QuickJS, WebViews de acceso y puente de extensión manejan datos no confiables/credenciales. | Auditoría específica de permisos, aislamiento, origen, redirecciones y filtrado de logs; esta revisión no demuestra ausencia de fugas. |
| Media | Backend acepta userId/displayName declarados por el cliente; su README explica que no verifica identidad Google. Tokens protegen membresía, no la veracidad del nombre. | Documentar el límite al usarlo y evaluar prueba de identidad en un trabajo posterior. |
| Media | Backend en memoria: reiniciar pierde grupos, múltiples instancias separan el estado. Configurar proxy/origins incorrectamente afecta CORS y límites por IP. | Despliegue único con TLS y configuración de confianza revisada. |
| Media | Binarios .so versionados, decodificadores C/C++, JNI y formatos multimedia no confiables amplían superficie; la compilación normal no reconstruye FFmpeg. | Reconstrucción reproducible, revisión de procedencia, vulnerabilidades y pruebas de archivos malformados antes de publicar. |
| Media | Dependencias JitPack/APIs de extracción/cifrado y servicios de letras cambian fuera del proyecto. No hay lockfile Gradle ni verificación general de artefactos en esta base. | Inventario resuelto y análisis de vulnerabilidades/licencias; no afirmar que no hay CVEs sin ejecutarlo. |
| Media | compileSdk 37 con AGP 8.10.1 excede la API oficialmente soportada indicada por AGP. Hay suppressUnsupportedCompileSdk; silenciar no demuestra compatibilidad. | Comprobar compilación real antes de decidir migrar herramientas. |
| Media | dev permite HTTP (manifest dev), prod lo desactiva. Los tokens de cuentas reales y el APK debug no son apropiados para distribución pública. | Pruebas aisladas; revisión de todas las rutas de red antes de release. |
| Media | Otros workflows heredados usan tags móviles y secretos para release/despliegue; deploy-backend comprueba un dominio del autor original. | Configurar despliegue del fork en una tarea explícita y revisar sus permisos antes de fusionar cambios de backend. |
| Media | Mezcla de GPLv3, LGPL, MIT, Apache y bibliotecas/modelos distribuidos. | Conservar avisos y fuentes; revisión de cumplimiento antes de distribuir. |

Medidas existentes observadas: cifrado preferente de credenciales, exclusiones
de backups, cleartext desactivado en prod, límites de tamaño/rate y origen del
WebSocket, token de grupo fuera de la URL, hashes de fuentes CMake.
No se presentan como prueba de seguridad completa.

## Verificación y límites

En el entorno local no se encontró Git, Gradle, Go, Node CLI ni compilador C++
en PATH y no hay ANDROID_HOME/ANDROID_SDK_ROOT configurado. La conexión de GitHub
permitió leer el fork y crear una rama sin modificar main. No se instaló un SDK
ni se compilaron aquí los binarios. La comprobación de CI real debe registrarse
en el Pull Request con enlaces, estado de cada job y fallos; no asumir éxito
porque exista un workflow.

Pendientes aunque CI resulte verde: instalación y arranque, login/integraciones
en vivo, reproducción real, estado tras actualización/migración, Cast, Auto,
TV, reloj, USB/DAC, consumo/rendimiento y escaneo exhaustivo de seguridad.
La prueba USB de host valida cálculo/formato en memoria, no salida física.

Referencias técnicas:
- [AGP 8.10 y herramientas compatibles](https://developer.android.com/build/releases/agp-8-10-0-release-notes).
- [Eventos y permisos de Actions](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).
