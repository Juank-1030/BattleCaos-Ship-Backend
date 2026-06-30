# Orden de Ejecución — BattleCaos-Ship

Orden basado en dependencias entre historias de usuario. Cada fase requiere que la anterior esté completada.

**Total: 33 historias de usuario | 51 tareas técnicas**

---

## Repositorios en la Organización GitHub

Un repositorio por microservicio. Cada repositorio es un proceso Node.js independiente con su propio `package.json`, `.env` y punto de entrada.

### Organización: `BattleCaos-Ship` (ejemplo)

| Repositorio | Tipo | Puerto | Tareas que lo construyen |
|---|---|:---:|---|
| `battlecaos-gateway` | Node.js + Socket.io | 3000 | DOMF001, DOMF002, DOMF102, DOMF1302 |
| `battlecaos-auth` | Node.js + Express | 3001 | DOMF001, DOMF101, DOMF1302 |
| `battlecaos-room` | Node.js | — | DOMF001, DOMF201, DOMF202†, DOMF203, DOMF402, DOMF403, DOMF1302 |
| `battlecaos-game` | Node.js | — | DOMF001, DOMF202†, DOMF301†, DOMF302, DOMF401†, DOMF501, DOMF502, DOMF601, DOMF602, DOMF701, DOMF801, DOMF802, DOMF901, DOMF1001, DOMF1002, DOMF1301, DOMF003 |
| `battlecaos-timer` | Node.js | — | DOMF001, DOMF1303 |
| `battlecaos-chat` | Node.js | — | DOMF001, DOMF1103, DOMF1302 |
| `battlecaos-bot` | Node.js | — | DOMF001, DOMF1101, DOMF1102 |
| `battlecaos-observability` | Node.js | — | DOMF001, DOMF1201, DOMF1202, DOMF1203 |
| `battlecaos-frontend` | React + Vite | 5173 | UIUXF001, UIUXF002, UIUXF101, UIUXF201, UIUXF301, UIUXF401, UIUXF501, UIUXF601, UIUXF701, UIUXF801, UIUXF901, UIUXF1001, UIUXF1101, UIUXF1201 |

> **† Tareas compartidas entre dos repositorios:** estas tareas establecen un contrato o patrón que dos servicios deben respetar. El desarrollo ocurre en paralelo en ambos repositorios.

---

### Aclaración de tareas que involucran más de un repositorio

| Tarea | Repositorios involucrados | Qué hace cada uno |
|---|---|---|
| **DOMF001** | Todos (×8) | Cada repo implementa su propio `init.js` con la misma estructura base: cargar `.env`, conectar a Redis, exponer `/health` si tiene HTTP. No es código compartido — es el mismo patrón repetido en cada repo. |
| **DOMF202** | `battlecaos-room` + `battlecaos-game` | Room define el nombre de las claves `sala:{codigo}`. Game sigue esa misma convención de naming. Es una restricción de diseño documentada, no un archivo compartido. |
| **DOMF301** | `battlecaos-game` + `battlecaos-gateway` | Game publica en `gw:broadcast`. Gateway se suscribe y hace el `io.to(roomId).emit()`. Cada repo solo implementa su mitad. |
| **DOMF401** | `battlecaos-gateway` + `battlecaos-game` | Gateway detecta la reconexión del socket y publica `evt:PlayerReconnected`. Game consume ese evento y restaura el estado del jugador. Cada repo solo implementa su mitad. |
| **DOMF1302** | `battlecaos-gateway`, `battlecaos-auth`, `battlecaos-room`, `battlecaos-chat`, `battlecaos-bot` | Cada uno de estos repos agrega la versión mejorada del `GET /health` que verifica el ping a Redis. |

---

### Estructura base de cada repositorio

Todos los repositorios de microservicio siguen la misma estructura mínima:

```
battlecaos-{servicio}/
├── .env                  ← variables de entorno (NO commitear)
├── .env.example          ← plantilla pública sin valores reales
├── .gitignore
├── package.json
├── src/
│   ├── index.js          ← punto de entrada: carga .env, conecta Redis, arranca
│   └── redis.js          ← copia local de la fábrica de conexión Redis
└── README.md
```

Cada servicio agrega sus propias carpetas según su responsabilidad:

```
battlecaos-game/src/
├── index.js
├── redis.js
├── domain/               ← funciones puras (DOMF302, DOMF501, DOMF601, etc.)
└── handlers/             ← orquestadores Redis + dominio (broker.js, handleShot.js, etc.)

battlecaos-room/src/
├── index.js
└── redis.js

battlecaos-timer/src/
├── index.js
└── redis.js

battlecaos-gateway/src/
├── index.js
└── redis.js
```

---

### Código compartido — estrategia para MVP académico

`redis.js` (fábrica de conexión) es el único archivo que se repite en todos los servicios. Para el MVP académico se **copia** en cada repositorio. No se crea un paquete npm privado (innecesario a esta escala).

```js
// src/redis.js — mismo contenido en todos los repositorios
import Redis from 'ioredis';

export function createRedis() {
  const client = new Redis(process.env.REDIS_URL, {
    lazyConnect: false,
    maxRetriesPerRequest: 3,
  });
  client.on('error', (err) => console.error('[redis] error:', err.message));
  return client;
}
```

---

### Variables de entorno por repositorio

Cada repositorio tiene su propio `.env`. Solo incluye las variables que ese servicio necesita.

| Variable | Gateway | Auth | Room | Game | Timer | Chat | Bot | Obs |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `REDIS_URL` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `GATEWAY_PORT` | ✓ | | | | | | | |
| `AUTH_PORT` | | ✓ | | | | | | |
| `ROOM_PORT` | | | ✓ | | | | | |
| `CHAT_PORT` | | | | | | ✓ | | |
| `BOT_PORT` | | | | | | | ✓ | |
| `JWT_SECRET` | ✓ | ✓ | | | | | | |
| `GOOGLE_CLIENT_ID` | | ✓ | | | | | | |
| `CLIENT_ORIGIN` | ✓ | ✓ | | | | | | |

---

### `package.json` base para microservicios de backend

```json
{
  "name": "battlecaos-{servicio}",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js"
  },
  "dependencies": {
    "ioredis": "^5.5.0",
    "dotenv": "^16.4.0"
  }
}
```

Los servicios con HTTP añaden: `express`, `helmet`, `cors`.
El Gateway añade: `socket.io`, `jsonwebtoken`.
Auth añade: `jsonwebtoken`, `google-auth-library`.
Game añade (para tests): `vitest`.

---

## Estructura de carpetas del proyecto completo

```
BattleCaos-Ship-Backend/
├── .env
├── package.json
└── server/
    ├── shared/
    │   └── redis.js                ← fábrica de conexión Redis (usada por todos)
    ├── gateway/
    │   └── index.js
    ├── auth/
    │   └── index.js
    ├── room/
    │   └── index.js
    ├── game/
    │   ├── domain/
    │   │   ├── engine.js
    │   │   ├── board.js
    │   │   ├── fleet.js
    │   │   ├── powers.js
    │   │   ├── energy.js
    │   │   ├── salvo.js
    │   │   ├── countermeasure.js
    │   │   ├── player.js
    │   │   └── errors.js
    │   ├── handlers/
    │   │   ├── handleShot.js
    │   │   ├── handlePower.js
    │   │   ├── handleSalvo.js
    │   │   ├── handlePlaceShips.js
    │   │   ├── handleDisconnect.js
    │   │   └── broker.js
    │   ├── redis.js
    │   └── index.js
    ├── timer/
    │   └── index.js
    ├── chat/
    │   └── index.js
    ├── bot/
    │   └── index.js
    └── observability/
        └── index.js
```

> **MVP:** Todos los servicios se ejecutan en el mismo proceso Node.js, cada uno importado desde `server/gateway/index.js`. En despliegue post-MVP, cada carpeta se convierte en un servicio independiente en Render.

---

## Variables de entorno (`.env` en raíz)

```env
# Redis
REDIS_URL=redis://localhost:6379

# Puertos (solo para servicios con HTTP)
GATEWAY_PORT=3000
AUTH_PORT=3001
ROOM_PORT=3002
CHAT_PORT=3004
BOT_PORT=3005

# Auth
GOOGLE_CLIENT_ID=tu-client-id.apps.googleusercontent.com
JWT_SECRET=cambiar_en_produccion_por_string_aleatorio_largo

# CORS
CLIENT_ORIGIN=http://localhost:5173
```

---

## Canales Redis Pub/Sub (referencia global)

| Canal | Dirección | Descripción |
|---|---|---|
| `svc:room` | Gateway → Room | Eventos `room:create`, `room:join` |
| `svc:game` | Gateway → Game | Disparos, poderes, colocación, salva, contramedida |
| `svc:chat` | Gateway → Chat | Mensajes de chat |
| `gw:broadcast` | Servicios → Gateway | Respuestas para reenviar a clientes |
| `evt:RoomReady` | Room → Game, Timer | Sala llena, comenzar partida |
| `evt:PlayerDisconnectedFromRoom` | Room → Game, Timer, Chat, Obs | Desconexión interpretada |
| `evt:PlayerReconnected` | Room → Game, Timer, Chat, Obs | Reconexión |
| `evt:RoomDestroyed` | Room → Obs | Sala eliminada |
| `evt:PlayerRoomJoined` | Room → Chat | Nuevo jugador, cargar historial |
| `evt:GameStarted` | Game → Timer, Obs | Inicio de COLOCACION |
| `evt:ShotFired` | Game → Obs, Bot | Disparo procesado |
| `evt:PhaseChanged` | Game → Timer, Gateway, Obs | Cambio de fase |
| `evt:PowerUsed` | Game → Obs | Poder activado |
| `evt:PowerCompensated` | Game → Obs | Contramedida exitosa |
| `evt:GameEnded` | Game → Room, Obs | Victoria |
| `evt:ShipsPlaced` | Game → Obs | Flota colocada |
| `evt:ShipSunk` | Game → Obs | Barco hundido |
| `evt:TimerEnd` | Timer → Game | Tiempo agotado |
| `evt:TimerTick` | Timer → Gateway | Tick 100ms para UI |
| `evt:BotDecision` | Bot → Game | Disparo del bot |
| `evt:ChatMessage` | Chat → Obs | Mensaje enviado |

---

## Claves Redis de estado (referencia global)

| Clave | Tipo | Contenido |
|---|---|---|
| `sala:{codigo}` | String (JSON) | Estado completo de la sala/partida |
| `sala:{codigo}:chat` | List | Últimos 100 mensajes |
| `sala:{codigo}:energia:{equipo}` | String (int) | Energía acumulada por equipo |
| `sala:{codigo}:salva:{x}:{y}` | String | Lock SETNX para salva simultánea |
| `timer:leader` | String | Heartbeat del Timer Service activo |
| `stats:kpi:{YYYY-MM-DD}` | Hash | KPIs diarios |
| `stats:games:started` | String (int) | Contador total partidas iniciadas |
| `stats:games:ended` | String (int) | Contador total partidas completadas |

---

## SPRINT 1 — Cimientos, Conexión y Core

---

### Fase 0: Fundación

---

#### Historia 1 — DOMH01: Estado centralizado como fuente única de verdad

> Como desarrollador, quiero que el sistema mantenga un estado centralizado y autoritativo de cada partida, para garantizar que ningún cliente pueda alterar el resultado del juego.

---

##### Tarea: DOMF001 — Inicialización de cada servicio y conexión al almacenamiento de estado compartido

**Microservicio:** Todos los repositorios (`battlecaos-gateway`, `battlecaos-auth`, `battlecaos-room`, `battlecaos-game`, `battlecaos-timer`, `battlecaos-chat`, `battlecaos-bot`, `battlecaos-observability`)

> Esta tarea se ejecuta **una vez por repositorio**. El código es idéntico en los 8 servicios — cada uno crea su propio `src/redis.js` y `src/index.js` con el patrón base.

**Qué hace:** Configura el punto de entrada de cada servicio, carga la configuración desde `.env` y establece la conexión a Redis. Expone `GET /health` en los servicios con HTTP.

**Scaffolding a crear:**

```
server/
├── shared/
│   └── redis.js
├── gateway/index.js
├── auth/index.js
├── room/index.js
├── game/index.js
├── game/redis.js
├── timer/index.js
├── chat/index.js
├── bot/index.js
└── observability/index.js
```

**Cómo implementarlo:**

**Paso 1 — `server/shared/redis.js` (fábrica compartida):**
```js
import Redis from 'ioredis';

export function createRedis() {
  const client = new Redis(process.env.REDIS_URL, {
    lazyConnect: false,
    maxRetriesPerRequest: 3,
  });
  client.on('error', (err) => console.error('[redis] error:', err.message));
  client.on('connect', () => console.log('[redis] conectado'));
  return client;
}
```

