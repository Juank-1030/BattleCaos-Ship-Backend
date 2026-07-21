# Guía de pruebas — BattleCaos (paso a paso)

Cómo probar TODO el proyecto: tests unitarios, cobertura, el stack local, el flujo del juego,
los servicios desplegados en Azure, y la observabilidad (Grafana, Loki, Prometheus, Jaeger).
Comandos para **PowerShell en Windows** desde la carpeta raíz del proyecto (`E:\ARSW\Proyecto`).

> Rutas de referencia usadas abajo:
> - Azure CLI: `az` (si no está en PATH: `"C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin\az.cmd"`)
> - Gateway en Azure (dev): `https://battlecaosdev-gateway.orangeforest-5c4090bc.eastus2.azurecontainerapps.io`

---

## 0. Prerrequisitos

| Herramienta | Para qué | Verificar |
|---|---|---|
| Node.js ≥ 20 | correr servicios y tests | `node -v` |
| Docker Desktop | stack local + monitoreo | `docker version` |
| Azure CLI (logueada) | probar lo desplegado | `az account show` |
| k6 (opcional) | pruebas de carga | `k6 version` — portable: descargar de github.com/grafana/k6/releases |

La primera vez, instala dependencias en cada servicio: `cd battlecaos-<servicio>; npm install`.

---

## 1. Tests unitarios

**Un servicio:**
```powershell
cd battlecaos-game
npm test          # vitest run — 106 tests del dominio del juego
```

**Los 9 servicios de una vez:**
```powershell
foreach ($s in @("gateway","game","room","chat","voice-channel","timer","bot","observability","auth")) {
  Write-Host "== $s ==" -ForegroundColor Cyan
  Push-Location "battlecaos-$s"; npm test; Pop-Location
}
```

Estado esperado (2026-07-18): **272 tests, todos verdes** — gateway 27 · game 106 · room 52 ·
chat 15 · voice-channel 15 · timer 11 · bot 12 · observability 12 · auth 22.

Qué prueban (además del dominio): anti-suplantación de identidad, bounds de disparo, lock
distribuido (`lock.test.js`), DLQ (`dlq.test.js`), backoff exponencial (`backoff.test.js`),
failover de Redis incl. cuota agotada (`redis.test.js`), idempotencia de la persistencia
(`persistencia.test.js`) y el contrato de eventos contra el código (`event-contracts.test.js`).

## 2. Cobertura

```powershell
cd battlecaos-game
npm run coverage    # vitest run --coverage (provider V8)
```
Imprime la tabla por archivo (líneas/branches/funciones) y deja el HTML navegable en
`coverage/index.html` — ábrelo con `start coverage\index.html`. Repite en cualquier servicio
(los 9 tienen el script `coverage`).

---

## 3. Levantar el stack local completo

```powershell
# Levanta Redis, réplica de lectura, Zookeeper, Kafka, Kafka-UI y los microservicios
docker compose up -d --build
# (o el atajo del proyecto)  .\levantar-todo.ps1
```

Verificar que todo arrancó:
```powershell
docker ps --format "{{.Names}}: {{.Status}}"
# Kafka-UI (topics y mensajes en vivo): http://localhost:8080
# Réplica de Redis replicando:
docker exec battlecaos-redis-replica redis-cli INFO replication | Select-String "role|master_link"
#   role:slave / master_link_status:up  ← replicación viva
```

Para bajar todo: `docker compose down` (o `.\bajar-todo.ps1`).

---

## 4. Probar el flujo del juego

### 4a. A mano (frontend)
```powershell
cd battlecaos-frontend
npm install; npm run dev     # abre http://localhost:5173
```
Flujo a validar: login → crear sala `1v1-bot` → colocar flota → la partida pasa a TURNOS →
disparar → el bot responde → hundir la flota → **overlay de fin con "🔄 Revancha (misma sala)"**.
Para 1v1/2v2 abre dos pestañas (una en incógnito para tener otro usuario).

