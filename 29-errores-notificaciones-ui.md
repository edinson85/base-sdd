---
id: ART-29
titulo: Alertas flotantes, errores y estados en la UI
version: 1.1.0
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-02, ART-11, ART-12, ART-17, ART-31, ART-32]
---

# ART-29 · Alertas flotantes, errores y estados en la UI

> **Instrucción para agentes IA**
> - Toda retroalimentación al usuario (éxito, información, advertencia, error) se muestra con el **sistema de alertas flotantes** de §1, a través del puerto `Notificador`. No uses `alert()`, `confirm()`, mensajes sueltos en la pantalla ni librerías de toasts distintas (P-UX-01).
> - Toda pantalla implementa los estados de §4 y presenta los errores según §3.
> - Los textos salen del catálogo de errores (ART-17) y de los criterios de aceptación (ART-05); no inventes mensajes técnicos.

---

## 1. Sistema de alertas flotantes (P-UX-01 a P-UX-05)

### 1.1 Anatomía

```
                                   ┌──────────────────────────────────────────┐
                                   │ ✓  Factura creada                    [×] │ ← X en la esquina superior derecha
                                   │    La factura FV-000123 se guardó        │
                                   │    correctamente.                        │
                                   │    [Ver factura]                         │ ← acción opcional (máx. 1)
                                   │▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬                │ ← barra de tiempo (se acorta)
                                   └──────────────────────────────────────────┘
  Pantalla de la aplicación ...    ┌──────────────────────────────────────────┐
                                   │ !  Hay productos sin precio.         [×] │
                                   │▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬      │
                                   └──────────────────────────────────────────┘
```

| Parte | Especificación |
|---|---|
| Contenedor | Flotante (posición fija), por encima de todo el contenido, incluidos los diálogos; no desplaza el contenido ni bloquea clics fuera de las alertas |
| Ubicación | Escritorio: esquina superior derecha, a 16 px de los bordes, ancho 384 px. Pantallas < 640 px y móvil: parte superior, ancho completo menos 8 px por lado, respetando el área segura (notch) |
| Apilado | La más reciente arriba; separación de 8 px; máximo **4 visibles**; las demás esperan en cola y aparecen al cerrarse una |
| Ícono | A la izquierda, según el tipo (✓, i, !, ×) en el color del tipo |
| Título | Opcional, una línea, peso 600 |
| Mensaje | Obligatorio, máximo 2-3 líneas; texto del catálogo o de la HU |
| Acción | Opcional, una sola, como botón de texto ("Ver factura", "Reintentar", "Deshacer"); al ejecutarla la alerta se cierra |
| Botón cerrar (X) | **Esquina superior derecha**, área táctil ≥ 32×32 px (≥ 44×44 en móvil), nombre accesible "Cerrar alerta", visible siempre (no solo al pasar el puntero) |
| Barra de tiempo | **Parte inferior**, ancho completo, 3 px de alto, color del tipo con opacidad ~60 % sobre un riel muy tenue; se acorta de derecha a izquierda de forma continua y lineal durante la duración |
| Estilo | Fondo `surface`, borde izquierdo de 4 px en el color del tipo, radio `md`, sombra `lg`, texto `foreground` (contraste AA) |
| Entrada / salida | Deslizar + desvanecer, 200 ms; sin animación con `prefers-reduced-motion` |

### 1.2 Tipos y duración

| Tipo | Color (token) | Ícono | Duración por defecto | Rol accesible | Ejemplo |
|---|---|---|---|---|---|
| Éxito | `success` | ✓ | 5 s | `status` (polite) | "Factura creada correctamente." |
| Información | `info` | i | 6 s | `status` (polite) | "Se agregaron 3 facturas nuevas." |
| Advertencia | `warning` | ! | 8 s | `alert` (assertive) | "Algunos productos no tienen precio actualizado." |
| Error | `danger` | × | 10 s | `alert` (assertive) | "No se pudo anular la factura. La factura ya fue anulada." |
| Error persistente | `danger` | × | Sin cierre automático (sin barra) | `alert` | "No se pudo enviar la factura a la DIAN. [Reintentar]" |

