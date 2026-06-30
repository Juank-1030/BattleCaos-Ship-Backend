# battlecaos-gateway — Contexto para Claude

## ¿Qué hace este servicio?
Hub central de la arquitectura. Es el único punto de contacto con los clientes web.
Recibe eventos de Socket.io, los enruta a los microservicios vía Redis Pub/Sub, y reenvía
las respuestas de los servicios como broadcasts a los clientes correctos.
También verifica JWT en cada conexión y aplica rate limiting.

**Regla de oro:** este servicio NO contiene lógica de negocio. Solo mapea y retransmite.

## Puerto y protocolo
- **HTTP:** `3000` (Express + health check)
- **WebSocket:** mismo puerto 3000 (Socket.io sobre HTTP)
- **Protocolo interno:** Redis Pub/Sub

## Scaffolding completo

```
battlecaos-gateway/
├── .env                      ← variables reales (no commitear)
├── .env.example              ← plantilla pública
├── .gitignore
├── package.json
├── CLAUDE.md                 ← este archivo
└── src/
    ├── index.js              ← punto de entrada: Express + Socket.io + Redis
    ├── redis.js              ← fábrica createRedis()
    └── logger.js             ← logger con prefijo [gateway]
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
GATEWAY_PORT=3000
JWT_SECRET=cambiar_en_produccion
CLIENT_ORIGIN=http://localhost:5173
```

## package.json

```json
{
  "name": "battlecaos-gateway",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js"
  },
  "dependencies": {
    "cors":           "^2.8.5",
    "dotenv":         "^16.4.0",
    "express":        "^4.19.0",
    "helmet":         "^7.1.0",
    "ioredis":        "^5.5.0",
    "jsonwebtoken":   "^9.0.0",
    "socket.io":      "^4.7.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
```

> `node --watch` falla silenciosamente en Node.js v22+ con ESM y top-level await.
> Se usa `nodemon` como reemplazo estable.
```

## Canales Redis

### Publica en:
| Canal | Cuándo |
|---|---|
| `svc:room` | Al recibir `room:create` o `room:join` del cliente |
| `svc:game` | Al recibir eventos de juego del cliente |
| `svc:chat` | Al recibir `chat:mensaje` del cliente |
| `evt:PlayerReconnected` | Al detectar reconexión de jugador con sala activa |

### Suscribe a:
| Canal | Qué hace con el mensaje |
|---|---|
| `gw:broadcast` | `io.to(roomId).emit(event, payload)` |
| `evt:TimerTick` | `io.to(data.codigo).emit('timer:tick', data)` |
| `evt:PhaseChanged` | `io.to(data.codigo).emit('phase:changed', data)` — Proyecto.md §7.1 |

## Formato exacto del evento publicado en Redis

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

- `correlationId`: usar `crypto.randomUUID()` (built-in Node.js 20, sin paquete extra)
- `playerId` viene de `socket.user.sub` (payload del JWT)
- Todos los servicios deben incluir `data.codigo` cuando aplique

## Tabla de enrutamiento Socket.io → Redis

```js
const ROUTES = {
  'room:create':          'svc:room',
  'room:join':            'svc:room',
  'disparo:realizar':     'svc:game',
  'salva:disparo':        'svc:game',
  'poder:usar':           'svc:game',
  'colocacion:set':       'svc:game',
  'contramedida:activar': 'svc:game',
  'chat:mensaje':         'svc:chat',
};
```

## Evento especial: `room:join-socket-room`
Cuando Room Service confirma que un jugador se unió, envía este evento con `{ codigo }`.
El Gateway debe ejecutar `socket.join(codigo)` para que ese socket reciba los broadcasts de sala.

```js
// En el handler de gw:broadcast:
if (event === 'room:join-socket-room') {
  const socket = io.sockets.sockets.get(roomId);
  socket?.join(payload.codigo);
  return;
}
```

## Sistema de logs

```js
// src/logger.js
const SVC = 'gateway';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[gateway] :3000
[gateway] cliente conectado: abc123
[gateway] WARN: rate_limit_exceeded 192.168.1.1
[gateway] ERROR: broadcast parse fail — SyntaxError
```

## Endpoints HTTP

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/health` | Estado del servicio + ping Redis. Devuelve `{ service, status, redis, uptime, timestamp }` |
| `GET` | `/kpis` | KPIs del día calculados por Observability Service y almacenados en Redis. Lee `stats:kpi:{YYYY-MM-DD}` y `metrics:pico:salas` |

