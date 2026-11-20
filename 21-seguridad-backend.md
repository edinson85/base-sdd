---
id: ART-21
titulo: Seguridad backend
version: 1.0.0
estado: vigente
aplica_a: [backend, infraestructura]
depende_de: [ART-02, ART-10, ART-19, ART-20, ART-34]
---

# ART-21 · Seguridad backend

> **Instrucción para agentes IA**
> Asume un atacante activo y automatizado. Para cada módulo, revisa las amenazas de §1 y los controles de §2; los endpoints completan la tabla OWASP de ART-46 §13. Principios P-SB-01..09.

---

## 1. Modelo de amenazas (STRIDE resumido)

| Amenaza | Ejemplos en este stack | Controles principales |
|---|---|---|
| **S**uplantación | Robo de tokens, credential stuffing, fuerza bruta | Argon2id, bloqueo, rate limit, MFA, tokens cortos, rotación de refresh (ART-20) |
| **T**ampering (alteración) | Modificar ids/precios/estados en el request, inyección SQL | Validación estricta, `extra="forbid"`, cálculos en servidor, SQL parametrizado |
| **R**epudio | "Yo no anulé esa factura" | Auditoría con actor, `trace_id`, eventos de dominio persistidos |
| **I**nformation disclosure | IDOR, errores verbosos, datos de más en respuestas, logs con PII | Alcance obligatorio, errores genéricos, esquemas de salida explícitos, enmascarado |
| **D**enial of service | Payloads gigantes, consultas costosas, paginación sin límite, ReDoS | Límites en Nginx y app, `statement_timeout`, `size` máx., regex seguras |
| **E**levation of privilege | Asignarse un perfil, acceder a endpoints administrativos | RBAC por permiso, reglas de administración (ART-20 §4.3), deny by default |

## 2. OWASP API Security Top 10 (2023) → controles de la base

| Riesgo | Control | Artefacto |
|---|---|---|
| API1 BOLA (IDOR) | Repositorios y query services reciben el alcance; 404 fuera de alcance; UUIDv7 | ART-10 §3.4, ART-20 §4.2 |
| API2 Autenticación rota | JWT validado completo, refresh rotativo, bloqueo, MFA | ART-20 |
| API3 Autorización a nivel de propiedad | `extra="forbid"` en entrada; esquemas de salida explícitos por perfil | ART-19, P-SB-09 |
| API4 Consumo sin restricción | Rate limit, tamaño de body, paginación máx., timeouts, límites de elementos en arrays | ART-34, ART-14 |
| API5 Autorización a nivel de función | `require_permission` + `actor.exigir` | ART-10 §4.1 |
| API6 Flujos de negocio sensibles | Idempotencia, límites por usuario en operaciones de negocio, CAPTCHA en registro/recuperación si hay abuso | ART-14 §5 |
| API7 SSRF | No aceptar URLs del cliente; si es necesario: lista blanca de dominios, resolver DNS y bloquear IPs privadas/locales, sin redirecciones | §5 |
| API8 Mala configuración | Headers, docs deshabilitadas, CORS estricto, sin stack traces, imágenes endurecidas | ART-33, ART-34 |
| API9 Inventario | OpenAPI versionado, endpoints retirados con `Sunset`, sin endpoints de prueba en producción | ART-14 |
| API10 Consumo inseguro de APIs | Validar respuestas externas con esquemas, timeouts, TLS verificado, no confiar en datos de terceros | §5 |

## 3. Configuración de la aplicación

```python
app = FastAPI(
    docs_url=None if settings.is_production else "/docs",
    redoc_url=None,
    openapi_url=None if settings.is_production else "/openapi.json",
)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=settings.allowed_hosts)
app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins, allow_credentials=True,
                   allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
                   allow_headers=["Authorization", "Content-Type", "X-Request-ID", "Idempotency-Key",
                                  "If-Match", "X-Requested-With"],
                   max_age=600)
```
- Si el front se sirve en el mismo dominio (recomendado, ART-09), `cors_origins` puede quedar vacío en producción.
- Cabeceras de seguridad: las pone Nginx; la API agrega `Cache-Control: no-store` en respuestas autenticadas por defecto.
- Ninguna ruta de depuración, `/admin` de librerías ni endpoints de prueba en producción.

