---
id: ART-00
titulo: Índice de la base de artefactos SDD
version: 1.1.0
---

# Base de artefactos para proyectos SDD con IA

Stack estándar: **PostgreSQL · Python (FastAPI) · React · Flutter · Docker · Nginx**, con arquitectura hexagonal organizada por módulo (`domain/`, `application/`, `infrastructure/`), eventos y observers, WebSockets cuando aplica, y seguridad por defecto.

> **Instrucción para agentes IA**
> 1. Lee primero **ART-02 (Constitución)**: tiene prioridad sobre todo lo demás.
> 2. Lee **ART-01 (Guía de uso)**: define las fases, entregables, puertas de aprobación y qué artefactos leer en cada fase.
> 3. No cargues todos los artefactos a la vez: lee los que indica ART-01 §5 para la fase actual.
> 4. Tipos de artefacto: **N** = normativo (se cumple), **P** = plantilla (se llena por proyecto), **C** = catálogo (se consulta y se extiende).

## Índice

### Proceso SDD
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-01 | Guía de uso de los artefactos | N | `01-guia-de-uso.md` |
| ART-02 | Constitución del proyecto | N | `02-constitucion.md` |
| ART-03 | Plantilla de brief / idea | P | `03-plantilla-brief.md` |
| ART-04 | Flujo SDD y plantillas (especificación, plan técnico, tareas) | N+P | `04-flujo-sdd.md` |
| ART-05 | Historias de usuario y criterios de aceptación | P | `05-plantilla-historias-usuario.md` |
| ART-06 | Definition of Ready / Done | N | `06-definition-of-ready-done.md` |
| ART-07 | Plantilla ADR y decisiones base del stack | P+C | `07-plantilla-adr.md` |
| ART-08 | Glosario técnico y lenguaje ubicuo | C+P | `08-glosario.md` |

### Arquitectura
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-09 | Arquitectura general del sistema | N | `09-arquitectura-general.md` |
| ART-10 | Arquitectura hexagonal – Backend Python | N | `10-arquitectura-hexagonal-backend.md` |
| ART-11 | Arquitectura hexagonal – Frontend React | N | `11-arquitectura-hexagonal-frontend-react.md` |
| ART-12 | Arquitectura hexagonal – Móvil Flutter | N | `12-arquitectura-hexagonal-flutter.md` |
| ART-13 | Convenciones de código y Git | N | `13-convenciones-codigo-git.md` |

### Backend
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-14 | Estándar de diseño de API (REST + WS) | N | `14-estandar-api.md` |
| ART-15 | Optimización de acceso a datos | N+C | `15-optimizacion-acceso-datos.md` |
| ART-16 | Estrategia de caché | N | `16-estrategia-cache.md` |
| ART-17 | Excepciones y catálogo de errores | N+C | `17-catalogo-errores.md` |
| ART-18 | Logging y observabilidad | N | `18-logging-observabilidad.md` |
| ART-19 | Validación de entradas | N+C | `19-validacion-entradas.md` |
| ART-20 | Autenticación y autorización (RBAC) | N | `20-autenticacion-autorizacion-rbac.md` |
| ART-21 | Seguridad backend | N | `21-seguridad-backend.md` |

### Base de datos
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-22 | Estándar de modelado PostgreSQL | N | `22-estandar-modelado-postgresql.md` |
| ART-23 | Modelo y diccionario de datos | P | `23-plantilla-modelo-datos.md` |
| ART-24 | Migraciones y semillas | N | `24-migraciones-semillas.md` |
| ART-25 | Índices y rendimiento | N | `25-indices-rendimiento.md` |
| ART-26 | Seguridad de la base de datos | N | `26-seguridad-base-datos.md` |

### Frontend y móvil
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-27 | Seguridad frontend web | N | `27-seguridad-frontend.md` |
| ART-28 | Seguridad móvil (Flutter) | N | `28-seguridad-movil.md` |
| ART-29 | Errores, notificaciones y estados en la UI | N | `29-errores-notificaciones-ui.md` |
| ART-30 | Validación de formularios | N+C | `30-validacion-formularios.md` |
| ART-31 | Sistema de diseño / UI kit | N | `31-sistema-diseno.md` |
| ART-32 | Accesibilidad | N | `32-accesibilidad.md` |

