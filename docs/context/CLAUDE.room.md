# battlecaos-room — Contexto para Claude

## ¿Qué hace este servicio?
Gestiona el ciclo de vida completo de las salas: creación con código de 6 dígitos,
unión de jugadores, asignación de equipos, desconexión/reconexión y destrucción.
Es la fuente de verdad del estado `sala:{codigo}` en Redis y el publicador de todos
los eventos de ciclo de vida de sala (`evt:RoomReady`, `evt:RoomDestroyed`, etc.).

**Regla de oro:** este servicio NO toca lógica de juego. Solo gestión de salas y jugadores.

## Puerto y protocolo
- **HTTP:** ninguno (no tiene endpoints REST)
- **Protocolo:** Redis Pub/Sub exclusivamente
- Requiere 2 conexiones Redis: una para comandos (`redis`) y otra para suscripciones (`sub`)

## Scaffolding completo

```
battlecaos-room/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← suscripción a svc:room + handlers
    ├── redis.js        ← fábrica createRedis()
    └── logger.js       ← logger con prefijo [room]
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
```

## package.json

```json
{
  "name": "battlecaos-room",
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
| Canal | Tipos de mensaje que procesa |
|---|---|
| `svc:room` | `room:create`, `room:join`, `PlayerDisconnected` |

### Publica en:
| Canal | Cuándo |
|---|---|
| `gw:broadcast` | Confirmar creación/unión a sala al cliente |
| `evt:RoomReady` | Sala llena (todos los slots ocupados) |
| `evt:PlayerRoomJoined` | Cada vez que un jugador entra a la sala |
| `evt:PlayerDisconnectedFromRoom` | Jugador pierde conexión |
| `evt:PlayerReconnected` | Jugador reconecta (Gateway lo dispara, Room confirma) |
| `evt:RoomDestroyed` | Todos los jugadores desconectados |

## Estructura del estado de sala en Redis

```json
{
  "codigo": "123456",
  "modo": "1v1",
  "fase": "LOBBY",
  "jugadores": [
    {
      "id": "google-uid-A",
      "name": "Jugador A",
      "socketId": "socket-abc",
      "equipo": "A",
      "conectado": true,
      "desconectadoEn": null
    }
  ],
  "slotsMax": 2,
  "creadoEn": 1234567890
}
```

**Clave Redis:** `sala:{codigo}` (tipo String JSON)

## Slots por modo de juego

```js
const SLOTS = { '1v1': 2, '1v1-bot': 1, '2v2': 4 };
```

## Asignación de equipos
- Jugador 1 (creador) → equipo `A`
- Jugador 2 → equipo `B`
- En 2v2: posición 0,2 → `A`; posición 1,3 → `B`

## Evento especial: `room:join-socket-room`
Tras crear o unirse exitosamente, publicar en `gw:broadcast` con `event: 'room:join-socket-room'`
y `payload: { codigo }`. El Gateway ejecutará `socket.join(codigo)` para aislar los broadcasts.

## Sistema de logs

```js
// src/logger.js
const SVC = 'room';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[room] suscrito a svc:room
[room] sala creada: 123456 — modo: 1v1
[room] jugador google-uid-B unido a sala 123456
[room] WARN: sala_llena — codigo: 123456
[room] sala 123456 lista — publicando RoomReady
[room] jugador google-uid-A desconectado de sala 123456
```

## Testing
Verificación manual con redis-cli:

```bash
# Terminal 1: arrancar servicio
npm run dev

# Terminal 2: publicar evento de prueba
redis-cli PUBLISH svc:room '{"type":"room:create","source":"gateway","timestamp":1234,"data":{"socketId":"test-socket","modo":"1v1","playerId":"uid-A","name":"Test"}}'

# Terminal 3: escuchar respuesta
redis-cli SUBSCRIBE gw:broadcast
# → debe aparecer { roomId: "test-socket", event: "room:created", payload: { codigo: "XXXXXX" } }

redis-cli SUBSCRIBE evt:PlayerRoomJoined
# → debe aparecer el evento
```

## Desconexión — regla crítica
Al marcar desconexión, el jugador NO se elimina del array.
Solo se actualiza `conectado: false` y `desconectadoEn: Date.now()`.
La flota y el estado del jugador se conservan para reconexión.

```js
// CORRECTO
jugador.conectado = false;
jugador.desconectadoEn = Date.now();

// INCORRECTO — NUNCA hacer esto
sala.jugadores = sala.jugadores.filter(j => j.id !== jugador.id);
```

## Scan de claves (MVP)
Para encontrar la sala de un socket desconectado, en MVP se usa `redis.keys('sala:*')`.
Filtrar claves que contengan `:` para evitar sub-claves (`sala:X:chat`, etc.).

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + conexión Redis | 1 - Fase 0 |
| DOMF201 | Crear/unirse a salas con asignación de equipo | 1 - Fase 1 |
| DOMF202 | Convención de naming `sala:{codigo}:*` (restricción de diseño) | 1 - Fase 1 |
| DOMF203 | Publicar RoomReady, PlayerDisconnectedFromRoom, RoomDestroyed | 1 - Fase 1 |
| DOMF402 | Marcar desconectado sin eliminar jugador | 1 - Fase 2 |
| DOMF403 | Eventos de desconexión/reconexión (ya cubierto por DOMF203) | 1 - Fase 2 |
| DOMF1302 | GET /health mejorado (si se añade HTTP en el futuro) | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env   # rellenar REDIS_URL
npm install
npm run dev
```
