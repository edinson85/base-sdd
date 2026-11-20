---
id: ART-46
titulo: Plantilla de especificación de endpoint (y de canal WebSocket)
version: 1.0.0
estado: vigente
aplica_a: [backend]
depende_de: [ART-02, ART-10]
---

# ART-46 · Plantilla de especificación de endpoint

> **Instrucción para agentes IA**
> - Genera **una especificación por endpoint** con esta plantilla durante el plan técnico, **antes** de escribir código (P-SDD-02).
> - Llena **todas** las secciones. Si una no aplica, escribe `N/A` y la razón en una línea. Nunca borres secciones.
> - Las secciones 6 (consultas), 7 (caché), 9 (errores) y 12 (pruebas adversarias) son obligatorias y son las que más se revisan.
> - Guarda cada especificación como `specs/endpoints/<modulo>/<METODO>-<recurso>.md`.
> - Si el endpoint publica eventos o usa WebSocket, completa también el **Anexo A** (evento) y/o el **Anexo B** (canal WebSocket).
> - Al final de este documento hay un **ejemplo completo** (§E) para usar como referencia de nivel de detalle.

---

## Plantilla

````markdown
---
id: EP-<MODULO>-<NNN>
modulo: <modulo>
metodo: GET | POST | PUT | PATCH | DELETE
ruta: /api/v1/<recurso>/...
historias: [HU-XXX]
estado: borrador | aprobado | implementado
version: 1.0.0
---

# EP-<MODULO>-<NNN> · <Nombre corto de la acción>

## 1. Propósito
<Qué hace, para quién y por qué. 2-4 líneas.>

## 2. Seguridad de acceso
| Aspecto | Valor |
|---|---|
| Autenticación | Requerida / Pública (justificar) |
| Permiso requerido | `<recurso>:<accion>` |
| Perfiles que lo tienen por defecto | <Admin, Cajero...> |
| Alcance (anti-IDOR) | <Filtro aplicado: empresa_id = actor.empresa_id, dueño = actor.user_id...> |
| Respuesta si está fuera de alcance | 404 `<CODIGO>_NO_ENCONTRADO` |
| Rate limit | <N req/min por usuario; N req/min por IP> |
| Idempotencia | <N/A / Header `Idempotency-Key`, ventana de X horas> |
| Tamaño máximo de body | <N KB> |
| CSRF | <Aplica si se usa cookie para autenticar / N/A> |

## 3. Entrada

### 3.1 Parámetros de ruta
| Campo | Tipo | Obligatorio | Reglas |
|---|---|---|---|

### 3.2 Parámetros de query
| Campo | Tipo | Obligatorio | Por defecto | Reglas (rango, valores permitidos, longitud, patrón) |
|---|---|---|---|---|

### 3.3 Body (`application/json`, `extra="forbid"`)
| Campo | Tipo | Obligatorio | Long. mín/máx | Patrón / caracteres permitidos | Rango / valores | Notas |
|---|---|---|---|---|---|---|

### 3.4 Archivos (si aplica)
| Campo | Tipos MIME (por contenido) | Extensiones | Tamaño máx. | Cantidad máx. | Almacenamiento |
|---|---|---|---|---|---|

### 3.5 Campos que el cliente NUNCA puede enviar (anti asignación masiva)
<Lista: empresa_id, estado, total, creado_por...>

### 3.6 Ejemplo de request
```json
```

## 4. Reglas de negocio
| ID | Regla | Dónde se valida (dominio / caso de uso) | Error si falla |
|---|---|---|---|
| RN-1 | | | |

## 5. Flujo del caso de uso
1. Router: valida esquema → `require_permission`.
2. Caso de uso: `actor.exigir(...)`.
3. ...
N. Después del commit: publica <eventos>.

## 6. Acceso a datos
### 6.1 Consultas esperadas
| # | Propósito | Tipo (SELECT/INSERT/UPDATE) | Tablas | Técnica (join, ANY, ventana, selectinload) |
|---|---|---|---|---|

**Presupuesto máximo de consultas:** `N` (sin contar la autenticación cuando los permisos están cacheados).

