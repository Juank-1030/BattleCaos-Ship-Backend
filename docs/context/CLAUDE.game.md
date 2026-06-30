# battlecaos-game — Contexto para Claude

## ¿Qué hace este servicio?
Motor central del juego. Implementa toda la lógica de dominio: colocación de flota,
disparos por turnos, salva simultánea, poderes especiales, energía atómica, contramedida
y detección de victoria. Mantiene el estado de cada partida en `sala:{codigo}` y publica
el estado actualizado en `gw:broadcast` tras cada acción.

**Regla de oro:** este servicio NUNCA toca Socket.io. Toda comunicación es Redis.
Las funciones de `domain/` son puras (sin imports externos, sin Redis).

## Puerto y protocolo
- **HTTP:** ninguno
- **Protocolo:** Redis Pub/Sub exclusivamente
- Requiere **2 conexiones Redis** obligatoriamente:
  - `redis` — para GET/SET/PUBLISH (comandos normales)
  - `redisSub` — para SUBSCRIBE (no puede mezclar roles en ioredis)

## Scaffolding completo

```
battlecaos-game/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js                    ← punto de entrada: crea redis + redisSub, importa broker
    ├── redis.js                    ← fábrica createRedis()
    ├── logger.js                   ← logger con prefijo [game]
    ├── domain/
    │   ├── engine.js               ← fases, validateTurn, processShot
    │   ├── board.js                ← createBoard, placeShip, shoot
    │   ├── fleet.js                ← FLEET_CONFIG, validateFleet, isFleetSunk
    │   ├── energy.js               ← energyGain, hasEnough
    │   ├── powers.js               ← POWERS, applyPower, validatePowerCost
    │   ├── salvo.js                ← acquireSalvoLock, releaseSalvoLock
    │   ├── countermeasure.js       ← isCountermeasureActive, applyCountermeasure
    │   ├── player.js               ← utilidades de jugador/equipo
    │   ├── errors.js               ← clase DomainError
    │   └── __tests__/
    │       ├── engine.test.js
    │       ├── board.test.js
    │       ├── fleet.test.js
    │       ├── energy.test.js
    │       └── powers.test.js
    └── handlers/
        ├── broker.js               ← suscripciones Redis + dispatch a handlers
        ├── handlePlaceShips.js     ← DOMF501: colocación de flota
        ├── handleShot.js           ← DOMF601: disparos por turno
        ├── handleSalvo.js          ← DOMF801: salva simultánea
        ├── handlePower.js          ← DOMF901: poderes especiales
        ├── handleDisconnect.js     ← DOMF1301: desconexión en partida
        └── handleCountermeasure.js ← DOMF1001: contramedida 5 s
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
```

## package.json

```json
{
  "name": "battlecaos-game",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "node --watch src/index.js",
    "test":  "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "dotenv":   "^16.4.0",
    "ioredis":  "^5.5.0"
  },
  "devDependencies": {
    "vitest": "^1.6.0"
  }
}
```

## Canales Redis

### Suscribe a:
| Canal | Tipos de mensaje |
|---|---|
| `svc:game` | `colocacion:set`, `disparo:realizar`, `salva:disparo`, `poder:usar`, `contramedida:activar` |
| `evt:RoomReady` | Inicia fase COLOCACION |
| `evt:TimerEnd` | Tiempo agotado en fase actual |
| `evt:PlayerDisconnectedFromRoom` | Manejar desconexión en partida |
| `evt:PlayerReconnected` | Restaurar estado al reconectar |
| `evt:BotDecision` | Procesar disparo del bot |

### Publica en:
| Canal | Cuándo |
|---|---|
| `gw:broadcast` | Tras cada acción: `{ roomId: codigo, event: 'game:state', payload }` |
| `evt:GameStarted` | Al pasar a fase COLOCACION |
| `evt:ShotFired` | Cada disparo procesado |
| `evt:PhaseChanged` | Cambio de fase |
| `evt:PowerUsed` | Poder activado |
| `evt:PowerCompensated` | Contramedida exitosa |
| `evt:GameEnded` | Victoria detectada |
| `evt:ShipsPlaced` | Flota colocada por un jugador |
| `evt:ShipSunk` | Barco hundido |

