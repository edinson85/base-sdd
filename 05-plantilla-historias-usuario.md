---
id: ART-05
titulo: Plantilla de historias de usuario y criterios de aceptación
version: 1.1.0   # 1.1.0: criterio transversal CA-T9 (alertas flotantes)
estado: vigente
aplica_a: [todos]
depende_de: [ART-04]
---

# ART-05 · Historias de usuario y criterios de aceptación

> **Instrucción para agentes IA**
> - Cada HU cumple INVEST: Independiente, Negociable, Valiosa, Estimable, Pequeña (≤ 1 sprint), Testeable.
> - Cada HU tiene criterios **positivos, negativos y de seguridad**. Una HU sin criterios negativos no está lista (ART-06 DoR).
> - Los criterios se escriben en Gherkin en español para que se conviertan directamente en pruebas E2E.

---

## 1. Plantilla

````markdown
### HU-NNN · <Título corto en infinitivo>

| Campo | Valor |
|---|---|
| Épica | EP-NN |
| Perfil(es) | <Perfil> |
| Permiso(s) | `<recurso>:<accion>` |
| Prioridad | Alta / Media / Baja (MoSCoW: Must/Should/Could/Won't) |
| Estimación | <puntos> |
| Plataformas | Web / Móvil / Ambas |
| Dependencias | HU-xxx |

**Como** <perfil>
**quiero** <acción>
**para** <beneficio de negocio>

#### Criterios de aceptación

**CA-1 · <Caso feliz>**
```gherkin
Dado que soy un usuario con el perfil "<perfil>"
Y <precondición>
Cuando <acción>
Entonces <resultado observable>
Y <efecto adicional: evento, notificación, registro>
```

**CA-2 · <Validación de datos>**
```gherkin
Dado ...
Cuando ingreso <dato inválido>
Entonces veo el mensaje "<mensaje exacto>" en el campo <campo>
Y no se guarda ningún cambio
```

**CA-3 · <Regla de negocio>**

**CA-S1 · Sin permiso**
```gherkin
Dado que soy un usuario sin el permiso "<permiso>"
Cuando intento <acción> (por la interfaz o directamente por la URL/API)
Entonces no veo la opción en la interfaz
Y si accedo por URL soy redirigido a la página de acceso denegado
Y el API responde 403 ACCESO_DENEGADO
```

**CA-S2 · Datos de otro alcance**
```gherkin
Dado que existe un <recurso> de otra empresa
Cuando intento consultarlo o modificarlo por su identificador
Entonces el sistema responde que no existe (404)
```

#### Notas
- Reglas de negocio: RN-x
- Datos sensibles involucrados: <...>
- Tiempo real: <sí/no, qué se actualiza>
- Fuera de alcance de esta HU: <...>
````

---

## 2. Reglas de redacción

| Regla | Correcto | Incorrecto |
|---|---|---|
| Resultado observable | "Veo la factura en el listado con estado EMITIDA" | "La factura se guarda bien" |
| Mensajes exactos | `Entonces veo "Ya existe una factura con ese número."` | "Se muestra un error" |
| Sin detalles de implementación en el "quiero" | "quiero anular una factura" | "quiero llamar a PATCH /facturas" |
| Datos concretos en ejemplos | "Cuando ingreso cantidad 0" | "Cuando ingreso una cantidad inválida" |
| Un comportamiento por criterio | CA separados | CA con 5 "Y" de cosas distintas |

Para variaciones de datos, usa `Esquema del escenario` con `Ejemplos`:

```gherkin
Esquema del escenario: Validación del número de factura
  Cuando ingreso "<numero>" en el campo número
  Entonces veo "<mensaje>"
  Ejemplos:
    | numero        | mensaje                                         |
    | AB            | Debe tener entre 3 y 20 caracteres.             |
    | FV 001        | Solo se permiten letras mayúsculas, números y guiones. |
    | FV-000123     |                                                 |
```

---

## 3. Criterios transversales que se agregan automáticamente

Toda HU que implique un formulario hereda (no hace falta repetirlos, se referencian como `CA-T*`):

| ID | Criterio |
|---|---|
| CA-T1 | Los campos obligatorios muestran error si están vacíos al enviar |
| CA-T2 | El botón de envío se deshabilita mientras se procesa |
| CA-T3 | Un error inesperado muestra un mensaje genérico con código de soporte (`trace_id`) |
| CA-T4 | Sin conexión se muestra "No hay conexión con el servidor..." y se puede reintentar |
| CA-T5 | La pantalla es usable con teclado y lector de pantalla (ART-32) |
| CA-T9 | El resultado de cada acción (éxito, advertencia o error) se informa con una alerta flotante que se puede cerrar con la X de la esquina superior derecha y que se cierra sola al terminar su barra de tiempo (ART-29 §1). Los textos exactos de las alertas se escriben en los criterios de la HU |

Toda HU que implique listar datos hereda:

| ID | Criterio |
|---|---|
| CA-T6 | Paginación (máx. 100 por página) |
| CA-T7 | Estado vacío con mensaje y acción sugerida |
| CA-T8 | Estado de carga (skeleton) |

---

## 4. Ejemplo completo

### HU-012 · Crear factura de venta

| Campo | Valor |
|---|---|
| Épica | EP-03 Facturación |
| Perfil(es) | Cajero, Administrador |
| Permiso(s) | `factura:crear` |
| Prioridad | Must |
| Estimación | 5 |
| Plataformas | Web |
| Dependencias | HU-008 (clientes), HU-010 (productos) |

**Como** cajero
**quiero** registrar una factura con sus productos
**para** dejar constancia de la venta y enviarla a la DIAN

**CA-1 · Creación exitosa**
```gherkin
Dado que soy un usuario con el perfil "Cajero"
Y existe el cliente "Ferretería El Tornillo" activo
Cuando creo la factura "FV-000123" con 2 unidades de "Martillo 16 oz" a $25.000
Entonces veo el mensaje "Factura creada correctamente."
Y la factura aparece en el listado con total $50.000 y estado EMITIDA
Y los demás usuarios de mi empresa que tienen abierto el listado la ven aparecer sin recargar
```

**CA-2 · Número duplicado**
```gherkin
Dado que ya existe la factura "FV-000123" en mi empresa
Cuando intento crear otra factura con el número "FV-000123"
Entonces veo "Ya existe una factura con ese número."
Y no se crea ninguna factura
```

**CA-S1 · Sin permiso** (perfil Consulta) — según plantilla.
**CA-S2 · Cliente de otra empresa** — según plantilla, respuesta "El cliente seleccionado no existe o está inactivo."