Reglas de duración:
- Mensajes largos: +1 s por cada 25 caracteres sobre 100, con máximo 15 s.
- Alerta con acción: duración mínima 10 s, o persistente si la acción es necesaria para continuar.
- La duración se puede ajustar por alerta (`duracionMs`), pero nunca por debajo de 4 s.
- Persistente solo para errores que exigen una acción del usuario (P-UX-03); se cierra con la X o al ejecutar la acción.

### 1.3 Comportamiento

| Situación | Comportamiento |
|---|---|
| Aparece | Entra arriba de la pila; **no roba el foco**; se anuncia al lector de pantalla |
| Tiempo cumplido | La barra llega a cero y la alerta se cierra (la animación de la barra es la única fuente del tiempo) |
| Clic en la X | Se cierra de inmediato |
| Puntero encima (hover) | Pausa el tiempo **y la barra**; al salir continúa desde donde iba (no reinicia) |
| Foco de teclado dentro de la alerta | Pausa; `Esc` la cierra; al salir el foco, continúa |
| Pestaña o app no visible | Pausa; continúa al volver (el usuario no pierde mensajes mientras no mira) |
| Toque prolongado (móvil) | Pausa mientras se mantiene |
| Mensaje repetido (mismo tipo y texto en < 3 s) | No se duplica: se reinicia el tiempo de la alerta existente y se muestra un contador "×2" |
| Más de 4 alertas | Las nuevas esperan en cola |
| Cierre de sesión | Se cierran todas (`cerrarTodas()`), incluidas las de la cola |
| Navegación entre pantallas | Las alertas se mantienen (se muestran sobre la nueva pantalla) |

### 1.4 Contenido
- Mensaje en español claro, segunda persona, qué pasó y qué hacer (§6).
- Errores 5xx y de servicios externos: línea "Código de soporte: `<8 primeros caracteres del trace_id>`" con botón para copiar.
- Nunca datos sensibles en el texto de la alerta (P-SF): "Cliente actualizado", no "Cliente con cédula 1.061... actualizado".

### 1.5 Puerto `Notificador`

```ts
// shared/application/ports/notificador.ts
export type TipoAlerta = 'exito' | 'info' | 'advertencia' | 'error';

export interface OpcionesAlerta {
  titulo?: string;
  duracionMs?: number;          // ≥ 4000
  persistente?: boolean;        // Solo errores que exigen acción
  accion?: { etiqueta: string; ejecutar: () => void };
  codigoSoporte?: string;       // trace_id
}

export interface Notificador {
  exito(mensaje: string, opciones?: OpcionesAlerta): string;      // Devuelve el id
  info(mensaje: string, opciones?: OpcionesAlerta): string;
  advertencia(mensaje: string, opciones?: OpcionesAlerta): string;
  error(mensaje: string, opciones?: OpcionesAlerta): string;
  desdeError(error: unknown): string;                               // AppError → tipo y mensaje (§3)
  cerrar(id: string): void;
  cerrarTodas(): void;
}
```

- El puerto vive en `shared/application` para que los casos de uso que lo necesiten (por ejemplo `cerrarSesion`) no dependan de React.
- La implementación (`AlertasStore` + componentes) vive en `shared/infrastructure/ui/alertas/` y se crea en el composition root (ART-11 §5.4).
- Los hooks de mutación usan el notificador en `onSuccess`/`onError`; los componentes nunca dibujan sus propias alertas.

```ts
export function useCrearFactura() {
  const { facturas, notificador } = useDependencies();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: facturas.crear,
    onSuccess: (f) => {
      qc.invalidateQueries({ queryKey: facturaKeys.listas() });
      notificador.exito('Factura creada correctamente.', {
        accion: { etiqueta: 'Ver factura', ejecutar: () => navigate(`/facturas/${f.id}`) },
      });
    },
    onError: (e) => { if (!esErrorDeCampos(e)) notificador.desdeError(e); },   // Los de campo van al formulario
  });
}
```

### 1.6 Implementación de referencia (web)

