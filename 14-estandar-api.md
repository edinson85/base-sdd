---
id: ART-14
titulo: Estándar de diseño de API
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil]
depende_de: [ART-02, ART-10, ART-17]
---

# ART-14 · Estándar de diseño de API (REST + WebSocket)

> **Instrucción para agentes IA**
> Todo endpoint nuevo sigue estas convenciones y se especifica con ART-46. Si un caso no encaja (operación que no es CRUD), usa §3.3 antes de inventar un estilo propio.

---

## 1. URLs

| Regla | Ejemplo |
|---|---|
| Prefijo y versión | `/api/v1/...` |
| Recursos en plural, kebab-case, sustantivos | `/api/v1/ordenes-compra` |
| Identificador en la ruta | `/api/v1/facturas/{factura_id}` |
| Subrecursos máximo un nivel | `/api/v1/facturas/{id}/items` |
| Sin verbos para CRUD | ❌ `/api/v1/crearFactura` |
| Acciones de negocio como subrecurso verbal (§3.3) | `POST /api/v1/facturas/{id}/anular` |
| Sin extensiones ni barra final | ❌ `/facturas.json`, ❌ `/facturas/` |
| Parámetros de query en snake_case | `?fecha_desde=2026-01-01&estado=EMITIDA` |

## 2. Métodos y códigos de estado

| Operación | Método | Éxito | Body de respuesta |
|---|---|---|---|
| Listar | GET colección | 200 | `{data: [...], meta}` |
| Obtener | GET elemento | 200 | `{data: {...}}` |
| Crear | POST colección | 201 + `Location` | `{data: {...}}` |
| Reemplazar | PUT elemento | 200 | `{data: {...}}` |
| Modificar parcial | PATCH elemento | 200 | `{data: {...}}` |
| Eliminar (lógico o físico) | DELETE elemento | 204 | vacío |
| Acción de negocio | POST `/{id}/<accion>` | 200 (o 202 si es asíncrona) | `{data: {...}}` |
| Proceso asíncrono | POST | 202 + `Location: /api/v1/procesos/{id}` | `{data: {proceso_id, estado}}` |

Errores: ver ART-17 (formato único, P-ER-02). Códigos usados: 400 (petición malformada, JSON inválido), 401, 403, 404, 405, 409, 412 (precondición de concurrencia), 413, 415, 422, 428 (falta `If-Match` cuando es obligatorio), 429, 500, 502, 503.

## 3. Cuerpos

### 3.1 Formato
- JSON UTF-8; `Content-Type: application/json` obligatorio en requests con body (otro → 415).
- Campos en `snake_case` (el front mapea a camelCase en su adaptador).
- Fechas y horas: ISO 8601 con zona en UTC (`2026-11-20T16:30:00Z`); fechas sin hora: `2026-11-20`.
- Dinero y decimales: **string** (`"50000.00"`) para no perder precisión.
- Identificadores: UUID string.
- Enumeraciones: MAYÚSCULAS (`"EMITIDA"`).
- `null` explícito solo cuando el campo existe pero no tiene valor; no se envían campos internos.

### 3.2 Respuesta exitosa
```json
{ "data": { }, "meta": { } }
```
`meta` solo cuando aporta (paginación, totales, avisos).

### 3.3 Operaciones que no son CRUD
| Caso | Diseño |
|---|---|
| Cambio de estado con reglas | `POST /facturas/{id}/anular` con body `{motivo}` |
| Búsqueda compleja (muchos filtros) | `POST /facturas/busquedas` con body de filtros (validado) |
| Operación masiva | `POST /facturas/lote` con máx. N elementos; respuesta con resultado por elemento |
| Reporte o exportación pesada | `POST /reportes/ventas` → 202 + proceso; notificación por WS al terminar; `GET /procesos/{id}/resultado` |

## 4. Listas: paginación, filtros y orden

### 4.1 Paginación por página (por defecto)
`GET /facturas?page=1&size=20` → `meta: {page, size, total}`; `size` máx. 100 (P-DA-04).

