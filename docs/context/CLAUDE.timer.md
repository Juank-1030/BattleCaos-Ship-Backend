# battlecaos-timer — Contexto para Claude

## ¿Qué hace este servicio?
Gestiona todos los temporizadores del juego. Hay un timer por sala activa.
Cada 100 ms publica un tick en `evt:TimerTick` para que el Gateway lo reenvíe al cliente.
Cuando el tiempo de una fase se agota, publica `evt:TimerEnd` para que Game Service
avance la fase. Implementa el patrón Leader Election con Redis para evitar
que múltiples instancias produzcan timers duplicados.

**Regla de oro:** este servicio NO toca lógica de juego ni de sala. Solo maneja tiempo.

## Puerto y protocolo
- **HTTP:** ninguno
- **Protocolo:** Redis Pub/Sub exclusivamente
- Requiere **2 conexiones Redis**: `redis` (comandos) y `redisSub` (suscripciones)

## Scaffolding completo

```
battlecaos-timer/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← punto de entrada: Redis + broker + leader election
    ├── redis.js        ← fábrica createRedis()
    ├── logger.js       ← logger con prefijo [timer]
    └── TimerManager.js ← gestión de timers por sala (Map de intervalos)
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
```

## package.json

```json
{
  "name": "battlecaos-timer",
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
| `evt:RoomReady` | Inicia timer de fase COLOCACION (60 s) |
| `evt:PhaseChanged` | Resetea timer para la nueva fase |
| `evt:GameEnded` | Detiene y limpia el timer de la sala |
| `evt:PlayerDisconnectedFromRoom` | Pausa el timer si aplica |
| `evt:PlayerReconnected` | Reanuda el timer |

### Publica en:
| Canal | Cuándo |
|---|---|
| `evt:TimerEnd` | Cuando el tiempo de una fase llega a 0 |
| `evt:TimerTick` | Cada 100 ms con el tiempo restante |

## Duración de fases

```js
const DURATIONS = {
  COLOCACION:   60_000,  // 60 segundos — todos colocan flota
  TURNO:        30_000,  // 30 segundos por turno individual
  SALVA:         8_000,  // 8 segundos — ventana de disparo simultáneo
  CONTRAMEDIDA:  5_000,  // 5 segundos — ventana de reacción a poder enemigo
};
```

> CONTRAMEDIDA es iniciada por Game Service (no por RoomReady/PhaseChanged).
> Timer Service inicia COLOCACION, TURNO y SALVA. Game Service gestiona CONTRAMEDIDA internamente con `setTimeout`.

## Formato de mensajes

### `evt:TimerTick`
```json
{
  "type": "TimerTick",
  "source": "timer",
  "timestamp": 1234567890,
  "data": {
    "codigo": "123456",
    "fase": "TURNOS",
    "remaining": 18500
  }
}
```

### `evt:TimerEnd`
```json
{
  "type": "TimerEnd",
  "source": "timer",
  "timestamp": 1234567890,
  "data": {
    "codigo": "123456",
    "fase": "TURNOS"
  }
}
```

## TimerManager — estructura interna

```js
// src/TimerManager.js
// Map<codigo, { interval, remaining, fase }>
const timers = new Map();

export function startTimer(redis, codigo, fase, durationMs) { ... }
export function stopTimer(codigo) { ... }
export function pauseTimer(codigo) { ... }
export function resumeTimer(redis, codigo) { ... }
```

Cada timer usa `setInterval(100ms)` para el tick y limpiar con `clearInterval` al parar.

## Leader Election — implementación exacta

```js
const ME = `timer-${process.pid}`;
let isLeader = false;

async function tryBecomeLeader() {
  // SETNX con TTL de 1s — solo el primero adquiere el lease
  const acquired = await redis.set('timer:leader', ME, 'NX', 'EX', 1);
  if (acquired) isLeader = true;
}

// Heartbeat cada 500ms
setInterval(async () => {
  if (isLeader) {
    // Renovar TTL (keepalive)
    await redis.set('timer:leader', ME, 'XX', 'EX', 1);
  } else {
    await tryBecomeLeader();
  }
}, 500);
```

**Failover:** Si el master muere, su TTL de 1s expira. Otra instancia adquiere el lease en el siguiente heartbeat (≤500ms). Tiempo total de failover: **~1.5s**.

Solo el leader ejecuta `startTimer()`. Las instancias en standby solo escuchan eventos y esperan.

## Sistema de logs

```js
// src/logger.js
const SVC = 'timer';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[timer] servicio iniciado
[timer] sala 123456 — timer COLOCACION 60s iniciado
[timer] sala 123456 — tick: 18.5s restantes
[timer] sala 123456 — TIEMPO AGOTADO en fase TURNOS
[timer] sala 123456 — timer detenido (GameEnded)
[timer] WARN: sala 123456 — timer pausado (PlayerDisconnected)
```

## Testing
No tiene lógica de dominio pura. Verificación con redis-cli:

```bash
# Publicar RoomReady para disparar el timer
redis-cli PUBLISH evt:RoomReady '{"type":"RoomReady","data":{"codigo":"123456","modo":"1v1"}}'

# Escuchar ticks (deben llegar cada 100 ms)
redis-cli SUBSCRIBE evt:TimerTick

# Escuchar cuando el tiempo se agota
redis-cli SUBSCRIBE evt:TimerEnd
```

## Limpieza de timers
Al recibir `evt:GameEnded` o `evt:RoomDestroyed`, llamar `stopTimer(codigo)` para
liberar el intervalo y eliminar la entrada del Map. Evitar memory leaks en partidas largas.

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + dos conexiones Redis | 1 - Fase 0 |
| DOMF1303 | Timer de fases: tick 100 ms + TimerEnd + Leader Election | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
