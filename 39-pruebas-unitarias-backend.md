---
id: ART-39
titulo: Pruebas unitarias e integración – Backend
version: 1.0.0
estado: vigente
aplica_a: [backend]
depende_de: [ART-10, ART-38]
---

# ART-39 · Pruebas unitarias e integración – Backend

> **Instrucción para agentes IA**
> Base en ART-10 §9 (estructura, fakes, contador de consultas). Este documento fija la configuración y los patrones de prueba por tipo de componente.

---

## 1. Configuración

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
  "integration: requiere PostgreSQL/Redis (testcontainers)",
  "api: endpoint completo",
  "adversarial: pruebas de ataque",
]
addopts = "-ra --strict-markers"

[tool.coverage.run]
source = ["app"]
branch = true
omit = ["app/main.py", "app/bootstrap/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

Cobertura medida sobre `app/*/domain` y `app/*/application` como puerta (script en CI que filtra el reporte).

## 2. Fixtures base (`tests/conftest.py`)

```python
@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16-alpine", driver="asyncpg") as pg:
        yield pg

@pytest.fixture(scope="session")
async def engine(postgres):
    eng = create_async_engine(postgres.get_connection_url())
    async with eng.begin() as conn:
        await conn.run_sync(lambda c: command.upgrade(alembic_cfg(c), "head"))   # Migraciones reales
    yield eng
    await eng.dispose()

@pytest.fixture
async def session(engine):
    async with engine.connect() as conn:
        trans = await conn.begin()
        s = AsyncSession(bind=conn, expire_on_commit=False,
                         join_transaction_mode="create_savepoint")   # Rollback al final de cada prueba
        yield s
        await s.close()
        await trans.rollback()

@pytest.fixture
def actor_factory():
    def _make(*, permisos=(), empresa_id=None):
        return Actor(user_id=uuid7(), empresa_id=empresa_id or uuid7(),
                     permisos=frozenset(permisos), trace_id="test")
    return _make

@pytest.fixture
def clock():
    return FixedClock(datetime(2026, 1, 15, 12, 0, tzinfo=timezone.utc))
```

## 3. Patrones por componente

### 3.1 Value object
```python
@pytest.mark.parametrize("valor", ["FV-001", "A1B", "X" * 20])
def test_numero_factura_valido(valor):
    assert NumeroFactura(valor).valor == valor

@pytest.mark.parametrize("valor", ["", "AB", "X" * 21, "fv-001", "FV 001"])
def test_numero_factura_invalido_lanza_error(valor):
    with pytest.raises(NumeroFacturaInvalido):
        NumeroFactura(valor)
```

### 3.2 Agregado
```python
def test_anular_factura_emitida_cambia_estado_y_registra_evento(clock):
    factura = FacturaFactory.emitida()
    factura.pull_events()

    factura.anular("Error en el cliente")

    assert factura.estado is EstadoFactura.ANULADA
    [evento] = factura.pull_events()
    assert isinstance(evento, FacturaAnulada) and evento.factura_id == factura.id

def test_anular_factura_anulada_lanza_error():
    factura = FacturaFactory.anulada()
    with pytest.raises(FacturaYaAnulada):
        factura.anular("otra vez")
```

### 3.3 Caso de uso
```python
async def test_crear_factura_sin_permiso_no_toca_el_repositorio(actor_factory, clock):
    repo, bus = FakeFacturaRepository(), SpyEventBus()
    uc = CrearFactura(FakeUnitOfWork(), repo, bus, clock)

    with pytest.raises(ForbiddenError):
        await uc.execute(CrearFacturaCommandFactory(), actor_factory(permisos=["factura:ver"]))

    assert repo.items == {} and bus.published == []

async def test_crear_factura_publica_evento_despues_del_commit(actor_factory, clock):
    uow, repo, bus = FakeUnitOfWork(), FakeFacturaRepository(), SpyEventBus()
    uow.on_commit(lambda: bus.assert_empty())          # Nada publicado antes del commit

    await CrearFactura(uow, repo, bus, clock).execute(CrearFacturaCommandFactory(),
                                                     actor_factory(permisos=["factura:crear"]))

    assert uow.committed
    assert [type(e) for e in bus.published] == [FacturaCreada]

async def test_crear_factura_si_falla_commit_no_publica(actor_factory, clock):
    uow = FakeUnitOfWork(fail_on_commit=True)
    bus = SpyEventBus()
    with pytest.raises(RuntimeError):
        await CrearFactura(uow, FakeFacturaRepository(), bus, clock).execute(
            CrearFacturaCommandFactory(), actor_factory(permisos=["factura:crear"]))
    assert bus.published == []
```

### 3.4 Observer (idempotencia)
```python
async def test_invalidar_cache_es_idempotente():
    cache = FakeCache({"app:factura:detalle:v1:X": b"..."})
    evento = FacturaAnuladaFactory()
    handler = InvalidarCacheFacturas(cache)

    await handler(evento)
    await handler(evento)

    assert cache.deleted_count_for(FacturaCacheKeys.detalle(evento.factura_id)) >= 1
    assert cache.get_sync(FacturaCacheKeys.detalle(evento.factura_id)) is None
```

### 3.5 Bus de eventos (aislamiento)
```python
async def test_falla_de_un_observer_no_impide_los_demas():
    bus = InMemoryEventBus()
    llamados = []
    async def falla(_): raise RuntimeError("x")
    async def ok(e): llamados.append(e)
    bus.subscribe(FacturaCreada, falla)
    bus.subscribe(FacturaCreada, ok)

    await bus.publish([FacturaCreadaFactory()])

    assert len(llamados) == 1
```

### 3.6 Repositorio y query service (integración)
```python
@pytest.mark.integration
async def test_query_service_no_devuelve_facturas_de_otra_empresa(session, seed):
    emp_a, emp_b = await seed.empresas(2)
    await seed.facturas(emp_b, n=3)
    page = await SqlFacturaQueryService(session).listar(emp_a.id, FiltroFacturas(), PageRequest(1, 20))
    assert page.items == [] and page.total == 0
```

### 3.7 Endpoint (API)
```python
@pytest.mark.api
async def test_post_facturas_responde_201_con_contrato(client, auth_as, seed):
    headers = await auth_as("cajero")
    resp = await client.post("/api/v1/facturas", json=payload_valido(seed),
                             headers=headers | {"Idempotency-Key": str(uuid4())})
    assert resp.status_code == 201
    assert resp.headers["Location"].startswith("/api/v1/facturas/")
    body = resp.json()["data"]
    assert set(body) == {"id", "numero", "cliente", "total", "estado"}    # Sin campos de más

@pytest.mark.api
async def test_error_tiene_formato_estandar(client, auth_as):
    resp = await client.get(f"/api/v1/facturas/{uuid7()}", headers=await auth_as("cajero"))
    err = resp.json()["error"]
    assert resp.status_code == 404
    assert err["code"] == "FACTURA_NO_ENCONTRADA" and err["trace_id"] and "details" in err
```

### 3.8 Caché con Redis caído
```python
async def test_detalle_funciona_si_redis_no_responde(...):
    service = CachedFacturaQueryService(inner, BrokenCache())
    assert await service.detalle(fid, eid) == esperado
```

### 3.9 WebSocket
```python
@pytest.mark.api
def test_ws_rechaza_sin_ticket(sync_client):
    with pytest.raises(WebSocketDisconnect) as exc:
        with sync_client.websocket_connect("/ws/v1/facturas", headers={"origin": ORIGEN_OK}):
            pass
    assert exc.value.code == 1008
```

## 4. Qué NO hacer
- No simular (mock) lo que se está probando ni el dominio: usar objetos reales del dominio y fakes de puertos.
- No usar `MagicMock` para puertos con lógica (repositorios): escribir fakes en memoria que respeten el alcance.
- No probar detalles privados; probar comportamiento observable.
- No usar SQLite como sustituto de PostgreSQL en pruebas de integración.
