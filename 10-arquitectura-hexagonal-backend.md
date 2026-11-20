---
id: ART-10
titulo: Arquitectura hexagonal – Backend Python
version: 1.0.1   # 1.0.1: ids UUIDv7 con new_id() (ADR-BASE-09) y dependencias completas
estado: vigente
aplica_a: [backend]
depende_de: [ART-02]
referenciado_por: [ART-46]
stack: [Python 3.12+, FastAPI, SQLAlchemy 2.x async, asyncpg, Pydantic v2, Alembic, Redis, PostgreSQL]
---

# ART-10 · Arquitectura hexagonal – Backend Python

> **Instrucción para agentes IA**
> - Usa este documento como molde obligatorio para crear cualquier módulo del backend.
> - Los nombres de archivos y carpetas de §2 son la convención: reemplaza `factura` por el módulo real.
> - Los fragmentos de código son **patrones de referencia**: adáptalos al dominio sin cambiar la capa donde viven.
> - Antes de entregar un módulo, verifica el checklist de §14.

---

## 1. Principios aplicados

| Principio | Cómo se aplica aquí |
|---|---|
| P-AR-02 | Cada módulo tiene `domain/`, `application/` e `infrastructure/` |
| P-AR-03 | `domain` es Python puro: sin FastAPI, SQLAlchemy, Pydantic ni Redis |
| P-AR-04 | Los módulos se comunican por puertos de `application` o por eventos |
| P-DA-* | Escrituras con repositorio + Unit of Work; lecturas con servicios de consulta optimizados |
| P-CA-* | Caché como decorador del servicio de consulta; invalidación con observers |
| P-ER-* | Jerarquía `AppError` con `code` y `kind`; manejador global en infraestructura |
| P-EV-* | Eventos de dominio publicados después del commit; observers; Outbox; WebSockets |

---

## 2. Estructura de carpetas

```
backend/
├── app/
│   ├── main.py                       # Crea la app FastAPI, monta routers, middlewares y handlers
│   ├── bootstrap/
│   │   ├── settings.py               # Configuración (pydantic-settings) leída de variables de entorno
│   │   ├── container.py              # Composition root: construye adaptadores y casos de uso
│   │   ├── routers.py                # Registro de routers HTTP y WebSocket de cada módulo
│   │   └── subscriptions.py          # Registro de observers de todos los módulos en el EventBus
│   │
│   ├── shared/                       # Transversal, con las mismas 3 capas
│   │   ├── domain/
│   │   │   ├── entity.py             # AggregateRoot (registro de eventos)
│   │   │   ├── events.py             # DomainEvent base
│   │   │   ├── errors.py             # AppError, ErrorKind y errores genéricos
│   │   │   └── value_objects.py      # Email, Money, Documento, etc.
│   │   ├── application/
│   │   │   ├── ports/
│   │   │   │   ├── unit_of_work.py
│   │   │   │   ├── event_bus.py
│   │   │   │   ├── cache.py
│   │   │   │   ├── realtime.py       # Puerto para notificar en tiempo real
│   │   │   │   └── clock.py
│   │   │   ├── actor.py              # Usuario autenticado: id, tenant, permisos
│   │   │   └── pagination.py         # PageRequest / Page
│   │   └── infrastructure/
│   │       ├── db/                   # Engine, session, base declarativa, UoW SQLAlchemy
│   │       ├── cache/                # RedisCache (implementa CachePort)
│   │       ├── events/               # InMemoryEventBus, Outbox (modelo + relay)
│   │       ├── realtime/             # ConnectionManager WS + Redis pub/sub
│   │       ├── http/                 # Middlewares, exception handlers, formato de respuesta
│   │       ├── security/             # JWT, Argon2, dependencias require_permission, rate limit
│   │       └── logging/              # Configuración de logs JSON, trace_id, enmascarado
│   │
│   ├── auth/                         # Módulo de autenticación, perfiles y permisos
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   └── factura/                      # ← MÓDULO DE EJEMPLO (uno por entidad/agregado)
│       ├── domain/
│       │   ├── factura.py            # Agregado Factura (entidad raíz) + entidades hijas
│       │   ├── value_objects.py      # NumeroFactura, EstadoFactura...
│       │   ├── events.py             # FacturaCreada, FacturaAnulada...
│       │   ├── errors.py             # FacturaNoEncontrada, FacturaYaAnulada...
│       │   └── repository.py         # Puerto FacturaRepository (Protocol)
│       ├── application/
│       │   ├── commands.py           # DTOs de entrada de casos de uso (dataclasses)
│       │   ├── results.py            # DTOs de salida / read models
│       │   ├── ports.py              # Puertos de salida propios: FacturaQueryService, etc.
│       │   ├── cache_keys.py         # Convención de claves de caché del módulo (lógica pura)
│       │   ├── use_cases/
│       │   │   ├── crear_factura.py
│       │   │   ├── anular_factura.py
│       │   │   ├── obtener_factura.py
│       │   │   └── listar_facturas.py
│       │   └── event_handlers/       # OBSERVERS del módulo
│       │       ├── invalidar_cache_facturas.py
│       │       └── notificar_factura_tiempo_real.py
│       └── infrastructure/
│           ├── http/
│           │   ├── router.py         # Endpoints REST (adaptador de entrada)
│           │   ├── schemas.py        # Pydantic: request/response con validaciones
│           │   └── dependencies.py   # Construcción de casos de uso por request
│           ├── websocket/
│           │   └── router.py         # Endpoint WS del módulo (si aplica)
│           ├── persistence/
│           │   ├── models.py         # Modelos SQLAlchemy (tablas)
│           │   ├── mappers.py        # Modelo ORM ⇄ entidad de dominio
│           │   ├── repository.py     # SqlAlchemyFacturaRepository
│           │   └── query_service.py  # Consultas de lectura optimizadas
│           ├── cache/
│           │   └── cached_query_service.py  # Decorador con caché del query service
│           └── events/
│               └── subscriptions.py  # Qué observers escuchan qué eventos
│
├── migrations/                       # Alembic
├── tests/
│   ├── unit/factura/{domain,application}/
│   ├── integration/factura/          # Repositorios, query services, caché (BD real en contenedor)
│   ├── api/factura/                  # Endpoints con TestClient: contrato, permisos, presupuesto de consultas
│   └── adversarial/factura/          # Pruebas de ataque (P-PR-05)
├── pyproject.toml
└── Dockerfile
```