### 6.2 Índices requeridos
| Tabla | Índice | Justificación | ¿Existe? |
|---|---|---|---|

### 6.3 Transacción
<Una transacción con las consultas X a Y / Solo lectura.>

## 7. Caché
| Aspecto | Valor |
|---|---|
| ¿Lee de caché? | Sí / No (justificar) |
| Clave | `app:<modulo>:<recurso>:v1:<ids>` |
| TTL | <segundos> |
| Invalidación | <Eventos que la invalidan y observer responsable> |
| Comportamiento si Redis falla | Degrada a BD + WARNING |
| ¿Este endpoint invalida otras claves? | <Cuáles, vía qué evento> |

## 8. Salida exitosa
**Status:** <200 | 201 | 204>

| Campo | Tipo | Descripción | Visible para |
|---|---|---|---|

Cabeceras: `Cache-Control: <no-store si hay datos sensibles>`, `Location` (si 201).

```json
{ "data": {} }
```

## 9. Errores
| HTTP | code | kind | Cuándo | Mensaje al usuario | Nivel de log |
|---|---|---|---|---|---|
| 401 | NO_AUTENTICADO | UNAUTHENTICATED | Token ausente o inválido | Tu sesión expiró. Ingresa nuevamente. | WARNING (audit) |
| 403 | ACCESO_DENEGADO | FORBIDDEN | Sin permiso | No tienes permiso para realizar esta acción. | WARNING (audit) |
| 422 | VALIDACION_FALLIDA | VALIDATION | Esquema inválido | Algunos datos no son válidos... | INFO |
| 429 | LIMITE_EXCEDIDO | RATE_LIMITED | Rate limit | Demasiadas solicitudes. Espera un momento. | WARNING (audit) |
| 500 | ERROR_INTERNO | INTERNAL | No controlado | Ocurrió un error inesperado... | ERROR |
| ... | <propios del endpoint> | | | | |

## 10. Eventos y efectos secundarios
| Evento publicado | Momento | Mecanismo (bus en memoria / Outbox) | Observers | Efecto |
|---|---|---|---|---|

¿Notifica por WebSocket? <Canal y evento, o N/A> → ver Anexo B.

## 11. Logging y auditoría
| Evento de log | Nivel | Campos (sin datos sensibles) |
|---|---|---|

Datos que NO se registran: <lista>.

## 12. Pruebas
### 12.1 Unitarias (dominio y caso de uso)
- [ ] <Caso feliz>
- [ ] <Cada regla RN-x falla con su error>
- [ ] <Sin permiso → ForbiddenError, sin llamar al repositorio>
- [ ] <Eventos publicados después del commit>

### 12.2 Integración y API
- [ ] Contrato de respuesta exitosa y de cada error de §9
- [ ] Presupuesto de consultas ≤ N
- [ ] Caché: hit, miss, invalidación, Redis caído

### 12.3 Adversarias (P-PR-05)
- [ ] Sin token / token expirado / firma manipulada / `alg: none`
- [ ] Perfil sin el permiso
- [ ] IDOR: recurso de otra empresa o usuario → 404
- [ ] Campos extra y asignación masiva (`empresa_id`, `estado`...) → 422
- [ ] Inyección SQL en cada campo de texto y de query
- [ ] XSS almacenado en campos de texto (se guarda como texto; la salida no se interpreta)
- [ ] Longitudes límite (máx, máx+1), valores vacíos, solo espacios, Unicode (emoji, RTL, homoglifos, caracteres de control)
- [ ] Tipos incorrectos, números negativos, desbordes, decimales con más precisión de la permitida
- [ ] Body mayor al tamaño máximo → 413
- [ ] Rate limit superado → 429
- [ ] <Específicas del endpoint: doble envío, carrera, archivos maliciosos...>

### 12.4 E2E
- [ ] <Flujo de usuario que usa este endpoint, por perfil>

