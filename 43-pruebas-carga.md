---
id: ART-43
titulo: Pruebas de carga y rendimiento
version: 1.0.0
estado: vigente
aplica_a: [backend, infraestructura]
depende_de: [ART-03, ART-38]
---

# ART-43 · Pruebas de carga y rendimiento (k6)

> **Instrucción para agentes IA**
> Los objetivos salen del brief (§9 requisitos no funcionales). Si el brief no los define, usa los SLO por defecto de §1. Las pruebas corren en staging con datos de volumen representativo, nunca en producción sin ventana acordada.

---

## 1. SLO por defecto
| Métrica | Objetivo |
|---|---|
| Latencia p95 lecturas | ≤ 300 ms |
| Latencia p95 escrituras | ≤ 800 ms |
| Latencia p99 | ≤ 1.5 s |
| Tasa de error (5xx) | < 0.5 % |
| Entrega de evento WS (desde el commit) | p95 ≤ 1 s |
| Uso de CPU de la API en carga nominal | < 70 % |

## 2. Escenarios

| Escenario | Perfil de carga | Objetivo |
|---|---|---|
| Smoke | 1-2 usuarios, 1 min | El script funciona |
| Carga nominal | Usuarios concurrentes del brief, 15 min | Cumplir SLO |
| Estrés | Subir hasta 2-3× lo nominal | Encontrar el punto de quiebre y cómo falla (debe degradar con 429/503, no con errores de datos) |
| Pico (spike) | Salto de nominal a 3× en 30 s | Recuperación sin intervención |
| Resistencia (soak) | Nominal durante 2-4 h | Fugas de memoria, conexiones, crecimiento de colas |
| WebSocket | N conexiones simultáneas + eventos | Conexiones estables, latencia de entrega |

## 3. Datos
- Staging con volumen equivalente a 1 año (brief §9) generado con factories (`scripts/seed_volumen.py`).
- Usuarios de prueba por perfil y por empresa (varios tenants).
- Rate limits de Nginx/app ajustados en staging o IPs de los generadores en lista permitida, para medir la aplicación y no el limitador (y una corrida aparte para validar el limitador).

## 4. Script base

```js
// tests/load/flujo-cajero.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  scenarios: {
    nominal: { executor: 'ramping-vus', startVUs: 0,
      stages: [{ duration: '2m', target: 100 }, { duration: '10m', target: 100 }, { duration: '2m', target: 0 }] },
  },
  thresholds: {
    'http_req_failed': ['rate<0.005'],
    'http_req_duration{tipo:lectura}': ['p(95)<300'],
    'http_req_duration{tipo:escritura}': ['p(95)<800'],
  },
};

const BASE = __ENV.BASE_URL;

export function setup() {
  // Un token por VU se obtiene en default(); aquí se preparan datos compartidos si hace falta
}

export default function () {
  const token = login(__VU);
  const h = { headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' } };

  group('listar facturas', () => {
    const r = http.get(`${BASE}/api/v1/facturas?page=1&size=20`, { ...h, tags: { tipo: 'lectura' } });
    check(r, { '200': (x) => x.status === 200 });
  });

  group('crear factura', () => {
    const r = http.post(`${BASE}/api/v1/facturas`, JSON.stringify(nuevaFactura()),
      { headers: { ...h.headers, 'Idempotency-Key': crypto.randomUUID() }, tags: { tipo: 'escritura' } });
    check(r, { '201': (x) => x.status === 201 });
  });

  sleep(Math.random() * 3 + 1);   // Tiempo de pensamiento
}
```
(`login` y `nuevaFactura` en un módulo compartido; el login se cachea por VU para no medir solo la autenticación.)

## 5. Qué observar durante la prueba
- Latencias por endpoint (tags) y errores.
- API: CPU, memoria, event loop lag.
- PostgreSQL: conexiones, consultas lentas (`pg_stat_statements`), bloqueos, CPU, I/O.
- Redis: memoria, latencia, hit ratio.
- Outbox: pendientes y latencia de publicación.

## 6. Reporte
| Escenario | VUs | RPS | p95 lectura | p95 escritura | Errores | Cuello de botella | Acción |
|---|---|---|---|---|---|---|---|

Se guarda en `docs/rendimiento/<version>.md`. Una regresión > 20 % en p95 frente al release anterior bloquea el release hasta explicarla.
