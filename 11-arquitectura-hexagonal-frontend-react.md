---
id: ART-11
titulo: Arquitectura hexagonal – Frontend React
version: 1.1.0   # 1.1.0: puerto Notificador y alertas flotantes (ART-29 §1)
estado: vigente
aplica_a: [frontend-web]
depende_de: [ART-02, ART-10]
referenciado_por: [ART-47]
stack: [React 19, TypeScript strict, Vite, TanStack Query, React Router, Zod, Vitest, Testing Library, MSW, Playwright]
---

# ART-11 · Arquitectura hexagonal – Frontend React

> **Instrucción para agentes IA**
> - Usa este documento como molde obligatorio para crear cualquier módulo del frontend web.
> - En hexagonal, **React es un adaptador** (infraestructura de entrada), igual que el cliente HTTP es un adaptador de salida. El dominio y los casos de uso no conocen React.
> - Antes de entregar un módulo, verifica el checklist de §12.

---

## 1. Principios aplicados

| Principio | Cómo se aplica aquí |
|---|---|
| P-AR-02 | Cada módulo tiene `domain/`, `application/` e `infrastructure/` |
| P-AR-03 | `domain` y `application` son TypeScript puro: sin React, sin fetch, sin Zod, sin TanStack |
| P-ER-05 | Los errores del API se traducen por `code` a mensajes de UI |
| P-SF-* | Guardas por permiso, tokens solo en memoria, cierre de sesión con limpieza total |
| P-EV-08 | Los eventos WebSocket invalidan la caché y se vuelve a consultar por HTTP |

---

## 2. Estructura de carpetas

```
frontend/
├── src/
│   ├── main.tsx                         # Punto de entrada: monta <App/>
│   ├── app/                             # Composition root y shell
│   │   ├── container.ts                 # Construye adaptadores y casos de uso (DI)
│   │   ├── DependenciesProvider.tsx     # Contexto React que expone el container
│   │   ├── router.tsx                   # Une las rutas de todos los módulos
│   │   ├── query-client.ts              # Configuración de TanStack Query
│   │   └── App.tsx                      # Providers, ErrorBoundary, layout
│   │
│   ├── shared/                          # Transversal, con las mismas 3 capas
│   │   ├── domain/
│   │   │   ├── app-error.ts             # AppError con code/kind
│   │   │   ├── permiso.ts               # Tipo Permiso = `${string}:${string}`
│   │   │   ├── pagina.ts                # Pagina<T>, PageRequest
│   │   │   └── result.ts                # (opcional) Result<T, E>
│   │   ├── application/
│   │   │   └── ports/
│   │   │       ├── http-client.ts
│   │   │       ├── session-store.ts
│   │   │       ├── data-cache.ts        # Abstracción de la caché de datos (query cache)
│   │   │       ├── realtime-client.ts
│   │   │       ├── tab-sync.ts          # Sincronización entre pestañas
│   │   │       ├── notificador.ts       # Alertas flotantes al usuario (ART-29 §1.5)
│   │   │       └── logger.ts
│   │   └── infrastructure/
│   │       ├── http/                    # FetchHttpClient, refresh de tokens, mapeo de errores
│   │       ├── session/                 # MemorySessionStore, hooks useSession/usePermiso
│   │       ├── realtime/                # WebSocketClient con reconexión
│   │       ├── security/                # RequirePermission, Can, IdleTimer, sanitize
│   │       ├── errors/                  # ErrorBoundary, catálogo de mensajes por code
│   │       └── ui/                      # Design system: Button, Input, Table, alertas/ (ART-29 §1)...
│   │
│   ├── auth/                            # Login, refresh, logout, perfil actual
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   └── factura/                         # ← MÓDULO DE EJEMPLO
│       ├── domain/
│       │   ├── factura.ts               # Tipos + reglas puras (puedeAnularse, calcularTotal)
│       │   ├── errors.ts                # Códigos de error del módulo
│       │   └── factura-repository.ts    # Puerto (interface)
│       ├── application/
│       │   ├── listar-facturas.ts
│       │   ├── obtener-factura.ts
│       │   ├── crear-factura.ts
│       │   └── anular-factura.ts
│       └── infrastructure/
│           ├── api/
│           │   ├── factura-api-repository.ts   # Implementa el puerto con HttpClient
│           │   ├── dto.ts                      # Esquemas Zod de las respuestas del API
│           │   └── mappers.ts                  # DTO ⇄ dominio
│           ├── ui/
│           │   ├── pages/                      # FacturasPage, FacturaDetallePage, NuevaFacturaPage
│           │   ├── components/                 # FacturaTable, FacturaForm, AnularDialog
│           │   ├── hooks/                      # useFacturas, useCrearFactura (TanStack Query)
│           │   ├── forms/                      # Esquemas Zod de formularios
│           │   └── query-keys.ts
│           ├── realtime/
│           │   └── factura-realtime.ts         # Suscripción a eventos → invalidación de queries
│           └── routes.tsx                      # Rutas del módulo con el permiso requerido
│
├── tests/e2e/                           # Playwright (flujos por perfil + adversarias)
├── index.html
├── vite.config.ts
└── Dockerfile                           # Build multi-stage → archivos estáticos servidos por Nginx
```

