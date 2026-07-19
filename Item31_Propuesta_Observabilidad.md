# Ítem 31 — Reto final: Propuesta de Observabilidad para BattleCaos-Ship

> Resuelve el punto 31 de `Guia_Observabilidad.md` ("Reto final") aplicado al proyecto real del curso.
> La guía está escrita sobre Spring Boot + Actuator + Micrometer; este documento traduce cada pieza a
> nuestro stack real: **Node.js 20 (ESM) + Express + Socket.io + Redis + Kafka + Pino**.

---

## 0. Alcance y decisión de diseño

La consigna pide una propuesta "inicialmente configurada para **2 microservicios desarrollados**".

De los 4 repositorios que ya existen y funcionan (`battlecaos-auth`, `battlecaos-gateway`,
`battlecaos-room`, `battlecaos-game`), solo **dos exponen un servidor HTTP** con puerto publicado en
`docker-compose.yml`:

| Servicio | Puerto | ¿Por qué es buen candidato? |
|---|:---:|---|
| `battlecaos-auth` | 3001 | Expone `/health`, ya usa Pino, flujo de negocio claro (login) que da pie a métricas custom. |
| `battlecaos-gateway` | 3000 | Expone `/health` y `/kpis`, es el único punto de entrada de clientes (Socket.io), concentra rate limiting, JWT, reconexión y enrutamiento a Kafka — el servicio con más señales interesantes de todo el sistema. |

`battlecaos-room` y `battlecaos-game` son consumidores puros de Kafka sin servidor HTTP (confirmado
leyendo su `src/index.js`), así que no pueden ser scrapeados por Prometheus sin añadirles un servidor
HTTP mínimo. Quedan fuera del alcance de este reto y se documentan como "siguiente iteración" en la
sección 9.

**Importante — no confundir dos cosas distintas del proyecto:**

1. **Este documento (ítem 31 de la guía):** observabilidad de infraestructura clásica — Prometheus
   scrapea métricas HTTP/proceso de cada servicio, Grafana las visualiza, Loki centraliza logs. No
   requiere ningún microservicio nuevo, solo instrumentar `auth` y `gateway` con un endpoint `/metrics`.
2. **`battlecaos-observability` (DOMF1201-DOMF1203 en `Orden_de_Ejecucion.md`):** un microservicio de
   **dominio** aparte, que se suscribe a los eventos de Kafka/Redis para llevar un audit trail y calcular
   los 4 KPIs de negocio (`tasa_completacion`, `tasa_reconexion`, `pico_salas`, latencia P95 de disparos)
   y guardarlos en Redis para que el Gateway los sirva en `/kpis`. Es complementario, no sustituto.

Si más adelante quieres que implemente `battlecaos-observability` como microservicio real, **avísame
para que primero inicialices ese repositorio en git** — no he creado ni voy a crear ningún repo por mi
cuenta. Todo lo de este documento son fragmentos de código de referencia para que tú (o yo, cuando me
lo pidas explícitamente) los apliques sobre `battlecaos-auth` y `battlecaos-gateway`, que ya son repos
existentes tuyos.

---

## 1. Descripción de la arquitectura de observabilidad

```
┌─────────────┐     ┌─────────────┐
│  auth :3001 │     │gateway :3000│
│ /metrics    │     │ /metrics    │
│ /health     │     │ /health     │
│ logs (Pino  │     │ logs (Pino  │
│  JSON stdout│     │  JSON stdout│
└──────┬──────┘     └──────┬──────┘
       │ scrape (5s)        │ scrape (5s)
       ▼                    ▼
   ┌────────────────────────────┐
   │        Prometheus          │◄── scrapea /metrics de cada servicio
   └──────────────┬─────────────┘
                   │
   ┌───────────────┴─────────────┐
   │  stdout de contenedores      │
   │  (docker logs auth/gateway)  │
   └──────────────┬───────────────┘
                   │ tail
                   ▼
             ┌───────────┐        ┌────────┐
             │ Promtail  ├───────►│  Loki  │
             └───────────┘        └────┬───┘
                                        │
   ┌────────────────────────────────────┴──┐
   │                Grafana                 │
   │  - Data source: Prometheus             │
   │  - Data source: Loki                   │
   │  - Dashboard "BattleCaos - Observ."    │
   │  - Alertas                             │
   └─────────────────────────────────────────┘
```

Diferencias clave frente al laboratorio original:

