# Diagramas de secuencia — una por interacción

Complemento de [`DIAGRAMAS.md`](DIAGRAMAS.md), que contiene el diagrama de secuencia
principal (el disparo). Aquí está **cada interacción del jugador por separado**, derivada del
código real: nombres de eventos, topics, puertos y claves de Redis tal como están hoy.

| # | Interacción | Servicios que intervienen |
|---|---|---|
| 1 | [Inicio de sesión](#1-inicio-de-sesión) | frontend · auth · Google · Mongo |
| 2 | [Crear sala](#2-crear-sala) | gateway · room |
| 3 | [Unirse a una sala](#3-unirse-a-una-sala) | gateway · room · chat · voice-channel |
| 4 | [Colocar los barcos](#4-colocar-los-barcos) | gateway · game · timer |
| 5 | [Disparar](#5-disparar) | gateway · game · timer · observability |
| 6 | [Usar un poder](#6-usar-un-poder) | gateway · game |
| 7 | [Escribir en el chat](#7-escribir-en-el-chat) | gateway · chat |
| 8 | [Entrar al chat de voz](#8-entrar-al-chat-de-voz) | gateway · voice-channel (WebRTC P2P) |
| 9 | [Fin de partida y revancha](#9-fin-de-partida-y-revancha) | game · room |
| 10 | [Cerrar sesión](#10-cerrar-sesión) | frontend · gateway |
| 11 | [Cómo se observa una jugada](#11-cómo-se-observa-una-jugada-métricas-logs-y-alertas) | Prometheus · promtail · Loki · Alertmanager · Grafana |

**Convención común a todas:** el gateway publica en `cmd.*` con `key = código de sala`
(garantiza orden por partición) y un `correlationId` que acompaña al evento por todos los
servicios. La respuesta al cliente vuelve por `gw.broadcast`, que las **3 réplicas** consumen
en fan-out.

---

## 1. Inicio de sesión

Dos caminos: cuenta local (email + contraseña) o Google. Ambos acaban en el **mismo JWT
propio** (HS256, 24 h) que el gateway verifica en el handshake del socket.

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend :5173
    participant AU as auth :3001
    participant GO as Google Identity
    participant MG as MongoDB Atlas
    participant GW as gateway :8090

    alt Cuenta local
        J->>FE: email + contraseña
        FE->>AU: POST /auth/login
        AU->>MG: usuarios.findOne({email})
        MG-->>AU: usuario
        Note over AU: scrypt: compara passwordHash<br/>timingSafeEqual (no comparación directa)
        AU->>AU: signToken({sub, name, picture})
        AU-->>FE: 200 {token}
    else Google
        J->>FE: botón "Entrar con Google"
        FE->>GO: flujo OAuth
        GO-->>FE: id_token
        FE->>AU: POST /auth/google {idToken}
        AU->>GO: verifyIdToken(idToken)
        GO-->>AU: {sub, name, picture}
        AU->>MG: updateOne({googleSub}, upsert)
        Note over AU: si el usuario ya cambió su apodo,<br/>se respeta ($setOnInsert, no $set)
        AU->>AU: signToken({sub, name: apodo, picture})
        AU-->>FE: 200 {token}
    end

    Note over FE: setSession(token) valida la FORMA del JWT<br/>(3 segmentos base64url, ≤4096) antes de guardarlo:<br/>nunca se persiste una cadena arbitraria
    FE->>FE: localStorage.setItem("token")
    FE->>GW: io(url, {auth: {token}})
    Note over GW: io.use(): jwt.verify(JWT_SECRET)<br/>+ rate limit de HANDSHAKE por IP
    GW-->>FE: connect
    FE->>J: navega a /lobby

    Note over AU: métricas: auth_login_success_total / auth_login_failed_total<br/>log: solo el id, NUNCA el apodo (evita inyección de logs)
```

**Detalle a defender:** `auth` registra únicamente el `id` en los logs, nunca el apodo. El
apodo lo escribe el usuario y no debe llegar crudo al log (hallazgo Sonar `S5145`).

---

## 2. Crear sala

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant GW as gateway :3000/:3010/:3020
    participant K as Kafka :9092
    participant RO as room :9101
    participant R as Redis :6379

    J->>FE: elige modo + nombre de sala
    FE->>GW: emit "room:create" {modo, name, nombreSala}
    Note over GW: ROUTES["room:create"] → topic cmd.room<br/>buildMessage(): playerId = socket.user.sub (del JWT)
    GW->>K: produce cmd.room (key = socket.id)
    K->>RO: consume cmd.room (grupo room-group)

    Note over RO: validarModo() · generarCodigo() (6 dígitos)<br/>normalizarNombreSala() · crearSala()

    RO->>R: SET sala:{codigo} EX 21600
    RO->>R: SET jugador:{id}:sala EX 43200
    RO->>R: SADD salas:activas

    RO->>K: produce gw.broadcast (room:created)
    RO->>K: produce gw.broadcast (room:joined)
    RO->>K: produce evt.room (RoomCreated)

    K->>GW: consume gw.broadcast
    GW->>FE: emit "room:created" {codigo, modo, nombre}
    GW->>FE: emit "room:joined" {jugadores, hostId}
    FE->>J: sala de espera con el código

    alt modo 1v1-bot
        Note over RO: no hay sala de espera: se publica RoomReady<br/>y la partida arranca sola en COLOCACION
    end
```

---

## 3. Unirse a una sala

```mermaid
sequenceDiagram
    autonumber
    actor J2 as Jugador 2
    participant FE as frontend
    participant GW as gateway
    participant K as Kafka
    participant RO as room :9101
    participant CH as chat :9103
    participant VO as voice-channel :9107
    participant R as Redis

    J2->>FE: escribe el código de 6 dígitos
    FE->>GW: emit "room:join" {codigo, name}
    GW->>K: produce cmd.room
    K->>RO: consume cmd.room
    RO->>R: GET sala:{codigo}

    alt La sala no existe
        RO->>K: gw.broadcast (room:error "sala_no_existe")
    else La sala está llena
        RO->>K: gw.broadcast (room:error "sala_llena")
    else La misma cuenta ya está dentro
        RO->>K: gw.broadcast (room:error "ya_estas_en_la_sala")
    else OK
        Note over RO: agregarJugador() → asignarEquipo()<br/>equilibra A/B automáticamente
        RO->>R: SET sala:{codigo} · SET jugador:{id}:sala
        RO->>K: produce gw.broadcast (room:join-socket-room)
        RO->>K: produce gw.broadcast (room:joined) → a TODOS
        RO->>K: produce evt.room (PlayerRoomJoined)

        par Reaccionan otros servicios
            K->>CH: consume evt.room
            CH->>K: gw.broadcast (chat:history)
        and
            K->>VO: consume evt.room
            Note over VO: prepara el canal de voz de la sala
        end

        K->>GW: consume gw.broadcast
        GW->>FE: emit "room:joined" (a ambos jugadores)
    end
```

---

## 4. Colocar los barcos

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant GW as gateway
    participant K as Kafka
    participant GA as game :9102
    participant TI as timer :9104
    participant R as Redis

    Note over FE: el anfitrión pulsó "Comenzar" → fase COLOCACION

    J->>FE: arrastra los 5 barcos al tablero
    opt Vista previa al compañero (solo 2v2)
        FE->>GW: emit "colocacion:preview" {codigo, ships}
        GW->>K: gw.broadcast (solo al compañero de equipo)
    end

    J->>FE: pulsa "Confirmar flota"
    FE->>GW: emit "colocacion:set" {codigo, ships}
    GW->>K: produce cmd.game (key = codigo)
    K->>GA: consume cmd.game

    GA->>R: SET lock:sala:{codigo} NX PX 3000
    GA->>R: GET sala:{codigo}

    Note over GA: validateFleet(ships): 5 barcos exactos,<br/>ids y tamaños de FLEET_CONFIG<br/>placeShip(): límites y solapamientos

    alt Flota inválida
        GA->>K: gw.broadcast (game:error: flota_incompleta /<br/>barcos_incorrectos / celda_ocupada / fuera_de_limites)
    else Flota válida
        GA->>R: SET sala:{codigo} (tablero del jugador)
        GA->>K: gw.broadcast (tu:flota → SOLO a ese jugador)

        alt Faltan jugadores por colocar
            Note over GA: espera · el timer de COLOCACION sigue corriendo
        else Todos colocaron
            Note over GA: nextFase(COLOCACION) → TURNOS
            GA->>K: produce gw.broadcast (phase:changed)
            GA->>K: produce evt.timer (arranca el turno)
            K->>TI: consume evt.timer
            TI->>K: produce evt.timer (tick cada 100ms)
        end
    end
    GA->>R: DEL lock:sala:{codigo}
```

**Nota sobre el bot:** en `1v1-bot`, `game` llama a `generarFlotaAleatoria()`
(`domain/autoFleet.js`), que coloca los 5 barcos sin solapes con hasta 300 intentos por barco.

---

## 5. Disparar

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant LB as nginx :8090
    participant GW as gateway gw2 :3010
    participant K as Kafka
    participant GA as game :9102
    participant TI as timer :9104
    participant BO as bot :9105
    participant OB as observability :9106
    participant R as Redis

    J->>FE: clic en celda rival
    FE->>LB: emit "disparo:realizar" {codigo, x, y}
    LB->>GW: least_conn

    Note over GW: INCR ratelimit:evt:{sub} (máx 30/s)<br/>si excede → gw:rate-limited y NO se publica

    GW->>K: produce cmd.game (key = codigo)
    K->>GA: consume cmd.game
    GA->>R: SET lock:sala:{codigo} NX PX 3000
    GA->>R: GET sala:{codigo}

    Note over GA: validateTurn() → ¿es su turno?<br/>shoot(board,x,y) → hit / miss / ya_tomada<br/>teamShieldActive() → ¿escudo del rival?

    alt No es su turno / fase incorrecta
        GA->>K: gw.broadcast (game:error)
    else Disparo válido
        Note over GA: energyGain() → +energía al equipo<br/>isShipSunk() / isFleetSunk()
        GA->>R: pipeline: SET sala + INCRBY energia:{equipo} + EXPIRE
        GA->>K: produce evt.game (Shot / ShipSunk / GameEnded)
        GA->>K: produce gw.broadcast (game:state)

        alt Flota rival hundida
            Note over GA: fase → FIN, winner = equipo
            GA->>K: produce evt.game (GameEnded)
            K->>OB: consume evt.game
            OB->>OB: persistirPartida() → Mongo (upsert idempotente)
        else Sigue la partida
            Note over GA: rotateTurn() · cada 3 rondas → fase SALVA
            GA->>K: produce evt.timer
            K->>TI: consume evt.timer
            TI->>K: produce evt.timer (tick del turno siguiente)
        end

        opt Modo 1v1-bot y toca al bot
            K->>BO: consume gw.broadcast
            Note over BO: mapaProbabilidad() + decidirDisparo()
            BO->>K: produce evt.bot
            K->>GA: consume evt.bot → resuelve el disparo del bot
        end
    end

    GA->>R: DEL lock (solo si el token sigue siendo suyo)
    K->>OB: consume evt.game → HINCRBY stats:kpi:{hoy}
    K->>GW: consume gw.broadcast (las 3 réplicas)
    GW->>FE: emit "game:state"
    FE->>J: tablero actualizado
```

---

## 6. Usar un poder

Cuatro poderes con coste en energía: **bombardeo** (2E), **sonar** (2E), **escudo** (1E) y
**tormenta** (3E, una sola vez por partida).

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant GW as gateway
    participant K as Kafka
    participant GA as game :9102
    participant R as Redis

    J->>FE: pulsa un poder en PowerPanel
    Note over FE: el botón ya está deshabilitado si:<br/>energia < coste · tormenta ya usada · no es su turno

    alt Poder con objetivo (bombardeo / sonar)
        J->>FE: además, clic en una celda rival
    end

    FE->>GW: emit "poder:usar" {codigo, poder, x?, y?}
    GW->>K: produce cmd.game
    K->>GA: consume cmd.game
    GA->>R: SET lock:sala:{codigo} NX PX 3000
    GA->>R: GET sala:{codigo} · GET sala:{codigo}:energia:{equipo}

    Note over GA: validatePower(id, energia)<br/>hasEnough() · ¿tormenta ya usada?

    alt Sin energía suficiente
        GA->>K: gw.broadcast (game:error "energia_insuficiente")
    else Poder inválido o ya usado
        GA->>K: gw.broadcast (game:error "poder_invalido" / "tormenta_ya_usada")
    else Válido
        alt bombardeo
            Note over GA: applyBombardeo() → área 3×3 del tablero rival
        else sonar
            Note over GA: applySonar() → revela 3×3 (no daña)
        else escudo
            Note over GA: applyEscudo() → protege a TODO el equipo<br/>del próximo impacto (consumeTeamShield)
        else tormenta
            Note over GA: applyTormenta() → turnosASaltar++<br/>tormentaUsada = true
        end

        GA->>R: DECRBY energia:{equipo} · SET sala:{codigo}
        GA->>K: produce evt.game (PowerUsed)
        GA->>K: produce gw.broadcast (poder:resultado + game:state)

        opt Poder contrarrestable (bombardeo / tormenta)
            Note over GA: se abre una VENTANA para que el rival<br/>use su contramedida (canCountermeasure)
            GA->>K: gw.broadcast (contramedida:disponible)
        end
    end
    GA->>R: DEL lock:sala:{codigo}
    K->>GW: gw.broadcast
    GW->>FE: emit "poder:resultado"
    FE->>J: efecto visual (celdas reveladas, explosión…)
```

---

## 7. Escribir en el chat

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant GW as gateway
    participant K as Kafka
    participant CH as chat :9103
    participant OB as observability
    participant R as Redis

    J->>FE: escribe y pulsa Enter
    FE->>GW: emit "chat:mensaje" {codigo, texto, canal?}
    Note over GW: rate limit por jugador (30/s)<br/>playerId del JWT, no del payload
    GW->>K: produce cmd.chat (key = codigo)
    K->>CH: consume cmd.chat (grupo chat-group)

    Note over CH: normalizarTexto() → trim<br/>esValido() → no vacío y ≤ MAX_LEN<br/>normalizarCanal() → "todos" | "equipo"

    alt Texto vacío o demasiado largo
        Note over CH: se descarta en silencio (no es un error del sistema)
    else Válido
        CH->>R: GET sala:{codigo}
        Note over CH: datosJugador() → apodo y equipo del emisor<br/>construirMensaje() → {de, texto, canal, ts}
        CH->>R: RPUSH sala:{codigo}:chat · EXPIRE

        loop Para cada destinatario
            Note over CH: puedeVerMensaje(msg, jugador)<br/>canal "equipo" → solo su bando
            CH->>K: produce gw.broadcast (chat:mensaje)
        end

        CH->>K: produce evt.chat (MessagePublished)
        K->>OB: consume evt.chat → KPIs
    end

    K->>GW: consume gw.broadcast
    GW->>FE: emit "chat:mensaje"
    FE->>J: mensaje en el panel
```

**Detalle:** el canal `equipo` se filtra **en el servidor**, no en el cliente. Si se filtrara
en el frontend, el mensaje del rival ya habría viajado al navegador y bastaría abrir las
herramientas de desarrollo para leerlo.

---

## 8. Entrar al chat de voz

La señalización va por el servidor; **el audio va directo entre navegadores** (WebRTC P2P),
sin pasar por la infraestructura.

```mermaid
sequenceDiagram
    autonumber
    actor J1 as Jugador 1
    actor J2 as Jugador 2
    participant FE1 as frontend (J1)
    participant GW as gateway
    participant K as Kafka
    participant VO as voice-channel :9107
    participant FE2 as frontend (J2)

    J1->>FE1: pulsa el micrófono
    Note over FE1: getUserMedia() → permiso del navegador
    FE1->>GW: emit "voice:join" {codigo, canal}
    GW->>K: produce cmd.voice
    K->>VO: consume cmd.voice (grupo voice-group)

    Note over VO: vozHabilitada(modo) · normalizarCanal()<br/>peersParaJugador() → con quién debe hablar

    VO->>K: produce gw.broadcast (voice:peers → lista de pares)
    K->>GW: consume gw.broadcast
    GW->>FE1: emit "voice:peers" [J2]

    Note over FE1,FE2: Negociación WebRTC — el gateway solo RELEVA,<br/>no interpreta el contenido
    FE1->>GW: emit "voice:signal" {to: J2, data: offer}
    Note over GW: voice:signal NO va a Kafka: se publica en<br/>gw.broadcast porque el destinatario puede estar<br/>en OTRA réplica de gateway (un io.to() local no llegaría)
    GW->>K: produce gw.broadcast (dirigido a J2)
    K->>GW: consume
    GW->>FE2: emit "voice:signal" {from: J1, data: offer}

    FE2->>GW: emit "voice:signal" {to: J1, data: answer}
    GW->>FE1: emit "voice:signal" {from: J2, data: answer}

    loop Candidatos ICE
        FE1->>GW: voice:signal {candidate}
        GW->>FE2: voice:signal {candidate}
    end

    FE1-->>FE2: 🔊 AUDIO P2P (RTP/SRTP, no pasa por el servidor)

    opt Silenciar
        J1->>FE1: pulsa mute
        FE1->>GW: emit "voice:mute" {codigo, muted}
        GW->>K: cmd.voice → VO → gw.broadcast
        GW->>FE2: emit "voice:estado" (indicador visual)
    end

    opt Salir de la voz
        J1->>FE1: pulsa colgar
        FE1->>GW: emit "voice:leave" {codigo}
        GW->>K: cmd.voice
        K->>VO: consume → recalcula peers
        VO->>K: gw.broadcast (voice:peers actualizado)
    end
```

---

## 9. Fin de partida y revancha

```mermaid
sequenceDiagram
    autonumber
    participant GA as game :9102
    participant K as Kafka
    participant OB as observability :9106
    participant MG as MongoDB
    participant GW as gateway
    participant FE as frontend
    participant RO as room :9101
    participant R as Redis
    actor J as Jugador

    Note over GA: isFleetSunk() → toda la flota rival hundida
    GA->>R: SET sala:{codigo} (fase FIN, winner, terminadoEn)
    GA->>K: produce evt.game (GameEnded)
    GA->>K: produce gw.broadcast (game:state con fase FIN)

    K->>OB: consume evt.game
    OB->>MG: partidas.updateOne({codigo}, $setOnInsert, upsert)
    Note over OB: IDEMPOTENTE: si Kafka reentrega el GameEnded,<br/>el índice único de `codigo` impide duplicar la fila
    OB->>R: HINCRBY stats:kpi:{hoy} games_ended

    K->>GW: consume gw.broadcast
    GW->>FE: emit "game:state" {fase: "FIN", winner}
    Note over FE: overlay dentro de la partida:<br/>"¡Ganaste!" / "Derrota"<br/>(no hay pantalla de estadísticas aparte)
    FE->>J: resultado + 2 acciones

    alt "Jugar otra vez"
        J->>FE: pulsa
        FE->>GW: emit "room:volver" {codigo, name}
        GW->>K: produce cmd.room
        K->>RO: consume cmd.room
        RO->>R: GET sala:{codigo}
        Note over RO: reiniciarParaRevancha():<br/>CONSERVA a los jugadores conectados,<br/>fase → LOBBY, borra tableros/turno/winner
        RO->>R: SET sala + DEL energia:A · energia:B · chat
        RO->>K: gw.broadcast (room:joined)
        GW->>FE: emit "room:joined"
        FE->>J: sala de espera, listos para otra ronda
    else "Elegir otro modo"
        J->>FE: pulsa
        FE->>GW: emit "room:salir" {codigo}
        GW->>K: produce cmd.room
        K->>RO: quitarJugador() → si queda vacía, destruye la sala
        RO->>R: DEL sala:{codigo} · DEL jugador:{id}:sala · SREM salas:activas
        FE->>J: navega al lobby
    end
```

**El fallo que corrige esta versión:** antes, "volver a la sala" dejaba en la sala **solo al
que pulsaba**, y el rival quedaba fuera sin enterarse. Además había una pantalla de
estadísticas aparte que repetía quién había ganado; se eliminó porque el overlay ya lo dice.

---

## 10. Cerrar sesión

```mermaid
sequenceDiagram
    autonumber
    actor J as Jugador
    participant FE as frontend
    participant ST as authStore
    participant SK as useSocket
    participant GW as gateway
    participant K as Kafka
    participant RO as room :9101
    participant R as Redis

    J->>FE: pulsa "Cerrar sesión"
    FE->>ST: clearSession()
    ST->>ST: localStorage.removeItem("token")
    ST->>SK: disconnectSharedSocket()
    SK->>GW: socket.disconnect()

    Note over GW: socket.on("disconnect")<br/>activeSocketsGauge.dec()

    GW->>K: produce cmd.room (PlayerDisconnected)
    K->>RO: consume
    RO->>R: GET sala:{codigo} (vía jugador:{id}:sala)
    Note over RO: marcarDesconectado(): conectado = false<br/>NO se le quita de la sala: puede reconectar
    RO->>R: SET sala:{codigo} (renueva TTL)
    RO->>K: produce evt.room (PlayerDisconnectedFromRoom)

    alt Quedan jugadores conectados
        Note over RO: la sala sigue viva ·<br/>game pausa el turno si era el suyo
        RO->>K: gw.broadcast (room:joined actualizado)
    else No queda nadie
        Note over RO: todosDesconectados() → destruir
        RO->>R: DEL sala:{codigo} + subclaves<br/>DEL jugador:{id}:sala · SREM salas:activas
    end

    FE->>J: navega a "/" (login)
```

**Por qué la desconexión no expulsa:** el jugador se marca como `conectado: false` pero
permanece en la sala. Si vuelve antes de que expire el TTL, el gateway lo reconecta a su
partida (`jugador:{id}:sala` lo localiza en O(1)) y el turno se reanuda. Solo cuando **todos**
están desconectados se destruye la sala.

---

## 11. Cómo se observa una jugada (métricas, logs y alertas)

Las nueve interacciones anteriores describen el **camino del juego**. Este describe el
**camino de la observabilidad**, que corre en paralelo y no bloquea ninguna jugada: es lo que
hace que un incidente se vea en vez de adivinarse.

```mermaid
sequenceDiagram
    autonumber
    participant GA as game :9102
    participant LF as logs/game.log
    participant PTL as promtail :9080
    participant LOK as Loki :3100
    participant PRO as Prometheus :9090
    participant ALM as Alertmanager :9093
    participant GRA as Grafana :3030
    actor OP as Operador

    Note over GA: se resuelve un disparo (interacción 5)

    par Métricas — Prometheus va a BUSCARLAS (pull)
        loop cada 5 segundos
            PRO->>GA: GET /metrics
            Note over GA: chequear() con límite de 1,5 s<br/>redis_node_up · mongo_up · kafka_consumer_up<br/>events_processed_total · event_processing_seconds
            GA-->>PRO: texto en formato Prometheus
        end
    and Logs — el servicio los EMPUJA (push)
        GA->>LF: línea JSON (pino) con name, level, cid
        PTL->>LF: sigue el archivo (tail)
        Note over PTL: pipeline: parsea el JSON,<br/>extrae etiquetas name / level_str / instancia,<br/>fecha con el campo `time` (no la hora de ingesta)
        PTL->>LOK: POST /loki/api/v1/push
    and Trazas — opcional
        GA-->>GA: si OTEL_EXPORTER_OTLP_ENDPOINT está definido,<br/>exporta spans a Jaeger :4318
    end

    Note over PRO: evalúa reglas cada 5 s<br/>slo.rules.yml calcula SLI y error budget

    alt Todo dentro de objetivo
        Note over PRO: las 13 alertas quedan en Inactive
    else Se incumple una condición
        Note over PRO: la alerta pasa a PENDING<br/>(aún no ha cumplido su `for:`)
        alt Fue un parpadeo y se recupera
            Note over PRO: vuelve a Inactive · NO se notifica
        else Persiste
            Note over PRO: pasa a FIRING
            PRO->>ALM: POST /api/v2/alerts
            Note over ALM: agrupa por alertname + job<br/>y las asigna al receptor ntfy-critico
        end
    end

    OP->>GRA: abre "BattleCaos — Operación"
    GRA->>PRO: consultas PromQL (18 paneles)
    GRA->>LOK: consultas LogQL (2 paneles de logs)
    GRA-->>OP: métricas y logs en la MISMA pantalla

    opt Investigar un incidente concreto
        OP->>GRA: Explore → Loki → {name=~".+"} |= "cid-abc"
        GRA->>LOK: query_range
        LOK-->>GRA: la traza del evento por los 9 servicios
        Note over OP: mismo correlationId → se reconstruye<br/>qué pasó y en qué orden
    end
```

**Los dos modelos que conviven, y conviene saber distinguir:**

| | Métricas | Logs |
|---|---|---|
| Quién toma la iniciativa | **Prometheus va a buscarlas** (pull, cada 5 s) | **El servicio los emite** y promtail los recoge (push) |
| Qué responden | "cuánto y con qué latencia" | "qué pasó exactamente, y a quién" |
| Coste | bajo y constante | crece con el volumen |

**Por qué `/metrics` tiene límite de tiempo:** ese endpoint pinguea Redis y Mongo. Sin un
tope, con Redis caído se quedaba colgado y Prometheus daba el target por muerto — el panel se
quedaba en blanco justo durante la incidencia. Con el límite de 1,5 s siempre responde y
publica `redis_node_up 0`, que es la información que hacía falta.

---

## Cómo ver estos diagramas

- **En GitHub**: se renderizan solos al abrir este archivo.
- **En VS Code**: extensión *Markdown Preview Mermaid Support*.
- **Exportar a imagen**: pega el bloque en <https://mermaid.live>.
