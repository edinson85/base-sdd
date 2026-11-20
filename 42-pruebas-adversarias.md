---
id: ART-42
titulo: Pruebas adversarias (seguridad) front y back
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, infraestructura]
depende_de: [ART-02, ART-21, ART-27, ART-34, ART-46]
---

# ART-42 · Pruebas adversarias

> **Instrucción para agentes IA**
> - Las pruebas adversarias simulan a un atacante sobre **nuestros propios ambientes** (local, CI, staging). Nunca se ejecutan contra sistemas de terceros ni contra producción sin autorización escrita.
> - Cada endpoint, pantalla y canal WS tiene su lista en la especificación (ART-46 §12.3, Anexo B §6). Este documento da el catálogo de ataques, los helpers y las herramientas.

---

## 1. Catálogo de ataques

### 1.1 Autenticación y sesión (`AUTH-*`)
| ID | Ataque | Esperado |
|---|---|---|
| AUTH-01 | Sin `Authorization` | 401 `NO_AUTENTICADO` |
| AUTH-02 | Token expirado | 401 |
| AUTH-03 | Firma alterada (cambiar un carácter del payload) | 401 |
| AUTH-04 | `alg: none` / `alg: HS256` con la llave pública como secreto | 401 |
| AUTH-05 | `kid` inexistente o con traversal (`../../`) | 401 |
| AUTH-06 | Token de otro ambiente (`iss`/`aud` distintos) | 401 |
| AUTH-07 | Token revocado (después de logout) | 401 |
| AUTH-08 | Reutilizar refresh ya usado | 401 + familia revocada |
| AUTH-09 | Fuerza bruta de login | Bloqueo + 429 |
| AUTH-10 | Enumeración de usuarios (login y recuperación) | Mismo mensaje y tiempo comparable |
| AUTH-11 | Refresh web sin encabezado anti-CSRF / con `Origin` ajeno | 403/401 |

### 1.2 Autorización (`AZ-*`)
| ID | Ataque | Esperado |
|---|---|---|
| AZ-01 | Perfil sin permiso llama al endpoint | 403 |
| AZ-02 | IDOR: id de recurso de otra empresa | 404 |
| AZ-03 | IDOR en referencias del body (`cliente_id` ajeno) | 404/422 sin revelar datos |
| AZ-04 | Asignación masiva (`empresa_id`, `estado`, `total`, `rol`) | 422 |
| AZ-05 | Escalada: usuario se asigna perfiles o permisos | 403 |
| AZ-06 | Acceso a endpoints administrativos con perfil básico | 403 |
| AZ-07 | Respuesta incluye campos no permitidos para el perfil | Ausentes |

### 1.3 Entradas (`IN-*`)
| ID | Payloads | Esperado |
|---|---|---|
| IN-01 SQLi | `' OR '1'='1`, `'; DROP TABLE x;--`, `1 UNION SELECT ...`, `\\'` | 422 o tratado como texto; nunca 500 |
| IN-02 XSS | `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>`, `"><svg onload=alert(1)>`, `javascript:alert(1)` | 422 por lista blanca o guardado como texto y mostrado escapado |
| IN-03 Longitudes | Vacío, solo espacios, máx., máx+1, 10 KB, 1 MB | 422 / 413 |
| IN-04 Unicode | Emojis, RTL (`‮`), ancho cero, homoglifos (`раypal`), NUL (`\u0000`), combinantes | Normalizado o 422 |
| IN-05 Tipos | String en número, array en objeto, `null`, booleanos, números enormes (`1e309`), negativos, decimales extra | 422 |
| IN-06 JSON | Malformado, profundamente anidado, claves duplicadas, `Content-Type` incorrecto | 400 / 415 |
| IN-07 Path traversal | `../../etc/passwd` en nombres de archivo o parámetros | 422 / nombre regenerado |
| IN-08 ReDoS | Cadenas largas diseñadas contra cada regex | Respuesta < 100 ms |
| IN-09 Archivos | Extensión falsa (`.php` renombrado `.pdf`), polyglot, tamaño > máx, bomba de descompresión de imagen | Rechazo |
| IN-10 Orden y filtros | `sort=password`, `sort=(SELECT ...)`, filtros no declarados | 422 |
| IN-11 Paginación | `size=100000`, `page=-1`, cursor manipulado | 422 |

### 1.4 Abuso de recursos y negocio (`AB-*`)
| ID | Ataque | Esperado |
|---|---|---|
| AB-01 | Ráfaga sobre un endpoint | 429 con `Retry-After` |
| AB-02 | Doble envío / reintentos concurrentes | Un solo efecto (idempotencia o unicidad) |
| AB-03 | Condición de carrera en reglas de estado (dos anulaciones simultáneas) | Un éxito, un error de negocio |
| AB-04 | Arrays con miles de elementos | 422 por máximo |
| AB-05 | Consultas costosas (filtros amplios, búsqueda con `%`) | Dentro de `statement_timeout`, paginado |

### 1.5 WebSocket (`WS-*`)
Ver ART-46 Anexo B §6: sin ticket, ticket reutilizado/expirado, origin ajeno, canal ajeno, mensajes malformados/gigantes/ráfagas, socket tras logout.