**Revancha:** al terminar, "Revancha (misma sala)" reinicia la partida con los MISMOS jugadores
y código, volviendo a COLOCACION sin pasar por el lobby. Cualquier jugador de la sala puede
dispararla; el resto vuelve a colocar su flota automáticamente.

### 4a-bis. Chat de voz entre redes distintas (WebRTC + TURN)

El audio va **P2P directo entre navegadores** (WebRTC); el backend solo dice con quién conectar y
relaya la señalización. Para que dos jugadores en **redes distintas** se escuchen hace falta un
**TURN** (relevo), porque el P2P directo falla cuando alguno está tras NAT simétrico (móvil/4G,
CGNAT, wifi corporativo). Sin TURN la voz queda "conectando/conectado" pero sin audio.

- **Config actual:** `battlecaos-voice-channel` sirve por defecto STUN de Google + TURN público
  OpenRelay. Es **best-effort** — los TURN públicos gratis son inestables.
- **Para juego cross-red CONFIABLE**, pon tu propio TURN vía la variable `VOICE_ICE_SERVERS`
  (JSON array de `RTCIceServer`) en el Container App de voz. Dos opciones:
  1. **metered.ca** (gratis con API key, 50GB/mes) — crea cuenta, copia tu lista de ICE servers.
  2. **coturn** self-hosted (necesita UDP + un rango de puertos → una VM, no Container Apps).
  ```powershell
  # Ejemplo (reemplaza por tus credenciales de TURN):
  $ice = '[{"urls":"stun:stun.l.google.com:19302"},{"urls":"turn:TU_TURN:3478","username":"USER","credential":"PASS"}]'
  az containerapp update -n battlecaosdev-voice-channel -g battlecaosdev-rg --set-env-vars "VOICE_ICE_SERVERS=$ice"
  ```
- **Diagnóstico:** en el navegador abre `chrome://webrtc-internals`, únete a la voz y mira los
  "candidate pairs": si el par seleccionado es tipo `relay`, está usando TURN (cross-red OK). Si
  se queda en `checking` sin llegar a `connected`, falta un TURN alcanzable.
- El cliente además **reintenta la conexión** (ICE restart hasta 2 veces) si falla el primer intento.

### 4b. Automatizado con k6 (el flujo REAL por WebSocket)
`tools/k6/juego.js` conecta por Engine.IO crudo, genera su propio JWT, crea una sala 1v1-bot,
coloca la flota y mide el round-trip hasta TURNOS:
```powershell
# Contra el stack LOCAL (JWT_SECRET del .env local):
k6 run -e BASE_GW_WS=ws://localhost:3000 -e SECRET=<JWT_SECRET local> -e VUS=10 tools/k6/juego.js

# Contra AZURE (SECRET = jwt_secret de battlecaos-infra/terraform/terraform.tfvars):
k6 run -e BASE_GW_WS=wss://battlecaosdev-gateway.orangeforest-5c4090bc.eastus2.azurecontainerapps.io `
       -e SECRET=<jwt_secret> -e VUS=20 tools/k6/juego.js
```
Umbrales que DEBEN pasar: `rt_crear_sala_ms p(95)<2000`, `rt_colocar_ms p(95)<2000`,
`llego_a_turnos rate>0.90`. Referencia real (2026-07-18, Azure): 99.7% completó, p95 ~300-380ms,
20 iter/s. También existen `tools/k6/carga.js` (HTTP) y `tools/k6/seguridad.js` (flood → verifica
que el rate limit dispara).

---

## 5. Probar los servicios desplegados en Azure

Los ~12 Container Apps (el "serverless" del proyecto) viven en el resource group `battlecaosdev-rg`.

### 5a. Estado y salud de cada servicio
```powershell
# ¿Qué está corriendo y con qué salud?
az containerapp list -g battlecaosdev-rg --query "[].{app:name, estado:properties.runningStatus}" -o table

