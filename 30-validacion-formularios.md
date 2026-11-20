---
id: ART-30
titulo: Validación de formularios (web y móvil)
version: 1.0.0
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-19, ART-29, ART-46, ART-47]
---

# ART-30 · Validación de formularios

> **Instrucción para agentes IA**
> - Las reglas de cada campo vienen de la especificación del endpoint (ART-46 §3) y deben ser **las mismas** que valida el backend (ART-19). Usa los esquemas equivalentes de §2.
> - La validación del cliente es para la experiencia de usuario; el backend siempre vuelve a validar (P-VA-06).

---

## 1. Comportamiento (UX)

| Momento | Regla |
|---|---|
| Mientras escribe | No mostrar errores (salvo contador de caracteres cerca del límite) |
| Al salir del campo (blur) | Validar ese campo y mostrar su error |
| Al enviar | Validar todo; foco al primer campo con error (o al resumen si hay > 3) |
| Después de un error | Revalidar al escribir en ese campo para quitar el error en cuanto se corrija |
| Errores del servidor | Mapear `details[].field` al campo; los que no tienen campo, a un banner del formulario |

Atributos HTML: `required`, `maxLength`, `inputMode` (`numeric`, `email`, `tel`, `decimal`), `autocomplete` correcto, `type` adecuado. Etiquetas visibles siempre (no solo placeholder).

## 2. Esquemas equivalentes al backend

### 2.1 Web (`shared/infrastructure/forms/tipos.ts`, Zod)

```ts
import { z } from 'zod';

const LETRAS = 'A-Za-zÁÉÍÓÚáéíóúÑñÜü';
const normalizar = (v: string) => v.normalize('NFC').replace(/[\u0000-\u0008\u000B\u000C\u000E-\u001F\u007F​-‏‪-‮⁦-⁩]/g, '').trim();
const texto = () => z.string().transform(normalizar);

export const nombrePersona = texto().pipe(z.string()
  .min(2, 'Debe tener al menos 2 caracteres.')
  .max(80, 'Debe tener máximo 80 caracteres.')
  .regex(new RegExp(`^[${LETRAS}]+(?:[ '\\-][${LETRAS}]+)*$`), 'Solo letras, espacios, apóstrofes y guiones.'));

export const codigo = texto().pipe(z.string()
  .min(1, 'Este campo es obligatorio.').max(30, 'Debe tener máximo 30 caracteres.')
  .regex(/^[A-Z0-9][A-Z0-9\-_]*$/, 'Solo letras mayúsculas, números, guiones y guion bajo.'));

export const textoCorto = texto().pipe(z.string()
  .min(1, 'Este campo es obligatorio.').max(200, 'Debe tener máximo 200 caracteres.')
  .regex(new RegExp(`^[${LETRAS}0-9 .,;:¿?¡!'"()\\-_/#%@+]+$`), 'Contiene caracteres no permitidos.'));

export const email = texto().pipe(z.string().max(254).email('Ingresa un correo válido.'));
export const telefonoCO = texto().pipe(z.string().regex(/^(\+57)?3\d{9}$|^(\+57)?60\d{8}$/, 'Ingresa un celular (10 dígitos) o fijo válido.'));
export const cedulaCO = texto().pipe(z.string().regex(/^\d{5,10}$/, 'Debe tener entre 5 y 10 dígitos.'));
export const cantidad = z.coerce.number({ message: 'Ingresa un número.' })
  .int('Debe ser un número entero.').min(1, 'Debe ser mayor o igual a 1.').max(1_000_000);
export const dinero = z.string()
  .regex(/^\d{1,10}(\.\d{1,2})?$/, 'Ingresa un valor válido con máximo 2 decimales.');
export const password = z.string().min(12, 'Debe tener al menos 12 caracteres.').max(128);
```

Esquema de formulario = composición de tipos + reglas del caso:
```ts
export const nuevaFacturaSchema = z.object({
  numero: z.string().transform(normalizar).pipe(z.string().min(3).max(20)
    .regex(/^[A-Z0-9\-]+$/, 'Solo letras mayúsculas, números y guiones.')),
  clienteId: z.string().uuid('Selecciona un cliente.'),
  items: z.array(z.object({
    productoId: z.string().uuid('Selecciona un producto.'),
    descripcion: textoCorto,
    cantidad,
    precioUnitario: dinero,
  })).min(1, 'Agrega al menos un producto.').max(200, 'Máximo 200 productos por factura.'),
});
```

### 2.2 Móvil (`shared/infrastructure/forms/validadores.dart`)

```dart
typedef Validador = String? Function(String? v);

String normalizar(String v) => v.trim(); // + eliminación de caracteres de control como en web

Validador requerido([String msg = 'Este campo es obligatorio.']) =>
    (v) => (v == null || normalizar(v).isEmpty) ? msg : null;

Validador longitud(int min, int max) => (v) {
  final n = normalizar(v ?? '').runes.length;
  if (n < min) return 'Debe tener al menos $min caracteres.';
  if (n > max) return 'Debe tener máximo $max caracteres.';
  return null;
};

Validador patron(RegExp re, String msg) => (v) => re.hasMatch(normalizar(v ?? '')) ? null : msg;

Validador todos(List<Validador> vs) => (v) {
  for (final f in vs) { final e = f(v); if (e != null) return e; }
  return null;
};
```

### 2.3 Sincronía con el backend
- Cada tipo reutilizable existe en los tres lugares con el **mismo nombre** (ART-19 §2 ↔ §2.1 ↔ §2.2).
- Opcional (ADR): generar los esquemas del front desde el OpenAPI para los formularios simples; los tipos de §2.1 siguen siendo la referencia de mensajes.
- Prueba: un archivo de casos compartido (`docs/validacion-casos.yaml`: valor, tipo, válido sí/no) que ejecutan pruebas en back, web y móvil.

## 3. Mensajes por regla
| rule (backend) | Mensaje |
|---|---|
| `missing` | Este campo es obligatorio. |
| `string_too_short` | Debe tener al menos {min} caracteres. |
| `string_too_long` | Debe tener máximo {max} caracteres. |
| `string_pattern_mismatch` | El formato no es válido. (o el mensaje específico del tipo) |
| `greater_than_equal` / `less_than_equal` | Debe ser mayor o igual a {min} / menor o igual a {max}. |
| `int_parsing`, `decimal_parsing` | Ingresa un número válido. |
| `uuid_parsing` | Selecciona una opción válida. |
| `extra_forbidden` | (No se muestra: indica un bug del cliente; se registra) |

## 4. Campos especiales
| Campo | Regla |
|---|---|
| Dinero | Entrada con separadores de miles visuales (`es-CO`), se envía como string con punto decimal |
| Fechas | Selector de fecha; se envía ISO `AAAA-MM-DD`; validar rangos (no futura, etc.) |
| Archivos | Validar tipo y tamaño antes de subir; mostrar progreso; el backend valida por contenido |
| Selects de entidades | Autocompletar con búsqueda al servidor (mín. 2 caracteres, debounce 300 ms) |
| Contraseña | Indicador de longitud, mostrar/ocultar, sin pegar bloqueado (permitir gestores de contraseñas) |

## 5. Pruebas
- Por formulario: casos válidos, cada regla inválida con su mensaje, mapeo de errores del servidor a campos, doble envío bloqueado.
- Los casos de `validacion-casos.yaml` pasan en back, web y móvil.
