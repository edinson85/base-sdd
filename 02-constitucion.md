---
id: ART-02
titulo: Constitución del proyecto
version: 1.1.0
estado: vigente
aplica_a: [backend, frontend-web, movil, base-de-datos, infraestructura]
depende_de: []
referenciado_por: [todos]
---

# ART-02 · Constitución del proyecto

> **Instrucción para agentes IA**
> Este documento tiene prioridad sobre cualquier otro artefacto, plantilla o especificación de un proyecto concreto.
> 1. Antes de planear, especificar o escribir código, lee este documento completo.
> 2. Cita los principios por su identificador (`P-XX-NN`) en especificaciones, tareas, PRs y pruebas.
> 3. Si una especificación o pedido contradice un principio, **DETENTE y reporta el conflicto**. No lo resuelvas por tu cuenta.
> 4. La única forma válida de desviarse de un principio es un ADR aprobado (ver §13).

---

## 0. Convenciones de lenguaje

| Término | Significado |
|---|---|
| **DEBE** / **NO DEBE** | Obligatorio. Incumplirlo bloquea la aprobación (Definition of Done). |
| **DEBERÍA** / **NO DEBERÍA** | Recomendado. Desviarse requiere justificarlo por escrito en la especificación o el PR. |
| **PUEDE** | Opcional, a criterio del equipo. |

Prefijos de los identificadores:

| Prefijo | Área |
|---|---|
| `P-ST` | Stack |
| `P-AR` | Arquitectura |
| `P-DA` | Datos y rendimiento |
| `P-CA` | Caché |
| `P-ER` | Errores y excepciones |
| `P-LO` | Logging y observabilidad |
| `P-VA` | Validación de entradas |
| `P-AU` | Autenticación y autorización |
| `P-SB` / `P-SF` / `P-SN` / `P-SD` | Seguridad backend / frontend y móvil / Nginx / Docker |
| `P-EV` | Eventos, observers y tiempo real |
| `P-UX` | Experiencia de usuario y alertas |
| `P-PR` | Pruebas |
| `P-SDD` | Proceso |

---

## 1. Stack base

| Capa | Tecnología |
|---|---|
| Base de datos | PostgreSQL 16+ |
| Backend | Python 3.12+, FastAPI, SQLAlchemy 2.x (async) + asyncpg, Pydantic v2, Alembic |
| Caché / mensajería ligera | Redis 7+ (caché, rate limiting, pub/sub para WebSockets, revocación de tokens) |
| Frontend web | React 19 + TypeScript (`strict`), Vite, TanStack Query, React Router, Zod |
| Móvil | Flutter (Dart con null-safety) |
| Proxy inverso | Nginx |
| Contenedores | Docker + Docker Compose |
| Pruebas | pytest, Vitest + Testing Library, flutter_test, Playwright, OWASP ZAP |

- **P-ST-01** Cambiar o agregar una pieza del stack DEBE registrarse en un ADR.
- **P-ST-02** Las versiones de dependencias DEBEN fijarse (lockfiles) y actualizarse de forma controlada.

---

## 2. Arquitectura

- **P-AR-01** Backend, frontend web y móvil DEBEN seguir arquitectura hexagonal (puertos y adaptadores).
- **P-AR-02** El código se organiza **por módulo** (una entidad o agregado del dominio). Cada módulo DEBE tener exactamente tres carpetas de capa:

  ```
  factura/
  ├── domain/          # Entidades, value objects, reglas, eventos, errores, puertos del dominio
  ├── application/     # Casos de uso, DTOs, puertos de salida, manejadores de eventos (observers)
  └── infrastructure/  # Adaptadores: HTTP, WebSocket, BD, caché, UI, clientes externos
  ```

- **P-AR-03** Regla de dependencia: `infrastructure → application → domain`. Nunca al revés.
  `domain` NO DEBE importar frameworks ni librerías de E/S (FastAPI, SQLAlchemy, Pydantic, Redis, React, fetch/axios, widgets de Flutter).
