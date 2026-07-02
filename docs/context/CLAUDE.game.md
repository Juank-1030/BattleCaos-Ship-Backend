# battlecaos-game — Contexto para Claude

## ¿Qué hace este servicio?
Motor central del juego. Implementa toda la lógica de dominio: colocación de flota,
disparos por turnos, salva simultánea, poderes especiales, energía atómica, contramedida
y detección de victoria. Mantiene el estado en Redis State y se comunica exclusivamente
por Kafka.

**Regla de oro:** NUNCA toca Socket.io. Todo es Kafka + Redis State.
Las funciones de `domain/` son puras (0 imports externos, 0 Redis, 0 Kafka).

## Puerto y protocolo
- **HTTP:** ninguno
- **Mensajería:** Kafka (consume y produce topics)
- **Estado:** Redis — lectura/escritura directa desde handlers
- **1 conexión Redis** (estado) + **1 cliente Kafka** (producer + consumer)

## Scaffolding completo

```
battlecaos-game/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js                    ← crea redis + kafka, importa broker
    ├── redis.js                    ← fábrica createRedis()
    ├── kafka.js                    ← fábrica producer + createConsumer
    ├── logger.js                   ← logger [game]
    ├── domain/
    │   ├── engine.js               ← FASES, nextFase, validateTurn
    │   ├── board.js                ← createBoard, placeShip, shoot
    │   ├── fleet.js                ← FLEET_CONFIG, validateFleet, isFleetSunk
    │   ├── energy.js               ← energyGain, hasEnough
    │   ├── powers.js               ← POWERS, applyPower, validatePowerCost
    │   ├── salvo.js                ← acquireSalvoLock, releaseSalvoLock
    │   ├── countermeasure.js       ← isCountermeasureActive, applyCountermeasure
    │   ├── errors.js               ← clase DomainError
    │   └── __tests__/
    │       ├── engine.test.js
    │       ├── board.test.js
    │       ├── fleet.test.js
    │       ├── energy.test.js
    │       └── powers.test.js
    └── handlers/
        ├── broker.js               ← consumer Kafka + dispatch a handlers
        ├── handlePlaceShips.js
        ├── handleShot.js
        ├── handleSalvo.js
        ├── handlePower.js
        ├── handleCountermeasure.js
        └── handleDisconnect.js
```

## Variables de entorno

```env
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-game
# KAFKA_USERNAME=   # solo Upstash
# KAFKA_PASSWORD=   # solo Upstash
```

## package.json

```json
{
  "name": "battlecaos-game",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js",
    "test":  "vitest run",
    "coverage": "vitest run --coverage"
  },
  "dependencies": {
    "dotenv":  "^16.4.0",
    "ioredis": "^5.5.0",
    "kafkajs": "^2.2.4"
  },
  "devDependencies": {
    "vitest": "^1.6.0",
    "@vitest/coverage-v8": "^1.6.0"
  }
}
```

## Kafka

### Consume de:
| Topic | Tipos de mensaje que procesa |
|---|---|
| `cmd.game` | `colocacion:set`, `disparo:realizar`, `salva:disparo`, `poder:usar`, `contramedida:activar` |
| `evt.room` | `RoomReady` (inicia COLOCACION), `PlayerDisconnectedFromRoom`, `PlayerReconnected` |
| `evt.timer` | `TimerEnd` (tiempo agotado en fase actual) |
| `evt.bot` | `BotDecision` (disparo del bot en modo 1v1-bot) |

### Produce en:
| Topic | Cuándo |
|---|---|
| `gw.broadcast` | Tras cada acción: `{ roomId: codigo, event: 'game:state', payload }` |
| `evt.game` | `GameStarted`, `ShotFired`, `PhaseChanged`, `PowerUsed`, `PowerCompensated`, `GameEnded`, `ShipsPlaced`, `ShipSunk` |

## Redis State