**Paso 2 — Patrón base para servicios con HTTP (Gateway, Auth, Room, Chat, Bot):**
```js
// Ejemplo: server/auth/index.js
import 'dotenv/config';
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { createRedis } from '../shared/redis.js';

const app = express();
const redis = createRedis();

app.use(helmet());
app.use(cors({ origin: process.env.CLIENT_ORIGIN }));
app.use(express.json());

app.get('/health', async (_req, res) => {
  try {
    await redis.ping();
    res.json({ service: 'auth', status: 'ok', timestamp: Date.now() });
  } catch {
    res.status(503).json({ service: 'auth', status: 'error' });
  }
});

const PORT = process.env.AUTH_PORT ?? 3001;
app.listen(PORT, () => console.log(`[auth] :${PORT}`));

export { redis };
```

**Paso 3 — Patrón base para servicios sin HTTP (Game, Timer, Observability):**
```js
// Ejemplo: server/game/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

export const redis = createRedis();
export const redisSub = createRedis(); // conexión separada para suscripciones

console.log('[game] servicio iniciado');
// Los handlers y broker se importan aquí cuando estén listos
```

> **Por qué dos conexiones en Game/Timer/Obs:** `ioredis` en modo suscripción (`subscribe`) no puede usarse simultáneamente para comandos normales (`get`, `set`). Se necesita una conexión para publicar/leer y otra para suscribirse.

**Dependencias npm:** ya están en `package.json` — `ioredis`, `express`, `helmet`, `cors`, `dotenv`

**Variables de entorno necesarias:** `REDIS_URL`, `AUTH_PORT`, `ROOM_PORT`, `CHAT_PORT`, `BOT_PORT`, `CLIENT_ORIGIN`

**Verificación:**
```bash
node server/auth/index.js
curl http://localhost:3001/health
# → { "service": "auth", "status": "ok" }
```

---

##### Tarea: DOMF002 — Enrutamiento de eventos del Gateway y control de tráfico

**Microservicio:** `battlecaos-gateway` → `src/index.js`

**Qué hace:** Implementa la tabla de enrutamiento que mapea eventos Socket.io entrantes a canales Redis Pub/Sub y reenvía las respuestas de los servicios como broadcasts a los clientes.

**Scaffolding a crear/modificar:**
```
server/gateway/index.js   ← archivo principal, contiene todo DOMF001 + DOMF002
```

**Cómo implementarlo:**

**Paso 1 — Tabla de enrutamiento (Socket.io → Redis):**
```js
// server/gateway/index.js
import 'dotenv/config';
import { createServer } from 'http';
import express from 'express';
import { Server } from 'socket.io';
import helmet from 'helmet';
import cors from 'cors';
import { createRedis } from '../shared/redis.js';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: process.env.CLIENT_ORIGIN, methods: ['GET', 'POST'] },
});

const pub = createRedis();   // para publicar a servicios
const sub = createRedis();   // para recibir resultados de servicios

// Tabla de enrutamiento: evento Socket.io → canal Redis destino
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

**Paso 2 — Middleware de autenticación y rate limiting:**
```js
// Rate limiting simple con Redis (ventana deslizante de 1 segundo)
async function rateLimitMiddleware(socket, next) {
  const key = `ratelimit:${socket.handshake.address}`;
  const count = await pub.incr(key);
  if (count === 1) await pub.expire(key, 1); // ventana de 1s
  if (count > 20) return next(new Error('rate_limit_exceeded'));
  next();
}

io.use(rateLimitMiddleware);
```

**Paso 3 — Recibir eventos de clientes y enrutar a Redis:**
```js
io.on('connection', (socket) => {
  console.log(`[gateway] cliente conectado: ${socket.id}`);

  // Registrar handlers para cada evento enrutable
  for (const [event, channel] of Object.entries(ROUTES)) {
    socket.on(event, (data) => {
      const payload = JSON.stringify({
        type: event,
        source: 'gateway',
        timestamp: Date.now(),
        version: 1,
        correlationId: data.correlationId ?? crypto.randomUUID(),
        data: { ...data, socketId: socket.id },
      });
      pub.publish(channel, payload);
    });
  }

  socket.on('disconnect', () => {
    // Publicar desconexión para que Room Service la interprete
    pub.publish('svc:room', JSON.stringify({
      type: 'PlayerDisconnected',
      source: 'gateway',
      timestamp: Date.now(),
      data: { socketId: socket.id },
    }));
  });
});
```

**Paso 4 — Suscribirse a resultados y hacer broadcast a clientes:**
```js
// Canal de broadcast: los servicios publican aquí con { roomId, event, payload }
sub.subscribe('gw:broadcast');
sub.on('message', (_channel, raw) => {
  try {
    const { roomId, event, payload } = JSON.parse(raw);
    if (roomId) {
      io.to(roomId).emit(event, payload);
    }
  } catch (err) {
    console.error('[gateway] error en broadcast:', err.message);
  }
});

// TimerTick se envía a todos (no solo a una sala)
sub.subscribe('evt:TimerTick');
sub.on('message', (channel, raw) => {
  if (channel === 'evt:TimerTick') {
    const { data } = JSON.parse(raw);
    io.to(data.codigo).emit('timer:tick', data);
  }
});

app.use(helmet());
app.use(cors({ origin: process.env.CLIENT_ORIGIN }));

app.get('/health', (_req, res) =>
  res.json({ service: 'gateway', status: 'ok', timestamp: Date.now() })
);

httpServer.listen(process.env.GATEWAY_PORT ?? 3000, () =>
  console.log(`[gateway] :${process.env.GATEWAY_PORT ?? 3000}`)
);
```

**Dependencias npm:** `socket.io`, `express`, `helmet`, `cors`, `ioredis` (ya en package.json)

**Variables de entorno:** `GATEWAY_PORT`, `CLIENT_ORIGIN`, `REDIS_URL`

**Verificación:**
- Cliente Socket.io se conecta a `http://localhost:3000`
- Emite `room:create` y con `redis-cli SUBSCRIBE svc:room` se ve el mensaje llegar

---

#### Historia 2 — UIUXH01: Identidad visual coherente

> Como usuario, quiero que la interfaz tenga una identidad visual consistente con el tema naval, para que la experiencia sea inmersiva.

---

##### Tarea: UIUXF001 — Pantallas principales de la aplicación cliente

**Microservicio:** Frontend (React + Vite) — repositorio separado

**Qué hace:** Crea el esqueleto React con React Router y las 4 pantallas principales: Login, Lobby, Tablero de juego y Resultado.

**Scaffolding a crear (en el repo frontend):**
```
src/
├── main.jsx
├── App.jsx                   ← React Router con 4 rutas
├── pages/
│   ├── LoginPage.jsx
│   ├── LobbyPage.jsx
│   ├── GamePage.jsx
│   └── ResultPage.jsx
├── components/               ← vacío por ahora
└── hooks/
    └── useSocket.js          ← hook de conexión Socket.io
```

**Cómo implementarlo:**

```jsx
// src/App.jsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import LoginPage from './pages/LoginPage';
import LobbyPage from './pages/LobbyPage';
import GamePage  from './pages/GamePage';
import ResultPage from './pages/ResultPage';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/"        element={<LoginPage />} />
        <Route path="/lobby"   element={<LobbyPage />} />
        <Route path="/game"    element={<GamePage />} />
        <Route path="/result"  element={<ResultPage />} />
        <Route path="*"        element={<Navigate to="/" />} />
      </Routes>
    </BrowserRouter>
  );
}
```

```js
// src/hooks/useSocket.js
import { useEffect, useRef } from 'react';
import { io } from 'socket.io-client';

const GATEWAY_URL = import.meta.env.VITE_GATEWAY_URL ?? 'http://localhost:3000';

export function useSocket(token) {
  const socketRef = useRef(null);

  useEffect(() => {
    if (!token) return;
    socketRef.current = io(GATEWAY_URL, { auth: { token } });
    return () => socketRef.current?.disconnect();
  }, [token]);

  return socketRef.current;
}
```

**Dependencias frontend:** `react-router-dom`, `socket.io-client`

---

##### Tarea: UIUXF002 — Tablero visual parametrizable y lenguaje de colores

**Microservicio:** Frontend

**Qué hace:** Crea el componente `Board` reutilizable (10×10 o N×N) con un sistema de tokens CSS para los colores del juego.

**Scaffolding:**
```
src/
├── components/
│   └── Board/
│       ├── Board.jsx
│       └── Board.module.css
└── styles/
    └── tokens.css            ← variables CSS de colores y tamaños
```

**Cómo implementarlo:**

```css
/* src/styles/tokens.css */
:root {
  --color-water:   #1a3a5c;
  --color-hit:     #e63946;
  --color-miss:    #a8dadc;
  --color-ship:    #457b9d;
  --color-sunk:    #6d2b2b;
  --color-fog:     #2b2d42;
  --color-select:  #f4a261;
  --cell-size:     2.5rem;
}
```

```jsx
// src/components/Board/Board.jsx
export default function Board({ size = 10, cells = {}, onCellClick }) {
  return (
    <div
      className={styles.grid}
      style={{ '--size': size, gridTemplateColumns: `repeat(${size}, var(--cell-size))` }}
    >
      {Array.from({ length: size * size }, (_, i) => {
        const x = i % size;
        const y = Math.floor(i / size);
        const state = cells[`${x},${y}`] ?? 'fog';
        return (
          <div
            key={`${x},${y}`}
            className={`${styles.cell} ${styles[state]}`}
            onClick={() => onCellClick?.(x, y)}
          />
        );
      })}
    </div>
  );
}
```

**Estados de celda:** `fog`, `ship`, `hit`, `miss`, `sunk`, `selected`

---

### Fase 1: Identidad y Salas

---

#### Historia 3 — DOMH02: Identificación segura del usuario

> Como usuario, quiero identificarme con mi cuenta de Google antes de jugar, para acceder de forma segura sin crear una contraseña nueva.

---

##### Tarea: DOMF101 — Identificación delegada y emisión de credencial de sesión

**Microservicio:** `battlecaos-auth` → `src/index.js`

**Qué hace:** Recibe el token de Google OAuth del cliente (ya verificado por Google en el frontend), lo verifica con `google-auth-library` y emite un JWT propio con la identidad del jugador.

**Scaffolding:**
```
server/auth/
└── index.js    ← contiene DOMF001 base + rutas OAuth
```

**Cómo implementarlo:**

```js
// server/auth/index.js (completo)
import 'dotenv/config';
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import jwt from 'jsonwebtoken';
import { OAuth2Client } from 'google-auth-library';
import { createRedis } from '../shared/redis.js';

const app = express();
const redis = createRedis();
const googleClient = new OAuth2Client(process.env.GOOGLE_CLIENT_ID);

app.use(helmet());
app.use(cors({ origin: process.env.CLIENT_ORIGIN }));
app.use(express.json());

// POST /auth/google  — recibe { idToken } del cliente
app.post('/auth/google', async (req, res) => {
  try {
    const ticket = await googleClient.verifyIdToken({
      idToken: req.body.idToken,
      audience: process.env.GOOGLE_CLIENT_ID,
    });
    const { sub, name, picture } = ticket.getPayload();

    const token = jwt.sign(
      { sub, name, picture },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({ token });
  } catch (err) {
    res.status(401).json({ error: 'token_invalido', detail: err.message });
  }
});

app.get('/health', async (_req, res) => {
  await redis.ping();
  res.json({ service: 'auth', status: 'ok' });
});

app.listen(process.env.AUTH_PORT ?? 3001, () =>
  console.log(`[auth] :${process.env.AUTH_PORT ?? 3001}`)
);
```

**Payload del JWT generado:**
```json
{ "sub": "google-uid-123", "name": "Jugador1", "picture": "https://...", "iat": ..., "exp": ... }
```

**Variables de entorno:** `GOOGLE_CLIENT_ID`, `JWT_SECRET`, `AUTH_PORT`

**Dependencias npm:** `jsonwebtoken`, `google-auth-library` (ya en package.json)

---

##### Tarea: DOMF102 — Verificación de la credencial en cada conexión

**Microservicio:** `battlecaos-gateway` → `src/index.js`

**Qué hace:** Middleware de Socket.io que verifica el JWT en cada nueva conexión antes de permitir que el cliente envíe eventos.

**Cómo implementarlo (añadir al Gateway):**

```js
// Añadir en server/gateway/index.js, ANTES de io.on('connection')
import jwt from 'jsonwebtoken';

io.use((socket, next) => {
  const token = socket.handshake.auth?.token;
  if (!token) return next(new Error('sin_token'));

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    socket.user = payload; // { sub, name, picture }
    next();
  } catch {
    next(new Error('token_invalido'));
  }
});
```

