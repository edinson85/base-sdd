---
id: ART-26
titulo: Seguridad de la base de datos
version: 1.0.0
estado: vigente
aplica_a: [base-de-datos, infraestructura]
depende_de: [ART-02, ART-22, ART-33, ART-35, ART-37]
---

# ART-26 · Seguridad de la base de datos

> **Instrucción para agentes IA**
> El script de §2 forma parte de la inicialización de todo proyecto (`infra/postgres/init/`). La aplicación nunca se conecta como superusuario ni como dueño del esquema.

---

## 1. Roles (mínimo privilegio)

| Rol | Uso | Permisos |
|---|---|---|
| `postgres` (superusuario) | Solo administración puntual | Todo; contraseña fuerte, sin uso desde aplicaciones |
| `<app>_owner` | Dueño de la BD y del esquema `app` | `CREATE` en el esquema; `NOLOGIN` (el migrador hereda sus permisos) |
| `<app>_migrator` | Job `migrate` | Miembro de `<app>_owner` (hereda DDL) |
| `<app>_app` | API y worker | `SELECT, INSERT, UPDATE, DELETE` en tablas; `USAGE` en secuencias; sin DDL, sin `TRUNCATE` |
| `<app>_readonly` | Reportes, BI, soporte | Solo `SELECT` (idealmente sobre vistas sin datos sensibles) |

## 2. Script de inicialización

> En la imagen oficial de Docker, `POSTGRES_USER` se crea como **superusuario**; por eso ese usuario es `postgres` (administración) y el dueño de la aplicación se crea aquí sin `LOGIN`. El script se ejecuta desde `infra/postgres/init/01-roles.sh`, que lee las contraseñas de `/run/secrets` y llama a `psql -v migrator_password=... -v app_password=... -v readonly_password=... -f /docker-entrypoint-initdb.d/sql/01-roles.sql` (el `.sql` va en una subcarpeta para que la imagen no lo ejecute por su cuenta sin variables).

```sql
-- infra/postgres/init/sql/01-roles.sql
CREATE ROLE proyecto_owner NOLOGIN;
ALTER DATABASE proyecto OWNER TO proyecto_owner;
REVOKE ALL ON DATABASE proyecto FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

CREATE ROLE proyecto_migrator LOGIN PASSWORD :'migrator_password' IN ROLE proyecto_owner;
CREATE ROLE proyecto_app LOGIN PASSWORD :'app_password' CONNECTION LIMIT 80;
CREATE ROLE proyecto_readonly LOGIN PASSWORD :'readonly_password' CONNECTION LIMIT 5;

CREATE SCHEMA IF NOT EXISTS app AUTHORIZATION proyecto_owner;
GRANT CONNECT ON DATABASE proyecto TO proyecto_app, proyecto_readonly;
GRANT USAGE ON SCHEMA app TO proyecto_app, proyecto_readonly;

-- Permisos por defecto para tablas que cree el dueño en el futuro
ALTER DEFAULT PRIVILEGES FOR ROLE proyecto_owner IN SCHEMA app
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO proyecto_app;
ALTER DEFAULT PRIVILEGES FOR ROLE proyecto_owner IN SCHEMA app
  GRANT USAGE, SELECT ON SEQUENCES TO proyecto_app;
ALTER DEFAULT PRIVILEGES FOR ROLE proyecto_owner IN SCHEMA app
  GRANT SELECT ON TABLES TO proyecto_readonly;

ALTER ROLE proyecto_app SET search_path = app;
ALTER ROLE proyecto_app SET statement_timeout = '5s';
ALTER ROLE proyecto_app SET idle_in_transaction_session_timeout = '30s';
ALTER ROLE proyecto_readonly SET default_transaction_read_only = on;
ALTER ROLE proyecto_readonly SET statement_timeout = '60s';
```

> Como las migraciones las ejecuta `proyecto_migrator` (miembro de `proyecto_owner`), en la migración inicial se ejecuta `SET ROLE proyecto_owner` para que los objetos queden con ese dueño y apliquen los `DEFAULT PRIVILEGES`.

Tablas append-only (auditoría, outbox histórico): revocar `UPDATE, DELETE` a `proyecto_app` explícitamente.

## 3. Red y autenticación
- PostgreSQL solo en la red interna `data` (ART-09 §6); sin puerto publicado.
- `password_encryption = scram-sha-256`; `pg_hba.conf` solo `scram-sha-256` desde la subred de Docker; nada de `trust`.
- TLS entre app y BD si la BD está en otro servidor o en la nube (`sslmode=verify-full`).
- Acceso administrativo por túnel SSH/VPN, con usuarios nominales (no compartir `postgres`).

## 4. Row Level Security (defensa en profundidad, opcional por ADR)
Para proyectos multi-tenant con datos muy sensibles, además del filtro en repositorios:

```sql
ALTER TABLE app.facturas ENABLE ROW LEVEL SECURITY;
CREATE POLICY p_facturas_empresa ON app.facturas
  USING (empresa_id = current_setting('app.empresa_id')::uuid);
```
La aplicación hace `SET LOCAL app.empresa_id = '<id>'` al iniciar cada transacción. Implica pruebas adicionales y cuidado con el pool de conexiones (siempre `SET LOCAL`, nunca `SET`). El rol dueño ignora RLS salvo `FORCE ROW LEVEL SECURITY`.

## 5. Datos sensibles
- Cifrado de campos en la aplicación (ART-21 §6) para los datos que el brief marca como críticos.
- Cifrado en reposo del disco/volumen (LUKS o el del proveedor de nube).
- Backups cifrados (ART-37).
- Vistas para `readonly` que excluyen o enmascaran columnas sensibles.
- `log_parameter_max_length = 0` y `log_statement = 'ddl'` en producción para no registrar datos en logs de PostgreSQL.

## 6. Auditoría
- `log_connections = on`, `log_disconnections = on`.
- Opcional (ADR): extensión `pgaudit` para registrar DDL y accesos de roles administrativos.

## 7. Mantenimiento de seguridad
- Actualizaciones menores de PostgreSQL (parches) en máximo 1 mes; mayores planificadas.
- Rotación de contraseñas de roles cada 12 meses o ante sospecha (ART-35).
- Revisión trimestral de roles y permisos (`\du`, `\dp`).

## 8. Pruebas
- [ ] El rol `app` no puede ejecutar `CREATE TABLE`, `DROP`, `TRUNCATE`, `ALTER`.
- [ ] El rol `readonly` no puede escribir.
- [ ] Una consulta que excede `statement_timeout` se cancela y la API responde `SERVICIO_NO_DISPONIBLE`.
- [ ] PostgreSQL no es alcanzable desde fuera del servidor (escaneo de puertos).
- [ ] (Si hay RLS) Sin `SET LOCAL app.empresa_id`, la consulta no devuelve filas.
