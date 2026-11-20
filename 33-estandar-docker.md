---
id: ART-33
titulo: Estándar Docker
version: 1.0.0
estado: vigente
aplica_a: [infraestructura]
depende_de: [ART-02, ART-09, ART-35]
---

# ART-33 · Estándar Docker

> **Instrucción para agentes IA**
> Usa estos Dockerfiles y compose como plantilla. Fija versiones exactas de imágenes base al crear el proyecto (por ejemplo `python:3.12.x-slim`) y actualízalas de forma controlada. Principios: P-SD-01..04.

---

## 1. Reglas

| # | Regla |
|---|---|
| D-1 | Imágenes multi-stage: dependencias de build no llegan a la imagen final |
| D-2 | Base mínima y versión fijada (nunca `latest`) |
| D-3 | Usuario no root (`USER app`, UID ≥ 10000) |
| D-4 | Sin secretos en `ARG`, `ENV` ni capas; se inyectan en ejecución (ART-35) |
| D-5 | `.dockerignore` que excluye `.git`, `.env*`, pruebas, `node_modules`, cachés |
| D-6 | `HEALTHCHECK` o healthcheck en compose para todo servicio |
| D-7 | Contenedores con `read_only: true`, `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]` salvo justificación |
| D-8 | Límites de memoria y CPU en producción |
| D-9 | Solo `nginx` publica puertos (P-SD-03) |
| D-10 | Escaneo de imágenes en CI (Trivy o Grype); sin vulnerabilidades críticas |
| D-11 | Logs a stdout/stderr (no archivos dentro del contenedor) |

---

## 2. Backend (`backend/Dockerfile`)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1

FROM base AS builder
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /build
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv export --frozen --no-dev -o requirements.txt \
    && pip wheel --wheel-dir /wheels -r requirements.txt

FROM base AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends libmagic1 \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --gid 10001 app && useradd --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-index --find-links=/wheels /wheels/* && rm -rf /wheels
COPY --chown=app:app app ./app
COPY --chown=app:app migrations ./migrations
COPY --chown=app:app alembic.ini ./
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health/live', timeout=2).status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--proxy-headers", "--forwarded-allow-ips", "*", "--no-server-header"]
```

> `--forwarded-allow-ips "*"` es aceptable **solo** porque la API no publica puertos y únicamente Nginx le habla desde la red interna. Si cambia la topología, restringir a la IP/red de Nginx.

En producción con varias CPU: `gunicorn -k uvicorn.workers.UvicornWorker -w <2×CPU> ...` o varias réplicas del contenedor.

## 3. Frontend + Nginx (`frontend/Dockerfile`)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts
COPY . .
RUN npm run build          # Genera /src/dist sin source maps públicos

FROM nginxinc/nginx-unprivileged:1.27-alpine AS runtime
COPY --from=build /src/dist /usr/share/nginx/html
COPY infra/nginx/nginx.conf /etc/nginx/nginx.conf
COPY infra/nginx/conf.d/ /etc/nginx/conf.d/
COPY infra/nginx/snippets/ /etc/nginx/snippets/
EXPOSE 8080 8443
```

La imagen sin privilegios escucha en 8080/8443; el compose mapea 80→8080 y 443→8443.

## 4. Compose de producción (`docker-compose.prod.yml`)

```yaml
x-hardening: &hardening
  read_only: true
  cap_drop: [ALL]
  security_opt: ["no-new-privileges:true"]
  restart: unless-stopped
  logging:
    driver: json-file
    options: { max-size: "20m", max-file: "5" }

services:
  nginx:
    image: ${REGISTRY}/proyecto-web:${VERSION}
    <<: *hardening
    ports: ["80:8080", "443:8443"]
    tmpfs: ["/tmp", "/var/cache/nginx"]
    volumes: ["./certs:/etc/nginx/certs:ro"]
    depends_on: { api: { condition: service_healthy } }
    networks: [edge, backend]

  api:
    image: ${REGISTRY}/proyecto-api:${VERSION}
    <<: *hardening
    env_file: .env.api
    secrets: [db_password, jwt_private_key, redis_password]
    tmpfs: ["/tmp"]
    depends_on:
      migrate: { condition: service_completed_successfully }
      redis: { condition: service_healthy }
    deploy: { replicas: 2, resources: { limits: { memory: 512m, cpus: "1.0" } } }
    networks: [backend, data]

  worker:
    image: ${REGISTRY}/proyecto-api:${VERSION}
    <<: *hardening
    command: ["taskiq", "worker", "app.bootstrap.worker:broker"]
    env_file: .env.api
    secrets: [db_password, redis_password]
    tmpfs: ["/tmp"]
    healthcheck: { disable: true }   # Reemplazar por un healthcheck propio del worker si se expone
    networks: [data]

  migrate:
    image: ${REGISTRY}/proyecto-api:${VERSION}
    command: ["alembic", "upgrade", "head"]
    env_file: .env.migrate             # Usuario de BD con permisos DDL (ART-26)
    secrets: [db_migrator_password]
    restart: "no"
    networks: [data]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: proyecto
      POSTGRES_USER: postgres                 # Superusuario solo para administración (ART-26 §1)
      POSTGRES_PASSWORD_FILE: /run/secrets/db_superuser_password
    secrets: [db_superuser_password, db_migrator_password, db_password, db_readonly_password]
    volumes: ["pgdata:/var/lib/postgresql/data", "./infra/postgres/init:/docker-entrypoint-initdb.d:ro"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U postgres -d proyecto"], interval: 10s, retries: 5 }
    shm_size: 256m
    restart: unless-stopped
    networks: [data]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes: ["./infra/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro"]
    healthcheck: { test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "$$REDIS_PASSWORD", "ping"], interval: 10s }
    restart: unless-stopped
    networks: [data]

networks:
  edge: {}
  backend: { internal: true }
  data: { internal: true }

volumes:
  pgdata: {}

secrets:
  db_password: { file: ./secrets/db_password }
  db_superuser_password: { file: ./secrets/db_superuser_password }
  db_readonly_password: { file: ./secrets/db_readonly_password }
  db_migrator_password: { file: ./secrets/db_migrator_password }
  jwt_private_key: { file: ./secrets/jwt_private_key.pem }
  redis_password: { file: ./secrets/redis_password }
```

Notas:
- `deploy.replicas` lo respeta Docker Compose v2 y Swarm; en un solo host, si no aplica, escalar con `docker compose up --scale api=2`.
- El healthcheck de Redis necesita la variable `REDIS_PASSWORD` en el contenedor o un script que lea el secreto; ajústalo según cómo se configure la ACL (ART-16 §6).

## 5. Compose de desarrollo (`docker-compose.yml`)

- Monta el código como volumen con recarga (`uvicorn --reload`, `vite` dev server).
- Publica PostgreSQL y Redis **solo en `127.0.0.1`** (`"127.0.0.1:5432:5432"`) para herramientas locales.
- Variables desde `.env` local (copiado de `.env.example`), nunca credenciales reales.
- Incluye servicios de apoyo opcionales: `mailpit` (correo de prueba), `adminer` solo en `127.0.0.1`.

## 6. Checklist
- [ ] Multi-stage, versión fijada, usuario no root.
- [ ] `.dockerignore` completo; `hadolint` sin errores.
- [ ] Sin secretos en la imagen (`docker history` limpio).
- [ ] Solo Nginx con puertos públicos; redes `internal`.
- [ ] Healthchecks, límites de recursos y rotación de logs.
- [ ] Escaneo Trivy sin críticos.