### 2.1 Qué va en cada capa

| Capa | Contiene | NO contiene |
|---|---|---|
| `domain` | Entidades, agregados, value objects, invariantes, eventos de dominio, errores de negocio, puertos de repositorio | Frameworks, ORM, Pydantic, HTTP, SQL, logs, fechas del sistema (se inyecta `Clock`) |
| `application` | Casos de uso (orquestación), commands/results, puertos de salida, verificación de permisos y alcance, observers | Detalles de HTTP, SQL, Redis o JSON; reglas de negocio que pertenecen al dominio |
| `infrastructure` | Routers, esquemas Pydantic, modelos ORM, repositorios, query services, caché, WebSocket, clientes externos | Reglas de negocio |

### 2.2 Reglas de importación (verificables con `import-linter`)

```ini
# .importlinter
[importlinter]
root_package = app

[importlinter:contract:capas]
name = Capas hexagonales por módulo
type = layers
containers =
    app.factura
    app.auth
layers =
    infrastructure
    application
    domain

[importlinter:contract:dominio-puro]
name = El dominio no importa frameworks
type = forbidden
source_modules =
    app.factura.domain
    app.auth.domain
    app.shared.domain
forbidden_modules =
    fastapi
    sqlalchemy
    pydantic
    redis

[importlinter:contract:aislamiento-modulos]
name = Un módulo no importa la infraestructura de otro
type = independence
modules =
    app.factura.infrastructure
    app.auth.infrastructure
```

El contrato de capas y el de dominio puro van en CI. El de independencia entre módulos se amplía con cada módulo nuevo.

---

## 3. Capa de dominio

### 3.1 Base de agregados y eventos (`shared/domain`)

```python
# shared/domain/events.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
from uuid import UUID, uuid4


@dataclass(frozen=True, kw_only=True)
class DomainEvent:
    event_id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    @property
    def name(self) -> str:
        return type(self).__name__
```

```python
# shared/domain/entity.py
from dataclasses import dataclass, field

from app.shared.domain.events import DomainEvent


@dataclass(kw_only=True)
class AggregateRoot:
    _events: list[DomainEvent] = field(default_factory=list, init=False, repr=False, compare=False)

    def _record(self, event: DomainEvent) -> None:
        self._events.append(event)

    def pull_events(self) -> list[DomainEvent]:
        events, self._events = self._events, []
        return events
```

```python
# shared/domain/ids.py — único punto de generación de identificadores (ADR-BASE-09)
from uuid import UUID

from uuid_utils.compat import uuid7   # En Python 3.14+: from uuid import uuid7


def new_id() -> UUID:
    return uuid7()
```

### 3.2 Errores (`shared/domain/errors.py`)

```python
from enum import StrEnum
from typing import Any


class ErrorKind(StrEnum):
    VALIDATION = "VALIDATION"
    NOT_FOUND = "NOT_FOUND"
    CONFLICT = "CONFLICT"
    BUSINESS_RULE = "BUSINESS_RULE"
    UNAUTHENTICATED = "UNAUTHENTICATED"
    FORBIDDEN = "FORBIDDEN"
    RATE_LIMITED = "RATE_LIMITED"
    EXTERNAL = "EXTERNAL"
    INTERNAL = "INTERNAL"


class AppError(Exception):
    kind: ErrorKind = ErrorKind.INTERNAL
    code: str = "ERROR_INTERNO"
    message: str = "Ocurrió un error inesperado. Intenta de nuevo más tarde."

    def __init__(self, message: str | None = None, *, details: list[dict[str, Any]] | None = None,
                 log_context: dict[str, Any] | None = None) -> None:
        self.message = message or self.message
        self.details = details or []            # Se envía al cliente: sin datos sensibles
        self.log_context = log_context or {}    # Solo para logs: nunca se envía al cliente
        super().__init__(self.message)


class NotFoundError(AppError):
    kind = ErrorKind.NOT_FOUND
    code = "RECURSO_NO_ENCONTRADO"
    message = "El recurso solicitado no existe o no tienes acceso a él."


class ConflictError(AppError):
    kind = ErrorKind.CONFLICT
    code = "CONFLICTO"


class BusinessRuleError(AppError):
    kind = ErrorKind.BUSINESS_RULE
    code = "REGLA_DE_NEGOCIO"


class ForbiddenError(AppError):
    kind = ErrorKind.FORBIDDEN
    code = "ACCESO_DENEGADO"
    message = "No tienes permiso para realizar esta acción."
```

### 3.3 Agregado del módulo (`factura/domain`)

```python
# factura/domain/events.py
from dataclasses import dataclass
from decimal import Decimal
from uuid import UUID

from app.shared.domain.events import DomainEvent


@dataclass(frozen=True, kw_only=True)
class FacturaCreada(DomainEvent):
    factura_id: UUID
    empresa_id: UUID
    total: Decimal


@dataclass(frozen=True, kw_only=True)
class FacturaAnulada(DomainEvent):
    factura_id: UUID
    empresa_id: UUID
    motivo: str
```