### Infraestructura
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-33 | Estándar Docker | N | `33-estandar-docker.md` |
| ART-34 | Nginx seguro | N | `34-nginx-seguro.md` |
| ART-35 | Variables de entorno y secretos | N | `35-variables-entorno-secretos.md` |
| ART-36 | Pipeline CI/CD | N | `36-pipeline-ci-cd.md` |
| ART-37 | Ambientes, despliegue y backups | N | `37-ambientes-backups.md` |

### Pruebas
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-38 | Estrategia general de pruebas | N | `38-estrategia-pruebas.md` |
| ART-39 | Pruebas unitarias e integración backend | N | `39-pruebas-unitarias-backend.md` |
| ART-40 | Pruebas unitarias y de componentes front y móvil | N | `40-pruebas-unitarias-frontend-movil.md` |
| ART-41 | Pruebas end to end | N | `41-pruebas-end-to-end.md` |
| ART-42 | Pruebas adversarias | N+C | `42-pruebas-adversarias.md` |
| ART-43 | Pruebas de carga | N | `43-pruebas-carga.md` |

### Plantillas por proyecto
| ID | Artefacto | Tipo | Archivo |
|---|---|---|---|
| ART-44 | Plan de proyecto / roadmap por sprints (incluye Sprint 0) | P | `44-plantilla-roadmap-sprints.md` |
| ART-45 | Especificación de módulo | P | `45-plantilla-especificacion-modulo.md` |
| ART-46 | Especificación de endpoint (+ evento + canal WS) | P | `46-plantilla-especificacion-endpoint.md` |
| ART-47 | Especificación de pantalla | P | `47-plantilla-especificacion-pantalla.md` |
| ART-48 | Plantilla de PR y checklist de revisión | P+N | `48-checklist-pr-seguridad.md` |
| ART-49 | Propuesta de interfaz gráfica y análisis funcional (fase F2: prototipos, dirección visual, hallazgos) | N+P | `49-propuesta-interfaz-analisis-funcional.md` |
| ART-50 | Insumos y accesos previos al desarrollo (cuentas, credenciales, datos, marca, decisiones) | N+P+C | `50-insumos-accesos-previos.md` |

## Mapa rápido: requisito → artefactos

| Requisito | Artefactos |
|---|---|
| Mínimo de consultas a la BD | ART-02 §3, ART-15, ART-25, ART-46 §6 |
| Caché | ART-02 §4, ART-16 |
| Excepciones con mensajes al usuario | ART-02 §5, ART-17, ART-29 |
| Logs de errores | ART-02 §6, ART-18 |
| Validación de entradas (caracteres, longitud, peso, permitido) | ART-02 §7, ART-19, ART-30 |
| Perfiles y acciones | ART-02 §8, ART-20 |
| Seguridad ante ataques | ART-21, ART-26, ART-27, ART-28, ART-34, ART-42 |
| Hexagonal por objeto del modelo (3 capas) | ART-02 §2, ART-10, ART-11, ART-12 |
| Eventos, observers y WebSockets | ART-02 §10, ART-10 §4.5-§6.9, ART-14 §9, ART-46 anexos |
| Pruebas unitarias | ART-39, ART-40 |
| Pruebas E2E | ART-41 |
| Pruebas adversarias | ART-42 |
| Nginx seguro | ART-34 |
| Docker | ART-33 |
| Alertas flotantes (X arriba a la derecha, cierre automático, barra de tiempo) | ART-02 §11, ART-29 §1, ART-31, ART-32, ART-40 §2.5, ART-41 E2E-T10 |
| Propuesta de interfaz y análisis funcional antes del plan técnico | ART-02 P-UX-06 y P-SDD-06, ART-01 §5 (F2), ART-49 |
| Solicitar insumos (cuentas, accesos, datos) antes de desarrollar | ART-02 P-SDD-07/08, ART-01 §5 (F5), ART-50, ART-06 DoR |

## Versiones
| Versión | Fecha | Cambio |
|---|---|---|
| 0.1.0 | 2026-11-20 | ART-02, 10, 11, 46 |
| 1.0.0 | 2026-11-20 | Base completa (ART-01 a ART-48) |
| 1.1.0 | 2026-11-20 | Alertas flotantes (ART-29 §1 y relacionados); fase F2 de propuesta de interfaz (ART-49); insumos previos al desarrollo (ART-50); fases renumeradas F0-F7; Constitución 1.1.0 |
