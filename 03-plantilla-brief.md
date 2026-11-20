---
id: ART-03
titulo: Plantilla de brief / idea del proyecto
version: 1.1.0   # 1.1.0: marca e insumos preliminares
estado: vigente
aplica_a: [todos]
depende_de: [ART-01]
---

# ART-03 · Plantilla de brief del proyecto

> **Instrucción para agentes IA**
> - Llena esta plantilla a partir de la idea del humano. Lo que no puedas deducir con certeza márcalo `[SUPUESTO]` o `[PREGUNTA]`.
> - Las secciones marcadas con ★ son **bloqueantes**: sin ellas no se puede pasar a F1.
> - Guarda el resultado como `docs/00-brief.md`.

---

````markdown
---
proyecto: <Nombre>
codigo: <SIGLA>             # Se usa en IDs y prefijos de caché: app → <sigla>
version: 0.1.0
estado: borrador | aprobado
fecha: AAAA-MM-DD
responsable: <Nombre>
---

# Brief · <Nombre del proyecto>

## 1. Problema ★
<Qué problema existe hoy, a quién afecta y cómo se resuelve actualmente.>

## 2. Objetivo y resultados esperados ★
| # | Resultado medible | Indicador | Meta |
|---|---|---|---|
| O-1 | | | |

## 3. Usuarios y perfiles ★
| Perfil | Descripción | Qué necesita hacer | Cantidad estimada |
|---|---|---|---|

## 4. Alcance
### 4.1 Incluido (MVP) ★
- <Funcionalidad 1>
### 4.2 Fuera de alcance
- <Lo que explícitamente NO se hará>
### 4.3 Futuras versiones
-

## 5. Plataformas ★
| Plataforma | ¿Aplica? | Notas |
|---|---|---|
| Web (React) | Sí/No | Navegadores objetivo |
| Móvil (Flutter) | Sí/No | Android / iOS, versión mínima |
| Tiempo real (WebSocket) | Sí/No | ¿Qué debe verse en vivo? |
| Procesos en segundo plano | Sí/No | Reportes, envíos, integraciones |

## 6. Multi-tenant / alcance de datos ★
<¿Una sola organización o varias empresas que no deben ver datos entre sí? ¿Sedes? ¿Datos por usuario?>

## 7. Integraciones externas
| Sistema | Propósito | Dirección (entrada/salida) | Protocolo | Criticidad |
|---|---|---|---|---|

## 8. Datos sensibles y cumplimiento ★
| Dato | Tipo (personal, financiero, salud...) | Quién puede verlo | Retención |
|---|---|---|---|

Normativa aplicable: <Ley 1581 de 2012 (habeas data, Colombia), facturación electrónica DIAN, PCI DSS si hay tarjetas, otras>.

## 9. Requisitos no funcionales
| Aspecto | Requisito |
|---|---|
| Usuarios concurrentes | |
| Tiempo de respuesta objetivo (p95) | Por defecto 300 ms lecturas / 800 ms escrituras |
| Disponibilidad | Por defecto 99.5 % |
| Volumen de datos (año 1) | |
| RPO / RTO | Por defecto 24 h / 4 h |
| Idiomas | Español |
| Zona horaria | America/Bogota (almacenar en UTC) |
| Accesibilidad | WCAG 2.2 AA |

## 10. Infraestructura y despliegue
<Servidor propio, VPS, nube; presupuesto; dominio; ¿quién opera?>

## 11. Restricciones
<Fechas, presupuesto, equipo, tecnologías impuestas o prohibidas.>

## 12. Riesgos conocidos
| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|

## 13. Glosario inicial
| Término del negocio | Significado |
|---|---|

## 14. Marca e identidad visual
<¿Existe logo (idealmente SVG), colores y tipografía corporativos? ¿Manual de marca? ¿Referencias de aplicaciones que les gustan o no? Si no existe, se propone en la fase de interfaz (ART-49).>

## 15. Insumos y accesos conocidos (lista preliminar, ART-50)
| Insumo | Para qué | ¿Quién lo tiene / entrega? | ¿Ya existe? | Obtención lenta |
|---|---|---|---|---|
| Cuenta de correo saliente (ej. Gmail/Workspace dedicada) | Notificaciones | | Sí/No | No |
| Servidor / nube | Despliegue | | | |
| Facturación electrónica DIAN | | | | Sí |

## 16. Supuestos
- [SUPUESTO-1] ...

## 17. Preguntas abiertas
- [PREGUNTA-1] ... (opciones: a / b / c)
````

---

## Lista de verificación del brief (para la IA antes de pedir aprobación)

- [ ] Todas las secciones ★ están completas o tienen preguntas explícitas.
- [ ] Cada perfil del §3 tiene al menos una necesidad que se traduce en funcionalidad del §4.1.
- [ ] Se sabe si hay multi-tenant (§6): afecta todo el modelo de datos y la seguridad.
- [ ] Los datos sensibles (§8) están identificados: afectan logging, caché, cifrado y pruebas.
- [ ] Se sabe si hay web, móvil o ambos (§5): afecta el flujo de autenticación (ART-20 §6).
- [ ] Los supuestos y preguntas están numerados.
- [ ] Cada integración, plataforma e infraestructura mencionada tiene su insumo en §15, y se generó `06-insumos.md` preliminar (ART-50) con los de obtención lenta ya solicitados.
- [ ] Ningún supuesto se refiere a cómo será una cuenta, acceso o sistema externo: esos casos son insumos, no supuestos.