```python
# factura/domain/errors.py
from app.shared.domain.errors import BusinessRuleError, ConflictError, NotFoundError


class FacturaNoEncontrada(NotFoundError):
    code = "FACTURA_NO_ENCONTRADA"
    message = "La factura no existe o no tienes acceso a ella."


class FacturaDuplicada(ConflictError):
    code = "FACTURA_DUPLICADA"
    message = "Ya existe una factura con ese número."


class FacturaYaAnulada(BusinessRuleError):
    code = "FACTURA_YA_ANULADA"
    message = "La factura ya fue anulada y no puede modificarse."
```

```python
# factura/domain/factura.py
from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal
from uuid import UUID

from app.factura.domain.errors import FacturaYaAnulada
from app.factura.domain.events import FacturaAnulada, FacturaCreada
from app.factura.domain.value_objects import EstadoFactura, NumeroFactura
from app.shared.domain.entity import AggregateRoot
from app.shared.domain.ids import new_id      # UUIDv7 (ADR-BASE-09)


@dataclass(kw_only=True)
class Factura(AggregateRoot):
    id: UUID
    empresa_id: UUID
    numero: NumeroFactura
    cliente_id: UUID
    total: Decimal
    estado: EstadoFactura
    creada_en: datetime
    items: list["ItemFactura"] = field(default_factory=list)

    @classmethod
    def crear(cls, *, empresa_id: UUID, numero: NumeroFactura, cliente_id: UUID,
              items: list["ItemFactura"], ahora: datetime) -> "Factura":
        total = sum((i.subtotal for i in items), Decimal("0"))
        factura = cls(id=new_id(), empresa_id=empresa_id, numero=numero, cliente_id=cliente_id,
                      total=total, estado=EstadoFactura.EMITIDA, creada_en=ahora, items=items)
        factura._record(FacturaCreada(factura_id=factura.id, empresa_id=empresa_id, total=total))
        return factura

    def anular(self, motivo: str) -> None:
        if self.estado is EstadoFactura.ANULADA:
            raise FacturaYaAnulada()
        self.estado = EstadoFactura.ANULADA
        self._record(FacturaAnulada(factura_id=self.id, empresa_id=self.empresa_id, motivo=motivo))
```

### 3.4 Puerto del repositorio (`factura/domain/repository.py`)

```python
from typing import Protocol
from uuid import UUID

from app.factura.domain.factura import Factura
from app.factura.domain.value_objects import NumeroFactura


class FacturaRepository(Protocol):
    async def obtener(self, factura_id: UUID, empresa_id: UUID) -> Factura | None: ...
    async def existe_numero(self, numero: NumeroFactura, empresa_id: UUID) -> bool: ...
    async def agregar(self, factura: Factura) -> None: ...
    async def guardar(self, factura: Factura) -> None: ...
```

> El repositorio **siempre recibe el alcance** (`empresa_id`/tenant) para que sea imposible olvidarlo (P-AU-03).

---

## 4. Capa de aplicación

### 4.1 Actor y permisos (`shared/application/actor.py`)

```python
from dataclasses import dataclass
from uuid import UUID

from app.shared.domain.errors import ForbiddenError


@dataclass(frozen=True, kw_only=True)
class Actor:
    user_id: UUID
    empresa_id: UUID
    permisos: frozenset[str]
    trace_id: str

    def puede(self, permiso: str) -> bool:
        return permiso in self.permisos

    def exigir(self, permiso: str) -> None:
        if not self.puede(permiso):
            raise ForbiddenError(log_context={"permiso": permiso, "user_id": str(self.user_id)})
```

El permiso se exige **dos veces**: en el router (`require_permission`, primera barrera) y en el caso de uso (`actor.exigir`, defensa en profundidad para cuando el caso de uso se invoca desde otro adaptador, como un worker o un WebSocket).

### 4.2 Puertos transversales

```python
# shared/application/ports/unit_of_work.py
from typing import Protocol, Self


class UnitOfWork(Protocol):
    async def __aenter__(self) -> Self: ...
    async def __aexit__(self, exc_type, exc, tb) -> None: ...   # rollback si no hubo commit
    async def commit(self) -> None: ...


# shared/application/ports/event_bus.py
from collections.abc import Awaitable, Callable, Sequence
from typing import Protocol

from app.shared.domain.events import DomainEvent

EventHandler = Callable[[DomainEvent], Awaitable[None]]


class EventBus(Protocol):
    def subscribe(self, event_type: type[DomainEvent], handler: EventHandler) -> None: ...
    async def publish(self, events: Sequence[DomainEvent]) -> None: ...


# shared/application/ports/cache.py
from typing import Protocol


class CachePort(Protocol):
    async def get(self, key: str) -> bytes | None: ...
    async def set(self, key: str, value: bytes, ttl_seconds: int) -> None: ...
    async def delete(self, *keys: str) -> None: ...
    async def delete_pattern(self, pattern: str) -> None: ...   # Implementar con SCAN, nunca KEYS


# shared/application/ports/realtime.py
from typing import Any, Protocol


class RealtimeNotifier(Protocol):
    async def publish(self, channel: str, event: str, data: dict[str, Any]) -> None: ...
```

### 4.3 Caso de uso de escritura

