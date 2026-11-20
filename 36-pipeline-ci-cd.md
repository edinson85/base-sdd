---
id: ART-36
titulo: Pipeline CI/CD
version: 2.0.0   # 2.0.0: GitHub Actions como plataforma por defecto (ADR-BASE-13); Bitbucket pasa al Anexo A
estado: vigente
aplica_a: [infraestructura]
depende_de: [ART-06, ART-13, ART-33, ART-35, ART-37, ART-38, ART-50]
---

# ART-36 · Pipeline CI/CD

> **Instrucción para agentes IA**
> - Las etapas y puertas de §1 son obligatorias en cualquier plataforma.
> - La plataforma por defecto es **GitHub + GitHub Actions** (ADR-BASE-13). §4 tiene los workflows de referencia y §5 el script de despliegue en el servidor.
> - Si el proyecto usa Bitbucket (por ejemplo por exigencia del cliente), usa el Anexo A y registra el cambio con un ADR.
> - Versiones de *actions* e imágenes: fíjalas al crear el proyecto (idealmente por SHA de commit) y actualízalas con Dependabot.

---

## 1. Etapas y puertas

| # | Etapa | Cuándo | Herramientas | Falla si |
|---|---|---|---|---|
| 1 | Secretos | Todo push y PR | gitleaks | Encuentra un secreto |
| 2 | Lint y formato | Todo push y PR | ruff, ESLint, Prettier, dart analyze, hadolint, sqlfluff | Cualquier error |
| 3 | Tipos y fronteras | Todo push y PR | mypy, tsc, import-linter, eslint-boundaries | Cualquier error |
| 4 | Pruebas unitarias | Todo push y PR | pytest, Vitest, flutter test | Falla o cobertura < 80 % en domain/application |
| 5 | Pruebas de integración y API | Todo push y PR | pytest + testcontainers (PostgreSQL, Redis) | Falla o presupuesto de consultas excedido |
| 6 | Seguridad estática | Todo push y PR | bandit, semgrep, pip-audit, npm audit | Hallazgo alto/crítico |
| 7 | Migraciones | PR | `alembic upgrade head` + `downgrade -1` + `upgrade head` sobre BD vacía | Error |
| 8 | Build de imágenes | PR y main | docker buildx | Error |
| 9 | Escaneo de imágenes | PR y main | Trivy | Vulnerabilidad crítica (alta con excepción documentada) |
| 10 | Pruebas adversarias de API | PR y main | Schemathesis contra la API levantada con compose | Error 5xx, violación de esquema o fuga de datos |
| 11 | E2E | main y nocturno | Playwright contra compose completo | Falla |
| 12 | Despliegue a staging | main (automático) | SSH + `infra/deploy.sh` | Health check falla → rollback |
| 13 | DAST | Staging (nocturno) | OWASP ZAP baseline + API scan | Hallazgo alto |
| 14 | Carga | Antes de release | k6 | Umbrales (ART-43) |
| 15 | Despliegue a producción | **Ejecución manual** con la versión a desplegar | SSH + `infra/deploy.sh` | Health check falla → rollback |

Las etapas 1-6 deben tardar < 10 minutos en total (trabajos en paralelo).

## 2. Reglas
- `main` protegida: merge solo por PR con las etapas 1-11 en verde.
- Las imágenes se construyen **una vez** por commit y se **promueven** entre ambientes con el mismo digest; nunca se reconstruye para producción.
- Etiquetas de imagen: `sha-<commit>` siempre; `vX.Y.Z` al promover a producción.
- Despliegue de producción: backup previo (ART-37), migraciones (job `migrate`), actualización de servicios, verificación de `/health/ready` y rollback automático si falla.
- Despliegues serializados por ambiente (`concurrency`): nunca dos despliegues simultáneos al mismo servidor.
- Reportes (cobertura, seguridad, E2E) se publican como artefactos del workflow.

## 3. Configuración en GitHub

### 3.1 Protección de producción según el plan
| Plan del repositorio privado | Qué hay disponible | Cómo se protege producción |
|---|---|---|
| GitHub Free | Sin *environments* en repositorios privados | Workflow de producción **solo manual** (`workflow_dispatch`), que solo pueden ejecutar personas con permiso de escritura; secretos de producción como secretos del repositorio con prefijo `PROD_` |
| GitHub Pro / Team | *Environments* con secretos propios y restricción de ramas/tags | Lo anterior + environment `production` con sus secretos y despliegue permitido solo desde `main` |
| GitHub Enterprise (o repositorio público) | Además, *required reviewers* | Lo anterior + aprobación obligatoria de un revisor |

