---
id: ART-24
titulo: Migraciones y datos semilla
version: 1.0.0
estado: vigente
aplica_a: [base-de-datos, backend, infraestructura]
depende_de: [ART-22, ART-26]
---

# ART-24 · Migraciones y datos semilla (Alembic)

> **Instrucción para agentes IA**
> Toda modificación del esquema es una migración Alembic revisada a mano (la autogeneración es un borrador). Principio P-DA-08.

---

## 1. Configuración
- `migrations/` en `backend/`, `env.py` usa `metadata` con `naming_convention` (ART-22 §1) y `compare_type=True`.
- Las migraciones se ejecutan con el usuario **migrador** (dueño del esquema), nunca con el de la aplicación (ART-26).
- `alembic upgrade head` lo ejecuta el job `migrate` antes de iniciar la API (ART-33).
- Nombre de archivo: `AAAAMMDD_HHMM_<descripcion>.py` (configurar `file_template` en `alembic.ini`).

## 2. Reglas

| # | Regla |
|---|---|
| M-1 | Toda migración tiene `downgrade` funcional, o un comentario que explique por qué es irreversible y el plan de recuperación (backup) |
| M-2 | Una migración = un cambio lógico; no mezclar cambios de esquema grandes con migraciones de datos masivas |
| M-3 | Nunca editar una migración ya aplicada en un ambiente compartido; crear otra |
| M-4 | Compatibilidad hacia atrás: la versión N-1 de la API debe funcionar con el esquema N (patrón expand/contract) |
| M-5 | Índices en tablas con datos: `CREATE INDEX CONCURRENTLY` (fuera de transacción) |
| M-6 | `lock_timeout` corto en migraciones para no bloquear producción (`SET lock_timeout = '5s'`) |
| M-7 | Probada en CI: `upgrade head` → `downgrade -1` → `upgrade head` (ART-36 etapa 7) |
| M-8 | Migraciones de datos con lotes (≤ 10 000 filas por lote) si la tabla es grande |
| M-9 | Sin datos personales reales en migraciones ni semillas |

## 3. Expand / contract (cambios sin interrupción)

Ejemplo: renombrar `facturas.numero` → `facturas.consecutivo`.

| Release | Migración | Código |
|---|---|---|
| N | Agregar `consecutivo` (nullable); copiar datos en lotes; trigger o doble escritura | Escribe en ambas columnas, lee `numero` |
| N+1 | `consecutivo` NOT NULL + índice único | Lee y escribe `consecutivo` |
| N+2 | Eliminar `numero` | — |

Operaciones peligrosas y alternativa:
| Operación | Riesgo | Alternativa |
|---|---|---|
| `ADD COLUMN ... NOT NULL DEFAULT <volátil>` | Reescritura de tabla | Agregar nullable, rellenar por lotes, luego `NOT NULL` |
| `ALTER COLUMN TYPE` | Reescritura y bloqueo | Columna nueva + copia |
| `ADD CONSTRAINT FOREIGN KEY` | Bloqueo de validación | `NOT VALID` y luego `VALIDATE CONSTRAINT` |
| `CREATE INDEX` | Bloquea escrituras | `CONCURRENTLY` |
| `DROP COLUMN` usada por la versión anterior | Errores en caliente | Contract en el release siguiente |

```python
def upgrade() -> None:
    op.execute("SET lock_timeout = '5s'")
    with op.get_context().autocommit_block():
        op.create_index("ix_facturas_cliente", "facturas", ["cliente_id"],
                        schema="app", postgresql_concurrently=True)
```

## 4. Datos semilla

| Tipo | Ejemplos | Mecanismo | Ambientes |
|---|---|---|---|
| Referencia obligatoria | Permisos, perfiles de sistema, catálogos fijos (tipos de documento, estados) | **Migración de datos** idempotente (`INSERT ... ON CONFLICT DO NOTHING/UPDATE`) | Todos |
| Configuración inicial | Primera empresa y usuario administrador | Comando CLI `python -m app.bootstrap.crear_admin` con datos por variables/prompt; contraseña generada y mostrada una vez | Por despliegue |
| Datos de desarrollo y demo | Clientes, productos, facturas de ejemplo | Script `scripts/seed_dev.py` con factories (Faker `es_CO`) | Solo local/staging; protegido para que falle si `APP_ENV=production` |
| Datos de pruebas | Por prueba | Factories en `tests/` | CI |

Ejemplo de semilla de permisos:
```python
PERMISOS = [
    ("factura:ver", "Ver facturas", "factura"),
    ("factura:crear", "Crear facturas", "factura"),
    ("factura:anular", "Anular facturas", "factura"),
]

def upgrade() -> None:
    op.execute(sa.text("""
        INSERT INTO app.permisos (codigo, descripcion, modulo)
        SELECT * FROM unnest(:codigos, :descripciones, :modulos)
        ON CONFLICT (codigo) DO UPDATE SET descripcion = EXCLUDED.descripcion
    """).bindparams(codigos=[p[0] for p in PERMISOS],
                     descripciones=[p[1] for p in PERMISOS],
                     modulos=[p[2] for p in PERMISOS]))
    # Asignar al perfil Administrador de sistema
    op.execute(sa.text("""
        INSERT INTO app.perfil_permisos (perfil_id, permiso_codigo)
        SELECT p.id, c FROM app.perfiles p, unnest(:codigos) c
        WHERE p.es_sistema AND p.nombre = 'Administrador'
        ON CONFLICT DO NOTHING
    """).bindparams(codigos=[p[0] for p in PERMISOS]))
```

## 5. Checklist de revisión de una migración
- [ ] Generada y revisada a mano; nombres según convención.
- [ ] `downgrade` probado.
- [ ] Sin operaciones bloqueantes en tablas grandes (§3).
- [ ] Índices para FKs y consultas nuevas.
- [ ] Permisos nuevos sembrados y asignados.
- [ ] Compatible con la versión anterior de la API.
- [ ] Probada en staging con volumen representativo si toca tablas grandes.