- No hay Spring Boot Actuator → se usa la librería `prom-client` (equivalente Node.js de Micrometer) para
  exponer `/metrics` en formato Prometheus.
- No hace falta reconfigurar el logger: `auth` y `gateway` **ya loguean JSON estructurado con Pino** a
  stdout. Promtail solo necesita leer los logs del contenedor Docker (`docker_sd_configs`), sin tocar
  código de aplicación.
- Redis y Kafka (broker + zookeeper) ya corren en el mismo `docker-compose.yml` raíz del proyecto — el
  stack de observabilidad se añade como servicios adicionales en ese mismo archivo, reutilizando la red
  por defecto de Compose para que `prometheus:9090`, `grafana:3000` (¡ojo, choca con el puerto interno
  3000 del gateway — se publica en `3300:3000` en el host, ver sección 3), `loki:3100` sean resolubles
  por nombre.

---

## 2. Instrumentación: exponer `/metrics` en `auth` y `gateway`

Dependencia nueva en ambos `package.json`:

```bash
npm install prom-client
```

### 2.1 `battlecaos-auth/src/metrics.js` (nuevo archivo)

```js
import client from 'prom-client';

export const register = new client.Registry();
client.collectDefaultMetrics({ register, prefix: 'auth_' }); // CPU, heap, event loop lag, GC...

export const httpDuration = new client.Histogram({
  name: 'http_server_requests_seconds',
  help: 'Duración de solicitudes HTTP',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  registers: [register],
});

// Métricas de negocio — equivalentes a orders_created_total / orders_failed_total de la guía
export const loginSuccessTotal = new client.Counter({
  name: 'auth_login_success_total',
  help: 'Logins con Google verificados correctamente',
  registers: [register],
});

export const loginFailedTotal = new client.Counter({
  name: 'auth_login_failed_total',
  help: 'Intentos de login con token inválido o expirado',
  registers: [register],
});
```

### 2.2 Middleware de latencia HTTP (reutilizable en ambos servicios)

```js
// src/httpMetricsMiddleware.js
export function httpMetricsMiddleware(httpDuration) {
  return (req, res, next) => {
    const end = httpDuration.startTimer();
    res.on('finish', () => {
      end({
        method: req.method,
        route:  req.route?.path ?? req.path,
        status: res.statusCode,
      });
    });
    next();
  };
}
```

### 2.3 Cambios en `battlecaos-auth/src/index.js`

```js
import { register, httpDuration, loginSuccessTotal, loginFailedTotal } from './metrics.js';
import { httpMetricsMiddleware } from './httpMetricsMiddleware.js';

app.use(httpMetricsMiddleware(httpDuration));

app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Dentro de POST /auth/google, en el bloque try:
loginSuccessTotal.inc();
// ...y en el catch:
loginFailedTotal.inc();
```

### 2.4 `battlecaos-gateway/src/metrics.js` (nuevo archivo)

```js
import client from 'prom-client';

export const register = new client.Registry();
client.collectDefaultMetrics({ register, prefix: 'gateway_' });

export const httpDuration = new client.Histogram({
  name: 'http_server_requests_seconds',
  help: 'Duración de solicitudes HTTP',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  registers: [register],
});

export const socketConnectionsTotal = new client.Counter({
  name: 'gateway_socket_connections_total',
  help: 'Conexiones Socket.io aceptadas',
  registers: [register],
});

export const eventsRoutedTotal = new client.Counter({
  name: 'gateway_events_routed_total',
  help: 'Eventos de cliente enrutados a Kafka, por tipo',
  labelNames: ['event'],
  registers: [register],
});

export const rateLimitExceededTotal = new client.Counter({
  name: 'gateway_rate_limit_exceeded_total',
  help: 'Conexiones/eventos rechazados por rate limiting',
  registers: [register],
});

export const reconnectionsTotal = new client.Counter({
  name: 'gateway_reconnections_total',
  help: 'Jugadores reconectados a una sala activa',
  registers: [register],
});

export const kafkaConsumerCrashTotal = new client.Counter({
  name: 'gateway_kafka_consumer_crash_total',
  help: 'Veces que el consumer de Kafka del gateway crasheó',
  registers: [register],
});

export const activeSocketsGauge = new client.Gauge({
  name: 'gateway_active_sockets',
  help: 'Sockets conectados actualmente',
  registers: [register],
});
```

