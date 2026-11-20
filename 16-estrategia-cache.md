---
id: ART-16
titulo: Estrategia de caché
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil, infraestructura]
depende_de: [ART-02, ART-10]
---

# ART-16 · Estrategia de caché

> **Instrucción para agentes IA**
> Antes de cachear, completa la matriz de §1 para el recurso. Si no puedes definir **clave, TTL e invalidación**, no lo caches (P-CA-01).

---

## 1. Qué cachear

| Recurso | ¿Cachear? | TTL sugerido | Invalidación |
|---|---|---|---|
| Permisos por perfil | Sí | 15 min | Evento `PermisosDePerfilCambiados` |
| Datos del usuario para `/auth/me` | Sí | 5 min | `UsuarioActualizado`, cambio de perfiles |
| Catálogos (tipos, ciudades, unidades, impuestos) | Sí | 1-24 h | Evento del catálogo o despliegue (cambio de versión de clave) |
| Configuración de empresa/tenant | Sí | 10 min | `ConfiguracionActualizada` |
| Detalle de recurso consultado con frecuencia | Depende | 1-5 min | Eventos de cambio del agregado |
| Listas con filtros variados | Rara vez | 30-60 s | Patrón por tenant |
| Resultados de reportes costosos | Sí | Según negocio | Por parámetros + versión de datos |
| Respuestas de servicios externos lentos | Sí | Según el servicio | TTL |
| Datos que cambian en cada request, saldos, inventario en tiempo real | **No** | — | — |
| Secretos, tokens, contraseñas | **Nunca** (P-CA-04) | — | — |

## 2. Capas

| Capa | Mecanismo | Uso |
|---|---|---|
| Backend → Redis | Cache-aside con `CachePort` | Principal (ART-10 §6.5) |
| Backend en memoria del proceso | `cachetools.TTLCache` muy corto (≤ 30 s) | Solo datos idénticos para todos y casi inmutables (ej. configuración global); se pierde la consistencia entre réplicas |
| HTTP | `ETag` + `If-None-Match` → 304 | Catálogos y recursos de lectura frecuente sin datos personales |
| Nginx | Estáticos con hash (`immutable`) | Build del front |
| Cliente | TanStack Query / Riverpod con `staleTime` | Evitar peticiones repetidas en la sesión; se limpia en logout |

## 3. Claves
`{app}:{modulo}:{recurso}:v{n}:{alcance}:{identificadores|hash_parametros}`

- `app` = código del proyecto (brief).
- `v{n}`: subir la versión cuando cambia la forma del dato cacheado (invalida todo sin borrar).
- `alcance`: `empresa_id` o `user_id` cuando el dato depende de ellos (P-CA-03).
- Parámetros complejos: hash SHA-256 de los parámetros normalizados (ordenados, validados).

Ejemplos:
```
fer:auth:permisos:v1:perfil:0193...
fer:catalogo:ciudades:v2:all
fer:factura:detalle:v1:emp:0193...:0193...
fer:reporte:ventas:v1:emp:0193...:3f9a1c...
```

## 4. Patrones

### 4.1 Cache-aside (por defecto)
Leer → si falla, consultar la BD → guardar con TTL. Ver ART-10 §6.5.

### 4.2 Protección contra estampida
Para claves calientes y costosas:
- **TTL con jitter**: `ttl = base + random(0, base*0.1)` para que no expiren todas a la vez.
- **Single flight**: lock corto en Redis (`SET lock:<clave> 1 NX EX 10`); quien no obtiene el lock espera brevemente y relee, o sirve el valor anterior si existe (*stale-while-revalidate*).

### 4.3 Invalidación por eventos
Observers (`InvalidarCache<Modulo>`) suscritos a los eventos del agregado (ART-10 §4.5). Borrado por patrón con `SCAN` + `UNLINK` en lotes; para muchas claves, preferir subir la versión del alcance (`fer:factura:lista:v1:emp:X:gen:<n>` con contador de generación por tenant).

### 4.4 Serialización
JSON (`orjson`) con un campo `_v` de esquema. Nunca `pickle` (riesgo de ejecución de código si alguien escribe en Redis).

## 5. Fallas
- Redis caído o lento (timeout de 100-200 ms en operaciones de caché): degradar a BD, `WARNING` con *circuit breaker* simple para no saturar logs (P-CA-05).
- Error al escribir en caché: se ignora y se registra; nunca falla la operación.
- Dato corrupto o con versión distinta: se descarta y se recalcula.

## 6. Configuración de Redis

| Uso | Política | Recomendación |
|---|---|---|
| Caché | `maxmemory-policy allkeys-lru` | Base lógica o instancia de caché |
| Rate limit, tickets WS, revocación de tokens, locks, idempotencia | **Sin desalojo** (`noeviction`) | Instancia separada en proyectos medianos/grandes; en pequeños, una sola instancia con `volatile-lru` y **todas las claves de caché con TTL** |
| Seguridad | `requirepass`/ACL por servicio, `rename-command` de `FLUSHALL`, `FLUSHDB`, `CONFIG`, `KEYS` | Red interna únicamente |

## 7. Métricas
- Tasa de aciertos por prefijo (contadores hit/miss en logs o métricas).
- Memoria usada y claves desalojadas.
- Una caché con tasa de aciertos < 50 % se revisa: quizás no vale la pena.

## 8. Caché en el cliente
- TanStack Query: `staleTime` según el recurso (catálogos 10 min, listas 30 s, detalle 0-30 s); invalidación por eventos WS y por mutaciones (ART-11 §5.3).
- Nunca persistir la caché del cliente en almacenamiento del navegador (P-SF-02).
- `queryClient.clear()` en logout.
