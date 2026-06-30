# battlecaos-observability — Contexto para Claude

## ¿Qué hace este servicio?
Consume todos los eventos `evt:*` del sistema y construye KPIs agregados en Redis.
Lleva un audit log en memoria de los últimos eventos por sala para diagnóstico.
Expone opcionalmente un `GET /metrics` para consultar el estado del sistema en tiempo real.
Es un servicio de solo lectura (nunca publica eventos que afecten el juego).

**Regla de oro:** este servicio es PASIVO. Solo escucha y agrega. Nunca influye en el juego.

## Puerto y protocolo
- **HTTP:** `3003` (opcional — endpoint de métricas)
- **Protocolo:** Redis Pub/Sub (solo suscripción, no publica en canales de juego)
- Requiere **2 conexiones Redis** si tiene HTTP, o 1 si solo Pub/Sub

## Scaffolding completo

```
battlecaos-observability/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← suscripción a todos los evt:* + HTTP opcional
    ├── redis.js        ← fábrica createRedis()
    ├── logger.js       ← logger con prefijo [obs]
    └── kpi.js          ← funciones de agregación de KPIs
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
OBS_PORT=3003
```

## package.json

```json
{
  "name": "battlecaos-observability",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "node --watch src/index.js"
  },
  "dependencies": {
    "cors":      "^2.8.5",
    "dotenv":    "^16.4.0",
    "express":   "^4.19.0",
    "helmet":    "^7.1.0",
    "ioredis":   "^5.5.0"
  }
}
```

## Canales Redis — solo suscripción

| Canal | KPI que actualiza |
|---|---|
| `evt:GameStarted` | `stats:games:started` INCR |
| `evt:GameEnded` | `stats:games:ended` INCR + duración en hash diario |
| `evt:ShotFired` | `stats:kpi:{YYYY-MM-DD}` HINCRBY shots_total |
| `evt:ShipSunk` | `stats:kpi:{YYYY-MM-DD}` HINCRBY ships_sunk |
| `evt:PowerUsed` | `stats:kpi:{YYYY-MM-DD}` HINCRBY powers_used |
| `evt:PowerCompensated` | `stats:kpi:{YYYY-MM-DD}` HINCRBY countermeasures |
| `evt:ChatMessage` | `stats:kpi:{YYYY-MM-DD}` HINCRBY messages_sent |
| `evt:PlayerDisconnectedFromRoom` | `stats:kpi:{YYYY-MM-DD}` HINCRBY disconnections |
| `evt:RoomReady` | `stats:kpi:{YYYY-MM-DD}` HINCRBY rooms_ready |

## Claves Redis que escribe

```
stats:games:started         →  String (int) — contador total
stats:games:ended           →  String (int) — contador total
stats:kpi:{YYYY-MM-DD}      →  Hash — KPIs del día
```

Hash `stats:kpi:{YYYY-MM-DD}` contiene:
- `shots_total` — disparos procesados
- `ships_sunk` — barcos hundidos
- `powers_used` — poderes activados
- `countermeasures` — contramedidas exitosas
- `messages_sent` — mensajes de chat
- `disconnections` — desconexiones detectadas
- `rooms_ready` — salas que iniciaron partida
- `duration_sum_ms` — suma de duraciones de partidas (para promedio)
- `games_ended_today` — partidas terminadas hoy

## Endpoint de métricas (opcional)

### `GET /metrics`
```json
{
  "service": "observability",
  "timestamp": 1234567890,
  "total": {
    "gamesStarted": 42,
    "gamesEnded":   38
  },
  "today": {
    "shotsTotal":      187,
    "shipsSunk":        23,
    "powersUsed":       15,
    "countermeasures":   4,
    "messagesSent":     96,
    "disconnections":    3,
    "roomsReady":       12,
    "avgDurationMs":  185000
  }
}
```

### `GET /health`
```json
{ "service": "observability", "status": "ok", "timestamp": 1234567890 }
```

## Audit log en memoria
Mantener los últimos 500 eventos en un array circular para diagnóstico.
Útil para depurar secuencias de eventos durante el desarrollo.

```js
const auditLog = [];
const MAX_AUDIT = 500;

function appendAudit(event) {
  auditLog.push({ ts: Date.now(), ...event });
  if (auditLog.length > MAX_AUDIT) auditLog.shift();
}
```

## Sistema de logs

```js
// src/logger.js
const SVC = 'obs';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[obs] suscrito a 9 canales evt:*
[obs] GameStarted — sala 123456 — total: 43
[obs] ShotFired — sala 123456 — shots hoy: 188
[obs] GameEnded — sala 123456 — duración: 3m 12s — ganador: A
[obs] ERROR: parse fail en evt:ShotFired — SyntaxError
```

## Testing
Verificación manual:

```bash
# Publicar evento de prueba
redis-cli PUBLISH evt:GameStarted '{"type":"GameStarted","data":{"codigo":"123456","modo":"1v1"}}'
redis-cli PUBLISH evt:GameEnded '{"type":"GameEnded","data":{"codigo":"123456","winner":"A","duracion":192000}}'

# Consultar KPIs acumulados
redis-cli GET stats:games:started
redis-cli HGETALL stats:kpi:2026-06-27

# Consultar endpoint (si está activo)
curl http://localhost:3003/metrics
```

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + dos conexiones Redis | 1 - Fase 0 |
| DOMF1201 | Suscripción a todos evt:* + HINCRBY en hash diario | 2 - Fase 3 |
| DOMF1202 | KPIs globales: stats:games:started/ended | 2 - Fase 3 |
| DOMF1203 | GET /metrics + audit log en memoria | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