### 2.1 Qué va en cada capa

| Capa | Contiene | NO contiene |
|---|---|---|
| `domain` | Tipos de entidades, value objects, reglas puras, códigos de error, puertos (interfaces) | React, hooks, fetch, Zod, fechas del sistema |
| `application` | Casos de uso como funciones que reciben puertos; orquestación; verificación de permisos antes de llamar al API | JSX, TanStack Query, URLs, JSON crudo |
| `infrastructure` | Componentes, páginas, hooks, formularios, repositorios HTTP, DTOs, WebSocket, rutas | Reglas de negocio (se delegan al dominio) |

### 2.2 Reglas de importación (ESLint)

Con `eslint-plugin-boundaries` (o `import/no-restricted-paths`):

```js
// eslint.config.js (fragmento)
'boundaries/element-types': ['error', {
  default: 'disallow',
  rules: [
    { from: 'domain',         allow: ['domain', 'shared-domain'] },
    { from: 'application',    allow: ['domain', 'application', 'shared-domain', 'shared-application'] },
    { from: 'infrastructure', allow: ['domain', 'application', 'infrastructure',
                                       'shared-domain', 'shared-application', 'shared-infrastructure'] },
  ],
}],
// Un módulo no importa la infraestructura de otro módulo (P-AR-04)
'no-restricted-imports': ['error', { patterns: ['@/*/infrastructure/*'] }], // Se excepciona el propio módulo y app/
```

Además: `no-restricted-syntax` para prohibir `dangerouslySetInnerHTML` fuera de `shared/infrastructure/security/SafeHtml.tsx`, y `no-console` en producción.

---

## 3. Dominio

```ts
// factura/domain/factura.ts
export type FacturaId = string & { readonly __brand: 'FacturaId' };
export type EstadoFactura = 'EMITIDA' | 'ANULADA';

export interface FacturaResumen {
  id: FacturaId;
  numero: string;
  cliente: string;
  total: number;
  estado: EstadoFactura;
}

export interface Factura extends FacturaResumen {
  items: ItemFactura[];
  creadaEn: Date;
}

export interface ItemFactura {
  productoId: string;
  descripcion: string;
  cantidad: number;
  precioUnitario: number;
}

export interface NuevaFactura {
  numero: string;
  clienteId: string;
  items: ItemFactura[];
}

export const puedeAnularse = (f: Pick<Factura, 'estado'>): boolean => f.estado === 'EMITIDA';

export const calcularTotal = (items: ItemFactura[]): number =>
  items.reduce((acc, i) => acc + i.cantidad * i.precioUnitario, 0);
```