```tsx
// shared/infrastructure/ui/alertas/AlertaFlotante.tsx
export function AlertaFlotante({ alerta, onCerrar }: { alerta: Alerta; onCerrar: (id: string) => void }) {
  const [interaccion, setInteraccion] = useState(false);
  const oculta = useDocumentoOculto();                 // document.visibilityState === 'hidden'
  const enPausa = interaccion || oculta;
  const esUrgente = alerta.tipo === 'error' || alerta.tipo === 'advertencia';

  return (
    <div
      role={esUrgente ? 'alert' : 'status'}
      className={`alerta alerta--${alerta.tipo}`}
      onMouseEnter={() => setInteraccion(true)}
      onMouseLeave={() => setInteraccion(false)}
      onFocus={() => setInteraccion(true)}
      onBlur={(e) => { if (!e.currentTarget.contains(e.relatedTarget as Node)) setInteraccion(false); }}
      onKeyDown={(e) => { if (e.key === 'Escape') onCerrar(alerta.id); }}
    >
      <IconoAlerta tipo={alerta.tipo} aria-hidden="true" />
      <div className="alerta__contenido">
        {alerta.titulo && <p className="alerta__titulo">{alerta.titulo}</p>}
        <p className="alerta__mensaje">{alerta.mensaje}{alerta.repeticiones > 1 && ` (×${alerta.repeticiones})`}</p>
        {alerta.codigoSoporte && <CodigoSoporte valor={alerta.codigoSoporte} />}
        {alerta.accion && (
          <button type="button" className="alerta__accion"
                  onClick={() => { alerta.accion!.ejecutar(); onCerrar(alerta.id); }}>
            {alerta.accion.etiqueta}
          </button>
        )}
      </div>
      <button type="button" className="alerta__cerrar" aria-label="Cerrar alerta" onClick={() => onCerrar(alerta.id)}>
        <X aria-hidden="true" />
      </button>
      {!alerta.persistente && (
        <span
          key={alerta.reinicios}                         // Reinicia la barra si el mensaje se repite
          className="alerta__progreso"
          aria-hidden="true"
          style={{ animationDuration: `${alerta.duracionMs}ms`, animationPlayState: enPausa ? 'paused' : 'running' }}
          onAnimationEnd={() => onCerrar(alerta.id)}
        />
      )}
    </div>
  );
}
```

```tsx
// shared/infrastructure/ui/alertas/ContenedorAlertas.tsx
export function ContenedorAlertas() {
  const { visibles, cerrar } = useAlertasStore();
  return createPortal(
    <section className="alertas" aria-label="Notificaciones">
      {visibles.map((a) => <AlertaFlotante key={a.id} alerta={a} onCerrar={cerrar} />)}
    </section>,
    document.body,
  );
}
```

```css
/* shared/infrastructure/ui/alertas/alertas.css (o equivalente con utilidades Tailwind) */
.alertas {
  position: fixed; top: 1rem; right: 1rem; z-index: 1000;
  display: flex; flex-direction: column; gap: .5rem;
  width: min(24rem, calc(100vw - 2rem));
  pointer-events: none;                       /* No bloquea la pantalla fuera de las alertas */
}
.alerta {
  --alerta-color: var(--info);
  pointer-events: auto; position: relative; overflow: hidden;
  display: grid; grid-template-columns: auto 1fr; gap: .75rem;
  padding: .875rem 2.75rem 1.125rem 1rem;      /* Espacio para la X y la barra */
  background: hsl(var(--surface)); color: hsl(var(--foreground));
  border-left: 4px solid hsl(var(--alerta-color));
  border-radius: var(--radius); box-shadow: var(--shadow-lg);
  animation: alerta-entrar 200ms ease-out;
}
.alerta--exito { --alerta-color: var(--success); }
.alerta--info { --alerta-color: var(--info); }
.alerta--advertencia { --alerta-color: var(--warning); }
.alerta--error { --alerta-color: var(--danger); }
.alerta__cerrar {
  position: absolute; top: .375rem; right: .375rem;
  width: 2rem; height: 2rem; display: grid; place-items: center;
  border-radius: var(--radius-sm); color: hsl(var(--muted-foreground));
}
.alerta__cerrar:hover { background: hsl(var(--muted)); }
.alerta__cerrar:focus-visible { outline: 2px solid hsl(var(--ring)); outline-offset: 2px; }
.alerta::after {                                /* Riel tenue de la barra */
  content: ""; position: absolute; left: 0; right: 0; bottom: 0; height: 3px;
  background: hsl(var(--alerta-color) / .12);
}
.alerta__progreso {
  position: absolute; left: 0; bottom: 0; z-index: 1; height: 3px; width: 100%;
  background: hsl(var(--alerta-color) / .6);
  transform-origin: left center;
  animation-name: alerta-progreso; animation-timing-function: linear; animation-fill-mode: forwards;
}
@keyframes alerta-progreso { from { transform: scaleX(1); } to { transform: scaleX(0); } }
@keyframes alerta-entrar { from { opacity: 0; transform: translateX(1rem); } to { opacity: 1; transform: none; } }
@media (max-width: 640px) {
  .alertas { top: max(.5rem, env(safe-area-inset-top)); left: .5rem; right: .5rem; width: auto; }
  .alerta__cerrar { width: 2.75rem; height: 2.75rem; top: 0; right: 0; }
}
@media (prefers-reduced-motion: reduce) {
  .alerta { animation: none; }               /* La barra se mantiene: es información, no decoración */
}
```