## 13. Checklist OWASP API Top 10
| Riesgo | Mitigación en este endpoint |
|---|---|
| API1 Broken Object Level Authorization | |
| API2 Broken Authentication | |
| API3 Broken Object Property Level Authorization | |
| API4 Unrestricted Resource Consumption | |
| API5 Broken Function Level Authorization | |
| API6 Unrestricted Access to Sensitive Business Flows | |
| API7 Server Side Request Forgery | |
| API8 Security Misconfiguration | |
| API9 Improper Inventory Management | |
| API10 Unsafe Consumption of APIs | |

## 14. Decisiones y pendientes
- <Supuestos, ADRs relacionados, preguntas abiertas>
````

---

## Anexo A · Plantilla de evento de dominio

````markdown
---
id: EV-<MODULO>-<NNN>
nombre: <NombreEnPasado>
modulo: <modulo>
version: 1
---

# EV-<MODULO>-<NNN> · <NombreEnPasado>

| Aspecto | Valor |
|---|---|
| Se emite cuando | <Condición de negocio> |
| Emitido por | <Agregado.metodo()> |
| Publicación | Bus en memoria después del commit / Outbox |
| Payload | <campos y tipos: solo identificadores y datos mínimos> |
| Datos excluidos del payload | <datos sensibles que NO viajan> |

## Observers
| Observer | Módulo | Efecto | Idempotencia (cómo) | Si falla |
|---|---|---|---|---|

## Pruebas
- [ ] El agregado registra el evento con el payload correcto
- [ ] Cada observer es idempotente (ejecutarlo dos veces = un solo efecto)
- [ ] La falla de un observer no afecta la operación principal ni a los otros observers
````

---

## Anexo B · Plantilla de canal WebSocket

````markdown
---
id: WS-<MODULO>-<NNN>
ruta: /ws/v1/<recurso>
---

# WS-<MODULO>-<NNN> · <Nombre del canal>

## 1. Propósito y justificación
<Por qué necesita tiempo real (P-EV-05). Si basta con refrescar, no usar WS.>

## 2. Seguridad
| Aspecto | Valor |
|---|---|
| Autenticación en handshake | Cookie / ticket de un solo uso (TTL ≤ 30 s) |
| Origins permitidos | <lista> |
| Permiso requerido | `<recurso>:<accion>` |
| Canal(es) asignados por el servidor | `empresa:{empresa_id}:<recurso>` |
| ¿El cliente puede suscribirse a canales concretos? | No / Sí: verificación de permiso + alcance por cada suscripción |
| Tamaño máx. de mensaje entrante | <bytes> |
| Frecuencia máx. de mensajes entrantes | <N/seg> |
| Conexiones máx. por usuario | <N> |
| Cierre forzado | Logout, expiración, cambio de permisos (canal `user:{id}:control`) |

## 3. Mensajes servidor → cliente
| event | Disparado por (evento de dominio) | data | Acción esperada del cliente |
|---|---|---|---|

## 4. Mensajes cliente → servidor
| type | Esquema | Respuesta | Errores |
|---|---|---|---|

## 5. Códigos de cierre
| Código | Motivo |
|---|---|
| 1000 | Cierre normal |
| 1008 | Política: sin autenticación, sin permiso u origin inválido |
| 1009 | Mensaje demasiado grande |
| 1013 | Intentar más tarde (sobrecarga o rate limit) |

## 6. Pruebas adversarias
- [ ] Conexión sin cookie/ticket, con ticket reutilizado o expirado → 1008
- [ ] Origin no permitido → 1008
- [ ] Usuario sin permiso → 1008
- [ ] Suscripción a canal de otra empresa → rechazada
- [ ] Mensaje malformado, tipo desconocido, mensaje gigante, ráfaga de mensajes
- [ ] Tras logout el socket se cierra y no recibe más eventos
- [ ] Un evento no filtra datos que el destinatario no puede ver
````

---

## §E · Ejemplo completo

````markdown
---
id: EP-FACTURA-001
modulo: factura
metodo: POST
ruta: /api/v1/facturas
historias: [HU-012]
estado: aprobado
version: 1.0.0
---

# EP-FACTURA-001 · Crear factura

