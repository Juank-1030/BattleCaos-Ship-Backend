# BattleCaos-Ship — Presentación de Arquitectura

> Documento fuente para las diapositivas. **El contenido fuera de los bloques "Notas del orador" es el texto literal de cada slide** — corto, directo, listo para proyectar. Las notas llevan el detalle para responder preguntas, no para leerlas en pantalla.

---

## Índice

1. Contexto del proyecto
   - 1.1 Mecánicas — diferenciadores
   - 1.2 Catálogo de poderes
2. Escenarios de calidad (4 ejemplos)
3. Requisitos de calidad seleccionados (4)
4. Argumentación de la arquitectura propuesta
5. Justificación tecnológica
6. Tensiones reconocidas
7. Diagramas de apoyo
8. Conclusiones

---

## 1. Contexto del proyecto

**BattleCaos-Ship** — batalla naval multijugador en tiempo real (1v1 · 1v1 vs Bot · 2v2)

> Proyecto académico de ARSW: un juego de batalla naval donde el servidor es la única fuente de verdad y todos los jugadores compiten en igualdad de condiciones, diseñado a propósito para poner a prueba la concurrencia y la comunicación en tiempo real de la arquitectura propuesta.

- Servidor autoritativo: el backend decide, el cliente solo renderiza
- Mecánicas diferenciadoras:
  - **Salva Simultánea** — todos disparan a la vez (ventana de 8s)
  - **Poderes + Contramedida** — energía acumulable, ventana de reacción 5s
  - **2v2 colaborativo** — flota en equipo + chat
- 4 KPIs de negocio: % partidas completadas · latencia P95 · % reconexiones · salas concurrentes

> **Notas del orador:**
> Proyecto académico de ARSW — el objetivo no es solo "un juego de batalla naval", sino demostrar manejo de concurrencia y tiempo real con una arquitectura justificada. La Salva Simultánea está diseñada a propósito para forzar condiciones de carrera reales (dos jugadores disparando a la misma casilla casi al mismo instante).
>
> Estado actual: monolito modular funcional, 19 archivos JS (~729 LOC), Node.js 20 + Express + Socket.io + ioredis. Roadmap de entrega: **2 sprints**, alineados con `Inception.md`. La arquitectura que se presenta es una evolución incremental de ese monolito, no un punto de partida desde cero.

---

## 1.1 Mecánicas — qué lo diferencia de un Battleship clásico

| Mecánica | Qué aporta |
|----------|------------|
| **Turnos clásicos** | Disparo por turno, agua / impacto / hundido — la base conocida |
| **Energía por equipo** | +1 por impacto, +3 por hundimiento — recurso estratégico, no cosmético |
| **Salva Simultánea** | Cada 3 rondas, ventana de 8s: todos disparan a la vez |
| **Poderes** | 5 habilidades que se compran con energía acumulada |
| **Contramedida** | Ventana de 5s para anular un poder enemigo |
| **2v2 colaborativo** | Flota y energía en equipo, chat privado de equipo |

> **Notas del orador:**
> Un Battleship clásico es puramente azar por turnos: no hay economía, no hay decisiones tácticas más allá de "dónde disparo", y no hay ningún momento de concurrencia real. BattleCaos-Ship le agrega 4 capas de profundidad estratégica sobre esa base:
> 1. **Energía como recurso** convierte cada impacto en una decisión de "gastar ahora o acumular para un poder más fuerte".
> 2. **Salva Simultánea** rompe el turno estricto cada 3 rondas: durante 8s todos disparan al mismo tiempo, con una cadencia mínima de 1.5s entre disparos propios — mide quién mantiene la cabeza fría bajo presión y, arquitectónicamente, es la mecánica que existe específicamente para demostrar concurrencia.
> 3. **Poderes** convierten "dónde disparo" en "qué herramienta uso" — cada uno con su propio costo y momento óptimo de uso.
> 4. **Contramedida** agrega una capa reactiva: ya no basta con atacar bien, hay que defenderse a tiempo.
>
> Esto es lo que justifica, de cara al jurado, por qué el proyecto necesita una arquitectura más sofisticada que un CRUD simple: cada mecánica nueva (Salva, poderes, contramedida) introduce un requisito de calidad concreto (concurrencia, modificabilidad, tiempo real) que un Battleship de turnos no tendría.

