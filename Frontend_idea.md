# Frontend_idea.md — Especificación completa para construir `battlecaos-frontend`

> **Para quién es este documento:** para una IA (o desarrollador) que va a construir el frontend
> **sin acceso al código del backend**. Por eso cada payload, cada evento y cada endpoint de aquí
> está escrito con la forma **exacta** verificada línea por línea contra el código real de
> `battlecaos-auth`, `battlecaos-gateway`, `battlecaos-room`, `battlecaos-game` y `battlecaos-timer`
> — no es una paráfrasis de la documentación de diseño (`Proyecto.md`, `desarrollo.md`,
> `CLAUDE.frontend.md`), que en varios puntos **no coincide** con lo que el backend realmente hace.
> Donde encontré esas discrepancias, las señalo explícitamente para que no se repitan en el frontend.
>
> Si algo de este documento entra en conflicto con otro documento del proyecto, **este documento
> gana** en todo lo referente a payloads/eventos/contratos (fue verificado contra código real), y
> `Orden_de_Ejecucion.md` gana en todo lo referente a qué tarea (`UIUXFxxx`) corresponde a qué pieza.

---

## 1. Resumen ejecutivo

**¿Repositorio propio?** Sí — `battlecaos-frontend`, en `github.com/BattleCaos-Ship/`. A diferencia
de los demás repos del proyecto, **no** es un proceso Node.js de servidor (no tiene Redis, no tiene
Kafka, no tiene Dockerfile, no expone `/health`). Es una SPA estática (Vite build) que en producción
se sirve desde un CDN y en tiempo de ejecución solo habla con el sistema por:
- **HTTP** → `battlecaos-auth`, una sola vez, para el login.
- **WebSocket (Socket.io)** → `battlecaos-gateway`, para absolutamente todo lo demás.

**Stack:**

| Herramienta | Para qué | Por qué |
|---|---|---|
| React 18 | UI | Ya asumido en toda la documentación previa del proyecto |
| Vite | Bundler/dev server | Elegido en `Orden_de_Ejecucion.md` (`battlecaos-frontend \| React + Vite \| 5173`) |
| react-router-dom v6 | Ruteo | Estándar de facto |
| socket.io-client | WebSocket | Obligatorio — el Gateway usa `socket.io`, no WebSocket nativo |
| CSS Modules | Estilos | Sin dependencias nuevas, viene con Vite |
| Vitest + Testing Library | Tests | Mismo runner que **todos** los repos backend del proyecto |
| Sin TypeScript | — | Ningún otro repo del proyecto usa TS |
| Sin Redux/Zustand/Context global | — | El estado real vive en el servidor (ver sección 2) |

**Regla de oro, la más importante de todo este documento:**
> El cliente **nunca** calcula estado de juego. Solo renderiza lo que llega en `game:state`, y emite
> la *intención* del usuario (quiero disparar aquí, quiero usar este poder) — nunca el resultado.

---

## 2. Contexto completo del juego — todas las mecánicas

### 2.1 Qué es BattleCaos-Ship

Batalla Naval multijugador en tiempo real, con equipos, poderes especiales alimentados por energía,
una ronda de fuego simultáneo ("Salva") y contramedidas para anular ataques del rival. El servidor
es siempre la autoridad.

### 2.2 Modos de juego y equipos

| Modo | Jugadores | Equipos |
|---|---|---|
| `1v1` | 2 humanos | 1 jugador por equipo (A y B) |
| `1v1-bot` | 1 humano + 1 bot | 1 vs 1, el bot ocupa el segundo slot |
| `2v2` | 4 humanos | 2 equipos de 2 (A y B) |

**Asignación de equipo es automática, no elegible por el usuario:** alterna por orden de entrada a
la sala — 1º y 3º jugador → equipo A; 2º y 4º → equipo B. El creador de la sala **siempre** es A.

Cada sala tiene un **código de 6 dígitos** generado por el servidor.

### 2.3 La flota — composición exacta

Tablero 10×10. Cada jugador coloca su propia flota de 5 barcos, sin superposición:

| Barco (`id` exacto a usar en el payload) | Tamaño (celdas) |
|---|:---:|
| `portaaviones` | 5 |
| `acorazado` | 4 |
| `crucero` | 3 |
| `submarino` | 3 |
| `destructor` | 2 |

Total: **17 celdas** a hundir para ganar. En 2v2, cada jugador tiene su propio tablero — no hay
tablero compartido de equipo.