### 4.2 Paginación por cursor (tablas grandes o feeds)
`GET /movimientos?size=50&cursor=<opaco>` → `meta: {next_cursor, size}`. El cursor es opaco (base64 de la última clave de orden, firmado o validado al decodificar).

### 4.3 Filtros
- Cada filtro permitido se declara en la especificación con su tipo y validación; filtros no declarados → 422.
- Rangos: `_desde` / `_hasta` (`fecha_desde`, `total_hasta`).
- Texto libre: `q` (mín. 2, máx. 100 caracteres), buscado en campos declarados.

### 4.4 Orden
`?sort=-creada_en,numero` (prefijo `-` = descendente). Solo campos en **lista blanca** del endpoint (nunca se interpola el nombre de columna recibido). Orden por defecto estable: incluye siempre `id` como desempate.

## 5. Concurrencia e idempotencia

| Necesidad | Mecanismo |
|---|---|
| Evitar sobrescribir cambios de otro usuario | Campo `version` (entero) en el recurso; el cliente envía `If-Match: "<version>"` en PUT/PATCH; si no coincide → 412 `CONFLICTO_DE_VERSION` |
| Evitar duplicados por reintentos o doble clic | Header `Idempotency-Key` (UUID) en POST críticos; se guarda en Redis la respuesta por 24 h asociada a usuario + key + hash del body; mismo key con body distinto → 422 `IDEMPOTENCY_KEY_REUTILIZADA` |

## 6. Cabeceras

| Cabecera | Uso |
|---|---|
| `Authorization: Bearer <token>` | Autenticación (excepto refresh web por cookie) |
| `X-Request-ID` | Trazabilidad (se genera si no viene; se devuelve siempre) |
| `Idempotency-Key` | §5 |
| `If-Match` / `ETag` | §5 y caché HTTP condicional en recursos de lectura frecuente |
| `Cache-Control: no-store` | Respuestas con datos personales o sensibles (P-SB-07) |
| `Retry-After` | En 429 y 503 |
| `Deprecation`, `Sunset` | Endpoints en retiro |

## 7. Versionado y evolución
- Cambios compatibles (agregar campos opcionales, nuevos endpoints, nuevos valores de enum que el cliente tolera) no cambian versión.
- Cambios incompatibles → `/api/v2/...` para el recurso afectado; `v1` convive con cabeceras `Deprecation`/`Sunset` al menos 3 meses (o el tiempo que tarde la app móvil en actualizarse).
- Los clientes deben ignorar campos desconocidos.

## 8. Documentación
- OpenAPI generado por FastAPI, con `summary`, `description`, ejemplos y respuestas de error declaradas (`responses={404: {...}}`).
- Se exporta `openapi.json` en CI y se versiona como artefacto; sirve para Schemathesis (ART-42) y para generar tipos del front si el proyecto lo decide.
- En producción `/docs` y `/openapi.json` deshabilitados (P-SB-08).

## 9. WebSocket
- Ruta: `/ws/v1/<recurso>`; autenticación por ticket (ART-20 §5).
- Mensajes servidor → cliente: `{"event": "<recurso>.<hecho>", "data": {...}, "event_id": "...", "trace_id": "...", "ts": "..."}`; nombres de evento en minúscula con punto (`factura.creada`).
- Mensajes cliente → servidor: `{"type": "<tipo>", ...}` validados con esquema; tipos mínimos `ping`, `subscribe`, `unsubscribe` (si el canal lo permite).
- Especificación por canal con ART-46 Anexo B.

## 10. Endpoints estándar de todo proyecto

| Endpoint | Descripción |
|---|---|
| `GET /health/live`, `GET /health/ready` | Internos (no expuestos por Nginx) |
| `POST /api/v1/auth/login` · `refresh` · `logout` · `mobile/refresh` | ART-20 |
| `GET /api/v1/auth/me` | Usuario actual + permisos |
| `POST /api/v1/auth/ws-ticket` | Ticket WS |
| `POST /api/v1/auth/password/forgot` · `reset` · `change` | ART-20 |
| `GET /api/v1/procesos/{id}` | Estado de procesos asíncronos |
