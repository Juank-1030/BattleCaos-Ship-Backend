# battlecaos-frontend — Contexto para Claude

## ¿Qué hace este servicio?
Cliente web del juego. Se conecta al Gateway vía Socket.io con JWT para autenticación.
Renderiza los estados del juego como snapshots (nunca calcula estado local).
Incluye login con Google, lobby de salas, tablero interactivo, chat, poderes y resultado.

**Regla de oro:** el cliente NUNCA calcula estado de juego. Solo renderiza lo que recibe
del servidor en el evento `game:state`. El servidor es la única fuente de verdad.

## Puerto y protocolo
- **Dev:** `5173` (Vite dev server)
- **Build:** archivos estáticos para desplegar en Vercel
- **WebSocket:** conecta al Gateway en `VITE_GATEWAY_URL` (por defecto `http://localhost:3000`)

## Scaffolding completo

```
battlecaos-frontend/
├── .env.local                    ← variables reales (no commitear)
├── .env.example
├── .gitignore
├── package.json
├── vite.config.js
├── index.html
├── CLAUDE.md
└── src/
    ├── main.jsx
    ├── App.jsx                   ← React Router: 4 rutas
    ├── styles/
    │   └── tokens.css            ← variables CSS: colores, tamaños
    ├── hooks/
    │   ├── useSocket.js          ← conexión Socket.io con JWT
    │   └── useGameState.js       ← suscripción a game:state
    ├── pages/
    │   ├── LoginPage.jsx         ← Google OAuth
    │   ├── LobbyPage.jsx         ← crear/unirse a sala
    │   ├── GamePage.jsx          ← tablero + chat + poderes
    │   └── ResultPage.jsx        ← pantalla de victoria/derrota
    └── components/
        ├── Board/
        │   ├── Board.jsx         ← tablero N×N reutilizable
        │   └── Board.module.css
        ├── ShipPlacer/
        │   ├── ShipPlacer.jsx    ← colocación interactiva de flota
        │   └── ShipPlacer.module.css
        ├── EnergyBar/
        │   ├── EnergyBar.jsx     ← barra de energía del equipo
        │   └── EnergyBar.module.css
        ├── PowerPanel/
        │   ├── PowerPanel.jsx    ← botones de poderes especiales
        │   └── PowerPanel.module.css
        ├── Chat/
        │   ├── Chat.jsx          ← chat en tiempo real
        │   └── Chat.module.css
        ├── Timer/
        │   ├── Timer.jsx         ← cuenta regresiva de fase/turno
        │   └── Timer.module.css
        └── PlayerList/
            └── PlayerList.jsx    ← lista de jugadores en lobby
```

## Variables de entorno

```env
# .env.example
VITE_GATEWAY_URL=http://localhost:3000
VITE_AUTH_URL=http://localhost:3001
VITE_GOOGLE_CLIENT_ID=tu-client-id.apps.googleusercontent.com
```

## package.json

```json
{
  "name": "battlecaos-frontend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev":     "vite",
    "build":   "vite build",
    "preview": "vite preview",
    "test":    "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "react":            "^18.3.0",
    "react-dom":        "^18.3.0",
    "react-router-dom": "^6.23.0",
    "socket.io-client": "^4.7.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "vite":                 "^5.3.0",
    "vitest":               "^1.6.0",
    "@testing-library/react": "^16.0.0"
  }
}
```

## Rutas (React Router)

| Ruta | Componente | Descripción |
|---|---|---|
| `/` | `LoginPage` | Login con Google |
| `/lobby` | `LobbyPage` | Crear o unirse a sala |
| `/game` | `GamePage` | Tablero de juego |
| `/result` | `ResultPage` | Resultado final |
| `*` | Redirect `/` | Cualquier otra ruta |

