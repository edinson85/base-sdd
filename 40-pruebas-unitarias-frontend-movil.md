---
id: ART-40
titulo: Pruebas unitarias y de componentes – Frontend y móvil
version: 1.1.0   # 1.1.0: pruebas de alertas flotantes (§2.5)
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-11, ART-12, ART-38]
---

# ART-40 · Pruebas unitarias y de componentes – Frontend y móvil

> **Instrucción para agentes IA**
> Base en ART-11 §10 y ART-12 §6. Prioriza pruebas de comportamiento desde el punto de vista del usuario (roles, textos, etiquetas), no de implementación.

---

## 1. Web: configuración

```ts
// vitest.config.ts
export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      include: ['src/**/domain/**', 'src/**/application/**'],
      thresholds: { lines: 80, branches: 70 },
    },
  },
  resolve: { alias: { '@': '/src' } },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { server } from './msw-server';
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Helper de renderizado con dependencias y sesión:
```tsx
export function renderConApp(ui: ReactElement, { permisos = [] as Permiso[], ruta = '/' } = {}) {
  const container = createTestContainer({ permisos });
  return render(
    <DependenciesProvider value={container}>
      <QueryClientProvider client={container.queryClient}>
        <MemoryRouter initialEntries={[ruta]}>{ui}</MemoryRouter>
      </QueryClientProvider>
    </DependenciesProvider>,
  );
}
```

## 2. Web: patrones

### 2.1 Dominio y casos de uso
Funciones puras y casos de uso con repositorios fake (ver ART-11 §10). Sin React, sin MSW.

### 2.2 Adaptador API (con MSW)
```ts
it('traduce el error del backend a AppError por code', async () => {
  server.use(http.post('/api/v1/facturas', () =>
    HttpResponse.json({ error: { code: 'FACTURA_DUPLICADA', message: 'Ya existe...', details: [], trace_id: 't1' } }, { status: 409 })));
  const repo = new FacturaApiRepository(crearHttpClientDePrueba());
  await expect(repo.crear(nuevaFacturaValida())).rejects.toMatchObject({ code: 'FACTURA_DUPLICADA', kind: 'CONFLICT' });
});

it('rechaza respuestas que no cumplen el contrato', async () => {
  server.use(http.get('/api/v1/facturas', () => HttpResponse.json({ data: [{ id: 1 }], meta: {} })));
  await expect(new FacturaApiRepository(crearHttpClientDePrueba()).listar({}, { number: 1, size: 20 }))
    .rejects.toBeInstanceOf(Error);
});

it('ante 401 refresca una sola vez y reintenta', async () => { /* ... */ });
```

### 2.3 Componentes y pantallas
```tsx
it('no muestra "Nueva factura" sin permiso factura:crear', async () => {
  renderConApp(<FacturasPage />, { permisos: ['factura:ver'] });
  expect(await screen.findByRole('heading', { name: 'Facturas' })).toBeInTheDocument();
  expect(screen.queryByRole('button', { name: 'Nueva factura' })).not.toBeInTheDocument();
});

it('muestra el error del servidor en el campo correspondiente', async () => {
  server.use(http.post('/api/v1/facturas', () => HttpResponse.json(
    { error: { code: 'VALIDACION_FALLIDA', message: '...', trace_id: 't', details: [{ field: 'numero', rule: 'string_too_short' }] } },
    { status: 422 })));
  const user = userEvent.setup();
  renderConApp(<NuevaFacturaPage />, { permisos: ['factura:crear'] });
  // ... llenar y enviar
  expect(await screen.findByText('Debe tener al menos 3 caracteres.')).toBeInTheDocument();
  expect(screen.getByLabelText('Número')).toHaveAttribute('aria-invalid', 'true');
});

it('el listado no tiene violaciones de accesibilidad', async () => {
  const { container } = renderConApp(<FacturasPage />, { permisos: ['factura:ver'] });
  await screen.findByRole('table');
  expect(await axe(container)).toHaveNoViolations();
});
```

### 2.4 Seguridad
```ts
it('cerrarSesion limpia todo aunque falle la revocación', async () => {
  const deps = depsFalsas({ revocarFalla: true });
  await cerrarSesion(deps)('manual');
  expect(deps.realtime.desconectar).toHaveBeenCalled();
  expect(deps.cache.limpiar).toHaveBeenCalled();
  expect(deps.session.limpiar).toHaveBeenCalled();
  expect(deps.tabs.notificarCierre).toHaveBeenCalledWith('manual');
  expect(deps.notificador.cerrarTodas).toHaveBeenCalled();
});

