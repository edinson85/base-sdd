---
id: ART-38
titulo: Estrategia general de pruebas
version: 1.0.0
estado: vigente
aplica_a: [todos]
depende_de: [ART-02, ART-06]
---

# ART-38 · Estrategia general de pruebas

> **Instrucción para agentes IA**
> Toda tarea de implementación entrega sus pruebas (ART-04 §5). Usa esta estrategia para decidir **qué tipo** de prueba escribir; los detalles por tipo están en ART-39 a ART-43. Principios P-PR-01..07.

---

## 1. Pirámide

```
            ▲  Carga (ART-43)              — por release
           ▲▲  E2E (ART-41)                — flujos críticos por perfil
          ▲▲▲  Adversarias (ART-42)        — por endpoint, pantalla y canal
        ▲▲▲▲▲  Integración / API           — adaptadores, contratos, presupuesto de consultas
   ▲▲▲▲▲▲▲▲▲▲  Unitarias (ART-39, ART-40)  — dominio y casos de uso
```

| Tipo | Alcance | Velocidad | Cuándo corre | Responsable |
|---|---|---|---|---|
| Unitarias | Una clase/función; sin E/S | ms | Cada push | Quien implementa |
| Integración | Un adaptador contra la tecnología real (PostgreSQL, Redis) | s | Cada push | Quien implementa |
| API / contrato | Endpoint completo vía HTTP con BD real | s | Cada push | Quien implementa |
| Componentes (front) | Componente/pantalla con API simulada (MSW) | ms-s | Cada push | Quien implementa |
| Adversarias | Ataques a endpoints, pantallas, WS, Nginx | s-min | Cada push (API) / main (E2E) | Quien implementa + QA |
| E2E | Sistema completo con Docker Compose | min | main / nocturno | QA |
| Carga | Sistema en staging | min-h | Antes de release | QA / DevOps |

## 2. Qué probar en cada capa

| Capa | Se prueba con | Qué |
|---|---|---|
| `domain` | Unitarias | Invariantes, transiciones de estado, cálculos, eventos registrados, value objects |
| `application` | Unitarias con fakes | Permisos, alcance, orquestación, errores, eventos publicados después del commit, observers idempotentes |
| `infrastructure` back | Integración / API | Mapeos ORM, consultas (resultado y presupuesto), caché, traducción de errores, routers (status, formato), WS |
| `infrastructure` front/móvil | Componentes / widgets | Renderizado por permiso, formularios, estados, mapeo de DTOs, interceptores |
| Sistema | E2E | Criterios de aceptación de las HU |

## 3. Trazabilidad
- El nombre o la descripción de cada prueba E2E referencia la HU y el criterio: `HU-012 CA-2: número duplicado`.
- Las pruebas adversarias referencian el ID de la especificación: `EP-FACTURA-001 adversarial: IDOR cliente`.
- La matriz de trazabilidad (ART-01 §7) lista las pruebas E2E por HU.

## 4. Datos de prueba
- **Factories** (factory_boy / Faker `es_CO` en Python; fábricas propias en TS/Dart) en lugar de fixtures estáticos grandes.
- Cada prueba crea lo que necesita y no depende del orden de ejecución.
- Escenarios multi-tenant siempre con **dos empresas** para probar aislamiento.
- Usuarios de prueba por perfil (`admin`, `cajero`, `consulta`, `sin_permisos`) creados por un helper.
- Nunca datos reales de producción.

## 5. Umbrales (puertas de CI)
| Métrica | Umbral |
|---|---|
| Cobertura de líneas en `domain` + `application` (back, front, móvil) | ≥ 80 % |
| Cobertura de ramas en `domain` | ≥ 70 % |
| Pruebas fallidas | 0 |
| Presupuesto de consultas | 100 % de endpoints con prueba |
| Pruebas adversarias por endpoint | Las de la plantilla ART-46 §12.3 |
| Violaciones de accesibilidad serias/críticas en E2E | 0 |

La cobertura es un mínimo, no un objetivo: una prueba sin aserciones relevantes no cuenta.

## 6. Pruebas inestables (flaky)
- Una prueba que falla de forma intermitente se marca, se abre un ticket y se corrige en el sprint; no se reintenta indefinidamente.
- Prohibido `sleep` fijo: esperar condiciones (`await expect(...).toBeVisible()`, polling con timeout).
- Tiempo controlado con un `Clock` inyectado (back) y `vi.useFakeTimers` / `fakeAsync` (front/móvil).

## 7. Nombres y estructura
- Python: `test_<accion>_<condicion>_<resultado>` → `test_crear_factura_numero_duplicado_lanza_conflicto`.
- TS: `describe('<unidad>')` + `it('<comportamiento esperado>')` en español.
- Estructura Arrange / Act / Assert visible (líneas en blanco o comentarios).
- Una razón de fallo por prueba.