> Usar la animación CSS de la barra como reloj único garantiza que la barra y el cierre estén siempre sincronizados, también al pausar. Si el proyecto prefiere una librería de toasts, debe cumplir todo §1 (incluida la barra con pausa) y registrarse con ADR.

### 1.7 Implementación de referencia (móvil, Flutter)
- `AlertasOverlay` en `shared/infrastructure/ui/alertas/`, insertado sobre el `MaterialApp.router` (por encima de la navegación), dentro de `SafeArea`.
- Cada alerta tiene un `AnimationController(duration: duracion)`; la barra es `Align(alignment: Alignment.centerLeft, child: FractionallySizedBox(widthFactor: 1 - controller.value, child: ColoredBox(...)))` de 3 px en la parte inferior; `controller.forward().whenComplete(() => cerrar(id))`.
- Pausa: `controller.stop()` en toque prolongado y cuando `AppLifecycleState` deja de ser `resumed`; reanuda con `controller.forward()` (continúa desde el valor actual).
- X: `IconButton(icon: Icon(Icons.close), tooltip: 'Cerrar alerta')` en la esquina superior derecha, área de 48×48 dp.
- Accesibilidad: `Semantics(liveRegion: true, ...)`; deslizar horizontalmente también cierra.
- Mismo puerto `Notificador` (en Dart) y mismas reglas de §1.2 a §1.4.

### 1.8 Pruebas obligatorias del sistema de alertas
- [ ] Se muestra con el tipo, ícono, rol accesible y texto correctos.
- [ ] Se cierra al hacer clic en la X (que está en la esquina superior derecha y tiene nombre accesible).
- [ ] Se cierra sola al cumplirse la duración (reloj simulado), y la barra existe solo si no es persistente.
- [ ] Pausa con hover, foco y pestaña oculta; continúa sin reiniciar.
- [ ] `Esc` cierra la alerta enfocada; la alerta no roba el foco al aparecer.
- [ ] Máximo 4 visibles; cola; mensajes repetidos no se duplican.
- [ ] `cerrarTodas()` en logout.
- [ ] Sin violaciones de accesibilidad (axe) y contraste AA en claro y oscuro.

---

## 2. Otros componentes de retroalimentación

| Componente | Uso | Accesibilidad |
|---|---|---|
| **Alerta flotante** (§1) | Resultado de acciones (éxito, error, advertencia) e información no ligada a un campo | Ver §1 |
| **Error en campo** | Validación de formularios | `aria-invalid="true"` + `aria-describedby` al mensaje |
| **Resumen de errores** | Formularios largos con varios errores | Al enviar, foco al resumen con enlaces a cada campo |
| **Banner en línea** | Estado de la pantalla que persiste mientras la condición exista (sin conexión, error al cargar, modo solo lectura) | `role="alert"` para errores |
| **Diálogo de confirmación** | Antes de acciones destructivas o irreversibles (no es una alerta: pide decisión) | Foco atrapado; botón por defecto = cancelar |
| **Página de error** | 403, 404, 500 | Título claro y acción para volver |
| **Skeleton** | Carga inicial | `aria-busy="true"` en el contenedor |
| **Estado vacío** | Lista sin resultados | Mensaje + acción sugerida |

