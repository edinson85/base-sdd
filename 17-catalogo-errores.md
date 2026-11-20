---
id: ART-17
titulo: Manejo de excepciones y catálogo de errores
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil]
depende_de: [ART-02, ART-10, ART-11]
---

# ART-17 · Manejo de excepciones y catálogo de errores

> **Instrucción para agentes IA**
> - El archivo `docs/errores.yaml` del proyecto es la **fuente única** de códigos de error. De él se generan las clases del backend y los mensajes del front y del móvil.
> - Todo error nuevo se agrega primero al YAML (con mensaje para el usuario) y después se usa en código.
> - Jerarquía y manejadores: ART-10 §3.2 y §6.6. Presentación: ART-29.

---

## 1. Errores globales (heredados por todo proyecto)

| code | kind | HTTP | Mensaje al usuario |
|---|---|---|---|
| `VALIDACION_FALLIDA` | VALIDATION | 422 | Algunos datos no son válidos. Revisa los campos indicados. |
| `PETICION_INVALIDA` | VALIDATION | 400 | La solicitud no tiene un formato válido. |
| `TIPO_CONTENIDO_NO_SOPORTADO` | VALIDATION | 415 | El tipo de contenido enviado no es compatible. |
| `CUERPO_DEMASIADO_GRANDE` | VALIDATION | 413 | La solicitud es demasiado grande. |
| `NO_AUTENTICADO` | UNAUTHENTICATED | 401 | Tu sesión expiró. Ingresa nuevamente. |
| `CREDENCIALES_INVALIDAS` | UNAUTHENTICATED | 401 | Usuario o contraseña incorrectos. |
| `CUENTA_BLOQUEADA` | UNAUTHENTICATED | 401 | Tu cuenta está bloqueada temporalmente. Intenta más tarde o recupera tu contraseña. |
| `ACCESO_DENEGADO` | FORBIDDEN | 403 | No tienes permiso para realizar esta acción. |
| `RECURSO_NO_ENCONTRADO` | NOT_FOUND | 404 | El recurso solicitado no existe o no tienes acceso a él. |
| `METODO_NO_PERMITIDO` | VALIDATION | 405 | Operación no permitida. |
| `CONFLICTO` | CONFLICT | 409 | La operación entra en conflicto con el estado actual. Actualiza e intenta de nuevo. |
| `CONFLICTO_DE_VERSION` | CONFLICT | 412 | Otro usuario modificó este registro. Recarga para ver los cambios. |
| `PRECONDICION_REQUERIDA` | VALIDATION | 428 | Falta información de versión para completar la operación. |
| `IDEMPOTENCY_KEY_REUTILIZADA` | VALIDATION | 422 | Esta solicitud ya fue usada con datos diferentes. |
| `LIMITE_EXCEDIDO` | RATE_LIMITED | 429 | Demasiadas solicitudes. Espera un momento e intenta de nuevo. |
| `SERVICIO_EXTERNO_NO_DISPONIBLE` | EXTERNAL | 502 | Un servicio externo no respondió. Intenta más tarde. |
| `SERVICIO_NO_DISPONIBLE` | INTERNAL | 503 | El servicio no está disponible en este momento. |
| `ERROR_INTERNO` | INTERNAL | 500 | Ocurrió un error inesperado. Intenta de nuevo más tarde. |

Solo cliente (front/móvil): `SIN_CONEXION`, `TIEMPO_AGOTADO`, `RESPUESTA_INVALIDA`.

## 2. Formato de `docs/errores.yaml`