```python
# factura/application/use_cases/crear_factura.py
from app.factura.application.commands import CrearFacturaCommand
from app.factura.application.results import FacturaCreadaResult
from app.factura.domain.errors import FacturaDuplicada
from app.factura.domain.factura import Factura
from app.factura.domain.repository import FacturaRepository
from app.shared.application.actor import Actor
from app.shared.application.ports.clock import Clock
from app.shared.application.ports.event_bus import EventBus
from app.shared.application.ports.unit_of_work import UnitOfWork


class CrearFactura:
    PERMISO = "factura:crear"

    def __init__(self, uow: UnitOfWork, facturas: FacturaRepository,
                 event_bus: EventBus, clock: Clock) -> None:
        self._uow = uow
        self._facturas = facturas
        self._event_bus = event_bus
        self._clock = clock

    async def execute(self, cmd: CrearFacturaCommand, actor: Actor) -> FacturaCreadaResult:
        actor.exigir(self.PERMISO)

        async with self._uow:                                   # Una transacción (P-DA-06)
            if await self._facturas.existe_numero(cmd.numero, actor.empresa_id):
                raise FacturaDuplicada()
            factura = Factura.crear(empresa_id=actor.empresa_id, numero=cmd.numero,
                                    cliente_id=cmd.cliente_id, items=cmd.items,
                                    ahora=self._clock.now())
            await self._facturas.agregar(factura)
            await self._uow.commit()

        await self._event_bus.publish(factura.pull_events())   # Después del commit (P-EV-03)
        return FacturaCreadaResult.from_entity(factura)
```

> **Regla:** el caso de uso **no** envía correos, no invalida caché ni notifica por WebSocket. Eso lo hacen los observers (P-EV-02).

### 4.4 Lecturas: servicio de consulta (minimizar consultas)

Las lecturas **no** pasan por el repositorio del agregado. Usan un puerto de consulta que devuelve *read models* listos para la respuesta, en el menor número de consultas posible.

```python
# factura/application/ports.py
from typing import Protocol
from uuid import UUID

from app.factura.application.results import FacturaDetalle, FacturaResumen, FiltroFacturas
from app.shared.application.pagination import Page, PageRequest


class FacturaQueryService(Protocol):
    async def listar(self, empresa_id: UUID, filtro: FiltroFacturas, page: PageRequest) -> Page[FacturaResumen]: ...
    async def detalle(self, factura_id: UUID, empresa_id: UUID) -> FacturaDetalle | None: ...
```

```python
# factura/application/use_cases/obtener_factura.py
class ObtenerFactura:
    PERMISO = "factura:ver"

    def __init__(self, consultas: FacturaQueryService) -> None:
        self._consultas = consultas

    async def execute(self, factura_id: UUID, actor: Actor) -> FacturaDetalle:
        actor.exigir(self.PERMISO)
        detalle = await self._consultas.detalle(factura_id, actor.empresa_id)
        if detalle is None:
            raise FacturaNoEncontrada()          # 404 también si es de otra empresa (P-AU-03)
        return detalle
```

### 4.5 Observers (manejadores de eventos)

```python
# factura/application/event_handlers/invalidar_cache_facturas.py
from app.factura.domain.events import FacturaAnulada, FacturaCreada
from app.factura.application.cache_keys import FacturaCacheKeys   # Lógica pura: vive en application
from app.shared.application.ports.cache import CachePort


class InvalidarCacheFacturas:
    def __init__(self, cache: CachePort) -> None:
        self._cache = cache

    async def __call__(self, event: FacturaCreada | FacturaAnulada) -> None:
        await self._cache.delete_pattern(FacturaCacheKeys.listas_empresa(event.empresa_id))
        await self._cache.delete(FacturaCacheKeys.detalle(event.factura_id))
```

```python
# factura/application/event_handlers/notificar_factura_tiempo_real.py
from app.factura.domain.events import FacturaCreada
from app.shared.application.ports.realtime import RealtimeNotifier


class NotificarFacturaCreada:
    def __init__(self, realtime: RealtimeNotifier) -> None:
        self._realtime = realtime

    async def __call__(self, event: FacturaCreada) -> None:
        # Solo identificadores, no datos sensibles (P-EV-08)
        await self._realtime.publish(
            channel=f"empresa:{event.empresa_id}:facturas",
            event="factura.creada",
            data={"factura_id": str(event.factura_id), "event_id": str(event.event_id)},
        )
```

Requisitos de todo observer:
1. **Idempotente:** se puede ejecutar dos veces con el mismo `event_id` sin efectos dobles (usar `event_id` como clave de deduplicación cuando el efecto no es naturalmente idempotente).
2. **Aislado:** si falla, no afecta al caso de uso ni a otros observers (el bus captura y registra).
3. **Rápido:** si tarda más de unos cientos de ms o llama a sistemas externos, se ejecuta en un worker por medio del Outbox.

---

## 5. Eventos: bus en memoria vs. Outbox

| Situación | Mecanismo |
|---|---|
| Efecto interno, tolerante a pérdida ocasional (invalidar caché, notificación WS) | `InMemoryEventBus` después del commit |
| Efecto crítico o externo (correo, facturación electrónica, webhooks, otro servicio) | **Outbox**: el evento se inserta en la tabla `outbox_events` en la misma transacción; un *relay* lo publica y reintenta |
| Proceso largo (reporte, importación) | Outbox + worker; el progreso se notifica por WebSocket |

### 5.1 Bus en memoria (`shared/infrastructure/events/in_memory_bus.py`)

```python
from collections import defaultdict
from collections.abc import Sequence

import structlog

from app.shared.application.ports.event_bus import EventHandler
from app.shared.domain.events import DomainEvent

log = structlog.get_logger("events")


class InMemoryEventBus:
    def __init__(self) -> None:
        self._subscribers: dict[type[DomainEvent], list[EventHandler]] = defaultdict(list)

    def subscribe(self, event_type: type[DomainEvent], handler: EventHandler) -> None:
        self._subscribers[event_type].append(handler)

    async def publish(self, events: Sequence[DomainEvent]) -> None:
        for event in events:
            for handler in self._subscribers.get(type(event), []):
                try:
                    await handler(event)
                except Exception:
                    log.exception("event_handler_failed", event_name=event.name,
                                  event_id=str(event.event_id),
                                  handler=type(handler).__qualname__)
```