```ts
// factura/domain/factura-repository.ts  (PUERTO)
import type { Pagina, PageRequest } from '@/shared/domain/pagina';

export interface FiltroFacturas { estado?: EstadoFactura; texto?: string; }

export interface FacturaRepository {
  listar(filtro: FiltroFacturas, page: PageRequest): Promise<Pagina<FacturaResumen>>;
  obtener(id: FacturaId): Promise<Factura>;
  crear(datos: NuevaFactura): Promise<Factura>;
  anular(id: FacturaId, motivo: string): Promise<void>;
}
```

```ts
// shared/domain/app-error.ts
export type ErrorKind =
  | 'VALIDATION' | 'NOT_FOUND' | 'CONFLICT' | 'BUSINESS_RULE' | 'UNAUTHENTICATED'
  | 'FORBIDDEN' | 'RATE_LIMITED' | 'EXTERNAL' | 'INTERNAL' | 'NETWORK';

export class AppError extends Error {
  constructor(
    readonly code: string,
    readonly kind: ErrorKind,
    message: string,
    readonly details: { field: string; rule: string }[] = [],
    readonly traceId?: string,
  ) {
    super(message);
  }
}
```

---

## 4. Aplicación (casos de uso)

Los casos de uso son funciones *curried*: reciben sus puertos y devuelven la función que ejecuta la acción. Así se prueban sin React.

```ts
// factura/application/anular-factura.ts
import { AppError } from '@/shared/domain/app-error';
import type { SessionStore } from '@/shared/application/ports/session-store';
import { puedeAnularse, type Factura } from '../domain/factura';
import type { FacturaRepository } from '../domain/factura-repository';

export const anularFactura =
  (repo: FacturaRepository, session: SessionStore) =>
  async (factura: Factura, motivo: string): Promise<void> => {
    if (!session.puede('factura:anular')) {
      throw new AppError('ACCESO_DENEGADO', 'FORBIDDEN', 'No tienes permiso para anular facturas.');
    }
    if (!puedeAnularse(factura)) {
      throw new AppError('FACTURA_YA_ANULADA', 'BUSINESS_RULE', 'La factura ya fue anulada.');
    }
    await repo.anular(factura.id, motivo.trim());
  };
```

> Las validaciones del front son para la experiencia de usuario (P-VA-06). El backend vuelve a validar todo.

---

## 5. Infraestructura

### 5.1 Puerto y adaptador HTTP

```ts
// shared/application/ports/http-client.ts
export interface HttpRequestOptions {
  query?: Record<string, string | number | boolean | undefined>;
  body?: unknown;
  signal?: AbortSignal;
}

export interface HttpClient {
  get(url: string, opts?: HttpRequestOptions): Promise<unknown>;
  post(url: string, opts?: HttpRequestOptions): Promise<unknown>;
  put(url: string, opts?: HttpRequestOptions): Promise<unknown>;
  delete(url: string, opts?: HttpRequestOptions): Promise<unknown>;
}
```

`FetchHttpClient` (en `shared/infrastructure/http`):

- Base URL relativa (`/api/v1`), servida por el mismo dominio a través de Nginx (evita CORS y permite cookies `SameSite=Strict`).
- `credentials: 'same-origin'` para la cookie del refresh token; agrega `Authorization: Bearer <access>` desde el `SessionStore` en memoria.
- Envía `X-Request-ID` y el encabezado anti-CSRF si el proyecto lo usa.
- Ante un **401**, hace **un solo** intento de refresh compartido entre peticiones concurrentes (una promesa única), y repite la petición original. Si el refresh falla, ejecuta `cerrarSesion('expirada')`.
- Convierte toda respuesta de error al formato `AppError` usando el `error.code` del backend. Si la respuesta no tiene ese formato: `AppError('ERROR_INTERNO', 'INTERNAL', mensajeGenérico)`. Un fallo de red se convierte en `AppError('SIN_CONEXION', 'NETWORK', ...)`.
- Timeout por defecto con `AbortController`.

### 5.2 Repositorio API (adaptador de salida)

