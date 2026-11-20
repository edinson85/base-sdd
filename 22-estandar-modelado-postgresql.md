---
id: ART-22
titulo: Estándar de modelado PostgreSQL
version: 1.0.0
estado: vigente
aplica_a: [base-de-datos, backend]
depende_de: [ART-02, ART-07, ART-08]
---

# ART-22 · Estándar de modelado PostgreSQL

> **Instrucción para agentes IA**
> Toda tabla nueva sigue este estándar. El modelo de cada proyecto se documenta con ART-23 y se crea con migraciones (ART-24).

---

## 1. Nombres

| Elemento | Regla | Ejemplo |
|---|---|---|
| Esquema | Uno por aplicación (`app`), no se usa `public` para tablas de negocio | `app.facturas` |
| Tablas | snake_case, plural, español sin tildes | `facturas`, `items_factura`, `ordenes_compra` |
| Columnas | snake_case singular | `numero`, `cliente_id`, `creado_en` |
| PK | `id` | |
| FK | `<entidad_singular>_id` | `cliente_id` |
| Booleanos | `es_`, `tiene_`, o adjetivo | `activo`, `es_sistema` |
| Fechas/horas | sufijo `_en` (timestamptz) o `fecha_` (date) | `emitida_en`, `fecha_vencimiento` |
| Índices | `ix_<tabla>_<columnas>`; únicos `ux_...` | `ix_facturas_empresa_creado` |
| Constraints | `pk_`, `fk_<tabla>_<ref>`, `ck_<tabla>_<regla>`, `uq_` | `ck_facturas_total_no_negativo` |

Se configura `MetaData(naming_convention=...)` en SQLAlchemy para que Alembic genere estos nombres.

```python
NAMING = {
    "ix": "ix_%(table_name)s_%(column_0_N_name)s",
    "uq": "uq_%(table_name)s_%(column_0_N_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}
metadata = MetaData(schema="app", naming_convention=NAMING)
```

## 2. Tipos

| Dato | Tipo | No usar |
|---|---|---|
| Identificador | `uuid` (UUIDv7 generado en la app, ADR-BASE-09) | `serial` expuesto al cliente |
| Texto | `text` + `CHECK (char_length(col) <= N)` o `varchar(N)` | `char(N)` |
| Dinero | `numeric(14,2)` (ajustar precisión al negocio) | `float`, `real`, `money` |
| Cantidades decimales | `numeric(p,s)` | `float` |
| Fecha y hora | `timestamptz` (UTC) | `timestamp` sin zona |
| Solo fecha | `date` | |
| Estados / enumeraciones | `text` + `CHECK (estado IN (...))`, o tabla catálogo si el negocio los administra | Tipo `ENUM` nativo (difícil de modificar) |
| Booleano | `boolean NOT NULL DEFAULT ...` | `int`, `char(1)` |
| JSON | `jsonb`, solo para datos sin estructura fija o snapshots | Para datos que se filtran o relacionan |
| IP | `inet` | `text` |
| Archivos | Referencia (ruta/clave) en `text`; el binario fuera de la BD | `bytea` para archivos grandes |

## 3. Columnas estándar

Toda tabla de negocio tiene:

```sql
id              uuid PRIMARY KEY,
empresa_id      uuid NOT NULL REFERENCES app.empresas(id),   -- si el proyecto es multi-tenant
creado_en       timestamptz NOT NULL DEFAULT now(),
creado_por      uuid REFERENCES app.usuarios(id),
actualizado_en  timestamptz NOT NULL DEFAULT now(),
actualizado_por uuid REFERENCES app.usuarios(id),
version         integer NOT NULL DEFAULT 1                   -- concurrencia optimista (ART-14 §5)
```
Opcional: `eliminado_en timestamptz` (borrado lógico).

- `actualizado_en` y `version` los actualiza la aplicación (repositorio) en cada UPDATE; un trigger es aceptable como respaldo.
- Tablas de catálogo globales (ciudades, monedas) no llevan `empresa_id`.

