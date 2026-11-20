---
id: ART-20
titulo: Autenticación y autorización (RBAC)
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, base-de-datos]
depende_de: [ART-02, ART-10, ART-16, ART-19]
---

# ART-20 · Autenticación y autorización (RBAC)

> **Instrucción para agentes IA**
> El módulo `auth` es el primero que se construye en todo proyecto (sprint 0, ART-44). Implementa este documento completo; ajusta solo lo que el brief indique (MFA obligatorio, SSO, etc.) con ADR. Principios P-AU-01..06.

---

## 1. Modelo de datos

```sql
CREATE TABLE empresas (
  id uuid PRIMARY KEY, nombre text NOT NULL, activo boolean NOT NULL DEFAULT true,
  creado_en timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE usuarios (
  id uuid PRIMARY KEY,
  empresa_id uuid NOT NULL REFERENCES empresas(id),
  email text NOT NULL,
  nombre text NOT NULL,
  password_hash text NOT NULL,
  activo boolean NOT NULL DEFAULT true,
  intentos_fallidos int NOT NULL DEFAULT 0,
  bloqueado_hasta timestamptz,
  mfa_secret_cifrado bytea,
  password_cambiado_en timestamptz NOT NULL DEFAULT now(),
  creado_en timestamptz NOT NULL DEFAULT now(),
  actualizado_en timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_usuarios_email ON usuarios (lower(email));

CREATE TABLE permisos (
  codigo text PRIMARY KEY CHECK (codigo ~ '^[a-z_]+:[a-z_]+$'),   -- factura:crear
  descripcion text NOT NULL,
  modulo text NOT NULL
);

CREATE TABLE perfiles (
  id uuid PRIMARY KEY,
  empresa_id uuid REFERENCES empresas(id),     -- NULL = perfil de sistema
  nombre text NOT NULL,
  es_sistema boolean NOT NULL DEFAULT false,
  UNIQUE (empresa_id, nombre)
);

CREATE TABLE perfil_permisos (
  perfil_id uuid NOT NULL REFERENCES perfiles(id) ON DELETE CASCADE,
  permiso_codigo text NOT NULL REFERENCES permisos(codigo),
  PRIMARY KEY (perfil_id, permiso_codigo)
);

CREATE TABLE usuario_perfiles (
  usuario_id uuid NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
  perfil_id uuid NOT NULL REFERENCES perfiles(id),
  PRIMARY KEY (usuario_id, perfil_id)
);

CREATE TABLE refresh_tokens (
  id uuid PRIMARY KEY,
  usuario_id uuid NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
  familia_id uuid NOT NULL,
  token_hash bytea NOT NULL UNIQUE,          -- SHA-256 del token; nunca el token
  cliente text NOT NULL CHECK (cliente IN ('WEB','MOVIL')),
  expira_en timestamptz NOT NULL,
  usado_en timestamptz,
  revocado_en timestamptz,
  ip inet, user_agent text,
  creado_en timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_refresh_familia ON refresh_tokens (familia_id);
CREATE INDEX ix_refresh_usuario_activos ON refresh_tokens (usuario_id) WHERE revocado_en IS NULL;
```

Permisos: se insertan por migración de semillas cuando se crea un módulo (ART-24). Perfiles de sistema por defecto (ajustables en el brief): `Administrador` (todos los permisos de la empresa), `Consulta` (solo `:ver`).

## 2. Tokens

| Token | Formato | Vida | Dónde |
|---|---|---|---|
| Access | JWT firmado (EdDSA Ed25519 o RS256), `kid` en cabecera | 15 min (P-AU-04) | Memoria del cliente |
| Refresh | Aleatorio opaco de 256 bits (`secrets.token_urlsafe(32)`) | 7 días (configurable); absoluto 30 días por familia | Cookie HttpOnly (web) / secure storage (móvil); en BD solo su hash |
| Ticket WS | Aleatorio opaco | 30 s, un solo uso | Redis `ws-ticket:<hash>` → user_id |
| Reset de contraseña | Aleatorio opaco | 30 min, un solo uso | BD (hash) |

Claims del access token: `sub` (user_id), `emp` (empresa_id), `iss`, `aud`, `iat`, `exp`, `jti`, `ver` (versión de permisos del usuario). **No** incluir permisos ni datos personales en el token: se cargan desde caché por perfil.

Validación: algoritmo fijo (lista con un solo valor), firma, `exp`, `iss`, `aud`, `jti` no revocado (lista de revocación en Redis hasta su `exp`, usada en logout y bloqueo).

## 3. Flujos

### 3.1 Login
1. `POST /api/v1/auth/login {email, password}` (rate limit estricto por IP y por email).
2. Buscar usuario por `lower(email)`. Si no existe, **igual** se ejecuta una verificación Argon2 contra un hash ficticio (tiempo constante, evita enumeración).
3. Si `bloqueado_hasta > now()` → `CUENTA_BLOQUEADA`.
4. Verificar Argon2id; si falla: `intentos_fallidos += 1`; al llegar a 5 → bloqueo 15 min (exponencial en bloqueos sucesivos); respuesta `CREDENCIALES_INVALIDAS` (mismo mensaje para usuario inexistente o contraseña errada).
5. Si el usuario tiene MFA: responder `{mfa_requerido: true, mfa_token}` (token de 5 min) y completar con `POST /auth/mfa`.
6. Éxito: resetear intentos; crear familia de refresh; responder access token y refresh (cookie o body según cliente); `audit: login_succeeded`.
7. Rehash si los parámetros de Argon2 cambiaron (`needs_rehash`).

