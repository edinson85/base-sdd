---
id: ART-41
titulo: Pruebas end to end
version: 1.1.0   # 1.1.0: E2E-T10 alertas flotantes
estado: vigente
aplica_a: [frontend-web, movil, backend]
depende_de: [ART-05, ART-38, ART-33]
---

# ART-41 · Pruebas end to end (E2E)

> **Instrucción para agentes IA**
> Cada criterio de aceptación de una HU con prioridad Must se convierte en al menos una prueba E2E, por cada perfil relevante. Las pruebas corren contra el sistema completo (Nginx + API + worker + PostgreSQL + Redis) levantado con Docker Compose.

---

## 1. Ambiente

`docker-compose.e2e.yml` = compose de producción con:
- Imágenes del commit actual.
- Certificado autofirmado y dominio `app.local` (Nginx real con CSP y headers reales).
- BD vacía + migraciones + semillas de prueba (`scripts/seed_e2e.py`).
- Servicio `mailpit` para capturar correos (recuperación de contraseña).
- Simuladores (WireMock o un servicio FastAPI mínimo) para integraciones externas.

## 2. Playwright (web)

### 2.1 Configuración
```ts
// tests/e2e/playwright.config.ts
export default defineConfig({
  testDir: './specs',
  fullyParallel: true,
  retries: process.env.CI ? 1 : 0,          // 1 reintento para detectar flaky; se reporta
  reporter: [['html', { open: 'never' }], ['junit', { outputFile: 'results.xml' }]],
  use: {
    baseURL: 'https://app.local',
    ignoreHTTPSErrors: true,
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    locale: 'es-CO',
    timezoneId: 'America/Bogota',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'mobile', use: { ...devices['Pixel 7'] } },
  ],
});
```

### 2.2 Sesiones por perfil
El access token vive en memoria y el refresh en una cookie HttpOnly que **rota en cada uso**. Por eso no se comparte un `storageState` entre pruebas: si dos pruebas usan el mismo refresh, el backend detecta la reutilización y revoca la sesión (comportamiento correcto, ART-20 §3.2). Cada prueba inicia sesión por API (rápido) y recibe su propia cookie:

```ts
// fixtures.ts
type Perfil = 'admin' | 'cajero' | 'consulta';

export const test = base.extend<{ loginComo: (perfil: Perfil) => Promise<Page> }>({
  loginComo: async ({ browser }, use) => {
    const contextos: BrowserContext[] = [];
    await use(async (perfil) => {
      const ctx = await browser.newContext();
      contextos.push(ctx);
      const resp = await ctx.request.post('/api/v1/auth/login', {
        data: { email: USUARIOS[perfil].email, password: USUARIOS[perfil].password },
      });
      expect(resp.ok()).toBeTruthy();          // La cookie de refresh queda en el contexto
      return ctx.newPage();                     // Al abrir la app, el refresh silencioso restaura la sesión
    });
    await Promise.all(contextos.map((c) => c.close()));
  },
});
```
El flujo de login por la interfaz se prueba aparte (E2E-T01). El rate limit de login en el ambiente E2E se ajusta para permitir la suite (o se usa una IP permitida).

### 2.3 Page Objects
```ts
export class FacturasPage {
  constructor(private page: Page) {}
  async abrir() { await this.page.goto('/facturas'); }
  nuevaFactura() { return this.page.getByRole('button', { name: 'Nueva factura' }); }
  filaPorNumero(numero: string) { return this.page.getByRole('row', { name: new RegExp(numero) }); }
}
```