### 2.5 Cambios en `battlecaos-gateway/src/index.js`

```js
import {
  register, httpDuration, socketConnectionsTotal, eventsRoutedTotal,
  rateLimitExceededTotal, reconnectionsTotal, kafkaConsumerCrashTotal, activeSocketsGauge,
} from './metrics.js';
import { httpMetricsMiddleware } from './httpMetricsMiddleware.js';

app.use(httpMetricsMiddleware(httpDuration));

app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// En io.on('connection', ...):
socketConnectionsTotal.inc();
activeSocketsGauge.inc();
socket.on('disconnect', () => activeSocketsGauge.dec());

// En el middleware de rate limiting, cuando count > 20:
rateLimitExceededTotal.inc();

// En el loop `for (const [event, topic] of Object.entries(ROUTES))`, dentro del socket.on(event, ...):
eventsRoutedTotal.inc({ event });

// En la reconexión exitosa (dentro del `if (jugador && sala.fase !== 'FIN')`):
reconnectionsTotal.inc();

// En consumer.on(consumer.events.CRASH, ...):
kafkaConsumerCrashTotal.inc();
```

> Nota: `/metrics` no lleva autenticación JWT (a diferencia de los eventos Socket.io) porque Prometheus
> no manda `auth.token`. En producción se restringe por red (solo la VPC del stack de monitoreo puede
> alcanzar el puerto) o con un `helmet`/proxy que exija un header estático `X-Metrics-Token`.

---

## 3. Stack Docker Compose (Prometheus + Grafana + Loki + Promtail)

Añadir estos servicios al `docker-compose.yml` raíz (`E:\ARSW\Proyecto\docker-compose.yml`), junto a los
que ya existen:

```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: battlecaos-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml
    depends_on:
      - auth
      - gateway

  grafana:
    image: grafana/grafana:latest
    container_name: battlecaos-grafana
    ports:
      - "3300:3000"          # 3000 interno del gateway ya está tomado en el host
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
      - loki

  loki:
    image: grafana/loki:latest
    container_name: battlecaos-loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yml
    volumes:
      - ./observability/loki-config.yml:/etc/loki/local-config.yml

  promtail:
    image: grafana/promtail:latest
    container_name: battlecaos-promtail
    volumes:
      - ./observability/promtail-config.yml:/etc/promtail/config.yml
      - /var/run/docker.sock:/var/run/docker.sock   # necesario para leer logs de otros contenedores
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
```

`observability/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'battlecaos-auth'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['auth:3001']

  - job_name: 'battlecaos-gateway'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['gateway:3000']
```

(Se usa el nombre del servicio Compose, `auth`/`gateway`, en vez de `host.docker.internal`, porque
Prometheus corre en la misma red interna de Docker Compose.)

`observability/loki-config.yml`: idéntico al de la guía (sección 14), sin cambios necesarios.

`observability/promtail-config.yml` — versión adaptada para leer logs de contenedores Docker vía
`docker_sd_configs` (más simple que mapear archivos, y capta automáticamente los logs JSON de Pino de
`auth` y `gateway`):

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: 'container'
      - source_labels: ['__meta_docker_container_name']
        regex: '/battlecaos-(auth|gateway|room|game)'
        target_label: 'service'
        replacement: '$1'