- **P-AR-04** Un módulo NO DEBE importar la capa `infrastructure` de otro módulo. La comunicación entre módulos se hace mediante puertos de `application` o mediante eventos de dominio.
- **P-AR-05** Lo transversal (errores base, sesión de BD, caché, logging, seguridad, bus de eventos, cliente HTTP) vive en `shared/`, que también tiene las tres capas.
- **P-AR-06** Las reglas de negocio viven en `domain` (invariantes) y `application` (orquestación). Los routers, componentes React y widgets NO DEBEN contener reglas de negocio.
- **P-AR-07** Las dependencias se inyectan por puertos (interfaces/protocolos) desde un único punto de composición (*composition root*).
- **P-AR-08** Detalle de implementación: backend en ART-10, frontend web en ART-11, móvil en ART-12.

---

## 3. Datos y rendimiento

- **P-DA-01** Cada endpoint DEBE declarar en su especificación (ART-46) las consultas que ejecuta y un **presupuesto máximo de consultas**. Superarlo es un defecto.
- **P-DA-02** Está prohibido el problema N+1. Se DEBEN usar joins, `selectinload`/`joinedload`, agregaciones en SQL y funciones de ventana (por ejemplo, el total de paginación con `count(*) OVER ()` en la misma consulta).
- **P-DA-03** Las lecturas DEBERÍAN usar proyecciones (solo las columnas necesarias) mediante servicios de consulta dedicados. NO DEBEN cargarse agregados completos para listar.
- **P-DA-04** Toda lista DEBE paginarse con un tamaño máximo (por defecto 100). Para tablas grandes se DEBERÍA usar paginación por cursor (keyset).
- **P-DA-05** Toda columna usada con frecuencia en filtros, joins u ordenamientos DEBE tener un índice justificado en la especificación.
- **P-DA-06** Las escrituras de un caso de uso DEBEN ocurrir en una sola transacción (Unit of Work).
- **P-DA-07** Solo se permite SQL parametrizado. Está prohibido concatenar entradas del usuario en SQL.
- **P-DA-08** Los cambios de esquema DEBEN hacerse solo con migraciones Alembic versionadas y reversibles.
- **P-DA-09** Las operaciones costosas (reportes, importaciones, envíos masivos) NO DEBEN ejecutarse dentro del request; se encolan y se notifica el resultado (ver §10).

---

## 4. Caché

- **P-CA-01** Se DEBE cachear la información de lectura frecuente y cambio poco frecuente: catálogos, permisos por perfil, configuración y consultas costosas. Cada uso de caché DEBE declarar su **clave, TTL y estrategia de invalidación**.
- **P-CA-02** El patrón por defecto es cache-aside. La invalidación se hace con eventos de dominio (observers), con el TTL como red de seguridad.
- **P-CA-03** Las claves siguen la convención `{app}:{modulo}:{recurso}:v{n}:{identificadores}`. Los datos por usuario o por tenant DEBEN incluir ese identificador en la clave.
- **P-CA-04** NO se DEBEN cachear secretos, tokens ni contraseñas. Los datos personales sensibles solo se cachean con un TTL corto y justificado.
- **P-CA-05** Una falla de Redis NO DEBE tumbar la operación: se degrada a la base de datos y se registra un `WARNING`.

---

## 5. Errores y excepciones

- **P-ER-01** Existe una única jerarquía de errores. Cada error tiene:
  - `code` estable en MAYÚSCULAS, por ejemplo `FACTURA_NO_ENCONTRADA`.
  - `kind`: `VALIDATION | NOT_FOUND | CONFLICT | BUSINESS_RULE | UNAUTHENTICATED | FORBIDDEN | RATE_LIMITED | EXTERNAL | INTERNAL`.
  - `message`: mensaje entendible por el usuario final.
- **P-ER-02** Todas las respuestas de error usan el mismo formato:

  ```json
  {
    "error": {
      "code": "FACTURA_NO_ENCONTRADA",
      "message": "La factura solicitada no existe o no tienes acceso a ella.",
      "details": [],
      "trace_id": "01JD3K..."
    }
  }
  ```