### 2.4 Máquina de estados de fases

```
LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN
```

| Fase (valor exacto en `game:state.fase`) | Qué pasa | Duración / salida |
|---|---|---|
| `LOBBY` | Esperando que se llene la sala | Hasta que la sala se llena |
| `COLOCACION` | Cada jugador coloca su flota | 60s, o antes si todos confirman |
| `TURNOS` | Disparos por turnos, rotando | 30s por turno (se reinicia cada turno), hasta cumplir 3 turnos → abre `SALVA` |
| `SALVA` | Todos disparan a la vez | 8s fijos |
| `FIN` | Un equipo hundió toda la flota rival | — |

### 2.5 Turnos

- Rotación round-robin simple sobre el orden de entrada a la sala.
- **30 segundos por turno**, y el reloj se reinicia cada vez que cambia el jugador activo (no es un
  único cronómetro para toda la fase).
- Si no actúa a tiempo, el turno rota automáticamente.
- Cada 3 turnos completados se abre `SALVA` automáticamente — no hay botón para activarla manual.

### 2.6 Energía

- Es **por equipo** (compartida entre los 2 miembros en 2v2), no por jugador individual.
- +1 por impacto; +3 adicionales si ese impacto hunde (total +4 en el disparo que hunde). Un fallo no da nada.
- Se descuenta al **activar** un poder (no al resolverse — importa para la Contramedida, ver 2.8).
- También se gana disparando durante `SALVA`.

### 2.7 Los 5 poderes

| Poder (`powerType` exacto) | Costo | Objetivo (`target`) | Efecto | ¿Contrarrestable? |
|---|:---:|---|---|:---:|
| `bombardeo` | 2E | `{x, y}` del tablero **rival** (centro del área) | Dispara las 9 celdas de un área 3×3 | Sí |
| `sonar` | 2E | `{axis: 'row'\|'col', index: number}` del tablero **rival** | Revela si hay barco en toda esa fila/columna (no dispara) | Sí |
| `escudo` | 1E | `{x, y}` de **tu propio** tablero | Protege esa celda del próximo disparo que la apunte, por 2 turnos; si nadie ataca esa celda en ese lapso, expira sola | No |
| `tormenta` | 3E | ninguno (`target: null`) | El turno salta al jugador siguiente al inmediato (en 1v1 el mismo atacante repite turno; en 2v2 salta al rival y pasa al compañero de equipo) — una sola vez por partida por jugador | No |
| `contramedida` | 0E (gratis) | ninguno | Anula un Bombardeo/Sonar que el rival acaba de activar contra tu equipo | — |

Si `tormenta` se intenta usar dos veces por el mismo jugador, el servidor la rechaza — deshabilitar
el botón permanentemente tras el primer uso exitoso (ver tabla de errores, 3.7, código
`tormenta_ya_usada`).

### 2.8 Ventana de Contramedida (5 segundos)

Cuando el equipo rival activa `bombardeo` o `sonar` contra ti:
1. El servidor abre una ventana de **5 segundos exactos** (cronometrados en el servidor).
2. Tu equipo puede activar Contramedida durante esos 5s.
3. Se cierra por: (a) pasar los 5s sin reacción → el poder se aplica, o (b) activarla a tiempo → el poder se anula.
4. **El atacante NUNCA recupera la energía gastada**, se anule o no el poder.
5. El turno del atacante sigue su curso normal en paralelo — la ventana no lo pausa ni lo extiende.

### 2.9 Salva Simultánea (8 segundos)

- Se activa sola cada 3 turnos completados.
- Todos disparan a la vez, sin importar de quién "sería" el turno.
- Cada jugador puede disparar **varias veces** en la ventana, con cadencia mínima de **0.5 segundos**
  entre sus propios disparos consecutivos — no es un solo disparo, pero tampoco ráfaga libre.
- Si dos compañeros de equipo apuntan a la misma celda casi a la vez, solo se resuelve el primero.
- Se puede ganar/perder la partida en plena Salva.
- Al cerrarse (por tiempo), vuelve a `TURNOS`.

### 2.10 Desconexión y reconexión

- El jugador desconectado **no se expulsa** ni se borra su flota — sigue en la sala, marcado
  `conectado: false`.
- Si era su turno, se pausa inmediatamente (no se le pasa al siguiente todavía).
- Ventana de gracia de **60 segundos**: si reconecta antes, retoma donde estaba; si no, el turno rota
  automáticamente y el juego sigue sin él.
