# BattleCaos-Ship — Inception Document

---

## Índice

- [Resumen del Proyecto](#resumen-del-proyecto)
- [Business Case](#business-case)
- [Priorización MoSCoW](#priorización-moscow)
- [Actores](#actores)
- [Epics](#epics)
- [Tabla de Historias de Usuario](#tabla-de-historias-de-usuario)
- [Cómo se aplicó INVEST en este replanteamiento](#cómo-se-aplicó-invest-en-este-replanteamiento)
- [Sprint 1 — Cimientos, Conexión y Core](#sprint-1--cimientos-conexión-y-core)
  - [DOMH01](#domh01) · [DOMH02](#domh02) · [DOMH03](#domh03) · [DOMH04](#domh04) · [DOMH05](#domh05) · [DOMH06](#domh06) · [DOMH07](#domh07) · [DOMH08](#domh08) · [DOMH09](#domh09)
  - [UIUXH01](#uiuxh01) · [UIUXH02](#uiuxh02) · [UIUXH03](#uiuxh03) · [UIUXH04](#uiuxh04) · [UIUXH05](#uiuxh05) · [UIUXH06](#uiuxh06) · [UIUXH07](#uiuxh07) · [UIUXH08](#uiuxh08)
  - [DOCH01](#doch01) · [DOCH02](#doch02) · [DOCH03](#doch03)
- [Sprint 2 — Mecánicas, Observabilidad y Cierre](#sprint-2--mecánicas-observabilidad-y-cierre)
  - [DOMH10](#domh10) · [DOMH11](#domh11) · [DOMH12](#domh12) · [DOMH13](#domh13) · [DOMH14](#domh14) · [DOMH15](#domh15)
  - [UIUXH09](#uiuxh09) · [UIUXH10](#uiuxh10) · [UIUXH11](#uiuxh11) · [UIUXH12](#uiuxh12) · [UIUXH13](#uiuxh13)
  - [DOCH04](#doch04) · [DOCH05](#doch05)
- [Principios INVEST](#principios-invest)
- [Definition of Ready (DoR)](#definition-of-ready-dor)
- [Definition of Done (DoD)](#definition-of-done-dod)
- [Historias en Tiempo Real](#historias-en-tiempo-real)
- [Observabilidad](#observabilidad)
- [Story Map (Sugerido)](#story-map-sugerido)
- [Reto de Concurrencia y Tiempo Real](#reto-de-concurrencia-y-tiempo-real)
- [Escenarios de Calidad vinculados a Historias de Usuario](#escenarios-de-calidad-vinculados-a-historias-de-usuario)
- [Escalabilidad Horizontal](#escalabilidad-horizontal)
- [Criterios de Evaluación](#criterios-de-evaluación)
- [Referencias](#referencias)

---

<a id="resumen-del-proyecto"></a>
## Resumen del Proyecto

**BattleCaos-Ship** es un juego de batalla naval multijugador en tiempo real con soporte para modos 1v1, 1v1 vs Bot y 2v2. El sistema emplea una arquitectura autoritativa: el backend (sin importar en cuántos servicios esté dividido) es la única fuente de verdad del estado de la partida, garantizando consistencia en escenarios de alta concurrencia (Salva Simultánea, Contramedidas, energía atómica). Los clientes solo reciben y muestran el estado autoritativo, nunca lo calculan por su cuenta. La arquitectura concreta que materializa este principio está definida en `Proyecto.md`.

### Integrantes

- *Por definir*

---

<a id="business-case"></a>
## Business Case

BattleCaos-Ship busca ofrecer una experiencia competitiva y estratégica de batalla naval en tiempo real, diferenciándose de los juegos tradicionales por turnos mediante mecánicas innovadoras:

- **Salva Simultánea (concurrencia):** Todos los jugadores disparan a la vez durante ventanas de 8 segundos, desafiando la velocidad, la estrategia y la infraestructura técnica.
- **Sistema de poderes y contramedidas:** Los jugadores acumulan energía para activar bombas de área, sonares, escudos y tormentas, con una ventana de reacción de 5 segundos para anular poderes enemigos.
- **Modo 2v2 colaborativo:** Colocación de flota en equipo y chat integrado para coordinar estrategias en tiempo real.
- **Arquitectura autoritativa:** El sistema es la fuente única de verdad, eliminando trampas y garantizando equidad competitiva.

### KPIs de Negocio

1. **% de partidas completadas** — ratio de salas que llegan a FIN sobre salas iniciadas.
2. **Latencia P95 disparo → snapshot** — experiencia de juego en tiempo real.
3. **% de reconexiones exitosas** — robustez ante caídas de red.
4. **Salas concurrentes sostenidas** — escalabilidad del sistema.

---

<a id="priorización-moscow"></a>
## Priorización MoSCoW

| Prioridad | Descripción |
|-----------|-------------|
| **Must** | Funcionalidades críticas para el MVP: autenticación, salas, sincronización RT, colocación, turnos, energía, poderes ofensivos, contramedida, salva, observabilidad básica, UI esencial, documentación arquitectónica. |
| **Should** | Poderes defensivos (escudo, tormenta), chat de equipo, timeouts de turno, healthcheck, métricas técnicas. |
| **Could** | Bot básico, dashboard de KPIs, métricas de negocio avanzadas. |
| **Won't** | Bot avanzado, dashboard completo con gráficas (Fase 2). |

---

<a id="actores"></a>
## Actores

Antes de listar las historias se definen los actores reales del sistema. **"Visitante", "jugador", "compañero de equipo" y "equipo" no son actores distintos**: son el mismo actor humano (alguien que juega la partida) en distintos momentos de su sesión (antes de identificarse, ya identificado, jugando en pareja). Por eso se consolidan en un único actor, **Usuario**. Los actores que sí son distintos —porque tienen un objetivo y una relación con el sistema diferentes— son el desarrollador, el operador y el evaluador.

| Actor | Quién es | Qué busca del sistema | Historias donde aparece |
|-------|----------|------------------------|--------------------------|
| **Usuario** | Persona que juega BattleCaos-Ship, sea antes de identificarse, ya identificada, en solitario o como parte de un equipo. | Jugar partidas justas, fluidas y sin perder progreso por fallas de conexión. | DOMH02, DOMH03, DOMH04, DOMH05, DOMH06, DOMH07, DOMH08, DOMH09, DOMH10, DOMH11, DOMH12, DOMH13, DOMH15, UIUXH01–UIUXH11, UIUXH13 |
| **Desarrollador** | Persona del equipo que construye y mantiene el sistema. | Una base técnica confiable y fácil de extender (ej. agregar un poder nuevo). | DOMH01 |
| **Operador** | Persona responsable de que el sistema funcione correctamente en producción durante la operación o la demo. | Visibilidad del comportamiento del sistema para detectar problemas a tiempo. | DOMH14, UIUXH12 |
| **Evaluador** | Persona externa (docente/jurado) que revisa el proyecto académicamente. | Documentación suficiente para entender la arquitectura, el dominio y el reto de concurrencia. | DOCH01, DOCH02, DOCH03, DOCH04, DOCH05 |

---

<a id="epics"></a>
## Epics

### DOM-0: DOMAIN
Agrupa las historias de usuario y tareas del área de dominio del MVP (2 sprints).

### UIUX-1: UI & UX
Agrupa las historias de usuario y tareas del área de interfaz y experiencia de usuario del MVP (2 sprints).

### DOC-2: DOCUMENTATION
Agrupa las historias de usuario y tareas del área de documentación del MVP (2 sprints).

---

<a id="tabla-de-historias-de-usuario"></a>
## Tabla de Historias de Usuario

| ID | Historia | Actor | Épica | Sprint | Prioridad |
|----|----------|-------|-------|--------|-----------|
| [DOMH01](#domh01) | Estado centralizado como fuente única de verdad | Desarrollador | DOM-0 | 1 | Must |
| [DOMH02](#domh02) | Identificación segura del usuario | Usuario | DOM-0 | 1 | Must |
| [DOMH03](#domh03) | Creación y unión a salas mediante código | Usuario | DOM-0 | 1 | Must |
| [DOMH04](#domh04) | Estado de partida siempre actualizado | Usuario | DOM-0 | 1 | Must |
| [DOMH05](#domh05) | Recuperación de partida tras una desconexión | Usuario | DOM-0 | 1 | Must |
| [DOMH06](#domh06) | Colocación de la flota propia | Usuario | DOM-0 | 1 | Must |
| [DOMH07](#domh07) | Colocación coordinada en equipo | Usuario | DOM-0 | 1 | Must |
| [DOMH08](#domh08) | Disparos por turnos con resultado visible | Usuario | DOM-0 | 1 | Must |
| [DOMH09](#domh09) | Acumulación de energía de equipo | Usuario | DOM-0 | 1 | Must |
| [UIUXH01](#uiuxh01) | Identidad visual coherente | Usuario | UIUX-1 | 1 | Must |
| [UIUXH02](#uiuxh02) | Inicio de sesión sin fricción | Usuario | UIUX-1 | 1 | Must |
| [UIUXH03](#uiuxh03) | Lobby para crear o unirse a salas | Usuario | UIUX-1 | 1 | Must |
| [UIUXH04](#uiuxh04) | Tablero siempre fiel al estado real | Usuario | UIUX-1 | 1 | Must |
| [UIUXH05](#uiuxh05) | Chat de equipo en tiempo real | Usuario | UIUX-1 | 1 | Should |
| [UIUXH06](#uiuxh06) | Colocación de flota cómoda e intuitiva | Usuario | UIUX-1 | 1 | Must |
| [UIUXH07](#uiuxh07) | Turno y resultado de disparos visibles | Usuario | UIUX-1 | 1 | Must |
| [UIUXH08](#uiuxh08) | Energía de equipo visible | Usuario | UIUX-1 | 1 | Must |
| [DOCH01](#doch01) | Arquitectura documentada | Evaluador | DOC-2 | 1 | Must |
| [DOCH02](#doch02) | Acceso y comunicación documentados | Evaluador | DOC-2 | 1 | Must |
| [DOCH03](#doch03) | Motor del juego documentado | Evaluador | DOC-2 | 1 | Must |
| [DOMH10](#domh10) | Uso de poderes mediante energía acumulada | Usuario | DOM-0 | 2 | Must |
| [DOMH11](#domh11) | Ventana de reacción para anular un poder | Usuario | DOM-0 | 2 | Must |
| [DOMH12](#domh12) | Ronda de disparo simultáneo | Usuario | DOM-0 | 2 | Must |
| [DOMH13](#domh13) | Oponente automático para jugar en solitario | Usuario | DOM-0 | 2 | Could |
| [DOMH14](#domh14) | Visibilidad del comportamiento del sistema | Operador | DOM-0 | 2 | Must |
| [DOMH15](#domh15) | Continuidad ante inactividad o desconexión | Usuario | DOM-0 | 2 | Should |
| [UIUXH09](#uiuxh09) | Panel de poderes claro | Usuario | UIUX-1 | 2 | Must |
| [UIUXH10](#uiuxh10) | Aviso visible de ventana de reacción | Usuario | UIUX-1 | 2 | Must |
| [UIUXH11](#uiuxh11) | Interfaz clara durante la ronda simultánea | Usuario | UIUX-1 | 2 | Must |
| [UIUXH12](#uiuxh12) | Panel de indicadores para el operador | Operador | UIUX-1 | 2 | Could |
| [UIUXH13](#uiuxh13) | Cierre de partida claro y manejo de errores | Usuario | UIUX-1 | 2 | Must |
| [DOCH04](#doch04) | Concurrencia documentada | Evaluador | DOC-2 | 2 | Must |
| [DOCH05](#doch05) | Documentación de cierre y demo | Evaluador | DOC-2 | 2 | Must |

---

<a id="cómo-se-aplicó-invest-en-este-replanteamiento"></a>
## Cómo se aplicó INVEST en este replanteamiento

Cada historia fue redactada para sostenerse por sí sola: su criterio de aceptación parte siempre de un **estado del sistema** (ej. "dado una sala en fase de colocación"), nunca de "una vez que la historia X esté terminada". Esto permite que cualquier historia se desarrolle, estime y pruebe usando datos de prueba que simulan ese estado, sin esperar a que otra historia exista en producción. Además, ninguna historia ni tarea menciona una tecnología de implementación específica (sin nombrar protocolos, librerías o frameworks): describen **qué** debe lograrse, no **cómo** se construye internamente — esa decisión se documenta aparte, en la arquitectura técnica.

La consolidación de actores (ver [Actores](#actores)) no introduce ninguna dependencia nueva entre historias: "Usuario" sigue siendo, en cada historia, quien dispara el estímulo descrito en su Gherkin — el criterio de aceptación nunca exige que otra historia de Usuario ya esté implementada, solo que el sistema se encuentre en un estado determinado.

---

<a id="sprint-1--cimientos-conexión-y-core"></a>
## Sprint 1 — Cimientos, Conexión y Core

### Épica: DOM-0 — DOMAIN

<a id="domh01"></a>
#### Historia: DOMH01 — Estado centralizado como fuente única de verdad
> Como desarrollador, quiero que el sistema mantenga un estado centralizado y autoritativo de cada partida —sin importar cuántos procesos o servicios la atiendan—, para garantizar que ningún cliente pueda alterar el resultado del juego.

| Atributo | Valor |
|----------|-------|
| ID | DOMH01 |
| Actor | Desarrollador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~10 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF001, DOMF002, DOMF003

**Criterios de aceptación (Gherkin):**
```gherkin
Given un entorno de despliegue configurado, con uno o varios procesos atendiendo a los jugadores
When el sistema se inicia
Then cada proceso debe quedar disponible y aceptando conexiones de jugadores
And todos deben leer y escribir el estado de cualquier partida en un almacenamiento compartido y persistente, de modo que da igual cuál de ellos atienda a un jugador
And cada proceso debe exponer una forma de verificar que está operativo
```

---

#### Tarea: DOMF001 — Inicialización de cada servicio y conexión al almacenamiento de estado compartido
> Configurar el punto de entrada de cada servicio, la carga de su configuración y la conexión al almacenamiento compartido donde vivirá el estado de las partidas (el mismo para todos los servicios, no uno por proceso). Exponer una verificación de disponibilidad por servicio.

| Atributo | Valor |
|----------|-------|
| ID | DOMF001 |
| Tipo | Tarea Técnica |
| Prioridad | Must |
| Sprint | Sprint 1 |

#### Tarea: DOMF002 — Enrutamiento de eventos del Gateway y control de tráfico
> Implementar la tabla de enrutamiento que mapea cada evento Socket.io entrante al canal del bus de eventos correspondiente: `room:create` y `room:join` → Room Service; `disparo:realizar`, `salva:disparo`, `poder:usar`, `colocacion:set`, `contramedida:activar` → Game Service; `chat:mensaje` → Chat Service. Suscribirse a los canales de resultado de cada servicio y reenviar sus publicaciones como broadcasts a los clientes de la sala correcta. Aplicar rate limiting por IP/token para prevenir abuso.

| Atributo | Valor |
|----------|-------|
| ID | DOMF002 |
| Tipo | Tarea Técnica |
| Prioridad | Must |
| Sprint | Sprint 1 |

---

#### Tarea: DOMF003 — Suite de tests unitarios del dominio del Game Service
> Crear la infraestructura mínima de pruebas del proyecto y escribir la suite de tests unitarios para las funciones puras del dominio del Game Service. Cubrir: `processShot` (impacto, fallo, celda repetida), `validatePower` + `applyPower` (los 5 poderes), `addEnergy` (por impacto y por hundimiento), `resolveSalvo` (ventana simultánea, bloqueo SETNX), `activateCountermeasure` (ventana de 5 s), y `evaluateVictory`. Cada test debe ejecutarse sin dependencias de Redis ni Socket.io. Objetivo: cobertura ≥ 85 % en la capa `domain/` del Game Service, satisfaciendo la meta de Testeabilidad de 30 % → 95 % declarada en el atributo de calidad EC-06 del Proyecto.md.

| Atributo | Valor |
|----------|-------|
| ID | DOMF003 |
| Tipo | Tarea Técnica |
| Prioridad | Should |
| Sprint | Sprint 2 |

---

<a id="domh02"></a>
#### Historia: DOMH02 — Identificación segura del usuario
> Como usuario, quiero identificarme con mi cuenta antes de jugar, para acceder de forma segura sin crear ni recordar una contraseña nueva.

| Atributo | Valor |
|----------|-------|
| ID | DOMH02 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~12 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF101, DOMF102

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario en la pantalla de inicio, sin identificarse aún
When elige identificarse con su cuenta
Then se le redirige al proveedor de identidad correspondiente
When la identificación es exitosa
Then recibe una credencial de sesión válida
And es dirigido al lobby del juego
```

---

#### Tarea: DOMF101 — Identificación delegada y emisión de credencial de sesión
> Redirigir al usuario a un proveedor de identidad externo, recibir la confirmación de que se autenticó correctamente y emitir, a cambio, una credencial de sesión propia del sistema con una vigencia definida.

#### Tarea: DOMF102 — Verificación de la credencial en cada conexión
> Al establecerse cualquier conexión de un usuario, validar su credencial de sesión. Si es válida, asociar la conexión al usuario correspondiente; si no lo es, rechazar la conexión.

---

<a id="domh03"></a>
#### Historia: DOMH03 — Creación y unión a salas mediante código
> Como usuario, quiero crear una sala de juego o unirme a una existente mediante un código, para jugar en un espacio aislado de otras partidas.

| Atributo | Valor |
|----------|-------|
| ID | DOMH03 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~14 h |
| Sprint | Sprint 1 |
| Tiempo Real | ✅ Sí |

**Tarea asociada:** DOMF201, DOMF202, DOMF203

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario identificado en el lobby
When elige crear una sala para un modo de juego
Then se genera un código único para esa sala
And la sala queda en estado de espera de jugadores
And el usuario queda asignado a un equipo

Given un usuario identificado con un código de sala válido
When ingresa el código y confirma su entrada
Then se valida que la sala exista y tenga cupo disponible
And se le asigna a un equipo
And todos los participantes de la sala ven la composición actualizada
```

---

#### Tarea: DOMF201 — Creación y unión a salas con asignación de equipo
> Registrar una nueva sala con un identificador único de acceso y el modo elegido. Al unirse, validar existencia y cupo disponible. Asignar a cada usuario un equipo según el modo de juego. Notificar a todos los participantes la composición actual de la sala.

#### Tarea: DOMF202 — Aislamiento entre salas concurrentes
> Garantizar que el estado y los eventos de una sala nunca afecten a otra, incluso cuando existen múltiples salas activas al mismo tiempo. Verificar que el sistema se mantenga estable con al menos 5 salas simultáneas.

#### Tarea: DOMF203 — Publicación de eventos de ciclo de vida de sala
> Cuando la sala queda completa con todos los jugadores asignados, publicar el evento `RoomReady` con los datos de modo, equipos y jugadores, para que el Game Service inicie la fase de colocación y el Timer Service arranque el temporizador correspondiente. Al unirse cada jugador a la sala, publicar `PlayerRoomJoined` para que el Chat Service cargue el historial de chat acumulado para ese jugador.

| Atributo | Valor |
|----------|-------|
| ID | DOMF203 |
| Tipo | Tarea Técnica |
| Prioridad | Must |
| Sprint | Sprint 1 |

---

<a id="domh04"></a>
#### Historia: DOMH04 — Estado de partida siempre actualizado
> Como usuario, quiero que el estado de mi partida se actualice automáticamente cuando ocurre algo relevante, para ver siempre la información autoritativa sin tener que calcularla yo mismo.

| Atributo | Valor |
|----------|-------|
| ID | DOMH04 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 8 pts |
| Estimación | ~18 h |
| Sprint | Sprint 1 |
| Tiempo Real | ✅ Sí |

**Tarea asociada:** DOMF301, DOMF302

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario participando en una partida activa
When el sistema procesa cualquier cambio de estado (disparo, turno, fase)
Then se entrega a todos los participantes una representación completa y actualizada del estado
And ningún participante calcula ni modifica ese estado por su cuenta
```

---

#### Tarea: DOMF301 — Distribución del estado completo tras cada cambio
> Cada vez que cambie algo relevante del estado de una partida (disparo, energía, turno, fase), entregar a todos los participantes de esa partida una representación completa y actualizada de dicho estado.

#### Tarea: DOMF302 — Definición de fases válidas de una partida
> Definir las fases de una partida y las transiciones permitidas entre ellas. Rechazar cualquier acción del usuario que no corresponda a la fase actual.

---

<a id="domh05"></a>
#### Historia: DOMH05 — Recuperación de partida tras una desconexión
> Como usuario, quiero poder recuperar mi partida si pierdo la conexión, para no perder mi progreso por una interrupción de red.

| Atributo | Valor |
|----------|-------|
| ID | DOMH05 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~10 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF401, DOMF402, DOMF403

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario que pierde la conexión durante una partida en curso
Then la partida no lo da por retirado: su lugar, su equipo y su progreso se conservan tal como estaban
And el resto de participantes son notificados de que ese usuario quedó desconectado, sin que la sala se cierre por eso

Given un usuario marcado como desconectado de una partida que aún existe
When vuelve a conectarse identificándose con su credencial de sesión
Then el sistema valida su identidad y localiza la partida donde ese usuario figura como desconectado
And la nueva conexión se asocia a ese usuario, quedando otra vez como conectado
And recibe el estado actual completo de la partida, igual que si nunca se hubiera ido
```

**Nota de aclaración (para evitar ambigüedad con DOMH15):** esta historia cubre el *regreso* del usuario y, como condición indispensable para que ese regreso tenga sentido, también la regla de que **desconectarse no es lo mismo que abandonar** (DOMF402, abajo). Lo que decide *cuánto tiempo se le espera antes de darlo por retirado definitivamente* es responsabilidad de [DOMH15](#domh15) / DOMF1301 — eso sí es un refinamiento posterior (Sprint 2), no un bloqueante para que la reconexión básica funcione.

---

#### Tarea: DOMF401 — Recuperación de estado al reconectar
> Al reconectarse un usuario, validar su credencial de sesión y localizar, entre las partidas activas, aquella donde ese usuario está registrado como desconectado. Si existe, asociar la nueva conexión a ese usuario, marcarlo nuevamente como conectado y entregarle el estado actual completo de esa partida. Si el usuario ya no figura en ninguna partida (porque se fue explícitamente o porque venció el tiempo de espera de DOMF1301), informarle que no hay partida para recuperar.

#### Tarea: DOMF402 — El usuario desconectado conserva su lugar (no se expulsa)
> Cuando un usuario pierde la conexión, no se le retira de la lista de jugadores ni de su equipo: se le marca como desconectado, conservando su tablero, su energía y su turno tal como estaban. La partida (o sala) solo se cierra cuando ya no queda ningún usuario conectado en ella, o cuando se cumple la condición de abandono definitivo de DOMF1301. **Esta tarea es la que hoy falta corregir primero**: sin ella, no hay nada que DOMF401 pueda recuperar.

#### Tarea: DOMF403 — Publicación de eventos de desconexión y reconexión desde la sala
> Tras marcar a un usuario como desconectado (DOMF402), publicar el evento `PlayerDisconnectedFromRoom` para que el Game Service pause el turno si es su turno, el Timer Service inicie el tiempo de espera de DOMF1301, y el Chat Service y el Observability Service registren la novedad. Al reasociar la conexión y marcar al usuario como conectado de nuevo (DOMF401), publicar `PlayerReconnected` para que el Game Service reanude la partida y el Timer Service cancele el tiempo de espera. Cuando todos los jugadores se hayan ido definitivamente, publicar `RoomDestroyed` para que Observability registre el cierre de sala.

| Atributo | Valor |
|----------|-------|
| ID | DOMF403 |
| Tipo | Tarea Técnica |
| Prioridad | Must |
| Sprint | Sprint 1 |

---

<a id="domh06"></a>
#### Historia: DOMH06 — Colocación de la flota propia
> Como usuario, quiero colocar mi flota de barcos antes de iniciar la partida, para preparar mi estrategia de defensa.

| Atributo | Valor |
|----------|-------|
| ID | DOMH06 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~12 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF501

**Criterios de aceptación (Gherkin):**
```gherkin
Given una sala en fase de colocación
When un usuario ubica un barco en su tablero
Then el sistema valida que no se solape con otro barco ni salga de los límites
When la validación es exitosa
Then la posición del barco queda registrada para ese usuario
And se confirma la colocación al usuario
```

---

#### Tarea: DOMF501 — Validación y registro de la colocación de flota
> Validar que cada barco colocado no se solape con otro ni exceda los límites del tablero. Rechazar colocaciones inválidas con el motivo correspondiente. Registrar la posición final de la flota de cada usuario y confirmar la colocación. Tras confirmar, publicar el evento `ShipsPlaced` en Redis Pub/Sub para que el Game Service lo propague al compañero de equipo en tiempo real (base de DOMF502).

---

<a id="domh07"></a>
#### Historia: DOMH07 — Colocación coordinada en equipo
> Como usuario, quiero ver en tiempo real cómo mi compañero de equipo coloca su flota, para coordinar nuestra estrategia conjunta antes de iniciar.

| Atributo | Valor |
|----------|-------|
| ID | DOMH07 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~10 h |
| Sprint | Sprint 1 |
| Tiempo Real | ✅ Sí |

**Tarea asociada:** DOMF502

**Criterios de aceptación (Gherkin):**
```gherkin
Given una sala en modo de equipo en fase de colocación, con un tiempo límite en curso
When un usuario coloca un barco
Then su compañero de equipo recibe la novedad en tiempo real
When ambos usuarios confirman su flota, o el tiempo límite expira
Then la sala avanza a la fase de turnos
```

---

#### Tarea: DOMF502 — Coordinación de colocación en equipo con límite de tiempo
> Iniciar un límite de tiempo para la fase de colocación. Notificar a los compañeros de equipo cada vez que se coloca un barco. Al expirar el límite o cuando ambos usuarios confirman, avanzar la fase de la sala.

---

<a id="domh08"></a>
#### Historia: DOMH08 — Disparos por turnos con resultado visible
> Como usuario, quiero disparar por turnos y conocer el resultado de cada disparo, para competir hasta hundir la flota rival.

| Atributo | Valor |
|----------|-------|
| ID | DOMH08 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 8 pts |
| Estimación | ~16 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF601, DOMF602

**Criterios de aceptación (Gherkin):**
```gherkin
Given una partida en fase de turnos
When es el turno del usuario actual y dispara a una coordenada del tablero rival
Then el sistema valida que la coordenada esté dentro de los límites y no haya sido disparada antes
And resuelve el resultado: agua, impacto o hundimiento
And entrega el resultado a todos los participantes
And el turno pasa al siguiente usuario
```

---

#### Tarea: DOMF601 — Validación y resolución de disparos por turno
> Mantener el orden rotativo de jugadores. Validar que el disparo corresponda al turno actual y a una coordenada válida no repetida. Resolver si el disparo es agua o impacto y comunicar el resultado.

#### Tarea: DOMF602 — Detección de hundimientos y condición de victoria
> Determinar cuándo todas las casillas de un barco han sido impactadas para declararlo hundido. Determinar cuándo toda la flota de un equipo está hundida para declarar ganador y cerrar la partida.

---

<a id="domh09"></a>
#### Historia: DOMH09 — Acumulación de energía de equipo
> Como usuario, quiero que mi equipo acumule energía cada vez que impacto o hundo un barco rival, para poder usar poderes especiales más adelante.

| Atributo | Valor |
|----------|-------|
| ID | DOMH09 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~6 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOMF701

**Criterios de aceptación (Gherkin):**
```gherkin
Given una partida en curso
When un usuario impacta un barco enemigo
Then la energía de su equipo aumenta en una cantidad fija
When un usuario hunde un barco enemigo
Then la energía de su equipo aumenta en una cantidad mayor
And la energía actualizada se refleja en el estado entregado a los jugadores
```

---

#### Tarea: DOMF701 — Acumulación de energía por equipo sin condiciones de carrera
> Llevar el conteo de energía de cada equipo. Incrementarla de forma consistente ante cada impacto y cada hundimiento, incluso si ocurren al mismo tiempo varios eventos de distintos usuarios del mismo equipo. Incluir el valor actualizado en el estado de la partida.

---

### Épica: UIUX-1 — UI & UX

<a id="uiuxh01"></a>
#### Historia: UIUXH01 — Identidad visual coherente
> Como usuario, quiero una aplicación con identidad visual coherente desde el primer momento, para tener una experiencia clara y reconocible.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH01 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~16 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF001, UIUXF002

---

#### Tarea: UIUXF001 — Pantallas principales de la aplicación cliente
> Definir y estructurar las pantallas principales del cliente (inicio de sesión, lobby, sala de espera, partida) y la navegación entre ellas. Dejar la aplicación disponible públicamente para los jugadores.

#### Tarea: UIUXF002 — Tablero visual parametrizable y lenguaje de colores
> Componente de tablero que se adapta al tamaño según el modo de juego, con una capa visual para representar los barcos sobre la cuadrícula. Definir una paleta de colores consistente para los distintos estados de una casilla (agua, impacto, hundido, protegida).

---

<a id="uiuxh02"></a>
#### Historia: UIUXH02 — Inicio de sesión sin fricción
> Como usuario, quiero iniciar sesión con un solo paso, para entrar al juego sin fricción.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH02 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 2 pts |
| Estimación | ~5 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF101

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario en la pantalla de inicio, sin identificarse aún
When ve la opción de identificarse
Then debe ver un botón claro para iniciar sesión
When hace clic en el botón
Then se muestra un indicador de "conectando"
When la identificación falla
Then se muestra un mensaje de error con opción de reintentar
```

---

#### Tarea: UIUXF101 — Pantalla de inicio de sesión con estados de carga y error
> Pantalla con un único botón para iniciar sesión. Estados visuales: en espera, conectando (indicador de carga) y error (mensaje con opción de reintento). Al completarse con éxito, conservar la credencial de sesión solo en memoria y continuar al lobby.

---

<a id="uiuxh03"></a>
#### Historia: UIUXH03 — Lobby para crear o unirse a salas
> Como usuario, quiero crear o unirme a una sala desde el lobby, para empezar a jugar.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH03 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF201

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario identificado en el lobby
When selecciona crear sala y elige un modo de juego
Then se muestra el código de la sala generada
When otro usuario ingresa ese código y confirma
Then ambos llegan a la sala
And ven la lista de jugadores conectados actualizándose en tiempo real
```

---

#### Tarea: UIUXF201 — Lobby con selector de modo, creación y unión a salas
> Pantalla de lobby con dos acciones: crear sala (con selector del modo de juego, mostrando el código generado) y unirse a sala (mediante un campo de código). Mostrar la lista de jugadores conectados a la sala, actualizada en tiempo real.

---

<a id="uiuxh04"></a>
#### Historia: UIUXH04 — Tablero siempre fiel al estado real
> Como usuario, quiero que mi tablero refleje exactamente lo que el sistema reporta, para que mi vista siempre coincida con el estado real de la partida.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH04 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~12 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF301

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario en una partida
When recibe una actualización del estado de la partida
Then el tablero se redibuja exactamente según esa información
And cada casilla muestra su estado correcto (agua, impacto, hundido, protegida)
And se muestran tanto el tablero propio como el del rival
And se muestra un indicador de calidad de conexión
```

---

#### Tarea: UIUXF301 — Render del tablero a partir del estado recibido
> El cliente solo refleja el estado que recibe del sistema; nunca lo calcula por su cuenta. Pintar cada casilla según ese estado. Mostrar ambos tableros (propio y rival). Mostrar un indicador visual de la calidad de la conexión.

---

<a id="uiuxh05"></a>
#### Historia: UIUXH05 — Chat de equipo en tiempo real
> Como usuario, quiero comunicarme con mi equipo en tiempo real, para coordinar la estrategia durante la partida.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH05 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Should |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF401, DOMF1103

---

#### Tarea: UIUXF401 — Chat de equipo y general en tiempo real
> Panel de mensajes con campo de entrada. En modos de equipo, separar el canal privado del equipo del canal general. Desplazamiento automático a los mensajes nuevos y visualización del nombre del emisor.

#### Tarea: DOMF1103 — Servicio de mensajería: recepción, persistencia y enrutamiento
> Recibir los eventos `chat:mensaje` que el Gateway reenvía desde el cliente. Validar que el emisor pertenezca a la sala. Persistir el mensaje en el historial de Redis de esa sala usando `LPUSH` + `LTRIM` para conservar únicamente los últimos 100 mensajes. En modo 2v2, enrutar el mensaje solo a los miembros del equipo del emisor si se trata de un canal privado, o a todos los participantes si es el canal general. Publicar el evento `ChatMessage` para que el Observability Service lo registre. Al recibir el evento `PlayerRoomJoined`, cargar el historial acumulado y entregárselo al nuevo jugador.

| Atributo | Valor |
|----------|-------|
| ID | DOMF1103 |
| Tipo | Tarea Técnica |
| Prioridad | Should |
| Sprint | Sprint 1 |

---

<a id="uiuxh06"></a>
#### Historia: UIUXH06 — Colocación de flota cómoda e intuitiva
> Como usuario, quiero colocar mi flota de forma cómoda e intuitiva, para preparar mi estrategia sin esfuerzo.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH06 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~14 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF501

---

#### Tarea: UIUXF501 — Colocación interactiva de barcos con vista previa
> Permitir mover y rotar cada barco sobre el tablero, mostrando una vista previa que indica si la posición es válida o inválida. Botón de confirmación de la flota. En modos de equipo, mostrar en tiempo real los barcos que coloca el compañero y el tiempo restante de la fase.

---

<a id="uiuxh07"></a>
#### Historia: UIUXH07 — Turno y resultado de disparos visibles
> Como usuario, quiero ver claramente de quién es el turno y el resultado de cada disparo, para seguir la partida sin confusión.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH07 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~10 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF601

---

#### Tarea: UIUXF601 — Indicador de turno y retroalimentación visual de disparos
> Mostrar de forma visible de quién es el turno actual. Permitir disparar haciendo clic en el tablero rival únicamente cuando es el turno propio. Retroalimentación visual diferenciada para agua, impacto y hundimiento.

---

<a id="uiuxh08"></a>
#### Historia: UIUXH08 — Energía de equipo visible
> Como usuario, quiero ver el nivel de energía acumulada de mi equipo, para decidir cuándo usar un poder.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH08 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 2 pts |
| Estimación | ~5 h |
| Sprint | Sprint 1 |

**Tarea asociada:** UIUXF701

---

#### Tarea: UIUXF701 — Barra de energía sincronizada con el estado de la partida
> Barra visual que refleja el nivel de energía del equipo según el estado recibido. Mostrar el valor numérico exacto y una animación al incrementarse. Indicar el costo de cada poder disponible.

---

### Épica: DOC-2 — DOCUMENTATION

<a id="doch01"></a>
#### Historia: DOCH01 — Arquitectura documentada
> Como evaluador, quiero la arquitectura del sistema documentada, para entender cómo está construido.

| Atributo | Valor |
|----------|-------|
| ID | DOCH01 |
| Actor | Evaluador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOCF001

---

#### Tarea: DOCF001 — Diagramas de despliegue y componentes
> Diagrama de despliegue mostrando el cliente, el punto único de entrada al sistema, los distintos servicios internos que componen el backend, y los servicios externos (identidad, almacenamiento de estado compartido). Diagrama de los componentes que conforman cada servicio. Tabla de tecnologías con su justificación.

---

<a id="doch02"></a>
#### Historia: DOCH02 — Acceso y comunicación documentados
> Como evaluador, quiero el flujo de acceso y comunicación documentado, para entender cómo opera el tiempo real.

| Atributo | Valor |
|----------|-------|
| ID | DOCH02 |
| Actor | Evaluador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 2 pts |
| Estimación | ~6 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOCF101

---

#### Tarea: DOCF101 — Secuencia de acceso y contrato de eventos
> Diagrama de secuencia del flujo completo de acceso (identificación, conexión, creación/unión a sala). Tabla con el contrato de todos los eventos en tiempo real: nombre, dirección, datos que transporta y propósito.

---

<a id="doch03"></a>
#### Historia: DOCH03 — Motor del juego documentado
> Como evaluador, quiero el motor del juego documentado, para entender la lógica del dominio.

| Atributo | Valor |
|----------|-------|
| ID | DOCH03 |
| Actor | Evaluador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 1 |

**Tarea asociada:** DOCF201

---

#### Tarea: DOCF201 — Diagrama de clases del motor y de fases
> Diagrama de clases con las entidades principales del dominio (sala, tablero, barco, jugador, equipo, poder, motor de juego). Diagrama de la máquina de fases de la partida.

---

<a id="sprint-2--mecánicas-observabilidad-y-cierre"></a>
## Sprint 2 — Mecánicas, Observabilidad y Cierre

### Épica: DOM-0 — DOMAIN

<a id="domh10"></a>
#### Historia: DOMH10 — Uso de poderes mediante energía acumulada
> Como usuario, quiero usar poderes especiales gastando la energía acumulada por mi equipo, para ganar ventaja estratégica.

| Atributo | Valor |
|----------|-------|
| ID | DOMH10 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 8 pts |
| Estimación | ~16 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOMF801, DOMF802

**Criterios de aceptación (Gherkin):**
```gherkin
Given un equipo con energía suficiente acumulada
When un usuario activa un poder disponible
Then se descuenta la energía correspondiente al equipo
And el efecto del poder se aplica según su tipo
When el equipo no tiene energía suficiente
Then el poder no puede activarse
```

---

#### Tarea: DOMF801 — Poderes ofensivos: bombardeo de área y detección
> Poder de bombardeo (impacta una zona de varias casillas contiguas). Poder de detección (revela si hay un barco en una línea completa del tablero rival). Validar energía suficiente antes de aplicar cualquiera de los dos.

#### Tarea: DOMF802 — Poderes defensivos: protección y bloqueo de turno
> Poder de protección (una casilla propia queda inmune a disparos por un número limitado de turnos). Poder de bloqueo (anula el siguiente turno del rival, uso único por partida). Validar energía suficiente antes de aplicar.

---

<a id="domh11"></a>
#### Historia: DOMH11 — Ventana de reacción para anular un poder
> Como usuario, quiero tener una breve ventana de tiempo para anular un poder activado en mi contra, para defenderme de forma reactiva.

| Atributo | Valor |
|----------|-------|
| ID | DOMH11 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~10 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOMF901

**Criterios de aceptación (Gherkin):**
```gherkin
Given un usuario objetivo de un poder ofensivo activado en su contra
Then se le abre una ventana de tiempo breve para reaccionar
When el usuario objetivo reacciona dentro de esa ventana
Then el poder queda anulado
And la energía gastada se devuelve a quien lo activó
When el usuario objetivo no reacciona a tiempo
Then la ventana se cierra y el poder se aplica con normalidad
```

---

#### Tarea: DOMF901 — Ventana de reacción, anulación y devolución de energía
> Al activarse un poder ofensivo, abrir una ventana de tiempo breve para que el objetivo pueda reaccionar. Si reacciona a tiempo, anular el efecto del poder y devolver la energía gastada a quien lo activó. Si la ventana se cierra sin reacción, aplicar el poder normalmente.

---

<a id="domh12"></a>
#### Historia: DOMH12 — Ronda de disparo simultáneo
> Como usuario, quiero vivir rondas donde todos disparamos al mismo tiempo, para enfrentar un ritmo de juego más intenso y competitivo.

| Atributo | Valor |
|----------|-------|
| ID | DOMH12 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 8 pts |
| Estimación | ~20 h |
| Sprint | Sprint 2 |
| Tiempo Real | ✅ Sí |

**Tarea asociada:** DOMF1001, DOMF1002

**Criterios de aceptación (Gherkin):**
```gherkin
Given una partida que alcanza la condición para iniciar una ronda simultánea
Then los turnos individuales se pausan y se abre una ventana de tiempo compartida para disparar

Given la ventana de disparo simultáneo activa
When dos usuarios disparan a la misma casilla casi al mismo instante
Then el sistema resuelve los disparos en orden de llegada
And solo el primero en llegar obtiene el resultado del impacto
And el segundo se rechaza por tratarse de una casilla ya resuelta

When un usuario dispara con una frecuencia menor a la cadencia mínima permitida
Then ese disparo es rechazado
```

---

#### Tarea: DOMF1001 — Ventana de disparo simultáneo
> Tras alcanzar la condición de activación (ej. cierto número de rondas), pausar los turnos individuales y abrir una ventana de tiempo compartida durante la cual todos los jugadores pueden disparar. Notificar el avance del tiempo restante. Al expirar, cerrar la ventana y reanudar los turnos.

#### Tarea: DOMF1002 — Resolución exclusiva por casilla y control de cadencia
> **Núcleo de concurrencia.** Garantizar que, cuando varios disparos llegan casi al mismo tiempo a la misma casilla, solo el primero en llegar obtenga el resultado y los demás se rechacen como repetidos. Validar que cada usuario respete una cadencia mínima entre disparos consecutivos.

---

<a id="domh13"></a>
#### Historia: DOMH13 — Oponente automático para jugar en solitario
> Como usuario, quiero enfrentarme a un oponente automático cuando juego en solitario, para poder jugar sin necesidad de otro jugador humano.

| Atributo | Valor |
|----------|-------|
| ID | DOMH13 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Could |
| Dificultad | 5 pts |
| Estimación | ~4 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOMF1101, DOMF1102

---

#### Tarea: DOMF1101 — Oponente automático con disparo y remate básico
> El oponente automático dispara a una coordenada no resuelta. Si impacta, prioriza las casillas adyacentes en su siguiente disparo para intentar hundir el barco.

#### Tarea: DOMF1102 — Integración del oponente automático con el bus de eventos
> Suscribir el servicio del oponente automático al bus de eventos para recibir los eventos `ShotFired` y `PhaseChanged` publicados por el Game Service. Al detectar que es el turno del oponente en una partida de modo `1v1-bot`, invocar la lógica de decisión de DOMF1101 y publicar el evento `BotDecision` con la coordenada elegida, para que el Game Service lo procese como un disparo ordinario del bot.

| Atributo | Valor |
|----------|-------|
| ID | DOMF1102 |
| Tipo | Tarea Técnica |
| Prioridad | Could |
| Sprint | Sprint 2 |

---

<a id="domh14"></a>
#### Historia: DOMH14 — Visibilidad del comportamiento del sistema
> Como operador, quiero poder observar el comportamiento del sistema mientras opera, para detectar problemas y medir su desempeño.

| Atributo | Valor |
|----------|-------|
| ID | DOMH14 |
| Actor | Operador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 8 pts |
| Estimación | ~19 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOMF1201, DOMF1202, DOMF1203

**Criterios de aceptación (Gherkin):**
```gherkin
Given un evento clave del sistema (disparo, hundimiento, turno, fase)
Then se registra de forma estructurada con marca de tiempo, tipo de evento, sala, jugador y resultado

Given una partida en curso
Then el sistema lleva un conteo de métricas técnicas: disparos totales, impactos, hundimientos, disparos en ronda simultánea

Given el cierre de una partida
Then los indicadores de negocio se actualizan
```

---

#### Tarea: DOMF1201 — Registro estructurado de eventos clave
> En cada evento clave (disparo, hundimiento, energía, turno, fase, ronda simultánea), registrar de forma estructurada: marca de tiempo, tipo de evento, identificador de sala, identificador de jugador y resultado.

#### Tarea: DOMF1202 — Métricas técnicas de concurrencia y rendimiento
> Llevar un conteo por partida de disparos totales, impactos, hundimientos y disparos en ronda simultánea. Medir el tiempo transcurrido entre un disparo y la actualización de estado correspondiente. Verificar que el conteo visto por cada cliente coincida con el del sistema al cierre de cada ronda simultánea.

#### Tarea: DOMF1203 — Cálculo de los indicadores de negocio
> Calcular y almacenar periódicamente: porcentaje de partidas completadas, latencia P95 entre disparo y actualización de estado, porcentaje de reconexiones exitosas y número de salas concurrentes sostenidas.

---

<a id="domh15"></a>
#### Historia: DOMH15 — Continuidad ante inactividad o desconexión de un jugador
> Como usuario, quiero que la partida continúe de forma estable aunque un participante se quede inactivo o se desconecte, para tener una experiencia confiable.

| Atributo | Valor |
|----------|-------|
| ID | DOMH15 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Should |
| Dificultad | 5 pts |
| Estimación | ~11 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOMF1301, DOMF1302, DOMF1303

**Nota de aclaración:** la regla básica de "desconectarse no es abandonar" vive en DOMF402 ([DOMH05](#domh05)), porque sin ella la reconexión no tendría nada que recuperar. Lo que aporta esta historia es el refinamiento: cuánto se espera antes de dar a alguien por retirado, y qué pasa con su turno mientras tanto.

---

#### Tarea: DOMF1301 — Límite de tiempo de turno y abandono definitivo por desconexión prolongada
> Cuando el Game Service recibe el evento `PlayerDisconnectedFromRoom` (DOMF403) durante el turno de un jugador, pausar ese turno inmediatamente —sin pasarlo al siguiente— y esperar a que el jugador reconecte o a que el tiempo de espera venza. Establecer además un límite de tiempo para que un usuario actúe en su turno en condiciones normales; si lo supera, pasar el turno automáticamente. Establecer el tiempo máximo de espera para un usuario marcado como desconectado (DOMF402); si se reconecta antes de que venza, continúa con normalidad; si no, se le da por retirado definitivamente y se libera su lugar en la partida.

#### Tarea: DOMF1302 — Verificación de disponibilidad previa a la operación
> Exponer una forma de verificar que el servicio está disponible y responde correctamente antes de iniciar la operación normal.

#### Tarea: DOMF1303 — Timer Service: elección de líder, heartbeat y gestión de temporizadores
> Implementar la elección de líder del Timer Service usando `SETNX` en Redis con TTL de 1 segundo: solo la instancia que adquiere el lease actúa como master; las demás esperan en standby. El master renueva el lease cada 500ms (heartbeat). Si el master falla y el lease expira, cualquier instancia standby puede tomar el liderazgo en aproximadamente 1.5 segundos (failover automático). Al convertirse en master, suscribirse a los eventos `RoomReady` y `PhaseChanged` para arrancar el temporizador de la fase correspondiente (COLOCACION: 60s, TURNO: 30s, SALVA: 8s, CONTRAMEDIDA: 5s). Publicar `TimerEnd` cuando el temporizador vence y `TimerTick` cada 100ms con el tiempo restante para que el Gateway lo distribuya a los clientes.

| Atributo | Valor |
|----------|-------|
| ID | DOMF1303 |
| Tipo | Tarea Técnica |
| Prioridad | Should |
| Sprint | Sprint 2 |

---

### Épica: UIUX-1 — UI & UX

<a id="uiuxh09"></a>
#### Historia: UIUXH09 — Panel de poderes claro
> Como usuario, quiero un panel que muestre claramente los poderes disponibles, su costo y su objetivo, para decidir cuándo y cómo usarlos.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH09 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~10 h |
| Sprint | Sprint 2 |

**Tarea asociada:** UIUXF801

---

#### Tarea: UIUXF801 — Panel de poderes con costo y selección de objetivo
> Listar los poderes disponibles junto a su costo de energía. Habilitar cada poder solo si el equipo tiene energía suficiente. Permitir seleccionar el objetivo del poder sobre el tablero. Mostrar tiempos de espera y usos restantes cuando aplique.

---

<a id="uiuxh10"></a>
#### Historia: UIUXH10 — Aviso visible de ventana de reacción
> Como usuario, quiero un aviso claro con el tiempo restante, para poder reaccionar a tiempo cuando un poder se activa en mi contra.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH10 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~6 h |
| Sprint | Sprint 2 |

**Tarea asociada:** UIUXF901

---

#### Tarea: UIUXF901 — Aviso de ventana de reacción con cuenta regresiva
> Al notificarse una ventana de reacción, mostrar un aviso destacado con cuenta regresiva y una acción para anular el poder. El tiempo mostrado siempre proviene del sistema, nunca se calcula en el cliente.

---

<a id="uiuxh11"></a>
#### Historia: UIUXH11 — Interfaz clara durante la ronda simultánea
> Como usuario, quiero una interfaz clara durante la ronda de disparo simultáneo, para disparar con control en medio del ritmo intenso.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH11 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 5 pts |
| Estimación | ~10 h |
| Sprint | Sprint 2 |

**Tarea asociada:** UIUXF1001

---

#### Tarea: UIUXF1001 — Interfaz de la ronda simultánea con cadencia y retroalimentación
> Mostrar la cuenta regresiva de la ventana de disparo simultáneo. Mantener el tablero rival activo para disparar. Mostrar visualmente el tiempo de espera entre disparos consecutivos y retroalimentación inmediata de cada resultado. En modos de equipo, distinguir las casillas marcadas por el compañero.

---

<a id="uiuxh12"></a>
#### Historia: UIUXH12 — Panel de indicadores para el operador
> Como operador, quiero un panel con los indicadores clave, para visualizar de un vistazo el estado y desempeño del sistema.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH12 |
| Actor | Operador |
| Tipo | Historia de Usuario |
| Prioridad | Could |
| Dificultad | 5 pts |
| Estimación | ~5 h |
| Sprint | Sprint 2 |

**Tarea asociada:** UIUXF1101

---

#### Tarea: UIUXF1101 — Panel mínimo de indicadores de negocio
> Panel con al menos dos indicadores visibles: comparación de disparos contados por el sistema frente a los contados por los clientes (deben coincidir), y la latencia P95 medida. Actualización periódica.

---

<a id="uiuxh13"></a>
#### Historia: UIUXH13 — Cierre de partida claro y manejo de errores
> Como usuario, quiero un cierre de partida claro y un manejo visible de errores, para tener una buena experiencia incluso cuando algo falla.

| Atributo | Valor |
|----------|-------|
| ID | UIUXH13 |
| Actor | Usuario |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~10 h |
| Sprint | Sprint 2 |

**Tarea asociada:** UIUXF1201

---

#### Tarea: UIUXF1201 — Pantalla de victoria y manejo visible de errores
> Pantalla de cierre que anuncia al equipo ganador con una opción para volver al lobby. Aviso visible ante errores del sistema. Indicación clara cuando el cliente intenta reconectarse automáticamente.

---

### Épica: DOC-2 — DOCUMENTATION

<a id="doch04"></a>
#### Historia: DOCH04 — Concurrencia documentada
> Como evaluador, quiero el manejo de concurrencia documentado, para evidenciar cómo se resuelve el principal reto técnico del proyecto.

| Atributo | Valor |
|----------|-------|
| ID | DOCH04 |
| Actor | Evaluador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOCF301

---

#### Tarea: DOCF301 — Diagramas de secuencia de la ronda simultánea y la ventana de reacción
> Diagrama de secuencia de la ronda de disparo simultáneo, mostrando cómo se resuelve la exclusión entre disparos concurrentes a la misma casilla. Diagrama de secuencia de la ventana de reacción, mostrando su apertura, posible anulación y cierre. **Diagrama clave para evidenciar el reto de concurrencia del curso.**

---

<a id="doch05"></a>
#### Historia: DOCH05 — Documentación de cierre y demo
> Como evaluador, quiero la documentación de cierre del proyecto, para poder reproducir la demo y conocer su evolución prevista.

| Atributo | Valor |
|----------|-------|
| ID | DOCH05 |
| Actor | Evaluador |
| Tipo | Historia de Usuario |
| Prioridad | Must |
| Dificultad | 3 pts |
| Estimación | ~8 h |
| Sprint | Sprint 2 |

**Tarea asociada:** DOCF401

---

#### Tarea: DOCF401 — Manual de demo y visión de evolución futura
> Manual paso a paso para reproducir la demostración. Documentación de la visión de evolución del sistema más allá del MVP: cómo el sistema actual migra, servicio por servicio, hacia una arquitectura de servicios independientes comunicados por eventos, sin romper lo que ya funciona en cada paso (ver el roadmap de migración de `Proyecto.md`).

---

<a id="principios-invest"></a>
## Principios INVEST

Cada historia de usuario en este documento cumple con los principios INVEST:

| Principio | Aplicación |
|-----------|------------|
| **I — Independent** | Cada criterio de aceptación parte de un estado del sistema (ej. "dado una partida en fase de turnos"), no de que otra historia ya esté implementada. Esto permite desarrollar, estimar y probar cada historia con datos de prueba, sin esperar a que otra exista en producción. La consolidación de actores no cambia esto: "Usuario" es quien dispara el estímulo, nunca una historia previa. |
| **N — Negotiable** | El alcance y los detalles se refinan durante el grooming (ej. el tamaño del tablero o la duración exacta de una ventana de tiempo son negociables). |
| **V — Valuable** | Toda historia entrega valor directo a su actor o al negocio (ej. DOMH08 — poder disparar y conocer el resultado). |
| **E — Estimable** | Cada historia tiene una estimación en puntos Fibonacci (2, 3, 5, 8) basada en su complejidad. |
| **S — Small** | Las historias se descomponen en tareas técnicas acotadas (ej. DOMH04 se divide en DOMF301 + DOMF302). |
| **T — Testable** | Cada historia incluye criterios de aceptación en formato Gherkin Given-When-Then, verificables mediante pruebas automatizadas o manuales. |

---

<a id="definition-of-ready-dor"></a>
## Definition of Ready (DoR)

Cada historia de usuario debe cumplir los siguientes criterios antes de pasar a desarrollo:

1. **Historias de Usuario claras y completas:** Redactadas con formato "Como [actor], quiero [acción], para [beneficio]".
2. **Actor identificado correctamente:** Asignada a uno de los 4 actores reales del sistema (Usuario, Desarrollador, Operador, Evaluador), no a un rol contextual transitorio.
3. **Criterios de aceptación definidos:** En formato Gherkin (Given-When-Then), expresados sobre un estado del sistema, no sobre otra historia.
4. **Sin tecnología de implementación impuesta:** La historia describe qué debe lograrse, no cómo se construye internamente.
5. **Factible técnicamente:** Validada con el equipo de desarrollo.
6. **Estimada:** Asignada con puntaje de historia (Fibonacci).
7. **Priorizada:** Clasificada con MoSCoW (Must, Should, Could).
8. **Vinculada a una Épica y a sus tareas técnicas:** Trazabilidad completa.

---

<a id="definition-of-done-dod"></a>
## Definition of Done (DoD)

Una historia de usuario se considera completa cuando cumple:

1. **Código implementado y subido** al repositorio en la rama correspondiente.
2. **Pruebas unitarias y de integración** superadas.
3. **Criterios de aceptación (Gherkin)** verificados.
4. **Code review** aprobado por al menos un compañero.
5. **Sin bugs conocidos** en la funcionalidad implementada.
6. **Integración continua** pasando todos los checks.
7. **Documentación actualizada** (wiki o README).
8. **Desplegada en entorno de pruebas** y validada.
9. **Métrica de observabilidad** registrada (logs, KPIs).

---

<a id="historias-en-tiempo-real"></a>
## Historias en Tiempo Real

Se identifican las historias principales con interacciones explícitas en tiempo real:

| Historia | Descripción | Mecanismo (conceptual) |
|----------|-------------|-----------|
| [DOMH03](#domh03) — Creación y unión a salas | Los jugadores ven en tiempo real quién se conecta a la sala. | Notificación a los participantes de la sala ante cada cambio |
| [DOMH04](#domh04) — Estado de partida siempre actualizado | El estado autoritativo se refleja instantáneamente en todos los clientes. | Distribución del estado completo a todos los participantes |
| [DOMH12](#domh12) — Ronda de disparo simultáneo | Todos los jugadores disparan a la vez en una ventana de tiempo, resuelta de forma exclusiva por casilla. | Resolución exclusiva por casilla + control de cadencia |
| [DOMH11](#domh11) — Ventana de reacción para anular un poder | Ventana de tiempo breve gestionada por el sistema, con cuenta regresiva visible. | Temporizador gestionado por el sistema, nunca por el cliente |

---

<a id="observabilidad"></a>
## Observabilidad

### Registro estructurado de eventos (DOMF1201)
- Cada evento clave se registra de forma estructurada con marca de tiempo, tipo de evento, sala, jugador y resultado.
- Disponible para trazabilidad durante la operación.

### Métricas Técnicas (DOMF1202)
- Conteo por partida: disparos totales, impactos, hundimientos, disparos en ronda simultánea.
- Tiempo entre disparo y actualización de estado.
- Comparación sistema vs. cliente (validación de sincronismo).

### KPIs de Negocio (DOMF1203)
1. **% Partidas completadas:** salas finalizadas / salas iniciadas.
2. **Latencia P95 disparo → actualización de estado.**
3. **% Reconexiones exitosas.**
4. **Salas concurrentes sostenidas.**

### Panel de indicadores (UIUXF1101)
- Panel mínimo durante la demo con los indicadores clave en pantalla.

---

<a id="story-map-sugerido"></a>
## Story Map (Sugerido)

Se recomienda crear un **Story Map** en Miro, FigJam o similar para visualizar:

1. **Eje horizontal (sprints):** Sprint 1 → Sprint 2.
2. **Eje vertical (prioridad):** Must → Should → Could.
3. **Agrupación por épicas:** DOM-0, UIUX-1, DOC-2.
4. **Relaciones funcionales entre tareas** (no dependencias de bloqueo, sino orden natural del flujo de juego).

Esto permite visualizar el flujo de valor, identificar cuellos de botella y comunicar el alcance al equipo.

---

<a id="reto-de-concurrencia-y-tiempo-real"></a>
## Reto de Concurrencia y Tiempo Real

El principal reto técnico del proyecto es la **ronda de disparo simultáneo (DOMF1002)**, que implementa:

1. **Resolución exclusiva por casilla**, garantizando que solo un disparo resuelva una casilla aunque lleguen varios casi al mismo tiempo.
2. **Control de cadencia** para evitar que gane el disparo más rápido por encima de la estrategia.
3. **Distribución del resultado** a todos los jugadores en tiempo real.

Adicionalmente, la **ventana de reacción (DOMF901)** introduce un control temporal gestionado por el sistema (no por el cliente), demostrando manejo de estado temporal concurrente.

**Documentación de concurrencia (DOCF301):** Diagramas de secuencia que evidencian estos mecanismos para la evaluación de ARSW.

---

<a id="escenarios-de-calidad-vinculados-a-historias-de-usuario"></a>
## Escenarios de Calidad vinculados a Historias de Usuario

No todas las historias requieren un escenario de calidad propio: solo se documentan los que aportan valor real de verificación sobre el reto técnico del proyecto (concurrencia, tiempo real, seguridad de acceso y evolución del sistema). Cada escenario sigue las 6 partes estándar (Fuente de Estímulo, Estímulo, Artefacto, Entorno, Respuesta, Medida de Respuesta) y se ancla a la historia de la que se deriva, de modo que sea trazable y verificable como caso de prueba.

---

### EC-01 — Desempeño en tiempo real (deriva de [DOMH04](#domh04))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario |
| Estímulo | Realiza una acción que cambia el estado de la partida (disparo, turno, fase) |
| Artefacto | Distribución del estado de la partida |
| Entorno | Operación normal, hasta 5 salas concurrentes |
| Respuesta | El sistema resuelve la acción y entrega el estado actualizado a todos los participantes de la sala |
| Medida de Respuesta | Latencia P95 menor a 200 ms entre la acción y la actualización recibida |

**Valor:** sin esto, el "estado siempre autoritativo" de DOMH04 es una promesa sin verificación; este escenario lo convierte en algo medible y ligado directamente al KPI de negocio de latencia.

---

### EC-02 — Concurrencia sin condiciones de carrera (deriva de [DOMH12](#domh12))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Dos o más usuarios |
| Estímulo | Disparan a la misma casilla casi en el mismo instante, durante la ventana de disparo simultáneo |
| Artefacto | Resolución exclusiva por casilla |
| Entorno | Ventana de disparo simultáneo activa, con disparos concurrentes de prueba |
| Respuesta | Solo el disparo que llega primero resuelve la casilla; los demás se rechazan como repetidos |
| Medida de Respuesta | 0% de casillas resueltas dos veces en pruebas con disparos concurrentes simulados |

**Valor:** es el reto de concurrencia central del curso (DOMF1002); sin esta verificación, "todos disparan a la vez" podría implicar resultados inconsistentes y la historia perdería su valor competitivo (equidad).

---

### EC-03 — Disponibilidad ante caída de red (deriva de [DOMH05](#domh05))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario |
| Estímulo | Pierde la conexión durante una partida activa y vuelve a conectarse |
| Artefacto | Recuperación de estado tras reconexión |
| Entorno | Operación normal, partida en curso |
| Respuesta | El sistema reasocia al usuario y le entrega el estado completo y actual de su partida |
| Medida de Respuesta | Más del 95% de reconexiones exitosas, recuperación en menos de 3 segundos |

**Valor:** liga directamente la historia a uno de los 4 KPIs de negocio del proyecto (% de reconexiones exitosas); sin el escenario, "recuperar la partida" no tendría un umbral verificable de éxito.

---

### EC-04 — Seguridad de acceso (deriva de [DOMH02](#domh02))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario sin credencial válida (sesión expirada, manipulada o inexistente) |
| Estímulo | Intenta establecer conexión o realizar una acción sobre una partida |
| Artefacto | Verificación de identidad en cada conexión |
| Entorno | Operación normal |
| Respuesta | El sistema rechaza la conexión o la acción y no expone ningún estado de partida |
| Medida de Respuesta | 0% de accesos no autorizados aceptados en pruebas con credenciales inválidas o expiradas |

**Valor:** el sistema autoritativo (pilar del business case) pierde sentido si un usuario no identificado puede leer o alterar una partida; este escenario es lo que sostiene esa garantía de equidad competitiva.

---

### EC-05 — Modificabilidad del catálogo de poderes (deriva de [DOMH10](#domh10))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Desarrollador |
| Estímulo | Agrega un nuevo poder al catálogo existente |
| Artefacto | Módulo de poderes del motor de juego |
| Entorno | Tiempo de diseño |
| Respuesta | El nuevo poder se integra sin modificar la validación de turnos, energía ni disparos |
| Medida de Respuesta | Máximo 2 archivos modificados o agregados; implementado y probado en menos de 3 horas |

**Valor:** el negocio depende de poder añadir mecánicas nuevas (poderes) sin arriesgar el resto del motor; sin este escenario, DOMH10 podría crecer como un módulo rígido y costoso de extender.

**Estado de cumplimiento:** esta medida (máx. 2 archivos, <3h) es la meta de la arquitectura propuesta en `Proyecto.md`, alcanzable separando el dominio puro de poderes (lógica) de su orquestación (lectura/escritura de estado) — hoy el módulo de poderes todavía mezcla ambas cosas en el mismo archivo, así que la medida aún no se cumple en el código actual. La separación que lo resuelve es deliberadamente simple (un archivo de dominio puro + un archivo que orquesta la lectura/escritura del estado, sin capas intermedias adicionales), por lo que se estima alcanzable en menos tiempo del que tomaría con un esquema de más capas — este escenario sirve como criterio de aceptación de ese trabajo, no como verificación del código actual.

---

### EC-06 — Escalabilidad por aislamiento de salas (deriva de [DOMH03](#domh03))

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuarios de distintas partidas |
| Estímulo | Crean y juegan simultáneamente en múltiples salas |
| Artefacto | Aislamiento entre salas concurrentes |
| Entorno | Carga sostenida de al menos 5 salas activas |
| Respuesta | Cada sala opera de forma aislada; ningún evento de una sala afecta a otra |
| Medida de Respuesta | 0 fugas de eventos entre salas; sin degradación perceptible de latencia con 5 o más salas concurrentes |

**Valor:** conecta directamente con el KPI de "salas concurrentes sostenidas" y es la base que permite que el sistema crezca sin rediseño.

---

<a id="escalabilidad-horizontal"></a>
## Escalabilidad Horizontal

La arquitectura oficial del proyecto es la híbrida (Domain Kernel + Event-Driven + Microservicios) descrita en `Proyecto.md`: el dominio del juego (las reglas puras) queda aislado de la infraestructura, los servicios se comunican por eventos, y cada dominio (salas, juego, chat, bot, timer, auth, observabilidad) escala de forma independiente. `docs/ESCALABILIDAD.md` y `docs/TIMER_MASTER.md` documentan el mecanismo de elección de líder (`Timer Master`) que esa arquitectura reutiliza dentro del Timer Service para coordinar los temporizadores entre réplicas, sin que ningún temporizador se duplique ni se pierda. Para el MVP de este documento, la migración completa se trata como trabajo de documentación y como un roadmap incremental (DOCF401, ver el roadmap de migración de `Proyecto.md`), no como una historia de desarrollo bloqueante.

---

<a id="criterios-de-evaluación"></a>
## Criterios de Evaluación

| Criterio | Descripción | Estado en este documento |
|----------|-------------|--------------------------|
| **Originalidad** | Propuesta innovadora con valor agregado frente a software existente. | BattleCaos-Ship introduce la ronda de disparo simultáneo, la ventana de reacción y el sistema de energía como diferenciadores clave. |
| **Completitud** | No se aceptan propuestas incompletas. Debe comprenderse el alcance total. | Se documentan 3 Épicas, 4 actores diferenciados, historias de usuario y tareas técnicas en 2 Sprints, cubriendo dominio, UI/UX y documentación. |
| **Claridad** | Redacción y ortografía correctas. | Todas las historias usan formato "Como [actor], quiero [acción], para [beneficio]". Criterios Gherkin redactados con Given-When-Then. |
| **Calidad de las HU** | Basada en INVEST, Gherkin y DoR. | Cada historia es independiente (parte de un estado, no de otra historia), negociable, valiosa, estimable, pequeña y testeable. Las 6 más críticas además tienen un escenario de calidad verificable (EC-01 a EC-06). |
| **Valor de negocio** | Business case correcto. | Se definen 4 KPIs de negocio alineados con el caso de uso: % partidas completadas, latencia P95, % reconexiones, salas concurrentes — cada uno verificado por un escenario de calidad (EC-01, EC-03, EC-06). |
| **Priorización MoSCoW** | Must, Should, Could, Won't. | Tabla MoSCoW incluida. Cada historia y tarea tiene su prioridad asignada. |
| **Desempeño en presentación final** | Habilidades blandas. | *(A evaluar en la presentación del 8 de julio)* |
| **Reto de concurrencia y real time** | Implementación de mecanismos concurrentes y tiempo real. | DOMF1002 (resolución exclusiva por casilla + cadencia) y DOMF901 (ventana de reacción) documentados con diagramas de secuencia en DOCF301, y verificados como escenario de calidad en EC-02. |

---

<a id="referencias"></a>
## Referencias

- [Principios INVEST](https://www.google.com)
- [Técnica MoSCoW](https://www.google.com)
- [Lenguaje Gherkin](https://www.google.com)
- [Definition of Ready vs Definition of Done](https://www.google.com)
- [Story Mapping](https://www.google.com)
- [Azure DevOps Wiki](https://www.google.com)