```

---

## 4. Dashboard "BattleCaos — Observabilidad" en Grafana

| # | Panel | Consulta PromQL / LogQL | Tipo |
|---|---|---|---|
| 1 | Disponibilidad por servicio | `up{job=~"battlecaos-.*"}` | Stat (uno por `job`) |
| 2 | Requests/seg por endpoint | `sum by (route, method, status) (rate(http_server_requests_seconds_count[1m]))` | Time series |
| 3 | Latencia promedio | `sum(rate(http_server_requests_seconds_sum[1m])) / sum(rate(http_server_requests_seconds_count[1m]))` | Time series |
| 4 | Errores HTTP 5xx | `sum(rate(http_server_requests_seconds_count{status=~"5.."}[1m]))` | Time series |
| 5 | Logins OK vs fallidos | `auth_login_success_total`, `auth_login_failed_total` | Stat (2 valores) |
| 6 | Sockets activos | `gateway_active_sockets` | Gauge |
| 7 | Eventos enrutados por tipo | `sum by (event) (rate(gateway_events_routed_total[1m]))` | Time series (barras) |
| 8 | Rate limit / reconexiones | `rate(gateway_rate_limit_exceeded_total[1m])`, `rate(gateway_reconnections_total[1m])` | Time series |
| 9 | Memoria del proceso | `auth_process_resident_memory_bytes`, `gateway_process_resident_memory_bytes` | Time series |
| 10 | CPU del proceso | `rate(auth_process_cpu_user_seconds_total[1m])`, `rate(gateway_process_cpu_user_seconds_total[1m])` | Time series |
| 11 | Logs en vivo | `{service=~"auth|gateway"}` (Loki, panel Logs) | Logs |
| 12 | Errores en logs | `{service=~"auth|gateway"} \|= "error"` | Logs |

---

## 5. Simulación de incidente (plantilla resuelta)

**Incidente elegido: ráfaga de eventos que dispara el rate limiter del Gateway.**

Es el incidente más representativo de este proyecto porque combina lógica de negocio propia
(`DOMF002`), no es un `Thread.sleep` artificial como en la guía original.

Reproducción:

```bash
# Con un cliente Socket.io autenticado, emitir > 20 eventos en < 1s
for i in $(seq 1 30); do
  node -e "require('socket.io-client')('http://localhost:3000',{auth:{token:process.env.TOKEN}}).emit('chat:mensaje',{})"
done
```

| Campo | Valor |
|---|---|
| Incidente observado | Ráfaga de eventos desde un mismo socket/IP en menos de 1 segundo. |
| Métrica afectada | `rate(gateway_rate_limit_exceeded_total[1m])` sube de 0 a > 0; `gateway_events_routed_total` se aplana (los eventos rechazados no llegan a Kafka). |
| Logs relacionados | `log.warn('rate_limit_exceeded', socket.handshake.address)` — visible en Loki filtrando `{service="gateway"} \|= "rate_limit_exceeded"`. |
| Endpoint involucrado | No es HTTP — es el middleware Socket.io `io.use(...)` de `gateway/src/index.js`. |
| Posible causa | Cliente con bug (loop sin control) o intento de abuso/spam. |
| Impacto para el usuario | El jugador recibe `next(new Error('rate_limit_exceeded'))` y su conexión se corta; sus disparos/mensajes no llegan a Kafka → estado inconsistente hasta que reconecta. |
| Acción correctiva propuesta | Backoff exponencial en el cliente frontend al recibir `rate_limit_exceeded`; considerar rate limit distinto para eventos de gameplay (`disparo:realizar`) vs. chat, ya que un jugador legítimo en fase `SALVA` puede disparar varias veces en ráfaga. |
| Alerta que debería existir | Ver Alerta 3 en la sección 6. |

---

## 6. Propuesta de 3 alertas (Grafana Alerting)

**Alerta 1 — Servicio caído**
```
up{job=~"battlecaos-auth|battlecaos-gateway"} == 0
```
Evalúa cada 30s, dispara tras 1 min en estado `firing`. Crítico: sin Gateway no hay jugadores nuevos ni
reconexiones; sin Auth nadie puede iniciar sesión.

**Alerta 2 — Tasa de login fallido anómala**
```
rate(auth_login_failed_total[5m]) / (rate(auth_login_failed_total[5m]) + rate(auth_login_success_total[5m])) > 0.3
```
Más del 30% de logins fallando en una ventana de 5 min. Indica `GOOGLE_CLIENT_ID` mal configurado, caída
de la API de Google, o intento de fuerza bruta con tokens inválidos.

**Alerta 3 — Rate limiting excesivo en el Gateway**
```
sum(rate(gateway_rate_limit_exceeded_total[1m])) > 5
```
Más de 5 rechazos/seg sostenidos: o hay un cliente abusivo, o el límite de 20 eventos/seg (hardcodeado en
`gateway/src/index.js`) es demasiado estricto para el patrón real de juego (p.ej. fase `SALVA` con varios
jugadores disparando a la vez) y debería revisarse.

Todas las alertas deben enrutarse a un *contact point* de Grafana (Slack/webhook/correo) — para el MVP
académico basta con el canal de notificaciones por correo integrado.

---

## 7. Evidencia a capturar (checklist para el estudiante)

Estos son los puntos 2–6 del reto que **requieren levantar el entorno y no pueden generarse sin
ejecutarlo**. Pasos:

1. `docker compose up -d` (con los servicios de observabilidad añadidos según sección 3).
2. Generar tráfico real: login con Google, crear/unirse a sala, disparar, forzar `simulate`-style errores
   (p. ej. mandar un `idToken` inválido a `/auth/google` para poblar `auth_login_failed_total`).
3. Capturas a incluir en la entrega final:
   - `http://localhost:9090/targets` → ambos jobs en estado `UP`.
   - Dashboard de Grafana completo (los 12 paneles de la sección 4) con datos reales.
   - `curl http://localhost:3001/metrics` y `curl http://localhost:3000/metrics` mostrando
     `http_server_requests_seconds_count` y las métricas custom con valor > 0.
   - Panel "Logins OK vs fallidos" con al menos un fallo real provocado a propósito.
   - Grafana → Explore → Loki, consulta `{service="gateway"} |= "rate_limit_exceeded"` con resultados.
   - El incidente de la sección 5 reproducido en vivo (métrica + log correlacionados).