### Respuesta `/health` (DOMF1302)
```json
{ "service": "gateway", "status": "ok", "redis": "ok", "uptime": 123.4, "timestamp": 1700000000 }
```

### Respuesta `/kpis` (UIUXF1101)
```json
{ "tasa_completacion": "80.0%", "tasa_reconexion": "95.0%", "pico_salas": "12", "date": "2026-06-28" }
```
> `/kpis` solo tiene datos si Observability Service está corriendo y acumulando eventos.

## Testing
Este servicio no tiene lógica de dominio, no requiere tests unitarios.
Verificación manual:
```bash
# 1. Arrancar Redis local
# 2. node src/index.js
# 3. curl http://localhost:3000/health
#    → { "service": "gateway", "status": "ok", "redis": "ok", "uptime": ..., "timestamp": ... }
# 4. curl http://localhost:3000/kpis
#    → { "tasa_completacion": "N/A", ... }   (N/A hasta que Observability acumule datos)
# 5. Conectar cliente Socket.io y emitir room:create
# 6. redis-cli SUBSCRIBE svc:room → ver el mensaje llegar
# 7. redis-cli PUBLISH evt:PhaseChanged '{"data":{"codigo":"123456","from":"COLOCACION","to":"TURNOS"}}'
#    → clientes en sala 123456 reciben evento 'phase:changed'
```

## Middleware de autenticación JWT
Verificar antes de cualquier evento. Rechazar con `Error('sin_token')` o `Error('token_invalido')`.
El payload del JWT queda en `socket.user = { sub, name, picture }`.
Todos los eventos enrutados deben incluir `socketId: socket.id` y el `playerId` del JWT.

## Rate limiting (Redis)
Ventana deslizante de 1 segundo, máximo 20 eventos por IP.
Clave: `ratelimit:{ip}` con TTL de 1 s via `INCR` + `EXPIRE`.

## Reconexión
Al conectar un socket, buscar en Redis si `socket.user.sub` tiene sala activa.
Si la tiene y `fase !== 'FIN'`: hacer `socket.join(codigo)`, actualizar `socketId` en la sala,
publicar `evt:PlayerReconnected` y enviar `game:state` solo a ese socket.

## Tareas del Gateway (Orden_de_Ejecucion.md + Proyecto.md)

| ID | Descripción | Sprint | Estado |
|---|---|---|---|
| DOMF001 | Inicialización base + conexión Redis + GET /health | 1 - Fase 0 | ✅ |
| DOMF002 | Tabla de enrutamiento Socket.io → Redis + rate limiting + PlayerDisconnected | 1 - Fase 0 | ✅ |
| DOMF102 | Middleware JWT en cada conexión Socket.io | 1 - Fase 1 | ✅ |
| DOMF301 | Suscribirse a `gw:broadcast`, `evt:TimerTick` y `evt:PhaseChanged` | 1 - Fase 2 | ✅ |
| DOMF401 | Detectar reconexión y publicar `evt:PlayerReconnected` | 1 - Fase 2 | ✅ |
| DOMF1302 | GET /health mejorado: ping Redis + `redis` + `uptime` en respuesta | 2 - Fase 3 | ✅ |
| UIUXF1101 | GET /kpis — expone KPIs de Observability Service leídos desde Redis | 2 - Fase 6 | ✅ |

> **Nota:** `UIUXF1101` y la suscripción a `evt:PhaseChanged` no estaban listadas en el `CLAUDE.gateway.md` original pero sí requeridas por `Orden_de_Ejecucion.md` y `Proyecto.md §7.1`.

## Arranque

```bash
cp .env.example .env   # rellenar valores reales
npm install
npm run dev            # node --watch
```
