---
id: ART-08
titulo: Glosario técnico base y plantilla de lenguaje ubicuo
version: 1.0.0
estado: vigente
aplica_a: [todos]
depende_de: [ART-02]
---

# ART-08 · Glosario y lenguaje ubicuo

> **Instrucción para agentes IA**
> - Usa los términos de §1 con este significado exacto en toda la documentación y el código.
> - Para cada proyecto crea `docs/glosario.md` con la plantilla de §2. Los nombres de módulos, entidades, eventos y permisos salen de ese glosario.
> - Si el negocio usa dos palabras para lo mismo, elige una, regístrala y lista la otra como sinónimo.

---

## 1. Glosario técnico base

| Término | Definición en esta base |
|---|---|
| **Módulo** | Carpeta que agrupa todo lo de una entidad o agregado del negocio, con las capas `domain`, `application`, `infrastructure` (P-AR-02) |
| **Agregado** | Grupo de entidades que se modifican juntas y mantienen invariantes; se accede a él por su raíz (ej. `Factura` con sus `ItemFactura`) |
| **Entidad** | Objeto con identidad propia que persiste en el tiempo |
| **Value object** | Objeto sin identidad definido por su valor, inmutable y autovalidado (ej. `NumeroFactura`, `Email`) |
| **Invariante** | Regla que siempre debe cumplirse en un agregado (ej. total = suma de ítems) |
| **Caso de uso** | Clase o función de `application` que orquesta una acción del usuario: permisos, transacción, dominio, eventos |
| **Puerto** | Interfaz que define lo que el núcleo necesita del exterior (repositorio, caché, notificador) o lo que ofrece |
| **Adaptador** | Implementación concreta de un puerto (SQLAlchemy, Redis, router FastAPI, componente React) |
| **Adaptador de entrada** | Lo que invoca casos de uso: router HTTP, WebSocket, worker, CLI, componente UI |
| **Adaptador de salida** | Lo que los casos de uso usan: repositorio, query service, caché, cliente externo |
| **Composition root** | Único lugar donde se crean los adaptadores y se inyectan en los casos de uso |
| **Repositorio** | Puerto/adaptador para guardar y recuperar agregados completos (escrituras) |
| **Query service** | Puerto/adaptador para lecturas optimizadas que devuelven read models (P-DA-03) |
| **Read model** | DTO de lectura con la forma exacta que necesita la respuesta |
| **Unit of Work (UoW)** | Delimita la transacción de un caso de uso |
| **Evento de dominio** | Hecho de negocio ocurrido, nombrado en pasado (`FacturaCreada`) |
| **Observer** | Manejador suscrito a un evento que produce un efecto secundario |
| **Outbox** | Tabla donde se guardan eventos en la misma transacción para publicarlos de forma confiable después |
| **Relay** | Proceso que lee el Outbox y publica/ejecuta los eventos pendientes |
| **Idempotencia** | Propiedad de una operación que, ejecutada varias veces con la misma entrada, produce el mismo efecto que una vez |
| **Canal (WS)** | Destino lógico de mensajes en tiempo real (`empresa:{id}:facturas`) |
| **Ticket WS** | Credencial de un solo uso y vida corta para autenticar el handshake WebSocket |
| **Actor** | Usuario autenticado con su alcance y permisos, que recibe cada caso de uso |
| **Perfil** | Rol del usuario; agrupa permisos |
| **Permiso** | Capacidad atómica `recurso:accion` |
| **Alcance (scope)** | Límite de datos que un actor puede ver: empresa/tenant, sede, propietario |
| **Tenant** | Organización cuyos datos están aislados de las demás |
| **IDOR** | Acceso indebido a un recurso cambiando su identificador |
| **Presupuesto de consultas** | Número máximo de consultas SQL que un endpoint puede ejecutar (P-DA-01) |
| **Cache-aside** | Leer de caché; si no está, leer de BD y guardar en caché |
| **trace_id** | Identificador único de un request que une logs, errores y eventos |
| **Código de error** | Identificador estable en MAYÚSCULAS de un error (`FACTURA_DUPLICADA`) |
| **Prueba adversaria** | Prueba que simula un ataque o un uso malicioso |
| **Puerta (gate)** | Punto de aprobación humana entre fases SDD |

---

## 2. Plantilla de glosario del proyecto (`docs/glosario.md`)

````markdown
# Glosario · <Proyecto>

| Término | Definición | Sinónimos (no usar) | Módulo | Identificador en código | Notas |
|---|---|---|---|---|---|
| Factura | Documento que soporta una venta | Cuenta de cobro | factura | `Factura`, tabla `facturas` | |
| Anular | Dejar sin efecto una factura emitida | Cancelar, eliminar | factura | `anular()`, evento `FacturaAnulada`, permiso `factura:anular` | No se borra físicamente |

## Estados
| Entidad | Estado | Significado | Transiciones permitidas |
|---|---|---|---|
| Factura | EMITIDA | | → ANULADA |

## Perfiles
| Perfil | Descripción |
|---|---|
````

---

## 3. Reglas de nombres derivadas del glosario

| Elemento | Regla | Ejemplo |
|---|---|---|
| Módulo / carpeta | Sustantivo singular, snake_case | `factura`, `orden_compra` |
| Entidad | PascalCase singular | `OrdenCompra` |
| Tabla | snake_case plural | `ordenes_compra` |
| Evento | PascalCase en pasado | `OrdenCompraAprobada` |
| Permiso | `<modulo>:<verbo>` | `orden_compra:aprobar` |
| Código de error | MAYÚSCULAS con guion bajo | `ORDEN_COMPRA_YA_APROBADA` |
| Ruta API | kebab-case plural | `/api/v1/ordenes-compra` |

Los identificadores no usan tildes ni ñ (`anio`, `numero`); la interfaz y los mensajes sí. Detalle en ART-13.
