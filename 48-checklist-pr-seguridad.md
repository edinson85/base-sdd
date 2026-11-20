---
id: ART-48
titulo: Plantilla de PR y checklist de revisión y seguridad
version: 1.1.0   # 1.1.0: interfaz/alertas e insumos en el checklist
estado: vigente
aplica_a: [todos]
depende_de: [ART-02, ART-06]
---

# ART-48 · Plantilla de PR y checklist de revisión

> **Instrucción para agentes IA**
> - Toda entrega de código (tuya o humana) usa la plantilla de §1 como descripción del PR.
> - Marca solo lo que verificaste. Lo que no aplica se marca `N/A` con razón. Un ítem sin marcar bloquea el merge.
> - Como revisor, usa §2 y reporta hallazgos con el formato de §3.

---

## 1. Plantilla de PR (`.github/pull_request_template.md` / `.bitbucket/...`)

````markdown
## Qué cambia
<Resumen en 2-4 líneas.>

## Trazabilidad
- HU: HU-xxx · Tarea: T-xxx
- Especificaciones: EP-..., PA-..., EV-..., WS-...
- Prototipo: docs/prototipo/<archivo>.html v<x.y> (si es de front)
- Insumos usados: INS-... (todos en estado verificado)
- ADR: (si aplica)

## Tipo
- [ ] feat  - [ ] fix  - [ ] refactor  - [ ] test  - [ ] docs  - [ ] chore  - [ ] security

## Checklist del autor

### Arquitectura
- [ ] Respeta la estructura `domain/application/infrastructure` del módulo
- [ ] `domain` sin dependencias de frameworks; linters de fronteras en verde
- [ ] Sin lógica de negocio en routers, componentes o widgets
- [ ] Efectos secundarios en observers, eventos publicados después del commit

### Datos y rendimiento
- [ ] Presupuesto de consultas declarado y probado (≤ N)
- [ ] Sin N+1; proyecciones en lecturas; paginación con máximo
- [ ] Migración reversible, compatible con la versión anterior, índices incluidos
- [ ] Caché: clave, TTL e invalidación (o N/A)

### Seguridad
- [ ] Permiso exigido en router y caso de uso
- [ ] Alcance (empresa/dueño) aplicado en todas las consultas por id
- [ ] Entradas validadas con tipos de ART-19; `extra="forbid"`
- [ ] Salida con esquema explícito, sin campos de más
- [ ] Sin secretos, tokens ni datos personales en código, logs o mensajes
- [ ] Front/móvil: rutas con permiso, nada sensible en almacenamiento, sin HTML sin sanitizar
- [ ] WS (si aplica): autenticación, origin, autorización de canal, validación de mensajes

### Interfaz (front/móvil)
- [ ] La pantalla coincide con el prototipo aprobado (estructura, textos, estados)
- [ ] Éxitos, advertencias y errores se informan solo con alertas flotantes vía `Notificador` (X arriba a la derecha, cierre automático con barra que se pausa) — sin `alert()`/`confirm()` ni mensajes improvisados
- [ ] Textos de las alertas iguales a los de la especificación (ART-47 §8.1)

### Errores y observabilidad
- [ ] Errores nuevos en `errores.yaml` con mensaje al usuario
- [ ] Logs estructurados con eventos estándar (ART-18)

### Pruebas
- [ ] Unitarias de dominio y aplicación
- [ ] Integración / API (contrato y formato de error)
- [ ] Adversarias de la especificación
- [ ] E2E actualizadas (si cambia un flujo)
- [ ] Cobertura ≥ 80 % en domain + application del módulo

### Documentación
- [ ] Especificación actualizada si hubo cambios aprobados
- [ ] OpenAPI con descripciones y errores

## Evidencia
<Capturas, salida de pruebas, plan EXPLAIN de consultas nuevas sobre tablas grandes.>

## Riesgos y plan de despliegue
<Migraciones, flags, orden de despliegue, rollback.>
````

---

## 2. Checklist del revisor (adicional)

| Área | Pregunta |
|---|---|
| Diseño | ¿El cambio está en el módulo y la capa correctos? ¿Se creó un acoplamiento nuevo entre módulos? |
| Dominio | ¿Las invariantes están protegidas en el dominio y no solo en el esquema? |
| Permisos | ¿Qué pasa si llamo a este endpoint con un perfil sin permiso? ¿Con el id de otra empresa? |
| Entradas | ¿Hay algún campo sin longitud máxima o sin patrón? ¿Algún parámetro interpolado (orden, columnas)? |
| Datos | ¿Cuántas consultas ejecuta en el peor caso? ¿Depende de N? |
| Concurrencia | ¿Qué pasa con dos requests simultáneos? ¿Doble clic? |
| Errores | ¿Algún `except`/`catch` que oculte errores? ¿Mensajes con detalles internos? |
| Eventos | ¿Los observers son idempotentes? ¿Qué pasa si fallan? ¿Debe ir por Outbox? |
| Caché | ¿Puede servir datos de otro tenant? ¿Se invalida en todos los cambios? |
| Front | ¿Se ven datos tras logout? ¿Hay datos sensibles en URL o consola? ¿Toda retroalimentación usa alertas flotantes? ¿Coincide con el prototipo? |
| Insumos | ¿El código depende de un insumo no verificado (API externa, cuenta, formato de datos)? ¿Hay valores supuestos que deberían venir de un insumo? |
| Pruebas | ¿Las pruebas fallarían si el código estuviera mal? ¿Cubren los casos negativos? |
| Operación | ¿La migración bloquea tablas grandes? ¿Hay variables de entorno nuevas documentadas? |

## 3. Formato de hallazgos de revisión

```
[BLOQUEANTE|IMPORTANTE|SUGERENCIA] <archivo>:<línea>
Qué: <problema>
Por qué: <principio P-XX-NN o riesgo concreto>
Propuesta: <cambio sugerido>
```

- **BLOQUEANTE:** viola un principio DEBE, introduce una vulnerabilidad o rompe funcionalidad.
- **IMPORTANTE:** viola un DEBERÍA sin justificación o deja deuda relevante.
- **SUGERENCIA:** mejora opcional.

## 4. Revisión por IA
Cuando un agente revisa un PR:
1. Lee la especificación referenciada y los principios aplicables.
2. Recorre §1 y §2 verificando en el código (no confiar en las casillas marcadas).
3. Ejecuta o lee los resultados de las pruebas.
4. Entrega los hallazgos con el formato de §3, ordenados por severidad, y una conclusión: aprobar / aprobar con cambios / rechazar.