> Después de este middleware, todos los handlers del Gateway pueden acceder a `socket.user.sub` (el ID de Google del jugador) para incluirlo en los eventos que publican a Redis.

**Variables de entorno:** `JWT_SECRET`

---

#### Historia 4 — UIUXH02: Inicio de sesión sin fricción

---

##### Tarea: UIUXF101 — Pantalla de inicio de sesión con estados de carga y error

**Microservicio:** Frontend

**Qué hace:** Pantalla de login con botón "Iniciar sesión con Google" usando Google Identity Services. Muestra estados: idle, cargando, error.

**Scaffolding:**
```
src/pages/
└── LoginPage.jsx
```

**Cómo implementarlo:**
```jsx
// src/pages/LoginPage.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';

const GATEWAY = import.meta.env.VITE_AUTH_URL ?? 'http://localhost:3001';

export default function LoginPage() {
  const [status, setStatus] = useState('idle'); // idle | loading | error
  const navigate = useNavigate();

  function handleGoogleResponse(response) {
    setStatus('loading');
    fetch(`${GATEWAY}/auth/google`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ idToken: response.credential }),
    })
      .then((r) => r.json())
      .then(({ token }) => {
        localStorage.setItem('token', token);
        navigate('/lobby');
      })
      .catch(() => setStatus('error'));
  }

  // Inicializar Google Identity Services
  useEffect(() => {
    window.google?.accounts.id.initialize({
      client_id: import.meta.env.VITE_GOOGLE_CLIENT_ID,
      callback: handleGoogleResponse,
    });
    window.google?.accounts.id.renderButton(
      document.getElementById('google-btn'),
      { theme: 'outline', size: 'large' }
    );
  }, []);

  return (
    <main>
      <h1>BattleCaos-Ship</h1>
      {status === 'error' && <p role="alert">Error al iniciar sesión. Inténtalo de nuevo.</p>}
      {status === 'loading' ? <p>Cargando...</p> : <div id="google-btn" />}
    </main>
  );
}
```

**Variables de entorno frontend:** `VITE_GOOGLE_CLIENT_ID`, `VITE_AUTH_URL`

---

#### Historia 5 — DOMH03: Creación y unión a salas mediante código

> Como usuario, quiero crear o unirme a una sala con un código de 6 dígitos, para jugar con amigos en el modo que elija.

---

##### Tarea: DOMF201 — Creación y unión a salas con asignación de equipo

**Microservicio:** `battlecaos-room` → `src/index.js`

**Qué hace:** Escucha los eventos `room:create` y `room:join` del canal `svc:room`. Crea salas en Redis con un código único de 6 dígitos, valida los slots disponibles según el modo de juego y asigna equipos.

**Scaffolding:**
```
server/room/
└── index.js
```

**Cómo implementarlo:**

```js
// server/room/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

const redis = createRedis();
const sub   = createRedis();

// Slots máximos por modo
const SLOTS = { '1v1': 2, '1v1-bot': 1, '2v2': 4 };

sub.subscribe('svc:room');
sub.on('message', async (_ch, raw) => {
  const msg = JSON.parse(raw);
  if (msg.type === 'room:create') await handleCreate(msg.data);
  if (msg.type === 'room:join')   await handleJoin(msg.data);
  if (msg.type === 'PlayerDisconnected') await handleDisconnect(msg.data);
});

async function handleCreate({ socketId, modo, playerId, name }) {
  const codigo = Math.random().toString().slice(2, 8); // 6 dígitos
  const sala = {
    codigo,
    modo,
    fase: 'LOBBY',
    jugadores: [{ id: playerId, name, socketId, equipo: 'A', conectado: true }],
    slotsMax: SLOTS[modo] ?? 2,
    creadoEn: Date.now(),
  };
  await redis.set(`sala:${codigo}`, JSON.stringify(sala));

  // Broadcast al cliente que creó la sala
  await redis.publish('gw:broadcast', JSON.stringify({
    roomId: socketId,          // sala temporal del socket
    event: 'room:created',
    payload: { codigo, modo },
  }));

  // Publicar evento de ciclo de vida
  await redis.publish('evt:PlayerRoomJoined', JSON.stringify({
    type: 'PlayerRoomJoined', source: 'room', timestamp: Date.now(),
    data: { codigo, playerId, name },
  }));
}

async function handleJoin({ socketId, codigo, playerId, name }) {
  const raw = await redis.get(`sala:${codigo}`);
  if (!raw) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: socketId, event: 'room:error', payload: { error: 'sala_no_existe' },
    }));
    return;
  }
  const sala = JSON.parse(raw);
  if (sala.jugadores.length >= sala.slotsMax) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: socketId, event: 'room:error', payload: { error: 'sala_llena' },
    }));
    return;
  }

  // Asignar equipo: B en 1v1, A o B en 2v2 según posición
  const equipo = sala.jugadores.length % 2 === 0 ? 'A' : 'B';
  sala.jugadores.push({ id: playerId, name, socketId, equipo, conectado: true });
  await redis.set(`sala:${codigo}`, JSON.stringify(sala));

  await redis.publish('gw:broadcast', JSON.stringify({
    roomId: codigo, event: 'room:joined', payload: { codigo, jugadores: sala.jugadores },
  }));
  await redis.publish('evt:PlayerRoomJoined', JSON.stringify({
    type: 'PlayerRoomJoined', source: 'room', timestamp: Date.now(),
    data: { codigo, playerId, name },
  }));
}
```

**Claves Redis que usa:**
- `sala:{codigo}` ← crea y lee

---

##### Tarea: DOMF202 — Aislamiento entre salas concurrentes

**Microservicio:** `battlecaos-room` (define la convención de naming de claves) + `battlecaos-game` (la aplica en sus propias claves)

> No es un servicio nuevo — es una restricción de diseño que cada repositorio implementa independientemente. `battlecaos-room` crea las claves `sala:{codigo}` y `battlecaos-game` sigue ese mismo prefijo para sus claves `sala:{codigo}:energia:*`, `sala:{codigo}:salva:*`, etc.

**Qué hace:** Garantiza que todos los eventos de sala y juego incluyan el `codigo` de sala y que las claves Redis incluyan ese código como namespace. Así 100 partidas simultáneas no se interfieren.

**Cómo implementarlo:**

Este no es código nuevo — es una **restricción de diseño** que debe cumplirse en **todos** los handlers:

1. Toda clave Redis debe tener el prefijo `sala:{codigo}:...`
2. Todo evento publicado en Redis debe incluir `data.codigo`
3. Cuando el Gateway hace broadcast, usa `io.to(codigo).emit(...)` (no `io.emit`)
4. El Room Service hace que cada socket se una a la room de Socket.io correspondiente al `codigo`:

```js
// En Room Service, tras crear o unirse a sala exitosamente:
await redis.publish('gw:broadcast', JSON.stringify({
  roomId: socketId,
  event: 'room:join-socket-room',   // Gateway ejecuta socket.join(codigo)
  payload: { codigo },
}));
```

```js
// En Gateway, manejar este evento especial:
sub.on('message', (_ch, raw) => {
  const { roomId, event, payload } = JSON.parse(raw);
  if (event === 'room:join-socket-room') {
    const socket = io.sockets.sockets.get(roomId);
    socket?.join(payload.codigo);
    return;
  }
  io.to(roomId).emit(event, payload);
});
```

---

##### Tarea: DOMF203 — Publicación de eventos de ciclo de vida de sala

**Microservicio:** `battlecaos-room` → ampliar `src/index.js`

**Qué hace:** Publica `RoomReady` cuando la sala está llena y `PlayerDisconnectedFromRoom` / `PlayerReconnected` / `RoomDestroyed` según el ciclo de desconexión.

**Cómo implementarlo (ampliar Room Service):**

```js
// Dentro de handleJoin(), después de guardar sala en Redis:
if (sala.jugadores.length === sala.slotsMax) {
  // Sala llena — publicar RoomReady
  await redis.publish('evt:RoomReady', JSON.stringify({
    type: 'RoomReady', source: 'room', timestamp: Date.now(),
    data: { codigo, modo: sala.modo, equipos: { A: [], B: [] }, jugadores: sala.jugadores },
  }));
}

// En handleDisconnect():
async function handleDisconnect({ socketId }) {
  // Buscar en qué sala estaba este socket
  // (en producción, mantener un índice socketId → codigo)
  // Por ahora, escanear (MVP):
  const keys = await redis.keys('sala:*');
  for (const key of keys) {
    if (key.includes(':chat')) continue;
    const sala = JSON.parse(await redis.get(key));
    const jugador = sala.jugadores.find((j) => j.socketId === socketId);
    if (!jugador) continue;

    jugador.conectado = false;
    await redis.set(key, JSON.stringify(sala));

    await redis.publish('evt:PlayerDisconnectedFromRoom', JSON.stringify({
      type: 'PlayerDisconnectedFromRoom', source: 'room', timestamp: Date.now(),
      data: { codigo: sala.codigo, playerId: jugador.id },
    }));

    const todosDesconectados = sala.jugadores.every((j) => !j.conectado);
    if (todosDesconectados) {
      await redis.publish('evt:RoomDestroyed', JSON.stringify({
        type: 'RoomDestroyed', source: 'room', timestamp: Date.now(),
        data: { codigo: sala.codigo },
      }));
    }
    break;
  }
}
```

---

#### Historia 6 — UIUXH03: Lobby para crear o unirse a salas

---

##### Tarea: UIUXF201 — Lobby con selector de modo, creación y unión a salas

**Microservicio:** Frontend

**Qué hace:** Pantalla de lobby con formulario para crear sala (elegir modo) o unirse con código, y lista de jugadores en sala en espera.

**Scaffolding:**
```
src/pages/LobbyPage.jsx
src/components/PlayerList.jsx
```

**Cómo implementarlo:**

```jsx
// src/pages/LobbyPage.jsx
import { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { useSocket } from '../hooks/useSocket';

export default function LobbyPage() {
  const token = localStorage.getItem('token');
  const socket = useSocket(token);
  const navigate = useNavigate();
  const [codigo, setCodigo] = useState('');
  const [modo, setModo] = useState('1v1');
  const [jugadores, setJugadores] = useState([]);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!socket) return;
    socket.on('room:created', ({ codigo }) => setCodigo(codigo));
    socket.on('room:joined',  ({ jugadores }) => setJugadores(jugadores));
    socket.on('room:error',   ({ error }) => setError(error));
    socket.on('game:start',   () => navigate('/game'));
    return () => socket.removeAllListeners();
  }, [socket]);

  return (
    <main>
      <section>
        <h2>Crear sala</h2>
        <select value={modo} onChange={(e) => setModo(e.target.value)}>
          <option value="1v1">1 vs 1</option>
          <option value="1v1-bot">1 vs Bot</option>
          <option value="2v2">2 vs 2</option>
        </select>
        <button onClick={() => socket.emit('room:create', { modo })}>Crear</button>
      </section>

      <section>
        <h2>Unirse</h2>
        <input value={codigo} onChange={(e) => setCodigo(e.target.value)} maxLength={6} />
        <button onClick={() => socket.emit('room:join', { codigo })}>Unirse</button>
      </section>

      {error && <p role="alert">{error}</p>}
      {codigo && <p>Código de sala: <strong>{codigo}</strong></p>}
      <ul>{jugadores.map((j) => <li key={j.id}>{j.name} — Equipo {j.equipo}</li>)}</ul>
    </main>
  );
}
```

---

### Fase 2: Loop Principal del Juego

---

#### Historia 7 — DOMH04: Estado de partida siempre actualizado

---

##### Tarea: DOMF301 — Distribución del estado completo tras cada cambio

**Microservicio:** `battlecaos-game` (publica en `gw:broadcast`) + `battlecaos-gateway` (se suscribe y hace broadcast a clientes)

> Cada repositorio implementa solo su mitad del contrato. `battlecaos-game` nunca toca Socket.io — solo publica en Redis. `battlecaos-gateway` nunca toca lógica de juego — solo retransmite lo que recibe.

**Qué hace:** Después de cada acción de juego, el Game Service publica el estado completo de la sala en `gw:broadcast` para que el Gateway lo envíe a todos los clientes de esa sala.

**Cómo implementarlo:**