### 5.2 Registro de suscripciones (`factura/infrastructure/events/subscriptions.py`)

```python
def register(bus: EventBus, c: "Container") -> None:
    invalidar = InvalidarCacheFacturas(c.cache)
    bus.subscribe(FacturaCreada, invalidar)
    bus.subscribe(FacturaAnulada, invalidar)
    bus.subscribe(FacturaCreada, NotificarFacturaCreada(c.realtime))
```

`bootstrap/subscriptions.py` llama al `register` de cada módulo al arrancar la aplicación.

### 5.3 Outbox (resumen)

- Tabla `outbox_events(id uuid pk, event_name text, payload jsonb, occurred_at timestamptz, published_at timestamptz null, attempts int, last_error text)` con un índice parcial `WHERE published_at IS NULL`.
- En el caso de uso: `await outbox.add(factura.pull_events())` **antes** de `uow.commit()`.
- Relay: proceso aparte que lee lotes con `SELECT ... FOR UPDATE SKIP LOCKED`, publica, marca `published_at` y reintenta con backoff exponencial hasta un máximo (luego alerta).

---

## 6. Capa de infraestructura

### 6.1 Esquemas de entrada y salida (validación, P-VA)

```python
# factura/infrastructure/http/schemas.py
from decimal import Decimal
from typing import Annotated
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field, StringConstraints

NumeroFacturaStr = Annotated[str, StringConstraints(
    strip_whitespace=True, min_length=3, max_length=20, pattern=r"^[A-Z0-9\-]+$")]
DescripcionStr = Annotated[str, StringConstraints(
    strip_whitespace=True, min_length=1, max_length=200,
    pattern=r"^[\w\sáéíóúÁÉÍÓÚñÑüÜ.,\-()/#]+$")]


class StrictModel(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)   # P-VA-03


class ItemIn(StrictModel):
    producto_id: UUID
    descripcion: DescripcionStr
    cantidad: Annotated[int, Field(ge=1, le=10_000)]
    precio_unitario: Annotated[Decimal, Field(gt=0, le=Decimal("999999999.99"), max_digits=12, decimal_places=2)]


class CrearFacturaIn(StrictModel):
    numero: NumeroFacturaStr
    cliente_id: UUID
    items: Annotated[list[ItemIn], Field(min_length=1, max_length=200)]
    # Nunca: empresa_id, estado, total (los define el servidor)


class FacturaOut(BaseModel):            # Esquema de salida explícito (P-SB-09)
    id: UUID
    numero: str
    cliente: str
    total: Decimal
    estado: str
```

### 6.2 Router (adaptador de entrada HTTP)

```python
# factura/infrastructure/http/router.py
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, status

from app.shared.infrastructure.security.dependencies import require_permission

router = APIRouter(prefix="/api/v1/facturas", tags=["facturas"])


@router.post("", status_code=status.HTTP_201_CREATED, response_model=DataResponse[FacturaOut])
async def crear_factura(
    body: CrearFacturaIn,
    actor: Annotated[Actor, Depends(require_permission("factura:crear"))],
    use_case: Annotated[CrearFactura, Depends(get_crear_factura)],
):
    result = await use_case.execute(body.to_command(), actor)
    return DataResponse(data=FacturaOut.model_validate(result, from_attributes=True))


@router.get("/{factura_id}", response_model=DataResponse[FacturaOut])
async def obtener_factura(
    factura_id: UUID,
    actor: Annotated[Actor, Depends(require_permission("factura:ver"))],
    use_case: Annotated[ObtenerFactura, Depends(get_obtener_factura)],
):
    return DataResponse(data=await use_case.execute(factura_id, actor))
```

El router **solo** traduce HTTP ⇄ caso de uso. Sin `if` de negocio ni consultas.

### 6.3 Formato de respuesta exitosa

```json
{ "data": { ... }, "meta": { "page": 1, "size": 20, "total": 134 } }
```

`meta` solo aparece en listas. Para la paginación por cursor: `"meta": { "next_cursor": "...", "size": 20 }`.

### 6.4 Servicio de consulta optimizado (P-DA)

```python
# factura/infrastructure/persistence/query_service.py
from sqlalchemy import func, select


class SqlFacturaQueryService:
    def __init__(self, session: AsyncSession) -> None:
        self._s = session

    async def listar(self, empresa_id, filtro, page) -> Page[FacturaResumen]:
        items_count = (select(ItemModel.factura_id, func.count().label("n_items"))
                       .group_by(ItemModel.factura_id).subquery())
        stmt = (
            select(
                FacturaModel.id, FacturaModel.numero, FacturaModel.total, FacturaModel.estado,
                ClienteModel.nombre.label("cliente"),
                func.coalesce(items_count.c.n_items, 0).label("n_items"),
                func.count().over().label("total_registros"),       # Total en la misma consulta
            )
            .join(ClienteModel, ClienteModel.id == FacturaModel.cliente_id)
            .outerjoin(items_count, items_count.c.factura_id == FacturaModel.id)
            .where(FacturaModel.empresa_id == empresa_id)            # Alcance obligatorio
            .where(*filtro.to_sql_conditions())
            .order_by(FacturaModel.creada_en.desc(), FacturaModel.id.desc())
            .limit(page.size).offset(page.offset)
        )
        rows = (await self._s.execute(stmt)).mappings().all()        # 1 consulta en total
        total = rows[0]["total_registros"] if rows else 0
        return Page(items=[FacturaResumen.from_row(r) for r in rows], total=total,
                    page=page.number, size=page.size)
```