- El reloj de turno de 30s se pausa mientras está desconectado.

### 2.11 Condición de victoria

Un equipo gana cuando las 17 celdas de la flota rival están todas `hit`. Puede pasar en `TURNOS` o en plena `SALVA`.

### 2.12 Tabla de temporizadores

| Temporizador | Duración | Se reinicia |
|---|:---:|---|
| Colocación | 60s | Una vez, al entrar a `COLOCACION` |
| Turno | 30s | Cada vez que cambia el jugador activo |
| Salva | 8s | Una vez, al entrar a `SALVA` |
| Contramedida | 5s | Una vez, al activarse un poder ofensivo |

### 2.13 KPIs de negocio (para `AdminPage`)

| KPI | Fórmula |
|---|---|
| % partidas completadas | `partidas_terminadas / partidas_iniciadas` |
| % reconexiones exitosas | `reconexiones / desconexiones` |
| Pico de salas concurrentes | máximo histórico de salas activas a la vez |
| Latencia P95 de disparos | pendiente — depende de `battlecaos-observability`, aún no construido, mostrar "N/A" |

---

## 3. Contrato de comunicación con el backend (la parte más importante de este documento)

### 3.1 Login — HTTP

```
POST {VITE_AUTH_URL}/auth/google
Content-Type: application/json

Body:     { "idToken": "<credential de Google Identity Services>" }

200 OK:   { "token": "<JWT propio del sistema>" }
401:      { "error": "token_invalido", "detail": "<mensaje>" }
```

Guardar el `token` en `localStorage`. Es lo único que se necesita del login — no hay endpoint
separado para pedir el perfil del usuario.

### 3.2 Decodificar el JWT en el cliente

El JWT no se verifica en el cliente (ya lo verificó el servidor al emitirlo) — solo se **decodifica**
para leer sus datos:

```js
function decodeJwt(token) {
  const base64 = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/');
  return JSON.parse(atob(base64));
}
// devuelve: { sub, name, picture, iat, exp }
```

**Esto es necesario porque:** el backend NO devuelve el nombre/foto por separado — hay que sacarlos
del propio JWT. Y son necesarios porque **el cliente debe enviar `name` manualmente** al crear/unirse
a una sala (ver 3.4 — el Gateway no lo inyecta automáticamente, solo inyecta `playerId`).

### 3.3 Conexión Socket.io

```js
io(GATEWAY_URL, { auth: { token } })
```

Si el JWT falta o es inválido, la conexión se rechaza (`connect_error` con mensaje `sin_token` o
`token_invalido`) — en ese caso, limpiar `localStorage` y redirigir a `/`.

### 3.4 Eventos que el cliente EMITE — payload exacto

> ⚠️ **Regla crítica:** nunca incluir un campo `playerId` en ningún payload. El Gateway lo inyecta
> automáticamente leyendo el JWT (`socket.user.sub`) en cada evento — cualquier `playerId` que el
> cliente mande sería redundante en el mejor caso.

| Evento | Payload exacto | Notas |
|---|---|---|
| `room:create` | `{ modo: '1v1'\|'1v1-bot'\|'2v2', name: string }` | `name` sale del JWT decodificado (3.2) |
| `room:join` | `{ codigo: string, name: string }` | Código de 6 dígitos |
| `colocacion:set` | `{ codigo, ships: [{ id, size, x, y, horizontal: boolean }, ...] }` | Array de exactamente 5, con los `id`/`size` de la tabla 2.3 |
| `disparo:realizar` | `{ codigo, x: number, y: number }` | Solo válido en fase `TURNOS` y si es tu turno |
| `salva:disparo` | `{ codigo, x, y }` | Solo válido en fase `SALVA` |
| `poder:usar` | `{ codigo, powerType, target }` | `target` según la tabla 2.7 |
| `contramedida:activar` | `{ codigo }` | Solo tiene efecto si hay una ventana activa para tu equipo |
| `chat:mensaje` | `{ codigo, text: string }` | ⚠️ **`battlecaos-chat` todavía no existe** — este evento se puede emitir sin error, pero hoy no tiene ningún efecto ni respuesta. Construir la UI igual, pero no bloquear el resto del frontend esperando que funcione. |

### 3.5 Eventos que el cliente ESCUCHA — payload exacto

