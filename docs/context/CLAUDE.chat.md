# battlecaos-chat — Contexto para Claude

## ¿Qué hace este servicio?
Recibe mensajes de chat via Kafka, los persiste en Redis State (máx. 100 por sala),
los enruta y envía el historial al jugador al unirse. Publica `ChatMessage` para Observabilidad.

**Regla de oro:** este servicio NO valida lógica de juego. Solo mensajería.

## Puerto y protocolo
- **HTTP:** ninguno
- **Mensajería:** Kafka (consume `cmd.chat` + `evt.room`, produce `gw.broadcast` + `evt.chat`)
- **Estado:** Redis — `sala:{codigo}:chat` (List)
- **1 conexión Redis** (estado) + **1 cliente Kafka** (producer + consumer)

## Scaffolding completo

```
battlecaos-chat/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js    ← consumer Kafka + handlers
    ├── redis.js    ← fábrica createRedis()
    ├── kafka.js    ← fábrica producer + createConsumer
    └── logger.js   ← logger [chat]
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-chat
# KAFKA_USERNAME=
# KAFKA_PASSWORD=
```

## Kafka

### Consume de:
| Topic | Tipo | Acción |
|---|---|---|
| `cmd.chat` | `chat:mensaje` | Persistir y distribuir mensaje |
| `evt.room` | `PlayerRoomJoined` | Enviar historial al jugador que entró |

### Produce en:
| Topic | Cuándo |
|---|---|
| `gw.broadcast` | Nuevo mensaje (`chat:message`) o historial (`chat:history`) |
| `evt.chat` | Tras cada mensaje (para Observabilidad — tipo `ChatMessage`) |

## Redis State

| Operación | Clave | Para qué |
|---|---|---|
| `LPUSH` | `sala:{codigo}:chat` | Insertar mensaje (más reciente primero) |
| `LTRIM` | `sala:{codigo}:chat` | Mantener máx. 100 mensajes |
| `LRANGE 0 99` | `sala:{codigo}:chat` | Leer historial para nuevo jugador |

Al enviar historial al cliente, hacer `.reverse()` para orden cronológico.

## Estructura de un mensaje

```json
{ "senderId": "uid-A", "senderName": "Jugador A", "text": "Hola", "equipo": "A", "ts": 1234 }
```

## Broadcast de historial

```js
// roomId = playerId (socket individual, no toda la sala)
await producer.send({ topic: 'gw.broadcast', messages: [{
  value: JSON.stringify({ roomId: playerId, event: 'chat:history', payload: messages })
}]});
```

## Tareas pendientes

| ID | Descripción |
|---|---|
| DOMF001 | Init + Redis + Kafka |
| DOMF1103 | Recepción + persistencia LTRIM + enrutamiento + historial |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
