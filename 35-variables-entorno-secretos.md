---
id: ART-35
titulo: Variables de entorno y gestión de secretos
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, infraestructura]
depende_de: [ART-02, ART-33]
---

# ART-35 · Variables de entorno y gestión de secretos

> **Instrucción para agentes IA**
> Nunca escribas un secreto real en código, documentación, pruebas, imágenes ni logs. Usa valores de ejemplo evidentes (`cambiar-esto`). Toda variable nueva se documenta en `.env.example` y en la tabla de §3 del plan técnico.

---

## 1. Clasificación

| Tipo | Ejemplos | Dónde vive | En el repositorio |
|---|---|---|---|
| Configuración no sensible | `APP_ENV`, `LOG_LEVEL`, `CORS_ORIGINS`, `ACCESS_TOKEN_TTL_SECONDS` | Variables de entorno | `.env.example` con valores por defecto |
| Secretos | Contraseñas de BD/Redis, llave privada JWT, API keys externas, llave de cifrado de campos | Docker secrets (archivos en `/run/secrets`) o gestor de secretos | Nunca; solo el nombre en `.env.example` |
| Configuración pública del front | `VITE_APP_NAME`, `VITE_SENTRY_DSN` | Build del front | Sí (son públicas por definición, P-SF-05) |

## 2. Lectura en el backend

```python
# bootstrap/settings.py
from pathlib import Path
from pydantic import SecretStr, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore", secrets_dir="/run/secrets")

    app_env: str = "development"
    database_host: str
    database_name: str
    database_user: str
    db_password: SecretStr                  # Lee /run/secrets/db_password
    redis_url_base: str                     # redis://redis:6379/0 (sin contraseña)
    redis_password: SecretStr
    jwt_private_key: SecretStr
    jwt_issuer: str
    access_token_ttl_seconds: int = 900
    cors_origins: list[str] = []

    @field_validator("cors_origins")
    @classmethod
    def sin_comodin(cls, v: list[str]) -> list[str]:
        if "*" in v:
            raise ValueError("CORS no admite '*' (P-SB-03)")
        return v
```

- `SecretStr` evita que el valor aparezca en `repr` y logs; se usa `.get_secret_value()` solo al construir el cliente.
- La app **falla al iniciar** si falta un secreto obligatorio (mejor que fallar en el primer request).
- En `production`, validaciones extra: `DEBUG` falso, docs deshabilitadas, orígenes CORS HTTPS.

## 3. `.env.example` (plantilla)

```dotenv
# --- App ---
APP_ENV=development
LOG_LEVEL=INFO
CORS_ORIGINS=["http://localhost:5173"]
ACCESS_TOKEN_TTL_SECONDS=900
REFRESH_TOKEN_TTL_DAYS=7
IDLE_TIMEOUT_MINUTES=30

# --- Base de datos ---
DATABASE_HOST=postgres
DATABASE_NAME=proyecto
DATABASE_USER=proyecto_app
# DB_PASSWORD -> secreto: /run/secrets/db_password (en dev: variable DB_PASSWORD=cambiar-esto)

# --- Redis ---
REDIS_URL_BASE=redis://redis:6379/0
# REDIS_PASSWORD -> secreto

# --- JWT ---
JWT_ISSUER=https://app.ejemplo.com
# JWT_PRIVATE_KEY -> secreto (PEM, Ed25519 o RSA 2048+)
```

## 4. Reglas

| # | Regla |
|---|---|
| S-1 | Un secreto distinto por ambiente; nunca reutilizar los de producción en otro ambiente |
| S-2 | Secretos generados con entropía suficiente (`openssl rand -base64 48`) |
| S-3 | Rotación: llaves JWT cada 6-12 meses (con `kid` para convivir dos llaves durante la transición), contraseñas de servicios cada 12 meses o ante sospecha |
| S-4 | Acceso a secretos de producción solo para quien opera; registrado |
| S-5 | `gitleaks` en pre-commit y en CI; si un secreto llega al repositorio, **se rota de inmediato** (borrar el commit no basta) |
| S-6 | Los secretos no se pasan por argumentos de línea de comandos (aparecen en `ps`) |
| S-7 | En el front y en la app móvil no existen secretos: cualquier llave embebida se considera pública |
| S-8 | Archivos de secretos en el servidor con permisos `0400`, dueño el usuario que ejecuta Docker |

## 5. Opciones de gestión según escala
| Escala | Opción |
|---|---|
| Pequeña | Docker secrets desde archivos en el servidor, respaldados cifrados (por ejemplo con `age` o `sops`) |
| Media | `sops` + llaves por ambiente en el repositorio de infraestructura, descifrado en el despliegue |
| Grande | Vault, AWS Secrets Manager, GCP Secret Manager o equivalente (ADR) |