- **P-ER-03** El mensaje DEBE explicar qué pasó y, si aplica, qué puede hacer el usuario. NO DEBE exponer stack traces, SQL, rutas de archivos, nombres de tablas ni versiones de software.
- **P-ER-04** Un error no controlado responde 500 con un mensaje genérico y el `trace_id`. El detalle completo va solo a los logs.
- **P-ER-05** El front y el móvil DEBEN traducir `code` a mensajes de interfaz y nunca mostrar respuestas crudas del servidor.
- **P-ER-06** Está prohibido silenciar errores (`except Exception: pass`, un `catch {}` vacío). Todo bloque que captura un error DEBE relanzarlo, traducirlo o registrarlo.

---

## 6. Logging y observabilidad

- **P-LO-01** Los logs son estructurados en JSON y tienen al menos: `timestamp`, `level`, `service`, `module`, `event`, `trace_id`, `user_id` (si existe) y `duration_ms` (si aplica).
- **P-LO-02** Cada request tiene un `trace_id` (cabecera `X-Request-ID`, generado si no viene) que se propaga a los logs, las respuestas de error, los eventos de dominio y los mensajes WebSocket.
- **P-LO-03** Niveles:
  - `ERROR`: 5xx y fallas de infraestructura.
  - `WARNING`: 401, 403, 429, conflictos relevantes y fallas de caché.
  - `INFO`: eventos de negocio e inicio/fin de procesos.
  - `DEBUG`: solo en desarrollo.
- **P-LO-04** NO se DEBEN registrar contraseñas, tokens, cookies, la cabecera `Authorization`, números completos de tarjeta o cuenta, ni documentos de identidad completos. Los datos personales se enmascaran.
- **P-LO-05** Los eventos de seguridad (login fallido, acceso denegado, cambio de permisos, rate limit superado, token reutilizado) DEBEN registrarse en un canal de auditoría (logger `audit`).

---

## 7. Validación de entradas

- **P-VA-01** Toda entrada externa DEBE validarse en el borde con un esquema explícito: body, query, path, cabeceras usadas, archivos, mensajes WebSocket y webhooks.
- **P-VA-02** Cada campo DEBE declarar tipo, obligatoriedad, longitud mínima y máxima, patrón de caracteres permitidos (lista blanca) y rango o valores permitidos.
- **P-VA-03** Los campos no declarados se rechazan (`extra="forbid"`). Está prohibida la asignación masiva: el cliente nunca envía `rol`, `owner_id`, `estado`, `empresa_id`, totales calculados ni similares.
- **P-VA-04** Para archivos se valida: tamaño máximo, tipo MIME verificado por contenido (magic bytes), extensión en lista blanca y nombre regenerado en el servidor. Se almacenan fuera del webroot.
- **P-VA-05** Cada endpoint tiene un tamaño máximo de body, aplicado en Nginx y en la aplicación.
- **P-VA-06** La validación del front es para la experiencia de usuario; la del backend es la autoridad. El dominio vuelve a proteger sus invariantes con value objects.
- **P-VA-07** Los mensajes de error de validación NO DEBEN reflejar el valor recibido; solo indican el campo y la regla incumplida.

---

## 8. Autenticación y autorización

- **P-AU-01** El modelo es RBAC: usuario → perfiles (roles) → permisos con la forma `recurso:accion` (por ejemplo `factura:crear`, `factura:anular`). Los endpoints y las pantallas exigen **permisos**, no nombres de perfiles.
- **P-AU-02** Se deniega por defecto: un endpoint, ruta, pantalla o canal WebSocket sin permiso declarado no se publica.
- **P-AU-03** Además del permiso, todo acceso a un recurso por identificador DEBE verificar su alcance (tenant, empresa o dueño) para evitar IDOR. Si el recurso está fuera del alcance del usuario se responde 404, para no revelar que existe.
- **P-AU-04** Tokens:
  - El access token es de vida corta (máximo 15 minutos).
  - El refresh token es rotativo y revocable, y viaja en una cookie `HttpOnly; Secure; SameSite=Strict`.
  - Si se detecta la reutilización de un refresh token, se revoca toda su familia.
