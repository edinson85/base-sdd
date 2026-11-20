---
id: ART-28
titulo: Seguridad móvil (Flutter)
version: 1.0.0
estado: vigente
aplica_a: [movil]
depende_de: [ART-02, ART-12, ART-20]
---

# ART-28 · Seguridad móvil (Flutter)

> **Instrucción para agentes IA**
> Referencia: OWASP MASVS. Cada control tiene su categoría. Los marcados "ADR" son opcionales según el riesgo del proyecto (pagos, salud, datos financieros → aplicarlos).

---

## 1. Controles por categoría MASVS

| Categoría | Control | Implementación en Flutter |
|---|---|---|
| STORAGE | Tokens y secretos solo en almacenamiento seguro | `flutter_secure_storage` (Keychain `first_unlock_this_device`; Android con cifrado del Keystore) |
| STORAGE | Nada sensible en `SharedPreferences`, archivos, logs, portapapeles | Revisión + lint; deshabilitar copiar en campos sensibles (`enableInteractiveSelection: false`) |
| STORAGE | Sin datos sensibles en backups | Android: `android:allowBackup="false"` o reglas `dataExtractionRules`; iOS: excluir archivos del backup |
| STORAGE | Caché de red sin datos sensibles | dio sin caché persistente para respuestas autenticadas |
| CRYPTO | Sin criptografía propia; sin llaves embebidas | Solo librerías estándar; las llaves de API públicas se asumen expuestas |
| AUTH | Tokens cortos, refresh rotativo, logout real | ART-20 + ART-12 §4 |
| AUTH | Biometría como desbloqueo local (opcional, ADR) | `local_auth` protegiendo el acceso al refresh token; nunca como único factor en el servidor |
| NETWORK | Solo TLS | Android `network_security_config` con `cleartextTrafficPermitted="false"`; iOS ATS activo |
| NETWORK | Certificate pinning (ADR) | Pin de la llave pública (SPKI) con al menos 2 pines (actual + respaldo) y plan de rotación; implementado en el `HttpClient` de dio |
| PLATFORM | Deep links validados | Rutas por `go_router` con redirect de permisos; parámetros validados; App Links / Universal Links verificados (no esquemas personalizados para operaciones sensibles) |
| PLATFORM | WebViews | Evitar; si se usan: sin JavaScript salvo necesidad, solo dominios permitidos, sin `addJavascriptChannel` con datos sensibles |
| PLATFORM | Capturas de pantalla | Android `FLAG_SECURE` en pantallas sensibles; iOS ocultar contenido al pasar a segundo plano (pantalla de privacidad) |
| PLATFORM | Permisos del sistema | Solo los necesarios, solicitados en contexto |
| CODE | Ofuscación | `flutter build --release --obfuscate --split-debug-info=build/symbols` |
| CODE | Sin logs en release | `debugPrint` neutralizado; logger que no imprime en release |
| CODE | Dependencias | `flutter pub outdated`, revisión de paquetes, versiones fijadas en `pubspec.lock` |
| RESILIENCE | Detección de root/jailbreak y emulador (ADR) | Paquetes de detección; respuesta: advertir o limitar funciones, nunca confiar solo en esto |
| PRIVACY | Datos mínimos, consentimiento | Ley 1581; texto de autorización en registro; sin SDKs de terceros que recolecten datos sin ADR |

## 2. Ciclo de vida de la sesión en móvil
1. Login → access en memoria, refresh en secure storage.
2. Inicio de la app → si hay refresh, llamar a `/auth/mobile/refresh`; si falla, borrar y mostrar login.
3. App en segundo plano > N minutos (configurable) → al volver, exigir desbloqueo (biometría/PIN si ADR) o cerrar sesión.
4. Logout → revocar en backend → `secureStorage.deleteAll()` → reiniciar el `ProviderContainer` (o invalidar todos los providers con datos) → cerrar WS → `go('/login')` limpiando la pila de navegación.
5. Desinstalar/reinstalar en iOS: el Keychain puede persistir; al primer arranque tras instalación (marca en `SharedPreferences` no sensible) borrar el secure storage.

## 3. Configuración de build
| Plataforma | Ajuste |
|---|---|
| Android | `minSdk` acordado; `android:usesCleartextTraffic="false"`; `allowBackup` según §1; firma con llave en secreto del CI; `debuggable` falso en release |
| iOS | ATS sin excepciones; `NSFaceIDUsageDescription` si hay biometría; protección de datos `NSFileProtectionComplete` |
| Ambos | Flavors `dev/staging/prod` con `--dart-define`; URL de API de producción solo HTTPS |

## 4. Pruebas
- [ ] Inspeccionar el almacenamiento de la app en un dispositivo de prueba: sin tokens fuera del Keychain/Keystore, sin datos personales en archivos.
- [ ] Interceptar el tráfico con un proxy (mitmproxy) sin certificado instalado: la app no debe conectar; con pinning, tampoco con certificado instalado.
- [ ] Deep link a una pantalla sin permiso → `/403` o login.
- [ ] Captura de pantalla en pantallas marcadas → bloqueada (Android) / oculta en el selector de apps (iOS).
- [ ] Logout y reapertura: no se ven datos previos.
- [ ] APK/IPA de release: sin símbolos legibles, sin URLs de desarrollo, sin logs.