| Evento | Payload exacto | Cuándo llega |
|---|---|---|
| `game:state` | Ver 3.6 — **tiene dos formas distintas**, ver advertencia | Cada vez que cambia el estado de la partida |
| `room:created` | `{ codigo, modo }` | Al crear una sala (solo le llega a quien la creó) |
| `room:joined` | `{ codigo, jugadores: [{id,name,socketId,equipo,conectado}, ...] }` | A todos los de la sala, cada vez que alguien se une |
| `room:error` | `{ error: 'modo_invalido'\|'sala_no_existe'\|'sala_llena' }` | Al fallar `room:create`/`room:join` |
| `game:error` | `{ error: '<código>' }` | Ver 3.7 para la lista completa — ⚠️ ver 3.8, riesgo de entrega |
| `timer:tick` | `{ codigo, tipo: 'COLOCACION'\|'TURNO'\|'SALVA', remaining: number(ms) }` | Cada 100ms mientras hay un temporizador activo |
| `poder:contramedida-disponible` | `{ powerType, remaining: 5000 }` | Cuando el rival activa Bombardeo/Sonar contra tu equipo |
| `chat:message` / `chat:history` | mensajes de chat | ⚠️ No funcional hasta que exista `battlecaos-chat` |

**Nota sobre `room:created`:** no incluye `jugadores` — al crear la sala, el creador todavía no tiene
la lista completa. `LobbyPage` debe mostrar optimistamente "tú" en la lista, y esperar el primer
`room:joined` (que sí llega a todos, incluido el creador) para tener la lista real.

**No existe ningún evento `game:start`.** La señal de "la partida arrancó" es simplemente que llegue
un `game:state` con `fase !== 'LOBBY'` — `LobbyPage` debe escuchar `game:state` (no solo los eventos
de sala) y navegar a `/game` en cuanto `fase` deje de ser `LOBBY`.

### 3.6 Forma de `game:state` — ⚠️ tiene DOS formas distintas, hay que manejar ambas

**Forma normal (sanitizada), la que llega en el 99% de los casos:**

```json
{
  "codigo": "123456",
  "fase": "TURNOS",
  "turno": { "jugadorActual": "google-uid-abc", "numeroTurno": 4, "pausado": false, "pausadoPor": null },
  "jugadores": [
    { "id": "google-uid-abc", "name": "Ana", "equipo": "A", "conectado": true },
    { "id": "google-uid-xyz", "name": "Beto", "equipo": "B", "conectado": true }
  ],
  "colocados": ["google-uid-abc", "google-uid-xyz"],
  "winner": null,
  "contramedidaActiva": null,
  "tableroPublico": {
    "google-uid-abc": { "size": 10, "cells": { "3,4": "hit" } },
    "google-uid-xyz": { "size": 10, "cells": { "5,5": "miss" } }
  }
}
```

**Forma "cruda" (SIN sanitizar), que llega SOLO al reconectar:** el Gateway, al detectar que un
socket reconectado tiene una partida activa, manda el objeto de sala **completo tal cual vive en
Redis** (no pasa por el mismo sanitizador que usa el resto del juego). Esto significa que en ese caso
puntual `game:state` puede traer, en vez de `tableroPublico`, un campo `tableros` con la forma
`{ [playerId]: { size, cells: { "x,y": "ship"|"hit"|"miss" } } }` — **incluyendo posiciones reales de
barcos no tocados** — más otros campos internos (`escudos`, `salvoActual`, `tormentaUsada`,
`iniciadoEn`) que no aparecen en la forma normal.

**Cómo manejarlo en el frontend, sin necesidad de arreglar el backend:** al leer el tablero desde
`game:state`, usar `gameState.tableroPublico ?? gameState.tableros` como *fallback* — y al pintar
celdas, tratar cualquier valor que no sea `hit`/`miss` como si fuera `fog` (no revelar nada extra al
usuario aunque el payload cargado del reconectar traiga más información de la que debería). Esto
evita que la UI truene por la forma inesperada, sin depender de que el backend lo corrija primero.

**Sobre el propio tablero (`tableroPublico`) — detalle importante:** el servidor **nunca** revela
`ship` en `tableroPublico`, ni siquiera al dueño del tablero. Por eso, **el frontend debe recordar
localmente** (en el estado de React, no del servidor) qué barcos colocó justo después de emitir
`colocacion:set`, para poder seguir dibujando su propio tablero con sus barcos visibles. El servidor
solo confirma si la colocación fue válida (avanzando `colocados`) o la rechaza con `game:error`.

