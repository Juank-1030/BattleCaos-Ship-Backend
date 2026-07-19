# battlecaos-bot — Contexto para Claude

## ¿Qué hace este servicio?
Implementa la IA del oponente para el modo `1v1-Bot`. Observa los eventos del juego
en Kafka para determinar cuándo le toca "disparar", calcula la celda objetivo y publica
`BotDecision` en `evt.bot` para que Game Service lo procese igual que un disparo humano.

**Regla de oro:** el bot NO escribe en Redis State. Solo consume Kafka y produce Kafka.
Game Service es quien valida y aplica el disparo del bot.

## Puerto y protocolo
- **HTTP:** ninguno
- **Mensajería:** Kafka (consume `evt.game`, produce `evt.bot`)
- **Estado:** ninguno — stateless
- **1 cliente Kafka** (producer + consumer)

## Scaffolding completo

```
battlecaos-bot/
├── .env
├── .env.example
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js        ← consumer Kafka + lógica de decisión
    ├── kafka.js        ← fábrica producer + createConsumer
    ├── logger.js       ← logger [bot]
    └── strategy.js     ← lógica de IA: hunt & target
```

## Variables de entorno

```env
# Kafka
KAFKA_BROKER=localhost:9092
KAFKA_CLIENT_ID=battlecaos-bot
# KAFKA_USERNAME=
# KAFKA_PASSWORD=
BOT_DELAY_MS=800
```

`BOT_DELAY_MS`: retraso artificial antes de disparar. Valor recomendado: 500–1500ms.

## Kafka

### Consume de:
| Topic | Tipo | Acción |
|---|---|---|
| `evt.game` | `ShotFired` | Si modo bot y turno del bot, decidir y disparar |
| `evt.game` | `GameEnded` | Limpiar estado interno de la sala |

### Produce en:
| Topic | Tipo | Cuándo |
|---|---|---|
| `evt.bot` | `BotDecision` | Disparo elegido (Game Service lo consume como disparo normal) |

## Formato de `BotDecision`

```json
{
  "type": "BotDecision",
  "source": "bot",
  "timestamp": 1234567890,
  "data": { "codigo": "123456", "playerId": "bot-player-id", "x": 3, "y": 7 }
}
```

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
const delay = parseInt(process.env.BOT_DELAY_MS ?? '800');
setTimeout(async () => {
  const [x, y] = decideShot(shotHistory, lastHit);
  await producer.send({ topic: 'evt.bot', messages: [{ key: codigo, value: JSON.stringify({
    type: 'BotDecision', source: 'bot', timestamp: Date.now(),
    data: { codigo, playerId: botId, x, y },
  })}]});
}, delay);
```

## Tareas pendientes

| ID | Descripción |
|---|---|
| DOMF001 | Init + Kafka |
| DOMF1101 | Consumer evt.game (ShotFired) + estrategia Hunt & Target |
| DOMF1102 | Produce BotDecision en evt.bot |

## Arranque

```bash
cp .env.example .env
npm install
npm run dev
```