```yaml
version: 1
modulos:
  factura:
    FACTURA_NO_ENCONTRADA:
      kind: NOT_FOUND
      http: 404
      mensaje: "La factura no existe o no tienes acceso a ella."
      cuando: "Id inexistente o de otra empresa"
      log: INFO
    FACTURA_DUPLICADA:
      kind: CONFLICT
      http: 409
      mensaje: "Ya existe una factura con ese número."
      cuando: "RN-1: número repetido en la empresa"
      log: INFO
    FACTURA_YA_ANULADA:
      kind: BUSINESS_RULE
      http: 422
      mensaje: "La factura ya fue anulada y no puede modificarse."
      cuando: "Se intenta anular o editar una factura ANULADA"
      log: INFO
```

Reglas del catálogo:
- `code` único en todo el proyecto, prefijado por el módulo (`FACTURA_`, `CLIENTE_`).
- `mensaje`: en español, dice qué pasó y, si aplica, qué hacer; sin datos internos (P-ER-03). Puede tener variables nombradas `{campo}` solo con valores no sensibles.
- Un código **nunca** cambia de significado; si cambia, se crea uno nuevo.

## 3. Generación
- Script `scripts/gen_errores.py` (en CI se verifica que lo generado esté al día):
  - Backend: `app/<modulo>/domain/errors.py` con una clase por código que hereda de la base según `kind`.
  - Front: `src/<modulo>/domain/errors.ts` (union de códigos) y `src/shared/infrastructure/errors/mensajes.<modulo>.ts`.
  - Móvil: `lib/<modulo>/domain/<modulo>_errors.dart`.
- Si el proyecto no usa el generador, el catálogo YAML sigue siendo obligatorio y la revisión del PR compara código vs. YAML.

## 4. Traducción de excepciones de infraestructura

| Origen | Excepción | Se traduce a |
|---|---|---|
| PostgreSQL | `UniqueViolation` (IntegrityError) | `ConflictError` específico según el constraint (mapa `constraint → code`); si no está mapeado, `CONFLICTO` |
| PostgreSQL | `ForeignKeyViolation` | `NOT_FOUND` de la referencia o `CONFLICTO` si es al borrar |
| PostgreSQL | `CheckViolation` | `VALIDACION_FALLIDA` (no debería ocurrir si el dominio valida: se registra como WARNING) |
| PostgreSQL | `QueryCanceled` (statement_timeout) | `SERVICIO_NO_DISPONIBLE` + ERROR en logs |
| PostgreSQL | Conexión | `SERVICIO_NO_DISPONIBLE` |
| Redis | Conexión / timeout | No se propaga (degradación, P-CA-05), salvo en funciones que dependen de Redis (rate limit → permitir y registrar; tickets WS → `SERVICIO_NO_DISPONIBLE`) |
| HTTP externo | Timeout, 5xx | `SERVICIO_EXTERNO_NO_DISPONIBLE` |
| HTTP externo | 4xx | Error de negocio específico del adaptador o `ERROR_INTERNO` si es un bug nuestro |
| Pydantic (request) | `RequestValidationError` | `VALIDACION_FALLIDA` con `details` campo/regla |
| JSON inválido | — | `PETICION_INVALIDA` |

La traducción ocurre **en el adaptador** que conoce la excepción (repositorio, cliente), nunca en el router.

## 5. `details`
```json
"details": [
  { "field": "items.0.cantidad", "rule": "greater_than_equal", "message": "Debe ser mayor o igual a 1." }
]
```
- `field`: ruta del campo con puntos e índices.
- `rule`: identificador de la regla (estable, sirve al front para mensajes propios).
- `message`: opcional, en español, sin el valor recibido (P-VA-07).

## 6. Reglas de manejo
1. El dominio lanza errores de negocio; la aplicación lanza `ForbiddenError`/`NotFoundError`; la infraestructura traduce lo técnico.
2. Nunca capturar `Exception` salvo en: manejador global, bus de eventos (aislar observers), workers (registrar y reintentar).
3. Nunca devolver 200 con un error en el body.
4. Errores 5xx siempre con `trace_id` y log `ERROR` con stack trace.
5. En workers: error de negocio → no reintentar, marcar `failed`; error técnico → reintentar con backoff.
