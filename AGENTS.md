# Instrucciones del proyecto

## Alcance y flujo
- Explica resultados, fallos y límites en español sencillo.
- Trabaja en una rama descriptiva y entrega un Pull Request; nunca publiques directamente en main ni fusiones sin instrucciones.
- Consulta CONTRIBUTING.md y la rama activa antes de empezar. En la revisión inicial del fork sólo existía main.
- Conserva todas las funcionalidades. No añadas funciones ni hagas refactorizaciones, actualizaciones masivas o cambios de identidad/firma sin petición explícita.
- No ocultes fallos con continue-on-error, exclusiones, borrado de pruebas o nuevos baselines de lint. Distingue fallos previos de los introducidos.

## Arquitectura
- :app: Android, Kotlin y Compose; servicios Media3, autenticación, descargas y adaptadores de plataforma.
- :shared: modelos y lógica Kotlin Multiplatform; commonMain sin Android ni Compose; jvmSharedMain para Android/JVM.
- :sharedUi: interfaz Compose compartida y adaptadores Android/JVM. Revisa las versiones de Compose específicas de Windows/Linux.
- :tv reutiliza código, recursos, assets y JNI de :app, con sustituciones explícitas. Un cambio del teléfono puede romper TV.
- :wearos y wearos/protocol: app de reloj y contrato común con el teléfono; emparejar exige mismo applicationId y certificado.
- native/ y app/src/main/cpp/: DSP, JNI, DSD, USB; preserva licencias y opciones de precisión.
- backend/: servidor Go de Listen Together; estado en memoria y WebSockets.
- web/: monorepo pnpm (app React, extension WXT, shared bridge). No es una interfaz del APK.
- settings.gradle.kts es la fuente de módulos incluidos. No hay módulo desktop de aplicación incluido en esta revisión, aunque hay targets JVM.

## Convenciones
- Kotlin oficial, paquetes y separación de responsabilidades existentes; no reformatees archivos ajenos.
- Mantén consistentes los contratos de colas, sesiones, descargas y protocolos entre clientes.
- Prueba lógica nueva o corregida con casos significativos; usa servidores simulados y fixtures cuando sea posible.
- No presentes telemetría de audio como prueba física de salida USB/DAC.
- No elimines el override R8 ni la transformación de NewPipe sin reproducir y comprobar el problema que resuelven.
- No cambies metadatos/versiones, identificadores, servidor de autenticación o actualizador como efecto secundario de mantenimiento.
- Dependencias nuevas: justificar tamaño, procedencia y compatibilidad GPLv3. Preservar avisos LGPL/MIT/Apache y fuentes correspondientes.

## Compilación y verificaciones
- JDK 17, Gradle wrapper 8.11.1; SDK Android 37 y 36, build-tools 35.0.0, NDK 27.0.12077973 y CMake 3.22.1 para el entorno CI inicial.
- Consulta docs/ANDROID_BUILD.md para comandos completos y entrega de APK.
- Con SDK válido, ejecuta:
  - ./gradlew :app:testDevDebugUnitTest :tv:testDevDebugUnitTest :shared:jvmTest :sharedUi:jvmTest
  - ./gradlew :app:lintDevDebug
  - ./gradlew :app:assembleDevDebug (incluye Wear OS por defecto)
- En Windows usa .\gradlew.bat en lugar de ./gradlew.
- En backend: go build ./..., go vet ./..., go test -race -count=1 ./... con la versión de go.mod.
- En web: pnpm install --frozen-lockfile, pnpm typecheck, pnpm test, pnpm build; Node >=22 y pnpm de packageManager; VITE_PARTY_SERVER_URL para build.
- Prueba USB de host: comando en .github/workflows/check.yml; no verifica hardware real.
- Reporta comandos realmente ejecutados, resultados y bloqueos. Un workflow escrito no equivale a una compilación comprobada.
- Antes de declarar preservación funcional, revisar el diff y ejecutar pruebas pertinentes; documentar la parte manual pendiente.

## Seguridad y CI
- PRs sin secretos, permisos mínimos contents: read y checkout sin credenciales persistidas.
- Nunca uses pull_request_target para compilar código de un PR.
- Usa Actions fijadas por SHA al modificar flujos; revisa su origen al actualizarlas.
- No guardes keystores, cookies, tokens, contraseñas, local.properties ni keystore.properties en Git, logs o artifacts.
- APK de prueba firmado con clave debug desechable; no uses la clave de distribución para PRs.
- El APK de una ejecución nueva puede exigir desinstalar el anterior por cambio de firma: advierte de pérdida de datos antes de recomendarlo.
- No des despliegues ni releases por realizados al compilar; el PR no autoriza fusionar.
- Consulta docs/PROJECT_AUDIT.md para riesgos y límites conocidos.