- **P-AU-05** Las contraseñas se guardan con Argon2id. Los intentos fallidos producen bloqueo o ralentización progresiva. El MFA DEBERÍA estar disponible para los perfiles administrativos.
- **P-AU-06** Un cambio de permisos o perfiles DEBE invalidar la caché de permisos y las sesiones afectadas.

---

## 9. Seguridad

> Contexto: las aplicaciones expuestas reciben ataques automatizados constantemente. Toda decisión asume un atacante activo.

### 9.1 Backend (`P-SB`)
- **P-SB-01** El OWASP API Security Top 10 es un checklist obligatorio para cada endpoint (ver ART-46).
- **P-SB-02** Hay rate limiting por IP y por usuario. Las operaciones sensibles (login, recuperación de contraseña, OTP, registro) tienen límites más estrictos.
- **P-SB-03** CORS solo admite una lista blanca explícita de orígenes. Nunca `*` cuando hay credenciales.
- **P-SB-04** Los secretos solo vienen de variables de entorno o de un gestor de secretos. Nunca van en el repositorio, en las imágenes ni en los logs.
- **P-SB-05** Las dependencias se escanean en CI (`pip-audit`, `npm audit`, análisis estático de seguridad).
- **P-SB-06** Si la autenticación usa cookies, los endpoints que cambian estado tienen protección CSRF (SameSite=Strict más un token o un encabezado personalizado).
- **P-SB-07** Las respuestas con datos sensibles llevan `Cache-Control: no-store`.
- **P-SB-08** La documentación OpenAPI/Swagger está deshabilitada o protegida en producción.
- **P-SB-09** Las respuestas solo devuelven los campos que el perfil puede ver (esquemas de salida explícitos, nunca el modelo ORM directo).

### 9.2 Frontend web y móvil (`P-SF`)
- **P-SF-01** Las rutas, pantallas y acciones están protegidas por permiso (guardas). El backend sigue siendo la autoridad.
- **P-SF-02** Los tokens y datos personales NO se guardan en `localStorage`, `sessionStorage` ni IndexedDB. El access token vive solo en memoria. En móvil se usa `flutter_secure_storage`.
- **P-SF-03** Al cerrar sesión (manual, por expiración o por inactividad) se DEBE:
  1. Revocar la sesión en el backend.
  2. Cerrar los WebSockets.
  3. Limpiar la caché de datos (query cache) y el estado en memoria.
  4. Limpiar cualquier almacenamiento local de la app.
  5. Notificar a las otras pestañas.
  6. Reemplazar el historial para que "atrás" no muestre información.
- **P-SF-04** Está prohibido `dangerouslySetInnerHTML` sin sanitizar con DOMPurify.
- **P-SF-05** No hay secretos en el bundle: las variables `VITE_*` son públicas.
- **P-SF-06** En producción no se publican source maps ni se escriben datos con `console.*`.
- **P-SF-07** Las URLs no llevan datos sensibles (tokens, documentos, correos).
- **P-SF-08** Existe un cierre por inactividad configurable por proyecto.

### 9.3 Nginx (`P-SN`) — detalle en ART-34
- **P-SN-01** Solo TLS 1.2/1.3, redirección HTTP → HTTPS y HSTS.
- **P-SN-02** Cabeceras: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `frame-ancestors 'none'`/`X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`.
- **P-SN-03** `server_tokens off`, `client_max_body_size`, timeouts y `limit_req`/`limit_conn` configurados.
- **P-SN-04** Proxy de WebSockets con `Upgrade`/`Connection` solo en las rutas `/ws/`.

### 9.4 Docker (`P-SD`) — detalle en ART-33
- **P-SD-01** Imágenes multi-stage, con base mínima y versión fijada, ejecutadas con usuario no root.
- **P-SD-02** No hay secretos en las capas de la imagen.
- **P-SD-03** Solo Nginx publica puertos (80/443). PostgreSQL y Redis viven en una red interna.
- **P-SD-04** Todos los servicios tienen healthcheck.

---

## 10. Eventos, observers y tiempo real

