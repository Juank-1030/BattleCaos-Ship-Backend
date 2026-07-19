# battlecaos-timer — Contexto para Claude

## ¿Qué hace este servicio?
Gestiona todos los temporizadores del juego. Consume eventos de Kafka para iniciar/detener
timers por sala. Cada 100ms publica un tick en `evt.timer`. Cuando el tiempo se agota,
publica `TimerEnd` para que Game Service avance la fase. Implementa Leader Election
con Redis State para evitar timers duplicados si hay múltiples instancias.

**Regla de oro:** este servicio NO toca lógica de juego ni de sala. Solo maneja tiempo.

## Puerto y protocolo
- **HTTP:** ninguno
- **Mensajería:** Kafka (consume `evt.room` + `evt.game`, produce `evt.timer`)
- **Estado:** Redis — solo `timer:leader` para leader election
- **1 conexión Redis** (estado) + **1 cliente Kafka** (producer + consumer)

## Scaffolding completo

```
battlecaos-timer/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← init Redis + Kafka + leader election + broker
    ├── redis.js        ← fábrica createRedis()
    ├── kafka.js        ← fábrica producer + createConsumer
    ├── logger.js       ← logger [timer]
    └── TimerManager.js ← Map<codigo, interval> + start/stop/pause/resume
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-timer
# KAFKA_USERNAME=
# KAFKA_PASSWORD=
```

## Kafka

### Consume de:
| Topic | Tipo de mensaje | Acción |
|---|---|---|
| `evt.room` | `RoomReady` | Inicia timer COLOCACION (60s) |
| `evt.game` | `PhaseChanged` | Resetea timer para nueva fase |
| `evt.game` | `TurnStarted` | Reinicia el timer TURNO (30s) para el nuevo jugador activo — **agregado 2026-07-05**: un disparo normal rota de jugador sin cambiar de fase, así que `PhaseChanged` no alcanza para reiniciar el reloj cada turno (ver DOMF1301 en `BattleCaos-Ship_Inception.md`, que exige el límite "por turno", no uno solo para toda la fase) |
| `evt.game` | `GameEnded` | Detiene y limpia timer de la sala |
| `evt.room` | `PlayerDisconnectedFromRoom` | Pausa timer si aplica |
| `evt.room` | `PlayerReconnected` | Reanuda timer |

### Produce en:
| Topic | Tipo | Cuándo |
|---|---|---|
| `evt.timer` | `TimerTick` | Cada 100ms con tiempo restante |
| `evt.timer` | `TimerEnd` | Cuando el tiempo de una fase llega a 0 |

## Redis State

| Operación | Clave | Para qué |
|---|---|---|
| `SET NX EX 1` | `timer:leader` | Adquirir lease de leader |
| `SET XX EX 1` | `timer:leader` | Renovar lease (heartbeat 500ms) |

## Duración de fases

```js
const DURATIONS = {
  COLOCACION:   60_000,
  TURNO:        30_000,
  SALVA:         8_000,
  // CONTRAMEDIDA: 5_000  ← gestionada con setTimeout en Game Service
};
```

## Leader Election

```js
const ME = `timer-${process.pid}`;
let isLeader = false;

async function tryBecomeLeader() {
  const acquired = await redis.set('timer:leader', ME, 'NX', 'EX', 1);
  if (acquired) isLeader = true;
}

setInterval(async () => {
  if (isLeader) await redis.set('timer:leader', ME, 'XX', 'EX', 1);
  else          await tryBecomeLeader();
}, 500);
```

Solo el leader ejecuta `startTimer()`. Failover en ~1.5s si el master muere.

## Formato de mensajes producidos

> **Corrección (2026-07-05):** el campo es `tipo`, no `fase` — `battlecaos-game` (`broker.js`,
> `handleTimerEnd`) ya lee `msg.data.tipo` (valores `'COLOCACION' | 'TURNO' | 'SALVA'`, distintos de
> `sala.fase` que usa `'TURNOS'` en plural). Se implementó así para que coincida con el código
> existente de `battlecaos-game`, que es quien consume estos eventos.

```json
{ "type": "TimerTick", "source": "timer", "timestamp": 1234,
  "data": { "codigo": "123456", "tipo": "TURNO", "remaining": 18500 } }

{ "type": "TimerEnd", "source": "timer", "timestamp": 1234,
  "data": { "codigo": "123456", "tipo": "TURNO" } }
```

## Tareas pendientes

| ID | Descripción | Estado |
|---|---|---|
| DOMF001 | Init + Redis + Kafka | ✅ Completado (2026-07-05) |
| DOMF1303 | Timer de fases: TimerTick + TimerEnd + Leader Election | ✅ Completado (2026-07-05) |

Verificado en vivo con Redis + Kafka reales, junto con `battlecaos-game` corriendo simultáneamente:
timer de Salva (8s) con ticks decrecientes y `TimerEnd` correcto, pausa/reanudación por
desconexión/reconexión, y ambos servicios reaccionando sin conflicto al mismo `PhaseChanged`
(idempotencia confirmada). 11 tests unitarios en `TimerManager.js` (97.87% cobertura).

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