# Salud REAL del gateway (verifica Redis Y el consumer de Kafka — si kafka!=ok, hay problema):
Invoke-WebRequest "https://battlecaosdev-gateway.orangeforest-5c4090bc.eastus2.azurecontainerapps.io/health" | Select-Object -Expand Content
# → {"service":"gateway","status":"ok","redis":"ok","kafka":"ok",...}

# KPIs de negocio en vivo (los escribe observability, los sirve el gateway):
Invoke-WebRequest ".../kpis" | Select-Object -Expand Content

# Logs en vivo de un servicio:
az containerapp logs show -n battlecaosdev-game -g battlecaosdev-rg --follow
```

### 5b. Autoescalado (verlo actuar)
```powershell
# Reglas configuradas (gateway: por conexiones; game/room/chat/voice: por lag de Kafka):
az containerapp show -n battlecaosdev-gateway -g battlecaosdev-rg --query "properties.template.scale"
# Genera carga con k6 (paso 4b, VUS=50) y observa las réplicas subir:
az containerapp replica list -n battlecaosdev-game -g battlecaosdev-rg -o table
```

### 5c. Deploy canary del gateway (blue/green)
```powershell
# Tras pushear una imagen :dev nueva al ACR — despliega a oscuras, verifica, 10%, promueve/revierte:
pwsh tools/deploy/canary.ps1 -CanaryPct 10 -ObservaS 60
```

### 5d. Chaos testing (prueba de recuperación)
```powershell
$env:PATH = "C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin;" + $env:PATH
$env:AZ="az"; $env:RG="battlecaosdev-rg"
$env:TARGET="redis"; node tools/chaos/experimento.mjs     # también: kafka | gateway
# MODO=duro (scale→0, caída real) — solo sin partidas activas. Ver tools/chaos/README.md
```

### 5e. Backup/restore de Mongo (DR)
Runbook completo y probado en `deploy/DR.md` (mongodump → restore a DB de prueba → verificación).

---

## 5.5. Cómo funciona el FAILOVER (cuando un servicio se cae, otro lo respalda)

La disponibilidad no es "que nunca se caiga" (eso no lo garantiza nadie) sino: **detectar** la
caída, **conmutar** automáticamente a un respaldo, y **perder poco y de forma acotada**. Cada
componente tiene su estrategia. Aquí está QUÉ pasa y CÓMO comprobarlo.

### Redis — Activo/Pasivo con doble escritura (el caso que preguntaste)

Es el ejemplo más claro de "base principal + respaldo". Hay **dos** Redis de proveedores
**independientes**: el PRIMARIO (Redis interno de Azure, rápido) y el RESPALDO (Upstash, en otra
nube). El cliente resiliente (`battlecaos-*/src/redis.js`) hace:

- **Escrituras** → van al primario Y se replican al respaldo (doble escritura). Así el respaldo
  SIEMPRE tiene el mismo estado → si el primario cae, no se pierde ninguna partida en curso.
- **Lecturas** → del primario; si no responde, del respaldo.
- **Detección + reincorporación** → un `PING` cada 3s marca cada nodo como vivo/caído y lo
  reincorpora solo cuando vuelve. Un error de conexión, timeout O **cuota agotada** (Upstash free
  tier) cuenta como "nodo caído" → conmuta al otro.

```
             escrituras          ┌───────────────┐
   servicio ───────────┬────────▶│ Redis PRIMARIO │ (Azure, rápido)
             lecturas   │  ▲      └───────────────┘
                        │  └─ si cae, lee/escribe aquí ▼
                        └───────▶┌───────────────┐
                    doble escritura│ Redis RESPALDO│ (Upstash, otra nube)
                                 └───────────────┘