---

## 1.2 Catálogo de poderes

| Poder | Costo | Tipo | Efecto |
|-------|:-----:|------|--------|
| **Bombardeo de área** | 2 | Ofensivo | Impacta una zona de 3×3 casillas |
| **Sonar** | 2 | Inteligencia | Revela si hay barco en una fila/columna completa |
| **Escudo** | 1 | Defensivo | Protege una casilla propia por 2 turnos |
| **Tormenta** | 3 | Sabotaje (1 vez/partida) | Anula el siguiente turno del rival |
| **Contramedida** | — | Reactivo | Anula un poder ofensivo si se activa dentro de la ventana de 5s |

> **Notas del orador:**
> Los poderes se dividen en 3 categorías con propósitos distintos:
> - **Ofensivos** (Bombardeo, Sonar): aceleran el ataque — Bombardeo sacrifica precisión por área, Sonar sacrifica daño por información.
> - **Defensivo/Sabotaje** (Escudo, Tormenta): cambian el ritmo del rival en vez de atacar directamente — Tormenta es de un solo uso por partida, así que su momento de activación es una decisión de alto riesgo.
> - **Reactivo** (Contramedida): es el único poder que no se "usa", se "activa justo a tiempo" — y solo aplica contra Bombardeo o Sonar (los ofensivos). Si el defensor reacciona dentro de los 5s, el poder se anula y la energía gastada se devuelve íntegra al atacante.
>
> Este catálogo es también el ejemplo vivo de Modificabilidad (`EC-05`, sección 3): agregar un sexto poder, en la arquitectura hexagonal propuesta, debería tocar como máximo el archivo del dominio de poderes — no el motor de turnos, no la energía, no la salva.

---

## 2. Escenarios de calidad (4 ejemplos)

Un escenario de calidad = requisito medible en 6 partes: Fuente, Estímulo, Artefacto, Entorno, Respuesta, Medida.

`Inception.md` documenta 6 (`EC-01`–`EC-06`); se presentan 4 como ejemplo, uno por atributo.

### EC-02 — Concurrencia sin condiciones de carrera

| | |
|---|---|
| Deriva de | `DOMH12` — Ronda de disparo simultáneo |
| Estímulo | Dos usuarios disparan casi al mismo tiempo a la misma casilla |
| Respuesta | Solo el primero en llegar resuelve la casilla |
| **Medida** | **0% de casillas resueltas dos veces** |

### EC-03 — Disponibilidad ante caída de red

| | |
|---|---|
| Deriva de | `DOMH05` — Recuperación tras desconexión |
| Estímulo | Usuario pierde la conexión y vuelve a conectarse |
| Respuesta | El sistema reasocia al usuario y entrega el estado completo |
| **Medida** | **>95% reconexiones exitosas**, recuperación **<3s** |

### EC-04 — Seguridad de acceso

| | |
|---|---|
| Deriva de | `DOMH02` — Identificación segura del usuario |
| Estímulo | Usuario sin credencial válida intenta conectarse |
| Respuesta | El sistema rechaza la conexión sin exponer estado |
| **Medida** | **0% de accesos no autorizados aceptados** |

### EC-06 — Escalabilidad por aislamiento de salas

| | |
|---|---|
| Deriva de | `DOMH03` — Salas mediante código |
| Estímulo | Varias salas activas simultáneamente |
| Respuesta | Cada sala opera aislada; ningún evento se filtra a otra |
| **Medida** | **0 fugas entre salas**, con 5+ salas concurrentes |

