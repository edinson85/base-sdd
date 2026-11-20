---
id: ART-07
titulo: Plantilla ADR y decisiones base del stack
version: 1.1.0   # 1.1.0: ADR-BASE-13 (GitHub)
estado: vigente
aplica_a: [todos]
depende_de: [ART-02]
---

# ART-07 · Architecture Decision Records (ADR)

> **Instrucción para agentes IA**
> - Crea un ADR cuando: se elige o cambia una tecnología, se desvía de un principio de ART-02, se elige entre alternativas con impacto a largo plazo, o se acepta un riesgo de seguridad.
> - Los ADR son inmutables una vez aceptados: para cambiar una decisión se crea un ADR nuevo que **reemplaza** al anterior.
> - Guarda en `docs/adr/ADR-NNN-<titulo-kebab>.md`. Los ADR base de §3 se heredan en todo proyecto (no se copian; se referencian como `ADR-BASE-NN`).

---

## 1. Plantilla

````markdown
---
id: ADR-NNN
titulo: <Decisión en forma de frase>
estado: propuesto | aceptado | rechazado | reemplazado por ADR-XXX
fecha: AAAA-MM-DD
decisores: [<nombres>]
principios_afectados: [P-XX-NN]
---

# ADR-NNN · <Título>

## Contexto
<Situación, fuerzas en juego, restricciones. ¿Por qué hay que decidir ahora?>

## Opciones consideradas
| Opción | Pros | Contras |
|---|---|---|
| A | | |
| B | | |

## Decisión
<La opción elegida, en una o dos frases, en voz activa: "Usaremos X para Y".>

## Consecuencias
- Positivas:
- Negativas / deuda aceptada:
- Acciones requeridas: <tareas, cambios a artefactos>

## Desviación de la constitución (si aplica)
| Principio | Desviación | Mitigación | Vigencia |
|---|---|---|---|
````

---

## 2. Cuándo NO hace falta un ADR

- Decisiones locales y reversibles en un PR (nombre de una función, estructura interna de un componente).
- Aplicar lo que ya dicen los artefactos normativos.

---

## 3. ADR base heredados

| ID | Decisión | Alternativas descartadas | Razón principal |
|---|---|---|---|
| ADR-BASE-01 | **FastAPI** como framework HTTP/WS del backend | Django + DRF, Litestar, Flask | Async nativo, WebSockets integrados, validación con Pydantic, inyección de dependencias simple, OpenAPI automático; encaja con hexagonal porque no impone ORM ni estructura |
| ADR-BASE-02 | **SQLAlchemy 2.x async + asyncpg**, sin repositorios genéricos | Django ORM, SQLModel, SQL crudo | Control fino de las consultas (P-DA), soporte de proyecciones, ventanas y `lazy="raise"` |
| ADR-BASE-03 | **Redis** para caché, rate limiting, pub/sub de WS, tickets WS y revocación | Memcached, PostgreSQL LISTEN/NOTIFY | Un solo servicio cubre todos los usos; estructuras atómicas |
| ADR-BASE-04 | **Taskiq** (broker Redis) para tareas en segundo plano y relay de Outbox | Celery, ARQ, RQ | Async nativo como el resto del backend; Celery queda como alternativa si el proyecto necesita su ecosistema (ADR del proyecto) |
| ADR-BASE-05 | **TanStack Query** para estado del servidor en React; estado local mínimo (Zustand solo si hace falta) | Redux Toolkit, SWR | Caché, invalidación y reintentos listos; facilita invalidar por eventos WS |
| ADR-BASE-06 | **Tailwind CSS + componentes accesibles basados en Radix (estilo shadcn/ui)** para el design system web | MUI, Ant Design, CSS Modules | Componentes copiados al repositorio (control total), accesibles, CSS estático compatible con CSP estricta |
| ADR-BASE-07 | **Riverpod + go_router + dio** en Flutter | Bloc, Provider, GetX | Inyección de dependencias y estado en uno; redirecciones de rutas por permiso |
| ADR-BASE-08 | **JWT de corta duración + refresh rotativo en cookie HttpOnly** (web) y en almacenamiento seguro (móvil) | Sesiones de servidor con cookie | Sirve a web y móvil con el mismo backend; revocación por familia de refresh |
| ADR-BASE-09 | **UUIDv7** como clave primaria, generado en la aplicación | bigserial, UUIDv4 | Sin enumeración de ids (anti-IDOR), ordenable por tiempo (índices eficientes), generado antes de insertar (eventos con id) |
| ADR-BASE-10 | **Playwright** para E2E web y **Schemathesis + OWASP ZAP** para pruebas adversarias automatizadas | Cypress, Selenium | Multi-navegador, paralelo, buen soporte de WebSocket; fuzzing guiado por OpenAPI |
| ADR-BASE-11 | **structlog** para logs JSON | logging estándar con formateador | Contexto (`trace_id`) por contextvars y procesadores de enmascarado |
| ADR-BASE-12 | **k6** para pruebas de carga | Locust, JMeter | Scripts versionables, umbrales como código, soporte WebSocket |
| ADR-BASE-13 | **GitHub + GitHub Actions + GHCR** para repositorios, CI/CD, registro de imágenes y despliegue por SSH | Bitbucket + Pipelines, GitLab | Mayor cupo gratuito de minutos de CI en repositorios privados, integración con Claude (sesiones conectadas a repositorios de GitHub), Dependabot y detección de secretos incluidos. Limitación aceptada: en repositorios privados sin Enterprise no hay aprobación obligatoria de despliegues, así que producción se despliega solo por ejecución manual (ART-36 §3.1) |