```

**Compruébalo (chaos):** reinicia el primario y observa que el juego NO se cae:
```powershell
$env:PATH="C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin;"+$env:PATH; $env:AZ="az"; $env:RG="battlecaosdev-rg"
$env:TARGET="redis"; node tools/chaos/experimento.mjs
# /kpis (que lee Redis) sigue respondiendo 200 durante todo el reinicio.
```
En los logs del gateway verás la conmutación: `redis PRIMARIO caído → operando con el otro nodo`
y luego `redis PRIMARIO recuperado`. (Para forzar una caída DURA que ejerza el failover completo:
`$env:MODO="duro"` — solo sin partidas activas.)

### MongoDB — Activo/Pasivo gestionado (replica set)

La base durable (usuarios + historial de partidas) es **MongoDB Atlas**, que ya es un
**replica set de 3 nodos**: 1 primario + 2 secundarios. Si el primario cae, Atlas **promueve** un
secundario a primario **solo**, en segundos, sin que la app haga nada (el driver reconecta al nuevo
primario). Compruébalo:
```powershell
# La topología: verás setName + 3 hosts. El driver siempre te conecta al primario vigente.
docker run --rm mongo:7 mongosh "<MONGO_URL>" --quiet --eval "const h=db.hello(); print('replica set: '+h.setName); print('nodos: '+h.hosts.length)"
```
Si Atlas mismo (toda la cuenta) cayera → eso es DR, no HA: se recupera restaurando un backup
(runbook §5e / `deploy/DR.md`), con RPO/RTO definidos.

### Gateway — Activo/Activo (2 réplicas, sin punto único)

El gateway corre con **2 réplicas** (min=2). Son **stateless** (la sesión es el JWT, el estado
vive en Redis) → cualquier réplica atiende a cualquier usuario. Si una cae, la otra **sigue
sirviendo sin gap** y la plataforma repone la caída. Compruébalo:
```powershell
az containerapp replica list -n battlecaosdev-gateway -g battlecaosdev-rg -o table   # 2 réplicas
$env:TARGET="gateway"; node tools/chaos/experimento.mjs   # /health nunca deja de responder 200
```

### Servicios de dominio (game/room/chat/voice) — recovery con pérdida CERO

No tienen "respaldo activo" pero tienen algo mejor para su caso: **Kafka amortigua**. Si `game`
se cae, los comandos de los jugadores **se encolan en Kafka** (no se pierden, RPO=0); cuando la
réplica reinicia (la probe la repone o KEDA la escala), el consumer **reanuda desde donde quedó**.
Además: la **probe** de liveness detecta un consumer colgado (no solo un proceso muerto) y reinicia;
los reintentos usan **backoff exponencial**; y los mensajes que fallan van a la **DLQ** (no se pierden).

### Kafka — el único punto único (honesto)

El bus es **1 solo broker** (limitación del sandbox). Si se cae, no hay otro broker que lo respalde
en caliente: mientras se reinicia, los eventos no fluyen (los consumers reconectan con backoff al
volver, y el estado en Redis persiste). El plan para eliminarlo (3 brokers) está en
`deploy/HA-RONDA-B.md` — requiere recursos que exceden el sandbox.

### Resumen

| Componente | Patrón | Respaldo | Detección | Pérdida al caer |
|---|---|---|---|---|
| Redis | Activo/Pasivo (doble escritura) | Upstash (otra nube) | PING 3s + errores/cuota | 0 (respaldo sincronizado) |
| Mongo | Activo/Pasivo gestionado | 2 secundarios (Atlas) | Atlas (automático) | 0 (replica set) |
| Gateway | Activo/Activo | la otra réplica | probe /health | 0 (stateless) |
| game/room/chat/voice | Recovery vía cola | Kafka encola + KEDA | probe + kafka_consumer_up | 0 (RPO=0, se reanuda) |
| Kafka | ⚠️ único broker | — (plan documentado) | probe | eventos en vuelo hasta reiniciar |

---

## 6. Observabilidad: Grafana, Prometheus, Loki, Jaeger

### 6a. Levantar el stack de monitoreo
```powershell
cd deploy/monitoring
docker compose -f docker-compose.monitoring.yml up -d
```

| Herramienta | URL | Credenciales |
|---|---|---|
| Grafana | http://localhost:3030 | admin / battlecaos |
| Prometheus | http://localhost:9090 | — |
| Alertmanager | http://localhost:9093 | — |
| Loki (API) | http://localhost:3100 | — |
| Jaeger (trazas) | http://localhost:16686 | — |
| Kafka-UI | http://localhost:8080 | — |

### 6b. Grafana — métricas y dashboards
1. Entra a http://localhost:3030 → **Dashboards** → "BattleCaos — Operación" (viene pre-cargado).
2. Qué mirar: RPS y latencia p95 por servicio, tasa de error, `kafka_consumer_up` (1 = consumer
   vivo), lag de Kafka por consumer group, **error budget** de los SLOs (99% éxito de eventos /
   99.5% uptime del gateway — definidos en `deploy/monitoring/SLO.md`).
3. **Prometheus** (http://localhost:9090 → Status → Targets): todos los targets deben estar UP.
   Prueba una query: `rate(events_processed_total[1m])`.

### 6c. Loki — logs centralizados (Grafana → Explore)
1. Grafana → **Explore** → datasource **Loki**.
2. Consultas útiles (LogQL):
```logql
{container="battlecaos-game"}                       # todos los logs de game
{name="game", level_str="error"}                    # solo ERRORES de game
{container=~"battlecaos-.*"} |= "rate_limit"        # buscar un texto en TODOS los servicios
{container=~"battlecaos-.*"} |= "cid-abc123"        # ⭐ rastrear UN correlationId cross-servicio
```
3. El caso estrella: toma un `cid` de cualquier log (campo `cid` del JSON) y pégalo en la última
   query — ves el camino completo del evento por gateway → room → game → chat en orden.
4. El mismo rastreo sin Loki (en Azure): `GET <gateway>/trace/<cid>` — observability indexa cada
   evento por correlationId (TTL 1h).

### 6d. Jaeger — trazas OpenTelemetry (opt-in, local)
```powershell
# Arranca un servicio CON trazado (gateway y game ya tienen las deps instaladas):
cd battlecaos-game
$env:OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"; $env:SERVICE_NAME="game"
npm run start:traced
```
Genera actividad (p.ej. `Invoke-WebRequest http://localhost:9100/health` varias veces, o juega
una partida) → abre http://localhost:16686 → Service: `game` → **Find Traces**. Verás spans de
HTTP, Redis y Kafka con sus duraciones. *Nota: sin actividad no hay spans; sin la variable
`OTEL_EXPORTER_OTLP_ENDPOINT` el trazado queda desactivado (cero impacto).*

### 6e. Alertas
- Reglas activas: http://localhost:9090/alerts (13 reglas: servicio caído, consumer de Kafka
  caído, Redis degradado, error rate, p95 alto, lag alto, **mensajes en la DLQ**, SLOs violados…).
- Para RECIBIR notificaciones: edita `deploy/monitoring/alertmanager.yml` y cambia el topic
  `battlecaos-alertas-CAMBIAME` por uno privado tuyo; instala la app **ntfy** y suscríbete a ese
  topic. Prueba: apaga un servicio (`docker stop battlecaos-game`) y espera ~1 min la alerta.
- La DLQ en acción: publica un mensaje malformado a un topic `cmd.*` desde Kafka-UI → el evento
  cae al topic `dlq` (míralo en Kafka-UI) y dispara la alerta `MensajesEnDLQ`.

---

## 7. Checklist rápido de humo (5 min)

1. `npm test` en game → 106 verdes.
2. `docker compose up -d` → `docker ps` todo `Up`.
3. Frontend: partida 1v1-bot completa.
4. `Invoke-WebRequest <gateway-azure>/health` → `kafka:ok`.
5. Grafana → dashboard con datos moviéndose.
6. Loki → un `cid` rastreado cross-servicio.