> **Notas del orador:**
> Se eligieron estos 4 (de 6) porque cubren atributos distintos sin solaparse:
> - **EC-02** es el más importante: demuestra que la Salva Simultánea no produce dos "ganadores" sobre la misma casilla. Apoyarse en el diagrama de secuencia de la sección 7 (`08-secuencia-salva-simultanea.puml`) y explicar el mecanismo: `SETNX` atómico en Redis como árbitro, sin importar a qué réplica del Game Service llegó cada petición.
> - **EC-03** tiene la mejor narrativa: en esta sesión de trabajo se encontró que el código real (`rooms/index.js`) hacía lo contrario de lo que pide este escenario — expulsaba al jugador apenas se desconectaba. Mostrar el "antes/después" de esa corrección demuestra que la arquitectura ya encontró y corrigió un defecto real, no es solo teoría.
> - **EC-04** es el más sencillo: sin esto, cualquiera podría falsificar su identidad y alterar una partida ajena — rompe el pilar de "servidor autoritativo".
> - **EC-06** conecta directo con un KPI de negocio ya declarado ("salas concurrentes sostenidas").
>
> `EC-01` (Desempeño general) y `EC-05` (Modificabilidad de poderes) siguen documentados en `Inception.md`; se mencionan solo si el jurado pregunta.

---

## 3. Requisitos de calidad seleccionados (4)

| Atributo | Por qué este proyecto lo necesita | Hoy → Meta |
|----------|-------------------------------------|:---:|
| **Desempeño** | La Salva exige resolver disparos concurrentes en milisegundos | 95% → 80% |
| **Disponibilidad** | Multijugador real-time sobre redes inestables: las caídas son la norma | 50% → 85% |
| **Seguridad** | El "servidor autoritativo" solo vale si nadie no identificado puede alterar una partida | 90% → 90% |
| **Escalabilidad** | "Salas concurrentes sostenidas" es un KPI de negocio declarado | 60% → 90% |

> **Notas del orador:**
> Cada atributo tiene su ejemplo de aplicación:
> - **Desempeño:** el Game Service usa `SETNX` de Redis como árbitro atómico en vez de un `if` en memoria — es lo único que garantiza `EC-02` aunque dos disparos lleguen a réplicas distintas. Es el único atributo que *empeora* en la meta (95%→80%), por la latencia de red entre servicios (~8ms→~40ms) — trade-off aceptado a propósito (sección 6).
> - **Disponibilidad:** al desconectarse, el jugador queda marcado `conectado: false` sin ser retirado de la sala; si reconecta a tiempo, recupera el estado exacto (`EC-03`). Hoy 50% porque el Timer Master no tiene redundancia real si crashea el proceso; meta 85% con Timer Service independiente y failover <1.5s.
> - **Seguridad:** cada conexión Socket.io pasa por un middleware que verifica el JWT antes de aceptar cualquier evento (`EC-04`). Se mantiene en 90%→90% porque ya estaba bien resuelta en el monolito — separar en microservicios no la mejora por sí sola.
> - **Escalabilidad:** cada sala vive bajo su propia clave Redis (`sala:{codigo}`), sin fuga de eventos entre salas (`EC-06`). Hoy 60% porque el Timer Master es cuello de botella y el broadcast crece O(N²); meta 90% con cada servicio escalando de forma independiente.
>
> Trazabilidad completa: **KPI de negocio → historia de usuario → escenario de calidad → táctica arquitectónica → tecnología concreta.** Modificabilidad y Testeabilidad (no presentados como slide propia) son, de hecho, los que más mejoran (30%→100% y 60%→100%) — mencionar al cierre si el tiempo lo permite.

---

## 4. Argumentación de la arquitectura propuesta

### 4.1 Principios que gobiernan toda decisión

1. Servidor autoritativo — el backend decide, nunca el cliente
2. Estado en Redis — externo a todos los servicios
3. Comunicación asíncrona — sin llamadas directas entre servicios
4. Consistencia atómica — `INCRBY`, `SETNX`, `MULTI`/`EXEC`
5. Gateway único — único punto de entrada público

> **Notas del orador:**
> Estos 5 principios (`Proyecto.md §2`) son el criterio con el que se evalúa cualquier decisión técnica posterior — no son decoración, son la prueba de consistencia: si una pieza nueva de la arquitectura los viola, está mal diseñada. Ejemplo: el principio #2 es la razón de fondo por la que un servicio puede reiniciarse sin perder partidas — el estado nunca vivió en su memoria.

### 4.2 Por qué una arquitectura híbrida

