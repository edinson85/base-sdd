---
id: ART-37
titulo: Estrategia de ambientes, despliegue y backups
version: 1.0.0
estado: vigente
aplica_a: [infraestructura, base-de-datos]
depende_de: [ART-09, ART-33, ART-35, ART-36]
---

# ART-37 · Ambientes, despliegue y backups

> **Instrucción para agentes IA**
> En el plan técnico, ajusta RPO/RTO y frecuencias según el brief (§9). Por defecto: RPO 24 h, RTO 4 h.

---

## 1. Ambientes

| Ambiente | Propósito | Datos | Acceso | Despliegue |
|---|---|---|---|---|
| `local` | Desarrollo | Semillas y datos sintéticos | Desarrollador | `docker compose up` |
| `ci` | Pruebas automáticas | Efímeros por ejecución | Pipeline | Automático |
| `staging` | Validación de negocio, E2E, DAST, carga | Sintéticos o anonimizados | Equipo | Automático desde `main` |
| `production` | Usuarios reales | Reales | Restringido | Tag + aprobación manual |

Reglas:
- **Nunca** se copian datos de producción a otros ambientes sin anonimizar (Ley 1581 de 2012). La anonimización se hace con un script versionado que reemplaza nombres, documentos, correos, teléfonos y direcciones.
- Los ambientes tienen dominios, secretos, llaves JWT y bases de datos distintos.
- `staging` replica la configuración de producción (mismas imágenes, Nginx, TLS); solo cambian el tamaño y los datos.

## 2. Despliegue

1. Verificar backup reciente (≤ 24 h) y hacer uno adicional antes de migraciones destructivas.
2. `docker compose pull` de las imágenes del tag.
3. Ejecutar `migrate` (las migraciones deben ser compatibles con la versión anterior de la API, patrón expand/contract, ART-24).
4. Actualizar `api` y `worker` réplica por réplica; esperar `/health/ready`.
5. Actualizar `nginx` (nuevo build del front).
6. Pruebas de humo automáticas (login, un listado, un WebSocket).
7. Si falla: volver al tag anterior; las migraciones expand no requieren rollback de esquema.

## 3. Backups de PostgreSQL

| Tipo | Herramienta | Frecuencia | Retención |
|---|---|---|---|
| Lógico completo | `pg_dump -Fc` | Diario | 7 diarios, 4 semanales, 12 mensuales |
| Físico + WAL (recuperación a un punto en el tiempo) | pgBackRest o WAL-G | Base semanal + WAL continuo | 14 días | 

- Proyectos pequeños: `pg_dump` diario es suficiente si el RPO acordado es 24 h. Si el RPO es menor, se requiere WAL continuo (ADR).
- Regla 3-2-1: 3 copias, 2 medios distintos, 1 fuera del servidor (almacenamiento de objetos en otra región/proveedor).
- Backups **cifrados** (pgBackRest con cifrado o `age`/`gpg` sobre el dump) y con acceso restringido.
- Script de backup versionado en `infra/backup/` y ejecutado por cron o contenedor programado; alerta si falla.

### Prueba de restauración (obligatoria)
- Mensual: restaurar el último backup en un servidor aislado, ejecutar `alembic current`, conteos de tablas principales y una consulta de verificación.
- Registrar resultado y tiempo de restauración (valida el RTO).
- Un backup que no se ha probado restaurar **no cuenta** como backup.

## 4. Redis
- Es caché y coordinación: **no** se respalda como fuente de verdad. Ningún dato que solo exista en Redis puede ser crítico.
- Excepción: si se usa Redis para colas con trabajos no reconstruibles, activar AOF y documentar con ADR; preferir siempre el Outbox en PostgreSQL.

## 5. Archivos subidos
- Almacenamiento fuera de los contenedores (volumen dedicado u objeto S3 compatible).
- Backup diario incremental con la misma política 3-2-1.

## 6. Monitoreo mínimo del ambiente productivo
- Disponibilidad externa de `/` y chequeo sintético (login de un usuario de monitoreo sin permisos + un listado). `/health/*` solo se consulta desde la red interna (ART-34).
- Uso de disco, CPU y memoria del servidor; tamaño de la BD; conexiones activas.
- Alertas: backup fallido, certificado TLS a < 15 días de vencer, tasa de 5xx > 1 %, latencia p95 por encima del SLO, cola de Outbox con eventos `failed`.
