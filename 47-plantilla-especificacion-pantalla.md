---
id: ART-47
titulo: Plantilla de especificación de pantalla
version: 1.1.0   # 1.1.0: vínculo con el prototipo (ART-49) y tabla de alertas flotantes
estado: vigente
aplica_a: [frontend-web, movil]
depende_de: [ART-11, ART-12, ART-27, ART-29, ART-30, ART-31, ART-32]
---

# ART-47 · Plantilla de especificación de pantalla

> **Instrucción para agentes IA**
> - Una especificación por pantalla en `docs/specs/pantallas/<modulo>/PA-<MODULO>-<NNN>.md` (fase F4).
> - La especificación **formaliza la pantalla aprobada en el prototipo** (ART-49): mismo ID, misma estructura y comportamiento. Si al especificar aparece una diferencia, se resuelve con el humano y se actualiza el prototipo.
> - Llena todas las secciones (`N/A` + razón si no aplica). Los textos visibles van exactos (salen de las HU).
> - Si la pantalla existe en web y móvil con diferencias, usa la columna "Móvil" o una sección §12 de diferencias.

---

````markdown
---
id: PA-<MODULO>-<NNN>
modulo: <modulo>
ruta_web: /<ruta>
ruta_movil: /<ruta>        # o N/A
historias: [HU-xxx]
prototipo: docs/prototipo/<archivo>.html v<x.y>   # Pantalla aprobada en ART-49
estado: borrador | aprobado | implementado
---

# PA-<MODULO>-<NNN> · <Nombre de la pantalla>

## 1. Propósito
<Qué logra el usuario aquí.>

## 2. Acceso
| Aspecto | Valor |
|---|---|
| Permiso para ver | `<recurso>:ver` |
| Perfiles | |
| Si no tiene permiso | Redirección a `/403` |
| Parámetros de ruta | `:id` (UUID; si no es válido → 404) |
| Pública | No |

## 3. Estructura (wireframe)
```
┌───────────────────────────────────────────────┐
│ Breadcrumbs: Inicio / Facturas                │
│ Facturas                       [Nueva factura]│  ← Can factura:crear
├───────────────────────────────────────────────┤
│ [Buscar...] [Estado ▾] [Desde] [Hasta] [Limpiar]│
├───────────────────────────────────────────────┤
│ Número │ Cliente │ Fecha │ Total │ Estado │ ⋯ │
│ ...                                           │
├───────────────────────────────────────────────┤
│                       < 1 2 3 >  20 por página│
└───────────────────────────────────────────────┘
```

## 4. Datos
| Dato | Fuente (endpoint) | Query key | staleTime | Actualización en tiempo real |
|---|---|---|---|---|
| Lista de facturas | EP-FACTURA-002 | `['facturas','lista',filtro,page]` | 30 s | `factura.creada`, `factura.anulada` → invalidar |

## 5. Componentes
| Componente (ART-31) | Contenido | Notas |
|---|---|---|
| DataTable | Columnas: ... | Orden por número y fecha; móvil: tarjetas |
| Badge | Estado | EMITIDA = info, ANULADA = muted |

## 6. Campos de formulario (si aplica)
| Campo | Etiqueta | Control | Tipo de validación (ART-30) | Obligatorio | Ayuda | Campo API |
|---|---|---|---|---|---|---|
| numero | Número | Input | patrón `^[A-Z0-9\-]+$`, 3-20 | Sí | "Ej.: FV-000123" | numero |

## 7. Acciones
| Acción | Ubicación | Permiso | Confirmación | Endpoint | Resultado éxito | Errores esperados |
|---|---|---|---|---|---|---|
| Nueva factura | Encabezado | factura:crear | No | — (navega) | `/facturas/nueva` | — |
| Anular | Menú de fila | factura:anular | Diálogo con motivo | EP-FACTURA-004 | Alerta de éxito "Factura anulada." + refrescar | FACTURA_YA_ANULADA (alerta de error) |

## 8. Estados (ART-29 §4)
| Estado | Presentación / texto |
|---|---|
| Cargando | Skeleton de 10 filas |
| Vacío | "Aún no hay facturas." + [Crear factura] si tiene permiso |
| Vacío por filtros | "No hay facturas con estos filtros." + [Limpiar filtros] |
| Error | Banner "No pudimos cargar las facturas." + [Reintentar] |

## 8.1 Alertas flotantes de la pantalla (ART-29 §1)
| Disparador | Tipo | Texto exacto | Duración / persistente | Acción en la alerta |
|---|---|---|---|---|
| Factura anulada | Éxito | "Factura anulada." | 5 s | — |
| FACTURA_YA_ANULADA | Error | "La factura ya fue anulada y no puede modificarse." | 10 s | — |
| Llegan facturas nuevas por WS | Información | "Se agregaron N facturas." | 6 s | "Ver" |

## 9. Seguridad (ART-27)
- Datos sensibles mostrados: <ninguno / cuáles y cómo se enmascaran>.
- Filtros en la URL: solo estado y fechas (sin datos personales).
- Limpieza al salir: <si aplica>.

## 10. Accesibilidad (ART-32)
- `<h1>`: "Facturas". Tabla con `caption` y `aria-sort`.
- Anuncio `aria-live` cuando llegan facturas nuevas por WS: "Se agregaron N facturas."
- Atajos o comportamiento de teclado particulares: <...>

## 11. Pruebas
- Componentes: render por permiso, estados, acciones, mapeo de errores.
- E2E: <criterios de HU que cubre esta pantalla>.
- Adversarias: navegación directa sin permiso, id ajeno en la URL, XSS en datos mostrados.

## 12. Diferencias web / móvil
| Aspecto | Web | Móvil |
|---|---|---|
````