```ts
// factura/infrastructure/api/dto.ts
import { z } from 'zod';

export const facturaResumenDto = z.object({
  id: z.string().uuid(),
  numero: z.string(),
  cliente: z.string(),
  total: z.coerce.number(),
  estado: z.enum(['EMITIDA', 'ANULADA']),
});

export const paginaFacturasDto = z.object({
  data: z.array(facturaResumenDto),
  meta: z.object({ page: z.number(), size: z.number(), total: z.number() }),
});
```

```ts
// factura/infrastructure/api/factura-api-repository.ts
export class FacturaApiRepository implements FacturaRepository {
  constructor(private readonly http: HttpClient) {}

  async listar(filtro: FiltroFacturas, page: PageRequest) {
    const raw = await this.http.get('/facturas', { query: { ...filtro, page: page.number, size: page.size } });
    const dto = paginaFacturasDto.parse(raw);          // Respuesta validada: nada inesperado entra a la app
    return toPaginaFacturas(dto);
  }
  // obtener, crear, anular...
}
```

> Las respuestas del API se validan con Zod **en el adaptador**. Si el contrato cambia, el error aparece en un solo lugar y no se propagan datos malformados a la UI.

### 5.3 Hooks (puente entre React y los casos de uso)

```ts
// factura/infrastructure/ui/query-keys.ts
export const facturaKeys = {
  all: ['facturas'] as const,
  listas: () => [...facturaKeys.all, 'lista'] as const,
  lista: (f: FiltroFacturas, p: PageRequest) => [...facturaKeys.listas(), f, p] as const,
  detalle: (id: FacturaId) => [...facturaKeys.all, 'detalle', id] as const,
};
```

```ts
// factura/infrastructure/ui/hooks/useFacturas.ts
export function useFacturas(filtro: FiltroFacturas, page: PageRequest) {
  const { facturas } = useDependencies();
  return useQuery({
    queryKey: facturaKeys.lista(filtro, page),
    queryFn: () => facturas.listar(filtro, page),
    placeholderData: keepPreviousData,
  });
}

export function useAnularFactura() {
  const { facturas } = useDependencies();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ factura, motivo }: { factura: Factura; motivo: string }) => facturas.anular(factura, motivo),
    onSuccess: (_, { factura }) => {
      qc.invalidateQueries({ queryKey: facturaKeys.listas() });
      qc.invalidateQueries({ queryKey: facturaKeys.detalle(factura.id) });
    },
  });
}
```

Los componentes **solo** usan hooks; nunca llaman al `HttpClient` ni a `fetch` directamente.

### 5.4 Composition root

```ts
// app/container.ts
export function createContainer() {
  const session = new MemorySessionStore();
  const queryClient = createQueryClient();
  const http = new FetchHttpClient({ baseUrl: '/api/v1', session });
  const realtime = new WebSocketRealtimeClient({ url: '/ws/v1' });
  const tabs = new BroadcastChannelTabSync('app-session');
  const notificador = new AlertasNotificador(alertasStore);   // ART-29 §1.5
  const facturaRepo = new FacturaApiRepository(http);

  return {
    session, queryClient, realtime, tabs, notificador,
    facturas: {
      listar: listarFacturas(facturaRepo),
      obtener: obtenerFactura(facturaRepo),
      crear: crearFactura(facturaRepo, session),
      anular: anularFactura(facturaRepo, session),
    },
    auth: buildAuthUseCases({ http, session, queryClient, realtime, tabs }),
  };
}
export type Container = ReturnType<typeof createContainer>;
```

---

## 6. Seguridad en el frontend

### 6.1 Sesión y tokens (P-SF-02)

| Dato | Dónde vive | Notas |
|---|---|---|
| Access token | Memoria (`MemorySessionStore`) | Se pierde al recargar; se recupera con refresh silencioso |
| Refresh token | Cookie `HttpOnly; Secure; SameSite=Strict` puesta por el backend | JavaScript nunca la lee |
| Usuario y permisos | Memoria, desde `GET /api/v1/auth/me` | Solo los campos necesarios para la UI |
| Preferencias no sensibles (tema, idioma) | `localStorage` | Único uso permitido de `localStorage` |