| Operación | Clave | Para qué |
|---|---|---|
| `GET / SET` | `sala:{codigo}` | Leer/guardar estado completo de partida |
| `INCRBY` | `sala:{codigo}:energia:{equipo}` | Acumular energía atómicamente |
| `SET NX EX` | `sala:{codigo}:salva:{x}:{y}` | Lock atómico para salva simultánea |
| `GET` | `sala:{codigo}:energia:{equipo}` | Leer energía actual |

## Fases de la partida

```
LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN
```

- `LOBBY → COLOCACION`: al recibir `RoomReady` de `evt.room`
- `COLOCACION → TURNOS`: todos colocan flota O `TimerEnd`
- `TURNOS → SALVA`: activación de salva simultánea
- `SALVA → TURNOS`: resolución de salva
- `TURNOS/SALVA → FIN`: `isFleetSunk(board) === true`

## Poderes especiales (5 poderes)

| ID | Coste | Efecto |
|---|:---:|---|
| `bombardeo` | 2 | Área 3×3 centrada en objetivo |
| `sonar` | 2 | Revela barco en fila/columna completa del rival |
| `escudo` | 1 | Protege una celda del siguiente disparo |
| `tormenta` | 3 | Bloquea siguiente turno del rival (1 uso/partida) |
| `contramedida` | 0 | Anula poder ofensivo enemigo — ventana 5s |

## Acumulación de energía

```js
// En handleShot.js, tras disparo:
const gain = energyGain(result.hit, shipSunk); // +1 hit, +4 hundir
if (gain > 0) {
  await redis.incrby(`sala:${codigo}:energia:${jugador.equipo}`, gain);
}
```

`INCRBY` es atómico — múltiples instancias del Game Service no generan race conditions.

## Salva Simultánea — lock SETNX

```js
// acquireSalvoLock: devuelve true si adquirió, false si ya estaba tomada
const acquired = await redis.set(
  `sala:${codigo}:salva:${x}:${y}`, '1', 'NX', 'EX', 10
);
```

## Patrón Domain Kernel

```
  Redis State (ioredis — directo desde handlers)
       ↑ lee/escribe
  HANDLERS (broker.js, handleShot.js, ...)
       ↑ llaman
  DOMAIN (engine, board, fleet, powers... — 0 imports externos)
```

El dominio es puro. Los handlers orquestan: leen Redis, llaman dominio, escriben Redis, publican Kafka.

## `broadcastState(codigo)` — función clave

```js
// handlers/broker.js
export async function broadcastState(codigo) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  await producer.send({
    topic:    'gw.broadcast',
    messages: [{ key: codigo, value: JSON.stringify({
      roomId: codigo, event: 'game:state', payload: sanitizeState(sala),
    })}],
  });
}
```

Llamar al final de CADA handler. Sanitizar: ocultar tableros del equipo contrario.

## Testing (Vitest)

Solo cubrir `src/domain/` — funciones puras, sin mocks de Redis ni Kafka.
Objetivo: **≥ 85% de cobertura** en `src/domain/`.

```bash
npm test           # una vez
npm run coverage   # con reporte HTML
```

## Tareas pendientes

| ID | Descripción |
|---|---|
| DOMF302 | `domain/engine.js` + `domain/errors.js` |
| DOMF501 | `domain/board.js` + `domain/fleet.js` + `handlers/handlePlaceShips.js` |
| DOMF502 | Todos colocados → fase TURNOS |
| DOMF601 | `handlers/handleShot.js` |
| DOMF602 | Hundimiento + victoria |
| DOMF701 | `domain/energy.js` + INCRBY en handleShot |
| DOMF801/802 | `handlers/handleSalvo.js` + SETNX |
| DOMF901 | `domain/powers.js` + `handlers/handlePower.js` |
| DOMF1001/1002 | `domain/countermeasure.js` + ventana 5s |
| DOMF1301 | `handlers/handleDisconnect.js` |
| DOMF003 | Tests Vitest ≥ 85% cobertura |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