### 3.7 Códigos de error — tabla completa (`game:error` y `room:error`)

| Código | Evento | Cuándo ocurre |
|---|---|---|
| `modo_invalido` | `room:error` | El `modo` enviado en `room:create` no es uno de los 3 válidos |
| `sala_no_existe` | `room:error` | El `codigo` enviado en `room:join` no corresponde a ninguna sala |
| `sala_llena` | `room:error` | La sala ya tiene todos sus slots ocupados |
| `fase_incorrecta` | `game:error` | Se intentó una acción que no corresponde a la fase actual (ej. disparar en `COLOCACION`) |
| `turno_pausado` | `game:error` | Se intentó disparar mientras el turno está pausado por desconexión |
| `no_es_tu_turno` | `game:error` | Se intentó disparar o usar poder fuera de tu turno |
| `tablero_no_disponible` | `game:error` | Error interno — el tablero rival no existe aún en el estado |
| `celda_protegida` | `game:error` | El disparo cayó en una celda con Escudo activo — se anuló, el escudo se consumió |
| `celda_ya_disparada` | `game:error` | Se intentó disparar dos veces a la misma celda |
| `flota_incompleta` / `barcos_incorrectos` / `tamano_incorrecto` | `game:error` | La flota enviada en `colocacion:set` no cumple la composición de 2.3 |
| `fuera_de_limites` / `celda_ocupada` | `game:error` | Un barco de la flota enviada se sale del tablero o se solapa con otro |
| `poder_invalido` | `game:error` | `powerType` no es uno de los 5 válidos |
| `energia_insuficiente` | `game:error` | No hay energía de equipo suficiente para el poder pedido |
| `tormenta_ya_usada` | `game:error` | Ese jugador ya usó Tormenta antes en esta partida |
| `contramedida_no_disponible` | `game:error` | Se intentó activar Contramedida sin una ventana abierta para tu equipo |
| `ventana_expirada` | `game:error` | Se intentó activar Contramedida después de los 5s |
| `cadencia_no_cumplida` | `game:error` | Se disparó en Salva antes de cumplir 0.5s desde tu disparo anterior |
| `celda_ya_tomada` | `game:error` | En Salva, un compañero de equipo ya reservó esa celda una fracción de segundo antes |

### 3.8 ⚠️ Riesgo conocido, sin confirmar: entrega de `game:error`

Al revisar el código del Gateway y de `battlecaos-game`, encontré que los errores de juego
(`game:error`) se envían dirigidos al **`playerId`** (el `sub` del JWT) como si fuera un "room" de
Socket.io al que hacer `io.to(playerId).emit(...)`. Pero en el Gateway, los sockets **solo** se unen
al room `codigo` de la sala — nunca a un room nombrado con su propio `playerId`. Los rooms
implícitos de Socket.io se indexan por `socket.id`, no por `playerId`. Esto significa que
`game:error` **podría no llegarle nunca al jugador correcto** tal como está hoy el backend (a
diferencia de `room:error`, que sí usa `socketId` correctamente y por lo tanto debería llegar bien).

**Qué hacer en el frontend mientras esto no se confirme/corrija en el backend:**
- No diseñar la UI asumiendo que **siempre** vas a recibir un `game:error` explícito ante cada acción
  rechazada — considerar también un timeout corto (ej. "si no cambió el estado en 2s, algo falló")
  como señal complementaria, no como reemplazo.
- Probar esto específicamente apenas se conecte el frontend real al backend real (ej. intentar
  disparar dos veces la misma celda y confirmar si el toast de error aparece) — si no llega, es este
  bug, no un error de la UI.

### 3.9 Otros endpoints HTTP (no de juego)

```
GET {VITE_GATEWAY_URL}/kpis
200: { "tasa_completacion": "66.7%"|"N/A", "tasa_reconexion": "80.0%"|"N/A", "pico_salas": 3, "date": "2026-07-05" }
```
Usado solo por `AdminPage` (sección 4.5). No requiere JWT.

---

## 4. Pantallas (screens) — detalladas una por una

### 4.1 `LoginPage` — ruta `/` — UIUXF101

**Qué muestra:** título del juego, botón "Iniciar sesión con Google" (Google Identity Services), y
tres estados: `idle` (botón visible), `loading` (spinner/texto), `error` (`role="alert"`).