## 4. Protección CSRF
- Endpoints que leen la cookie de refresh (`/auth/refresh`, `/auth/logout`): exigir `SameSite=Strict` **y** el encabezado `X-Requested-With: fetch` (un formulario de otro sitio no puede enviarlo sin CORS) **y** validar `Origin` contra la lista permitida.
- El resto del API usa `Authorization: Bearer`, que no se envía automáticamente: no requiere token CSRF.

## 5. Integraciones externas
- Cliente `httpx.AsyncClient` compartido con `timeout` (connect 5 s, read 15 s), `verify=True`, límites de conexión.
- Reintentos con backoff solo para operaciones idempotentes.
- Respuestas validadas con Pydantic en el adaptador; datos externos nunca se confían para decisiones de autorización.
- Webhooks entrantes: firma HMAC + timestamp (ART-19 §5) y lista de IPs si el proveedor la publica.
- SSRF: si el negocio exige descargar desde URL de usuario, servicio aislado con lista blanca y bloqueo de rangos privados (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, `::1`, `fc00::/7`) verificados **después** de resolver DNS.

## 6. Criptografía y datos sensibles
| Necesidad | Solución |
|---|---|
| Contraseñas | Argon2id (ART-20 §7) |
| Tokens opacos almacenados | SHA-256 del token (son aleatorios de alta entropía; no requieren hash lento) |
| Campos sensibles en BD (secretos MFA, números de cuenta, documentos si el brief lo exige) | AES-256-GCM con `cryptography`; llave en secreto (ART-35), `key_id` guardado junto al dato para rotar |
| Búsqueda por campo cifrado | Columna adicional con HMAC-SHA256 (llave distinta) para igualdad exacta |
| Firma JWT | Ed25519 o RSA ≥ 2048 con `kid` y rotación |
| Aleatoriedad | Solo `secrets` (nunca `random`) |
| Comparación de secretos | `hmac.compare_digest` |

Nunca implementar criptografía propia.

## 7. Protección de datos personales (Colombia)
- La Ley 1581 de 2012 y el Decreto 1377 de 2013 (habeas data) aplican a datos personales: finalidad informada, autorización del titular, derecho a consultar, actualizar y suprimir.
- El brief (ART-03 §8) identifica datos personales; el plan técnico define: autorización (registro de consentimiento con fecha y versión de la política), endpoints de consulta/actualización/supresión cuando aplique, retención y anonimización.
- Minimización: no pedir ni guardar datos que el negocio no necesita.
- *Esto es una guía técnica, no asesoría legal: el responsable del proyecto debe validar los requisitos con un asesor.*

## 8. Dependencias y cadena de suministro
- Lockfiles obligatorios; instalación con `--frozen`/`npm ci`.
- `pip-audit`, `npm audit`, Trivy en CI (ART-36); actualización mensual de dependencias (o inmediata ante CVE crítica).
- Revisar mantenimiento y popularidad de una librería antes de agregarla; evitar paquetes con nombres parecidos a otros conocidos (typosquatting).
- `npm ci --ignore-scripts` en CI cuando sea posible.

## 9. Herramientas de verificación
| Herramienta | Qué detecta | Dónde |
|---|---|---|
| bandit, semgrep (reglas python, fastapi, secrets) | Patrones inseguros en código | CI |
| gitleaks | Secretos | pre-commit y CI |
| pip-audit, npm audit, Trivy | Dependencias e imágenes vulnerables | CI |
| Schemathesis | Fuzzing guiado por OpenAPI, 5xx, violaciones de contrato | CI |
| OWASP ZAP (baseline + API scan) | Configuración, headers, inyecciones comunes | Staging nocturno |
| Suite adversaria propia | Lógica de negocio, IDOR, permisos | CI (ART-42) |

## 10. Respuesta a incidentes (mínimo)
1. Detectar (alertas de auditoría: `token_reuse_detected`, picos de `access_denied`, 5xx).
2. Contener: revocar sesiones afectadas, rotar secretos, bloquear IPs en Nginx.
3. Investigar con logs por `trace_id`/`user_id`.
4. Corregir con prueba que reproduce (P-PR-06).
5. Notificar según la normativa (la SIC en Colombia para incidentes con datos personales) y documentar lecciones como cambio a la base.