### 2.4 Ejemplo trazado a la HU
```ts
test.describe('HU-012 Crear factura', () => {
  test('CA-1: creación exitosa y aparición en tiempo real para otro usuario', async ({ loginComo }) => {
    const cajero = await loginComo('cajero');
    const admin = await loginComo('admin');
    await new FacturasPage(admin).abrir();

    await crearFactura(cajero, { numero: 'FV-E2E-001', cliente: 'Ferretería El Tornillo', items: [['Martillo 16 oz', 2]] });

    await expect(cajero.getByText('Factura creada correctamente.')).toBeVisible();
    await expect(new FacturasPage(admin).filaPorNumero('FV-E2E-001')).toBeVisible();   // Llega por WS
  });

  test('CA-S1: perfil consulta no puede crear', async ({ loginComo }) => {
    const page = await loginComo('consulta');
    await page.goto('/facturas');
    await expect(page.getByRole('button', { name: 'Nueva factura' })).toHaveCount(0);
    await page.goto('/facturas/nueva');
    await expect(page).toHaveURL(/\/403$/);
  });
});
```

### 2.5 Flujos transversales obligatorios (todo proyecto)
| ID | Flujo |
|---|---|
| E2E-T01 | Login correcto, incorrecto y bloqueo tras intentos |
| E2E-T02 | Logout: el botón atrás no muestra datos; otra pestaña queda sin sesión |
| E2E-T03 | Expiración de sesión (access y refresh) y cierre por inactividad (con reloj controlado `page.clock`) |
| E2E-T04 | Recuperación de contraseña (correo capturado en mailpit) |
| E2E-T05 | Navegación directa a cada ruta protegida con cada perfil |
| E2E-T06 | Cambio de permisos de un usuario conectado se refleja sin volver a iniciar sesión |
| E2E-T07 | Accesibilidad (axe) en cada pantalla principal (ART-32) |
| E2E-T08 | Sin datos sensibles en `localStorage`/`sessionStorage`/IndexedDB tras recorrer la app |
| E2E-T09 | La app funciona con la CSP de producción (sin errores de CSP en consola) |
| E2E-T10 | Alertas flotantes: aparecen arriba a la derecha sin desplazar el contenido, se cierran con la X, se cierran solas al terminar la barra, se pausan con el puntero encima (ART-29 §1) |

```ts
test('E2E-T10: alerta de éxito se cierra sola y se pausa con hover', async ({ loginComo }) => {
  const page = await loginComo('cajero');
  await crearFactura(page, facturaDePrueba());
  const alerta = page.getByRole('status').filter({ hasText: 'Factura creada correctamente.' });
  await expect(alerta).toBeVisible();
  await expect(alerta.locator('.alerta__progreso')).toBeVisible();

  await alerta.hover();                                   // Pausa
  await page.waitForTimeout(6000);                        // Más que la duración (5 s): sigue visible
  await expect(alerta).toBeVisible();

  await page.mouse.move(0, 0);                            // Reanuda
  await expect(alerta).toHaveCount(0, { timeout: 7000 });
});
```
> Aquí se usa una espera real porque el reloj simulado de Playwright no controla las animaciones CSS; es la única excepción permitida a la regla de no usar esperas fijas (ART-38 §6).

```ts
test('E2E-T09: sin violaciones de CSP', async ({ page }) => {
  const violaciones: string[] = [];
  page.on('console', (m) => { if (m.text().includes('Content Security Policy')) violaciones.push(m.text()); });
  await recorrerPantallasPrincipales(page);
  expect(violaciones).toEqual([]);
});
```

## 3. Móvil
- `integration_test` (o Patrol para interacción con diálogos nativos) contra el backend E2E.
- Mismos flujos transversales aplicables (login, logout, deep links sin permiso, segundo plano).
- Ejecución en emulador Android en CI; iOS en macOS por release.

## 4. Reglas
- Selectores por rol y texto visible (lo que ve el usuario); `data-testid` solo si no hay alternativa.
- Cada prueba crea sus datos (API de semillas de prueba o factories vía endpoint solo habilitado en `APP_ENV=e2e`) con identificadores únicos.
- Sin dependencias entre pruebas.
- Tiempo máximo de la suite: 15 min en paralelo; si crece, dividir por etiquetas (`@critico` en cada PR a main, todo nocturno).