| Opción | Resultado |
|--------|-----------|
| Monolito puro | Simple, pero no testeable sin infraestructura y no escala por dominio |
| Microservicios puros desde el día 1 | Sobre-ingeniería para 2 sprints — riesgo alto de no llegar a tiempo |
| **Híbrida (adoptada)** | Empieza dentro del monolito, migra a procesos separados después, sin reescribir el dominio |

> **Notas del orador:**
> La decisión clave a defender: la híbrida **no exige separar procesos para dar sus beneficios**. El refactor hexagonal de `game/` es ejecutable dentro del Sprint 1 sin tocar el despliegue (`Proyecto.md §10`: "esto NO cambia el despliegue, todo sigue en un solo proceso"). Se gana testeabilidad y modificabilidad de inmediato; la separación física en microservicios queda como evolución post-MVP con un roadmap concreto, no una promesa vaga.
>
> Detalle de la comparación: el monolito puro tiene cero latencia de red (~8ms/disparo) pero solo 30% de testeabilidad, y un servicio no crítico (Chat) puede arrastrar a uno crítico (Game) si comparten proceso. Los microservicios puros exigirían separar 7 procesos y resolver consistencia distribuida antes de tener el juego jugable — demasiado riesgo para el tiempo disponible.

### 4.3 Los tres paradigmas

**a) Hexagonal (Puertos y Adaptadores)**
- El dominio del juego no importa Redis, ni Socket.io, ni Express
- Puertos (interfaces) que los adaptadores implementan
- Se aplica en el **Game Service**

```javascript
// DOMINIO — función pura
function processShot(room, playerId, target, board) {
  if (room.turno.jugadorActual !== playerId) return { ok: false };
  return { ok: true, hit: board.shoot(target.x, target.y).hit };
}
```

**b) Event-Driven**
- Los servicios nunca se llaman directamente, publican y consumen eventos
- 17 eventos documentados, cada uno con publicador y consumidores explícitos
- Se aplica en toda la comunicación entre servicios

**c) Microservicios**
- Cada dominio (salas, juego, chat, bot, timer, auth, observabilidad) escala por separado
- Se aplica como meta de despliegue post-MVP

> **Notas del orador:**
> **Hexagonal:** `processShot` se prueba con un objeto `room` cualquiera, en milisegundos, sin Redis ni red real — eso es lo que mide `EC-05` y la táctica de Testeabilidad "Control & monitor state". Es la única forma de cumplir Modificabilidad (agregar un poder sin tocar infraestructura) y Testeabilidad a la vez.
>
> **Event-Driven:** ejemplo concreto — cuando Game Service publica `ShotFired`, lo consumen Observability y Bot Service sin que Game Service lo sepa. Si Bot Service se cae, Game Service sigue publicando con normalidad; la partida no se detiene. Regla explícita (`Proyecto.md §6.5`): "ningún servicio crítico depende de uno no crítico para funcionar". Caso ilustrativo corregido en esta sesión: el evento "crudo" `PlayerDisconnected` del Gateway (solo informa que el socket cayó) vs. el evento derivado `PlayerDisconnectedFromRoom` que publica Room Service tras decidir qué significa esa desconexión.
>
> **Microservicios:** no todos los servicios necesitan los mismos recursos — Game Service escala con partidas activas (250 partidas con 5 réplicas, `Proyecto.md §9.6`), Auth Service casi no necesita escalar. Separarlos evita escalar juntos cosas que no comparten necesidad.
>
> **Por qué los tres juntos y no por separado:** Event-Driven permite separar en Microservicios sin que un servicio conozca la dirección de red de otro. Hexagonal permite que, al separar el Game Service en su propio proceso, su dominio no se reescriba — solo se inyecta otro adaptador. Sin Event-Driven, Microservicios sería una maraña de HTTP síncrono frágil; sin Microservicios, Hexagonal seguiría dando testeabilidad pero no resolvería escalabilidad independiente. Conexión con la sección 3: Desempeño/Disponibilidad dependen del principio #2 (estado externo) y #4 (atomicidad); Seguridad depende del principio #5 (Gateway único); Escalabilidad depende directamente de Microservicios.

---

## 5. Justificación tecnológica

