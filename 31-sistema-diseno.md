---
id: ART-31
titulo: Sistema de diseño / UI kit
version: 1.1.0   # 1.1.0: alertas flotantes (ART-29 §1) y origen de tokens en la propuesta de interfaz (ART-49)
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-07, ART-11, ART-12, ART-29, ART-32, ART-49]
---

# ART-31 · Sistema de diseño / UI kit

> **Instrucción para agentes IA**
> - Las pantallas se construyen **solo** con componentes de `shared/infrastructure/ui` y tokens de diseño. Prohibido usar colores, tamaños o espaciados literales en los módulos.
> - Si falta un componente, agrégalo a `shared/infrastructure/ui` (con sus estados y pruebas de accesibilidad) antes de usarlo.
> - La identidad visual (colores de marca, tipografía) la define cada proyecto en `tokens`; la estructura es común.

---

## 1. Base tecnológica
- Web: Tailwind CSS + componentes accesibles sobre primitivas Radix (estilo shadcn/ui, copiados al repositorio), íconos `lucide-react` (ADR-BASE-06).
- Móvil: `ThemeData` de Material 3 con `ColorScheme` y `TextTheme` generados desde los mismos tokens; componentes propios en `shared/infrastructure/ui`.

## 2. Tokens

Fuente única `design-tokens.json` (en `docs/` o paquete compartido) → genera variables CSS para Tailwind y constantes Dart. Los valores de marca (colores, tipografía, logo) se definen y aprueban en la **propuesta de interfaz** del proyecto (ART-49) y de allí pasan a este archivo.

| Categoría | Tokens |
|---|---|
| Color semántico | `primary`, `primary-foreground`, `secondary`, `background`, `surface`, `foreground`, `muted`, `muted-foreground`, `border`, `ring`, `success`, `warning`, `danger`, `info` (cada uno con `-foreground`) |
| Tipografía | `font-sans`; escala `xs 12`, `sm 14`, `base 16`, `lg 18`, `xl 20`, `2xl 24`, `3xl 30`; pesos 400/500/600/700; interlineado 1.5 en texto |
| Espaciado | Escala de 4 px: `1=4`, `2=8`, `3=12`, `4=16`, `6=24`, `8=32`, `12=48` |
| Radios | `sm 4`, `md 8`, `lg 12`, `full` |
| Sombras | `sm`, `md`, `lg` |
| Movimiento | `duration-fast 120ms`, `duration-normal 200ms`; respetar `prefers-reduced-motion` |
| Alertas | `alerta-ancho 384px`, `alerta-max-visibles 4`, `alerta-barra-alto 3px`, duraciones `exito 5s`, `info 6s`, `advertencia 8s`, `error 10s` (ART-29 §1.2) |
| Breakpoints | `sm 640`, `md 768`, `lg 1024`, `xl 1280` |

```css
/* Ejemplo (tema claro); el oscuro redefine las mismas variables bajo .dark */
:root {
  --background: 0 0% 100%;
  --foreground: 222 47% 11%;
  --primary: 221 83% 53%;
  --primary-foreground: 0 0% 100%;
  --danger: 0 72% 51%;
  --danger-foreground: 0 0% 100%;
  --border: 214 32% 91%;
  --radius: 0.5rem;
}
```

Todos los pares color/fondo cumplen contraste WCAG AA (4.5:1 texto normal, 3:1 texto grande y componentes) en claro y oscuro.

## 3. Catálogo de componentes

| Grupo | Componentes |
|---|---|
| Acciones | `Button` (primary, secondary, outline, ghost, danger; tamaños sm/md/lg; estados loading/disabled), `IconButton` (con `aria-label` obligatorio), `Link` |
| Formularios | `Form`, `Field` (label + control + ayuda + error), `Input`, `Textarea`, `Select`, `Combobox` (búsqueda remota), `Checkbox`, `RadioGroup`, `Switch`, `DatePicker`, `MoneyInput`, `FileUpload`, `PasswordInput` |
| Datos | `DataTable` (orden, paginación servidor, selección, columnas por permiso, estado vacío/carga), `DescriptionList`, `Badge` (estados de negocio), `Card`, `Stat`, `DatoSensible` (ART-27 §4) |
| Navegación | `AppShell` (sidebar + topbar), `Sidebar` (ítems filtrados por permiso), `Breadcrumbs`, `Tabs`, `Pagination` |
| Retroalimentación | `AlertaFlotante` + `ContenedorAlertas` (flotantes, X arriba a la derecha, cierre automático con barra de tiempo; especificación completa en ART-29 §1), `Banner` (alerta en línea persistente), `Dialog`, `ConfirmDialog`, `Skeleton`, `Spinner`, `EmptyState`, `ErrorState`, `ProgressBar` |
| Seguridad | `RequirePermission`, `Can`, `IdleWarningDialog` |
| Páginas | `LoginPage`, `ForbiddenPage` (403), `NotFoundPage` (404), `ErrorPage` (500), `OfflineBanner` |

Cada componente documenta: props, variantes, estados (hover, focus, active, disabled, loading, error), comportamiento de teclado y rol ARIA.

## 4. Patrones de pantalla

| Patrón | Estructura |
|---|---|
| Listado | Título + acción principal (con `Can`) → filtros → `DataTable` → paginación |
| Detalle | Breadcrumbs → título + estado (`Badge`) + acciones → secciones (`Card`/`DescriptionList`) → historial |
| Formulario de creación/edición | Título → secciones con `Field` → barra de acciones (Cancelar / Guardar) fija al final |
| Tablero | Fila de `Stat` → gráficos → listas recientes (actualizadas por WS) |

Responsive: listados se convierten en tarjetas bajo `md`; formularios a una columna bajo `md`.

## 5. Contenido
- Español, mayúscula solo inicial en títulos y botones ("Crear factura").
- Verbos en botones (Guardar, Anular, Exportar), no "Aceptar"/"OK" genéricos.
- Formatos: moneda `es-CO` COP, fechas `dd/mm/aaaa` y horas de 24 h en `America/Bogota`, números con separador de miles.

## 6. Pruebas
- Cada componente con prueba de accesibilidad automática (`vitest-axe`) y pruebas de teclado.
- Catálogo visual opcional con Storybook (ADR) para revisión de diseño.