```js
// server/game/handlers/broker.js — helper para broadcast de estado
import { redis } from '../index.js';

export async function broadcastState(codigo) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  // Versión sanitizada: no enviar tableros enemigos completos
  const stateForClients = sanitizeState(sala);
  await redis.publish('gw:broadcast', JSON.stringify({
    roomId: codigo,
    event: 'game:state',
    payload: stateForClients,
  }));
}

function sanitizeState(sala) {
  // Ocultar posiciones de barcos del equipo contrario (solo revelar impactos)
  return {
    fase: sala.fase,
    turno: sala.turno,
    energia: sala.energia,
    jugadores: sala.jugadores.map((j) => ({ id: j.id, name: j.name, equipo: j.equipo, conectado: j.conectado })),
    tableroPublico: sala.tableroPublico, // solo celdas disparadas
  };
}
```

> `broadcastState(codigo)` se llama al final de cada handler (handleShot, handlePower, etc.) para mantener a todos los clientes sincronizados.

---

##### Tarea: DOMF302 — Definición de fases válidas de una partida

**Microservicio:** `battlecaos-game` → `src/domain/engine.js`

**Qué hace:** Define la máquina de estados de fases `LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN` y la función `processShot` pura (sin Redis).

**Scaffolding:**
```
server/game/
├── domain/
│   ├── engine.js     ← fases, validación de turno
│   └── errors.js     ← clase DomainError
├── handlers/
│   └── broker.js     ← suscripciones a Redis
└── index.js
```

**Cómo implementarlo:**

```js
// server/game/domain/errors.js
export class DomainError extends Error {
  constructor(code, message) {
    super(message);
    this.code = code;
  }
}
```

```js
// server/game/domain/engine.js — funciones puras, 0 imports externos
import { DomainError } from './errors.js';

export const FASES = ['LOBBY', 'COLOCACION', 'TURNOS', 'SALVA', 'FIN'];

export function nextFase(faseActual) {
  const transiciones = {
    LOBBY:      'COLOCACION',
    COLOCACION: 'TURNOS',
    TURNOS:     'SALVA',
    SALVA:      'TURNOS',
  };
  const siguiente = transiciones[faseActual];
  if (!siguiente) throw new DomainError('fase_invalida', `No hay transición desde ${faseActual}`);
  return siguiente;
}

export function validateTurn(sala, playerId) {
  if (sala.fase !== 'TURNOS') throw new DomainError('fase_incorrecta', 'No es fase de turnos');
  if (sala.turno?.jugadorActual !== playerId) throw new DomainError('no_es_tu_turno', 'No es tu turno');
  return true;
}

export function processShot(sala, playerId, x, y, board) {
  validateTurn(sala, playerId);
  return board.shoot(x, y); // delega en board.js
}
```

```js
// server/game/handlers/broker.js — suscripciones y enrutamiento
import { redisSub } from '../index.js';
import { handlePlaceShips } from './handlePlaceShips.js';
import { handleShot }       from './handleShot.js';

const HANDLERS = {
  'colocacion:set':      handlePlaceShips,
  'disparo:realizar':    handleShot,
  // Se irán añadiendo: handlePower, handleSalvo, handleDisconnect
};

redisSub.subscribe('svc:game', 'evt:RoomReady', 'evt:TimerEnd',
                   'evt:PlayerDisconnectedFromRoom', 'evt:PlayerReconnected', 'evt:BotDecision');

redisSub.on('message', async (_ch, raw) => {
  const msg = JSON.parse(raw);
  const handler = HANDLERS[msg.type] ?? HANDLERS[msg.data?.tipo];
  if (handler) {
    try { await handler(msg.data); }
    catch (err) { console.error('[game] handler error:', err.message); }
  }
});
```

---

#### Historia 8 — UIUXH04: Tablero siempre fiel al estado real

---

##### Tarea: UIUXF301 — Render del tablero a partir del estado recibido

**Microservicio:** Frontend

**Qué hace:** `GamePage` escucha el evento `game:state` del socket y renderiza los dos tableros (propio y rival) usando el componente `Board` creado en UIUXF002.

**Scaffolding:**
```
src/pages/GamePage.jsx
```

```jsx
// src/pages/GamePage.jsx
import { useState, useEffect } from 'react';
import { useSocket } from '../hooks/useSocket';
import Board from '../components/Board/Board';

export default function GamePage() {
  const socket = useSocket(localStorage.getItem('token'));
  const [gameState, setGameState] = useState(null);

  useEffect(() => {
    if (!socket) return;
    socket.on('game:state', setGameState);
    return () => socket.off('game:state', setGameState);
  }, [socket]);

  if (!gameState) return <p>Conectando...</p>;

  return (
    <main>
      <h2>Fase: {gameState.fase}</h2>
      <section>
        <h3>Tu tablero</h3>
        <Board cells={gameState.tableroPublico?.propio} />
      </section>
      <section>
        <h3>Tablero rival</h3>
        <Board cells={gameState.tableroPublico?.rival}
               onCellClick={(x, y) => socket.emit('disparo:realizar', { x, y })} />
      </section>
    </main>
  );
}
```

---

#### Historia 9 — DOMH06: Colocación de la flota propia

---

##### Tarea: DOMF501 — Validación y registro de la colocación de flota

**Microservicio:** `battlecaos-game` → `src/handlers/handlePlaceShips.js` + `src/domain/board.js` + `src/domain/fleet.js`

**Qué hace:** Valida que la flota no se solape ni exceda los límites, persiste la flota en Redis y publica `ShipsPlaced`.

**Scaffolding:**
```
server/game/
├── domain/
│   ├── board.js
│   └── fleet.js
└── handlers/
    └── handlePlaceShips.js
```

**Cómo implementarlo:**

```js
// server/game/domain/board.js — puro
export function createBoard(size = 10) {
  return { size, cells: {} }; // cells: { "x,y": 'ship'|'hit'|'miss'|'sunk' }
}

export function placeShip(board, ship) {
  const cells = getShipCells(ship);
  for (const [x, y] of cells) {
    if (x < 0 || y < 0 || x >= board.size || y >= board.size)
      throw new Error('fuera_de_limites');
    if (board.cells[`${x},${y}`])
      throw new Error('celda_ocupada');
  }
  for (const [x, y] of cells) board.cells[`${x},${y}`] = 'ship';
  return board;
}

export function shoot(board, x, y) {
  const key = `${x},${y}`;
  if (board.cells[key] === 'hit' || board.cells[key] === 'miss') return { valid: false };
  const hit = board.cells[key] === 'ship';
  board.cells[key] = hit ? 'hit' : 'miss';
  return { valid: true, hit };
}

function getShipCells({ x, y, size, horizontal }) {
  return Array.from({ length: size }, (_, i) =>
    horizontal ? [x + i, y] : [x, y + i]
  );
}
```

```js
// server/game/domain/fleet.js — puro
export const FLEET_CONFIG = [
  { id: 'portaaviones', size: 5 },
  { id: 'acorazado',    size: 4 },
  { id: 'crucero',      size: 3 },
  { id: 'submarino',    size: 3 },
  { id: 'destructor',   size: 2 },
];

export function validateFleet(ships) {
  if (ships.length !== FLEET_CONFIG.length) throw new Error('flota_incompleta');
  const ids = ships.map((s) => s.id).sort().join(',');
  const expected = FLEET_CONFIG.map((s) => s.id).sort().join(',');
  if (ids !== expected) throw new Error('barcos_incorrectos');
}

export function isFleetSunk(board) {
  return !Object.values(board.cells).includes('ship');
}
```

```js
// server/game/handlers/handlePlaceShips.js
import { redis } from '../index.js';
import { createBoard, placeShip } from '../domain/board.js';
import { validateFleet } from '../domain/fleet.js';
import { broadcastState } from './broker.js';

export async function handlePlaceShips({ codigo, playerId, ships }) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  if (sala.fase !== 'COLOCACION') return;

  try {
    validateFleet(ships);
    const board = createBoard();
    for (const ship of ships) placeShip(board, ship);

    sala.tableros ??= {};
    sala.tableros[playerId] = board;
    sala.colocados ??= [];
    if (!sala.colocados.includes(playerId)) sala.colocados.push(playerId);

    await redis.set(`sala:${codigo}`, JSON.stringify(sala));

    // Evento ShipsPlaced para Observabilidad y compañero de equipo (DOMF502)
    const jugador = sala.jugadores.find((j) => j.id === playerId);
    await redis.publish('evt:ShipsPlaced', JSON.stringify({
      type: 'ShipsPlaced', source: 'game', timestamp: Date.now(),
      data: { codigo, playerId, equipo: jugador?.equipo },
    }));

    await broadcastState(codigo);
  } catch (err) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: playerId, event: 'game:error', payload: { error: err.message },
    }));
  }
}
```

---

#### Historia 10 — UIUXH06: Colocación de flota cómoda e intuitiva

---

##### Tarea: UIUXF501 — Colocación interactiva de barcos con vista previa

**Microservicio:** Frontend

**Qué hace:** Permite arrastrar o hacer clic para colocar cada barco en el tablero. Muestra vista previa del barco a colocar y lo gira con un botón. Envía la flota al servidor con `colocacion:set`.

**Scaffolding:**
```
src/components/
└── ShipPlacer/
    ├── ShipPlacer.jsx
    └── ShipPlacer.module.css
```

**Cómo implementarlo:**

```jsx
// src/components/ShipPlacer/ShipPlacer.jsx
const FLEET = [
  { id: 'portaaviones', size: 5 },
  { id: 'acorazado',    size: 4 },
  { id: 'crucero',      size: 3 },
  { id: 'submarino',    size: 3 },
  { id: 'destructor',   size: 2 },
];

export default function ShipPlacer({ socket, codigo }) {
  const [placed, setPlaced] = useState([]);    // barcos ya colocados
  const [current, setCurrent] = useState(0);  // índice del barco actual
  const [horizontal, setHorizontal] = useState(true);
  const [preview, setPreview] = useState(null);

  function handleCellClick(x, y) {
    if (current >= FLEET.length) return;
    const ship = { ...FLEET[current], x, y, horizontal };
    setPlaced((prev) => [...prev, ship]);
    setCurrent((c) => c + 1);
  }

  function handleConfirm() {
    socket.emit('colocacion:set', { codigo, ships: placed });
  }

  return (
    <div>
      {current < FLEET.length && (
        <p>Coloca: <strong>{FLEET[current].id}</strong> ({FLEET[current].size} celdas)
          <button onClick={() => setHorizontal((h) => !h)}>
            Girar ({horizontal ? 'H' : 'V'})
          </button>
        </p>
      )}
      <Board size={10} cells={buildCells(placed)} onCellClick={handleCellClick} />
      {current >= FLEET.length && (
        <button onClick={handleConfirm}>Confirmar flota</button>
      )}
    </div>
  );
}
```

---

#### Historia 11 — DOMH07: Colocación coordinada en equipo

---

##### Tarea: DOMF502 — Coordinación de colocación en equipo con límite de tiempo

**Microservicio:** `battlecaos-game` → ampliar `src/handlers/handlePlaceShips.js`

**Qué hace:** En modo 2v2, cuando un compañero confirma su flota, notifica al otro en tiempo real. Cuando todos los jugadores colocan su flota (o vence el timer de 60s de DOMF1303), avanza a fase TURNOS.

**Cómo implementarlo (dentro de handlePlaceShips.js):**

```js
// Al final de handlePlaceShips(), después de broadcastState():
const todosColocados = sala.jugadores.every((j) => sala.colocados?.includes(j.id));
if (todosColocados) {
  sala.fase = 'TURNOS';
  sala.turno = {
    jugadorActual: sala.jugadores[0].id,
    numeroTurno: 1,
  };
  await redis.set(`sala:${codigo}`, JSON.stringify(sala));
  await redis.publish('evt:PhaseChanged', JSON.stringify({
    type: 'PhaseChanged', source: 'game', timestamp: Date.now(),
    data: { codigo, from: 'COLOCACION', to: 'TURNOS' },
  }));
  await broadcastState(codigo);
}
```

---

#### Historia 12 — DOMH08: Disparos por turnos con resultado visible

---

##### Tarea: DOMF601 — Validación y resolución de disparos por turno

**Microservicio:** `battlecaos-game` → `src/handlers/handleShot.js`

**Qué hace:** Valida que es el turno del jugador, aplica el disparo en el tablero del rival, acumula energía y publica `ShotFired`.

**Scaffolding:**
```
server/game/handlers/handleShot.js
```

**Cómo implementarlo:**