Reglas:
- Detalle con hijos: `selectinload` (2 consultas fijas) o JSON agregado en SQL (`json_agg`, 1 consulta). Nunca un loop de consultas.
- Validar varias referencias (por ejemplo, que todos los `producto_id` existan): **una** consulta con `WHERE id = ANY(:ids)` y comparar conjuntos.
- Escrituras masivas: `insert().values([...])` o `executemany`, no un loop de `session.add` + `flush`.
- Configurar `lazy="raise"` en las relaciones ORM para que una carga perezosa accidental falle en las pruebas en vez de generar N+1.

### 6.5 Caché como decorador (P-CA)

```python
# factura/infrastructure/cache/cached_query_service.py
class CachedFacturaQueryService:
    TTL_DETALLE = 300

    def __init__(self, inner: FacturaQueryService, cache: CachePort) -> None:
        self._inner, self._cache = inner, cache

    async def detalle(self, factura_id, empresa_id):
        key = FacturaCacheKeys.detalle(factura_id)
        try:
            if (raw := await self._cache.get(key)) is not None:
                return FacturaDetalle.from_json(raw)
        except CacheUnavailable:
            log.warning("cache_unavailable", key=key)          # Degradar, no fallar (P-CA-05)
        detalle = await self._inner.detalle(factura_id, empresa_id)
        if detalle is not None and detalle.empresa_id == empresa_id:
            await self._safe_set(key, detalle.to_json(), self.TTL_DETALLE)
        return detalle
```

> Al leer de la caché, **verifica el alcance** (`empresa_id`) del objeto cacheado o inclúyelo en la clave. Nunca devuelvas un dato cacheado de otro tenant.

Convención de claves (`application/cache_keys.py`):

```python
class FacturaCacheKeys:
    V = "v1"

    @staticmethod
    def detalle(factura_id) -> str:
        return f"app:factura:detalle:{FacturaCacheKeys.V}:{factura_id}"

    @staticmethod
    def listas_empresa(empresa_id) -> str:
        return f"app:factura:lista:{FacturaCacheKeys.V}:{empresa_id}:*"
```

### 6.6 Manejo global de errores (`shared/infrastructure/http/error_handlers.py`)

```python
STATUS_BY_KIND = {
    ErrorKind.VALIDATION: 422, ErrorKind.NOT_FOUND: 404, ErrorKind.CONFLICT: 409,
    ErrorKind.BUSINESS_RULE: 422, ErrorKind.UNAUTHENTICATED: 401, ErrorKind.FORBIDDEN: 403,
    ErrorKind.RATE_LIMITED: 429, ErrorKind.EXTERNAL: 502, ErrorKind.INTERNAL: 500,
}


def error_body(code: str, message: str, trace_id: str, details=None) -> dict:
    return {"error": {"code": code, "message": message, "details": details or [], "trace_id": trace_id}}


async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    status_code = STATUS_BY_KIND[exc.kind]
    logger = audit_log if exc.kind in (ErrorKind.FORBIDDEN, ErrorKind.UNAUTHENTICATED) else log
    level = "error" if status_code >= 500 else "warning"
    getattr(logger, level)("app_error", code=exc.code, status=status_code,
                           path=request.url.path, **exc.log_context)
    return JSONResponse(status_code=status_code,
                        content=error_body(exc.code, exc.message, request.state.trace_id, exc.details))


async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    # Solo campo y regla: NUNCA el valor recibido (P-VA-07)
    details = [{"field": ".".join(map(str, e["loc"][1:])), "rule": e["type"]} for e in exc.errors()]
    log.info("validation_failed", path=request.url.path, fields=[d["field"] for d in details])
    return JSONResponse(status_code=422, content=error_body(
        "VALIDACION_FALLIDA", "Algunos datos no son válidos. Revisa los campos indicados.",
        request.state.trace_id, details))


async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    log.exception("unhandled_error", path=request.url.path)          # Stack trace solo en logs
    return JSONResponse(status_code=500, content=error_body(
        "ERROR_INTERNO", "Ocurrió un error inesperado. Intenta de nuevo más tarde.",
        request.state.trace_id))
```

Las excepciones de infraestructura (por ejemplo `IntegrityError` o un timeout de un cliente externo) se **traducen** dentro del adaptador a un `AppError` (`ConflictError`, `ExternalServiceError`) y no se dejan subir crudas.

### 6.7 Logging

- Librería: `structlog`, con salida JSON en producción y salida legible en desarrollo.
- Un middleware asigna `trace_id` (de `X-Request-ID` o un ULID nuevo), lo pone en `contextvars` y lo devuelve en la cabecera de respuesta.
- Un procesador de enmascarado elimina o enmascara las claves `password`, `token`, `authorization`, `cookie`, `secret`, `numero_tarjeta` y `documento`.
- Loggers con nombre: `app` (general), `audit` (seguridad, P-LO-05), `events` (bus y outbox), `ws` (tiempo real).
- Log de acceso por request: método, ruta **plantilla** (`/facturas/{id}`, no el valor), status, duración, user_id.

### 6.8 Autenticación y permisos (`shared/infrastructure/security`)

```python
def require_permission(permiso: str):
    async def dependency(request: Request,
                         current: Annotated[Actor, Depends(get_current_actor)]) -> Actor:
        if not current.puede(permiso):
            audit_log.warning("access_denied", permiso=permiso, user_id=str(current.user_id),
                              path=request.url.path)
            raise ForbiddenError()
        return current
    return dependency
```