## 1. Propósito
Permite a un usuario con permiso registrar una factura de venta con sus ítems para la empresa a la que pertenece. Notifica en tiempo real a los usuarios de la misma empresa que tienen abierto el listado.

## 2. Seguridad de acceso
| Aspecto | Valor |
|---|---|
| Autenticación | Requerida (Bearer access token) |
| Permiso requerido | `factura:crear` |
| Perfiles que lo tienen por defecto | Administrador, Cajero |
| Alcance (anti-IDOR) | `empresa_id` se toma de `actor.empresa_id`; `cliente_id` y cada `producto_id` deben pertenecer a esa empresa |
| Respuesta si está fuera de alcance | 404 `CLIENTE_NO_ENCONTRADO` / 404 `PRODUCTO_NO_ENCONTRADO` |
| Rate limit | 30 req/min por usuario |
| Idempotencia | Header `Idempotency-Key` (UUID) obligatorio; ventana de 24 h |
| Tamaño máximo de body | 64 KB |
| CSRF | N/A: autenticación por Bearer, no por cookie |

## 3. Entrada
### 3.1 Parámetros de ruta
N/A: no tiene.

### 3.2 Parámetros de query
N/A: no tiene.

### 3.3 Body
| Campo | Tipo | Obligatorio | Long. mín/máx | Patrón | Rango / valores | Notas |
|---|---|---|---|---|---|---|
| numero | string | Sí | 3 / 20 | `^[A-Z0-9\-]+$` | — | Único por empresa |
| cliente_id | UUID | Sí | — | UUID v4/v7 | — | Debe existir en la empresa |
| items | array | Sí | 1 / 200 elementos | — | — | |
| items[].producto_id | UUID | Sí | — | — | — | Debe existir y estar activo en la empresa |
| items[].descripcion | string | Sí | 1 / 200 | `^[\w\sáéíóúÁÉÍÓÚñÑüÜ.,\-()/#]+$` | — | |
| items[].cantidad | int | Sí | — | — | 1 a 10 000 | |
| items[].precio_unitario | decimal | Sí | — | máx. 12 dígitos, 2 decimales | > 0 y ≤ 999 999 999.99 | |

### 3.4 Archivos
N/A: no recibe archivos.

### 3.5 Campos que el cliente NUNCA puede enviar
`id`, `empresa_id`, `estado`, `total`, `creada_en`, `creado_por`.

### 3.6 Ejemplo de request
```json
{
  "numero": "FV-000123",
  "cliente_id": "0193a1f2-7c4e-7b11-9d2a-4f1b2c3d4e5f",
  "items": [
    { "producto_id": "0193a1f2-8a10-7c22-8e3b-1a2b3c4d5e6f", "descripcion": "Martillo 16 oz", "cantidad": 2, "precio_unitario": "25000.00" }
  ]
}
```

## 4. Reglas de negocio
| ID | Regla | Dónde | Error |
|---|---|---|---|
| RN-1 | El número de factura es único por empresa | Caso de uso (consulta) + índice único en BD | 409 `FACTURA_DUPLICADA` |
| RN-2 | El cliente existe, está activo y es de la empresa | Caso de uso | 404 `CLIENTE_NO_ENCONTRADO` |
| RN-3 | Todos los productos existen, están activos y son de la empresa | Caso de uso | 404 `PRODUCTO_NO_ENCONTRADO` (details con los índices de ítems) |
| RN-4 | El total lo calcula el servidor | Dominio (`Factura.crear`) | — |

## 5. Flujo del caso de uso
1. Router: valida `CrearFacturaIn` y `Idempotency-Key` → `require_permission("factura:crear")`.
2. Si la `Idempotency-Key` ya fue procesada, devuelve la respuesta guardada (Redis).
3. Caso de uso `CrearFactura`: `actor.exigir("factura:crear")`.
4. Abre UoW. Consulta #1 valida RN-1, RN-2 y RN-3 en una sola consulta.
5. `Factura.crear(...)` calcula el total y registra `FacturaCreada`.
6. Consultas #2 y #3: inserta la factura y los ítems (inserción múltiple). Commit.
7. Publica `FacturaCreada` (bus en memoria) → observers.
8. Guarda la respuesta para la `Idempotency-Key` y responde 201.