### 3.2 Refresh con rotación
1. Web: `POST /auth/refresh` con cookie + encabezado `X-Requested-With: fetch` (anti-CSRF). Móvil: `POST /auth/mobile/refresh {refresh_token}`.
2. Buscar por hash. Si no existe → 401.
3. Si ya fue **usado** o revocado → **reutilización detectada**: revocar toda la familia, `audit: token_reuse_detected`, 401.
4. Si expiró → 401.
5. Marcar `usado_en`, emitir un nuevo refresh en la misma familia y un nuevo access token.
6. Tolerancia: si dos pestañas refrescan a la vez, el cliente web serializa el refresh (ART-11 §5.1); no se agrega período de gracia salvo ADR.

### 3.3 Logout
`POST /auth/logout`: revoca la familia actual, agrega el `jti` del access a la lista de revocación, publica en `user:{id}:control` para cerrar WebSockets (ART-10 §6.9), borra la cookie (`Max-Age=0`) y responde con `Clear-Site-Data: "cache", "storage"`.

"Cerrar todas las sesiones": revoca todas las familias del usuario.

### 3.4 Recuperación de contraseña
- `POST /auth/password/forgot {email}`: siempre responde 202 con el mismo mensaje ("Si el correo está registrado, recibirás instrucciones"); rate limit por IP y email.
- Enlace con token de un solo uso (30 min). `POST /auth/password/reset {token, nueva}`: valida política (ART-19), revoca todas las sesiones, `audit: password_changed`.

### 3.5 Cambio de contraseña autenticado
Requiere contraseña actual; revoca las demás sesiones.

## 4. Autorización

### 4.1 Carga de permisos
`get_current_actor` → valida JWT → obtiene perfiles del usuario y permisos efectivos desde caché `app:auth:permisos:v1:usr:{id}:{ver}`; si no está, 1 consulta con joins y guarda (TTL 15 min).

Al cambiar perfiles o permisos: evento `PermisosDeUsuarioCambiados` / `PermisosDePerfilCambiados` → observer incrementa `ver` del usuario (o de todos los usuarios del perfil), invalida caché y publica en `user:{id}:control` (el front recarga `/auth/me`; si perdió acceso a la pantalla actual, redirige).

### 4.2 Verificación
1. Router: `require_permission("<recurso>:<accion>")` (ART-10 §6.8).
2. Caso de uso: `actor.exigir(...)` (defensa en profundidad).
3. Alcance: todo repositorio y query service filtra por `empresa_id` del actor; los recursos de propiedad individual también por `usuario_id` cuando la regla lo dice (P-AU-03).
4. Permisos de campo: si un perfil no puede ver ciertos campos (ej. costo), el caso de uso elige el read model reducido; el esquema de salida no incluye el campo.

### 4.3 Administración de perfiles
Permisos `perfil:ver`, `perfil:gestionar`, `usuario:gestionar`. Reglas:
- Un usuario no puede asignarse perfiles a sí mismo ni otorgar permisos que no tiene.
- Los perfiles de sistema no se eliminan.
- Siempre debe quedar al menos un administrador activo por empresa.
- Todo cambio se audita (`permissions_changed`).

## 5. WebSocket
`POST /api/v1/auth/ws-ticket` (autenticado) → `{ticket}`; el cliente abre `wss://host/ws/v1/<recurso>?ticket=...`; el servidor hace `GETDEL` en Redis (un solo uso), valida `Origin` y permisos (ART-09 §3.2).

## 6. Diferencias web / móvil
Ver ART-09 §4. Web: cookie `__Host-` no aplica porque se usa `Path=/api/v1/auth`; nombre `rt`, atributos `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`.

## 7. Contraseñas (Argon2id)
`argon2-cffi` con `PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)` como punto de partida; ajustar para que el hash tome 200-500 ms en el servidor de producción y registrar en ADR.

## 8. MFA (TOTP)
- `pyotp`; secreto cifrado en BD con la llave de cifrado de campos (ART-21 §6).
- Obligatorio para perfiles con permisos administrativos cuando el brief lo indique; códigos de recuperación de un solo uso (hash).
- Rate limit en verificación de códigos.

## 9. Pruebas obligatorias del módulo auth (además de ART-42)
- [ ] Login correcto, contraseña errada, usuario inexistente (mismo mensaje y tiempo similar).
- [ ] Bloqueo tras 5 intentos y desbloqueo por tiempo.
- [ ] Refresh rota el token; reutilizar un refresh revoca la familia.
- [ ] Access token: expirado, firma alterada, `alg: none`, `kid` desconocido, `aud` distinto → 401.
- [ ] Logout revoca refresh y access (`jti`) y cierra WS.
- [ ] Cambio de permisos se refleja en el siguiente request.
- [ ] Recuperación: no revela si el correo existe; token de un solo uso y con expiración.
- [ ] Un usuario no puede escalar sus propios permisos.
