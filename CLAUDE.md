# BattleCaos-Ship — Contexto maestro para Claude

## ¿Qué es este repositorio?
Repositorio de **coordinación y documentación** del proyecto BattleCaos-Ship.
No contiene código de producción. Aquí viven:
- `Orden_de_Ejecucion.md` — fuente de verdad de todas las tareas ordenadas por dependencia
- `docs/context/` — CLAUDE.md listo para copiar en cada repositorio de microservicio

## Proyecto
Juego naval multijugador real-time (materia ARSW 2026).
- Modos: 1v1, 1v1-Bot, 2v2
- Mecánica especial: Salva Simultánea (concurrencia), 5 poderes, energía atómica, Contramedida 5 s

## Organización GitHub: `BattleCaos-Ship`

| Repositorio | Tipo | Puerto | Descripción |
|---|---|:---:|---|
| `battlecaos-gateway` | Node.js + Socket.io | 3000 | Hub central: enruta eventos cliente → Redis |
| `battlecaos-auth` | Node.js + Express | 3001 | Google OAuth + emisión JWT |
| `battlecaos-room` | Node.js | — | Crear/unirse a salas, ciclo de vida |
| `battlecaos-game` | Node.js | — | Motor del juego: disparos, poderes, fases |
| `battlecaos-timer` | Node.js | — | Temporizadores de fases y turnos |
| `battlecaos-chat` | Node.js | — | Mensajería persistente por sala/equipo |
| `battlecaos-bot` | Node.js | — | IA para modo 1v1-Bot |
| `battlecaos-observability` | Node.js | — | KPIs, métricas, audit log |
| `battlecaos-frontend` | React + Vite | 5173 | Cliente web |

## Stack tecnológico
- **Runtime:** Node.js 20 LTS, ESM (`"type": "module"`)
- **Tiempo real:** Socket.io 4.x
- **Estado:** Redis (Upstash free tier) vía ioredis 5.x
- **Auth:** Google OAuth 2.0 + JWT (`jsonwebtoken`)
- **Frontend:** React 18 + Vite 5 + React Router 6
- **Tests:** Vitest (game domain) — sin TypeScript, sin ORM

## Principios arquitectónicos (de Proyecto.md)

1. **Servidor autoritativo:** el servidor es la única fuente de verdad. Los clientes envían intenciones y reciben snapshots. Nunca calculan resultados.
2. **Estado en Redis:** todo el estado vive en Redis (externo a los servicios). Si un servicio muere y reinicia, el estado no se pierde.
3. **Comunicación asíncrona:** los servicios se comunican solo vía eventos Redis Pub/Sub. No hay HTTP directo entre servicios de dominio (excepción: Auth vía HTTP desde Gateway).
4. **Consistencia fuerte en el core:** operaciones críticas usan comandos atómicos Redis (`INCRBY`, `SETNX`, `MULTI`/`EXEC`).
5. **Gateway como único punto de entrada:** los clientes solo conocen el Gateway. Los servicios internos no están expuestos públicamente.

## Patrón Domain Kernel (Game Service)

```
  REDIS ← handlers leen/escriben directamente
     ↑
  HANDLERS — orquestan: leen Redis, llaman domain, escriben Redis, publican
     ↑
  DOMAIN — funciones puras, 0 imports externos, testeables sin mocks
```

El dominio es puro (`engine.js`, `board.js`, etc.) — nunca importa `ioredis` ni `socket.io`.
Los handlers son delgados: lógica de negocio en domain, orquestación en handler.
Agregar poder nuevo = 1 archivo `domain/` + 1 archivo `handlers/` + 1 línea en `broker.js`.

## Reglas de implementación — NUNCA romper

1. **Naming Redis:** toda clave lleva prefijo `sala:{codigo}:...`
2. **Eventos Redis:** todo mensaje publicado incluye `data.codigo`
3. **Gateway solo enruta:** no contiene lógica de dominio, solo mapea Socket.io → Redis
4. **Game solo usa Redis:** no toca Socket.io directamente, publica en `gw:broadcast`
5. **Dos conexiones Redis en servicios con suscripción:** una para comandos, otra para `subscribe`
6. **Broadcasts de sala:** siempre `io.to(codigo).emit(...)`, nunca `io.emit(...)`
7. **redis.js se copia** en cada repo (no paquete npm compartido — MVP académico)
8. **`playerId` siempre de JWT:** nunca confiar en el `playerId` que manda el cliente en el body

## SPOFs y failover

