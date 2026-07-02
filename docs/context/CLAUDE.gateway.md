# battlecaos-gateway — Contexto para Claude

## ¿Qué hace este servicio?
Hub central. Único punto de contacto con los clientes web.
Recibe eventos Socket.io, los publica en Kafka (mensajería), y consume de Kafka
las respuestas de los servicios para reenviarlas como broadcasts a los clientes.
Verifica JWT en cada conexión y aplica rate limiting.

**Regla de oro:** este servicio NO contiene lógica de negocio. Solo mapea y retransmite.

## Puerto y protocolo
- **HTTP:** `3000` (Express + health check)
- **WebSocket:** mismo puerto 3000 (Socket.io)
- **Mensajería interna:** Kafka (produce y consume topics)
- **Estado:** Redis solo para rate limiting, reconexión y KPIs

## Scaffolding completo

```
battlecaos-gateway/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js              ← Express + Socket.io + Redis + Kafka
    ├── redis.js              ← fábrica createRedis() — solo estado
    ├── kafka.js              ← fábrica producer + createConsumer
    ├── logger.js             ← logger con prefijo [gateway]
    └── domain/
        ├── router.js         ← ROUTES (evento → topic Kafka) + buildMessage
        └── __tests__/
            └── router.test.js
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379
GATEWAY_PORT=3000
JWT_SECRET=cambiar_en_produccion_minimo_32_caracteres
CLIENT_ORIGIN=http://localhost:5173

# Kafka (Upstash o local Docker)
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-gateway
# KAFKA_USERNAME=   # solo Upstash
# KAFKA_PASSWORD=   # solo Upstash
```

## package.json

```json
{
  "name": "battlecaos-gateway",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js",
    "test":  "vitest run",
    "coverage": "vitest run --coverage"
  },
  "dependencies": {
    "cors":         "^2.8.5",
    "dotenv":       "^16.4.0",
    "express":      "^4.19.0",
    "helmet":       "^7.1.0",
    "ioredis":      "^5.5.0",
    "jsonwebtoken": "^9.0.0",
    "kafkajs":      "^2.2.4",
    "socket.io":    "^4.7.0"
  }
}
```

## Kafka topics

### Produce en:
| Topic | Cuándo |
|---|---|
| `cmd.room` | Al recibir `room:create`, `room:join`, `PlayerDisconnected` del socket |
| `cmd.game` | Al recibir disparos, poderes, colocación, salva, contramedida |
| `cmd.chat` | Al recibir `chat:mensaje` |
| `evt.room` | Al detectar reconexión — publica `PlayerReconnected` |

### Consume de:
| Topic | Qué hace con el mensaje |
|---|---|
| `gw.broadcast` | `io.to(roomId).emit(event, payload)` |
| `evt.timer` | Si `type === 'TimerTick'` → `io.to(codigo).emit('timer:tick', data)` |
| `evt.game` | Si `type === 'PhaseChanged'` → `io.to(codigo).emit('phase:changed', data)` |

## Redis State (solo lectura/escritura de datos)

| Operación | Clave | Para qué |
|---|---|---|
| `INCR / EXPIRE` | `ratelimit:{ip}` | Rate limiting |
| `KEYS / GET / SET` | `sala:*` | Detectar reconexión y actualizar socketId |
| `HGETALL` | `stats:kpi:{fecha}` | Endpoint /kpis |
| `GET` | `metrics:pico:salas` | Endpoint /kpis |
| `PING` | — | GET /health |

## Tabla de enrutamiento Socket.io → Kafka

```js
const ROUTES = {
  'room:create':          'cmd.room',
  'room:join':            'cmd.room',
  'disparo:realizar':     'cmd.game',
  'salva:disparo':        'cmd.game',
  'poder:usar':           'cmd.game',
  'colocacion:set':       'cmd.game',
  'contramedida:activar': 'cmd.game',
  'chat:mensaje':         'cmd.chat',
};
```

## Formato del mensaje publicado en Kafka

```json
{
  "type":          "room:create",
  "source":        "gateway",
  "timestamp":     1700000000123,
  "version":       1,
  "correlationId": "uuid-trazabilidad",
  "data": {
    "socketId": "abc123",
    "playerId": "google-uid-456",
    "modo":     "1v1"
  }
}
```

- `correlationId`: `crypto.randomUUID()` (built-in Node.js 20)
- `playerId` viene de `socket.user.sub` (JWT), nunca del body del cliente
- La `key` de Kafka es `payload.codigo ?? socket.id` para ordenar mensajes por sala

## Evento especial: `room:join-socket-room`
Room Service publica en `gw.broadcast` con `event: 'room:join-socket-room'`.
El Gateway ejecuta `socket.join(payload.codigo)` y retorna sin hacer emit.

## Reconexión
Al conectar un socket, buscar en Redis State si `socket.user.sub` tiene sala activa.
Si la tiene y `fase !== 'FIN'`:
1. `socket.join(codigo)`
2. Actualizar `socketId` y `conectado: true` en Redis State
3. Publicar `PlayerReconnected` en `evt.room` (Kafka)
4. `socket.emit('game:state', sala)` — solo a ese socket

## Rate limiting
Clave Redis: `ratelimit:{ip}` con `INCR` + `EXPIRE 1s`. Máx 20 eventos/IP/s.
Si supera el límite, rechazar con `Error('rate_limit_exceeded')` en el middleware.

## Tareas completadas

| ID | Descripción | Estado |
|---|---|---|
| DOMF001 | Init + Redis connect + GET /health | ✅ |
| DOMF002 | Tabla enrutamiento + rate limiting + PlayerDisconnected | ✅ |
| DOMF102 | Middleware JWT | ✅ |
| DOMF301 | Consume gw.broadcast, evt.timer, evt.game → broadcast Socket.io | ✅ |
| DOMF401 | Reconexión + PlayerReconnected | ✅ |
| DOMF1302 | GET /health con ping Redis | ✅ |
| UIUXF1101 | GET /kpis desde Redis State | ✅ |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
