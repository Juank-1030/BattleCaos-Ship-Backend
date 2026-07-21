# Diagramas de clases — uno por microservicio

Complemento de [`DIAGRAMAS.md`](DIAGRAMAS.md), que contiene la vista **unificada** del
dominio (útil para explicar las reglas del juego de un vistazo). Aquí está el detalle
**repositorio por repositorio**, incluidos los módulos de infraestructura que aquella vista
omite a propósito.

## Nota sobre la notación

El código es **JavaScript con módulos ES**, no orientado a objetos: casi todo son *funciones
exportadas* agrupadas por archivo. Se representan como clases con el estereotipo
`<<module: ruta>>` porque el archivo **es** la unidad de encapsulamiento del proyecto.

Solo hay **una clase real** (`class`) en todo el backend: `DomainError`, en
`game/src/domain/errors.js`.

| Repositorio | Módulos propios | Diagrama |
|---|---|---|
| Comunes (7 servicios) | correlation · logger · redis · kafka · dlq · lock · backoff · observability | [Ver](#0-módulos-comunes) |
| `battlecaos-auth` | auth/google · auth/jwt · auth/local · mongo · metrics | [Ver](#1-auth) |
| `battlecaos-gateway` | domain/router · metrics · httpMetricsMiddleware | [Ver](#2-gateway) |
| `battlecaos-room` | domain/room | [Ver](#3-room) |
| `battlecaos-game` | 8 módulos de dominio + 9 handlers | [Ver](#4-game) |
| `battlecaos-timer` | TimerManager | [Ver](#5-timer) |
| `battlecaos-chat` | domain/chat | [Ver](#6-chat) |
| `battlecaos-voice-channel` | domain/voice | [Ver](#7-voice-channel) |
| `battlecaos-bot` | strategy | [Ver](#8-bot) |
| `battlecaos-observability` | domain/kpi · persistencia | [Ver](#9-observability) |

---

## 0. Módulos comunes

Estos archivos son **idénticos** en los servicios que los usan (verificado por hash). No se
comparten como paquete npm: se replican, que es una decisión consciente para que cada
microservicio se despliegue sin depender de una librería interna versionada.

```mermaid
classDiagram
    direction LR

    class Correlation {
        <<module: src/correlation.js>>
        +conCorrelation(correlationId, fn)
        +correlationActual() string
    }

    class Logger {
        <<module: src/logger.js>>
        +log
    }

    class RedisClient {
        <<module: src/redis.js>>
        +READ_CMDS
        +WRITE_CMDS
        +esErrorDeConexion(err) boolean
        +makeResilient(primary, secondary, opts) Proxy
        +createRedis() Cliente
    }

    class KafkaClient {
        <<module: src/kafka.js>>
        +producer
        +createConsumer(groupId) Consumer
    }

    class DLQ {
        <<module: src/dlq.js>>
        +enviarADLQ(datos)
    }

    class Lock {
        <<module: src/lock.js>>
        +conLockSala(redisLike, codigo, fn, opts)
    }

    class Backoff {
        <<module: src/backoff.js>>
        +crearBackoff(opts) Backoff
    }

    class Observability {
        <<module: src/observability.js>>
        +register
        +eventsProcessed
        +eventLatency
        +redisNodeUp
        +mongoUp
        +kafkaConsumerUp
        +redisSlowlogLen
        +trackConsumer(consumer)
        +instrumentar(tipo, fn) Function
        +configurarSlowQueries(redis, opts)
        +startObservability(deps) Server
    }

    Logger ..> Correlation : inyecta el cid en cada línea
    Observability ..> Correlation : expone el cid actual
    DLQ ..> KafkaClient : publica en el topic dlq
    DLQ ..> Logger : registra el fallo
    Lock ..> RedisClient : SET NX PX

    note for RedisClient "createRedis() devuelve un cliente ÚNICO, o uno RESILIENTE si existe REDIS_FALLBACK_URL: escribe en ambos nodos y lee del sano."
    note for Logger "Sanea saltos de línea y caracteres de control antes de escribir (inyección de logs, Sonar S5145)."
    note for Observability "Sirve /health y /metrics. El chequeo de Redis y Mongo lleva límite de 1,5 s para no colgarse durante una caída."
```

| Módulo | auth | gateway | room | game | timer | chat | voice | bot | observ. |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `correlation.js` | | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `logger.js` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `redis.js` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | | ✅ |
| `kafka.js` | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `dlq.js` | | | ✅ | ✅ | | ✅ | ✅ | | |
| `lock.js` | | ✅ | ✅ | ✅ | | | | | |
| `backoff.js` | | ✅ | | ✅ | | | | | |
| `observability.js` | | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

> `auth` y `gateway` no usan `observability.js`: exponen `/metrics` con Express a través de
> su propio `metrics.js` + `httpMetricsMiddleware.js`.

---

## 1. auth

```mermaid
classDiagram
    direction LR

    class Usuario {
        +string id
        +string email
        +string apodo
        +string passwordHash
        +string googleSub
        +string picture
        +number creadoEn
        +number ultimoLogin
    }

    class AuthLocal {
        <<module: src/auth/local.js>>
        +hashPassword(password) string
        +verifyPassword(password, hash) boolean
        +validarRegistro(datos) string
        +crearUsuario(deps, datos) Usuario
        +autenticarUsuario(deps, credenciales) Usuario
    }

    class AuthGoogle {
        <<module: src/auth/google.js>>
        +verifyGoogleToken(idToken) Perfil
    }

    class Jwt {
        <<module: src/auth/jwt.js>>
        +signToken(payload) string
        +verifyToken(token) Payload
    }

    class Mongo {
        <<module: src/mongo.js>>
        +createMongo() Conexion
    }

    class Metrics {
        <<module: src/metrics.js>>
        +register
        +httpDuration
        +loginSuccessTotal
        +loginFailedTotal
    }

    class HttpMetricsMiddleware {
        <<module: src/httpMetricsMiddleware.js>>
        +httpMetricsMiddleware(httpDuration) Middleware
    }

    AuthLocal ..> Usuario : crea y valida
    AuthLocal ..> Mongo : persiste en usuarios
    AuthGoogle ..> Usuario : deriva el perfil
    AuthLocal ..> Jwt : el índice se firma aparte
    HttpMetricsMiddleware ..> Metrics : alimenta httpDuration

    note for AuthLocal "hashPassword usa scrypt · verifyPassword compara con timingSafeEqual para no filtrar información por el tiempo de respuesta."
    note for Jwt "HS256, 24 h. El gateway lo verifica LOCALMENTE con el mismo JWT_SECRET: no hay llamada a auth en cada conexión."
    note for Mongo "Índices: email único sparse · googleSub sparse. El único garantiza la unicidad en la BD, no en el código."
```

---

## 2. gateway

```mermaid
classDiagram
    direction LR

    class Router {
        <<module: src/domain/router.js>>
        +ROUTES
        +buildMessage(event, socketId, playerId, payload) Mensaje
    }

    class Mensaje {
        +string type
        +string source
        +number timestamp
        +int version
        +string correlationId
        +object data
    }

    class Metrics {
        <<module: src/metrics.js>>
        +register
        +httpDuration
        +socketConnectionsTotal
        +eventsRoutedTotal
        +rateLimitExceededTotal
        +reconnectionsTotal
        +kafkaConsumerCrashTotal
        +activeSocketsGauge
        +kafkaConsumerUp
    }

    class HttpMetricsMiddleware {
        <<module: src/httpMetricsMiddleware.js>>
        +httpMetricsMiddleware(httpDuration) Middleware
    }

    Router ..> Mensaje : construye
    HttpMetricsMiddleware ..> Metrics : alimenta httpDuration

    note for Router "ROUTES mapea 15 eventos de cliente a 4 topics: cmd.room, cmd.game, cmd.chat y cmd.voice."
    note for Mensaje "En buildMessage el payload va PRIMERO y socketId/playerId DESPUÉS, así la identidad verificada del JWT siempre gana sobre lo que envíe el cliente."
    note for Metrics "activeSocketsGauge es un GAUGE (sube y baja). Si fuera contador, el panel de jugadores conectados nunca reflejaría a quien se va."
```

---

## 3. room

```mermaid
classDiagram
    direction LR

    class Sala {
        +string codigo
        +string modo
        +string nombre
        +string fase
        +string hostId
        +Jugador[] jugadores
        +int slotsMax
        +number creadoEn
    }

    class Jugador {
        +string id
        +string name
        +string socketId
        +string equipo
        +boolean conectado
        +number desconectadoEn
        +boolean esBot
    }

    class RoomDomain {
        <<module: src/domain/room.js>>
        +validarModo(modo)
        +generarCodigo() string
        +normalizarNombreSala(nombre, fallback) string
        +crearSala(codigo, modo, playerId, name, socketId, nombreSala) Sala
        +contarEquipo(sala, equipo) int
        +asignarEquipo(sala) string
        +estaLlena(sala) boolean
        +agregarJugador(sala, playerId, name, socketId) string
        +cambiarEquipo(sala, playerId, equipo, swapConId)
        +puedeComenzar(sala) boolean
        +quitarJugador(sala, playerId) boolean
        +reiniciarParaRevancha(sala, playerId, name, socketId) Sala
        +marcarDesconectado(sala, socketId, playerId) Jugador
        +todosDesconectados(sala) boolean
        +calcularEquipos(sala) object
    }

    Sala "1" *-- "2..4" Jugador
    RoomDomain ..> Sala : opera sobre

    note for RoomDomain "Módulo PURO: no toca Redis ni Kafka. Recibe la sala, la muta y la devuelve. Por eso se puede probar entero sin levantar nada."
    note for RoomDomain "reiniciarParaRevancha CONSERVA a los jugadores conectados. Antes dejaba solo al que volvía y eso expulsaba al rival de la sala."
    note for Sala "Fases: LOBBY → COLOCACION → TURNOS ⇄ SALVA → FIN"
```

---

## 4. game

El repositorio más grande: **8 módulos de dominio** (reglas puras) y **9 handlers**
(orquestación con Redis y Kafka). La separación es deliberada: las reglas se prueban sin
infraestructura.

### 4.1 Dominio (reglas puras)

```mermaid
classDiagram
    direction LR

    class DomainError {
        <<class real: src/domain/errors.js>>
        +string code
        +string message
    }

    class Tablero {
        +int size
        +Map cells
        +Map ships
    }

    class Barco {
        +string id
        +int size
        +int x
        +int y
        +boolean horizontal
    }

    class Board {
        <<module: src/domain/board.js>>
        +BOARD_SIZE_BY_MODE
        +sizeForMode(modo) int
        +createBoard(size) Tablero
        +placeShip(board, ship) Tablero
        +shoot(board, x, y) Resultado
        +isShipSunk(board, shipId) boolean
        +markShipSunk(board, shipId)
    }

    class Fleet {
        <<module: src/domain/fleet.js>>
        +FLEET_CONFIG
        +validateFleet(ships)
        +isFleetSunk(board) boolean
    }

    class AutoFleet {
        <<module: src/domain/autoFleet.js>>
        +generarFlotaAleatoria(size) object
    }

    class Engine {
        <<module: src/domain/engine.js>>
        +FASES
        +SALVA_EVERY_N_ROUNDS
        +nextFase(faseActual) string
        +validateTurn(partida, playerId)
        +rotateTurn(partida) string
    }

    class Energy {
        <<module: src/domain/energy.js>>
        +ENERGY_CAP
        +energyGain(resultado) int
        +hasEnough(energia, coste) boolean
    }

    class Powers {
        <<module: src/domain/powers.js>>
        +POWER_COSTS
        +OFFENSIVE_POWERS
        +validatePower(id, energia)
        +applyBombardeo(board, x, y)
        +applySonar(board, x, y)
        +applyEscudo(partida, equipo)
        +teamShieldActive(partida, equipo) boolean
        +consumeTeamShield(partida, equipo)
        +applyTormenta(partida)
    }

    class Salvo {
        <<module: src/domain/salvo.js>>
        +SALVO_CADENCIA_MS
        +SALVO_CADENCIA_TOLERANCIA_MS
        +canFireInSalvo(estado, playerId, ahora) boolean
        +registerShot(estado, playerId, ahora)
    }

    class Countermeasure {
        <<module: src/domain/countermeasure.js>>
        +COUNTERABLE_POWERS
        +canCountermeasure(partida, equipo) boolean
        +isWindowExpired(evento, ahora) boolean
    }

    Tablero "1" *-- "5" Barco
    Board ..> Tablero : crea y muta
    Board ..> DomainError : lanza
    Fleet ..> Barco : valida
    Fleet ..> DomainError : lanza
    AutoFleet ..> Board : coloca
    AutoFleet ..> Fleet : respeta FLEET_CONFIG
    Engine ..> DomainError : lanza
    Powers ..> Tablero : aplica efectos
    Powers ..> Energy : comprueba coste
    Salvo ..> Engine : solo en fase SALVA
    Countermeasure ..> Powers : contrarresta

    note for DomainError "ÚNICA clase real del backend. Lleva un `code` estable (fase_incorrecta, celda_ocupada…) que el frontend traduce a texto en constants/copy.js."
    note for Tablero "cells usa claves de tipo x,y con valor ship / hit / miss. Se eligió un mapa en vez de una matriz porque el estado viaja como JSON a Redis en cada jugada."
    note for Fleet "validateFleet rechaza cualquier flota que no coincida EXACTAMENTE en ids y tamaños: es la defensa contra un cliente manipulado."
```

### 4.2 Handlers (orquestación)

```mermaid
classDiagram
    direction TB

    class Broker {
        <<module: src/handlers/broker.js>>
        +startBroker()
    }

    class HandlePlaceShips {
        <<module: src/handlers/handlePlaceShips.js>>
        +handlePlaceShips(data)
    }

    class HandleShot {
        <<module: src/handlers/handleShot.js>>
        +handleShot(data)
    }

    class HandleSalvo {
        <<module: src/handlers/handleSalvo.js>>
        +handleSalvo(data)
    }

    class HandlePower {
        <<module: src/handlers/handlePower.js>>
        +handlePower(data)
    }

    class HandleCountermeasure {
        <<module: src/handlers/handleCountermeasure.js>>
        +handleCountermeasure(data)
    }

    class HandleDisconnect {
        <<module: src/handlers/handleDisconnect.js>>
        +handleDisconnect(data)
        +handleReconnect(data)
    }

    class TurnFlow {
        <<module: src/handlers/turnFlow.js>>
        +advanceTurn(sala)
        +endSalvo(sala)
    }

    class Helpers {
        <<module: src/handlers/helpers.js>>
        +SALA_TTL_SEG
        +SALA_FIN_TTL_SEG
        +broadcastState(codigo, salaPreCargada)
        +publish(type, data)
        +broadcastEvent(evento)
        +sendOwnFleets(sala)
        +sendTeamFleet(sala, playerId)
        +sendError(socketId, error)
    }

    Broker --> HandlePlaceShips : cmd.game
    Broker --> HandleShot : cmd.game
    Broker --> HandleSalvo : cmd.game
    Broker --> HandlePower : cmd.game
    Broker --> HandleCountermeasure : cmd.game
    Broker --> HandleDisconnect : evt.room

    HandleShot ..> TurnFlow : al resolver el disparo
    HandleSalvo ..> TurnFlow : al cerrar la salva
    HandlePlaceShips ..> Helpers
    HandleShot ..> Helpers
    HandleSalvo ..> Helpers
    HandlePower ..> Helpers
    HandleCountermeasure ..> Helpers
    HandleDisconnect ..> Helpers
    TurnFlow ..> Helpers

    note for Broker "Punto de entrada del consumer. Suscribe a cmd.game, evt.room, evt.timer y evt.bot, y despacha por msg.type. Si un handler lanza, el mensaje va a la DLQ."
    note for Helpers "Toda la salida al cliente pasa por aquí: broadcastState renueva el TTL de la sala y lee la energía en UN solo pipeline, para no pagar dos round-trips por acción."
    note for HandleShot "Se ejecuta dentro de conLockSala(): serializa el read-modify-write de la sala frente al gateway, que también la escribe al reconectar."
```

---

## 5. timer

```mermaid
classDiagram
    direction LR

    class TimerManager {
        <<module: src/TimerManager.js>>
        +DURATIONS
        +startTimer(codigo, tipo, onTick, onEnd) boolean
        +stopTimer(codigo)
        +pauseTimer(codigo)
        +resumeTimer(codigo)
        +hasActiveTimer(codigo) boolean
        +getRemaining(codigo) number
    }

    class EstadoTimer {
        +string tipo
        +number remaining
        +boolean paused
        +intervalId
        +timeoutId
    }

    TimerManager "1" *-- "0..*" EstadoTimer : uno por sala

    note for TimerManager "Estado EN MEMORIA, indexado por código de sala. Solo la réplica LÍDER ejecuta timers reales: el liderazgo se disputa con un lease en Redis (SET NX EX 1) renovado cada 500 ms."
    note for TimerManager "DURATIONS define el tiempo de cada fase: COLOCACION, TURNO y SALVA."
    note for EstadoTimer "pauseTimer conserva `remaining`: si un jugador se desconecta en su turno, al reconectar el turno se reanuda donde estaba, no desde cero."
```

---

## 6. chat

```mermaid
classDiagram
    direction LR

    class MensajeChat {
        +string de
        +string texto
        +string canal
        +string equipo
        +number ts
    }

    class ChatDomain {
        <<module: src/domain/chat.js>>
        +MAX_LEN
        +normalizarTexto(texto) string
        +esValido(texto) boolean
        +normalizarCanal(canal) string
        +construirMensaje(datos) MensajeChat
        +puedeVerMensaje(msg, jugador) boolean
        +datosJugador(sala, playerId) object
    }

    ChatDomain ..> MensajeChat : construye y filtra

    note for ChatDomain "puedeVerMensaje se evalúa en el SERVIDOR. Si el canal `equipo` se filtrara en el cliente, el mensaje del rival ya habría viajado al navegador y bastaría abrir las herramientas de desarrollo para leerlo."
    note for MensajeChat "Los mensajes se guardan en la lista Redis sala:{codigo}:chat, con el mismo TTL que la sala."
```

---

## 7. voice-channel

```mermaid
classDiagram
    direction LR

    class VoiceDomain {
        <<module: src/domain/voice.js>>
        +vozHabilitada(modo) boolean
        +normalizarCanal(canal) string
        +mismoCanal(a, b) boolean
        +peersParaJugador(sala, playerId) Jugador[]
    }

    class Peer {
        +string id
        +string name
        +string socketId
        +string canal
        +boolean muted
    }

    VoiceDomain ..> Peer : calcula la lista

    note for VoiceDomain "Solo SEÑALIZACIÓN: decide con quién debe hablar cada jugador. El audio va directo entre navegadores por WebRTC y nunca pasa por el servidor."
    note for VoiceDomain "peersParaJugador respeta el canal: en 2v2 el canal `equipo` aísla a cada bando."
```

---

## 8. bot

```mermaid
classDiagram
    direction LR

    class BotStrategy {
        <<module: src/strategy.js>>
        +TABLERO
        +FLOTA_INICIAL
        +crearEstado(size) Estado
        +registrarDisparo(estado, x, y, resultado)
        +mapaProbabilidad(estado) number[][]
        +decidirDisparo(estado) Celda
        +estadoDesdeTablero(board) Estado
    }

    class Estado {
        +int size
        +Map disparos
        +int[] flotaRestante
        +Celda[] pendientes
    }

    BotStrategy ..> Estado : mantiene y consulta

    note for BotStrategy "No dispara al azar: mapaProbabilidad cuenta, para cada celda, en cuántas posiciones válidas cabría todavía un barco no hundido, y elige el máximo. Tras un impacto prioriza las celdas adyacentes."
    note for Estado "estadoDesdeTablero lo reconstruye a partir del tablero de Redis: el bot es SIN ESTADO entre mensajes y puede escalar o reiniciarse sin perder la partida."
```

---

## 9. observability

```mermaid
classDiagram
    direction LR

    class KpiDomain {
        <<module: src/domain/kpi.js>>
        +percentil(valores, p) number
        +tasaPct(parte, total) number
        +claveDia(fecha) string
    }

    class Persistencia {
        <<module: src/persistencia.js>>
        +persistirPartida(partidas, msg) Resultado
    }

    class Mongo {
        <<module: src/mongo.js>>
        +createMongo() Conexion
    }

    class Partida {
        +string codigo
        +string winner
        +string modo
        +number duracionMs
        +number terminadaEn
    }

    Persistencia ..> Partida : inserta
    Persistencia ..> Mongo : colección partidas
    KpiDomain ..> Partida : agrega métricas

    note for Persistencia "IDEMPOTENTE por diseño: updateOne con filtro por codigo + $setOnInsert + upsert. Kafka entrega at-least-once, así que una reentrega del mismo GameEnded NO duplica la partida. El índice único sobre codigo lo garantiza en la BD."
    note for KpiDomain "claveDia genera stats:kpi:{YYYY-MM-DD}, la clave del hash de Redis donde se acumulan los KPIs del día."
```

---

## Cobertura de la auditoría

Todos los módulos con lógica propia de cada repositorio están representados. Lo que **no**
aparece, y por qué:

| Excluido | Motivo |
|---|---|
| `index.js` (los 9) | Arranque y cableado: conecta Redis/Kafka, suscribe topics y registra handlers. No tiene modelo que dibujar. |
| `tracing.js` (los 9) | Bootstrap opt-in de OpenTelemetry. Sin `OTEL_EXPORTER_OTLP_ENDPOINT` no ejecuta nada. |

La correspondencia entre estos diagramas y el código se comprueba de forma automática:

```powershell
node tools/verificar-diagrama-clases.mjs
```

El script contrasta cada operación citada contra los `export` reales del módulo indicado y
falla si alguno no existe — así el diagrama no se desincroniza en silencio cuando se
renombra una función.