| SPOF | Tiempo de failover | Mitigación MVP | Mitigación futura |
|---|---|---|---|
| **Redis (Upstash)** | ∞ (crítico) | Backups automáticos + restauración manual | Redis Sentinel / Cluster |
| **Gateway** | Render reinicia automáticamente (~30s) | 1 instancia | 2+ réplicas con Load Balancer |
| **Timer master** | ~1.5s | Leader election SETNX + heartbeat 500ms | Automático (ya implementado) |

## Canales Redis globales

| Canal | Dirección | Quién publica → quién consume |
|---|---|---|
| `svc:room` | Gateway → Room | Crear/unir salas |
| `svc:game` | Gateway → Game | Acciones de juego |
| `svc:chat` | Gateway → Chat | Mensajes |
| `gw:broadcast` | Servicios → Gateway | Respuestas a clientes |
| `evt:RoomReady` | Room → Game, Timer | Sala llena |
| `evt:PlayerDisconnectedFromRoom` | Room → Game, Timer, Chat, Obs | Desconexión |
| `evt:PlayerReconnected` | Room → Game, Timer, Chat, Obs | Reconexión |
| `evt:RoomDestroyed` | Room → Obs | Sala eliminada |
| `evt:PlayerRoomJoined` | Room → Chat | Nuevo jugador en sala |
| `evt:GameStarted` | Game → Timer, Obs | Inicio COLOCACION |
| `evt:ShotFired` | Game → Obs, Bot | Disparo procesado |
| `evt:PhaseChanged` | Game → Timer, Gateway, Obs | Cambio de fase |
| `evt:PowerUsed` | Game → Obs | Poder activado |
| `evt:PowerCompensated` | Game → Obs | Contramedida exitosa |
| `evt:GameEnded` | Game → Room, Obs | Victoria |
| `evt:ShipsPlaced` | Game → Obs | Flota colocada |
| `evt:ShipSunk` | Game → Obs | Barco hundido |
| `evt:TimerEnd` | Timer → Game | Tiempo agotado |
| `evt:TimerTick` | Timer → Gateway | Tick 100 ms para UI |
| `evt:BotDecision` | Bot → Game | Disparo del bot |
| `evt:ChatMessage` | Chat → Obs | Mensaje enviado |

## Claves Redis de estado

| Clave | Tipo | Contenido |
|---|---|---|
| `sala:{codigo}` | String (JSON) | Estado completo de sala/partida |
| `sala:{codigo}:chat` | List | Últimos 100 mensajes |
| `sala:{codigo}:energia:{equipo}` | String (int) | Energía acumulada |
| `sala:{codigo}:salva:{x}:{y}` | String | Lock SETNX para salva simultánea |
| `timer:leader` | String | Heartbeat del Timer activo |
| `stats:kpi:{YYYY-MM-DD}` | Hash | KPIs diarios |
| `stats:games:started` | String (int) | Partidas iniciadas |
| `stats:games:ended` | String (int) | Partidas completadas |

## Definition of Done (DoD) — Historia completa cuando cumple:

1. Código implementado y subido al repositorio en la rama correspondiente
2. Pruebas unitarias y de integración superadas
3. Criterios de aceptación Gherkin verificados
4. Code review aprobado por al menos un compañero
5. Sin bugs conocidos en la funcionalidad implementada
6. CI pasando todos los checks
7. Documentación actualizada (README o wiki)
8. Desplegada en entorno de pruebas y validada
9. Métrica de observabilidad registrada (logs, KPIs)

## Definition of Ready (DoR) — Historia lista para desarrollar cuando:

1. Redactada con formato "Como [actor], quiero [acción], para [beneficio]"
2. Actor asignado correctamente (Usuario / Desarrollador / Operador / Evaluador)
3. Criterios de aceptación en formato Gherkin (Given-When-Then)
4. Sin tecnología de implementación impuesta (describe qué, no cómo)
5. Factible técnicamente (validada con el equipo)
6. Estimada con puntos Fibonacci
7. Priorizada con MoSCoW (Must/Should/Could/Won't)
8. Vinculada a épica y tareas técnicas

## Escenarios de calidad clave (con métricas exigidas)

| ID | Deriva de | Métrica |
|---|---|---|
| **EC-01** | DOMH04 — Estado RT | Latencia P95 < 200ms entre acción y snapshot |
| **EC-02** | DOMH12 — Salva simultánea | 0% de casillas resueltas dos veces (lock SETNX) |
| **EC-03** | DOMH05 — Reconexión | >95% reconexiones exitosas, recuperación <3s |
| **EC-04** | DOMH02 — Auth | 0% de accesos no autorizados aceptados |
| **EC-05** | DOMH10 — Poderes | Nuevo poder: máx. 2 archivos, <3h implementación |
| **EC-06** | DOMH03 — Salas concurrentes | 0 fugas entre salas, sin degradación con ≥5 salas activas |

## Fecha de presentación: **8 de julio de 2026**

## Estado actual del proyecto

**Sprint:** 1 — Cimientos, Conexión y Core
**Fase:** 1 — Auth + Gateway completos
**Próximo repositorio a crear:** `battlecaos-game`
**Tarea:** DOMF302 + DOMF501 + DOMF502 + DOMF601 + DOMF602 + DOMF701

| Repositorio | Estado | Tareas completadas |
|---|---|---|
| `battlecaos-gateway` | ✅ completo | DOMF001, DOMF002, DOMF102, DOMF301, DOMF401, DOMF1302, UIUXF1101 |
| `battlecaos-auth` | ✅ completo | DOMF001, DOMF101, DOMF1302 |
| `battlecaos-room` | ✅ completo | DOMF001, DOMF201, DOMF202, DOMF203, DOMF402, DOMF403 |
| `battlecaos-game` | ⬜ pendiente | DOMF301, DOMF302, DOMF501, DOMF502, DOMF601, DOMF602, DOMF701 |
| `battlecaos-timer` | ⬜ pendiente | DOMF1303 |
| `battlecaos-chat` | ⬜ pendiente | DOMF1103 |
| `battlecaos-bot` | ⬜ pendiente | DOMF1101, DOMF1102 |
| `battlecaos-observability` | ⬜ pendiente | DOMF1201, DOMF1202, DOMF1203 |

> Actualizar esta sección cada vez que se avanza de fase o repositorio.

## Justificación de tecnologías (no cambiar sin razón de peso)

| Tecnología | Por qué se eligió |
|---|---|
| **Redis** | Comandos atómicos (`INCRBY`, `SETNX`), Pub/Sub nativo, sub-ms latency, estructuras (listas para chat). Ya en uso en el proyecto. |
| **Socket.io** | Rooms (broadcast a sala), auto-reconexión, fallback a long-polling, adapter Redis para escala horizontal. |
| **Redis Pub/Sub** | Sub-ms latency, 0 configuración extra, misma instancia Redis del estado. Suficiente para el volumen del MVP. |
| **Node.js** | El equipo lo conoce. Ideal para I/O intensiva (WebSockets + Redis). Para CPU intensiva (bot), se aísla en servicio separado. |
| **No TypeScript** | Decisión del equipo para velocidad académica. No agregar TS sin consenso del grupo. |

## Estimación de latencia por disparo

| Paso | Tiempo estimado |
|---|---|
| Cliente → Gateway (WebSocket) | 5ms |
| Gateway → publish Redis | 2ms |
| Game Service: GET sala de Redis | 10ms |
| Game Service: procesar disparo (domain puro) | <1ms |
| Game Service: SET sala + INCRBY energía | 15ms |
| Game Service: publish broadcast | 2ms |
| **Total estimado** | **~35-40ms** (meta: P95 < 200ms) |

## Cómo usar los contextos de cada repo
Cada archivo en `docs/context/` es el `CLAUDE.md` listo para ese repositorio.
Al crear el repo, copiar el archivo correspondiente como `CLAUDE.md` en su raíz.

| Archivo | Repo destino |
|---|---|
| `docs/context/CLAUDE.gateway.md` | `battlecaos-gateway/CLAUDE.md` |
| `docs/context/CLAUDE.auth.md` | `battlecaos-auth/CLAUDE.md` |
| `docs/context/CLAUDE.room.md` | `battlecaos-room/CLAUDE.md` |
| `docs/context/CLAUDE.game.md` | `battlecaos-game/CLAUDE.md` |
| `docs/context/CLAUDE.timer.md` | `battlecaos-timer/CLAUDE.md` |
| `docs/context/CLAUDE.chat.md` | `battlecaos-chat/CLAUDE.md` |
| `docs/context/CLAUDE.bot.md` | `battlecaos-bot/CLAUDE.md` |
| `docs/context/CLAUDE.observability.md` | `battlecaos-observability/CLAUDE.md` |
| `docs/context/CLAUDE.frontend.md` | `battlecaos-frontend/CLAUDE.md` |