Al iniciar la app: `POST /auth/refresh` → si responde OK, se cargan `/auth/me` y los permisos; si no, la sesión queda como anónima.

### 6.2 Guardas de rutas y de acciones (P-SF-01, P-AU-02)

```tsx
// factura/infrastructure/routes.tsx
export const facturaRoutes: AppRoute[] = [
  { path: '/facturas',       permiso: 'factura:ver',   lazy: () => import('./ui/pages/FacturasPage') },
  { path: '/facturas/nueva', permiso: 'factura:crear', lazy: () => import('./ui/pages/NuevaFacturaPage') },
  { path: '/facturas/:id',   permiso: 'factura:ver',   lazy: () => import('./ui/pages/FacturaDetallePage') },
];
```

```tsx
// shared/infrastructure/security/RequirePermission.tsx
export function RequirePermission({ permiso, children }: { permiso: Permiso; children: ReactNode }) {
  const { status, puede } = useSession();
  const location = useLocation();
  if (status === 'cargando') return <PantallaCarga />;
  if (status === 'anonimo') return <Navigate to="/login" replace state={{ from: location.pathname }} />;
  if (!puede(permiso)) return <Navigate to="/403" replace />;
  return <>{children}</>;
}

// Para botones y acciones dentro de una pantalla
export function Can({ permiso, children }: { permiso: Permiso; children: ReactNode }) {
  return useSession().puede(permiso) ? <>{children}</> : null;
}
```

- `app/router.tsx` envuelve **toda** ruta que tenga `permiso` con `RequirePermission`. Una ruta sin `permiso` solo puede ser pública si está en la lista explícita `PUBLIC_ROUTES` (login, recuperar contraseña, 404).
- Los módulos se cargan con `lazy` para no descargar el código de pantallas a las que el perfil no accede. Esto no es un control de seguridad (el backend lo es), pero reduce la exposición.
- El parámetro `from` del redirect a login solo se acepta si es una ruta interna (empieza por `/` y no por `//`), para evitar *open redirect*.

### 6.3 Cierre de sesión (P-SF-03)

```ts
// auth/application/cerrar-sesion.ts
export type MotivoCierre = 'manual' | 'expirada' | 'inactividad' | 'otra-pestana' | 'permisos-cambiados';

export const cerrarSesion =
  (deps: { auth: AuthRepository; session: SessionStore; cache: DataCache;
           realtime: RealtimeClient; tabs: TabSync; notificador: Notificador; logger: Logger }) =>
  async (motivo: MotivoCierre): Promise<void> => {
    try {
      if (motivo !== 'otra-pestana') await deps.auth.revocar();   // Backend invalida el refresh
    } catch (e) {
      deps.logger.warn('logout_revocation_failed', { motivo });    // Igual se limpia todo
    } finally {
      deps.realtime.desconectar();
      deps.cache.limpiar();            // queryClient.clear(): ningún dato queda en memoria
      deps.session.limpiar();          // Token, usuario, permisos
      deps.notificador.cerrarTodas();  // Ninguna alerta con información queda visible
      if (motivo !== 'otra-pestana') deps.tabs.notificarCierre(motivo);
    }
  };
```

En la UI, después de `cerrarSesion`:
1. `navigate('/login', { replace: true })` para que "atrás" no vuelva a una pantalla con datos.
2. Resetear formularios y stores locales (por ejemplo, `reset()` de Zustand si se usa).
3. El backend responde a `/auth/logout` con `Clear-Site-Data: "cache", "storage"` y las respuestas con datos sensibles llevan `Cache-Control: no-store`, para que el navegador no muestre páginas cacheadas con el botón atrás.

El cierre por inactividad (`IdleTimer`) muestra un aviso un minuto antes y luego ejecuta `cerrarSesion('inactividad')`. El tiempo se configura por proyecto.