**Flujo exacto:**
1. Inicializar `window.google.accounts.id` con `VITE_GOOGLE_CLIENT_ID`.
2. El callback de Google entrega `{ credential: idToken }`.
3. `POST {VITE_AUTH_URL}/auth/google` con `{ idToken }` (ver 3.1).
4. Guardar `token` en `localStorage`.
5. Redirigir a `/lobby`.

No toca Socket.io todavía.

---

### 4.2 `LobbyPage` — ruta `/lobby` — UIUXF201

**Qué muestra:**
- Selector de modo (2.2).
- Botón "Crear sala" → emite `room:create` (3.4).
- Input de código + botón "Unirse" → emite `room:join` (3.4).
- `PlayerList` con el equipo ya asignado por el servidor (2.2) — nunca ofrecer elegir equipo.
- Código de sala visible tras crearla.

**Eventos que escucha:** `room:created`, `room:joined`, `room:error` (3.5), y **`game:state`** — en
cuanto llegue con `fase !== 'LOBBY'`, navegar a `/game` (no existe un evento `game:start`, ver 3.5).

**Guardia:** si no hay JWT en `localStorage`, redirigir a `/` antes de conectar el socket.

---

### 4.3 `GamePage` — ruta `/game` — la pantalla más compleja (UIUXF002, 301, 401, 501, 601, 701, 801, 901, 1001)

Contenedor que orquesta componentes según `gameState.fase` (2.4):

| Zona | Componente | Se muestra cuando |
|---|---|---|
| Cabecera | Indicador de fase + `Timer` | Siempre |
| Tablero propio | `Board` (lectura) | Siempre — ver nota de 3.6 sobre recordar la flota localmente |
| Tablero rival | `Board` (interactivo) | Siempre |
| Colocación | `ShipPlacer` | Solo en `COLOCACION` |
| Energía | `EnergyBar` | `TURNOS`/`SALVA` en adelante |
| Poderes | `PowerPanel` | `TURNOS`, si hay energía para al menos uno |
| Alerta de contramedida | `CountermeasureAlert` | Al recibir `poder:contramedida-disponible` |
| Banner de Salva | `SalvoBanner` | Fase `SALVA` |
| Indicador de desconexión | Badge en `PlayerList`/cabecera | Cuando algún `jugador.conectado === false` |
| Chat | `Chat` | Siempre (no funcional hasta `battlecaos-chat`, ver 3.4) |

**Fuente de datos:**
```jsx
const gameState = useGameState(socket); // último game:state — ver 3.6 para las dos formas posibles
```

**Lógica de fase:**
```
COLOCACION → ShipPlacer visible; Board rival deshabilitado
TURNOS     → Board rival interactivo SOLO si gameState.turno.jugadorActual === miId
SALVA      → Board rival interactivo para TODOS, cadencia 0.5s (el servidor la valida, no el cliente)
FIN        → navegar a /result con { winner, modo, duracion } en el state de React Router
```

**`Board` (UIUXF002/301):** N×N (2.3), estados `fog`/`ship`/`hit`/`miss`/`sunk`/`selected`. El
tablero rival solo tiene `hit`/`miss`/`sunk` reales — nunca posiciones de barcos no tocados. El
tablero propio requiere la flota recordada localmente (ver 3.6).

**`ShipPlacer` (UIUXF501):** un barco a la vez de la tabla 2.3, botón girar, clic para posicionar,
"Confirmar flota" → `colocacion:set` con los 5. Guardar esa misma lista de ships en el estado local
de `GamePage` para poder pintar el tablero propio después (ver 3.6).

**`EnergyBar` (UIUXF701):** energía de **equipo** (2.6), no de jugador — dejarlo claro visualmente en 2v2.

**`PowerPanel` (UIUXF801):** 4 botones con costo exacto (2.7). Objetivo según poder (bombardeo/sonar
apuntan al tablero rival, escudo al propio, tormenta sin objetivo). Deshabilitar por energía
insuficiente y, para Tormenta, deshabilitar permanentemente tras el primer uso exitoso de ese
jugador.

**`CountermeasureAlert` (UIUXF901):** cuenta regresiva de 5s exactos (2.8), botón que emite
`contramedida:activar`. Desaparece sola si pasan los 5s.

**`SalvoBanner` (UIUXF1001):** cuenta regresiva de 8s, permite varios disparos en la ventana (2.9),
refleja (no previene) los rechazos por cadencia o celda tomada.