it.each(['//evil.com', 'https://evil.com', '/\\evil.com', 'javascript:alert(1)'])(
  'no redirige a destinos externos: %s', (destino) => {
    expect(destinoSeguro(destino)).toBe('/');
  });
```

### 2.5 Alertas flotantes (ART-29 §1.8)
```tsx
describe('AlertaFlotante', () => {
  beforeEach(() => vi.useFakeTimers());

  it('se cierra con la X de la esquina superior derecha', async () => {
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    const { notificador } = renderConAlertas();
    act(() => { notificador.exito('Factura creada correctamente.'); });
    await user.click(screen.getByRole('button', { name: 'Cerrar alerta' }));
    expect(screen.queryByText('Factura creada correctamente.')).not.toBeInTheDocument();
  });

  it('se cierra sola cuando la barra de tiempo termina', () => {
    const { notificador } = renderConAlertas();
    act(() => { notificador.exito('Guardado.'); });
    const barra = document.querySelector('.alerta__progreso')!;
    expect(barra).toHaveStyle({ animationDuration: '5000ms' });
    fireEvent.animationEnd(barra);                 // jsdom no ejecuta animaciones CSS: se simula su fin
    expect(screen.queryByText('Guardado.')).not.toBeInTheDocument();
  });

  it('pausa la barra al pasar el puntero y al enfocar', async () => {
    const { notificador } = renderConAlertas();
    act(() => { notificador.error('No se pudo guardar.'); });
    const alerta = screen.getByRole('alert');
    fireEvent.mouseEnter(alerta);
    expect(document.querySelector('.alerta__progreso')).toHaveStyle({ animationPlayState: 'paused' });
    fireEvent.mouseLeave(alerta);
    expect(document.querySelector('.alerta__progreso')).toHaveStyle({ animationPlayState: 'running' });
  });

  it('las persistentes no tienen barra ni se cierran solas', () => {
    const { notificador } = renderConAlertas();
    act(() => { notificador.error('Falló el envío.', { persistente: true }); });
    expect(document.querySelector('.alerta__progreso')).toBeNull();
  });

  it('muestra máximo 4 y encola el resto', () => {
    const { notificador } = renderConAlertas();
    act(() => { for (let i = 1; i <= 6; i++) notificador.info(`Mensaje ${i}`); });
    expect(screen.getAllByRole('status')).toHaveLength(4);
  });
});
```
> El cierre real por tiempo (animación CSS completa) se verifica en E2E con Playwright (ART-41, E2E-T10).

## 3. Móvil (Flutter)

```dart
// Caso de uso con mocktail
class MockRepo extends Mock implements FacturaRepository {}

test('no llama al repositorio sin permiso', () async {
  final repo = MockRepo();
  final session = FakeSession(permisos: {'factura:ver'});
  expect(() => AnularFactura(repo, session)(facturaEmitida, 'x'),
         throwsA(isA<AppError>().having((e) => e.code, 'code', 'ACCESO_DENEGADO')));
  verifyNever(() => repo.anular(any(), any()));
});

// Widget
testWidgets('oculta el botón anular sin permiso', (tester) async {
  await tester.pumpWidget(appDePrueba(permisos: {'factura:ver'}, child: const FacturaDetallePage(id: '1')));
  await tester.pumpAndSettle();
  expect(find.text('Anular'), findsNothing);
});

testWidgets('cumple guías de accesibilidad', (tester) async {
  final handle = tester.ensureSemantics();
  await tester.pumpWidget(appDePrueba(child: const LoginPage()));
  await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
  await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
  await expectLater(tester, meetsGuideline(textContrastGuideline));
  handle.dispose();
});
```

Interceptores de dio: probar refresh único ante 401 concurrentes, traducción de errores y cierre de sesión si el refresh falla.

## 4. Reglas
- Consultas por rol y nombre accesible (`getByRole`, `getByLabelText`) antes que `getByTestId`.
- `userEvent` (no `fireEvent`) para interacciones.
- MSW con `onUnhandledRequest: 'error'`: ninguna llamada real escapa.
- Sin snapshots de árboles grandes; snapshots solo para salidas pequeñas y estables.
- Golden tests en Flutter solo para componentes del design system (opcional).
