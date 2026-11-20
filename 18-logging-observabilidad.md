---
id: ART-18
titulo: Logging y observabilidad
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, infraestructura]
depende_de: [ART-02, ART-10]
---

# ART-18 · Logging y observabilidad

> **Instrucción para agentes IA**
> Usa los nombres de evento y campos de este documento. Nunca registres datos de la lista de §4. Principios P-LO-01..05.

---

## 1. Esquema de un log (JSON, una línea)

```json
{
  "timestamp": "2026-11-20T16:30:00.123Z",
  "level": "info",
  "logger": "app",
  "service": "api",
  "env": "production",
  "version": "1.4.2",
  "event": "factura_created",
  "trace_id": "01JD3K6Z8Q...",
  "user_id": "0193a1f2-...",
  "empresa_id": "0193a1f2-...",
  "module": "factura",
  "duration_ms": 42,
  "factura_id": "0193a1f3-..."
}
```

Campos fijos: `timestamp`, `level`, `logger`, `service`, `env`, `version`, `event`, `trace_id`. Contexto cuando existe: `user_id`, `empresa_id`, `module`. El resto son campos del evento (identificadores, conteos, duraciones; no contenido).

## 2. Configuración (structlog)

```python
# shared/infrastructure/logging/config.py
import logging, structlog

SENSITIVE_KEYS = {"password", "contrasena", "token", "access_token", "refresh_token", "authorization",
                  "cookie", "secret", "api_key", "numero_tarjeta", "cvv", "documento", "ticket"}

def mask_sensitive(_, __, event_dict):
    for key in list(event_dict):
        if key.lower() in SENSITIVE_KEYS:
            event_dict[key] = "***"
    return event_dict

def configure_logging(env: str, level: str) -> None:
    processors = [
        structlog.contextvars.merge_contextvars,        # trace_id, user_id, empresa_id
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        mask_sensitive,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer() if env != "development" else structlog.dev.ConsoleRenderer(),
    ]
    structlog.configure(processors=processors,
                        wrapper_class=structlog.make_filtering_bound_logger(logging.getLevelName(level)))
```

Middleware: al iniciar el request, `structlog.contextvars.bind_contextvars(trace_id=..., path_template=...)`; al autenticar, `bind_contextvars(user_id=..., empresa_id=...)`; al terminar, `clear_contextvars()`.

Los logs de librerías (uvicorn, sqlalchemy) se redirigen al mismo formato; `sqlalchemy.engine` solo en DEBUG.

## 3. Eventos estándar

| Logger | event | Nivel | Campos |
|---|---|---|---|
| app | `request_completed` | INFO (WARNING si 4xx relevantes, ERROR si 5xx) | method, path_template, status, duration_ms, db_queries |
| app | `app_error` | según kind | code, status, path_template |
| app | `unhandled_error` | ERROR | path_template, exc_info |
| app | `<entidad>_<accion>` (ej. `factura_created`) | INFO | ids |
| audit | `login_succeeded` / `login_failed` | INFO / WARNING | user_id o email enmascarado, ip, motivo |
| audit | `account_locked` | WARNING | user_id, ip |
| audit | `access_denied` | WARNING | user_id, permiso, path_template |
| audit | `token_reuse_detected` | ERROR | user_id, familia |
| audit | `permissions_changed` | INFO | actor_id, perfil_id, agregados, quitados |
| audit | `rate_limited` | WARNING | ip, user_id, ruta |
| audit | `password_changed` / `password_reset_requested` | INFO | user_id |
| events | `event_published` / `event_handler_failed` | DEBUG / ERROR | event_name, event_id, handler |
| events | `outbox_failed` | ERROR | event_id, attempts, error_code |
| ws | `ws_connected` / `ws_disconnected` / `ws_rejected` | INFO / INFO / WARNING | user_id, canal, código de cierre |
| cache | `cache_unavailable` | WARNING | prefijo |

## 4. Nunca se registra (P-LO-04)
Contraseñas, tokens (access, refresh, tickets WS, reset), cookies, cabecera `Authorization`, bodies completos de requests o respuestas, números de tarjeta o cuenta, documentos de identidad completos, datos de salud, contenido libre escrito por usuarios. Correos y teléfonos se enmascaran (`ed***@d***.com`, `***4567`).

## 5. Métricas
Exponer `/metrics` (Prometheus) **solo en la red interna**:
- HTTP: solicitudes por ruta plantilla y status, latencia (histograma).
- BD: consultas por request (histograma), duración, conexiones del pool.
- Caché: hits/misses por prefijo.
- WS: conexiones activas, mensajes enviados.
- Outbox: pendientes, fallidos, latencia de publicación.
- Negocio: contadores de eventos de dominio relevantes.

Herramientas sugeridas: `prometheus-fastapi-instrumentator`, Prometheus + Grafana (o servicio administrado). Para proyectos pequeños basta con logs + alertas (ART-37 §6).

## 6. Trazas (opcional, por ADR)
OpenTelemetry con instrumentación de FastAPI, SQLAlchemy, Redis y httpx; `trace_id` de OTel reemplaza el ULID y se propaga en `traceparent`.

## 7. Frontend y móvil
- Sin `console.*` en producción (regla de lint + eliminación en build).
- Errores no controlados (ErrorBoundary, `window.onerror`, `FlutterError.onError`) se reportan a un servicio (Sentry o endpoint propio `POST /api/v1/client-errors` con rate limit), **con `beforeSend` que elimina datos personales**, URLs con parámetros y cuerpos.
- Se incluye `trace_id` del último request fallido para correlacionar con el backend.

## 8. Retención y acceso
- Logs de aplicación: 30 días en caliente, 90 días archivados.
- Logs de auditoría: 1 año (o lo que exija la normativa del proyecto).
- Acceso a logs de producción restringido y registrado.