> Las aprobaciones obligatorias de despliegue (*required reviewers*) **no** están disponibles en repositorios privados con Free, Pro ni Team. Por eso la puerta de producción por defecto es la **ejecución manual**: nadie despliega a producción con un push.

### 3.2 Secretos y variables
| Nombre | Tipo | Uso |
|---|---|---|
| `STAGING_SSH_HOST`, `STAGING_SSH_USER` | Variable | Servidor de staging |
| `STAGING_SSH_KEY` | Secreto | Llave privada SSH de un usuario `deploy` sin contraseña, solo para despliegues |
| `STAGING_SSH_KNOWN_HOSTS` | Secreto | Salida de `ssh-keyscan <host>` verificada (evita ataques de intermediario) |
| `PROD_SSH_HOST`, `PROD_SSH_USER`, `PROD_SSH_KEY`, `PROD_SSH_KNOWN_HOSTS` | Igual para producción | |

- Los secretos de la **aplicación** (BD, JWT, Redis) **no** pasan por GitHub: viven en el servidor como Docker secrets (ART-35). GitHub solo guarda lo necesario para conectarse.
- El servidor descarga imágenes de GHCR con un token de solo lectura (`read:packages`) configurado una vez con `docker login ghcr.io` por el usuario `deploy`. Este token y el acceso SSH son **insumos** del Sprint 0 (ART-50).
- Usuario `deploy` en el servidor: miembro del grupo `docker`, sin sudo, con acceso solo a `/opt/<app>`.

### 3.3 Otras configuraciones del repositorio
- *Branch protection* o *ruleset* en `main`: PR obligatorio, checks requeridos (`ci`), sin push forzado.
- Dependabot para `pip`, `npm`, `pub`, `docker` y `github-actions` (archivo `.github/dependabot.yml`).
- Secret scanning y push protection activados.
- GHCR: paquetes privados, visibles solo para el repositorio.

## 4. Workflows de referencia

### 4.1 `.github/workflows/ci.yml` (PR y main; despliega a staging desde main)

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_API: ghcr.io/${{ github.repository }}-api
  IMAGE_WEB: ghcr.io/${{ github.repository }}-web

jobs:
  secretos:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  backend:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: backend } }
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with: { python-version: "3.12", enable-cache: true }
      - run: uv sync --frozen
      - run: uv run ruff check . && uv run ruff format --check .
      - run: uv run mypy app
      - run: uv run lint-imports
      - run: uv run bandit -q -r app -ll
      - run: uv run pip-audit
      - name: Migraciones (upgrade / downgrade / upgrade)
        run: uv run pytest tests/migrations -q
      - name: Pruebas (unitarias, integración con testcontainers, API, adversarias)
        run: uv run pytest --cov=app --cov-report=xml --cov-fail-under=80
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: cobertura-backend, path: backend/coverage.xml }

  frontend:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: frontend } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci --ignore-scripts
      - run: npm run lint && npm run typecheck
      - run: npm audit --audit-level=high
      - run: npm run test -- --coverage

  imagenes:
    needs: [secretos, backend, frontend]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        if: github.event_name == 'push'
        with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
      - name: Build API
        uses: docker/build-push-action@v6
        with:
          context: backend
          load: true
          tags: ${{ env.IMAGE_API }}:sha-${{ github.sha }}
          cache-from: type=gha,scope=api
          cache-to: type=gha,mode=max,scope=api
      - name: Build Web (React + Nginx)
        uses: docker/build-push-action@v6
        with:
          context: .
          file: frontend/Dockerfile
          load: true
          tags: ${{ env.IMAGE_WEB }}:sha-${{ github.sha }}
          cache-from: type=gha,scope=web
          cache-to: type=gha,mode=max,scope=web
      - name: Escaneo Trivy (API)
        uses: aquasecurity/trivy-action@SHA_FIJADO
        with: { image-ref: "${{ env.IMAGE_API }}:sha-${{ github.sha }}", severity: CRITICAL, exit-code: "1", ignore-unfixed: true }
      - name: Escaneo Trivy (Web)
        uses: aquasecurity/trivy-action@SHA_FIJADO
        with: { image-ref: "${{ env.IMAGE_WEB }}:sha-${{ github.sha }}", severity: CRITICAL, exit-code: "1", ignore-unfixed: true }
      - name: Publicar en GHCR (solo main)
        if: github.event_name == 'push'
        run: |
          docker push ${{ env.IMAGE_API }}:sha-${{ github.sha }}
          docker push ${{ env.IMAGE_WEB }}:sha-${{ github.sha }}

  e2e:
    needs: imagenes
    runs-on: ubuntu-latest
    env:
      VERSION: sha-${{ github.sha }}
    steps:
      - uses: actions/checkout@v4
      - name: Levantar sistema completo (compose e2e, ART-41)
        run: docker compose -f docker-compose.e2e.yml up -d --build --wait
      - name: Pruebas adversarias de API (Schemathesis)
        run: docker compose -f docker-compose.e2e.yml run --rm schemathesis
      - name: E2E (Playwright)
        run: docker compose -f docker-compose.e2e.yml run --rm playwright
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: reporte-e2e, path: tests/e2e/playwright-report }
      - if: always()
        run: docker compose -f docker-compose.e2e.yml down -v

  desplegar-staging:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: e2e
    runs-on: ubuntu-latest
    concurrency: { group: deploy-staging, cancel-in-progress: false }
    steps:
      - name: Configurar SSH
        run: |
          install -m 700 -d ~/.ssh
          echo "${{ secrets.STAGING_SSH_KEY }}" > ~/.ssh/id_deploy && chmod 600 ~/.ssh/id_deploy
          echo "${{ secrets.STAGING_SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
      - name: Desplegar
        run: ssh -i ~/.ssh/id_deploy ${{ vars.STAGING_SSH_USER }}@${{ vars.STAGING_SSH_HOST }} "/opt/app/infra/deploy.sh sha-${{ github.sha }}"
