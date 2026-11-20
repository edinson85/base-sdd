---
id: ART-13
titulo: Convenciones de código y Git
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, infraestructura]
depende_de: [ART-02, ART-08]
---

# ART-13 · Convenciones de código y Git

> **Instrucción para agentes IA**
> Aplica estas convenciones a todo archivo que crees. Si una herramienta de formato las cubre, deja que la herramienta decida (no formatees a mano en contra de ella).

---

## 1. Idioma de los identificadores

| Elemento | Idioma | Ejemplo |
|---|---|---|
| Conceptos del negocio (entidades, campos, eventos, casos de uso, permisos) | **Español**, sin tildes ni ñ | `Factura`, `numero`, `anio`, `FacturaAnulada`, `CrearFactura` |
| Patrones y sufijos técnicos | Inglés | `FacturaRepository`, `FacturaQueryService`, `use_cases`, `router`, `dto` |
| Palabras clave de framework | Lo que exija el framework | `execute`, `__call__`, `useQuery` |
| Comentarios y docstrings | Español | |
| Mensajes al usuario | Español con tildes | "La factura ya fue anulada." |
| Logs (nombre del evento) | Inglés snake_case | `factura_created`, `access_denied` |

## 2. Nombres por lenguaje

| Elemento | Python | TypeScript | Dart |
|---|---|---|---|
| Archivos | `snake_case.py` | `kebab-case.ts`; componentes `PascalCase.tsx` | `snake_case.dart` |
| Clases / tipos | `PascalCase` | `PascalCase` | `PascalCase` |
| Funciones / variables | `snake_case` | `camelCase` | `camelCase` |
| Constantes | `UPPER_SNAKE` | `UPPER_SNAKE` | `camelCase` (convención Dart) |
| Hooks | — | `useAlgo` | — |
| Pruebas | `test_<comportamiento>.py`, función `test_<cuando>_<entonces>` | `<archivo>.test.ts(x)` | `<archivo>_test.dart` |
| Booleanos | `es_`, `tiene_`, `puede_` | `es`, `tiene`, `puede` | `es`, `tiene`, `puede` |

## 3. Reglas de código

- Funciones ≤ 40 líneas; archivos ≤ 400 líneas (dividir si se excede).
- Tipado estricto: `mypy --strict` en `domain` y `application`; `strict: true` en TS; `strict-casts`, `strict-raw-types` en Dart.
- Sin números ni strings mágicos: constantes con nombre.
- Fechas siempre con zona horaria (UTC en back; se convierten a `America/Bogota` solo para mostrar).
- Dinero: `Decimal` (Python), `string` en JSON, formateo en el front con `Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP' })`.
- Sin dependencias nuevas sin justificar en el PR (y ADR si es estructural).

## 4. Herramientas

| Lenguaje | Formato | Lint | Tipos | Fronteras |
|---|---|---|---|---|
| Python | `ruff format` | `ruff` (reglas E, F, I, B, S, UP, N, SIM, ASYNC) | `mypy` | `import-linter` |
| TypeScript | Prettier | ESLint (typescript-eslint strict, react-hooks, jsx-a11y, security) | `tsc --noEmit` | `eslint-plugin-boundaries` |
| Dart | `dart format` | `very_good_analysis` o `flutter_lints` + reglas propias | analizador | `custom_lint` |
| SQL | `sqlfluff` (dialecto postgres) | `sqlfluff` | — | — |
| Docker | — | `hadolint` | — | — |
| YAML/MD | Prettier | `markdownlint` | — | — |

Todo esto corre en **pre-commit** y en CI (ART-36).

## 5. Git

### 5.1 Ramas (trunk-based ligero)
| Rama | Uso |
|---|---|
| `main` | Siempre desplegable; protegida; solo por PR |
| `feature/HU-NNN-descripcion-corta` | Trabajo de una HU o tarea |
| `fix/<ticket>-descripcion` | Correcciones |
| `hotfix/<ticket>-descripcion` | Correcciones urgentes en producción |
| `release/vX.Y.Z` | Opcional, si el proyecto necesita estabilización |

Ramas de vida corta (≤ 3 días). Rebase sobre `main` antes del merge.

### 5.2 Commits (Conventional Commits)
```
<tipo>(<modulo>): <descripción en imperativo, minúscula, ≤ 72 caracteres>

[cuerpo opcional: qué y por qué]

Refs: HU-012, T-003
```
Tipos: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `security`, `ci`, `build`.

Ejemplo: `feat(factura): crear endpoint POST /facturas con validación de cliente`

### 5.3 Pull requests
- Título = commit principal. Descripción con la plantilla de ART-48.
- ≤ 400 líneas cambiadas (sin contar archivos generados ni migraciones automáticas).
- Al menos 1 aprobación; CI en verde; sin conversaciones abiertas.
- Squash merge para mantener historia lineal.

### 5.4 Versionado
SemVer para releases (`vMAJOR.MINOR.PATCH`) con tags en `main`. El API versiona por URL (`/api/v1`), independiente del release.

## 6. Archivos que nunca se versionan
`.env*` (excepto `.env.example`), llaves, certificados, dumps de BD, `node_modules/`, `.venv/`, `build/`, `dist/`, `coverage/`, archivos del IDE. `.gitignore` base en la plantilla del repositorio y `gitleaks` en pre-commit.