**`Timer`:** un componente para `COLOCACION`/`TURNO`/`SALVA` vía `timer:tick` (3.5) — la ventana de
Contramedida NO usa `timer:tick`, usa su propio evento.

**Indicador de desconexión:** sin componente propio — un badge condicionado a
`jugador.conectado === false` en `PlayerList`/cabecera, y aclarar visualmente "turno en pausa" si
coincide con `gameState.turno.pausadoPor`.

**`Chat` (UIUXF401):** historial + mensajes en vivo, construir la UI aunque hoy no tenga efecto real
(ver 3.4).

---

### 4.4 `ResultPage` — ruta `/result` — UIUXF1201

"¡Equipo A/B gana!", modo, duración (formateada desde ms), botón "Nueva partida" → `/lobby`. Los
datos llegan por `navigate('/result', { state: { winner, modo, duracion } })` desde `GamePage` — no
hace falta ningún fetch ni evento nuevo.

`ErrorBoundary` en `App.jsx`, envolviendo todas las rutas, para cualquier error de render inesperado
(ej. si llega la forma "cruda" de `game:state` de 3.6 y algo no se manejó bien).

---

### 4.5 `AdminPage` — ruta `/admin` — UIUXF1101

Panel de solo lectura con los KPIs de 2.13, consumidos de `GET {VITE_GATEWAY_URL}/kpis` (3.9),
refrescado cada 5s. Sin guardia de autenticación especial.

---

## 5. Tabla de rutas completa

| Ruta | Componente | Requiere JWT | Tarea |
|---|---|:---:|---|
| `/` | `LoginPage` | No | UIUXF101 |
| `/lobby` | `LobbyPage` | Sí | UIUXF201 |
| `/game` | `GamePage` | Sí | UIUXF002, 301, 401, 501, 601, 701, 801, 901, 1001 |
| `/result` | `ResultPage` | Sí | UIUXF1201 |
| `/admin` | `AdminPage` | No | UIUXF1101 |
| `*` | Redirect a `/` | — | — |

> **Nota:** `CLAUDE.frontend.md` (otro documento del proyecto) le falta la ruta `/admin` y tiene los
> IDs UIUXF801/901/1001/1101/1201 mal cruzados con tareas que no son. Esta tabla usa los IDs reales
> de `Orden_de_Ejecucion.md`.

---

## 6. Estructura de carpetas propuesta

```
battlecaos-frontend/
├── .env.example
├── .env.local                     ← real, NO se commitea
├── .gitignore
├── package.json
├── vite.config.js
├── index.html
└── src/
    ├── main.jsx
    ├── App.jsx                    ← React Router: 5 rutas + ErrorBoundary
    ├── styles/
    │   └── tokens.css
    ├── hooks/
    │   ├── useSocket.js           ← conexión Socket.io autenticada (3.3)
    │   ├── useGameState.js        ← suscripción a game:state (3.6)
    │   └── useAuth.js             ← JWT en localStorage + decodeJwt (3.2)
    ├── pages/
    │   ├── LoginPage.jsx
    │   ├── LobbyPage.jsx
    │   ├── GamePage.jsx
    │   ├── ResultPage.jsx
    │   └── AdminPage.jsx
    └── components/
        ├── Board/
        ├── ShipPlacer/
        ├── EnergyBar/
        ├── PowerPanel/
        ├── CountermeasureAlert/
        ├── SalvoBanner/
        ├── Chat/
        ├── Timer/
        ├── PlayerList/
        └── ErrorBoundary/
```

---

## 7. Componentes reutilizables — props

| Componente | Props principales | Emite (Socket.io) |
|---|---|---|
| `Board` | `size`, `cells`, `onCellClick`, `interactive` | (delegado al padre) |
| `ShipPlacer` | `socket`, `codigo`, `onConfirm(ships)` | `colocacion:set` |
| `EnergyBar` | `energia`, `max` | — |
| `PowerPanel` | `socket`, `codigo`, `energia`, `tormentaUsada` | `poder:usar` |
| `CountermeasureAlert` | `socket`, `codigo` | `contramedida:activar` |
| `SalvoBanner` | `remaining` | — |
| `Timer` | `tipo`, `remaining` | — |
| `Chat` | `socket`, `codigo` | `chat:mensaje` |
| `PlayerList` | `jugadores` (con `equipo`, `conectado`) | — |
| `ErrorBoundary` | `children` | — |

---

## 8. Hooks — implementación de referencia

