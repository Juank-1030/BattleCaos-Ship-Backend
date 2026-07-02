# battlecaos-observability — Contexto para Claude

## ¿Qué hace este servicio?
Consume todos los eventos `evt.*` de Kafka y construye KPIs agregados en Redis State.
Lleva un audit log en memoria de los últimos eventos por sala para diagnóstico.
Los KPIs son expuestos por el Gateway en `GET /kpis` leyendo Redis State directamente.
Es un servicio de solo lectura (nunca publica eventos que afecten el juego).

**Regla de oro:** este servicio es PASIVO. Solo escucha Kafka y escribe KPIs en Redis State. Nunca influye en el juego.

## Puerto y protocolo
- **HTTP:** ninguno (los KPIs los sirve el Gateway en `/kpis`)
- **Mensajería:** Kafka (consume todos los `evt.*`)
- **Estado:** Redis State — escribe KPIs (HINCRBY / SET)
- **1 conexión Redis** (escritura KPIs) + **1 cliente Kafka** (consumer)

## Scaffolding completo

```
battlecaos-observability/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js    ← consumer Kafka + handlers de KPIs
    ├── redis.js    ← fábrica createRedis()
    ├── kafka.js    ← fábrica createConsumer
    └── logger.js   ← logger [obs]
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-obs
# KAFKA_USERNAME=
# KAFKA_PASSWORD=
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
    "dotenv":  "^16.4.0",
    "ioredis": "^5.5.0",
    "kafkajs": "^2.2.4"
  }
}
```

## Kafka

### Consume de:
| Topic | Tipos relevantes |
|---|---|
| `evt.room` | `RoomReady`, `PlayerDisconnectedFromRoom`, `PlayerReconnected`, `RoomDestroyed` |
| `evt.game` | `GameStarted`, `GameEnded`, `ShotFired`, `ShipSunk`, `PowerUsed`, `PowerCompensated` |
| `evt.timer` | `TimerEnd` |
| `evt.chat` | `ChatMessage` |
| `evt.bot` | `BotDecision` |

### No produce en Kafka.

## Redis State — claves que escribe

```
stats:kpi:{YYYY-MM-DD}   →  Hash — KPIs del día
metrics:pico:salas        →  String (int) — pico de salas concurrentes
```

Hash `stats:kpi:{YYYY-MM-DD}` contiene:
- `games_started` — partidas iniciadas
- `games_ended` — partidas completadas
- `shots_total` — disparos procesados
- `ships_sunk` — barcos hundidos
- `powers_used` — poderes activados
- `countermeasures` — contramedidas exitosas
- `messages_sent` — mensajes de chat
- `disconnections` — desconexiones detectadas
- `reconnections` — reconexiones exitosas
- `rooms_ready` — salas que iniciaron partida

## Handlers principales

```js
// GameStarted:
await redis.hincrby(`stats:kpi:${today}`, 'games_started', 1);

// GameEnded:
await redis.hincrby(`stats:kpi:${today}`, 'games_ended', 1);

// PlayerDisconnectedFromRoom:
await redis.hincrby(`stats:kpi:${today}`, 'disconnections', 1);

// PlayerReconnected:
await redis.hincrby(`stats:kpi:${today}`, 'reconnections', 1);
```

## Tareas pendientes

| ID | Descripción |
|---|---|
| DOMF001 | Init + Redis + Kafka consumer |
| DOMF1201 | Consumer todos los evt.* + HINCRBY en hash diario |
| DOMF1202 | KPIs globales: games_started / games_ended |
| DOMF1203 | Pico de salas concurrentes (metrics:pico:salas) |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