```

> En el job `e2e` del ejemplo el compose construye las imágenes de nuevo para simplificar; en proyectos con builds lentos, publica las imágenes en GHCR también desde PR (con etiqueta `pr-<n>`) y haz que el compose de E2E las descargue.

### 4.2 `.github/workflows/deploy-production.yml` (solo manual)

```yaml
name: deploy-production
on:
  workflow_dispatch:
    inputs:
      commit:
        description: "SHA completo del commit de main ya desplegado y validado en staging"
        required: true
      version:
        description: "Versión semántica a publicar (ej. v1.4.0)"
        required: true

permissions:
  contents: write     # Crear el tag de la versión
  packages: write     # Etiquetar imágenes existentes (sin reconstruir)

concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  desplegar:
    runs-on: ubuntu-latest
    # environment: production   # Activar si el plan lo permite (Pro/Team: secretos propios; Enterprise: aprobación)
    steps:
      - name: Validar entradas
        run: |
          [[ "${{ inputs.commit }}" =~ ^[0-9a-f]{40}$ ]] || { echo "SHA inválido"; exit 1; }
          [[ "${{ inputs.version }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || { echo "Versión inválida"; exit 1; }
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Verificar que el commit está en main
        run: git merge-base --is-ancestor ${{ inputs.commit }} origin/main
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
      - name: Promover imágenes (mismo digest, nueva etiqueta)
        run: |
          for img in api web; do
            docker buildx imagetools create \
              -t ghcr.io/${{ github.repository }}-$img:${{ inputs.version }} \
              ghcr.io/${{ github.repository }}-$img:sha-${{ inputs.commit }}
          done
      - name: Configurar SSH
        run: |
          install -m 700 -d ~/.ssh
          echo "${{ secrets.PROD_SSH_KEY }}" > ~/.ssh/id_deploy && chmod 600 ~/.ssh/id_deploy
          echo "${{ secrets.PROD_SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
      - name: Desplegar
        run: ssh -i ~/.ssh/id_deploy ${{ vars.PROD_SSH_USER }}@${{ vars.PROD_SSH_HOST }} "/opt/app/infra/deploy.sh ${{ inputs.version }}"
      - name: Crear tag de la versión
        run: |
          git tag ${{ inputs.version }} ${{ inputs.commit }}
          git push origin ${{ inputs.version }}
```

### 4.3 Otros workflows
| Archivo | Disparador | Contenido |
|---|---|---|
| `nightly.yml` | `schedule` (diario, fuera de horario laboral) | E2E completo, ZAP baseline + API scan contra staging, `pip-audit`/`npm audit` |
| `load-test.yml` | Manual | k6 contra staging (ART-43) |
| `.github/dependabot.yml` | — | Actualizaciones semanales agrupadas por ecosistema |

## 5. Script de despliegue en el servidor (`infra/deploy.sh`)

```bash
#!/usr/bin/env bash
# Uso: deploy.sh <etiqueta-de-imagen>   (ej. sha-3f2c... o v1.4.0)
set -euo pipefail
VERSION="${1:?Falta la versión}"
[[ "$VERSION" =~ ^(sha-[0-9a-f]{40}|v[0-9]+\.[0-9]+\.[0-9]+)$ ]] || { echo "Versión inválida"; exit 1; }

cd /opt/app
COMPOSE="docker compose -f docker-compose.prod.yml"
ANTERIOR="$(cat .version 2>/dev/null || echo '')"

exec 9>/opt/app/.deploy.lock
flock -n 9 || { echo "Hay otro despliegue en curso"; exit 1; }

echo "1/5 Backup previo"
./infra/backup/backup.sh pre-deploy-"$VERSION"

echo "2/5 Descargar imágenes $VERSION"
export VERSION
$COMPOSE pull

echo "3/5 Migraciones"
$COMPOSE run --rm migrate

echo "4/5 Actualizar servicios"
$COMPOSE up -d --remove-orphans

echo "5/5 Verificar salud"
for i in $(seq 1 30); do
  if $COMPOSE exec -T api python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health/ready', timeout=2).status==200 else 1)"; then
    echo "$VERSION" > .version
    echo "Despliegue OK: $VERSION"
    exit 0
  fi
  sleep 2
done

echo "Health check falló. Rollback a ${ANTERIOR:-<ninguna>}"
if [[ -n "$ANTERIOR" ]]; then
  VERSION="$ANTERIOR" $COMPOSE up -d --remove-orphans
fi
exit 1
```

Notas:
- `docker-compose.prod.yml` usa `image: ghcr.io/<owner>/<repo>-api:${VERSION}` (ART-33).
- El rollback devuelve los contenedores a la versión anterior; las migraciones son compatibles hacia atrás por el patrón expand/contract (ART-24), así que no se revierte el esquema.
- El script vive en el repositorio y se actualiza en el servidor con `git pull` del usuario `deploy` (o se copia en cada despliegue); nunca contiene secretos.

---

## Anexo A · Equivalente en Bitbucket Pipelines

Usar solo si el proyecto exige Bitbucket (registrar ADR). El ejemplo omite la publicación de imágenes: antes de desplegar, agrega el `docker login` y `docker push` al registro elegido (GHCR, Docker Hub u otro) con la etiqueta `sha-<commit>`, porque el servidor las descarga de ahí. Las etapas y reglas de §1 y §2 se mantienen; el despliegue usa el mismo `infra/deploy.sh`. En Bitbucket las variables seguras se definen por *deployment* (`staging`, `production`) y el despliegue a producción se marca `trigger: manual`. Revisa el cupo de minutos del plan: en el plan gratuito es muy bajo para este pipeline.

```yaml
image: atlassian/default-image:4

definitions:
  services:
    docker: { memory: 3072 }
  caches:
    uv: ~/.cache/uv
    npm-cache: ~/.npm
  steps:
    - step: &secrets
        name: Secretos
        script:
          - docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:latest detect --source /repo   # fijar versión
    - step: &backend
        name: Backend - lint, tipos, pruebas
        image: python:3.12-slim
        caches: [uv]
        services: [docker]
        script:
          - pip install uv && cd backend && uv sync --frozen
          - uv run ruff check . && uv run ruff format --check .
          - uv run mypy app
          - uv run lint-imports
          - uv run bandit -q -r app -ll
          - uv run pip-audit
          - uv run pytest --cov=app --cov-report=xml --cov-fail-under=80
        artifacts: [backend/coverage.xml]
    - step: &frontend
        name: Frontend - lint, tipos, pruebas
        image: node:22-alpine
        caches: [npm-cache]
        script:
          - cd frontend && npm ci --ignore-scripts
          - npm run lint && npm run typecheck
          - npm audit --audit-level=high
          - npm run test -- --coverage
    - step: &images
        name: Build y escaneo de imágenes
        services: [docker]
        script:
          - docker build -t api:$BITBUCKET_COMMIT backend
          - docker build -t web:$BITBUCKET_COMMIT -f frontend/Dockerfile .
          - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity CRITICAL api:$BITBUCKET_COMMIT
          - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity CRITICAL web:$BITBUCKET_COMMIT

pipelines:
  pull-requests:
    '**':
      - step: *secrets
      - parallel:
          - step: *backend
          - step: *frontend
      - step: *images
  branches:
    main:
      - step: *secrets
      - parallel:
          - step: *backend
          - step: *frontend
      - step: *images
      - step:
          name: E2E y adversarias de API
          services: [docker]
          script:
            - docker compose -f docker-compose.e2e.yml up -d --build --wait
            - docker compose -f docker-compose.e2e.yml run --rm schemathesis
            - docker compose -f docker-compose.e2e.yml run --rm playwright
      - step:
          name: Desplegar a staging
          deployment: staging
          script: [ssh $SSH_USER@$SSH_HOST "/opt/app/infra/deploy.sh sha-$BITBUCKET_COMMIT"]
      - step:
          name: Desplegar a producción
          deployment: production
          trigger: manual
          script: [ssh $SSH_USER@$SSH_HOST "/opt/app/infra/deploy.sh sha-$BITBUCKET_COMMIT"]
```