```js
// server/game/handlers/handleShot.js
import { redis } from '../index.js';
import { validateTurn } from '../domain/engine.js';
import { shoot } from '../domain/board.js';
import { broadcastState } from './broker.js';

export async function handleShot({ codigo, playerId, x, y }) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));

  try {
    validateTurn(sala, playerId);
  } catch (err) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: playerId, event: 'game:error', payload: { error: err.code },
    }));
    return;
  }

  // Determinar tablero del rival
  const jugador = sala.jugadores.find((j) => j.id === playerId);
  const rival   = sala.jugadores.find((j) => j.equipo !== jugador.equipo);
  const board   = sala.tableros?.[rival.id];

  const result = shoot(board, x, y);
  if (!result.valid) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: playerId, event: 'game:error', payload: { error: 'celda_ya_disparada' },
    }));
    return;
  }

  // Energía se acumula en DOMF701 — por ahora solo turno
  // Rotar turno al siguiente jugador
  const idx = sala.jugadores.findIndex((j) => j.id === playerId);
  sala.turno.jugadorActual = sala.jugadores[(idx + 1) % sala.jugadores.length].id;
  sala.turno.numeroTurno++;

  await redis.set(`sala:${codigo}`, JSON.stringify(sala));

  await redis.publish('evt:ShotFired', JSON.stringify({
    type: 'ShotFired', source: 'game', timestamp: Date.now(),
    data: { codigo, playerId, x, y, result: result.hit ? 'hit' : 'miss' },
  }));

  await broadcastState(codigo);
}
```

---

##### Tarea: DOMF602 — Detección de hundimientos y condición de victoria

**Microservicio:** `battlecaos-game` → ampliar `src/handlers/handleShot.js` + `src/domain/fleet.js`

**Qué hace:** Después de cada disparo comprueba si el barco tocado se hundió y si todos los barcos rivales están hundidos (victoria).

**Cómo implementarlo (ampliar handleShot.js):**

```js
import { isFleetSunk } from '../domain/fleet.js';

// Después de shoot(board, x, y) en handleShot.js:
if (result.hit) {
  const shipSunk = checkShipSunk(board, result.shipId);
  if (shipSunk) {
    await redis.publish('evt:ShipSunk', JSON.stringify({
      type: 'ShipSunk', source: 'game', timestamp: Date.now(),
      data: { codigo, shipId: result.shipId, equipoAtacante: jugador.equipo, equipoDueno: rival.equipo },
    }));
  }

  if (isFleetSunk(board)) {
    sala.fase = 'FIN';
    sala.winner = jugador.equipo;
    await redis.set(`sala:${codigo}`, JSON.stringify(sala));
    await redis.publish('evt:GameEnded', JSON.stringify({
      type: 'GameEnded', source: 'game', timestamp: Date.now(),
      data: { codigo, winner: jugador.equipo, modo: sala.modo, duracion: Date.now() - sala.iniciadoEn },
    }));
    await broadcastState(codigo);
    return;
  }
}
```

---

#### Historia 13 — UIUXH07: Turno y resultado de disparos visibles

---

##### Tarea: UIUXF601 — Indicador de turno y retroalimentación visual de disparos

**Microservicio:** Frontend

**Qué hace:** Muestra quién tiene el turno activo y reproduce una animación/sonido en la celda disparada (hit/miss/sunk).

**Cómo implementarlo:**

```jsx
// En GamePage.jsx, añadir:
const [lastShot, setLastShot] = useState(null);

useEffect(() => {
  if (!socket) return;
  socket.on('game:state', setGameState);
  socket.on('shot:result', setLastShot); // { x, y, result: 'hit'|'miss'|'sunk' }
}, [socket]);

// En el render:
<p>
  {gameState?.turno?.jugadorActual === myId
    ? '¡Es tu turno!'
    : `Turno de ${turnoNombre}`}
</p>
{lastShot && (
  <div className={`shot-feedback ${lastShot.result}`}>
    {lastShot.result === 'hit' ? '💥 Impacto!' : lastShot.result === 'sunk' ? '⚓ Hundido!' : 'Agua'}
  </div>
)}
```

---

#### Historia 14 — DOMH09: Acumulación de energía de equipo

---

##### Tarea: DOMF701 — Acumulación de energía por equipo sin condiciones de carrera

**Microservicio:** `battlecaos-game` → `src/domain/energy.js` + integración en `src/handlers/handleShot.js`

**Qué hace:** Usa `INCRBY` atómico de Redis para acumular energía por equipo. +1 por impacto, +3 por hundimiento.

**Scaffolding:**
```
server/game/domain/energy.js
```

**Cómo implementarlo:**

```js
// server/game/domain/energy.js — puro
export function energyGain(hit, sunk) {
  if (sunk) return 4; // +1 por hit + 3 por hundir = 4
  if (hit)  return 1;
  return 0;
}

export function hasEnough(currentEnergy, cost) {
  return currentEnergy >= cost;
}
```

```js
// En handleShot.js, después de detectar hit/sunk — usar INCRBY atómico:
const gain = energyGain(result.hit, shipSunk);
if (gain > 0) {
  await redis.incrby(`sala:${codigo}:energia:${jugador.equipo}`, gain);
}

// Para leer energía actual:
const energia = parseInt(await redis.get(`sala:${codigo}:energia:${jugador.equipo}`) ?? '0');
```

> `INCRBY` es atómico en Redis: aunque 2 procesos del Game Service procesen disparos simultáneamente, la energía nunca se corrompe.

---

#### Historia 15 — UIUXH08: Energía de equipo visible

---

##### Tarea: UIUXF701 — Barra de energía sincronizada con el estado de la partida

**Microservicio:** Frontend

**Scaffolding:**
```
src/components/EnergyBar/
├── EnergyBar.jsx
└── EnergyBar.module.css
```

```jsx
// src/components/EnergyBar/EnergyBar.jsx
export default function EnergyBar({ energia = 0, max = 10 }) {
  const pct = Math.min((energia / max) * 100, 100);
  return (
    <div className={styles.container} role="meter" aria-valuenow={energia} aria-valuemax={max}>
      <div className={styles.fill} style={{ width: `${pct}%` }} />
      <span>{energia} E</span>
    </div>
  );
}
```

---

#### Historia 16 — DOMH05: Recuperación de partida tras una desconexión

---

##### Tarea: DOMF401 — Recuperación de estado al reconectar

**Microservicio:** `battlecaos-gateway` (detecta el socket reconectado, publica `evt:PlayerReconnected`) + `battlecaos-game` (consume ese evento y restaura el estado del jugador en partida)

> `battlecaos-gateway` hace el trabajo de asociar el nuevo `socket.id` al jugador. `battlecaos-game` hace el trabajo de reanudar el turno o la espera. Son dos responsabilidades distintas en dos repositorios distintos.

**Qué hace:** Cuando un jugador reconecta, el Gateway detecta que ya tiene un JWT válido con su `sub`, busca su sala en Redis y le envía el estado completo.

**Cómo implementarlo:**

```js
// En Gateway, tras verificar JWT en el middleware, añadir lógica de reconexión:
io.on('connection', async (socket) => {
  const playerId = socket.user.sub;

  // Buscar si este jugador tiene sala activa (MVP: scan de claves)
  const keys = await pub.keys('sala:*');
  for (const key of keys) {
    if (key.includes(':')) continue; // saltar sub-claves
    const sala = JSON.parse(await pub.get(key));
    const jugador = sala?.jugadores?.find((j) => j.id === playerId);
    if (jugador && sala.fase !== 'FIN') {
      // Reconectar: unir al socket room y notificar al Room Service
      socket.join(sala.codigo);
      jugador.socketId = socket.id;
      await pub.set(key, JSON.stringify(sala));

      // Notificar reconexión
      await pub.publish('evt:PlayerReconnected', JSON.stringify({
        type: 'PlayerReconnected', source: 'gateway', timestamp: Date.now(),
        data: { codigo: sala.codigo, playerId },
      }));

      // Enviar estado completo al cliente reconectado
      socket.emit('game:state', sala);
      break;
    }
  }
});
```

---

##### Tarea: DOMF402 — El usuario desconectado conserva su lugar (no se expulsa)

**Microservicio:** `battlecaos-room` → ampliar `src/index.js`

**Qué hace:** Al recibir `PlayerDisconnected`, Room Service marca `conectado: false` en el jugador pero **no lo elimina** del array ni borra su flota. El jugador sigue existiendo en la sala.

Esta restricción ya está implementada en DOMF203 (`handleDisconnect`). Lo que hay que garantizar adicionalmente:

```js
// En Room Service handleDisconnect — NO hacer esto:
// sala.jugadores = sala.jugadores.filter(j => j.id !== jugador.id); // ❌ INCORRECTO

// Solo marcar como desconectado — jugador y flota se conservan
jugador.conectado = false;
jugador.desconectadoEn = Date.now();
await redis.set(key, JSON.stringify(sala));
```

---

##### Tarea: DOMF403 — Publicación de eventos de desconexión y reconexión desde la sala

**Microservicio:** `battlecaos-room` → ya cubierto por DOMF203

Esta tarea ya está implementada en DOMF203. Los eventos `PlayerDisconnectedFromRoom`, `PlayerReconnected` y `RoomDestroyed` se publican en los canales `evt:*` correspondientes.

**Verificación adicional — el Game Service debe suscribirse:**

```js
// En server/game/handlers/broker.js, añadir consumidor:
redisSub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);
  if (channel === 'evt:PlayerDisconnectedFromRoom') {
    await handleDisconnect(msg.data); // implementado en DOMF1301
  }
  if (channel === 'evt:PlayerReconnected') {
    await handleReconnect(msg.data);
  }
});
```

---

#### Historia 17 — UIUXH05: Chat de equipo en tiempo real

---

##### Tarea: DOMF1103 — Servicio de mensajería: recepción, persistencia y enrutamiento

**Microservicio:** `battlecaos-chat` → `src/index.js`

**Qué hace:** Recibe mensajes del canal `svc:chat`, los persiste en Redis (máximo 100 por sala con `LTRIM`), los enruta por equipo (en 2v2) y publica `ChatMessage` para Observabilidad.

**Scaffolding:**
```
server/chat/index.js
```

**Cómo implementarlo:**

```js
// server/chat/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

const redis = createRedis();
const sub   = createRedis();

sub.subscribe('svc:chat', 'evt:PlayerRoomJoined');

sub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);
  if (channel === 'svc:chat')            await handleMessage(msg.data);
  if (channel === 'evt:PlayerRoomJoined') await sendHistory(msg.data);
});

async function handleMessage({ codigo, senderId, senderName, text, equipo }) {
  const entry = JSON.stringify({ senderId, senderName, text, equipo, ts: Date.now() });
  const key   = `sala:${codigo}:chat`;

  // Persistir y mantener solo los últimos 100 mensajes
  await redis.lpush(key, entry);
  await redis.ltrim(key, 0, 99);

  // Broadcast: en 2v2 solo al equipo, en 1v1 a todos
  await redis.publish('gw:broadcast', JSON.stringify({
    roomId: codigo,
    event: 'chat:message',
    payload: { senderId, senderName, text, equipo, ts: Date.now() },
  }));

  // Evento para Observabilidad
  await redis.publish('evt:ChatMessage', JSON.stringify({
    type: 'ChatMessage', source: 'chat', timestamp: Date.now(),
    data: { codigo, senderId, senderName, text, equipo },
  }));
}

async function sendHistory({ codigo, playerId }) {
  const key      = `sala:${codigo}:chat`;
  const rawList  = await redis.lrange(key, 0, 99);
  const messages = rawList.map((r) => JSON.parse(r)).reverse();

  await redis.publish('gw:broadcast', JSON.stringify({
    roomId: playerId,
    event: 'chat:history',
    payload: messages,
  }));
}
```

---

##### Tarea: UIUXF401 — Chat de equipo y general en tiempo real

**Microservicio:** Frontend

**Scaffolding:**
```
src/components/Chat/
├── Chat.jsx
└── Chat.module.css
```

```jsx
// src/components/Chat/Chat.jsx
export default function Chat({ socket, codigo }) {
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState('');

  useEffect(() => {
    if (!socket) return;
    socket.on('chat:history', (hist) => setMessages(hist));
    socket.on('chat:message', (msg)  => setMessages((m) => [...m, msg]));
    return () => { socket.off('chat:history'); socket.off('chat:message'); };
  }, [socket]);

  function send() {
    if (!text.trim()) return;
    socket.emit('chat:mensaje', { codigo, text });
    setText('');
  }

  return (
    <aside>
      <ul>{messages.map((m, i) => <li key={i}><b>{m.senderName}:</b> {m.text}</li>)}</ul>
      <input value={text} onChange={(e) => setText(e.target.value)}
             onKeyDown={(e) => e.key === 'Enter' && send()} />
      <button onClick={send}>Enviar</button>
    </aside>
  );
}
```

---

## SPRINT 2 — Mecánicas, Observabilidad y Cierre

---

### Fase 3: Calidad y Preparación

---

##### Tarea: DOMF003 — Suite de tests unitarios del dominio del Game Service

**Microservicio:** `battlecaos-game` → capa `src/domain/` (tests en `src/domain/__tests__/`)

