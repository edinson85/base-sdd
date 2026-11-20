---
id: ART-36
titulo: Pipeline CI/CD
version: 1.0.0
estado: vigente
aplica_a: [infraestructura]
depende_de: [ART-06, ART-13, ART-33, ART-38]
---

# ART-36 · Pipeline CI/CD

> **Instrucción para agentes IA**
> Las etapas y puertas de §1 son obligatorias; la herramienta (Bitbucket Pipelines, GitHub Actions, GitLab CI) la define el proyecto. §3 da un ejemplo en Bitbucket Pipelines; tradúcelo si el proyecto usa otra.

---

## 1. Etapas y puertas

| # | Etapa | Cuándo | Herramientas | Falla si |
|---|---|---|---|---|
| 1 | Secretos | Todo push | gitleaks | Encuentra un secreto |
| 2 | Lint y formato | Todo push | ruff, ESLint, Prettier, dart analyze, hadolint, sqlfluff | Cualquier error |
| 3 | Tipos y fronteras | Todo push | mypy, tsc, import-linter, eslint-boundaries | Cualquier error |
| 4 | Pruebas unitarias | Todo push | pytest, Vitest, flutter test | Falla o cobertura < 80 % en domain/application |
| 5 | Pruebas de integración y API | Todo push | pytest + testcontainers (PostgreSQL, Redis) | Falla o presupuesto de consultas excedido |
| 6 | Seguridad estática | Todo push | bandit, semgrep, pip-audit, npm audit | Hallazgo alto/crítico |
| 7 | Migraciones | PR | `alembic upgrade head` + `downgrade -1` + `upgrade head` sobre BD vacía | Error |
| 8 | Build de imágenes | PR y main | docker buildx | Error |
| 9 | Escaneo de imágenes | PR y main | Trivy | Vulnerabilidad crítica (alta con excepción documentada) |
| 10 | Pruebas adversarias de API | PR y main | Schemathesis contra la API levantada con compose | Error 5xx, violación de esquema o fuga de datos |
| 11 | E2E | main y nocturno | Playwright contra compose completo | Falla |
| 12 | Despliegue a staging | main | compose/SSH o orquestador | — |
| 13 | DAST | Staging (nocturno) | OWASP ZAP baseline + API scan | Hallazgo alto |
| 14 | Carga | Antes de release | k6 | Umbrales (ART-43) |
| 15 | Despliegue a producción | Tag `vX.Y.Z` + aprobación manual | — | — |

Las etapas 1-6 deben tardar < 10 minutos en total (paralelizar).

## 2. Reglas
- `main` protegida: merge solo con etapas 1-11 en verde y aprobación.
- Las imágenes se construyen **una vez** y se promueven por ambientes con el mismo digest (no se reconstruye para producción).
- Tag de imagen = SHA del commit + tag semántico en releases.
- Los secretos del pipeline viven en las variables seguras de la herramienta, con acceso restringido al despliegue.
- Despliegue de producción: backup previo (ART-37), migraciones (job `migrate`), despliegue gradual de `api`, verificación de `/health/ready`, rollback automático si falla.
- Resultados (cobertura, reportes de seguridad, reportes E2E) se publican como artefactos del pipeline.

## 3. Ejemplo (Bitbucket Pipelines)

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
          - pipe: atlassian/git-secrets-scan:3.1.0   # o: docker run zricethezav/gitleaks detect
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
          - cd frontend && npm ci
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
            - docker compose -f docker-compose.ci.yml up -d --wait
            - docker compose -f docker-compose.ci.yml run --rm schemathesis
            - docker compose -f docker-compose.ci.yml run --rm playwright
      - step:
          name: Desplegar a staging
          deployment: staging
          script: [./infra/deploy.sh staging $BITBUCKET_COMMIT]
  tags:
    'v*':
      - step:
          name: Desplegar a producción
          deployment: production
          trigger: manual
          script: [./infra/deploy.sh production $BITBUCKET_TAG]
```

> Las versiones de *pipes* e imágenes del ejemplo deben verificarse y fijarse al crear el proyecto.