- **P-EV-01** Los cambios de estado relevantes para el negocio DEBEN registrarse como **eventos de dominio** nombrados en pasado (`FacturaCreada`, `FacturaAnulada`).
- **P-EV-02** Los efectos secundarios (notificaciones, invalidación de caché, auditoría, tiempo real, integraciones) DEBEN implementarse como **observers**, es decir, manejadores suscritos al bus de eventos, y no dentro del caso de uso.
- **P-EV-03** Los eventos se publican **después del commit**. Si el efecto es crítico (no puede perderse) o va a sistemas externos, se usa el patrón **Outbox** (el evento se guarda en la misma transacción).
- **P-EV-04** Los observers DEBEN ser idempotentes. Su falla NO DEBE revertir la operación principal: se registra y se reintenta según la política definida.
- **P-EV-05** Los WebSockets se usan solo cuando el usuario necesita ver cambios en tiempo real: tableros, notificaciones, colas de trabajo, chat y estados de procesos largos. Para todo lo demás se usa HTTP.
- **P-EV-06** Seguridad de WebSockets:
  - Se autentican en el handshake con cookie o con un ticket de un solo uso. Nunca con un token de larga duración en la URL.
  - Se valida el `Origin` y se autoriza la suscripción a cada canal.
  - Cada mensaje entrante se valida con un esquema y tiene límites de tamaño y frecuencia.
  - La conexión se cierra cuando la sesión termina o expira.
- **P-EV-07** Con más de una instancia de backend, la difusión DEBE pasar por Redis pub/sub.
- **P-EV-08** Un mensaje WebSocket NO DEBE incluir más datos de los que el destinatario tiene permiso de ver. Para datos sensibles se notifica solo "cambió X" y el cliente consulta por HTTP.

---

## 11. Experiencia de usuario y alertas (`P-UX`)

- **P-UX-01** Toda retroalimentación al usuario final (éxito, información, advertencia y error) DEBE mostrarse con el **sistema único de alertas** del proyecto (ART-29). Está prohibido usar `alert()`, `confirm()` o `prompt()` del navegador, o mensajes improvisados por pantalla.
- **P-UX-02** Las alertas son **flotantes**: se superponen al contenido sin desplazarlo. Se ubican en la esquina superior derecha en escritorio y en la parte superior en pantallas pequeñas y móvil, y se apilan.
- **P-UX-03** Cada alerta DEBE poder cerrarse con un botón **X en su esquina superior derecha** y DEBE cerrarse sola al cumplirse su duración. La única excepción son las alertas marcadas como persistentes (errores que exigen una acción del usuario), que se cierran solo con la X o con la acción.
- **P-UX-04** Las alertas que se cierran solas DEBEN mostrar una **barra sutil en la parte inferior** que se acorta de forma continua según el tiempo restante. El tiempo (y la barra) se pausa mientras el puntero está sobre la alerta, mientras tiene el foco del teclado y mientras la pestaña o la app no está visible.
- **P-UX-05** Las alertas DEBEN ser accesibles: se anuncian a lectores de pantalla (`aria-live`), la X tiene nombre accesible ("Cerrar alerta") y se cierran con `Esc` cuando tienen el foco (ART-32).
- **P-UX-06** La interfaz de usuario DEBE proponerse, revisarse y aprobarse con prototipos (ART-49) antes de especificar pantallas (ART-47) y antes de implementar.

---

## 12. Pruebas

- **P-PR-01** Las pruebas unitarias son obligatorias para `domain` y `application` (back, front y móvil). Usan puertos simulados (fakes), sin BD ni red. La cobertura mínima de esas capas es 80 %.
- **P-PR-02** Hay pruebas de integración para los adaptadores de infraestructura: repositorios contra un PostgreSQL real en contenedor y caché contra Redis.
- **P-PR-03** Cada endpoint tiene una prueba que verifica su presupuesto de consultas (P-DA-01).
- **P-PR-04** Hay pruebas end to end de los flujos críticos para cada perfil (Playwright en web, `integration_test` en Flutter).
- **P-PR-05** Las **pruebas adversarias** son obligatorias para cada endpoint, pantalla y canal:
  - Entradas maliciosas: inyección SQL, XSS, payloads gigantes, tipos incorrectos, Unicode problemático y campos extra (asignación masiva).
  - Acceso sin token, con token expirado o manipulado, y con un perfil sin permiso.
  - IDOR: acceso a recursos de otro usuario o tenant.
  - Abuso de rate limit.
  - WebSocket: conexión sin autenticación, suscripción a un canal ajeno y mensajes malformados o excesivos.
