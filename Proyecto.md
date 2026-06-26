# Proyecto — BattleCaos-Ship

## Arquitectura Híbrida: Domain Kernel + Event-Driven + Microservicios

---

## Índice

1. [Resumen del Proyecto](#1-resumen-del-proyecto)
2. [Arquitectura General](#2-arquitectura-general)
3. [Catálogo de Microservicios](#3-catálogo-de-microservicios)
4. [Tecnologías](#4-tecnologías)
5. [Arquitectura Domain Kernel en Game Service](#5-arquitectura-domain-kernel-en-game-service)
6. [Comunicación entre Servicios](#6-comunicación-entre-servicios)
7. [Mapa de Eventos](#7-mapa-de-eventos)
8. [Despliegue y Escalabilidad](#8-despliegue-y-escalabilidad)
9. [Cumplimiento de Atributos de Calidad](#9-cumplimiento-de-atributos-de-calidad)
10. [Roadmap de Migración](#10-roadmap-de-migración)

---

## 1. Resumen del Proyecto

**BattleCaos-Ship** es un juego multijugador en tiempo real del estilo "Batalla Naval" con mecánicas avanzadas: poderes, energía, contra-medidas, salvas simultáneas y modo 2v2. El servidor es **autoritativo** (los clientes solo renderizan, nunca deciden resultados), con estado centralizado en Redis y comunicación vía WebSockets (Socket.io).

### Estado actual

- **Monolito modular** funcional con 19 archivos JavaScript (~729 LOC)
- Node.js 20 LTS + Express + Socket.io + ioredis
- Timer Master con leader election vía `SETNX` en Redis
- Diseñado para escalar horizontalmente con `@socket.io/redis-adapter`

### Arquitectura destino

Arquitectura **híbrida** que combina tres paradigmas:

| Paradigma | Propósito |
|-----------|-----------|
| **Domain Kernel** | El core del juego (reglas de negocio) es puro, sin dependencias de infraestructura. Los handlers orquestan llamando directamente a Redis y al dominio |
| **Event-Driven** | Los servicios se comunican exclusivamente mediante eventos asíncronos |
| **Microservicios** | Cada dominio (salas, juego, chat, bot, timer, auth, observabilidad) es un servicio independiente, desplegable y escalable por separado |

---

## 2. Arquitectura General

```
                         ┌──────────────────────────────────────────┐
                         │           API GATEWAY                    │
                         │  (Express + Socket.io + Rate Limiting)    │
                         │  - Auth middleware (JWT)                  │
                         │  - Enrutamiento de eventos por servicio   │
                         │  - Health check: GET /health              │
                         └──────────┬───────────────────────────────┘
                                    │
                         ┌──────────▼───────────────────────────────┐
                         │         MESSAGE BROKER                  │
                         │     (Redis Pub/Sub)                     │
                         │   Cada evento se publica una vez y       │
                         │   lo consumen los suscriptores           │
                         └──┬───┬───┬───┬───┬───┬───┬──────────────┘
                            │   │   │   │   │   │   │
              ┌─────────────┘   │   │   │   │   │   └──────────────┐
              │                 │   │   │   │   │                  │
     ┌────────▼──────┐  ┌──────▼───▼───▼───▼───▼───┐  ┌──────────▼──────┐
     │  Timer Service │  │     Game Service         │  │ Observability  │
     │  (micro)       │  │     (domain kernel)       │  │ (micro)        │
     │                │  │                           │  │                │
     │  Leader elect  │  │  Domain (puro):           │  │  Solo consume  │
     │  Timers reales │  │    GameEngine, Board,     │  │  eventos del   │
     │  Ticks Pub/Sub │  │    Fleet, Powers, Salvo,  │  │  broker        │
     │                │  │    Energy, Countermeasure │  │  KPIs y logs   │
     └────────────────┘  │                           │  └─────────────────┘
                          │  Handlers (orquestan):    │  ┌──────────────────┐
                          │    handleShot,              │  │  Bot Service     │
                          │    handlePower,             │  │  (micro)         │
                          │    handleSalvo              │  │                  │
                          │                            │  │  IA del bot      │
                          │  Redis directo:             │  │  Random/Minimax  │
                          │    redis.get/set,            │  │  Decide disparos  │
                          │    redis.incrby,             │  └──────────────────┘
                          │    redis.setnx               │
                          │    redis.publish             │  ┌──────────────────┐
                         └───────────────────────────┘  │  Auth Service     │
              ┌──────────────────┐  ┌──────────────────┐│  (micro)          │
              │  Room Service    │  │  Chat Service    ││                   │
              │  (micro)         │  │  (micro)         ││  JWT + Google     │
              │                  │  │                  ││  OAuth            │
              │  Crear/Unir salas│  │  Mensajes por    ││  Solo HTTP int.   │
              │  Matchmaking     │  │  equipo y global │└──────────────────┘
              │  Desconexión     │  │  Historial Redis │
              └──────────────────┘  └──────────────────┘
                                    ┌──────────────────┐
                                    │   Redis State    │
                                    │   (Upstash)      │
                                    │                  │
                                    │  - salas:{codigo}│
                                    │  - energía       │
                                    │  - locks salva   │
                                    │  - KPIs          │
                                    │  - chat history  │
                                    └──────────────────┘
```

> **⚠️ Nota sobre el alcance de este documento:** La arquitectura descrita aquí es la **visión destino** del proyecto. Varias piezas (separación en microservicios, Domain Kernel completo, Event Sourcing) están planificadas para Sprints futuros y no existen hoy en el código. El proyecto actual es un **monolito modular** funcional con 19 archivos. La sección [9.7](#97-resumen-de-cumplimiento--hoy-vs-meta) separa explícitamente lo que funciona hoy vs lo que es meta futura.
>
> **Tensión filosófica reconocida:** La arquitectura descentraliza el cómputo en 7 servicios independientes, pero centraliza el estado en un único Redis y el tráfico en un único Gateway. Esto crea 2 SPOFs que diluyen parcialmente la resiliencia ganada. Es una decisión deliberada: se prioriza simplicidad operativa y presupuesto $0 sobre disponibilidad absoluta. Para un proyecto académico con <100 jugadores simultáneos, es aceptable. Para producción, se mitigaría con Redis Sentinel/Cluster y Gateway redundante.

### Principios arquitectónicos

1. **Servidor autoritativo**: el servidor es la única fuente de verdad. Los clientes envían intenciones y reciben snapshots de estado. Nunca calculan resultados.
2. **Estado en Redis**: todo el estado de juego vive en Redis (Upstash), externo a todos los servicios. Si un servicio muere y reinicia, el estado no se pierde.
3. **Comunicación asíncrona**: los servicios se comunican exclusivamente vía eventos. No hay llamadas HTTP directas entre servicios de dominio (solo Auth Service vía HTTP desde Gateway).
4. **Consistencia fuerte en el core**: las operaciones críticas (disparos, energía, locks de salva) usan comandos atómicos de Redis (`INCRBY`, `SETNX`, `MULTI`/`EXEC`).
5. **Gateway como único punto de entrada**: los clientes solo conocen el Gateway. Los servicios internos no están expuestos públicamente.

---

## 3. Catálogo de Microservicios

### 3.1 Gateway

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Punto único de entrada. Enruta conexiones WebSocket al servicio correspondiente. |
| **Estado** | Nuevo |
| **Puerto** | 443 (público) |
| **Tecnología** | Express + Socket.io |
| **Escalabilidad** | Horizontal (N réplicas con balanceador HTTP) |

**Responsabilidades:**
- Recibir todas las conexiones Socket.io de los clientes
- Autenticar cada conexión contra Auth Service (JWT)
- Rate limiting por IP/token (prevenir DDoS)
- Re-enviar eventos entrantes al servicio correspondiente:
  - `room:create`, `room:join` → Room Service
  - `disparo:realizar`, `salva:disparo`, `poder:usar`, `colocacion:set`, `contramedida:activar` → Game Service
  - `chat:mensaje` → Chat Service
- Recibir broadcasts de los servicios y reenviarlos a los clientes
- Health check: `GET /health`

**Riesgo:** SPOF (Single Point of Failure). Mitigación: 2+ réplicas con balanceador externo (Render Load Balancer o Cloudflare).

---

### 3.2 Auth Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Autenticación y autorización |
| **Estado** | Existe como `auth/index.js` (35 LOC) |
| **Puerto** | 3001 (interno) |
| **Tecnología** | Express (solo HTTP, no WebSocket) |
| **Escalabilidad** | Vertical (baja carga, 1-2 réplicas) |

**Responsabilidades:**
- Verificar tokens JWT en cada conexión Socket.io (middleware)
- Generar tokens JWT a partir del payload de Google OAuth
- (Futuro) Refrescar tokens expirados

**Eventos que publica:** Ninguno (solo responde a llamadas HTTP síncronas del Gateway)
**Eventos que consume:** Ninguno

**Payload del JWT:**
```json
{
  "sub": "google-uid-123",
  "name": "Jugador1",
  "picture": "https://...",
  "iat": 1700000000,
  "exp": 1700086400
}
```

---

### 3.3 Room Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Ciclo de vida de salas (crear, unir, salir) |
| **Estado** | Existe como `rooms/index.js` (144 LOC) |
| **Puerto** | 3002 (WebSocket interno) |
| **Tecnología** | Socket.io + ioredis |
| **Escalabilidad** | Horizontal (por demanda de creación de salas) |

**Responsabilidades:**
- Escuchar eventos `room:create` y `room:join`
- Generar código único de 6 dígitos para cada sala
- Validar slots según modo (`1v1` = 2, `1v1-bot` = 1, `2v2` = 4)
- Asignar equipos (aleatorio en 2v2, A/B en 1v1)
- **Gestionar desconexiones sin perder al jugador:** al recibir `PlayerDisconnected` del Gateway, marcar `conectado: false` en la sala (sin eliminar al jugador ni su flota) y publicar `PlayerDisconnectedFromRoom`. Si el jugador reconecta, publicar `PlayerReconnected`. Solo eliminar la sala si todos los jugadores se fueron y venció el tiempo de espera.
- **Publicar evento `RoomReady`** cuando la sala está llena
- **Publicar evento `RoomDestroyed`** cuando ya no hay jugadores conectados ni posibilidad de reconexión

**Eventos que publica:**
| Evento | Trigger | Payload |
|--------|---------|---------|
| `RoomReady` | Sala llena y equipos asignados | `{ codigo, modo, equipos, jugadores }` |
| `PlayerDisconnectedFromRoom` | Gateway notifica socket caído; Room marca `conectado: false` | `{ codigo, playerId }` |
| `PlayerReconnected` | Jugador reconecta y Room reasocia la conexión | `{ codigo, playerId }` |
| `RoomDestroyed` | Todos los jugadores se fueron sin reconexión | `{ codigo }` |
| `PlayerRoomJoined` | Jugador entra a sala | `{ codigo, playerId, name }` |

**Eventos que consume:**
| Evento | Publicado por | Acción |
|--------|--------------|--------|
| `GameEnded` | Game Service | Limpiar sala (marcar para reúso o eliminar) |
| `PlayerDisconnected` | Gateway | Evento crudo: el socket se cayó. Room Service lo interpreta: marca `conectado: false` sin retirar al jugador, luego publica `PlayerDisconnectedFromRoom`. |

---

### 3.4 Game Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Core del juego: motor de fases, tableros, disparos, poderes, energía |
| **Estado** | Existe como `game/*` (215 LOC en 7 archivos) + `state/index.js` (37 LOC) |
| **Puerto** | Ninguno (solo Redis Pub/Sub, no expone HTTP ni WebSocket) |
| **Tecnología** | Node.js + ioredis (sin Socket.io — la comunicación es solo vía Redis) |
| **Escalabilidad** | **Horizontal** (es el servicio crítico, escala con partidas activas) |

**Responsabilidades:**
- Manejar las fases del juego: LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN
- Procesar colocación de barcos, disparos, poderes y contra-medidas
- Validar turnos, energía, reglas de juego
- Calcular impacto de disparos (agua, impacto, hundido)
- Manejar la salva simultánea con locks atómicos en Redis (`SETNX`)
- Publicar eventos de resultado a Redis Pub/Sub (el Gateway se encarga del broadcast a clientes)

**Arquitectura Domain Kernel** (detalle en sección 5).

**Eventos que publica:**
| Evento | Trigger | Payload |
|--------|---------|---------|
| `GameStarted` | Comienza COLOCACION | `{ codigo, modo, equipos, timestamp }` |
| `ShotFired` | Disparo procesado | `{ codigo, playerId, x, y, result, energiaRestante }` |
| `PhaseChanged` | Cambio de fase | `{ codigo, from, to }` |
| `PowerUsed` | Poder activado | `{ codigo, playerId, powerType, target }` |
| `PowerCompensated` | Contra-medida exitosa | `{ codigo, powerType, refund }` |
| `GameEnded` | Alguien gana | `{ codigo, winner, modo, duracion }` |
| `ShipsPlaced` | Jugador coloca flota | `{ codigo, playerId, equipo }` |
| `ShipSunk` | Barco hundido | `{ codigo, shipId, equipoAtacante, equipoDueno }` |

**Eventos que consume:**
| Evento | Publicado por | Acción |
|--------|--------------|--------|
| `RoomReady` | Room Service | Iniciar countdown de COLOCACION |
| `TimerEnd` | Timer Service | Forzar avance de fase (colocación agotada, turno agotado, salva terminada) |
| `PlayerDisconnectedFromRoom` | Room Service | Si era su turno, pausarlo (no pasarlo) hasta que reconecte o venza el tiempo de espera |
| `PlayerReconnected` | Room Service | Reanudar al jugador donde estaba |
| `BotDecision` | Bot Service | Procesar disparo del bot |

---

### 3.5 Timer Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Ejecutar y tickear todos los temporizadores del juego |
| **Estado** | Existe como `timer/master.js` (112 LOC) |
| **Puerto** | No expone puerto (solo Redis Pub/Sub) |
| **Tecnología** | Node.js puro + ioredis |
| **Escalabilidad** | Horizontal con leader election (1 master + N standbys) |

**Responsabilidades:**
- Leader election vía `SETNX` en Redis (solo 1 instancia es master activo)
- Ejecutar timers reales con `setTimeout`:
  - COLOCACION: 60s
  - TURNO: 30s
  - SALVA: 8s
  - CONTRAMEDIDA: 5s
- Publicar ticks de countdown cada 100ms para la UI del cliente
- Failover automático en ~1.5s si el master falla

**Motivo de separación:** Es el cuello de botella del monolito. Al separarlo, el Game Service no compite por CPU con los timers. Además, elimina la complejidad del leader election del servicio de juego.

**Eventos que publica:**
| Evento | Trigger | Payload |
|--------|---------|---------|
| `TimerEnd` | Timer expira | `{ codigo, tipo }` |
| `TimerTick` | Cada 100ms | `{ codigo, tipo, remaining }` |

**Eventos que consume:**
| Evento | Publicado por | Acción |
|--------|--------------|--------|
| `PhaseChanged` | Game Service | Iniciar timer correspondiente a la nueva fase |
| `RoomReady` | Room Service | Iniciar timer de COLOCACION |

---

### 3.6 Chat Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Mensajería en equipo y general |
| **Estado** | Existe como `chat/index.js` (46 LOC) |
| **Puerto** | 3004 (WebSocket interno) |
| **Tecnología** | Socket.io + ioredis |
| **Escalabilidad** | Horizontal (baja prioridad, puede escalar a 0) |

**Responsabilidades:**
- Recibir mensajes de chat (`chat:mensaje`)
- Persistir últimos 100 mensajes en Redis (`sala:{codigo}:chat`)
- Enrutar mensajes por equipo (2v2) o globales
- Publicar evento `ChatMessage` para observabilidad

**Motivo de separación:** No es crítico para el juego. Si el Chat Service se cae, las partidas continúan sin interrupción. Los mensajes se acumulan en Redis y se entregan cuando el servicio se recupera.

**Eventos que publica:**
| Evento | Trigger | Payload |
|--------|---------|---------|
| `ChatMessage` | Mensaje enviado | `{ codigo, senderId, senderName, text, equipo, timestamp }` |

**Eventos que consume:**
| Evento | Publicado por | Acción |
|--------|--------------|--------|
| `PlayerRoomJoined` | Room Service | Cargar historial de chat para el nuevo jugador |

---

### 3.7 Bot Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Inteligencia artificial del bot "Capitán Caos" |
| **Estado** | Existe como `bot/index.js` (22 LOC) |
| **Puerto** | 3005 (WebSocket interno) |
| **Tecnología** | Node.js puro + Redis |
| **Escalabilidad** | Horizontal (si la IA se vuelve compleja, escala con partidas vs bot) |

**Responsabilidades:**
- Escuchar eventos `ShotFired` en partidas modo `1v1-bot`
- Decidir la siguiente jugada del bot (hoy random, futuro minimax)
- Publicar evento `BotDecision` con el disparo calculado

**Motivo de separación:** Aísla la carga CPU-intensive de la IA. Si en el futuro la IA usa minimax, búsqueda en árbol o ML, no afecta la latencia del Game Service.

**Eventos que publica:**
| Evento | Trigger | Payload |
|--------|---------|---------|
| `BotDecision` | Bot decide su próximo disparo | `{ codigo, playerId, x, y }` |

**Eventos que consume:**
| Evento | Publicado por | Acción |
|--------|--------------|--------|
| `ShotFired` | Game Service | Si es modo bot y turno del bot, calcular respuesta |
| `PhaseChanged` | Game Service | Detectar inicio de turno del bot |

---

### 3.8 Observability Service

| Propiedad | Valor |
|-----------|-------|
| **Rol** | Métricas, KPIs, logging y auditoría |
| **Estado** | Existe como `observability/index.js` (54 LOC) |
| **Puerto** | Sin puerto (solo consume eventos) |
| **Tecnología** | Node.js + ioredis |
| **Escalabilidad** | Vertical (best-effort, 1 réplica) |

**Responsabilidades:**
- Consumir **todos** los eventos del broker
- Calcular KPIs:
  - Tasa de finalización (partidas terminadas / partidas iniciadas)
  - Latencia P95 de resolución de disparos
  - Tasa de reconexión (reconexiones / desconexiones)
  - Pico de salas concurrentes
- Almacenar KPIs en Redis (`stats:kpi:{date}`)
- Audit trail completo: cada evento se guarda con timestamp para no repudio

**Motivo de separación:** Es el servicio menos prioritario. Si se cae o va lento, el juego no se afecta en absoluto. Los eventos se acumulan en el broker hasta que el servicio los procese.

**Eventos que publica:** Ninguno (solo consume y escribe KPIs a Redis)
**Eventos que consume:** Todos (`*`)

---

## 4. Tecnologías

### 4.1 Stack de desarrollo

| Capa | Tecnología | Versión | Propósito |
|------|-----------|---------|-----------|
| **Runtime** | Node.js | 20 LTS | Ejecución del servidor |
| **Lenguaje** | JavaScript | ES2022 | Lenguaje del proyecto |
| **Web framework** | Express | ^4.21.0 | API REST y middleware HTTP |
| **WebSockets** | Socket.io | ^4.8.0 | Comunicación bidireccional en tiempo real |
| **Cliente Redis** | ioredis | ^5.5.0 | Acceso a Redis desde Node.js |
| **Autenticación** | jsonwebtoken | ^9.0.2 | Generación y verificación de JWT |
| **Autenticación** | google-auth-library | ^9.15.0 | Verificación de tokens de Google OAuth |
| **Seguridad HTTP** | helmet | ^8.0.0 | Headers de seguridad |
| **CORS** | cors | ^2.8.5 | Control de acceso cross-origin |
| **UUID** | uuid | ^11.0.0 | Generación de identificadores únicos |
| **Variables de entorno** | dotenv | ^16.4.0 | Carga de configuración desde `.env` |

### 4.2 Stack de infraestructura

| Componente | Tecnología | Proveedor | Propósito | Costo |
|-----------|-----------|-----------|-----------|:-----:|
| **Base de datos** | Redis | Upstash | Estado de juego, energía, locks, KPIs, chat | Gratis (100MB, 10K cmd/día) |
| **Broker de eventos** | Redis Pub/Sub | Upstash (misma instancia) | Comunicación asíncrona entre servicios | Incluido (misma conexión Redis) |
| **Balanceador de carga** | Render Load Balancer | Render | Distribución de tráfico entre réplicas | Incluido en plan Starter |
| **Hosting** | Render Web Service | Render | Despliegue del servidor Node.js | Gratis (1 servicio) / Starter ~$7 |
| **Frontend** | React + Vite | Vercel / Render Static | Cliente de juego | Gratis |

> **Nota de costos:** Todo el stack funciona en tier gratuito para el MVP. Render da 1 servicio web gratis + PostgreSQL (no usado aquí). Upstash da 100MB gratis. Vercel da hosting estático gratis. Si se necesita replicación (2+ instancias), Render Starter cuesta ~$7/mes por servicio.

### 4.3 Stack de desarrollo y herramientas

| Herramienta | Propósito |
|------------|-----------|
| **npm** | Gestor de paquetes |
| `node --watch` | Hot-reload en desarrollo |
| **PlantUML** | Diagramas de arquitectura (`.puml`) |

### 4.4 Justificación de cada tecnología

| Tecnología | ¿Por qué? | Alternativas consideradas | ¿Por qué no? |
|-----------|-----------|--------------------------|--------------|
| **Redis** | El proyecto ya lo usa. Es la base de datos ideal para estado de juego: rapidísima (sub-ms), comandos atómicos (`INCRBY`, `SETNX`), Pub/Sub nativo para eventos, soporta estructuras (listas para chat, streams para event sourcing). | PostgreSQL, MongoDB | Agregan latencia y no tienen operaciones atómicas tan naturales para juegos en tiempo real. |
| **Socket.io** | Conexiones persistentes, rooms (canales de broadcast), auto-reconexión, fallback a long-polling, adapter para Redis (escalabilidad horizontal). | WebSocket nativo (`ws`) | WebSocket nativo requeriría implementar rooms, reconexión y broadcast manualmente. |
| **Redis Pub/Sub** | El proyecto ya lo usa. Latencia sub-ms, cero configuración adicional, misma conexión Redis que el estado de juego. Suficiente para el volumen del MVP. | RabbitMQ (futuro) | RabbitMQ añade ~5-10ms de latencia y un servicio extra que administrar. Se migraría solo si el volumen supera 10K eventos/segundo. |
| **Node.js** | El equipo ya lo conoce. Es el runtime ideal para I/O intensiva (WebSockets + Redis). Para CPU intensiva (bot), se aísla en un servicio separado. | Python, Go, Rust | Costo de cambio alto para un proyecto académico. |

---

## 5. Arquitectura Domain Kernel en Game Service

El Game Service es el componente más crítico y donde se aplica la arquitectura **Domain Kernel**: handlers delgados que orquestan llamando directo a Redis y funciones de dominio puras.

### 5.1 Principio de capas

```
  REDIS (ioredis — llamado directo desde handlers)
     ↑  lee/escribe
     │
  HANDLERS (handleShot, handlePower, handleSalvo...)
     ↑  llaman
     │
  DOMAIN (engine, board, powers, salvo — funciones puras, 0 imports externos)
```

**El dominio es puro.** `engine.js` nunca importa `ioredis`, `socket.io` ni nada del `package.json`. Solo recibe datos y devuelve resultados. Los handlers son quienes orquestan: leen de Redis, llaman al dominio, escriben en Redis, publican eventos.

### 5.2 Estructura de carpetas

```
game/
├── domain/                          # PURO — 0 dependencias externas
│   ├── engine.js                    # GameEngine: fases, turnos, validaciones
│   ├── board.js                     # Board: grid, placeShip, shoot, markShot, isShipSunk
│   ├── fleet.js                     # Fleet: crear flota, verificar hundimiento
│   ├── powers.js                    # Powers: validar y ejecutar poderes (5 tipos)
│   ├── energy.js                    # Energy: sistema de energía (por equipo)
│   ├── salvo.js                     # Salvo: lógica de salva simultánea (pura)
│   ├── countermeasure.js            # Countermeasure: ventana de reacción
│   ├── player.js                    # Player: value object del jugador
│   └── errors.js                    # Errores del dominio (DomainError)
│
├── handlers/                        # ORQUESTACIÓN — llaman Redis + domain
│   ├── handleShot.js                # Recibe evento, lee Redis, llama processShot, escribe Redis, publica resultado
│   ├── handlePower.js              # Recibe evento, lee Redis, llama applyPower, escribe Redis, publica resultado
│   ├── handleSalvo.js              # Recibe evento, gestiona locks SETNX, llama resolveSalvo
│   ├── handlePlaceShips.js         # Recibe evento, valida flota, guarda en Redis
│   ├── handleDisconnect.js         # Recibe desconexión, pausa timers, notifica
│   └── broker.js                   # Consume eventos de Redis Pub/Sub y los enruta al handler correspondiente
│
└── redis.js                         # Conexión compartida a Redis (ioredis)
```

### 5.3 Ejemplo de flujo Domain Kernel

```javascript
// DOMAIN — puro
// engine.js
export function processShot(room, playerId, target, board) {
  if (room.fase !== 'TURNOS') return { ok: false, error: 'fase_incorrecta' };
  if (room.turno.jugadorActual !== playerId) return { ok: false, error: 'no_es_tu_turno' };
  const result = board.shoot(target.x, target.y);
  if (!result.valid) return { ok: false, error: 'coordenada_invalida' };
  return { ok: true, hit: result.hit, shipId: result.shipId, sunk: result.sunk };
}

// HANDLER — orquesta, no contiene lógica de negocio
// handleShot.js
import redis from '../redis.js';
import { processShot } from '../domain/engine.js';
import { markHit, isShipSunk } from '../domain/board.js';
import { addEnergy } from '../domain/energy.js';

export async function handleShot(event) {
  const room = JSON.parse(await redis.get(`sala:${event.codigo}`));
  const board = room.tableros[event.playerId];
  const result = processShot(room, event.playerId, { x: event.x, y: event.y }, board);

  if (!result.ok) {
    await redis.publish('game:resultado', JSON.stringify({ ok: false, error: result.error }));
    return;
  }

  markHit(board, event.x, event.y, result.hit);
  if (result.hit) {
    addEnergy(room, room.turno.equipoActual, 1);
    if (result.sunk) {
      addEnergy(room, room.turno.equipoActual, 3);
    }
  }

  await redis.set(`sala:${event.codigo}`, JSON.stringify(room));
  await redis.publish('game:resultado', JSON.stringify({
    codigo: event.codigo, playerId: event.playerId,
    x: event.x, y: event.y, hit: result.hit, sunk: result.sunk
  }));
}
```

### 5.4 Ventajas del Domain Kernel

| Aspecto | Hexagonal (plan original) | Domain Kernel (propuesta) |
|---------|--------------------------|---------------------------|
| Archivos por operación | ~5 (domain + port + adapter + use case + handler) | ~2 (domain + handler) |
| Flujo para entender | Handler → UseCase → Port → Adapter → Domain → Adapter | Handler → Redis → Domain → Redis (lineal) |
| Curva de aprendizaje | Alta (DI, puertos, inversión de dependencias) | Baja (funciones + Redis directo) |
| Tests de dominio | ✅ Puros (igual) | ✅ Puros (igual) |
| Tests de integración | Mockear N interfaces | Usar `ioredis-mock` (1 línea) |
| Cambiar Redis por X | ✅ Solo adapter nuevo | ❌ Tocar handlers (cambio mecánico, no lógico) |
| Velocidad de agregar features | ~1h por poder nuevo | ~15min por poder nuevo |

> **Decisión:** Se prioriza velocidad de desarrollo y simplicidad sobre el aislamiento de infraestructura. Redis no va a cambiar (ver §4.4), por lo que el acoplamiento handlers-Redis es aceptable. El dominio crítico sigue siendo puro y testeable.

### 5.5 Extensibilidad

La arquitectura Domain Kernel + Event-Driven + Microservicios hace que el proyecto sea extensible en 3 dimensiones:

#### 5.5.1 Extender la lógica del juego (nuevos poderes, armas, fases)

Cada nueva mecánica sigue el mismo patrón de **2 archivos**, sin modificar nada existente:

```
1. game/domain/miPoder.js     → función pura (valida, ejecuta, calcula)
2. game/handlers/miPoder.js   → orquesta: lee Redis, llama dominio, escribe Redis
3. game/handlers/broker.js    → 1 línea: registrar el evento
```

**Ejemplo concreto — agregar "Torpedo":**

```javascript
// 1. domain/torpedo.js — puro, testeable sin infraestructura
export function validateTorpedo(room, playerId) {
  if (room.fase !== 'TURNOS') return { ok: false, error: 'fase_incorrecta' };
  if (room.torpedosRestantes?.[playerId] <= 0) return { ok: false, error: 'sin_torpedos' };
  return { ok: true };
}

export function applyTorpedo(board, x, y) {
  // Impacto en 3x3 alrededor de (x, y)
  const hits = [];
  for (let dx = -1; dx <= 1; dx++) {
    for (let dy = -1; dy <= 1; dy++) {
      const result = board.shoot(x + dx, y + dy);
      if (result.hit) hits.push({ x: x + dx, y: y + dy, shipId: result.shipId });
    }
  }
  return { hits };
}

// 2. handlers/handleTorpedo.js — orquesta, sin lógica de negocio
import redis from '../redis.js';
import { validateTorpedo, applyTorpedo } from '../domain/torpedo.js';

export async function handleTorpedo(event) {
  const room = JSON.parse(await redis.get(`sala:${event.codigo}`));
  const validation = validateTorpedo(room, event.playerId);
  if (!validation.ok) return redis.publish('game:resultado', JSON.stringify(validation));

  const board = room.tableros[event.playerId];
  const result = applyTorpedo(board, event.x, event.y);
  room.torpedosRestantes[event.playerId]--;

  await redis.set(`sala:${event.codigo}`, JSON.stringify(room));
  await redis.publish('game:resultado', JSON.stringify({ codigo: event.codigo, ...result }));
}

// 3. broker.js — 1 línea
const handlers = {
  'torpedo:lanzar': handleTorpedo,  // ← única línea nueva
  // ... resto de handlers existentes
};
```

**Lo que NO se toca:** engine.js, board.js, ningún otro handler, ningún otro servicio, el Gateway, la comunicación entre servicios, el despliegue.

#### 5.5.2 Extender con nuevos servicios

Cada servicio nuevo es **independiente, desplegable y escalable por separado**. Solo necesita conectarse a Redis Pub/Sub y escuchar los eventos que le interesan:

| Nuevo servicio | Eventos que escucha | ¿Afecta al juego? |
|---|---|---|
| **Rankings** | `GameEnded`, `ShotFired` | ❌ No, juego sigue si se cae |
| **Replays** | Todos los eventos de sala | ❌ No, juego sigue si se cae |
| **Anti-cheat** | `ShotFired`, `PowerUsed` | ❌ No, solo alerta |
| **Notificaciones push** | `PhaseChanged`, `TimerTick` | ❌ No, son externas |

**Para crear un servicio nuevo:**

```
1. Crear carpeta: services/rankings/
2. Conectarse a Redis: const redis = createClient(process.env.REDIS_URL)
3. Suscribirse a eventos: redis.subscribe('game:resultado', handleGameEnd)
4. Procesar y guardar: redis.incrby('rankings:jugador:x', puntos)
5. Desplegar como servicio independiente en Render/Railway
```

**El servicio nuevo nunca toca el código de Game, Chat, Gateway ni ningún otro servicio.**

#### 5.5.3 Extender el Gateway (nuevos eventos del cliente)

El Gateway es un mapa plano de eventos Socket.io a canales de Redis:

```javascript
// Gateway — enrutamiento plano
socket.on('disparo:realizar', (data) => redis.publish('game:accion', { tipo: 'disparo', ...data }));
socket.on('torpedo:lanzar', (data) => redis.publish('game:accion', { tipo: 'torpedo', ...data })); // ← 1 línea
socket.on('chat:mensaje', (data) => redis.publish('chat:accion', data));
```

Agregar un nuevo evento de cliente es una línea. No hay lógica que validar aquí, solo enrutar.

#### 5.5.4 Resumen de extensibilidad

| Qué quieres agregar | Archivos a crear | Archivos a modificar | Tiempo estimado |
|---|---|---|---|
| Nuevo poder/arma | 2 (`domain/` + `handlers/`) | 1 (`broker.js` + 1 línea) | ~20-30 min |
| Nuevo servicio | 1 carpeta nueva | 0 | ~1-2h |
| Nuevo evento de cliente | 0 | 1 (Gateway, 1 línea) | ~5 min |
| Nueva fase de juego | 1 (`domain/`) | 1 (`handlers/`) | ~30 min |

---

## 6. Comunicación entre Servicios

### 6.1 Patrón: Event-Driven con Broker

Ningún servicio conoce la existencia de otro. Todos se comunican exclusivamente a través del **Message Broker**.

```
Servicio A → Broker: publish(EventoX)
Broker → Servicio B: consume(EventoX)
Broker → Servicio C: consume(EventoX)
```

### 6.2 Protocolos de comunicación

| Tipo | Protocolo | Uso |
|------|-----------|-----|
| Cliente → Gateway | WebSocket (Socket.io) | Todas las interacciones de juego |
| Gateway → Auth Service | HTTP | Verificación de JWT (síncrono, cacheable) |
| Servicios ↔ Broker | Redis Pub/Sub | Eventos asíncronos entre servicios |

### 6.3 Formato de eventos

```json
{
  "type": "ShotFired",
  "source": "game-service-instance-3",
  "timestamp": 1700000000123,
  "version": 1,
  "data": {
    "codigo": "482917",
    "playerId": "google-uid-456",
    "x": 3,
    "y": 5,
    "result": "hit",
    "shipId": "frigate_0"
  },
  "correlationId": "uuid-para-trazabilidad"
}
```

### 6.4 Garantías de entrega

| Escenario | Garantía | Mecanismo |
|-----------|----------|-----------|
| Redis Pub/Sub (hoy) | Best-effort | Si el consumidor no está conectado, el evento se pierde |
| RabbitMQ (futuro) | At-least-once | Colas persistentes + ACK |
| Comandos críticos (disparo, energía) | Exactly-once | Redis `MULTI`/`EXEC` atómico |

### 6.5 Manejo de errores entre servicios

```
Servicio A → publica evento → Broker
                                 ↓
Servicio B NO responde (caído o lento)
                                 ↓
Broker mantiene el evento en cola (RabbitMQ) o lo descarta (Redis)
                                 ↓
Servicio C (Observability) detecta ausencia de heartbeats
                                 ↓
Gateway: el cliente ve "sala:estado" con indicador de servicio degradado
```

**Regla:** Ningún servicio crítico (Game, Timer) depende de servicios no críticos (Chat, Observability, Bot) para funcionar.

---

## 7. Mapa de Eventos

### 7.1 Eventos del dominio de juego

```
GAME SERVICE
  │
  ├── GameStarted ──────────────→ Timer (inicia timer COLOCACION)
  │                              → Observability (métrica: game:start)
  │
  ├── ShotFired ────────────────→ Observability (latencia, accuracy)
  │                              → Bot (si modo bot, responde)
  │
  ├── PhaseChanged ────────────→ Timer (inicia timer de la nueva fase)
  │   (TURNOS→SALVA)            → Gateway (broadcast a clientes)
  │                              → Observability
  │
  ├── PowerUsed ───────────────→ Observability (uso de poder)
  │                              → Countermeasure (ventana de reacción)
  │
  ├── GameEnded ────────────────→ Room (limpiar sala)
  │                              → Observability (métrica: game:end)
  │
  └── ShipSunk ────────────────→ Observability
```

### 7.2 Eventos de infraestructura

```
TIMER SERVICE
  │
  ├── TimerEnd(codigo, COLOCACION) → Game (forzar fase TURNOS)
  ├── TimerEnd(codigo, TURNO)      → Game (cambiar turno o iniciar SALVA)
  ├── TimerEnd(codigo, SALVA)      → Game (cerrar salva, calcular resultados)
  └── TimerTick(codigo, tipo, remaining) → Gateway → todos los clientes

GATEWAY
  │
  └── PlayerDisconnected(codigo, playerId) → Room (única consumidora directa)
      Evento "crudo": el socket se cayó. Solo informa que la conexión
      física se perdió — todavía no se sabe si es un corte de red breve
      o un abandono. Quien decide eso es Room Service (ver 3.3).

ROOM SERVICE
  │
  ├── RoomReady(codigo, modo, equipos)         → Game, Timer (iniciar partida y su timer)
  ├── PlayerDisconnectedFromRoom(codigo, playerId)
  │     Se publica DESPUÉS de marcar conectado:false en la sala
  │     (no es lo mismo que el evento crudo de arriba)            → Game (pausar turno si era su turno)
  │                                                                → Timer (iniciar el tiempo de espera de DOMF1301)
  │                                                                → Chat, Observability (notificar)
  ├── PlayerReconnected(codigo, playerId)
  │     Se publica al reasociar la conexión y marcar conectado:true → Game (reanudar)
  │                                                                  → Timer (cancelar el tiempo de espera)
  │                                                                  → Chat, Observability (notificar)
  └── RoomDestroyed(codigo)
        Solo cuando ya nadie quedó conectado, o venció el tiempo
        de espera de todos los desconectados                       → Observability
```

### 7.3 Matriz publicador-consumidor

```
Evento                      Publica       Consume
───────────────────────   ───────────  ─────────────────────────
RoomReady                 Room         Game, Timer
PlayerDisconnected         Gateway      Room (evento crudo, solo Room lo interpreta)
PlayerDisconnectedFromRoom Room         Game, Timer, Chat, Observability
PlayerReconnected          Room         Game, Timer, Chat, Observability
RoomDestroyed              Room         Observability
GameStarted                Game         Timer, Observability
ShotFired                  Game         Observability, Bot
PhaseChanged                Game         Timer, Gateway, Observability
PowerUsed                   Game         Observability
PowerCompensated            Game         Observability
GameEnded                   Game         Room, Observability
ShipsPlaced                 Game         Observability
ShipSunk                    Game         Observability
TimerEnd                    Timer        Game
TimerTick                   Timer        Gateway (→ clientes)
BotDecision                 Bot          Game
ChatMessage                 Chat         Observability
```

---

## 8. Despliegue y Escalabilidad

### 8.1 Estrategia de escalamiento

| Servicio | Tipo | Estrategia | Disparador |
|----------|------|-----------|-----------|
| Gateway | Horizontal | Balanceador HTTP (Render LB) + N réplicas | Conexiones activas > 1000 |
| Auth Service | Vertical | 1-2 réplicas fijas | Baja carga (solo verifica tokens) |
| Room Service | Horizontal | N réplicas | Creación de salas > 50/min |
| **Game Service** | **Horizontal** | **N réplicas** | **Partidas activas > 50** |
| Timer Service | Horizontal | Leader election (1 master + N standby) | Solo 1 activo, failover automático |
| Chat Service | Horizontal | N réplicas (escalable a 0) | Mensajes > 1000/min |
| Bot Service | Horizontal | N réplicas | Partidas vs bot > 20 |
| Observability | Vertical | 1 réplica | Best-effort |

### 8.2 Diagrama de despliegue (MVP — presupuesto $0)

```
                         ┌─────────────────────┐
                         │   Render Load        │
                         │   Balancer           │
                         │   (opcional)         │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │   Gateway (x1)       │
                         │   Render Web Service │
                         │   (puerto 443)       │
                         └──────────┬──────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
    ┌─────▼─────┐           ┌───────▼───────┐          ┌──────▼──────┐
    │ Auth      │           │  Redis        │          │  Todo el    │
    │ (módulo)  │           │  Upstash      │          │  resto      │
    │           │           │  (estado +    │          │  en el      │
    │           │           │   Pub/Sub)    │          │  mismo      │
    │           │           │               │          │  proceso:   │
    │           │           │               │          │  Game,      │
    │           │           │               │          │  Room,      │
    └───────────┘           └───────────────┘          │  Chat,      │
                                                       │  Timer,     │
                                                       │  Bot,       │
                                                       │  Obs        │
                                                       └─────────────┘
```

> **MVP:** 1 servicio web en Render + 1 Redis en Upstash. Todo gratis.

### 8.2b Diagrama de despliegue (post-MVP — escalado horizontal)

```
                         ┌─────────────────────┐
                         │   Render Load        │
                         │   Balancer           │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │   Gateway x2-3       │
                         │   (Web Service)     │
                         └──────────┬──────────┘
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │                         │                          │
    ┌─────▼─────┐           ┌───────▼───────┐          ┌───────▼───────┐
    │ Auth x1   │           │  Redis        │          │  Game,Room,   │
    │           │           │  Upstash      │          │  Chat,Timer,  │
    │           │           │  (HA: Sentinel│          │  Bot,Obs —    │
    │           │           │   o Cluster)  │          │  cada uno en  │
    └───────────┘           └───────────────┘          │  su propio    │
                                                       │  Web Service  │
                                                       │  (Render)     │
                                                       └───────────────┘
```

> **Post-MVP:** Si se necesita, cada módulo puede extraerse a su propio servicio en Render. El costo sería ~$7/mes por servicio adicional + plan pago de Upstash (~$15/mes).

### 8.3 Estrategia de failover

| Fallo | Comportamiento |
|-------|---------------|
| **Game Service** cae | Clientes pierden conexión temporalmente. Gateway los redirige a otra réplica. El estado está en Redis, la nueva réplica carga el estado y continúa. |
| **Timer Service master** cae | Heartbeat se detiene. Redis TTL de 1s expira. Otra instancia toma el lease con `SETNX`. Failover en ~1.5s. Ventana de gracia de 1s para contra-medidas. |
| **Chat Service** cae | Los mensajes no se entregan temporalmente. El juego continúa sin chat. Cuando el servicio se recupera, lee los mensajes acumulados en Redis. |
| **Redis** cae | **SPOF crítico.** Sin Redis no hay estado, locks, energía ni Pub/Sub. Todo el sistema se detiene. Mitigación en MVP: backups automáticos de Upstash (restauración manual). A futuro: Upstash soporta Redis Sentinel (HA automático con réplica en otra zona) o Redis Cluster (sharding + replicación). Con Event Sourcing (Sprint 6), se reconstruye el estado desde los streams. |
| **Gateway** cae | **SPOF.** Mitigación en MVP: 1 sola instancia, el servicio se degrada pero Render reinicia automáticamente. A futuro: 2+ réplicas con Render Load Balancer o Cloudflare. |

---

## 9. Cumplimiento de Atributos de Calidad

A continuación se evalúa la arquitectura híbrida contra cada atributo de calidad definido en `Criterios_de_calidad.md`, citando las tácticas específicas implementadas.

---

### 9.1 Disponibilidad

*Definición: Capacidad del sistema de estar operativo y accesible cuando se requiere.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Heartbeat** | Timer Service: heartbeat cada 500ms renovando lease en Redis. Todos los servicios exponen `GET /health` para el balanceador. |
| **Ping/Echo** | Socket.io tiene ping/pong nativo (pingInterval: 25s, pingTimeout: 20s). Gateway monitorea conexiones. |
| **Active Redundancy** | Timer Service: leader election con failover automático en <1.5s. Game Service y demás: N réplicas activas detrás del balanceador. |
| **Passive Redundancy** | Gateway: 2+ instancias; si una falla, el balanceador envía todo a la otra. |
| **State Resynchronization** | Todo el estado en Redis (externo). Si un servicio reinicia, lee el estado actual sin pérdida. |
| **Transactions** | `MULTI`/`EXEC` de Redis para operaciones multi-key (ej: disparo + energía = atómico). |
| **Exception Detection** | Observability Service monitorea eventos del broker. Si un servicio deja de publicar, se detecta por ausencia de eventos. |
| **Rollback** | Tres capas: (1) Atomicidad Redis previene estados inconsistentes, (2) Saga compensatoria con eventos `PowerCompensated` para poderes+contramedida, (3) Event Sourcing con snapshots para reconstrucción completa desde eventos inmutables. |
| **Monitor** | ioredis con `retryStrategy` y `maxRetriesPerRequest` para reconexión automática a Redis. |

**Escenario del documento:**
- *Fuente:* Externa al sistema
- *Estímulo:* Mensaje inesperado / Crash de instancia
- *Respuesta:* Informar al operador, continuar operando (failover)
- *Métrica:* Sin tiempo de inactividad (No Downtime) — failover de Timer Service en <1.5s

**Brechas identificadas:**

| SPOF | Impacto | Mitigación hoy (MVP) | Mitigación futura |
|------|---------|---------------------|-------------------|
| **Redis** | Sin Redis todo el sistema se detiene | Backups automáticos Upstash + restauración manual | Redis Sentinel (failover automático multi-AZ) o Redis Cluster (sharding). Upstash lo ofrece en plan pago. |
| **Gateway** | Clientes no pueden conectarse | Render reinicia el servicio automáticamente en crash | 2+ réplicas con Render Load Balancer o Cloudflare |

> **Nota:** El sistema tiene 2 SPOFs reales (Redis y Gateway), lo que diluye parcialmente la resiliencia ganada al dividir en 7 servicios. Es una tensión deliberada: se priorizó simplicidad operativa y presupuesto $0 sobre disponibilidad absoluta. Para un proyecto académico con <100 jugadores simultáneos, es aceptable.

---

### 9.2 Modificabilidad

*Definición: Capacidad del sistema para ser modificado efectiva y eficientemente.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Split Module** | El monolito de 19 archivos se divide en 7 microservicios. Cada servicio es un módulo pequeño y con una responsabilidad única. |
| **Increase Semantic Coherence** | Cada microservicio cubre UN bounded context: Room, Game, Chat, Bot, Timer, Auth, Observability. Sin responsabilidades mezcladas. |
| **Encapsulate** | Domain Kernel: el dominio del Game Service es puro y no expone sus internos. Los handlers son la única capa que conoce Redis, el dominio nunca. |
| **Use an Intermediary** | El Message Broker es el intermediario entre servicios. Room Service nunca llama directo a Game Service; publica `RoomReady` y Game Service reacciona. |
| **Restrict Dependencies** | El dominio de Game Service tiene **0 dependencias externas**. No importa `ioredis`, `socket.io`, `express` ni ningún paquete del `package.json`. Solo Node.js puro. Los handlers son la única capa con acceso a Redis. |
| **Abstract Common Services** | Gateway abstrae autenticación, rate limiting y enrutamiento. Auth Service abstrae la lógica de JWT/Google OAuth. |

**Escenario del documento:**
- *Fuente:* Developer
- *Estímulo:* Desea cambiar la UI, agregar un poder nuevo, o cambiar Redis por PostgreSQL
- *Artefacto:* Código
- *Entorno:* Design time
- *Respuesta:* Cambio realizado y probado con pruebas unitarias
- *Métrica:* En tres horas

**Ejemplo concreto:** Agregar un nuevo poder "Torpedo" requiere:
1. Definir `Torpedo` en `game/domain/powers.js` (30 min)
2. Escribir tests unitarios sin infraestructura (30 min)
3. Agregar caso en `handlers/handlePower.js` (10 min)
4. Desplegar solo Game Service (5 min)

**Total: ~1h 15min.** Sin necesidad de crear puertos, adaptadores ni use cases.

---

### 9.3 Desempeño (Performance)

*Definición: Desempeño relativo a la cantidad de recursos utilizados bajo condiciones determinadas.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Introduce Concurrency** | Múltiples réplicas del Game Service manejan partidas en paralelo. Redis maneja la concurrencia atómicamente (`SETNX` para locks de salva). |
| **Maintain Multiple Copies** | Timer Service mantiene N candidatos a master (solo 1 activo). El resto espera como standby. |
| **Maintain Multiple Copies of Data** | Redis es la copia única de estado compartida. Los read models se cachean. Observability tiene sus propias copias de KPIs. |
| **Prioritize Events** | (Futuro con RabbitMQ) Los eventos del Game Service (ShotFired, PhaseChanged) tienen mayor prioridad que ChatMessage. |
| **Limit Event Response** | Rate limiting en el Gateway protege contra bursts de eventos de clientes maliciosos o maliciosos. |
| **Reduce Overhead** | Comunicación inter-servicios vía Redis Pub/Sub (sub-milisegundo) en lugar de HTTP, minimizando latencia. |
| **Increase Resources** | Cada servicio escala independientemente. Game + Timer escalan con partidas activas; Chat y Observability quedan igual. |
| **Bound Queue Sizes** | Chat: `LTRIM` mantiene solo últimos 100 mensajes. Redis TTL de 1s para locks de salva evita acumulación. |

**Escenario del documento:**
- *Fuente:* Usuarios (jugadores)
- *Estímulo:* Disparar en fase normal o salva simultánea
- *Artefacto:* Sistema (Game Service + Redis)
- *Entorno:* Operación normal, pico de 100 partidas concurrentes
- *Respuesta:* Disparo procesado y broadcast a todos los jugadores
- *Métrica:* Latencia promedio < 200ms | Cadencia de salva de 1.5s mantenida

**Estimación de latencia por disparo:**
| Paso | Tiempo |
|------|--------|
| Cliente → Gateway (WS) | 5ms |
| Gateway → Game Service (reenvío) | 2ms |
| Game Service: validación en dominio | <1ms (Node puro) |
| Game Service: Redis getRoom | 10ms |
| Game Service: procesar disparo | <1ms |
| Game Service: Redis saveRoom | 10ms |
| Game Service: Redis INCRBY energía | 5ms |
| Game Service: publicar evento | 2ms |
| Game Service: broadcast state | 5ms |
| **Total estimado** | **~40ms** |

La salva simultánea requiere cadencia de 1.5s entre disparos. Con 40ms de latencia, caben cómodamente ~37 disparos por jugador en la ventana de 8s.

---

### 9.4 Seguridad

*Definición: Capacidad de protección de la información y los datos contra accesos no autorizados.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Identify Actors** | Auth Service: JWT obligatorio en cada conexión Socket.io (middleware en Gateway). |
| **Authenticate Actors** | Google OAuth + JWT. El Gateway verifica el token antes de enrutar cualquier evento. |
| **Authorize Actors** | Socket.io middleware verifica que el `playerId` del JWT coincida con el `playerId` del evento. |
| **Separate Entities** | Microservicios en procesos separados (posiblemente contenedores distintos). Red privada interna. Solo Gateway tiene puerto público. |
| **Limit Access** | Servicios internos solo accesibles desde la red interna (misma red privada en Render o Docker network). Auth Service solo responde a Gateway. |
| **Encrypt Data** | Conexiones TLS entre Gateway y servicios internos. Redis soporta TLS nativamente. |
| **Maintain Audit Trail** | Observability Service registra TODOS los eventos con timestamp y correlationId. Esto constituye un audit trail completo para no repudio. |
| **Verify Message Integrity** | (Futuro) HMAC en mensajes del broker para garantizar que no fueron modificados en tránsito. |
| **Detect Service Denial** | Rate limiting en Gateway por IP/token. Límite de eventos por segundo por conexión. |

**Escenario del documento:**
- *Fuente:* Individuo correctamente identificado
- *Estímulo:* Intenta modificar información (ej: cambiar energía, alterar tablero)
- *Artefacto:* Datos dentro del sistema (Redis)
- *Entorno:* Operación normal
- *Respuesta:* El sistema mantiene una pista de auditoría (Audit Trail)
- *Métrica:* Los datos correctos son restaurados dentro de un día

El servidor autoritativo previene modificación directa: los clientes solo envían intenciones, nunca escriben a Redis. El audit trail de Observability + Event Sourcing (futuro) permiten restaurar el estado exacto a cualquier punto en el tiempo.

---

### 9.5 Testeabilidad

*Definición: Facilidad con la que se pueden establecer criterios de prueba y ejecutarlos.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Control & monitor state** | Domain Kernel: el dominio es puro y se prueba sin infraestructura. Los handlers se prueban con `ioredis-mock` (Redis en memoria) o tests de integración con Redis real. |
| **Execute test suite** | Cada microservicio tiene su propio test suite. El dominio del Game Service se prueba sin Redis ni Socket.io. |
| **Component interface for controlling behavior** | Las funciones del dominio son puras y reciben datos planos. Los handlers se pueden probar con una conexión Redis mockeada (`ioredis-mock`) o real. |
| **Capture cause of fault** | Los eventos del broker tienen `correlationId` para trazar el flujo completo de una operación. |
| **Boundary testing** | Las funciones del dominio (`processShot`, `validatePower`, `checkSink`) reciben datos planos y devuelven resultados planos. Sin side effects. |

**Ejemplo de test unitario del dominio (sin infraestructura):**

```javascript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { processShot } from '../domain/engine.js';
import { createBoard } from '../domain/board.js';

describe('GameEngine — processShot', () => {
  it('debe rechazar disparo fuera de turno', () => {
    const room = makeRoom({ fase: 'TURNOS', turno: { jugadorActual: 'playerA' } });
    const board = createBoard(10);
    board.placeShip('frigate_0', [{ x: 2, y: 3 }, { x: 2, y: 4 }, { x: 2, y: 5 }]);

    const result = processShot(room, 'playerB', { x: 5, y: 5 }, board);

    assert.equal(result.ok, false);
    assert.equal(result.error, 'no_es_tu_turno');
  });

  it('debe registrar impacto en barco', () => {
    const room = makeRoom({ fase: 'TURNOS', turno: { jugadorActual: 'playerA' } });
    const board = createBoard(10);
    board.placeShip('frigate_0', [{ x: 2, y: 3 }, { x: 2, y: 4 }, { x: 2, y: 5 }]);

    const result = processShot(room, 'playerA', { x: 2, y: 3 }, board);

    assert.equal(result.ok, true);
    assert.equal(result.hit, true);
    assert.equal(result.shipId, 'frigate_0');
  });
});
```

**Cobertura esperada:** 85%+ en el dominio del Game Service (todo el código puro). 60-70% en los handlers (requieren ioredis-mock o Redis real para integración).

---

### 9.6 Escalabilidad

*Definición: Propiedad de un sistema de manejar cantidades cada vez mayores de trabajo.*

| Táctica (del documento) | Implementación |
|------------------------|----------------|
| **Load Balancing** | Render Load Balancer o Cloudflare distribuye conexiones entre réplicas de cada servicio. Estrategias: Round Robin o por número de conexiones. |
| **Clustering** | Game Service: N réplicas como cluster. Timer Service: cluster con leader election (1 master + N standby). |
| **Asynchronous Processing (colas)** | Message Broker funciona como cola de trabajo. Game Service publica eventos; Observability los consume asíncronamente sin bloquear. Bot Service consume y responde sin bloquear. |
| **Horizontal Scaling** | Cada servicio escala independientemente. El Timer Service elimina el cuello de botella del Timer Master del monolito. |
| **SPOF Awareness** | Gateway y Redis son SPOF. Mitigación Gateway: 2+ réplicas con Render LB. Mitigación Redis: Upstash Sentinel/Cluster (plan pago, futuro). Por ahora, backups automáticos + restauración manual. |

**Estimación de capacidad:**

| Componente | 1 réplica | 5 réplicas | 10 réplicas |
|-----------|:---------:|:----------:|:-----------:|
| Gateway (conexiones) | 5,000 | 25,000 | 50,000 |
| Game Service (partidas) | 50 | 250 | 500 |
| Room Service (salas/min) | 100 | 500 | 1,000 |
| Chat Service (msg/min) | 10,000 | 50,000 | 100,000 |
| Bot Service (partidas) | 30 | 150 | 300 |
| Timer Service | 1 master + failover | 1 master + failover | 1 master + failover |
| Auth Service | 10,000 req/min | 20,000 req/min | — (no escala) |
| Observability | Best-effort | Best-effort | Best-effort |

**Cuello de botella:** Redis (Upstash) es el límite real. Upstash escala automáticamente con el plan, pero el ancho de banda y los comandos por segundo tienen un tope. Para 500+ partidas concurrentes (~4,000 comandos/segundo), se necesita plan Enterprise de Upstash.

---

### 9.7 Resumen de cumplimiento — HOY vs META

> **Importante:** Esta tabla evalúa la **arquitectura destino** (post-migración). Las tácticas marcadas como "(futuro)" no existen hoy en el código. Se incluyen para mostrar la visión completa, pero en la presentación debe aclararse qué está implementado y qué falta.

| Atributo | Hoy (monolito actual) | Meta (arquitectura híbrida) | Brechas |
|--------------------|:---:|:---:|---------|
| **Disponibilidad** | ❌ (50%) Timer Master en mismo proceso. Si crashea, no hay timers. Sin redundancia real. | ✅✅ (85%) Timer Service independiente con failover. Heartbeat en todos los servicios. Redis externo. | Gateway y Redis son SPOF. Redis Sentinel es (futuro). Rollback total requiere Event Sourcing (futuro). |
| **Modificabilidad** | ⚠️ (60%) Módulos separados por carpeta, pero game/* importa Redis directamente. No hay puertos/adaptadores. | ✅✅ (90%) Domain Kernel + Event-Driven + Microservicios. Dominio puro, handlers acoplados a Redis (aceptado). | Se pierde aislamiento total de infraestructura a cambio de simplicidad. Redis no cambiará. |
| **Desempeño** | ✅✅ (95%) Todo en un proceso. Sin latencia de red entre módulos. ~8ms por disparo. | ✅✅ (90%) Latencia estimada ~35ms por disparo (cruza Gateway + Broker). Sin overhead de capas de abstracción (puertos, adaptadores, inyección). | Mayor latencia es el costo de la descomposición. Domain Kernel reduce overhead al eliminar capas intermedias (~5ms menos por operación). |
| **Seguridad** | ✅✅ (90%) JWT en cada conexión. Auth module en el mismo proceso. Sin exposición de servicios. | ✅✅ (90%) Gateway centraliza autenticación. Servicios aislados. Audit trail completo. | HMAC en mensajes del broker es (futuro). |
| **Testeabilidad** | ❌ (30%) No se puede probar game/salvo.js sin Redis. No hay mocks ni puertos. | ✅✅ (95%) Dominio puro testeable sin infraestructura. Handlers requieren ioredis-mock o Redis real para integración. | Los handlers necesitan mock/infraestructura para pruebas, pero el dominio (90% de la lógica) se prueba sin nada. |
| **Escalabilidad** | ⚠️ (60%) Timer Master es cuello de botella (1 líder). Escala replicando, pero broadcast crece O(N²). | ✅✅ (90%) Cada servicio escala independientemente. Timer Service elimina cuello de botella. | Redis sigue siendo el límite real (Upstash plan gratuito: 10K cmd/día). Para 500+ partidas se necesita plan pago. |

---

## 10. Roadmap del Proyecto — 2 Sprints

La arquitectura se construye incrementalmente en **2 sprints**, alineados con las historias de usuario del `Inception.md`. No se separan microservicios como procesos independientes (eso es post-MVP); en su lugar, cada sprint avanza la madurez del **monolito modular** aplicando los patrones arquitectónicos de forma interna.

### Sprint 1 — Cimientos, Conexión y Core

| Épica | Historias | Entrega |
|-------|-----------|---------|
| **DOM-0** | DOMH01 a DOMH09 | Servidor autoritativo + auth + salas + estado RT + colocación + turnos + energía |
| **UIUX-1** | UIUXH01 a UIUXH08 | Login + lobby + tablero + chat + colocación + turnos + energía |
| **DOC-2** | DOCH01 a DOCH03 | Arquitectura + comunicación + motor del juego documentados |

**Refactor arquitectónico en Sprint 1:**

```
scripts/
  sprint1-refactor.js  ← script que describe el orden de los cambios
                          (no es código, es una guía de trabajo)

server/
├── config/              ← Config centralizada (ya existe)
├── auth/                ← JWT + Google OAuth (ya existe)
├── rooms/               ← Crear/unir salas (ya existe)
├── state/               ← Estado en Redis (ya existe)
├── game/
│   ├── domain/          ← Lógica pura (ya existe: engine, board, fleet, powers, salvo, countermeasure)
│   │   ├── engine.js
│   │   ├── board.js
│   │   ├── fleet.js
│   │   ├── powers.js
│   │   ├── salvo.js
│   │   └── countermeasure.js
│   └── handlers/        ← Orquestación (hoy VACÍOS — completar en Sprint 1)
│       ├── handlePlaceShips.js   → DOMH06
│       └── handleShot.js        → DOMH08
├── timer/master.js      ← Timer Master (ya existe)
├── chat/                ← Chat (ya existe)
├── bot/                 ← Bot básico (ya existe)
├── events/              ← Logging (ya existe)
└── index.js             ← Entry point (ya existe)
```

**Estructura Domain Kernel desde Sprint 1:**

El dominio se organiza desde el inicio con Domain Kernel:

```
game/
├── domain/              ← puro, sin imports de infraestructura
│   ├── engine.js
│   ├── board.js
│   ├── fleet.js
│   ├── powers.js
│   ├── energy.js
│   ├── salvo.js
│   └── countermeasure.js
└── handlers/            ← orquestan: leen Redis, llaman domain, escriben Redis
    ├── handleShot.js
    ├── handlePower.js
    ├── handleSalvo.js
    ├── handlePlaceShips.js
    └── broker.js
```

---

### Sprint 2 — Mecánicas, Observabilidad y Cierre

| Épica | Historias | Entrega |
|-------|-----------|---------|
| **DOM-0** | DOMH10 a DOMH15 | Poderes + contramedida + salva simultánea + bot + observabilidad + timeouts |
| **UIUX-1** | UIUXH09 a UIUXH13 | Panel de poderes + ventana de reacción + UI salva + panel operador + cierre |
| **DOC-2** | DOCH04 a DOCH05 | Concurrencia documentada + documentación de cierre |

**Refactor arquitectónico en Sprint 2:**

```
server/
├── game/
│   └── handlers/        ← Completar handlers RESTANTES:
│       └── handleSalvo.js  ('salva:disparo')        → DOMH12
│       └── handlePower.js  ('poder:usar')           → DOMH10
│       └── handleCM.js     ('contramedida:activar')  → DOMH11
├── observability/       ← KPIs + métricas (ya existe, completar DOMH14)
├── timer/master.js      ← Timeouts de turno (conectar DOMH15)
└── bot/                 ← Mejorar IA con búsqueda de adyacentes (DOMF1101)
```

**Mejora opcional — Event Sourcing básico (si el equipo quiere el reto técnico adicional):**

```
server/state/
├── index.js             ← read model (el de siempre)
└── eventStore.js        ← Redis Streams: append-only por sala
                          (sala:{codigo}:events)
                          Permite reconstruir estado desde eventos
                          y audit trail completo.
```

Esto NO es necesario para el MVP. Se incluye como documentación de visibilidad futura.

---

### Resumen de los 2 Sprints

```
 SPRINT 1 (6 semanas)                          SPRINT 2 (6 semanas)
 ┌─────────────────────────────┐              ┌─────────────────────────────┐
 │ DOMH01-DOMH09               │              │ DOMH10-DOMH15               │
 │ UIUXH01-UIUXH08             │              │ UIUXH09-UIUXH13             │
 │ DOCH01-DOCH03               │              │ DOCH04-DOCH05               │
 │                             │              │                             │
 │ [server]                    │              │ [server]                    │
 │  auth/        ✅            │              │  game/handlers/ (poderes)   │
 │  rooms/       ✅            │              │  game/handlers/ (salva)     │
 │  state/       ✅            │              │  game/handlers/ (CM)        │
 │  game/engine  ✅            │              │  observability/ (KPIs)      │
 │  game/board   ✅            │              │  timer/        (timeouts)   │
 │  game/fleet   ✅            │              │  bot/          (IA básica)  │
 │  game/powers  ✅            │              │                             │
 │  game/salvo   ✅            │              │                             │
 │  game/CM      ✅            │              │ [opcional]                  │
 │  timer/master ✅            │              │  state/eventStore (Streams) │
 │  chat/        ✅            │              └─────────────────────────────┘
 │  bot/         ⚠️ básico     │
 │  events/      ✅            │              DESPLIEGUE (ambos sprints):
 │                             │              ┌─────────────────────────┐
 │ [PENDIENTES]                │              │ 1 Web Service (Render)  │
 │  game/handlers/ (colocacion + disparos) │              │ 1 Redis (Upstash)      │
 │                              │              │ Ambos gratis            │
 │  rooms/reconexión           │              └─────────────────────────┘
 └─────────────────────────────┘
```