Regla: **alerta flotante** para el resultado de algo que el usuario hizo o que acaba de pasar; **banner** para una condición que sigue vigente en la pantalla; **error en campo** para datos que el usuario debe corregir.

## 3. De `kind` a presentación

| kind / code | Presentación | Acción ofrecida |
|---|---|---|
| VALIDATION (con `details`) | Errores en campos + resumen (sin alerta flotante) | Corregir |
| VALIDATION (sin campo) | Alerta de error | — |
| BUSINESS_RULE, CONFLICT | Alerta de error | Según caso: "Recargar" |
| CONFLICTO_DE_VERSION | Diálogo: "Otro usuario modificó este registro." | "Recargar" (pierde cambios) / "Ver cambios" |
| NOT_FOUND (al cargar pantalla) | Página 404 del recurso | "Volver al listado" |
| NOT_FOUND (en acción) | Alerta de error + refrescar lista | — |
| FORBIDDEN (al cargar) | Página 403 | "Ir al inicio" |
| FORBIDDEN (en acción) | Alerta de error + refrescar permisos (`/auth/me`) | — |
| UNAUTHENTICATED | Manejado por el cliente HTTP; en login: alerta de información "Tu sesión expiró. Ingresa nuevamente." | — |
| RATE_LIMITED | Alerta de advertencia con el tiempo de espera si viene `Retry-After` | Deshabilitar el botón durante la espera |
| EXTERNAL, INTERNAL | Alerta de error con código de soporte | "Reintentar" (si la operación es idempotente) |
| SIN_CONEXION | Banner persistente "Sin conexión" + alerta de éxito "Conexión restablecida" al volver | Reintento automático |
| Éxito de una acción | Alerta de éxito con el texto de la HU | Acción opcional ("Ver", "Deshacer") |
| Evento en tiempo real relevante | Alerta de información (agrupada: "Se agregaron N facturas") | "Ver" |

## 4. Estados obligatorios de una pantalla de datos

| Estado | Comportamiento |
|---|---|
| Cargando (primera vez) | Skeleton con la forma del contenido |
| Recargando (ya hay datos) | Indicador discreto; se mantienen los datos anteriores |
| Con datos | Contenido |
| Vacío | Ilustración simple + texto ("Aún no hay facturas") + acción si tiene permiso ("Crear factura") |
| Vacío por filtros | "No hay resultados para estos filtros" + "Limpiar filtros" |
| Error al cargar | Banner con mensaje y "Reintentar" |
| Sin permiso | Página 403 (no se muestra contenido parcial) |

## 5. Acciones
- Botón en estado "procesando" (spinner + texto "Guardando...") y deshabilitado para evitar doble envío.
- Acciones destructivas: diálogo que nombra el objeto ("¿Anular la factura FV-000123?"), pide motivo si el negocio lo exige, botón destructivo en color de peligro. El resultado se informa con una alerta flotante.
- Tras éxito: alerta de éxito + navegación o actualización de la lista; el foco va a un lugar lógico (el elemento creado o el título), no a la alerta.
- Operaciones largas (procesos asíncronos): alerta de información "Generando reporte; te avisaremos al terminar" y, al terminar (evento WS), alerta de éxito con la acción "Descargar".

## 6. Textos
- Español claro, en segunda persona, sin culpar al usuario ("No pudimos..." en vez de "Usted cometió un error").
- Decir qué pasó y qué hacer.
- Sin códigos HTTP, nombres de excepciones ni jerga técnica.
- Consistentes con los criterios de aceptación (los textos exactos están en las HU, ART-05).

## 7. Captura global
- `ErrorBoundary` global y por ruta (web), `FlutterError.onError` + `PlatformDispatcher.instance.onError` (móvil): pantalla "Algo salió mal" con "Recargar" y código de soporte; reporte al servicio de errores (ART-18 §7).
- Errores de TanStack Query: manejador global por defecto (`QueryCache({ onError })`) que llama a `notificador.desdeError(e)` salvo que la query maneje su propio error.
