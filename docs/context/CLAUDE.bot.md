# battlecaos-bot — Contexto para Claude

## ¿Qué hace este servicio?
Implementa la IA del oponente para el modo `1v1-Bot`. Observa los eventos del juego
(`evt:ShotFired`, `evt:GameStarted`) para determinar cuándo le toca "disparar",
calcula la celda objetivo usando una estrategia de IA (hunt & target), y publica
`evt:BotDecision` para que Game Service procese el disparo como si fuera un jugador humano.

**Regla de oro:** el bot NO tiene estado propio más allá de lo que lee de Redis.
Game Service es quien valida y aplica el disparo del bot, igual que con humanos.

## Puerto y protocolo
- **HTTP:** ninguno
- **Protocolo:** Redis Pub/Sub exclusivamente
- Requiere **2 conexiones Redis**: `redis` (comandos/publish) y `sub` (suscripciones)

## Scaffolding completo

```
battlecaos-bot/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← suscripción + broker
    ├── redis.js        ← fábrica createRedis()
    ├── logger.js       ← logger con prefijo [bot]
    └── strategy.js     ← lógica de IA: hunt & target
```

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
BOT_DELAY_MS=800
```

`BOT_DELAY_MS`: retraso artificial antes de disparar (simula "pensar").
Valor recomendado: 500–1500 ms para que parezca natural.

## package.json

```json
{
  "name": "battlecaos-bot",
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
| `evt:GameStarted` | Registra qué salas en modo bot deben ser gestionadas |
| `evt:ShotFired` | Si fue turno del humano, el bot calcula y publica su disparo |
| `evt:GameEnded` | Limpia estado interno de la sala |

### Publica en:
| Canal | Cuándo |
|---|---|
| `evt:BotDecision` | Disparo elegido por la IA (Game Service lo consume como disparo normal) |

## Formato de `evt:BotDecision`

```json
{
  "type": "BotDecision",
  "source": "bot",
  "timestamp": 1234567890,
  "data": {
    "codigo": "123456",
    "playerId": "bot-player-id",
    "x": 3,
    "y": 7
  }
}
```

El `playerId` del bot se almacena en la sala cuando se crea en modo `1v1-Bot`.
El bot tiene un ID fijo como `bot-{codigo}` o similar.

## Estrategia de IA: Hunt & Target

```js
// src/strategy.js

// Fase Hunt: disparo aleatorio en celdas no disparadas
function huntMode(shotHistory) {
  const all = generateAllCells(10);
  const available = all.filter(([x, y]) => !shotHistory.has(`${x},${y}`));
  return available[Math.floor(Math.random() * available.length)];
}

// Fase Target: si hay un hit reciente, atacar celdas adyacentes
function targetMode(lastHit, shotHistory) {
  const [hx, hy] = lastHit;
  const adjacent = [
    [hx+1, hy], [hx-1, hy], [hx, hy+1], [hx, hy-1]
  ].filter(([x, y]) =>
    x >= 0 && x < 10 && y >= 0 && y < 10 && !shotHistory.has(`${x},${y}`)
  );
  if (adjacent.length === 0) return huntMode(shotHistory);
  return adjacent[Math.floor(Math.random() * adjacent.length)];
}

export function decideShot(shotHistory, lastHit) {
  return lastHit ? targetMode(lastHit, shotHistory) : huntMode(shotHistory);
}
```

## Retraso artificial

```js
// En el handler de evt:ShotFired, tras determinar que es turno del bot:
const delay = parseInt(process.env.BOT_DELAY_MS ?? '800');
setTimeout(async () => {
  const [x, y] = decideShot(shotHistory, lastHit);
  await redis.publish('evt:BotDecision', JSON.stringify({
    type: 'BotDecision', source: 'bot', timestamp: Date.now(),
    data: { codigo, playerId: botId, x, y },
  }));
}, delay);
```

## Sistema de logs

```js
// src/logger.js
const SVC = 'bot';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[bot] suscrito a evt:GameStarted, evt:ShotFired
[bot] sala 123456 — modo bot activo (playerId: bot-123456)
[bot] sala 123456 — pensando... (800ms)
[bot] sala 123456 — disparo en (3,7): modo hunt
[bot] sala 123456 — disparo en (4,7): modo target (hit anterior en 3,7)
[bot] sala 123456 — partida terminada, limpiando estado
```

## Testing
Verificación manual:

```bash
# Publicar GameStarted con modo bot
redis-cli PUBLISH evt:GameStarted '{"type":"GameStarted","data":{"codigo":"123456","modo":"1v1-bot","jugadores":[{"id":"uid-A","equipo":"A"},{"id":"bot-123456","equipo":"B","esBot":true}]}}'

# Publicar ShotFired del jugador humano (turno pasa al bot)
redis-cli PUBLISH evt:ShotFired '{"type":"ShotFired","data":{"codigo":"123456","playerId":"uid-A","x":0,"y":0,"result":"miss"}}'

# Escuchar decisión del bot (después de BOT_DELAY_MS)
redis-cli SUBSCRIBE evt:BotDecision
```

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| DOMF001 | Inicialización base + dos conexiones Redis | 1 - Fase 0 |
| DOMF1101 | Estrategia Hunt & Target + retraso artificial | 2 - Fase 3 |
| DOMF1102 | Publicar evt:BotDecision en turno correcto | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