- `get_current_actor` valida el JWT (firma, `exp`, `iss`, `aud`, algoritmo fijo, nunca `none`), consulta la lista de revocación en Redis y carga los permisos del perfil **desde la caché** (`app:auth:permisos:v1:{rol_id}`, invalidada con el evento `PermisosDePerfilCambiados`).
- Tablas mínimas del módulo `auth`: `usuarios`, `perfiles`, `permisos`, `perfil_permisos`, `usuario_perfiles`, `refresh_tokens` (hash, familia, expiración, revocado).
- Rate limiting: `limit_req` en Nginx más un límite en la app con Redis (ventana deslizante) para login, refresh y recuperación de contraseña.

### 6.9 WebSockets (adaptador de entrada en tiempo real)

Componentes:

| Componente | Capa | Responsabilidad |
|---|---|---|
| `RealtimeNotifier` (puerto) | `shared/application` | Publicar un evento a un canal |
| `RedisRealtimeNotifier` | `shared/infrastructure/realtime` | Implementa el puerto con `PUBLISH` en Redis |
| `ConnectionManager` | `shared/infrastructure/realtime` | Conexiones locales por canal; se suscribe a Redis y reenvía a los sockets locales |
| `router.py` WS del módulo | `factura/infrastructure/websocket` | Handshake, autenticación, autorización de canal, validación de mensajes |

```python
# factura/infrastructure/websocket/router.py
router = APIRouter()


@router.websocket("/ws/v1/facturas")
async def facturas_ws(websocket: WebSocket,
                      manager: Annotated[ConnectionManager, Depends(get_connection_manager)]):
    if not origin_permitido(websocket.headers.get("origin")):          # P-EV-06
        await websocket.close(code=1008)
        return
    actor = await authenticate_websocket(websocket)                    # Cookie o ticket de un solo uso
    if actor is None or not actor.puede("factura:ver"):
        await websocket.close(code=1008)
        return

    channel = f"empresa:{actor.empresa_id}:facturas"                   # El canal lo decide el SERVIDOR
    await manager.connect(websocket, channel, actor)
    try:
        while True:
            raw = await websocket.receive_text()
            if len(raw) > 4096 or not manager.allow_message(actor):    # Tamaño y frecuencia
                await websocket.close(code=1009)
                break
            msg = WsClientMessage.model_validate_json(raw)              # Esquema estricto
            if msg.type == "ping":
                await websocket.send_json({"type": "pong"})
    except (WebSocketDisconnect, ValidationError):
        pass
    finally:
        await manager.disconnect(websocket, channel)
```

Reglas:
- El cliente **no** elige canales arbitrarios. Si se permite suscribirse a un canal concreto (por ejemplo, una factura), el servidor verifica permiso y alcance antes de aceptar.
- Formato de mensaje del servidor: `{"event": "factura.creada", "data": {...}, "event_id": "...", "trace_id": "..."}`.
- Cuando se revoca la sesión (logout o cambio de permisos), se publica en el canal de control `user:{id}:control` para que el `ConnectionManager` cierre los sockets de ese usuario.
- Un heartbeat (ping/pong) cierra las conexiones inactivas.
- El cliente, al recibir un evento, normalmente **invalida su caché y vuelve a consultar por HTTP** (P-EV-08).

---

## 7. Composition root (`bootstrap/container.py`)

```python
class Container:
    def __init__(self, settings: Settings) -> None:
        self.settings = settings
        self.engine = create_async_engine(settings.database_url, pool_pre_ping=True)
        self.session_factory = async_sessionmaker(self.engine, expire_on_commit=False)
        self.redis = Redis.from_url(settings.redis_url)
        self.cache = RedisCache(self.redis)
        self.realtime = RedisRealtimeNotifier(self.redis)
        self.event_bus = InMemoryEventBus()
        self.clock = SystemClock()
        register_all_subscriptions(self.event_bus, self)
```

Por request, `dependencies.py` construye la sesión, el UoW, los repositorios y el caso de uso:

```python
async def get_crear_factura(c: ContainerDep, session: SessionDep) -> CrearFactura:
    return CrearFactura(uow=SqlAlchemyUnitOfWork(session),
                        facturas=SqlAlchemyFacturaRepository(session),
                        event_bus=c.event_bus, clock=c.clock)
```

---

## 8. Base de datos (convenciones mínimas; detalle en ART-22)

- Tablas en `snake_case` y en plural; PK `id uuid` (UUIDv7 o `gen_random_uuid()`).
- Columnas de auditoría: `creado_en`, `actualizado_en`, `creado_por`; borrado lógico `eliminado_en` cuando aplique.
- Toda tabla de negocio multi-tenant tiene `empresa_id` con un índice compuesto que empieza por ese campo.
- Dinero en `numeric(14,2)`, nunca `float`; fechas en `timestamptz`.
- La aplicación se conecta con un usuario sin privilegios DDL. Las migraciones usan otro usuario.

---

## 9. Pruebas del backend

| Tipo | Ubicación | Qué prueba | Dependencias |
|---|---|---|---|
| Unitarias de dominio | `tests/unit/<modulo>/domain` | Invariantes, transiciones de estado, eventos registrados | Ninguna |
| Unitarias de aplicación | `tests/unit/<modulo>/application` | Casos de uso con fakes: permisos, errores, eventos publicados, observers | Fakes en memoria |
| Integración | `tests/integration/<modulo>` | Repositorios, query services, caché, outbox | PostgreSQL y Redis en contenedor (testcontainers) |
| API | `tests/api/<modulo>` | Contrato HTTP, formato de error, permisos, **presupuesto de consultas** | App + BD de prueba |
| Adversarias | `tests/adversarial/<modulo>` | P-PR-05 | App + BD de prueba |

Fakes de ejemplo (viven en `tests/fakes/`):