## Fases de la partida

```
LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN
```

Transiciones:
- `LOBBY → COLOCACION`: al recibir `evt:RoomReady`
- `COLOCACION → TURNOS`: cuando todos los jugadores colocan su flota (o `evt:TimerEnd`)
- `TURNOS → SALVA`: cuando se activa salva simultánea
- `SALVA → TURNOS`: cuando se resuelve la salva
- `TURNOS/SALVA → FIN`: cuando `isFleetSunk(board)` es true

## Poderes especiales (5 poderes)

| ID | Nombre | Coste energía | Efecto |
|---|---|:---:|---|
| `bombardeo` | Bombardeo 3×3 | 2 | Impacta área 3×3 centrada en objetivo |
| `sonar` | Sonar | 2 | Revela si hay barco en una fila o columna completa del rival |
| `escudo` | Escudo | 1 | Protege una celda propia del siguiente disparo enemigo |
| `tormenta` | Tormenta | 3 | Bloquea el siguiente turno del rival (uso único por partida) |
| `contramedida` | Contramedida | 0 | Anula un poder ofensivo enemigo — solo usable en ventana de 5s |

> **Poderes contrables** (`bombardeo`, `sonar`): al activarse, se abre ventana de 5s para contramedida del rival.
> **Tormenta**: solo 1 uso por jugador por partida. Se valida `sala.tormentaUsada[playerId]`.
> **Escudo**: se almacena en `sala.escudos["{playerId}:{x},{y}"] = true`.

## Acumulación de energía
- +1 por impacto (`hit`)
- +3 adicional por hundimiento (`sunk`) → total +4 en ese disparo
- Usar `INCRBY` atómico de Redis: `redis.incrby('sala:{codigo}:energia:{equipo}', gain)`
- Costos de poderes: bombardeo=2, sonar=2, escudo=1, tormenta=3, contramedida=0
- Descontar con `INCRBY -cost` (valor negativo)

## Salva Simultánea — mecánica de concurrencia
Ventana de **8 segundos** donde todos disparan al mismo tiempo.
Usar `SETNX` como lock por celda:
- Clave: `sala:{codigo}:salva:{x}:{y}`
- Solo el primero en llegar adquiere el lock (adquirido = 1, rechazado = 0)
- El segundo recibe `{ error: 'celda_ya_tomada' }`
- TTL de 10s en el lock para limpiar si el jugador abandona
- Si todos dispararon antes de los 8s, resolver inmediatamente sin esperar `evt:TimerEnd`

## Contramedida (ventana de reacción)
Al activar `bombardeo` o `sonar`, abrir ventana de **5 segundos** para que el rival reaccione:
1. Guardar en sala: `sala.contramedidaActiva = { powerType, atacanteId, targetEquipo, expiraEn: Date.now()+5000 }`
2. Publicar `gw:broadcast` con `event: 'poder:contramedida-disponible'` al equipo rival
3. Iniciar `setTimeout(5000)`: si no hubo reacción, aplicar el poder y limpiar `contramedidaActiva`
4. Si el rival activa contramedida dentro de los 5s:
   - Anular el poder (no aplicar efecto)
   - El atacante **pierde** la energía gastada (no se devuelve)
   - Publicar `evt:PowerCompensated`

> Nota: la Inception dice "devolver energía al atacante" pero el Orden_de_Ejecucion dice "atacante pierde la energía". Usar la versión de Orden_de_Ejecucion (sin devolución).

## Patrón Domain Kernel — arquitectura central

