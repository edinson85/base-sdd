---
id: ART-15
titulo: Optimización de acceso a datos
version: 1.0.0
estado: vigente
aplica_a: [backend, base-de-datos]
depende_de: [ART-02, ART-10, ART-25]
---

# ART-15 · Optimización de acceso a datos

> **Instrucción para agentes IA**
> Objetivo: que cada endpoint obtenga lo que necesita con el **menor número de consultas** (P-DA-01..04). Antes de escribir una consulta, busca el patrón en §2. Declara el presupuesto en la especificación (ART-46 §6) y pruébalo (ART-10 §9).

---

## 1. Presupuestos de referencia

| Tipo de endpoint | Presupuesto típico | Cómo |
|---|---|---|
| Lista paginada con datos relacionados | 1 | Joins + `count(*) OVER ()` |
| Detalle de agregado con hijos | 1-2 | `json_agg` (1) o `selectinload` (2) |
| Creación simple | 2 | Validación de referencias en 1 consulta + INSERT |
| Creación con hijos | 3 | Validación + INSERT padre + INSERT múltiple hijos |
| Actualización con regla | 2-3 | SELECT `FOR UPDATE` + UPDATE (+ hijos) |
| Acción masiva sobre N elementos | Constante (no depende de N) | `ANY(:ids)`, `UPDATE ... FROM (VALUES ...)` |
| Autenticación por request | 0 | Permisos desde caché (ART-16) |

Si un endpoint necesita más, se justifica en la especificación.

## 2. Catálogo de patrones (SQLAlchemy 2.x)

### 2.1 Lista con total en una consulta
Ver ART-10 §6.4 (`func.count().over()`). Si la tabla es muy grande y el total exacto es costoso, usar paginación por cursor sin total o un total estimado (ADR).

### 2.2 Detalle con hijos en una consulta (`json_agg`)
```python
from sqlalchemy import func, literal_column, select
from sqlalchemy.dialects.postgresql import aggregate_order_by

item_json = func.json_build_object(
    "producto_id", ItemModel.producto_id, "descripcion", ItemModel.descripcion,
    "cantidad", ItemModel.cantidad, "precio_unitario", ItemModel.precio_unitario,
)
items = (
    select(func.coalesce(func.json_agg(aggregate_order_by(item_json, ItemModel.orden)),
                         literal_column("'[]'::json")))
    .where(ItemModel.factura_id == FacturaModel.id)
    .scalar_subquery()
)
stmt = (select(FacturaModel.id, FacturaModel.numero, ClienteModel.nombre.label("cliente"),
               FacturaModel.total, items.label("items"))
        .join(ClienteModel, ClienteModel.id == FacturaModel.cliente_id)
        .where(FacturaModel.id == factura_id, FacturaModel.empresa_id == empresa_id))
```
> Alternativa equivalente y más simple: `selectinload` (2 consultas fijas). Usar `json_agg` cuando el detalle se consulta mucho o tiene varios hijos.

### 2.3 Cargar agregado para escritura
```python
stmt = (select(FacturaModel)
        .options(selectinload(FacturaModel.items))
        .where(FacturaModel.id == fid, FacturaModel.empresa_id == eid)
        .with_for_update())          # Bloquea la fila para evitar carreras en reglas de estado
```

### 2.4 Validar muchas referencias en una consulta
```python
ids = {i.producto_id for i in cmd.items}
encontrados = set((await s.execute(
    select(ProductoModel.id).where(ProductoModel.id.in_(ids),
                                   ProductoModel.empresa_id == eid,
                                   ProductoModel.activo.is_(True))
)).scalars())
faltantes = ids - encontrados
```

### 2.5 Varias validaciones en una sola ida a la BD
```python
stmt = select(
    exists().where(FacturaModel.empresa_id == eid, FacturaModel.numero == numero).label("numero_existe"),
    exists().where(ClienteModel.id == cid, ClienteModel.empresa_id == eid, ClienteModel.activo).label("cliente_ok"),
)
row = (await s.execute(stmt)).one()
```

### 2.6 Inserción múltiple
```python
await s.execute(insert(ItemModel), [item_dict_1, item_dict_2, ...])   # executemany eficiente con asyncpg
```

### 2.7 Upsert
```python
stmt = pg_insert(StockModel).values(rows).on_conflict_do_update(
    index_elements=[StockModel.empresa_id, StockModel.producto_id],
    set_={"cantidad": StockModel.cantidad + pg_insert(StockModel).excluded.cantidad},
)
```

### 2.8 Actualización masiva con valores distintos
`UPDATE ... FROM (VALUES ...)` con `bindparam` o una tabla temporal; nunca un UPDATE por fila en un loop.

### 2.9 Existencia
`EXISTS` en vez de `COUNT(*) > 0`.

### 2.10 Paginación por cursor (keyset)
```python
stmt = (select(...).where(MovModel.empresa_id == eid)
        .where(tuple_(MovModel.creado_en, MovModel.id) < (cursor.creado_en, cursor.id))
        .order_by(MovModel.creado_en.desc(), MovModel.id.desc())
        .limit(size + 1))           # +1 para saber si hay página siguiente
```
Requiere índice `(empresa_id, creado_en DESC, id DESC)`.

## 3. Reglas del ORM
- Relaciones con `lazy="raise"` por defecto: cualquier carga perezosa no planificada falla en pruebas.
- `expire_on_commit=False` en la sesión async.
- Lecturas: seleccionar columnas (proyecciones), no entidades completas (P-DA-03).
- Nada de `session.refresh()` innecesario después de insertar: el id ya existe (UUIDv7 generado en la app, ADR-BASE-09).
- Prohibido `SELECT *` en SQL escrito a mano.
- SQL crudo solo con `text()` y parámetros enlazados (P-DA-07).

## 4. Límites y protección de la BD
- `statement_timeout` por rol de aplicación (ej. 5 s) y por consulta pesada cuando haga falta (`SET LOCAL statement_timeout`).
- `idle_in_transaction_session_timeout` (ej. 30 s).
- Pool: `pool_size` + `max_overflow` × réplicas ≤ `max_connections` − margen (ver ART-25 §6).
- Transacciones cortas: nada de llamadas HTTP externas dentro de una transacción.

## 5. Detección
- Prueba de presupuesto por endpoint (P-PR-03).
- En desarrollo: log de consultas por request (conteo y tiempo) con un middleware en DEBUG.
- En staging/producción: `pg_stat_statements` + revisión semanal de las 10 consultas más costosas (ART-25).
- Toda consulta nueva sobre tablas que superarán 100 mil filas se revisa con `EXPLAIN (ANALYZE, BUFFERS)` y el resultado se anota en el PR.