## 6. Acceso a datos
### 6.1 Consultas esperadas
| # | Propósito | Tipo | Tablas | Técnica |
|---|---|---|---|---|
| 1 | Validar número duplicado, cliente y productos | SELECT | facturas, clientes, productos | Una consulta con subconsultas `EXISTS` y `id = ANY(:producto_ids)` que devuelve banderas y los ids de productos encontrados |
| 2 | Insertar factura | INSERT | facturas | — |
| 3 | Insertar ítems | INSERT | items_factura | Inserción múltiple (`executemany`) |

**Presupuesto máximo de consultas:** `3`.

### 6.2 Índices requeridos
| Tabla | Índice | Justificación | ¿Existe? |
|---|---|---|---|
| facturas | `UNIQUE (empresa_id, numero)` | RN-1; protege contra condiciones de carrera | No, crear en la migración |
| clientes | PK `id` + columna `empresa_id` | RN-2 | Sí |
| productos | PK `id` + columna `empresa_id` | RN-3 | Sí |
| items_factura | `(factura_id)` | Joins de detalle y lista | No, crear en la migración |

### 6.3 Transacción
Una transacción con las consultas 1 a 3. Si el índice único lanza `IntegrityError` por una carrera, el repositorio lo traduce a `FacturaDuplicada`.

## 7. Caché
| Aspecto | Valor |
|---|---|
| ¿Lee de caché? | No: es una escritura |
| Clave | N/A |
| TTL | N/A |
| Invalidación | N/A |
| Comportamiento si Redis falla | La idempotencia degrada a solo la restricción única de BD + WARNING |
| ¿Invalida otras claves? | Sí: `app:factura:lista:v1:{empresa_id}:*` mediante el observer `InvalidarCacheFacturas` ante `FacturaCreada` |

## 8. Salida exitosa
**Status:** 201. Cabeceras: `Location: /api/v1/facturas/{id}`, `Cache-Control: no-store`.

| Campo | Tipo | Descripción | Visible para |
|---|---|---|---|
| id | UUID | Identificador | `factura:crear` |
| numero | string | | |
| cliente | string | Nombre del cliente | |
| total | decimal (string) | Calculado por el servidor | |
| estado | string | `EMITIDA` | |

```json
{ "data": { "id": "0193a1f3-...", "numero": "FV-000123", "cliente": "Ferretería El Tornillo", "total": "50000.00", "estado": "EMITIDA" } }
```

## 9. Errores
| HTTP | code | kind | Cuándo | Mensaje al usuario | Nivel de log |
|---|---|---|---|---|---|
| 401 | NO_AUTENTICADO | UNAUTHENTICATED | Token ausente o inválido | Tu sesión expiró. Ingresa nuevamente. | WARNING (audit) |
| 403 | ACCESO_DENEGADO | FORBIDDEN | Sin `factura:crear` | No tienes permiso para realizar esta acción. | WARNING (audit) |
| 404 | CLIENTE_NO_ENCONTRADO | NOT_FOUND | RN-2 | El cliente seleccionado no existe o está inactivo. | INFO |
| 404 | PRODUCTO_NO_ENCONTRADO | NOT_FOUND | RN-3 | Uno o más productos no existen o están inactivos. | INFO |
| 409 | FACTURA_DUPLICADA | CONFLICT | RN-1 | Ya existe una factura con ese número. | INFO |
| 413 | CUERPO_DEMASIADO_GRANDE | VALIDATION | Body > 64 KB (Nginx/app) | La solicitud es demasiado grande. | WARNING |
| 422 | VALIDACION_FALLIDA | VALIDATION | Esquema inválido o `Idempotency-Key` ausente | Algunos datos no son válidos. Revisa los campos indicados. | INFO |
| 429 | LIMITE_EXCEDIDO | RATE_LIMITED | > 30 req/min | Demasiadas solicitudes. Espera un momento. | WARNING (audit) |
| 500 | ERROR_INTERNO | INTERNAL | No controlado | Ocurrió un error inesperado. Intenta de nuevo más tarde. | ERROR |