### 6.4 Exposición de datos

- Prohibido renderizar HTML del servidor sin `SafeHtml` (DOMPurify).
- No poner datos personales en URLs, títulos de página, `console`, mensajes de error ni herramientas de analítica.
- Enmascarar en la UI los datos sensibles (documento, cuenta, teléfono) salvo cuando el permiso explícito permite verlos completos.
- No guardar respuestas del API en `localStorage` como "caché offline".
- Build de producción sin source maps públicos (`build.sourcemap: false` o subirlos solo al sistema de monitoreo).
- Solo variables `VITE_*` no sensibles (URLs públicas, flags).
- La aplicación debe funcionar con la CSP estricta que define Nginx (sin `eval` ni scripts inline).

---

## 7. Manejo de errores en la UI

```ts
// shared/infrastructure/errors/mensajes.ts
export const MENSAJES: Record<string, string> = {
  SIN_CONEXION: 'No hay conexión con el servidor. Revisa tu internet e intenta de nuevo.',
  ACCESO_DENEGADO: 'No tienes permiso para realizar esta acción.',
  VALIDACION_FALLIDA: 'Algunos datos no son válidos. Revisa los campos marcados.',
  ERROR_INTERNO: 'Ocurrió un error inesperado. Si persiste, comunícate con soporte.',
  // Cada módulo agrega los suyos: FACTURA_DUPLICADA, FACTURA_YA_ANULADA...
};

export const mensajeDeError = (e: unknown): string =>
  e instanceof AppError ? (MENSAJES[e.code] ?? e.message) : MENSAJES.ERROR_INTERNO;
```

| Tipo de error | Presentación |
|---|---|
| `VALIDATION` | Errores debajo de cada campo usando `details[].field` |
| `BUSINESS_RULE`, `CONFLICT` | Alerta flotante de error |
| `FORBIDDEN` | Alerta flotante de error y, si fue al cargar una pantalla, redirección a `/403` |
| `UNAUTHENTICATED` | Lo maneja el cliente HTTP (refresh o cierre de sesión) |
| `NOT_FOUND` | Pantalla "no encontrado" |
| `INTERNAL`, `NETWORK` | Alerta flotante de error con el código de soporte (`trace_id`) |

Toda retroalimentación (éxito, información, advertencia, error) pasa por el puerto `Notificador` (`shared/application/ports/notificador.ts`) y su implementación de **alertas flotantes**: se cierran con la X de la esquina superior derecha o solas al terminar el tiempo, que se muestra con una barra que se acorta en la parte inferior y se pausa al pasar el puntero o enfocar la alerta. Especificación completa, código de referencia y pruebas en **ART-29 §1** (P-UX-01 a P-UX-05).

Además, un `ErrorBoundary` global y uno por página muestran una pantalla de recuperación sin el stack trace.

---

## 8. Tiempo real (WebSocket)

- `WebSocketRealtimeClient` (en `shared/infrastructure/realtime`) se conecta al mismo origen (`wss://<host>/ws/v1/...`). La autenticación va por cookie o con un ticket de un solo uso obtenido por HTTP; nunca con el access token en la URL.
- Reconexión con backoff exponencial y jitter. Al reconectar, se invalidan las queries del módulo para recuperar lo que se perdió.
- Se desconecta en `cerrarSesion` y cuando la pestaña recibe un cierre desde otra pestaña.
- Cada módulo traduce los eventos en invalidaciones:

```ts
// factura/infrastructure/realtime/factura-realtime.ts
export function registrarEventosFactura(realtime: RealtimeClient, qc: QueryClient) {
  return realtime.on('factura.creada', () => {
    qc.invalidateQueries({ queryKey: facturaKeys.listas() });
  });
}
```

- Los mensajes recibidos también se validan con Zod. Un mensaje inválido se descarta y se registra.

---

## 9. Formularios