## Tokens CSS (paleta naval)

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
  --color-energy:  #f4d03f;
  --cell-size:     2.5rem;
}
```

## Estados de celda del tablero

| Estado | Color | Descripción |
|---|---|---|
| `fog` | `--color-fog` | Celda no revelada |
| `ship` | `--color-ship` | Propio barco |
| `hit` | `--color-hit` | Impacto |
| `miss` | `--color-miss` | Agua |
| `sunk` | `--color-sunk` | Barco hundido |
| `selected` | `--color-select` | Celda seleccionada (colocación) |

## Eventos Socket.io que escucha el cliente

| Evento | Quién lo emite | Qué renderiza |
|---|---|---|
| `game:state` | Gateway | Estado completo → re-render de tableros |
| `room:created` | Gateway | Muestra código de sala |
| `room:joined` | Gateway | Actualiza lista de jugadores |
| `room:error` | Gateway | Muestra error al usuario |
| `chat:message` | Gateway | Nuevo mensaje en chat |
| `chat:history` | Gateway | Carga historial al entrar |
| `timer:tick` | Gateway | Actualiza cuenta regresiva |
| `shot:result` | Gateway | Animación hit/miss/sunk |
| `game:error` | Gateway | Error de acción de juego |

## Eventos Socket.io que emite el cliente

| Evento | Cuándo | Payload |
|---|---|---|
| `room:create` | Clic "Crear sala" | `{ modo }` |
| `room:join` | Clic "Unirse" | `{ codigo }` |
| `colocacion:set` | Confirmar flota | `{ codigo, ships }` |
| `disparo:realizar` | Clic en celda rival | `{ codigo, x, y }` |
| `salva:disparo` | Fase SALVA | `{ codigo, x, y }` |
| `poder:usar` | Clic en poder | `{ codigo, powerId, targetX, targetY }` |
| `contramedida:activar` | Clic en contramedida | `{ codigo }` |
| `chat:mensaje` | Enviar mensaje | `{ codigo, text }` |

## JWT — flujo de autenticación

1. Google Identity Services llama al callback con `{ credential: idToken }`
2. El cliente hace `POST /auth/google` con `{ idToken }`
3. Auth Service devuelve `{ token: jwtPropio }`
4. Guardar en `localStorage.setItem('token', jwtPropio)`
5. `useSocket(token)` conecta Socket.io con `auth: { token }`

## Testing (Vitest + Testing Library)

```bash
npm test
```

Cubrir principalmente: `Board.jsx` (renderizado de celdas), `EnergyBar.jsx` (props → width),
`useSocket.js` (mock de socket.io-client).

## Sistema de logs (desarrollo)
Usar `console.log` con prefijo para depuración:
```js
console.log('[socket] conectado:', socket.id);
console.log('[game:state]', gameState.fase, gameState.turno);
console.warn('[auth] token expirado — redirigiendo a login');
```

En producción: remover con `vite-plugin-strip-debug` o similar.

## Tareas pendientes (de Orden_de_Ejecucion.md)

| ID | Descripción | Sprint |
|---|---|---|
| UIUXF001 | App.jsx + 4 páginas + useSocket.js | 1 - Fase 0 |
| UIUXF002 | Board.jsx + tokens.css | 1 - Fase 0 |
| UIUXF101 | LoginPage con Google Identity Services | 1 - Fase 1 |
| UIUXF201 | LobbyPage: crear/unirse + PlayerList | 1 - Fase 1 |
| UIUXF301 | GamePage: render de tableros desde game:state | 1 - Fase 2 |
| UIUXF401 | Chat.jsx: mensajes en tiempo real + historial | 1 - Fase 2 |
| UIUXF501 | ShipPlacer.jsx: colocación interactiva | 1 - Fase 2 |
| UIUXF601 | Indicador de turno + animación hit/miss/sunk | 1 - Fase 2 |
| UIUXF701 | EnergyBar.jsx sincronizada con game:state | 1 - Fase 2 |
| UIUXF801 | Timer.jsx — cuenta regresiva via timer:tick | 2 - Fase 3 |
| UIUXF901 | PowerPanel.jsx — botones con coste de energía | 2 - Fase 3 |
| UIUXF1001 | ResultPage — victoria/derrota + stats | 2 - Fase 3 |
| UIUXF1101 | Pantalla de colocación con vista previa | 2 - Fase 3 |
| UIUXF1201 | Responsive y accesibilidad (ARIA roles) | 2 - Fase 3 |

## Arranque

```bash
cp .env.example .env.local   # rellenar VITE_GOOGLE_CLIENT_ID
npm install
npm run dev                  # http://localhost:5173
```
