---
id: ART-12
titulo: Arquitectura hexagonal – Móvil Flutter
version: 1.1.0   # 1.1.0: alertas flotantes (§5.1)
estado: vigente
aplica_a: [movil]
depende_de: [ART-02, ART-09, ART-11]
stack: [Flutter estable, Dart null-safety, Riverpod, go_router, dio, freezed, json_serializable, flutter_secure_storage, mocktail, integration_test]
---

# ART-12 · Arquitectura hexagonal – Móvil Flutter

> **Instrucción para agentes IA**
> Misma idea que ART-11: los widgets son adaptadores de entrada; dio es un adaptador de salida. `domain` y `application` son Dart puro (sin `package:flutter`, sin dio, sin Riverpod).

---

## 1. Estructura

```
mobile/
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── providers.dart            # Composition root (Riverpod providers de adaptadores y casos de uso)
│   │   ├── router.dart               # go_router con redirect por sesión y permiso
│   │   └── app.dart                  # MaterialApp.router, tema, localización
│   ├── shared/
│   │   ├── domain/                   # AppError, Permiso, Pagina, value objects comunes
│   │   ├── application/ports/        # HttpClient, SessionStore, SecureStorage, RealtimeClient, Logger
│   │   └── infrastructure/
│   │       ├── http/                 # DioHttpClient + interceptores (auth, refresh, errores, trace)
│   │       ├── session/              # MemorySessionStore, SecureTokenStorage
│   │       ├── realtime/             # WebSocketChannel con reconexión
│   │       ├── security/             # Guardas, PermissionGate widget, protección de pantallas
│   │       ├── errors/               # Catálogo de mensajes por code
│   │       └── ui/                   # Design system: tema, componentes
│   ├── auth/{domain,application,infrastructure}/
│   └── factura/
│       ├── domain/
│       │   ├── factura.dart          # Entidad (freezed NO: el dominio no depende de generadores) + reglas
│       │   ├── factura_errors.dart
│       │   └── factura_repository.dart   # Puerto (abstract interface class)
│       ├── application/
│       │   ├── listar_facturas.dart
│       │   └── crear_factura.dart
│       └── infrastructure/
│           ├── api/                  # DTOs (freezed + json_serializable), mappers, FacturaApiRepository
│           ├── ui/
│           │   ├── pages/
│           │   ├── widgets/
│           │   └── controllers/      # Notifiers/AsyncNotifiers de Riverpod
│           └── routes.dart           # GoRoutes del módulo con permiso requerido
├── test/                             # Unitarias y de widgets (espejo de lib/)
└── integration_test/                 # E2E
```

Reglas de importación: se verifican con `custom_lint` / reglas de análisis propias o revisión en PR (ART-48): `domain` no importa `package:flutter`, `package:dio`, `package:flutter_riverpod`.

---

## 2. Dominio y aplicación

```dart
// factura/domain/factura.dart
enum EstadoFactura { emitida, anulada }

final class Factura {
  const Factura({required this.id, required this.numero, required this.cliente,
                 required this.total, required this.estado});
  final String id;
  final String numero;
  final String cliente;
  final double total;
  final EstadoFactura estado;

  bool get puedeAnularse => estado == EstadoFactura.emitida;
}

// factura/domain/factura_repository.dart
abstract interface class FacturaRepository {
  Future<Pagina<Factura>> listar(FiltroFacturas filtro, PageRequest page);
  Future<Factura> crear(NuevaFactura datos);
  Future<void> anular(String id, String motivo);
}
```

```dart
// factura/application/anular_factura.dart
final class AnularFactura {
  AnularFactura(this._repo, this._session);
  final FacturaRepository _repo;
  final SessionStore _session;

  Future<void> call(Factura factura, String motivo) async {
    if (!_session.puede('factura:anular')) {
      throw const AppError(code: 'ACCESO_DENEGADO', kind: ErrorKind.forbidden,
                           message: 'No tienes permiso para anular facturas.');
    }
    if (!factura.puedeAnularse) {
      throw const AppError(code: 'FACTURA_YA_ANULADA', kind: ErrorKind.businessRule,
                           message: 'La factura ya fue anulada.');
    }
    await _repo.anular(factura.id, motivo.trim());
  }
}
```

---

## 3. Infraestructura

### 3.1 HTTP (dio)
- `baseUrl` desde configuración por flavor (`dev`, `staging`, `prod`) con `--dart-define`; solo HTTPS fuera de dev.
- Interceptores en este orden: `TraceInterceptor` (X-Request-ID) → `AuthInterceptor` (Bearer desde memoria) → `RefreshInterceptor` (401 → un solo refresh concurrente con `/auth/mobile/refresh` → reintento; si falla → cerrar sesión) → `ErrorInterceptor` (convierte a `AppError` por `error.code`).
- Timeouts: conexión 10 s, recepción 30 s.
- Las respuestas se parsean con DTOs generados; un JSON inesperado es `AppError('RESPUESTA_INVALIDA')`.