```python
class FakeFacturaRepository:
    def __init__(self) -> None:
        self.items: dict[UUID, Factura] = {}

    async def obtener(self, factura_id, empresa_id):
        f = self.items.get(factura_id)
        return f if f and f.empresa_id == empresa_id else None

    async def existe_numero(self, numero, empresa_id):
        return any(f.numero == numero and f.empresa_id == empresa_id for f in self.items.values())

    async def agregar(self, factura):
        self.items[factura.id] = factura

    guardar = agregar


class SpyEventBus:
    def __init__(self) -> None:
        self.published: list[DomainEvent] = []

    def subscribe(self, *_): ...

    async def publish(self, events):
        self.published.extend(events)
```

Verificación del presupuesto de consultas:

```python
@pytest.fixture
def query_counter(engine):
    counter = {"n": 0}

    def _count(*_):
        counter["n"] += 1

    event.listen(engine.sync_engine, "before_cursor_execute", _count)
    yield counter
    event.remove(engine.sync_engine, "before_cursor_execute", _count)


async def test_listar_facturas_presupuesto(client, query_counter, auth_headers):
    resp = await client.get("/api/v1/facturas?size=50", headers=auth_headers)
    assert resp.status_code == 200
    assert query_counter["n"] <= 2   # Presupuesto declarado en la spec (ART-46); incluye la consulta de permisos si no está cacheada
```

---

## 10. Docker y Nginx (puntos que afectan al backend)

- Se ejecuta con `uvicorn` (o `gunicorn` con workers `uvicorn`) detrás de Nginx, con `--proxy-headers` y `--forwarded-allow-ips` limitado a la red interna.
- Nginx hace proxy de `/api/` a HTTP y de `/ws/` con `proxy_http_version 1.1`, `Upgrade` y `Connection "upgrade"`, y un `proxy_read_timeout` mayor que el intervalo de heartbeat.
- Healthchecks: `/health/live` (proceso vivo) y `/health/ready` (BD y Redis disponibles), sin datos sensibles.

---

## 11. Cómo agregar un módulo nuevo (receta para agentes)

1. Crear `app/<modulo>/{domain,application,infrastructure}` con la estructura de §2.
2. **Domain:** entidad o agregado, value objects, eventos, errores con `code` únicos y puerto de repositorio.
3. **Application:** commands y results, puerto `QueryService`, un caso de uso por acción (con su `PERMISO`), observers y claves de caché.
4. **Infrastructure:** modelos ORM + migración Alembic, mappers, repositorio, query service (con su presupuesto), caché decoradora si aplica, esquemas Pydantic estrictos, router, WebSocket si aplica, suscripciones.
5. Registrar el router en `bootstrap/routers.py` y las suscripciones en `bootstrap/subscriptions.py`.
6. Insertar los permisos nuevos (`<modulo>:<accion>`) en la migración de semillas de `auth`.
7. Agregar el módulo a los contratos de `import-linter`.
8. Escribir las pruebas de §9.
9. Completar una especificación ART-46 por endpoint (y una de canal si hay WebSocket).

---

## 12. Antipatrones prohibidos

| Antipatrón | En su lugar |
|---|---|
| Consultas dentro de un `for` | Una consulta con `IN`/`ANY`, joins o `selectinload` |
| Devolver modelos ORM en la respuesta | Esquema de salida Pydantic explícito |
| Reglas de negocio en el router | Dominio o caso de uso |
| Enviar correos o notificar dentro del caso de uso | Evento + observer |
| Publicar eventos antes del commit | Después del commit, o con Outbox |
| `except Exception: pass` | Traducir a `AppError` o registrar y relanzar |
| Permisos por nombre de perfil (`if rol == "admin"`) | Permisos `recurso:accion` |
| Consultar un recurso por id sin el filtro de tenant | El repositorio y el query service siempre reciben el alcance |
| `KEYS *` en Redis | `SCAN` o claves con versión |
| Token JWT en la query string del WebSocket | Cookie o ticket de un solo uso |

---

## 13. Dependencias base sugeridas (`pyproject.toml`)

`fastapi`, `uvicorn[standard]`, `pydantic`, `pydantic-settings`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `redis`, `structlog`, `argon2-cffi`, `pyjwt`, `python-multipart`, `python-magic` (MIME por contenido), `uuid-utils` (UUIDv7), `orjson`, `httpx`, `taskiq` + `taskiq-redis` (worker, ADR-BASE-04).

Desarrollo: `pytest`, `pytest-asyncio`, `httpx`, `testcontainers`, `import-linter`, `ruff`, `mypy`, `pip-audit`, `bandit`, `schemathesis` (fuzzing de API para las pruebas adversarias).

---

## 14. Checklist del módulo (Definition of Done backend)

- [ ] Estructura `domain/application/infrastructure` y contratos de `import-linter` en verde.
- [ ] El dominio no importa frameworks; las invariantes están protegidas en las entidades y value objects.
- [ ] Cada caso de uso exige su permiso y aplica el alcance de tenant.
- [ ] Los esquemas de entrada son estrictos (tipo, longitud, patrón, rango, `extra="forbid"`).
- [ ] Los esquemas de salida son explícitos, sin campos de más.
- [ ] Los errores del módulo tienen `code` únicos y documentados en la especificación.
- [ ] El presupuesto de consultas está declarado y probado; los índices tienen su migración.
- [ ] La caché tiene clave, TTL e invalidación por observer, y degrada si Redis falla.
- [ ] Los eventos se publican después del commit; los observers son idempotentes; se usa Outbox para lo crítico.
- [ ] El WebSocket (si aplica) valida Origin, autenticación, autorización de canal y mensajes, con límites.
- [ ] Los logs son estructurados, con `trace_id` y sin datos sensibles.
- [ ] Hay pruebas unitarias, de integración, API y adversarias en verde, con cobertura ≥ 80 % en domain y application.