```js
// useSocket.js
export function useSocket(token) {
  const socketRef = useRef(null);
  useEffect(() => {
    if (!token) return;
    socketRef.current = io(import.meta.env.VITE_GATEWAY_URL, { auth: { token } });
    return () => socketRef.current?.disconnect();
  }, [token]);
  return socketRef.current;
}
```

```js
// useGameState.js
export function useGameState(socket) {
  const [gameState, setGameState] = useState(null);
  useEffect(() => {
    if (!socket) return;
    socket.on('game:state', setGameState);
    return () => socket.off('game:state', setGameState);
  }, [socket]);
  return gameState;
}
```

```js
// useAuth.js
export function useAuth() {
  const token = localStorage.getItem('token');
  const profile = token ? decodeJwt(token) : null; // { sub, name, picture } — ver 3.2
  return { token, profile, isAuthenticated: !!token, logout: () => localStorage.removeItem('token') };
}
```

Sin Context API global ni store — cada página pide lo que necesita con estos hooks.

---

## 9. Sistema de diseño — tokens CSS

```css
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

---

## 10. Variables de entorno

```env
VITE_GATEWAY_URL=http://localhost:3000
VITE_AUTH_URL=http://localhost:3001
VITE_GOOGLE_CLIENT_ID=tu-client-id.apps.googleusercontent.com
```

Ninguna es secreta — todo `VITE_*` queda embebido en el bundle público.

---

## 11. Testing

| Prioridad | Qué testear | Por qué |
|---|---|---|
| Alta | `Board.jsx` | Componente visual más crítico |
| Alta | `useGameState.js` | Con mock de `socket.io-client`, confirmar ambas formas de `game:state` (3.6) |
| Media | `EnergyBar.jsx`, `PowerPanel.jsx` | Lógica de props simple pero verificable |
| Baja | Páginas completas | Dependen de Google Identity Services externo — mejor verificación manual |

---

## 12. Build y despliegue

- Dev: `npm run dev` → `http://localhost:5173`.
- Build: `npm run build` → `dist/`.
- Despliegue: Vercel (gratis, detecta Vite, deploy por push).
- Actualizar `CLIENT_ORIGIN` en `battlecaos-gateway` a la URL de Vercel cuando se despliegue.

---

## 13. Orden de implementación sugerido

1. **UIUXF001** — esqueleto: `App.jsx`, 5 páginas vacías, `useSocket.js`.
2. **UIUXF101** — Login (o un JWT de prueba vía `gen-token.mjs` del repo raíz, para no bloquearse en Google OAuth desde el día 1).
3. **UIUXF201** — Lobby.
4. **UIUXF002 + UIUXF301** — `Board` + `game:state`.
5. **UIUXF501** — Colocación de flota (recordando la flota localmente, ver 3.6).
6. **UIUXF601 + UIUXF701** — Turnos + energía.
7. **UIUXF801 + UIUXF901** — Poderes + contramedida.
8. **UIUXF1001** — Salva.
9. **UIUXF1201** — Resultado + manejo de errores.
10. **UIUXF401** — Chat (no bloquea nada, no funcional hasta `battlecaos-chat`).
11. **UIUXF1101** — Admin/KPIs.

Con 1-9 hay una partida 1v1 jugable de punta a punta cubriendo todas las mecánicas de la sección 2.

---

## 14. Resumen de riesgos/inconsistencias encontradas (para no perder tiempo debugueando "bugs" que en realidad son del backend)

1. **`game:error` puede no llegar** (sección 3.8) — se dirige por `playerId`, pero el Gateway solo
   soporta entrega por `socketId` o por room `codigo`. Verificar en vivo antes de asumir que la UI
   de errores está mal escrita.
2. **`game:state` tiene dos formas distintas** (sección 3.6) — sanitizada (`tableroPublico`) en el
   flujo normal, cruda (`tableros`, con posiciones reales) al reconectar. Programar defensivamente.
3. **El propio tablero nunca llega con `ship` en el estado sanitizado** — hay que recordar la flota
   colocada en el estado local del cliente, no esperar que el servidor la vuelva a mandar.
4. **No existe evento `game:start`** — la señal de que la partida arrancó es que `game:state.fase`
   deje de ser `LOBBY`.
5. **`chat:mensaje` no tiene efecto hoy** — `battlecaos-chat` no está construido. Construir la UI de
   todos modos, pero no depender de que funcione para probar el resto del frontend.
