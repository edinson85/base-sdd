---
id: ART-06
titulo: Definition of Ready / Definition of Done
version: 1.1.0   # 1.1.0: prototipo aprobado e insumos verificados en DoR; alertas en DoD
estado: vigente
aplica_a: [todos]
depende_de: [ART-02, ART-05]
---

# ART-06 · Definition of Ready / Definition of Done

> **Instrucción para agentes IA**
> No empieces una tarea que no cumpla la DoR. No declares terminada una tarea que no cumpla la DoD: reporta qué ítems faltan.

---

## 1. Definition of Ready (DoR)

### 1.1 Historia de usuario
- [ ] Tiene el formato ART-05 completo (perfil, permiso, prioridad, estimación).
- [ ] Tiene criterios positivos, negativos y de seguridad (CA-S1, CA-S2 cuando aplica).
- [ ] Los mensajes al usuario están escritos textualmente.
- [ ] Las dependencias están terminadas o planeadas antes.
- [ ] No tiene `[PREGUNTA]` abiertas que bloqueen.
- [ ] Cabe en un sprint.
- [ ] Sus pantallas están en el prototipo aprobado (ART-49) y los hallazgos del análisis funcional que la afectan están resueltos.
- [ ] Sus insumos (ART-50) están **verificados** o tienen fecha confirmada antes del inicio del sprint.

### 1.2 Tarea
- [ ] Referencia HU y especificación (EP/PA/EV/WS) **aprobadas**.
- [ ] Lista los archivos esperados y la capa (domain/application/infrastructure).
- [ ] Define las pruebas requeridas.
- [ ] Estimación ≤ 1 día.
- [ ] Dependencias técnicas resueltas (migraciones, permisos semilla, contratos).
- [ ] Todos sus insumos están en estado **verificado** en `06-insumos.md` (P-SDD-07). Sin esto la tarea no inicia.
- [ ] Si es de front: referencia la pantalla del prototipo aprobado.

---

## 2. Definition of Done (DoD)

### 2.1 Tarea / PR
**Código**
- [ ] Cumple la especificación referenciada y sus criterios de aceptación.
- [ ] Respeta la estructura hexagonal por módulo; linters de fronteras en verde (`import-linter`, ESLint boundaries, reglas Dart).
- [ ] Linters y tipado en verde: ruff, mypy, ESLint, `tsc --noEmit`, `dart analyze`.
- [ ] Sin código muerto, `TODO` sin ticket, ni `print`/`console.log`.

**Seguridad**
- [ ] Entradas validadas (ART-19), permisos y alcance verificados (ART-20).
- [ ] Sin secretos ni datos sensibles en código, logs o mensajes.
- [ ] Escaneo de dependencias y SAST sin hallazgos altos o críticos.

**Datos y rendimiento**
- [ ] Presupuesto de consultas cumplido y probado.
- [ ] Migraciones reversibles e índices declarados.
- [ ] Caché con clave, TTL e invalidación (si aplica).

**Errores y observabilidad**
- [ ] Errores del catálogo (`errores.yaml`) con mensajes al usuario; formato estándar.
- [ ] Front/móvil: éxitos, advertencias y errores informados solo con alertas flotantes (ART-29 §1); la pantalla coincide con el prototipo aprobado.
- [ ] Logs estructurados con `trace_id`.

**Pruebas**
- [ ] Unitarias, integración y adversarias según la especificación, en verde.
- [ ] Cobertura ≥ 80 % en domain y application del módulo.
- [ ] Checklist ART-48 marcado en el PR.

**Documentación**
- [ ] OpenAPI actualizado; especificación actualizada si hubo cambios aprobados.
- [ ] ADR si se tomó una decisión de arquitectura.

### 2.2 Historia de usuario
- [ ] Todas sus tareas cumplen DoD.
- [ ] Prueba E2E de cada criterio de aceptación en verde (por perfil).
- [ ] Funcionalidades que dependen de insumos probadas con el insumo real (no con alternativas temporales, ART-50 §5).
- [ ] Revisada en ambiente de staging por el responsable de negocio.

### 2.3 Sprint
- [ ] HU comprometidas cumplen DoD o se movieron con justificación.
- [ ] Suite E2E y adversaria completa en verde en staging.
- [ ] Sin vulnerabilidades altas o críticas abiertas.
- [ ] Documento del sprint cerrado (ART-04 §5).

### 2.4 Release a producción
- [ ] DoD de sprint cumplida.
- [ ] Prueba de carga del release (ART-43) dentro de SLO.
- [ ] Escaneo DAST (OWASP ZAP) sin hallazgos altos.
- [ ] Backup verificado antes del despliegue y plan de rollback (ART-37).
- [ ] Migraciones probadas en staging con datos de volumen similar.
- [ ] Notas de versión.