| Tecnología | Para qué | Por qué esta |
|------------|----------|--------------|
| **Node.js 20** | Runtime | I/O intensiva (WebSockets + Redis), equipo ya lo conoce |
| **Socket.io** | Tiempo real | Salas, reconexión automática, adapter de Redis incluido |
| **Redis (Upstash)** | Estado + locks + Pub/Sub | Única pieza que da atomicidad sub-ms entre instancias |
| **JWT + Google OAuth** | Identificación | Delega autenticación, sin manejar contraseñas |
| **Render** | Hosting backend | Plan gratuito real, soporta WebSockets nativos |
| **Upstash** | Redis administrado | Gratis para el MVP, HA disponible sin migrar de proveedor |
| **Vercel** | Hosting cliente | Gratis, despliegue automático |

> **Notas del orador:**
> Criterio transversal: *"¿esta tecnología resuelve un problema que ya tenemos, o uno que todavía no?"* Por eso RabbitMQ, Event Sourcing y HMAC en el broker quedan como evolución futura documentada, no como pila actual.
>
> Redis fue elegido sobre PostgreSQL/MongoDB porque ningún otro ofrece la misma atomicidad (`SETNX`, `INCRBY`) en sub-milisegundos, condición indispensable para la Salva Simultánea. Socket.io sobre WebSocket nativo porque da rooms, reconexión y el adapter de Redis ya construidos. Redis Pub/Sub (no RabbitMQ) porque ya se paga la misma conexión y es suficiente para el volumen del MVP; RabbitMQ añadiría 5-10ms y un servicio más que administrar, sin necesidad hasta superar ~10.000 eventos/segundo.
>
> Mencionar que se corrigió una decisión anterior: la primera versión de este documento usaba Azure y RabbitMQ como broker principal, rompiendo el presupuesto $0 del resto del proyecto. La versión actual usa solo Render + Upstash + Vercel. Decirlo demuestra revisión crítica, no una sola pasada.

---

## 6. Tensiones reconocidas

1. **Redis y Gateway son SPOF** — el cómputo se descentraliza en 7 servicios, pero el estado y el tráfico siguen centralizados.
   - Mitigación hoy: backups automáticos + reinicio automático
   - Mitigación futura: Redis Sentinel/Cluster + Gateway replicado
2. **El desempeño empeora** — ~8ms (monolito) → ~40ms (híbrida), por latencia de red entre servicios. Aceptado a cambio de Testeabilidad y Modificabilidad. Sigue muy por debajo del umbral de 200ms.

> **Notas del orador:**
> Presentar estas tensiones de forma proactiva es, en sí mismo, señal de madurez en arquitectura de software: ninguna decisión es gratuita, y reconocerlo con números (no con vaguedades del tipo "no tiene desventajas") es más convincente frente a un jurado.

---

## 7. Diagramas de apoyo

| Diagrama | Slide |
|----------|-------|
| `01-arquitectura-tres-paradigmas.puml` | Cómo se combinan los 3 paradigmas |
| `02-vision-general-arquitectura.puml` | Vista general: Cliente → Gateway → Servicios → Redis |
| `03-tacticas-por-atributo-calidad.puml` | Atributo de calidad → táctica concreta |
| `04-evolucion-monolito-a-microservicios.puml` | Antes / después / roadmap de 2 sprints |

> **Notas del orador:**
> Estos 4 son versiones simplificadas para proyectar. Si el jurado pide más detalle técnico, los 14 diagramas completos están en `docs/diagramas/` (componentes de Game Service y Room Service, secuencia de la Salva, secuencia de desconexión/reconexión, máquinas de estado, mapa de eventos completo).

---

## 8. Conclusiones

- La arquitectura híbrida permite ganar testeabilidad y modificabilidad **ya**, sin arriesgar el plazo de 2 sprints.
- Cada pieza de la propuesta está trazada hasta una historia de usuario o un KPI real — nada por "buena práctica" sin justificación específica.
- El reto de concurrencia central (Salva Simultánea) **no depende** de que la migración a microservicios se complete.
- Las debilidades (SPOF, latencia) se reconocen con números, junto con su mitigación.