### 3.2 Composition root (Riverpod)

```dart
final httpClientProvider = Provider<HttpClient>((ref) => DioHttpClient(
      baseUrl: Env.apiUrl, session: ref.watch(sessionStoreProvider)));
final facturaRepositoryProvider =
    Provider<FacturaRepository>((ref) => FacturaApiRepository(ref.watch(httpClientProvider)));
final anularFacturaProvider = Provider((ref) =>
    AnularFactura(ref.watch(facturaRepositoryProvider), ref.watch(sessionStoreProvider)));
```

Los widgets solo leen controladores (`AsyncNotifier`) que invocan casos de uso.

### 3.3 Rutas y permisos (go_router)

```dart
GoRouter buildRouter(SessionStore session) => GoRouter(
  refreshListenable: session,
  redirect: (context, state) {
    final ruta = rutasProtegidas[state.matchedLocation];
    if (session.estado == EstadoSesion.anonimo && !rutasPublicas.contains(state.matchedLocation)) {
      return '/login';
    }
    if (ruta != null && !session.puede(ruta.permiso)) return '/403';
    return null;
  },
  routes: [...authRoutes, ...facturaRoutes],
);
```

Los deep links (`app://`, Universal/App Links) pasan por el mismo `redirect`; los parámetros recibidos se validan como cualquier entrada.

---

## 4. Seguridad móvil (resumen; detalle en ART-28)

| Tema | Regla |
|---|---|
| Tokens | Access en memoria; refresh en `flutter_secure_storage` (Keychain con `first_unlock_this_device`, Keystore con cifrado). Nunca en `SharedPreferences` |
| Datos sensibles | No se persisten en disco salvo necesidad offline aprobada por ADR (cifrados) |
| Logout | Revocar en backend → borrar secure storage → limpiar providers (`ref.invalidate`/contenedor nuevo) → cerrar WS → `go('/login')` |
| Pantallas sensibles | `FLAG_SECURE` en Android y ocultar contenido al pasar a segundo plano en iOS |
| Red | Solo TLS; pinning opcional por ADR (con plan de rotación) |
| Build release | `--obfuscate --split-debug-info`, sin logs, `debugPrint` deshabilitado |
| Inactividad | Cierre de sesión por inactividad y al volver de segundo plano tras X minutos |

---

## 5. Tiempo real

- `web_socket_channel` con ticket de un solo uso (ART-09 §3.2), reconexión con backoff, cierre en logout y al ir a segundo plano prolongado.
- Los eventos invalidan providers (`ref.invalidate(listaFacturasProvider)`); no se usan datos del mensaje para pintar información sensible.

---

## 5.1 Alertas al usuario

- Toda retroalimentación (éxito, información, advertencia, error) usa el puerto `Notificador` (Dart, en `shared/application/ports/`) y su implementación `AlertasOverlay` en `shared/infrastructure/ui/alertas/` (P-UX-01).
- Alertas flotantes en la parte superior (dentro de `SafeArea`), con X en la esquina superior derecha, cierre automático y barra de tiempo inferior que se acorta y se pausa con toque prolongado o con la app en segundo plano. Especificación y referencia de implementación en **ART-29 §1.7**.
- Prohibido usar `SnackBar` o `AlertDialog` sueltos para informar resultados; `AlertDialog` solo para confirmaciones (diálogo de confirmación).
- En logout: `notificador.cerrarTodas()`.

---

## 6. Pruebas

| Tipo | Herramienta | Qué prueba |
|---|---|---|
| Unitarias | `test` + `mocktail` | Dominio y casos de uso |
| Adaptadores | `test` + `http_mock_adapter`/mocks de dio | Mapeo de DTOs, interceptores de refresh y errores |
| Widgets | `flutter_test` | Pantallas por permiso, formularios, estados |
| E2E | `integration_test` (o Patrol) | Flujos por perfil contra backend de pruebas |
| Adversarias | `integration_test` + inspección | Logout limpia secure storage, navegación por deep link sin permiso, datos tras segundo plano |

Cobertura ≥ 80 % en `domain` y `application`.

---

## 7. Checklist del módulo

- [ ] `domain` y `application` sin dependencias de Flutter, dio ni Riverpod.
- [ ] Rutas con permiso; acciones con `PermissionGate`.
- [ ] DTOs validados; errores por `code`.
- [ ] Nada sensible en `SharedPreferences`, logs ni capturas.
- [ ] Logout completo y verificado por prueba.
- [ ] Pruebas unitarias, de widgets e integración en verde.
