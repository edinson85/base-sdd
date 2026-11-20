---
id: ART-32
titulo: Accesibilidad
version: 1.1.0   # 1.1.0: accesibilidad de alertas flotantes
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-29, ART-31]
---

# ART-32 · Accesibilidad

> **Instrucción para agentes IA**
> Nivel objetivo: **WCAG 2.2 AA**. Aplica esta lista a toda pantalla y componente. Las pruebas automáticas detectan una parte de los problemas; los ítems de teclado y lector de pantalla se verifican en las pruebas de componentes y E2E.

---

## 1. Lista de verificación (web)

**Estructura**
- [ ] Un `<h1>` por página; jerarquía de encabezados sin saltos.
- [ ] Regiones semánticas: `<header>`, `<nav>`, `<main>`, `<footer>`.
- [ ] Enlace "Saltar al contenido" al inicio.
- [ ] `lang="es"` en `<html>`; `document.title` descriptivo por ruta (sin datos sensibles).

**Teclado y foco**
- [ ] Todo es operable con teclado (Tab, Shift+Tab, Enter, Espacio, Esc, flechas en menús y tablas).
- [ ] Foco visible siempre (anillo con contraste ≥ 3:1); nunca `outline: none` sin reemplazo.
- [ ] Orden de foco lógico; diálogos atrapan el foco y lo devuelven al cerrar.
- [ ] Al cambiar de ruta, el foco va al `<h1>` o a `<main>`.
- [ ] Objetivos táctiles de al menos 24×24 px (WCAG 2.2) — preferible 44×44 en móvil.

**Formularios**
- [ ] Toda entrada tiene `<label>` asociado (no solo placeholder).
- [ ] Errores con texto (no solo color), asociados con `aria-describedby`, `aria-invalid`.
- [ ] Campos obligatorios indicados en texto o con `aria-required`.
- [ ] `autocomplete` adecuado para datos del usuario.
- [ ] No se pide reescribir información ya ingresada en el mismo proceso (WCAG 2.2, "entrada redundante").
- [ ] Autenticación sin pruebas cognitivas (permitir pegar contraseñas y usar gestores).

**Contenido visual**
- [ ] Contraste ≥ 4.5:1 en texto normal y 3:1 en texto grande e íconos significativos.
- [ ] El color no es el único medio para transmitir información (estados con texto o ícono).
- [ ] Imágenes informativas con `alt`; decorativas con `alt=""`.
- [ ] Zoom al 200 % y ancho de 320 px sin pérdida de contenido ni scroll horizontal (salvo tablas).
- [ ] Respeta `prefers-reduced-motion`.

**Contenido dinámico**
- [ ] Alertas flotantes y actualizaciones en tiempo real anunciadas (`role="status"` para éxito/información, `role="alert"` para advertencia/error) sin robar el foco.
- [ ] Alertas con cierre automático: el tiempo se pausa con hover y con foco (WCAG 2.2.1 tiempo ajustable), se pueden cerrar con la X (nombre accesible "Cerrar alerta") y con `Esc`; la barra de tiempo es `aria-hidden`; las alertas con acción duran ≥ 10 s o son persistentes (ART-29 §1).
- [ ] Cargas con `aria-busy`.
- [ ] Sin límites de tiempo sin aviso (el cierre por inactividad avisa y permite extender, ART-27 §3).

**Tablas**
- [ ] `<th scope>`, `<caption>` o `aria-label`; orden indicado con `aria-sort`.

## 2. Móvil (Flutter)
- [ ] `Semantics` en widgets personalizados (`label`, `button: true`, etc.); íconos con `semanticLabel` o `tooltip`.
- [ ] Respeta el tamaño de texto del sistema (`MediaQuery.textScaler`) hasta 200 % sin cortar contenido.
- [ ] Objetivos táctiles ≥ 48×48 dp.
- [ ] Orden de lectura lógico con TalkBack y VoiceOver.
- [ ] Contraste como en web.

## 3. Herramientas
| Herramienta | Uso |
|---|---|
| `eslint-plugin-jsx-a11y` | Lint en desarrollo y CI |
| `vitest-axe` / `jest-axe` | Pruebas de componentes |
| `@axe-core/playwright` | Pruebas E2E por pantalla (sin violaciones serias o críticas) |
| Lector de pantalla (NVDA, VoiceOver) | Revisión manual de flujos críticos por release |
| `flutter test` con `meetsGuideline(androidTapTargetGuideline)`, `textContrastGuideline`, `labeledTapTargetGuideline` | Móvil |

```ts
// E2E
import AxeBuilder from '@axe-core/playwright';
test('listado de facturas es accesible', async ({ page }) => {
  await page.goto('/facturas');
  const { violations } = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag22aa']).analyze();
  expect(violations.filter(v => ['serious', 'critical'].includes(v.impact ?? ''))).toEqual([]);
});
```