**Qué hace:** Escribe tests unitarios para todas las funciones puras del dominio. Objetivo: cobertura ≥ 85% en `game/domain/`.

**Scaffolding:**
```
server/game/domain/__tests__/
├── engine.test.js
├── board.test.js
├── fleet.test.js
├── energy.test.js
└── powers.test.js  (cuando estén implementados)
```

**Instalar dependencias de testing:**
```bash
npm install --save-dev vitest @vitest/coverage-v8
```

**Añadir scripts en `package.json`:**
```json
{
  "scripts": {
    "test":     "vitest run",
    "test:cov": "vitest run --coverage"
  }
}
```

**Ejemplo de test:**
```js
// server/game/domain/__tests__/board.test.js
import { describe, it, expect } from 'vitest';
import { createBoard, placeShip, shoot } from '../board.js';

describe('board', () => {
  it('placeShip coloca barco horizontal', () => {
    const board = createBoard();
    placeShip(board, { id: 'destructor', size: 2, x: 0, y: 0, horizontal: true });
    expect(board.cells['0,0']).toBe('ship');
    expect(board.cells['1,0']).toBe('ship');
  });

  it('shoot devuelve hit en celda con barco', () => {
    const board = createBoard();
    placeShip(board, { id: 'destructor', size: 2, x: 3, y: 3, horizontal: true });
    const result = shoot(board, 3, 3);
    expect(result.valid).toBe(true);
    expect(result.hit).toBe(true);
  });

  it('shoot devuelve miss en celda vacía', () => {
    const board = createBoard();
    const result = shoot(board, 5, 5);
    expect(result.hit).toBe(false);
  });

  it('shoot en celda ya disparada devuelve valid: false', () => {
    const board = createBoard();
    shoot(board, 1, 1);
    const result = shoot(board, 1, 1);
    expect(result.valid).toBe(false);
  });
});
```

---

### Fase 4: Poderes y Contramedida

---

#### Historia 18 — DOMH10: Uso de poderes mediante energía acumulada

---

##### Tarea: DOMF801 — Poderes ofensivos: bombardeo de área y detección (Sonar)

**Microservicio:** `battlecaos-game` → `src/domain/powers.js` + `src/handlers/handlePower.js`

**Qué hace:** Implementa los poderes **Bombardeo 3×3** (costo 2E, dispara en área) y **Sonar** (costo 2E, revela fila o columna del tablero rival).

**Scaffolding:**
```
server/game/
├── domain/powers.js
└── handlers/handlePower.js
```

**Cómo implementarlo:**

```js
// server/game/domain/powers.js — puro
import { DomainError } from './errors.js';
import { hasEnough } from './energy.js';

export const POWER_COSTS = {
  bombardeo: 2,
  sonar:     2,
  escudo:    1,
  tormenta:  3,
  contramedida: 0,
};

export function validatePower(sala, playerId, powerType, energia) {
  if (sala.fase !== 'TURNOS') throw new DomainError('fase_incorrecta', 'Solo en fase TURNOS');
  if (sala.turno?.jugadorActual !== playerId)
    throw new DomainError('no_es_tu_turno', 'No es tu turno');
  const cost = POWER_COSTS[powerType];
  if (cost === undefined) throw new DomainError('poder_invalido', `Poder ${powerType} no existe`);
  if (!hasEnough(energia, cost)) throw new DomainError('energia_insuficiente', 'No tienes suficiente energía');
  return { cost };
}

export function applyBombardeo(board, centerX, centerY) {
  const results = [];
  for (let dx = -1; dx <= 1; dx++) {
    for (let dy = -1; dy <= 1; dy++) {
      const x = centerX + dx, y = centerY + dy;
      if (x >= 0 && y >= 0 && x < board.size && y < board.size) {
        const key = `${x},${y}`;
        const hit = board.cells[key] === 'ship';
        if (board.cells[key] !== 'hit' && board.cells[key] !== 'miss') {
          board.cells[key] = hit ? 'hit' : 'miss';
        }
        results.push({ x, y, hit });
      }
    }
  }
  return results;
}

export function applySonar(board, axis, index) {
  // axis: 'row' | 'col'
  const cells = [];
  for (let i = 0; i < board.size; i++) {
    const [x, y] = axis === 'row' ? [i, index] : [index, i];
    const key = `${x},${y}`;
    if (board.cells[key] === 'ship') cells.push({ x, y });
  }
  return cells; // revela posiciones de barcos en esa fila/columna
}
```

```js
// server/game/handlers/handlePower.js
import { redis } from '../index.js';
import { validatePower, applyBombardeo, applySonar, POWER_COSTS } from '../domain/powers.js';
import { broadcastState } from './broker.js';

export async function handlePower({ codigo, playerId, powerType, target }) {
  const sala   = JSON.parse(await redis.get(`sala:${codigo}`));
  const jugador = sala.jugadores.find((j) => j.id === playerId);
  const rival   = sala.jugadores.find((j) => j.equipo !== jugador.equipo);
  const energia = parseInt(await redis.get(`sala:${codigo}:energia:${jugador.equipo}`) ?? '0');

  try {
    const { cost } = validatePower(sala, playerId, powerType, energia);

    let result;
    if (powerType === 'bombardeo') result = applyBombardeo(sala.tableros[rival.id], target.x, target.y);
    if (powerType === 'sonar')     result = applySonar(sala.tableros[rival.id], target.axis, target.index);

    // Descontar energía
    await redis.incrby(`sala:${codigo}:energia:${jugador.equipo}`, -cost);
    await redis.set(`sala:${codigo}`, JSON.stringify(sala));

    await redis.publish('evt:PowerUsed', JSON.stringify({
      type: 'PowerUsed', source: 'game', timestamp: Date.now(),
      data: { codigo, playerId, powerType, target, result },
    }));

    await broadcastState(codigo);
  } catch (err) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: playerId, event: 'game:error', payload: { error: err.code ?? err.message },
    }));
  }
}
```

---

##### Tarea: DOMF802 — Poderes defensivos: Escudo y Tormenta

**Microservicio:** `battlecaos-game` → ampliar `src/domain/powers.js` + `src/handlers/handlePower.js`

**Qué hace:** Implementa **Escudo** (costo 1E, protege una celda del siguiente disparo) y **Tormenta** (costo 3E, bloquea el turno del rival, una vez por partida).

**Cómo implementarlo (ampliar powers.js):**

```js
// Añadir en domain/powers.js:
export function applyEscudo(sala, playerId, x, y) {
  sala.escudos ??= {};
  sala.escudos[`${playerId}:${x},${y}`] = true;
  return { protegido: true, x, y };
}

export function applyTormenta(sala, playerId) {
  if (sala.tormentaUsada?.[playerId])
    throw new DomainError('tormenta_ya_usada', 'Solo puedes usar Tormenta una vez por partida');
  sala.tormentaUsada ??= {};
  sala.tormentaUsada[playerId] = true;

  // Saltar el turno del rival
  const idx = sala.jugadores.findIndex((j) => j.id === sala.turno.jugadorActual);
  sala.turno.jugadorActual = sala.jugadores[(idx + 2) % sala.jugadores.length].id;
  return { turnoSaltado: true };
}
```

---

#### Historia 19 — UIUXH09: Panel de poderes claro

---

##### Tarea: UIUXF801 — Panel de poderes con costo y selección de objetivo

**Microservicio:** Frontend

**Scaffolding:**
```
src/components/PowerPanel/
├── PowerPanel.jsx
└── PowerPanel.module.css
```

```jsx
const POWERS = [
  { id: 'bombardeo', label: 'Bombardeo 3×3', cost: 2, needsTarget: true },
  { id: 'sonar',     label: 'Sonar',          cost: 2, needsTarget: true },
  { id: 'escudo',    label: 'Escudo',          cost: 1, needsTarget: true },
  { id: 'tormenta',  label: 'Tormenta',        cost: 3, needsTarget: false },
];

export default function PowerPanel({ socket, codigo, energia }) {
  const [selected, setSelected] = useState(null);

  function usePower(power) {
    if (energia < power.cost) return;
    if (power.needsTarget) { setSelected(power.id); return; } // activar modo selección en tablero
    socket.emit('poder:usar', { codigo, powerType: power.id, target: null });
  }

  return (
    <aside>
      <h3>Poderes — {energia}E</h3>
      {POWERS.map((p) => (
        <button key={p.id} disabled={energia < p.cost} onClick={() => usePower(p)}
                className={selected === p.id ? styles.active : ''}>
          {p.label} ({p.cost}E)
        </button>
      ))}
    </aside>
  );
}
```

---

#### Historia 20 — DOMH11: Ventana de reacción para anular un poder

---

##### Tarea: DOMF901 — Ventana de reacción, anulación y devolución de energía

**Microservicio:** `battlecaos-game` → `src/domain/countermeasure.js` + ampliar `src/handlers/handlePower.js`

**Qué hace:** Cuando se activa un poder ofensivo (`bombardeo`, `sonar`), el rival tiene **5 segundos** para usar Contramedida. Si la activa, anula el poder y el atacante pierde la energía (no se devuelve).

**Scaffolding:**
```
server/game/domain/countermeasure.js
```

**Cómo implementarlo:**

```js
// server/game/domain/countermeasure.js — puro
export function canCountermeasure(sala, playerId, powerType) {
  const COUNTERABLES = ['bombardeo', 'sonar'];
  if (!COUNTERABLES.includes(powerType)) return false;
  const jugador = sala.jugadores.find((j) => j.id === playerId);
  // Contramedida es del rival del atacante
  return jugador && sala.contramedidaActiva?.targetEquipo === jugador.equipo;
}
```

```js
// En handlePower.js, tras aplicar poder ofensivo:
// Abrir ventana de 5s para contramedida
sala.contramedidaActiva = {
  powerType,
  atacanteId: playerId,
  targetEquipo: rival.equipo,
  expiraEn: Date.now() + 5000,
};
await redis.set(`sala:${codigo}`, JSON.stringify(sala));

// Notificar al rival de la ventana de contramedida
await redis.publish('gw:broadcast', JSON.stringify({
  roomId: codigo,
  event: 'poder:contramedida-disponible',
  payload: { powerType, remaining: 5000 },
}));

// Timer de 5s: si no hubo respuesta, aplicar el poder
setTimeout(async () => {
  const salaActual = JSON.parse(await redis.get(`sala:${codigo}`));
  if (salaActual.contramedidaActiva?.atacanteId === playerId) {
    // No hubo contramedida — aplicar el poder efectivamente
    salaActual.contramedidaActiva = null;
    await redis.set(`sala:${codigo}`, JSON.stringify(salaActual));
    await broadcastState(codigo);
  }
}, 5000);

// En broker.js, añadir handler para 'contramedida:activar':
// Si el rival activa dentro de los 5s → cancelar poder y publicar PowerCompensated
```

---

#### Historia 21 — UIUXH10: Aviso visible de ventana de reacción

---

##### Tarea: UIUXF901 — Aviso de ventana de reacción con cuenta regresiva

**Microservicio:** Frontend

```jsx
// src/components/CountermeasureAlert/CountermeasureAlert.jsx
import { useState, useEffect } from 'react';

export default function CountermeasureAlert({ socket, codigo }) {
  const [alert, setAlert] = useState(null);
  const [remaining, setRemaining] = useState(0);

  useEffect(() => {
    if (!socket) return;
    socket.on('poder:contramedida-disponible', ({ powerType, remaining }) => {
      setAlert(powerType);
      setRemaining(remaining);
      const interval = setInterval(() => {
        setRemaining((r) => {
          if (r <= 100) { clearInterval(interval); setAlert(null); return 0; }
          return r - 100;
        });
      }, 100);
    });
  }, [socket]);

  if (!alert) return null;
  return (
    <div className={styles.alert} role="alert">
      ¡Contramedida disponible! ({(remaining / 1000).toFixed(1)}s)
      <button onClick={() => {
        socket.emit('contramedida:activar', { codigo });
        setAlert(null);
      }}>
        Activar
      </button>
    </div>
  );
}
```

---

### Fase 5: Salva, Bot y Timeouts

---

#### Historia 22 — DOMH12: Ronda de disparo simultáneo

---

##### Tarea: DOMF1001 — Ventana de disparo simultáneo

**Microservicio:** `battlecaos-game` → `src/domain/salvo.js` + `src/handlers/handleSalvo.js`

**Qué hace:** En fase `SALVA`, todos los jugadores tienen 8 segundos para disparar simultáneamente. Registra cada disparo en Redis y los resuelve cuando vence el timer.

**Scaffolding:**
```
server/game/
├── domain/salvo.js
└── handlers/handleSalvo.js
```

**Cómo implementarlo:**

