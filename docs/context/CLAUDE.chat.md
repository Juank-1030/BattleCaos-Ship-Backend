# battlecaos-chat — Contexto para Claude

## ¿Qué hace este servicio?
Recibe mensajes de chat del Gateway, los persiste en Redis (máximo 100 por sala con LTRIM),
los enruta correctamente (en 2v2 solo al equipo, en 1v1 a todos) y envía el historial
completo al jugador cuando entra a una sala. Publica `evt:ChatMessage` para Observabilidad.

**Regla de oro:** este servicio NO valida lógica de juego. Solo mensajería.

## Puerto y protocolo
- **HTTP:** ninguno
- **Protocolo:** Redis Pub/Sub exclusivamente
- Requiere **2 conexiones Redis**: `redis` (comandos/publish) y `sub` (suscripciones)

## Scaffolding completo

```
battlecaos-chat/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← suscripción + handlers
    ├── redis.js        ← fábrica createRedis()
    └── logger.js       ← logger con prefijo [chat]
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
```

## package.json

```json
{
  "name": "battlecaos-chat",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "node --watch src/index.js"
  },
  "dependencies": {
    "dotenv":   "^16.4.0",
    "ioredis":  "^5.5.0"
  }
}
```

## Canales Redis

### Suscribe a:
| Canal | Qué hace |
|---|---|
| `svc:chat` | Procesar y distribuir nuevo mensaje |
| `evt:PlayerRoomJoined` | Enviar historial al jugador que acaba de entrar |

### Publica en:
| Canal | Cuándo |
|---|---|
| `gw:broadcast` | Nuevo mensaje (`chat:message`) o historial (`chat:history`) |
| `evt:ChatMessage` | Tras cada mensaje (para Observabilidad) |

## Clave Redis de mensajes

```
sala:{codigo}:chat   →  List (tipo Redis)
```

- `LPUSH` para insertar al inicio (más reciente primero)
- `LTRIM` para mantener máximo 100 mensajes: `ltrim(key, 0, 99)`
- `LRANGE(key, 0, 99)` para leer historial
- Al enviar historial al cliente, hacer `.reverse()` para orden cronológico

## Estructura de un mensaje persistido

```json
{
  "senderId":   "google-uid-A",
  "senderName": "Jugador A",
  "text":       "¡Buena jugada!",
  "equipo":     "A",
  "ts":         1234567890123
}
```

## Enrutamiento por equipo (modo 2v2)
En modo `2v2`, el chat es por equipo. El mensaje incluye el campo `equipo`.
El Gateway debe tener rooms de Socket.io separadas por equipo si se implementa
el filtrado en destino. En MVP, se puede enviar a toda la sala y filtrar en el cliente.

## Broadcast de historial al unirse
Al recibir `evt:PlayerRoomJoined`, leer `sala:{codigo}:chat`,
deserializar, invertir orden y enviar solo al socket del jugador que se unió:

```js
// roomId = playerId (se enruta al socket individual, no a toda la sala)
await redis.publish('gw:broadcast', JSON.stringify({
  roomId: playerId,     // ← socket individual, no sala completa
  event: 'chat:history',
  payload: messages,
}));
```

## Sistema de logs

```js
// src/logger.js
const SVC = 'chat';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[chat] suscrito a svc:chat, evt:PlayerRoomJoined
[chat] sala 123456 — mensaje de Jugador A (equipo A): "¡Buena jugada!"
[chat] sala 123456 — historial enviado a google-uid-B (42 mensajes)
[chat] WARN: mensaje vacío ignorado — senderId: uid-A
```

## Testing
Verificación manual:

```bash
# Terminal 1: arrancar servicio
npm run dev

# Terminal 2: simular mensaje
redis-cli PUBLISH svc:chat '{"type":"chat:mensaje","data":{"codigo":"123456","senderId":"uid-A","senderName":"Jugador A","text":"Hola","equipo":"A"}}'

# Terminal 3: escuchar broadcast
redis-cli SUBSCRIBE gw:broadcast
# → { roomId: "123456", event: "chat:message", payload: { ... } }

redis-cli SUBSCRIBE evt:ChatMessage
# → evento para observabilidad
```

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + dos conexiones Redis | 1 - Fase 0 |
| DOMF1103 | Mensajería: recepción + persistencia LTRIM + enrutamiento + historial | 1 - Fase 2 |
| DOMF1302 | Health check mejorado (si se añade HTTP) | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