- **P-PR-06** Cada bug corregido incluye una prueba que lo reproduce.
- **P-PR-07** No se hace merge con pruebas fallando ni por debajo de la cobertura mínima.

---

## 13. Proceso SDD (`P-SDD`)

- **P-SDD-01** El flujo es: `brief → especificación (HU + criterios de aceptación) → propuesta de interfaz y análisis funcional (prototipos) → plan técnico (módulos, modelo de datos, eventos) → contratos (endpoints ART-46, pantallas ART-47) → planeación y solicitud de insumos → implementación → verificación` (detalle en ART-01).
- **P-SDD-02** No se escribe código sin una especificación aprobada.
- **P-SDD-03** Cada tarea referencia su historia de usuario y los principios que aplican.
- **P-SDD-04** La especificación es la fuente de verdad. Si el código y la especificación difieren, se corrige uno de los dos de forma explícita.
- **P-SDD-05** Toda desviación de esta constitución requiere un ADR con: contexto, decisión, alternativas, consecuencias y principios afectados.
- **P-SDD-06** Los hallazgos del análisis funcional sobre los prototipos (funcionalidades faltantes, sobrantes o mal definidas) DEBEN reflejarse en la especificación (nueva versión aprobada) antes del plan técnico (ART-49).
- **P-SDD-07** Antes de iniciar el desarrollo se DEBEN solicitar y verificar los **insumos** necesarios: accesos, cuentas, credenciales de prueba, ambientes, datos, documentación de terceros, activos de marca y decisiones del negocio (ART-50). Ninguna tarea inicia si sus insumos no están en estado "verificado", y el desarrollo NO DEBE avanzar sobre supuestos de cómo será un insumo.
- **P-SDD-08** Las credenciales y secretos que se entregan como insumos NO DEBEN escribirse en documentos, tickets, chats ni prompts de IA. Se entregan por el gestor de secretos acordado (ART-35) y los documentos solo registran dónde están y quién los custodia.

---

## 14. Definition of Done mínima

- [ ] Cumple la especificación y sus criterios de aceptación.
- [ ] Respeta la estructura hexagonal por módulo y la regla de dependencia (P-AR-02, P-AR-03).
- [ ] Las entradas se validan en el borde (P-VA) y los permisos y alcance se verifican (P-AU-01, P-AU-03).
- [ ] Los errores usan el catálogo y el formato estándar (P-ER) y se registran correctamente (P-LO).
- [ ] Se cumple el presupuesto de consultas, con índices declarados (P-DA).
- [ ] La caché tiene clave, TTL e invalidación declaradas, si aplica (P-CA).
- [ ] Los eventos y observers están documentados y son idempotentes, si aplica (P-EV).
- [ ] Hay pruebas unitarias, de integración, adversarias y E2E según corresponda, todas en verde (P-PR).
- [ ] Sin secretos, sin datos sensibles en logs y con dependencias escaneadas (P-SB, P-LO-04).
- [ ] La interfaz corresponde al prototipo aprobado y toda retroalimentación usa el sistema de alertas (P-UX).
- [ ] Documentación (OpenAPI y especificación) actualizada.

---

## 15. Versionado de este documento

Se usa SemVer:
- **MAJOR**: se elimina un principio o se cambia de forma incompatible.
- **MINOR**: se agrega un principio.
- **PATCH**: aclaraciones de redacción.

| Versión | Fecha | Cambio |
|---|---|---|
| 1.0.0 | 2026-11-20 | Versión inicial |
| 1.1.0 | 2026-11-20 | Nueva §11 (P-UX: alertas flotantes y prototipos); P-SDD-06 a P-SDD-08 (análisis funcional, insumos previos, manejo de credenciales); flujo SDD actualizado; secciones 11-14 renumeradas a 12-15 |