```js
// server/game/domain/salvo.js — puro
export function collectShot(salvoState, playerId, x, y) {
  if (salvoState.disparos[playerId]) return { ok: false, error: 'ya_disparaste' };
  salvoState.disparos[playerId] = { x, y, ts: Date.now() };
  return { ok: true };
}

export function resolveSalvo(salvoState, tableros, jugadores) {
  const resultados = {};
  for (const [pId, shot] of Object.entries(salvoState.disparos)) {
    const atacante = jugadores.find((j) => j.id === pId);
    const rival    = jugadores.find((j) => j.equipo !== atacante.equipo);
    const board    = tableros[rival.id];
    const key      = `${shot.x},${shot.y}`;
    const hit      = board.cells[key] === 'ship';
    if (board.cells[key] !== 'hit' && board.cells[key] !== 'miss') {
      board.cells[key] = hit ? 'hit' : 'miss';
    }
    resultados[pId] = { x: shot.x, y: shot.y, hit };
  }
  return resultados;
}
```

---

##### Tarea: DOMF1002 — Resolución exclusiva por casilla y control de cadencia

**Microservicio:** `battlecaos-game` → ampliar `src/handlers/handleSalvo.js`

**Qué hace:** Usa `SETNX` de Redis para garantizar que si dos jugadores apuntan a la misma celda, solo uno la resuelve. Previene condiciones de carrera en el modo 2v2.

**Cómo implementarlo:**

```js
// server/game/handlers/handleSalvo.js
import { redis } from '../index.js';
import { collectShot, resolveSalvo } from '../domain/salvo.js';
import { broadcastState } from './broker.js';

export async function handleSalvo({ codigo, playerId, x, y }) {
  // Lock atómico por celda: solo el primero gana
  const lockKey = `sala:${codigo}:salva:${x}:${y}`;
  const acquired = await redis.setnx(lockKey, playerId);
  if (!acquired) {
    await redis.publish('gw:broadcast', JSON.stringify({
      roomId: playerId, event: 'game:error', payload: { error: 'celda_ya_tomada' },
    }));
    return;
  }
  // TTL de 10s para que el lock expire si el jugador abandona
  await redis.expire(lockKey, 10);

  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  sala.salvoActual ??= { disparos: {} };
  const result = collectShot(sala.salvoActual, playerId, x, y);
  if (!result.ok) return;

  await redis.set(`sala:${codigo}`, JSON.stringify(sala));
  await broadcastState(codigo);

  // Si todos los jugadores dispararon, resolver la salva ya
  const todosDispararon = sala.jugadores.every((j) => sala.salvoActual.disparos[j.id]);
  if (todosDispararon) await resolveAllSalvo(codigo, sala);
}

async function resolveAllSalvo(codigo, sala) {
  const resultados = resolveSalvo(sala.salvoActual, sala.tableros, sala.jugadores);
  sala.salvoActual = null;
  sala.fase = 'TURNOS';
  sala.turno = { jugadorActual: sala.jugadores[0].id, numeroTurno: sala.turno.numeroTurno + 1 };
  await redis.set(`sala:${codigo}`, JSON.stringify(sala));

  await redis.publish('evt:PhaseChanged', JSON.stringify({
    type: 'PhaseChanged', source: 'game', timestamp: Date.now(),
    data: { codigo, from: 'SALVA', to: 'TURNOS' },
  }));
  await broadcastState(codigo);
}
```

---

#### Historia 23 — UIUXH11: Interfaz clara durante la ronda simultánea

---

##### Tarea: UIUXF1001 — Interfaz de la ronda simultánea con cadencia y retroalimentación

**Microservicio:** Frontend

**Qué hace:** Durante la fase SALVA, muestra un banner de cuenta regresiva (8s) y permite disparar exactamente una vez. Bloquea el tablero tras disparar hasta que resuelve la salva.

```jsx
// En GamePage.jsx, añadir lógica de SALVA:
const [salvaDisparado, setSalvaDisparado] = useState(false);

function handleCellClick(x, y) {
  if (gameState?.fase === 'SALVA' && !salvaDisparado) {
    socket.emit('salva:disparo', { codigo, x, y });
    setSalvaDisparado(true);
  } else if (gameState?.fase === 'TURNOS') {
    socket.emit('disparo:realizar', { codigo, x, y });
  }
}

// Resetear cuando vuelve a TURNOS
useEffect(() => {
  if (gameState?.fase === 'TURNOS') setSalvaDisparado(false);
}, [gameState?.fase]);
```

---

#### Historia 24 — DOMH13: Oponente automático para jugar en solitario

---

##### Tarea: DOMF1101 — Oponente automático con disparo y remate básico

**Microservicio:** `battlecaos-bot` → `src/index.js`

**Qué hace:** Implementa el algoritmo del bot. Fase 1 (random): dispara a una celda aleatoria no disparada. Fase 2 (hunt): si hubo impacto, continúa en celdas adyacentes hasta hundir.

**Scaffolding:**
```
server/bot/index.js
```

**Cómo implementarlo:**

```js
// server/bot/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

const redis = createRedis();
const sub   = createRedis();

function decidirDisparo(historialDisparos, ultimoImpacto) {
  // Hunt mode: si hubo impacto sin hundir, disparar adyacente
  if (ultimoImpacto) {
    const { x, y } = ultimoImpacto;
    const candidatos = [[x+1,y],[x-1,y],[x,y+1],[x,y-1]]
      .filter(([cx,cy]) => cx >= 0 && cy >= 0 && cx < 10 && cy < 10)
      .filter(([cx,cy]) => !historialDisparos.has(`${cx},${cy}`));
    if (candidatos.length) {
      const [bx, by] = candidatos[Math.floor(Math.random() * candidatos.length)];
      return { x: bx, y: by };
    }
  }
  // Random mode
  let x, y;
  do {
    x = Math.floor(Math.random() * 10);
    y = Math.floor(Math.random() * 10);
  } while (historialDisparos.has(`${x},${y}`));
  return { x, y };
}

export { decidirDisparo };
```

---

##### Tarea: DOMF1102 — Integración del oponente automático con el bus de eventos

**Microservicio:** `battlecaos-bot` → ampliar `src/index.js`

**Qué hace:** Suscribe al Bot Service a `ShotFired` y `PhaseChanged`. Cuando detecta que es el turno del bot, calcula el disparo y publica `BotDecision`.

**Cómo implementarlo:**

```js
// Añadir en server/bot/index.js:
import { decidirDisparo } from './index.js'; // o en el mismo archivo

const historial = {}; // { [codigo]: Set<"x,y"> }
const ultimoImpacto = {}; // { [codigo]: { x, y } | null }

sub.subscribe('evt:ShotFired', 'evt:PhaseChanged');

sub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);
  const { codigo } = msg.data;

  if (channel === 'evt:ShotFired') {
    const sala = JSON.parse(await redis.get(`sala:${codigo}`));
    if (sala?.modo !== '1v1-bot') return;

    const { x, y, result, playerId } = msg.data;
    historial[codigo] ??= new Set();
    historial[codigo].add(`${x},${y}`);

    const esBot = sala.jugadores.find((j) => j.id === playerId)?.esBot;
    if (!esBot && result === 'hit') ultimoImpacto[codigo] = { x, y };
    if (!esBot && result === 'sunk') ultimoImpacto[codigo] = null;

    // Si ahora es el turno del bot, disparar
    if (sala.turno?.jugadorActual === 'bot') await dispararComoBot(codigo, sala);
  }

  if (channel === 'evt:PhaseChanged') {
    const sala = JSON.parse(await redis.get(`sala:${codigo}`));
    if (sala?.modo === '1v1-bot' && sala.turno?.jugadorActual === 'bot') {
      await dispararComoBot(codigo, sala);
    }
  }
});

async function dispararComoBot(codigo, sala) {
  historial[codigo] ??= new Set();
  const shot = decidirDisparo(historial[codigo], ultimoImpacto[codigo]);
  historial[codigo].add(`${shot.x},${shot.y}`);

  await redis.publish('evt:BotDecision', JSON.stringify({
    type: 'BotDecision', source: 'bot', timestamp: Date.now(),
    data: { codigo, playerId: 'bot', x: shot.x, y: shot.y },
  }));
}
```

---

#### Historia 25 — DOMH15: Continuidad ante inactividad o desconexión

---

##### Tarea: DOMF1301 — Límite de tiempo de turno y abandono definitivo por desconexión prolongada

**Microservicio:** `battlecaos-game` → `src/handlers/handleDisconnect.js`

**Qué hace:** Al recibir `PlayerDisconnectedFromRoom`, si era el turno del jugador lo **pausa inmediatamente** (no lo pasa). Arranca un timeout (ej. 60s). Si reconecta antes, reanuda. Si no, pasa el turno o elimina al jugador según el modo.

**Scaffolding:**
```
server/game/handlers/handleDisconnect.js
```

**Cómo implementarlo:**

```js
// server/game/handlers/handleDisconnect.js
import { redis } from '../index.js';
import { broadcastState } from './broker.js';

const DISCONNECT_TIMEOUT_MS = 60_000;
const pendingTimeouts = {}; // { [codigo:playerId]: NodeJS.Timeout }

export async function handleDisconnect({ codigo, playerId }) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  if (!sala || sala.fase === 'FIN') return;

  const esSuTurno = sala.turno?.jugadorActual === playerId;

  // Marcar turno en pausa SIN pasarlo
  if (esSuTurno) {
    sala.turno.pausado = true;
    sala.turno.pausadoPor = playerId;
    await redis.set(`sala:${codigo}`, JSON.stringify(sala));
    await broadcastState(codigo);
  }

  // Arrancar timeout de abandono
  const key = `${codigo}:${playerId}`;
  pendingTimeouts[key] = setTimeout(async () => {
    delete pendingTimeouts[key];
    const salaActual = JSON.parse(await redis.get(`sala:${codigo}`));
    if (!salaActual) return;

    const jugador = salaActual.jugadores.find((j) => j.id === playerId);
    if (jugador?.conectado) return; // reconectó antes de que venciera

    // Tiempo agotado: pasar el turno al siguiente jugador
    if (salaActual.turno?.pausadoPor === playerId) {
      const idx = salaActual.jugadores.findIndex((j) => j.id === playerId);
      salaActual.turno.jugadorActual = salaActual.jugadores[(idx + 1) % salaActual.jugadores.length].id;
      salaActual.turno.pausado = false;
      salaActual.turno.pausadoPor = null;
    }
    await redis.set(`sala:${codigo}`, JSON.stringify(salaActual));
    await broadcastState(codigo);
  }, DISCONNECT_TIMEOUT_MS);
}

export async function handleReconnect({ codigo, playerId }) {
  // Cancelar timeout si aún está pendiente
  const key = `${codigo}:${playerId}`;
  if (pendingTimeouts[key]) {
    clearTimeout(pendingTimeouts[key]);
    delete pendingTimeouts[key];
  }

  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  if (!sala) return;

  // Reanudar turno si estaba pausado por este jugador
  if (sala.turno?.pausadoPor === playerId) {
    sala.turno.pausado = false;
    sala.turno.pausadoPor = null;
    await redis.set(`sala:${codigo}`, JSON.stringify(sala));
    await broadcastState(codigo);
  }
}
```

---

##### Tarea: DOMF1302 — Verificación de disponibilidad previa a la operación

**Microservicio:** `battlecaos-gateway`, `battlecaos-auth`, `battlecaos-room`, `battlecaos-chat`, `battlecaos-bot`

> Los servicios sin HTTP (`battlecaos-game`, `battlecaos-timer`, `battlecaos-observability`) no exponen `/health` HTTP, pero pueden verificar su estado con `redis.ping()` internamente.

**Qué hace:** Ya implementado en DOMF001 como `GET /health`. Esta tarea lo extiende para incluir el estado de Redis en la respuesta.

```js
// Versión mejorada del health check en todos los servicios:
app.get('/health', async (_req, res) => {
  try {
    const pingResult = await redis.ping(); // devuelve 'PONG'
    res.json({
      service: 'nombre-del-servicio',
      status: 'ok',
      redis: pingResult === 'PONG' ? 'ok' : 'degraded',
      uptime: process.uptime(),
      timestamp: Date.now(),
    });
  } catch (err) {
    res.status(503).json({ service: 'nombre', status: 'error', redis: 'down', error: err.message });
  }
});
```

---

##### Tarea: DOMF1303 — Timer Service: elección de líder, heartbeat y gestión de temporizadores

**Microservicio:** `battlecaos-timer` → `src/index.js`

**Qué hace:** Implementa leader election con `SETNX` + TTL=1s y heartbeat cada 500ms. El líder activo ejecuta los timers de cada fase y publica `TimerTick` cada 100ms y `TimerEnd` al expirar.

**Scaffolding:**
```
server/timer/index.js
```