### 1.6 Frontend (`FE-*`)
Ver ART-27 §8: XSS almacenado visible, rutas sin permiso, manipulación de respuestas, logout multi-pestaña, almacenamiento del navegador limpio, open redirect.

### 1.7 Infraestructura (`INF-*`)
Ver ART-34 §8: headers, archivos ocultos, docs expuestas, TLS antiguo, métodos, tamaño de body, puertos de BD/Redis no accesibles desde fuera.

## 2. Implementación en el backend

```python
# tests/adversarial/payloads.py
SQLI = ["' OR '1'='1", "'; DROP TABLE app.facturas;--", "1 UNION SELECT password_hash FROM app.usuarios", "\\'"]
XSS = ["<script>alert(1)</script>", "<img src=x onerror=alert(1)>", "\"><svg onload=alert(1)>", "javascript:alert(1)"]
UNICODE = ["‮abc", "a​b", "\u0000", "😀" * 50, "раypal"]
LONGITUDES = ["", "   ", "A" * 10_000]
TODOS_TEXTO = SQLI + XSS + UNICODE + LONGITUDES
```

```python
# tests/adversarial/factura/test_post_facturas.py
@pytest.mark.adversarial
@pytest.mark.parametrize("valor", TODOS_TEXTO)
@pytest.mark.parametrize("campo", ["numero", "items.0.descripcion"])
async def test_campos_texto_rechazan_o_neutralizan(client, auth_as, seed, campo, valor):
    resp = await client.post("/api/v1/facturas", json=set_path(payload_valido(seed), campo, valor),
                             headers=await auth_as("cajero") | idem())
    assert resp.status_code in (201, 422)
    assert resp.status_code != 500
    if resp.status_code == 422:
        assert valor not in resp.text            # No refleja el valor (P-VA-07)

@pytest.mark.adversarial
@pytest.mark.parametrize("campo,valor", [("empresa_id", str(uuid7())), ("estado", "ANULADA"), ("total", "1.00")])
async def test_asignacion_masiva_rechazada(client, auth_as, seed, campo, valor):
    body = payload_valido(seed) | {campo: valor}
    resp = await client.post("/api/v1/facturas", json=body, headers=await auth_as("cajero") | idem())
    assert resp.status_code == 422

@pytest.mark.adversarial
async def test_idor_detalle_de_otra_empresa(client, auth_as, seed):
    ajena = await seed.factura(empresa=seed.otra_empresa)
    resp = await client.get(f"/api/v1/facturas/{ajena.id}", headers=await auth_as("admin"))
    assert resp.status_code == 404
```

Helpers compartidos (`tests/adversarial/helpers.py`):
- `tokens_manipulados(token)` → expirado, firma alterada, `alg none`, `kid` raro, `aud` distinto.
- `para_cada_endpoint(openapi)` → recorre rutas para pruebas genéricas (AUTH-01 en todos los endpoints no públicos, AZ-01 con usuario sin permisos).
- `concurrente(n, coro)` → lanza N requests simultáneos (AB-02, AB-03).

Prueba genérica que protege contra endpoints olvidados:
```python
@pytest.mark.adversarial
@pytest.mark.parametrize("ruta", rutas_no_publicas(app))
async def test_todo_endpoint_no_publico_exige_autenticacion(client, ruta):
    resp = await client.request(ruta.method, ruta.ejemplo_url)
    assert resp.status_code == 401
```

## 3. Fuzzing con Schemathesis
```bash
schemathesis run http://api:8000/openapi.json \
  --header "Authorization: Bearer $TOKEN_CAJERO" \
  --checks all --hypothesis-max-examples 200 \
  --exclude-checks ignored_auth   # se activa en una corrida separada sin token
```
Falla si: algún 5xx, respuesta que no cumple el esquema declarado, o códigos de estado no documentados. En CI se ejecuta contra la API en modo `e2e` (con OpenAPI habilitado solo en ese ambiente).

> Verifica las opciones exactas de la versión de Schemathesis instalada; los nombres de *checks* pueden variar entre versiones.

## 4. DAST con OWASP ZAP (staging)
- Baseline (pasivo) en cada despliegue a staging; API scan (activo) nocturno con el OpenAPI y un usuario de prueba.
- Archivo de reglas `zap-rules.tsv` versionado: cada regla ignorada tiene justificación.
- Hallazgos altos bloquean el release (ART-06 §2.4).

## 5. Frontend
Playwright con `page.route` para manipular respuestas y con payloads de §1.3 escritos en formularios; ver ART-27 §8 y ART-41 E2E-T08/T09.

```ts
test('FE: XSS almacenado se muestra como texto', async ({ page }) => {
  await crearClienteViaApi({ nombre: '<img src=x onerror="window.__xss=1">' });   // vía semilla e2e
  await page.goto('/clientes');
  await expect(page.getByText('<img src=x onerror="window.__xss=1">')).toBeVisible();
  expect(await page.evaluate(() => (window as any).__xss)).toBeUndefined();
});
```
(El backend rechaza `<` en nombres por lista blanca; la prueba usa la semilla para verificar la defensa del front de forma independiente.)

## 6. Reporte
Cada hallazgo: ID del ataque, endpoint/pantalla, pasos, resultado esperado vs. obtenido, severidad (CVSS aproximado: crítica, alta, media, baja), corrección y prueba que lo cubre (P-PR-06).