---

## 8. Recomendaciones para llevar la solución a producción

1. **No exponer `/metrics` públicamente.** Restringir por red interna (VPC/security group) o exigir un
   token estático, ya que revela topología interna y volumen de tráfico.
2. **Retención y coste de Loki/Prometheus.** El proyecto ya usa Redis administrado (Upstash) y probable
   despliegue en Render; en producción conviene un backend de métricas gestionado (Grafana Cloud free
   tier, o Prometheus con `remote_write` a un almacenamiento de largo plazo) en vez de contenedores
   efímeros con `emptyDir`.
3. **Correlación de logs con `correlationId`.** El proyecto ya incluye `correlationId`/`codigo` en cada
   evento Redis/Kafka (ver `DOMF202`, `buildMessage()` en `gateway/domain/router.js`). Basta con que
   Pino incluya ese campo en cada log (`log.info({ codigo, correlationId }, 'mensaje')`) para poder cruzar
   una traza completa Gateway → Kafka → Room/Game en Loki sin necesitar un backend de trazas dedicado.
   Es el camino más barato hacia lo que la guía llama "trazas" (sección 25-26) sin añadir el
   OpenTelemetry Collector todavía.
4. **Alertas accionables, no ruidosas.** Las 3 alertas propuestas deben ir a un canal con dueño claro
   (el equipo del curso) y con umbrales ajustados tras observar tráfico real un par de días — un umbral
   copiado literal de la guía (`> 1` seg de latencia) puede no aplicar a un juego en tiempo real donde
   Socket.io, no HTTP, es el canal principal.
5. **Extender la instrumentación a `room` y `game`.** Ambos son consumidores puros de Kafka sin servidor
   HTTP; para que Prometheus los scrapee basta con levantar un mini servidor Express interno solo con
   `/health` + `/metrics` (patrón ya usado en `auth`), sin exponerlo a Internet — solo a la red del stack
   de monitoreo.
6. **Dashboards separados por audiencia.** Un dashboard técnico (el de este documento) para el equipo de
   desarrollo, y uno de negocio simplificado reutilizando el endpoint `/kpis` que ya expone el Gateway
   (`tasa_completacion`, `tasa_reconexion`, `pico_salas`) para quien solo necesita ver salud del producto.

---

## 9. Próximos pasos posibles (requieren tu confirmación)

- [x] Aplicar los cambios de la sección 2 sobre `battlecaos-auth` y `battlecaos-gateway` — **hecho y
      verificado en vivo** (2026-07-04): `npm install prom-client` en ambos repos, `/metrics` responde
      en los dos, `auth_login_failed_total` se incrementó con un login inválido real, y
      `gateway_socket_connections_total` / `gateway_events_routed_total{event="room:create"}` se
      incrementaron con un cliente Socket.io real conectado y autenticado. Pendiente: hacer commit en
      cada repo cuando lo revises.
- [ ] Añadir los servicios de la sección 3 al `docker-compose.yml` raíz y crear la carpeta
      `observability/` con los 3 archivos de configuración (no es un microservicio, son solo configs).
- [ ] Instrumentar `room` y `game` con un mini `/health` + `/metrics` (sección 8, punto 5).
- [ ] Implementar el microservicio de dominio `battlecaos-observability` (DOMF1201-1203) — **este sí es
      un repositorio nuevo; si lo quieres, avísame para que lo inicialices en git primero.**

Dime cuáles de estos pasos quieres que ejecute y en qué orden.