## 10. Eventos y efectos secundarios
| Evento publicado | Momento | Mecanismo | Observers | Efecto |
|---|---|---|---|---|
| FacturaCreada (EV-FACTURA-001) | Después del commit | Bus en memoria | InvalidarCacheFacturas | Borra las listas cacheadas de la empresa |
| | | | NotificarFacturaCreada | Publica `factura.creada` en `empresa:{id}:facturas` |
| FacturaCreada | En la transacción | Outbox | EnviarFacturaElectronicaDIAN (worker) | Envío al proveedor de facturación electrónica con reintentos |

¿Notifica por WebSocket? Sí → WS-FACTURA-001, evento `factura.creada`.

## 11. Logging y auditoría
| Evento de log | Nivel | Campos |
|---|---|---|
| factura_creada | INFO | trace_id, user_id, empresa_id, factura_id, n_items, duration_ms |
| idempotency_replay | INFO | trace_id, user_id, idempotency_key |
| access_denied | WARNING (audit) | trace_id, user_id, permiso, path |

Datos que NO se registran: el body completo, los precios por ítem y los datos del cliente.

## 12. Pruebas
### 12.1 Unitarias
- [ ] Crea la factura, calcula el total y registra `FacturaCreada`
- [ ] RN-1, RN-2 y RN-3 lanzan su error y no insertan nada
- [ ] Sin permiso → `ForbiddenError`, sin llamar al repositorio
- [ ] No publica eventos si el commit falla

### 12.2 Integración y API
- [ ] 201 con el contrato de §8 y la cabecera `Location`
- [ ] Cada error de §9 con su `code` y formato estándar
- [ ] Presupuesto de consultas ≤ 3
- [ ] Carrera: dos requests simultáneos con el mismo número → un 201 y un 409
- [ ] Misma `Idempotency-Key` → misma respuesta y una sola factura

### 12.3 Adversarias
- [ ] Sin token, token expirado, firma alterada, `alg: none` → 401
- [ ] Perfil Consulta (sin `factura:crear`) → 403
- [ ] `cliente_id` y `producto_id` de otra empresa → 404
- [ ] Enviar `empresa_id`, `total` o `estado` → 422
- [ ] `numero` con `' OR 1=1 --`, `<script>`, emoji y caracteres de control → 422
- [ ] `descripcion` con `<img src=x onerror=alert(1)>` → 422 (fuera de la lista blanca)
- [ ] 201 ítems, cantidad 0, precio negativo, precio con 3 decimales → 422
- [ ] Body de 65 KB → 413
- [ ] 31 requests en un minuto → 429

### 12.4 E2E
- [ ] El Cajero crea una factura desde la UI y el Administrador la ve aparecer en su listado en tiempo real
- [ ] El perfil Consulta no ve el botón "Nueva factura" y, si navega a `/facturas/nueva`, termina en `/403`

## 13. Checklist OWASP API Top 10
| Riesgo | Mitigación |
|---|---|
| API1 | Alcance por `empresa_id` en cliente, productos y factura |
| API2 | JWT validado (firma, exp, aud, algoritmo fijo) + revocación |
| API3 | `extra="forbid"`; esquema de salida explícito |
| API4 | Máx. 200 ítems, body ≤ 64 KB, rate limit |
| API5 | `require_permission` + `actor.exigir` |
| API6 | `Idempotency-Key` y rate limit contra creación masiva |
| API7 | N/A: no consume URLs del cliente |
| API8 | Errores sin detalles internos; `no-store` |
| API9 | Endpoint versionado `/v1` y documentado en OpenAPI |
| API10 | Respuesta del proveedor de facturación electrónica validada en su adaptador (worker) |

## 14. Decisiones y pendientes
- El envío a la DIAN es asíncrono vía Outbox (ADR-004).
- Pendiente: definir si se permiten descuentos por ítem (HU-015).
````
