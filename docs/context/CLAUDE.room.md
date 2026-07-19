# battlecaos-room — Contexto para Claude

## ¿Qué hace este servicio?
Gestiona el ciclo de vida de las salas: creación, unión, asignación de equipos,
desconexión y destrucción. Es la fuente de verdad del estado `sala:{codigo}` en Redis State.
Consume comandos de Kafka y publica eventos de vuelta en Kafka.

**Regla de oro:** este servicio NO toca lógica de juego. Solo gestión de salas.

## Puerto y protocolo
- **HTTP:** ninguno
- **Mensajería:** Kafka (consume `cmd.room`, produce `gw.broadcast` + `evt.room`)
- **Estado:** Redis — solo lectura/escritura de `sala:{codigo}`
- **1 conexión Redis** (estado) + **1 cliente Kafka** (producer + consumer)

## Scaffolding completo

```
battlecaos-room/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js            ← consumer Kafka + handlers + helpers
    ├── redis.js            ← fábrica createRedis() — solo estado
    ├── kafka.js            ← fábrica producer + createConsumer
    ├── logger.js           ← logger con prefijo [room]
    └── domain/
        ├── room.js         ← funciones puras (sin Redis, sin Kafka)
        └── __tests__/
            └── room.test.js
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-room
# KAFKA_USERNAME=   # solo Upstash
# KAFKA_PASSWORD=   # solo Upstash
```

## package.json

```json
{
  "name": "battlecaos-room",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js",
    "test":  "vitest run"
  },
  "dependencies": {
    "dotenv":   "^16.4.0",
    "ioredis":  "^5.5.0",
    "kafkajs":  "^2.2.4"
  }
}
```

## Kafka

### Consume de:
| Topic | Tipos de mensaje |
|---|---|
| `cmd.room` | `room:create`, `room:join`, `PlayerDisconnected` |

### Produce en:
| Topic | Cuándo |
|---|---|
| `gw.broadcast` | Confirmar creación/unión (`room:created`, `room:joined`, errores) |
| `gw.broadcast` | Evento especial `room:join-socket-room` (Gateway hace `socket.join`) |
| `evt.room` | `PlayerRoomJoined`, `RoomReady`, `PlayerDisconnectedFromRoom`, `RoomDestroyed` |

## Redis State

| Operación | Clave | Para qué |
|---|---|---|
| `SET` | `sala:{codigo}` | Guardar sala al crear/unir/desconectar |
| `GET` | `sala:{codigo}` | Leer sala para validar/modificar |
| `KEYS sala:*` | — | Buscar sala de un socket desconectado (MVP) |

## Estructura de `sala:{codigo}` en Redis

```json
{
  "codigo":   "325982",
  "modo":     "1v1",
  "fase":     "LOBBY",
  "jugadores": [
    { "id": "google-uid-A", "name": "Jugador A", "socketId": "abc", "equipo": "A", "conectado": true, "desconectadoEn": null }
  ],
  "slotsMax": 2,
  "creadoEn": 1234567890
}
```

## Slots y equipos

```js
const SLOTS = { '1v1': 2, '1v1-bot': 1, '2v2': 4 };
const equipo = sala.jugadores.length % 2 === 0 ? 'A' : 'B';
```

`1v1-bot`: slotsMax=1, la sala queda lista al crearse (solo 1 humano).

## Regla crítica de desconexión (DOMF402)

```js
// CORRECTO — conservar jugador y su flota para reconexión
jugador.conectado      = false;
jugador.desconectadoEn = Date.now();

// INCORRECTO — NUNCA hacer esto
sala.jugadores = sala.jugadores.filter(j => j.id !== jugador.id);
```

## Evento especial: `room:join-socket-room`
Publicar en `gw.broadcast` con `event: 'room:join-socket-room'` y `payload: { codigo }`.
El Gateway ejecuta `socket.join(codigo)` para que ese socket reciba broadcasts de sala.

## Tareas completadas

| ID | Descripción | Estado |
|---|---|---|
| DOMF001 | Init + Redis + consumer Kafka | ✅ |
| DOMF201 | Crear/unirse a salas + equipos | ✅ |
| DOMF202 | Naming `sala:{codigo}` + `room:join-socket-room` | ✅ |
| DOMF203 | RoomReady, PlayerRoomJoined, PlayerDisconnectedFromRoom, RoomDestroyed | ✅ |
| DOMF402 | Marcar desconectado sin eliminar jugador | ✅ |
| DOMF403 | Eventos desconexión/reconexión (cubierto por DOMF203) | ✅ |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
