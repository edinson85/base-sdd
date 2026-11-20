---
id: ART-27
titulo: Seguridad frontend web
version: 1.0.0
estado: vigente
aplica_a: [frontend-web]
depende_de: [ART-02, ART-11, ART-20, ART-34]
---

# ART-27 · Seguridad frontend web

> **Instrucción para agentes IA**
> ART-11 §6 define la implementación base (sesión, guardas, logout). Este documento amplía las amenazas y controles y es la lista de verificación de seguridad de toda pantalla. Principios P-SF-01..08.
> Regla mental: **el frontend no es una barrera de seguridad**; es una capa para no exponer datos ni facilitar el abuso. La autoridad es el backend.

---

## 1. Amenazas y controles

| Amenaza | Control |
|---|---|
| XSS (robo de token en memoria, acciones en nombre del usuario) | React escapa por defecto; prohibido `dangerouslySetInnerHTML` salvo `SafeHtml` (DOMPurify); CSP estricta sin `unsafe-inline` en scripts (ART-34); no construir URLs `javascript:` (validar `href` con lista de esquemas `https:`, `mailto:`, `tel:`) |
| Robo de sesión por almacenamiento | Tokens solo en memoria + cookie HttpOnly (ART-11 §6.1) |
| Acceso a pantallas sin permiso | `RequirePermission` en todas las rutas; deny by default; `PUBLIC_ROUTES` explícitas |
| Exposición de datos tras logout | Limpieza total (ART-11 §6.3) + `Cache-Control: no-store` + `Clear-Site-Data` |
| Datos de un usuario visibles para otro en equipos compartidos | Cierre por inactividad; nada persistido en el navegador; autocompletado desactivado en campos sensibles |
| Open redirect | Solo rutas internas en parámetros `from`/`next` |
| Clickjacking | `frame-ancestors 'none'` (Nginx) |
| CSRF | Bearer token para el API; refresh con cookie `SameSite=Strict` + encabezado personalizado (ART-21 §4) |
| Fuga por URL | Sin datos personales ni tokens en rutas o query; `Referrer-Policy` estricta |
| Fuga por terceros | Sin scripts de terceros por defecto; si se requieren (analítica, chat), ADR + CSP específica + SRI + sin datos personales |
| Dependencias maliciosas o vulnerables | `npm ci`, `npm audit`, revisión de paquetes nuevos, lockfile en el repositorio |
| Ingeniería inversa del bundle | Asumir que todo el código y las variables `VITE_*` son públicos; sin lógica secreta ni llaves |
| Manipulación de la UI (habilitar botones, editar el DOM) | El backend valida todo; la UI no envía campos calculados ni de control |

## 2. Manejo de sesión en múltiples pestañas
- `BroadcastChannel('app-session')`: mensajes `logout`, `token-refreshed` (sin el token), `permisos-cambiados`.
- Al recibir `logout`: `cerrarSesion('otra-pestana')` sin volver a llamar al backend.
- El refresh se coordina entre pestañas con `navigator.locks.request('refresh', ...)` para evitar refrescos simultáneos que disparen la detección de reutilización (ART-20 §3.2).

## 3. Inactividad
- `IdleTimer` escucha `pointerdown`, `keydown`, `visibilitychange` (sin `mousemove` para no despertar por movimientos involuntarios).
- Configuración por proyecto (`IDLE_TIMEOUT_MINUTES`, por defecto 30; menor para sistemas con datos financieros o de salud).
- Aviso 60 s antes con opción "Seguir conectado" (que hace refresh).
- Al vencer: `cerrarSesion('inactividad')` y mensaje en login: "Tu sesión se cerró por inactividad."

## 4. Visualización de datos sensibles
- Componente `DatoSensible` que muestra enmascarado (`***4567`) y permite "Mostrar" solo si el actor tiene el permiso `<recurso>:ver_sensible`; la acción de mostrar puede requerir una llamada al backend que se audita.
- Nunca incluir datos sensibles en: `document.title`, notificaciones del navegador, mensajes de error, nombres de archivos descargados, eventos de analítica.
- Descargas de archivos: por `fetch` autenticado y `URL.createObjectURL`, revocando la URL tras usarla.

## 5. Formularios
- `autocomplete="off"` en campos de datos sensibles de terceros; `autocomplete="current-password"` / `"new-password"` correctos en login para que el gestor de contraseñas funcione.
- Nunca precargar contraseñas ni mostrarlas en texto plano por defecto.
- Limpiar el estado del formulario al desmontar pantallas con datos sensibles.

## 6. Build y despliegue
- `build.sourcemap: false` (o `hidden` y subidos solo al servicio de monitoreo).
- `esbuild.drop: ['console', 'debugger']` en producción.
- Integridad: los archivos llevan hash en el nombre; `index.html` con `no-store` (ART-34).
- Variables de entorno revisadas en el PR: ninguna con apariencia de secreto (`KEY`, `SECRET`, `TOKEN`, `PASSWORD`) en `VITE_*`.

## 7. Lista de verificación por pantalla
- [ ] Ruta con `permiso` y acciones con `Can`.
- [ ] Solo muestra campos que el perfil puede ver; datos sensibles enmascarados.
- [ ] Sin `dangerouslySetInnerHTML`; enlaces externos con `rel="noopener noreferrer"` y validados.
- [ ] Sin datos sensibles en URL, título, logs o analítica.
- [ ] Estados de error sin detalles técnicos.
- [ ] Funciona con la CSP de producción (probado en E2E contra Nginx).
- [ ] Tras logout y botón atrás no se ven datos (E2E).

## 8. Pruebas adversarias del front (ver ART-42)
- Inyectar `<img src=x onerror=alert(1)>`, `javascript:alert(1)` y `"><svg onload=...>` en todos los campos de texto y verificar que se muestran como texto.
- Navegar directamente a rutas sin permiso y a ids de otra empresa.
- Modificar respuestas con un proxy (Playwright `route`) para inyectar campos extra o tipos inválidos: la validación Zod del adaptador debe rechazarlos sin romper la app.
- Cerrar sesión en una pestaña y verificar que la otra queda sin datos.
- Revisar `localStorage`, `sessionStorage`, IndexedDB y Cache Storage después de usar la app: no deben contener tokens ni datos personales.