**Cómo implementarlo:**

```js
// server/timer/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

const redis  = createRedis();
const sub    = createRedis();
const ME     = `timer-${process.pid}`;

// ── Leader Election ──────────────────────────────────────────
let isLeader = false;

async function tryBecomeLeader() {
  const acquired = await redis.set('timer:leader', ME, 'NX', 'EX', 1);
  if (acquired) isLeader = true;
}

setInterval(async () => {
  if (isLeader) {
    await redis.set('timer:leader', ME, 'XX', 'EX', 1); // renovar TTL (heartbeat)
  } else {
    await tryBecomeLeader();
  }
}, 500);

// ── Gestión de timers por sala ───────────────────────────────
const DURATIONS = { COLOCACION: 60_000, TURNO: 30_000, SALVA: 8_000, CONTRAMEDIDA: 5_000 };
const activeTimers = {}; // { [codigo:tipo]: { interval, timeout } }

function startTimer(codigo, tipo) {
  if (!isLeader) return;
  const duration = DURATIONS[tipo];
  if (!duration) return;

  stopTimer(codigo, tipo);

  const start     = Date.now();
  const intervalId = setInterval(async () => {
    const remaining = duration - (Date.now() - start);
    await redis.publish('evt:TimerTick', JSON.stringify({
      type: 'TimerTick', source: 'timer', timestamp: Date.now(),
      data: { codigo, tipo, remaining: Math.max(0, remaining) },
    }));
  }, 100);

  const timeoutId = setTimeout(async () => {
    clearInterval(intervalId);
    delete activeTimers[`${codigo}:${tipo}`];
    await redis.publish('evt:TimerEnd', JSON.stringify({
      type: 'TimerEnd', source: 'timer', timestamp: Date.now(),
      data: { codigo, tipo },
    }));
  }, duration);

  activeTimers[`${codigo}:${tipo}`] = { intervalId, timeoutId };
}

function stopTimer(codigo, tipo) {
  const key = `${codigo}:${tipo}`;
  if (activeTimers[key]) {
    clearInterval(activeTimers[key].intervalId);
    clearTimeout(activeTimers[key].timeoutId);
    delete activeTimers[key];
  }
}

// ── Suscripciones ────────────────────────────────────────────
sub.subscribe('evt:RoomReady', 'evt:PhaseChanged');

sub.on('message', async (_ch, raw) => {
  const msg = JSON.parse(raw);
  const { codigo } = msg.data;

  if (msg.type === 'RoomReady')    startTimer(codigo, 'COLOCACION');
  if (msg.type === 'PhaseChanged') {
    const { to } = msg.data;
    if (to === 'TURNOS')   startTimer(codigo, 'TURNO');
    if (to === 'SALVA')    startTimer(codigo, 'SALVA');
    if (to === 'FIN')      Object.keys(activeTimers).filter(k => k.startsWith(codigo)).forEach(k => {
      const [, tipo] = k.split(':');
      stopTimer(codigo, tipo);
    });
  }
});

console.log(`[timer] ${ME} iniciado`);
```

> **Failover:** Si el líder muere, su TTL expira en 1s. Otra instancia hace `SETNX` en el siguiente heartbeat (500ms) y se convierte en líder. El failover total es ~1.5s.

---

### Fase 6: Observabilidad, Cierre y Documentación

---

#### Historia 26 — DOMH14: Visibilidad del comportamiento del sistema

---

##### Tarea: DOMF1201 — Registro estructurado de eventos clave

**Microservicio:** `battlecaos-observability` → `src/index.js`

**Qué hace:** Suscribe a todos los canales de eventos y registra cada evento con timestamp para el audit trail.

**Scaffolding:**
```
server/observability/index.js
```

**Cómo implementarlo:**

```js
// server/observability/index.js
import 'dotenv/config';
import { createRedis } from '../shared/redis.js';

const redis = createRedis();
const sub   = createRedis();

const ALL_EVENTS = [
  'evt:RoomReady', 'evt:PlayerDisconnectedFromRoom', 'evt:PlayerReconnected',
  'evt:RoomDestroyed', 'evt:PlayerRoomJoined', 'evt:GameStarted', 'evt:ShotFired',
  'evt:PhaseChanged', 'evt:PowerUsed', 'evt:PowerCompensated', 'evt:GameEnded',
  'evt:ShipsPlaced', 'evt:ShipSunk', 'evt:TimerEnd', 'evt:BotDecision', 'evt:ChatMessage',
];

sub.subscribe(...ALL_EVENTS);

sub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);

  // Audit trail: guardar en lista por sala
  if (msg.data?.codigo) {
    await redis.lpush(
      `audit:${msg.data.codigo}`,
      JSON.stringify({ channel, ...msg, receivedAt: Date.now() })
    );
    await redis.ltrim(`audit:${msg.data.codigo}`, 0, 999); // máximo 1000 eventos por sala
  }

  console.log(`[obs] ${msg.type} @ ${msg.data?.codigo ?? 'global'}`);
});
```

---

##### Tarea: DOMF1202 — Métricas técnicas de concurrencia y rendimiento

**Microservicio:** `battlecaos-observability` → ampliar `src/index.js`

**Qué hace:** Mide la latencia P95 de disparos (tiempo entre `ShotFired` recibido y resultado) y el pico de salas concurrentes.

**Cómo implementarlo:**

```js
// Añadir en observability/index.js:
const shotTimestamps = {}; // { [correlationId]: number }

sub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);

  if (msg.type === 'ShotFired') {
    const latencia = Date.now() - msg.timestamp;
    await redis.lpush('metrics:latencia:shots', latencia);
    await redis.ltrim('metrics:latencia:shots', 0, 999); // últimas 1000

    // Pico de salas activas
    const salasActivas = await redis.keys('sala:*');
    const soloSalas = salasActivas.filter(k => !k.includes(':')).length;
    const picoActual = parseInt(await redis.get('metrics:pico:salas') ?? '0');
    if (soloSalas > picoActual) await redis.set('metrics:pico:salas', soloSalas);
  }
});
```

---

##### Tarea: DOMF1203 — Cálculo de los indicadores de negocio (KPIs)

**Microservicio:** `battlecaos-observability` → ampliar `src/index.js`

**Qué hace:** Calcula y almacena los 4 KPIs del proyecto en Redis, actualizados en tiempo real.

**KPIs:**
1. `% partidas completadas` = GameEnded / GameStarted
2. `Latencia P95` de resolución de disparos
3. `% reconexiones exitosas` = PlayerReconnected / PlayerDisconnectedFromRoom
4. `Pico de salas concurrentes`

**Cómo implementarlo:**

```js
sub.on('message', async (channel, raw) => {
  const msg = JSON.parse(raw);
  const today = new Date().toISOString().split('T')[0];

  if (msg.type === 'GameStarted') {
    await redis.hincrby(`stats:kpi:${today}`, 'games_started', 1);
  }
  if (msg.type === 'GameEnded') {
    await redis.hincrby(`stats:kpi:${today}`, 'games_ended', 1);
  }
  if (msg.type === 'PlayerDisconnectedFromRoom') {
    await redis.hincrby(`stats:kpi:${today}`, 'disconnections', 1);
  }
  if (msg.type === 'PlayerReconnected') {
    await redis.hincrby(`stats:kpi:${today}`, 'reconnections', 1);
  }
});

// Leer KPIs (exponer como endpoint o via Redis directamente):
async function getKPIs(date = new Date().toISOString().split('T')[0]) {
  const raw = await redis.hgetall(`stats:kpi:${date}`);
  const started = parseInt(raw.games_started ?? 0);
  const ended   = parseInt(raw.games_ended   ?? 0);
  const disc    = parseInt(raw.disconnections ?? 0);
  const recon   = parseInt(raw.reconnections  ?? 0);

  return {
    tasa_completacion: started ? ((ended / started) * 100).toFixed(1) + '%' : 'N/A',
    tasa_reconexion:   disc    ? ((recon / disc)    * 100).toFixed(1) + '%' : 'N/A',
    pico_salas: await redis.get('metrics:pico:salas') ?? 0,
  };
}
```

---

#### Historia 27 — UIUXH12: Panel de indicadores para el operador

---

##### Tarea: UIUXF1101 — Panel mínimo de indicadores de negocio

**Microservicio:** Frontend

**Qué hace:** Pantalla de admin (ruta `/admin`) que muestra los 4 KPIs en tiempo real consumiéndolos del Gateway.

```jsx
// src/pages/AdminPage.jsx
import { useState, useEffect } from 'react';
const GATEWAY = import.meta.env.VITE_GATEWAY_URL ?? 'http://localhost:3000';

export default function AdminPage() {
  const [kpis, setKpis] = useState(null);
  useEffect(() => {
    fetch(`${GATEWAY}/kpis`).then(r => r.json()).then(setKpis);
    const id = setInterval(() =>
      fetch(`${GATEWAY}/kpis`).then(r => r.json()).then(setKpis), 5000);
    return () => clearInterval(id);
  }, []);
  if (!kpis) return <p>Cargando métricas...</p>;
  return (
    <main>
      <h2>Panel de observabilidad</h2>
      <dl>
        <dt>Partidas completadas</dt><dd>{kpis.tasa_completacion}</dd>
        <dt>Reconexiones exitosas</dt><dd>{kpis.tasa_reconexion}</dd>
        <dt>Pico de salas</dt><dd>{kpis.pico_salas}</dd>
      </dl>
    </main>
  );
}
```

```js
// En Gateway, añadir endpoint:
app.get('/kpis', async (_req, res) => {
  const kpis = await getKPIs(); // función del Observability Service (importar o duplicar)
  res.json(kpis);
});
```

---

#### Historia 28 — UIUXH13: Cierre de partida claro y manejo de errores

---

##### Tarea: UIUXF1201 — Pantalla de victoria y manejo visible de errores

**Microservicio:** Frontend

```jsx
// src/pages/ResultPage.jsx
import { useLocation, useNavigate } from 'react-router-dom';

export default function ResultPage() {
  const { state } = useLocation(); // { winner, modo, duracion }
  const navigate  = useNavigate();
  return (
    <main>
      <h1>{state?.winner === 'A' ? '¡Equipo A gana!' : '¡Equipo B gana!'}</h1>
      <p>Modo: {state?.modo} — Duración: {Math.round((state?.duracion ?? 0) / 1000)}s</p>
      <button onClick={() => navigate('/lobby')}>Nueva partida</button>
    </main>
  );
}
```

```jsx
// ErrorBoundary global en App.jsx:
import { Component } from 'react';
class ErrorBoundary extends Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  render() {
    if (this.state.error) return <p role="alert">Error inesperado: {this.state.error.message}</p>;
    return this.props.children;
  }
}
```

---

#### Historias 29–33 — DOC-2: Documentación

Las historias DOCH01–DOCH05 y sus tareas DOCF001–DOCF401 producen **diagramas y documentos**, no código. El artefacto de cada tarea es un archivo `.puml` (PlantUML) o `.md`.

| Tarea | Artefacto | Herramienta |
|---|---|---|
| DOCF001 | `docs/diagramas/despliegue.puml`, `componentes.puml` | PlantUML |
| DOCF101 | `docs/diagramas/secuencia-acceso.puml`, `contrato-eventos.md` | PlantUML + Markdown |
| DOCF201 | `docs/diagramas/clases-dominio.puml`, `fases-partida.puml` | PlantUML |
| DOCF301 | `docs/diagramas/secuencia-salva.puml`, `secuencia-contramedida.puml` | PlantUML |
| DOCF401 | `docs/demo-manual.md` | Markdown |

---

## Resumen de dependencias entre tareas

```
DOMF001 (init Redis)
  └─ DOMF002 (Gateway routing)
       └─ DOMF101 (Auth OAuth)
            └─ DOMF102 (JWT verify en Gateway)
                 └─ DOMF201 (crear/unirse sala)
                      └─ DOMF202 (aislamiento)
                           └─ DOMF203 (RoomReady, events)
                                └─ DOMF302 (fases)
                                     └─ DOMF501 (colocación flota)
                                          └─ DOMF601 (disparos por turno)
                                               ├─ DOMF602 (victoria)
                                               ├─ DOMF701 (energía)
                                               │    └─ DOMF801/802 (poderes)
                                               │         └─ DOMF901 (contramedida)
                                               └─ DOMF1002 (salva, locks)
                                                    └─ DOMF1303 (timer leader)
DOMF401/402/403 (reconexión) depende de DOMF201
DOMF1101/1102 (bot)          depende de DOMF601
DOMF1201-1203 (observ.)      depende de todos los eventos anteriores
DOMF003 (tests)              depende de DOMF501, DOMF601, DOMF701, DOMF801, DOMF901, DOMF1002
```