## 4. Integridad
- `NOT NULL` por defecto; nullable solo si el negocio lo requiere.
- FK en todas las relaciones, con índice en la columna FK (PostgreSQL no lo crea solo).
- `ON DELETE`: `RESTRICT` por defecto; `CASCADE` solo para hijos de un agregado (ej. `items_factura`).
- `CHECK` para invariantes simples que también valida el dominio (`total >= 0`, `cantidad > 0`, estados permitidos).
- Unicidad de negocio con el alcance: `UNIQUE (empresa_id, numero)`.
- Con borrado lógico, unicidad parcial: `CREATE UNIQUE INDEX ... ON clientes (empresa_id, documento) WHERE eliminado_en IS NULL`.

## 5. Borrado
| Caso | Estrategia |
|---|---|
| Documentos con valor legal o contable (facturas) | Nunca se borran; se anulan (cambio de estado + evento) |
| Entidades maestras (clientes, productos) | Borrado lógico (`eliminado_en`) o `activo = false` |
| Datos personales con solicitud de supresión | Anonimización (reemplazo de campos) preservando integridad referencial |
| Datos técnicos (tokens expirados, outbox publicado) | Borrado físico por tarea programada |

Los query services excluyen eliminados por defecto (`WHERE eliminado_en IS NULL`).

## 6. Auditoría de cambios
- Eventos de dominio + logs de auditoría cubren la mayoría de casos.
- Si el brief exige historial de cambios por registro: tabla `auditoria_cambios(id, tabla, registro_id, accion, cambios jsonb, actor_id, trace_id, creado_en)` alimentada por un observer, sin datos sensibles en `cambios`.

## 7. Tablas técnicas estándar
| Tabla | Propósito | Artefacto |
|---|---|---|
| `outbox_events` | Eventos confiables | ART-10 §5.3 |
| `refresh_tokens` | Sesiones | ART-20 |
| `idempotencia` (opcional si no se usa Redis) | Respuestas por `Idempotency-Key` | ART-14 §5 |
| `procesos` | Estado de procesos asíncronos | ART-14 §3.3 |
| `alembic_version` | Versión del esquema | ART-24 |

## 8. Ejemplo

```sql
CREATE TABLE app.facturas (
  id              uuid PRIMARY KEY,
  empresa_id      uuid NOT NULL REFERENCES app.empresas(id),
  numero          text NOT NULL CHECK (char_length(numero) BETWEEN 3 AND 20),
  cliente_id      uuid NOT NULL REFERENCES app.clientes(id),
  total           numeric(14,2) NOT NULL CHECK (total >= 0),
  estado          text NOT NULL CHECK (estado IN ('EMITIDA','ANULADA')),
  motivo_anulacion text CHECK (char_length(motivo_anulacion) <= 500),
  emitida_en      timestamptz NOT NULL,
  creado_en       timestamptz NOT NULL DEFAULT now(),
  creado_por      uuid REFERENCES app.usuarios(id),
  actualizado_en  timestamptz NOT NULL DEFAULT now(),
  actualizado_por uuid REFERENCES app.usuarios(id),
  version         integer NOT NULL DEFAULT 1,
  CONSTRAINT uq_facturas_empresa_numero UNIQUE (empresa_id, numero),
  CONSTRAINT ck_facturas_anulacion CHECK (estado <> 'ANULADA' OR motivo_anulacion IS NOT NULL)
);
CREATE INDEX ix_facturas_empresa_creado ON app.facturas (empresa_id, creado_en DESC, id DESC);
CREATE INDEX ix_facturas_cliente ON app.facturas (cliente_id);

CREATE TABLE app.items_factura (
  id               uuid PRIMARY KEY,
  factura_id       uuid NOT NULL REFERENCES app.facturas(id) ON DELETE CASCADE,
  orden            smallint NOT NULL,
  producto_id      uuid NOT NULL REFERENCES app.productos(id),
  descripcion      text NOT NULL CHECK (char_length(descripcion) <= 200),
  cantidad         integer NOT NULL CHECK (cantidad > 0),
  precio_unitario  numeric(14,2) NOT NULL CHECK (precio_unitario > 0),
  UNIQUE (factura_id, orden)
);
CREATE INDEX ix_items_factura_producto ON app.items_factura (producto_id);
```
(La FK `factura_id` queda indexada por el `UNIQUE (factura_id, orden)`.)