```
  REDIS (ioredis — llamado directo desde handlers)
       ↑ lee/escribe
  HANDLERS (handleShot, handlePower, handleSalvo...)
       ↑ llaman
  DOMAIN (engine, board, powers, salvo — funciones puras, 0 imports externos)
```

**El dominio es puro.** `engine.js`, `board.js`, etc. nunca importan `ioredis`, `socket.io` ni nada del `package.json`.
**Los handlers orquestan:** leen de Redis, llaman al dominio, escriben en Redis, publican eventos.
No hay capas intermedias (puertos, adaptadores, use cases) — el acoplamiento handler-Redis es deliberado.

## Reglas de dominio (funciones puras en `domain/`)
- Cero imports externos — sin Redis, sin logger
- Solo reciben datos, devuelven datos o lanzan `DomainError`
- Son completamente testeables con Vitest sin mocks
- Extensibilidad: nuevo poder = 1 archivo en `domain/` + 1 archivo en `handlers/` + 1 línea en `broker.js`

## Sistema de logs

```js
// src/logger.js
const SVC = 'game';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[game] servicio iniciado
[game] sala 123456 — fase COLOCACION iniciada
[game] sala 123456 — jugador uid-A coloca flota OK
[game] sala 123456 — disparo (3,7): hit — barco crucero
[game] sala 123456 — FIN equipo A gana
[game] WARN: no_es_tu_turno — playerId: uid-B
[game] ERROR: handler exception — handleShot — RangeError
```

## Testing (Vitest)

```bash
npm test          # correr una vez
npm run test:watch  # modo watch
```

Los tests solo cubren `src/domain/` — funciones puras, sin mocks de Redis.

```js
// Ejemplo: src/domain/__tests__/board.test.js
import { describe, it, expect } from 'vitest';
import { createBoard, placeShip, shoot } from '../board.js';

describe('board', () => {
  it('dispara y devuelve miss en celda vacía', () => {
    const board = createBoard();
    const result = shoot(board, 0, 0);
    expect(result).toEqual({ valid: true, hit: false });
  });

  it('rechaza disparo en celda ya disparada', () => {
    const board = createBoard();
    shoot(board, 0, 0);
    expect(shoot(board, 0, 0).valid).toBe(false);
  });
});
```

Cobertura objetivo: **≥ 85%** en `src/domain/`

## `broadcastState(codigo)` — función clave
Llamar al final de CADA handler. Lee `sala:{codigo}`, sanitiza (oculta tableros enemigos)
y publica en `gw:broadcast` con `event: 'game:state'`.

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + dos conexiones Redis | 1 - Fase 0 |
| DOMF301 | broadcastState — publicar estado en gw:broadcast | 1 - Fase 2 |
| DOMF302 | domain/engine.js — fases, validateTurn + domain/errors.js | 1 - Fase 2 |
| DOMF501 | handlePlaceShips + domain/board.js + domain/fleet.js | 1 - Fase 2 |
| DOMF502 | Coordinación de colocación en equipo + avance a TURNOS | 1 - Fase 2 |
| DOMF601 | handleShot — validación turno + disparo + rotar turno | 1 - Fase 2 |
| DOMF602 | Hundimiento + condición de victoria | 1 - Fase 2 |
| DOMF701 | domain/energy.js + INCRBY en handleShot | 1 - Fase 2 |
| DOMF801 | handleSalvo — lock SETNX salva simultánea | 2 - Fase 3 |
| DOMF802 | Resolución de salva y broadcast | 2 - Fase 3 |
| DOMF901 | domain/powers.js + handlePower | 2 - Fase 3 |
| DOMF1001 | domain/countermeasure.js + handleCountermeasure | 2 - Fase 3 |
| DOMF1002 | Ventana 5 s + evt:PowerCompensated | 2 - Fase 3 |
| DOMF1301 | handleDisconnect — pausar turno en partida | 2 - Fase 3 |
| DOMF003 | Suite tests Vitest ≥ 85% en domain/ | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
