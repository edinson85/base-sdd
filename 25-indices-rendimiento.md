---
id: ART-25
titulo: Índices y rendimiento de PostgreSQL
version: 1.0.0
estado: vigente
aplica_a: [base-de-datos, backend]
depende_de: [ART-15, ART-22]
---

# ART-25 · Índices y rendimiento

> **Instrucción para agentes IA**
> Los índices se diseñan a partir de las consultas especificadas (ART-46 §6), no "por si acaso". Cada índice tiene un costo en escrituras y espacio.

---

## 1. Tipos de índice y cuándo usarlos

| Tipo | Uso | Ejemplo |
|---|---|---|
| B-tree compuesto | Igualdad + rango/orden | `(empresa_id, creado_en DESC, id DESC)` |
| Parcial | Subconjunto consultado con frecuencia | `ON facturas (empresa_id) WHERE estado = 'EMITIDA'` |
| Cobertura (`INCLUDE`) | Evitar ir a la tabla en listados | `(empresa_id, creado_en DESC) INCLUDE (numero, total, estado)` |
| Expresión | Búsqueda case-insensitive | `ON usuarios (lower(email))` |
| GIN + `pg_trgm` | Búsqueda por contenido (`ILIKE '%texto%'`) | `USING gin (nombre gin_trgm_ops)` |
| GIN full-text | Búsqueda de texto en español | `USING gin (to_tsvector('spanish', unaccent_inmutable(descripcion)))` |
| GIN sobre `jsonb` | Filtros por claves de JSON | `USING gin (atributos jsonb_path_ops)` |
| BRIN | Tablas enormes con inserción ordenada por tiempo | `USING brin (creado_en)` en logs/movimientos |

Reglas para compuestos:
1. Primero las columnas de igualdad (`empresa_id`, `estado`), luego la de rango u orden.
2. El orden del índice coincide con el `ORDER BY` de la consulta (incluido el desempate `id`).
3. Un índice `(a, b)` sirve para consultas por `a`; no hace falta otro solo en `a`.

> `unaccent` no es inmutable; para usarlo en un índice se crea un wrapper `IMMUTABLE` (`unaccent_inmutable`) con el diccionario explícito. Documentarlo en la migración.

## 2. Cuándo crear un índice
- Toda FK (ART-22 §4).
- Columnas de filtros y orden de las consultas especificadas, cuando la tabla superará ~10 000 filas.
- Restricciones de unicidad del negocio.

Cuándo no: tablas pequeñas (< 1 000 filas), columnas de baja selectividad sin índice parcial, duplicados de prefijo de otro índice.

## 3. Lectura de `EXPLAIN (ANALYZE, BUFFERS)`

| Señal | Problema probable | Acción |
|---|---|---|
| `Seq Scan` en tabla grande con filtro selectivo | Falta índice | Índice compuesto según §1 |
| `Rows Removed by Filter` alto | Índice no cubre el filtro | Agregar columna o índice parcial |
| `Sort` con `external merge` | Orden sin índice / `work_mem` bajo | Índice con el mismo orden |
| `Nested Loop` con muchas iteraciones | Estimaciones malas o falta índice en el lado interno | `ANALYZE`, índice en la FK |
| Estimado vs. real con diferencia ×10 | Estadísticas desactualizadas | `ANALYZE`; estadísticas extendidas (`CREATE STATISTICS`) para columnas correlacionadas |
| `Buffers: shared read` alto | Datos fuera de memoria | Índice de cobertura, revisar `shared_buffers` |

El plan se adjunta en el PR para consultas sobre tablas grandes (ART-15 §5).

## 4. Monitoreo
- Extensión `pg_stat_statements` activada en todos los ambientes no locales.
- Revisión semanal: top 10 por `total_exec_time` y por `mean_exec_time`.
- Índices sin uso (`pg_stat_user_indexes.idx_scan = 0` tras un mes) → candidatos a eliminar.
- Tablas con muchos `seq_scan` sobre muchas filas → candidatas a índice.
- `log_min_duration_statement = 500ms` para registrar consultas lentas (sin parámetros sensibles: `log_parameter_max_length = 0` en producción).

## 5. Configuración base de PostgreSQL (servidor de 4 GB RAM, ajustar proporcionalmente)

```conf
shared_buffers = 1GB                 # ~25 % de RAM
effective_cache_size = 3GB           # ~75 % de RAM
work_mem = 16MB                      # Por operación de orden/hash; cuidado con muchas conexiones
maintenance_work_mem = 256MB
random_page_cost = 1.1               # SSD
effective_io_concurrency = 200       # SSD
max_connections = 100
wal_compression = on
shared_preload_libraries = 'pg_stat_statements'
log_min_duration_statement = 500
log_lock_waits = on
idle_in_transaction_session_timeout = 30s
```
Para el rol de aplicación: `ALTER ROLE app_user SET statement_timeout = '5s';` (ART-26).

## 6. Conexiones
- `max_connections` total ≥ (`pool_size` + `max_overflow`) × réplicas de API + worker + migrador + administración + margen.
- Ejemplo: 3 réplicas × (10 + 5) + worker 10 + 10 admin = 65 → `max_connections = 100`.
- Con muchas réplicas o conexiones cortas: PgBouncer en modo `transaction` (ADR). Con asyncpg + PgBouncer en modo transacción, desactivar la caché de sentencias preparadas (`statement_cache_size=0` o `prepared_statement_cache_size=0` en SQLAlchemy) o usar una versión de PgBouncer con soporte de prepared statements.

## 7. Tablas grandes
- Particionamiento declarativo por rango de fecha (mensual) para tablas de alto volumen e historial (> 50 M filas o borrado masivo por fecha); la clave de partición debe estar en la PK y en las consultas.
- Archivado: mover datos antiguos a tablas o almacenamiento de archivo según la retención del brief.
- `autovacuum` ajustado por tabla para tablas con mucha actualización (`autovacuum_vacuum_scale_factor = 0.05`).