- React Hook Form + Zod (`forms/`), con las mismas reglas de la especificación del endpoint (ART-46): tipo, longitud, patrón y rango.
- `maxLength` también en el input HTML; `inputMode` y `autocomplete` adecuados (`autocomplete="off"` o `"one-time-code"` en campos sensibles).
- Para archivos: se validan tamaño y tipo antes de subir (UX); el backend vuelve a validar por contenido.
- El botón de envío se deshabilita mientras la petición está en curso, para evitar dobles envíos. Para operaciones críticas se envía una `Idempotency-Key`.

---

## 10. Pruebas del frontend

| Tipo | Herramienta | Qué prueba |
|---|---|---|
| Unitarias de dominio | Vitest | Reglas puras (`puedeAnularse`, `calcularTotal`) |
| Unitarias de aplicación | Vitest + repositorios fake | Casos de uso: permisos, reglas, llamadas al puerto |
| Adaptadores | Vitest + MSW | Repositorio API: mapeo, validación Zod, traducción de errores, refresh en 401 |
| Componentes | Testing Library + MSW | Renderizado por permiso (`Can`), formularios, estados de carga, error y vacío |
| Seguridad | Vitest | `RequirePermission` redirige; `cerrarSesion` limpia caché, sesión y WS aunque falle la revocación; open redirect bloqueado |
| E2E | Playwright | Flujos por perfil, navegación directa por URL a rutas sin permiso, logout + botón atrás, expiración de sesión |
| Adversarias | Playwright + payloads | Inputs con XSS (`<img src=x onerror=...>`), manipulación de IDs en la URL, intento de usar la app después del logout en otra pestaña |

```ts
// factura/application/anular-factura.test.ts
it('no llama al API si el usuario no tiene permiso', async () => {
  const repo = fakeFacturaRepository();
  const session = fakeSession({ permisos: ['factura:ver'] });
  await expect(anularFactura(repo, session)(facturaEmitida(), 'error')).rejects.toMatchObject({ code: 'ACCESO_DENEGADO' });
  expect(repo.anular).not.toHaveBeenCalled();
});
```

---

## 11. Antipatrones prohibidos

| Antipatrón | En su lugar |
|---|---|
| `fetch` o `axios` dentro de componentes | Hook → caso de uso → repositorio |
| Tokens o respuestas del API en `localStorage`/`sessionStorage` | Memoria + cookie HttpOnly |
| Mostrar u ocultar por nombre de perfil (`if (rol === 'admin')`) | `puede('recurso:accion')` |
| Reglas de negocio en JSX | Funciones del dominio |
| `dangerouslySetInnerHTML` directo | `SafeHtml` con DOMPurify |
| Mostrar `error.message` crudo de una excepción desconocida | `mensajeDeError` por `code` |
| Logout que solo borra el token | `cerrarSesion` completo (§6.3) |
| Datos sensibles en la URL o en `console.log` | Estado en memoria; logger que se apaga en producción |
| Importar la infraestructura de otro módulo | Exponer lo necesario por `application` o mover a `shared` |

---

## 12. Checklist del módulo (Definition of Done frontend)

- [ ] Estructura `domain/application/infrastructure` y reglas de ESLint de fronteras en verde.
- [ ] El dominio y los casos de uso no importan React, Zod, TanStack ni el cliente HTTP.
- [ ] Toda ruta del módulo declara `permiso` y las acciones usan `Can`.
- [ ] Las respuestas del API se validan con Zod en el adaptador.
- [ ] Los errores se muestran por `code`, con errores por campo en formularios.
- [ ] Éxitos, advertencias y errores se informan solo con alertas flotantes vía `Notificador` (ART-29 §1).
- [ ] Ningún dato sensible en almacenamiento del navegador, URLs ni consola.
- [ ] El logout limpia caché, sesión, WS y pestañas; el botón atrás no muestra datos.
- [ ] Los eventos de tiempo real invalidan queries; los mensajes se validan.
- [ ] Hay pruebas unitarias, de componentes, E2E y adversarias en verde, con cobertura ≥ 80 % en domain y application.
