# BattleCaos-ship

### Play with your crew • Battleship multijugador en tiempo real

## Business Case

```
Arquitecturas de Software (ARSW) — Grupo 2 — 2026 - 1
```
Autor Juan Carlos Bohórquez Monroy

Programa Ingeniería en Sistemas

Institución Escuela Colombiana de Ingeniería Julio Garavito

Asignatura Arquitectura de Software (ARSW)

Periodo 2026 - I

Fecha Junio de 2026


## 1. Resumen Ejecutivo

Lo que distingue al proyecto no es solo el formato por equipos, sino la capa de decisiones
estratégicas que añade: los jugadores acumulan energía disparando con precisión y la gastan en
cinco poderes especiales que pueden cambiar el rumbo de la partida. A esto se suma la Salva
Simultánea, una ventana de fuego libre en la que todos los jugadores disparan a la vez — fuera
del flujo de turnos — y que convierte el juego en un escenario de concurrencia real, no solo de
sincronización rápida.

Desde la perspectiva del curso, el proyecto demuestra en la práctica los conceptos centrales de
Arquitecturas de Software: un servidor autoritativo que gestiona estado compartido entre
múltiples clientes concurrentes mediante WebSockets, aislamiento de sesiones entre múltiples
salas corriendo en paralelo, arquitectura orientada a eventos y resolución de condiciones de
carrera con operaciones atómicas. Todo sobre un stack moderno, desplegable en infraestructura
gratuita y completamente demostrable en vivo.

## 2. Problema / Oportunidad

2.1 Contexto actual

Battleship existe en versión digital desde los años 90, pero ninguna iteración actual se adaptó a
cómo juegan las personas hoy. Las opciones disponibles comparten limitaciones estructurales
que generan una oportunidad real:

- Casi todas las versiones conocidas son duelos 1 vs 1. No existe una plataforma accesible
    que permita jugarlo por equipos de forma coordinada.
- Las versiones con mayor visibilidad requieren cuenta, instalación o pago, barreras que
    eliminan la espontaneidad del juego entre grupos.
- Ninguna versión digital incorpora mecánicas de interacción entre jugadores: no hay
    comunicación, no hay poderes, no hay decisiones tácticas más allá del disparo básico.
- El juego ha desaparecido del radar de las nuevas generaciones porque su formato digital no
    ha cambiado desde que era un tablero físico.

2.2 Por qué esto importa

Battleship tiene mecánicas que no requieren explicación, genera tensión genuina y promueve el
pensamiento espacial y la deducción. Que esas cualidades estén desaprovechadas en el entorno
digital es un problema concreto: los grupos que quieren jugar algo juntos en el momento no
tienen acceso a esa experiencia sin instalar aplicaciones o registrarse, y el formato 1 vs 1 excluye
por definición a cualquier grupo que quiera competir en equipo.

2.3 La oportunidad


BattleCaos-ship ocupa un espacio que hoy no existe: una versión de Battleship gratuita, sin
registro, accesible desde el navegador en segundos, con modos individuales y por equipos,
mecánicas de interacción que añaden profundidad táctica, y un modo contra la máquina que
permite practicar o probar el juego en solitario. El juego que todos conocen, rediseñado para que
personas reales lo disfruten en tiempo real.

## 3. Objetivos del Proyecto

3.1 Objetivos específicos

- Construir un servidor autoritativo en Node.js con Socket.io que gestione el estado
    completo de cada partida — tableros, disparos, poderes activos, turnos y la ventana de
    Salva — sin delegar lógica crítica al cliente.
- Implementar salas privadas accesibles por código para 2 a 4 jugadores en sus tres modos
    (1 vs 1 contra la máquina, 1 vs 1, y 2 vs 2), con reconexión automática y recuperación de
    estado ante desconexiones.
- Garantizar el aislamiento de múltiples salas corriendo en paralelo dentro del mismo
    proceso, de modo que muchas partidas independientes puedan jugarse simultáneamente sin
    interferencia entre ellas.
- Desarrollar la fase de colocación colaborativa: en el modo 2 vs 2, ambos miembros del
    equipo posicionan los barcos del tablero compartido en tiempo real durante 60 segundos
    cronometrados en el servidor.
- Implementar los cinco poderes especiales con lógica completa de energía y la Salva
    Simultánea, una ventana de fuego libre donde todos los jugadores disparan a la vez con
    cadencia controlada por el servidor.
- Mantener un motor de juego parametrizado: el tamaño del tablero y la composición de la
    flota se inyectan como configuración según el modo, sin duplicar la lógica del juego.

3.2 Indicadores de éxito medibles

```
Indicador Meta Método de medición
Tiempo entre acción del jugador y respuesta
visible en todos los clientes < 200 ms^
```
```
Pruebas con varios dispositivos
simultáneos
Salas (partidas) en paralelo sin degradación de
rendimiento ≥ 5 salas^
```
```
Prueba de carga con Socket.io
rooms
Disparos procesados sin pérdida durante la
Salva Simultánea 100 %^
```
```
Conteo servidor vs clientes en la
ventana de fuego libre
Consistencia del estado del tablero entre todos
los clientes 100 %^
```
```
Comparación de snapshots entre
clientes
Tiempo de carga inicial desde navegador frío < 3 s Lighthouse / DevTools en red 4G
```

```
Indicador Meta Método de medición
Tiempo desde cero hasta primera partida
activa < 60 s^ Medición manual con grupos reales^
```
## 4. Alcance de la Solución

4.1 Modos de juego

El juego ofrece tres modos sobre el mismo motor. El tablero y la flota se ajustan por
configuración según el modo seleccionado:

```
Modo Jugadores Oponente Tablero Flota
1 vs 1 vs máquina 1 humano Bot del servidor 10 × 10 6 barcos / 17 casillas
1 vs 1 2 Humano 10 × 10 6 barcos / 17 casillas
2 vs 2 4 (2 por equipo) Humanos 13 × 13 8 barcos / 25 casillas
```
La flota de 1 vs 1 se compone de 1 portaaviones (5), 1 acorazado (4), 2 fragatas (3) y 2
submarinos (2). En 2 vs 2, el tablero crece a 13 × 13 y la flota suma 1 portaaviones (5), 2
acorazados (4), 2 fragatas (3) y 3 submarinos (2), manteniendo una densidad de barcos similar a
la del modo clásico y dando más espacio para que dos jugadores coloquen sus piezas sin pisarse.

4.2 Sistema de salas y equipos

La base técnica del proyecto es el sistema de salas en tiempo real. Cada partida ocurre dentro de
una sala aislada, con estado propio e independiente persistido en Redis:

- Generación de código de sala único al crear la partida, sin lobby público ni búsqueda de
    partidas.
- Múltiples salas en paralelo dentro del mismo proceso Node.js: el aislamiento de sesiones
    garantiza que los eventos de una sala nunca afecten a otra.
- En 2 vs 2, canal de chat privado por equipo y canal general de partida; sistema de energía
    por equipo que se acumula con impactos y barcos hundidos.
- Reconexión automática con recuperación de estado: un jugador que se desconecta puede
    reingresar con el mismo código y retomar su posición.

4.3 Mecánicas del juego **—** MVP (2026-1)

```
Mecánica Categoría Descripción Costo
```
```
Colocación
colaborativa Fase de inicio^
```
```
En 2 vs 2, 60 segundos para posicionar juntos
los barcos del tablero compartido en tiempo
real.
```
#### —

```
Turno de disparo Mecánica base Los equipos se alternan; en 2 vs 2 los miembros disparan en rotación. —
```

```
Mecánica Categoría Descripción Costo
```
```
Salva Simultánea Evento simultáneo
```
```
Ventana de 8 s sin turnos: todos disparan a la
vez, con cadencia de 1 disparo cada 1.5 s
controlada por el servidor.
```
```
Gratis
```
```
Bomba de área Poder — Ataque Dispara a las 9 casillas de una zona 3×3 seleccionada. 2 energía
```
```
Sonar Poder Información—^ Revela si hay barco en una fila o columna completa del tablero rival. 2 energía
```
```
Escudo Poder Defensa— Protege una casilla disparo rival falla sin revelar información.propia durante 2 turnos; el 1 energía
```
```
Tormenta Poder Sabotaje— Anula el siguiente turno completo del equipo rival. Uso único por partida. 3 energía
```
```
Contramedida Poder Reacción—^
```
```
Dentro del turno del atacante: 5 s para anular
Bomba o Sonar y devolver su energía. Si se
activa, el turno termina.
```
```
1 energía
```
4.4 La Salva Simultánea y la Contramedida

La Salva Simultánea

Cada cierto número de rondas, el servidor pausa el sistema de turnos y anuncia una ventana de
Salva de 8 segundos. Durante ese lapso, todos los jugadores disparan libremente, pero cada uno
con una cadencia máxima de un disparo cada 1.5 segundos. Esta cadencia es deliberada: evita
que el resultado dependa de la velocidad de clic, el hardware o la latencia de cada jugador, y
desplaza el peso de la mecánica hacia la decisión de dónde disparar. El servidor recibe una
ráfaga de disparos concurrentes de todos los clientes, los serializa en su cola de eventos, valida
cada uno (casilla válida, no repetida, energía generada) y emite el estado resultante a todos. La
mecánica produce dos niveles de concurrencia simultánea: entre equipos y, dentro de cada
equipo, entre compañeros que deben coordinarse para no disparar a la misma casilla.

La Contramedida

A diferencia de la Salva, la Contramedida ocurre dentro del turno normal y no suma tiempo
adicional al juego. Cuando el equipo atacante activa Bomba o Sonar, el servidor abre una
ventana de 5 segundos durante la cual el equipo defensor puede hacer clic en un botón de
Contramedida para anular el poder. Esta ventana cierra en tres escenarios: (1) al pasar los 5
segundos, (2) si el equipo defensor hace clic con éxito en Contramedida, o (3) si el equipo
defensor no reacciona. En todos los casos, el turno del equipo atacante termina normalmente y
pasa al siguiente jugador en rotación. Si la Contramedida se activa a tiempo, el poder se anula y
el atacante recupera su energía gastada. Si no, el poder se resuelve según lo normal. El servidor
es el único árbitro del tiempo: los 5 segundos viven en el servidor, no en los clientes.

4.5 Escalabilidad horizontal (post-MVP)

El servidor está diseñado para escalar horizontalmente replicando el mismo monolito modular detrás de un balanceador de carga, todas conectadas al mismo Redis:

```
[Azure Load Balancer]
    ↙        ↓        ↘
[Node App 1] [Node App 2] [Node App 3]
    ↖        ↓        ↗
[Redis Upstash — estado + locks + pub/sub]
    ↖        ↓        ↗
[Timer Master — solo 1 instancia con el lease]
```

Para lograrlo se requiere:
1. **Socket.io Redis adapter** (`@socket.io/redis-adapter`) — sincroniza broadcasts entre instancias (~5 líneas).
2. **Timer Master** — patrón de elección con `SETNX` + TTL de 1s para gestionar los 4 timers de juego (colocación 60s, turno 30s, Salva 8s, Contramedida 5s) de forma tolerante a fallos.
3. **Scripts Lua** (opcional) — atomicidad compuesta para operaciones multi-key.

**Lo que NO cambia:** energía (INCRBY/DECRBY), locks de Salva (GETSET), estado de salas (Redis), snapshots y broadcasts — todo distribuido por diseño.

Ver documentación detallada en `docs/ESCALABILIDAD.md` y `docs/TIMER_MASTER.md`.

4.5 Funcionalidades de Fase 2 (iteraciones futuras)


```
Funcionalidad Descripción
```
```
Modo torneo Bracket eliminatorio entre equipos ganadores de distintas salas, reagrupados automáticamente por el servidor.
```
```
Niebla de guerra Cada equipo solo ve su propio tablero y los disparos ya realizados, sin vista completa del tablero rival.
```
```
Historial de partidas Registro persistente de estadísticas por jugador: partidas, hundidos, precisión y poderes usados. victorias, barcos
```
```
Vista espectador Jugadores sin equipo pueden observar la partida en tiempo real con acceso a ambos tableros.
```
```
Dificultad del bot Niveles de dificultad para el oponente máquina, desde disparo hasta búsqueda dirigida avanzada. aleatorio
```
```
Documentación técnica docs/ESCALABILIDAD.md, docs/TIMER_MASTER.md
```
4.6 Diferenciadores frente a otras versiones

```
Aspecto Battleship digital actual BattleCaos-ship
```
```
Formato de juego Duelo 1 vs 1 sin excepción 1 vs 1, 1 vs 1 vs máquina y 2 vs 2 por equipos
```
```
Acceso App, descarga o cuenta requerida Navegador, código de sala, sin registro
```
```
Comunicación Inexistente Chat privado por equipo en tiempo real
```
```
Interacción táctica Solo disparo básico 5 poderes y la Salva Simultánea
Preparación Individual y sin coordinación Colocación colaborativa en 2 vs 2
```
```
Sincronización Alta latencia — polling HTTP Baja latencia persistente —^ WebSocket
```
```
Costo de acceso Pago o publicidad invasiva Completamente gratuito
```
## 5. Beneficios Esperados

5.1 Cuantitativos

- El tiempo de acceso se reduce más del 90%: de 5 o más minutos (descargar, instalar y
    registrarse) a menos de 60 segundos compartiendo un código de sala.
- Latencia de sincronización entre clientes estimada entre 30 y 80 ms en red local o WiFi
    doméstica, frente a los 300–800 ms típicos de soluciones basadas en polling HTTP.
- El módulo de WebSockets y gestión de salas es agnóstico al tipo de juego y al tamaño del
    tablero: puede reutilizarse como base para cualquier proyecto multijugador futuro sin
    rediseñar el backend.
- Infraestructura con costo operativo conocido y controlable: Render free tier para el
    servidor y Upstash free tier para el estado de partidas en Redis.


5.2 Cualitativos

- Devuelve relevancia a un juego clásico, atractivo para quienes lo conocen y para
    generaciones que nunca lo jugaron, sin curva de aprendizaje.
- Transforma una actividad individual y silenciosa en una experiencia social activa: la
    colocación colaborativa, el chat de equipo y los poderes generan conversación y decisiones
    grupales.
- El modo contra la máquina permite practicar o probar el juego en solitario, sin depender de
    que haya otras personas disponibles — útil tanto para los jugadores como para el
    desarrollo y las demostraciones.
- Demuestra en la práctica patrones de ARSW — WebSockets, eventos, estado distribuido,
    aislamiento de sesiones y concurrencia — en un contexto funcional y demostrable en vivo.

## 6. Análisis de Viabilidad

6.1 Viabilidad técnica **—** Arquitectura y stack

El juego opera bajo una arquitectura cliente-servidor autoritativa: toda la lógica crítica (estado
del tablero, validación de disparos, activación de poderes, control de turnos y timers) vive
exclusivamente en el servidor. Los clientes envían acciones y renderizan el estado que reciben,
nunca calculan resultados de forma autónoma. Esto garantiza consistencia entre todos los
jugadores simultáneos y resuelve de forma central las condiciones de carrera que surgen durante
la Salva Simultánea.

```
Capa Tecnología Justificación
```
```
Servidor Node.js 20 LTS
```
```
Event loop no bloqueante ideal para muchas conexiones
WebSocket y ráfagas de eventos concurrentes. Mismo
lenguaje que el frontend.
```
```
Tiempo real Socket.io 4.x Abstrae WebSocket con rooms por sala, broadcast selectivo, acuse de recibo y reconexión automática.
```
```
Frontend React + Vite Actualización reactiva del DOM ante cada evento de socket. Vite reduce los tiempos de build en desarrollo.
```
```
Estado de salas Redis — Upstash
```
```
Estado de cada sala en memoria para recuperación y
operaciones atómicas. Free tier: 500K comandos/mes, 256
MB.
```
```
Despliegue Render Service —^ Web Free tier real sin tarjeta, detección automática de Node.js y deploy por push a Git.
```
```
UI del tablero CSS Grid + SVG Tablero construido programáticamente, parametrizable a 10×10 o 13×13 sin dependencias gráficas externas.
```
Para la autenticación y gestión de identidad:


```
Capa Tecnología Justificación
```
```
Autenticación Google OAuth 2.0 Login de un clic sin contraseñas. Los usuarios inician sesión con su cuenta Google existente.
```
```
ID del jugador JWT + Google ID Token transmitido en cada conexión WebSocket.firmado que contiene el ID de Google del usuario,
```

**Escalabilidad horizontal:** El servidor es un monolito modular (auth, rooms, game-engine, state-manager) diseñado para replicarse horizontalmente. Con el Redis adapter de Socket.io y el patrón Timer Master (SETNX + heartbeat), N instancias pueden ejecutarse detrás de un balanceador sin cambios en la lógica de negocio. Ver `docs/ESCALABILIDAD.md`.

```
6.2 Viabilidad económica **—** Costos reales verificados (junio 2026)

```
Recurso / Servicio Plan Costo Condiciones y límites
Render — Servidor
Node.js Free^ USD 0 / mes^
```
```
750 h/mes de cómputo. Se duerme tras 15 min
de inactividad; arranque en 30–60 s.
```
```
Upstash — Redis Free USD 0 / mes 500.000 comandos/mes y 256 MB. Suficiente para el MVP con varias salas concurrentes.
```
```
Vercel — Frontend
React Hobby^ USD 0 / mes^
```
```
Hosting de la SPA con CDN global. Sin límite
de solicitudes para proyectos personales.
```
```
Dominio (opcional) — USD 12 / año Solo si la presentación requiere URL propia en lugar del subdominio de Render.
```
```
Repositorio — GitHub Free USD 0 / mes Repositorio privado gratuito, integrado con Render para CI/CD por push.
```
```
TOTAL estimado — USD 0 – 12 Viable con presupuesto cero para desarrollo, pruebas y demostración académica completa.
```
6.3 Viabilidad operativa **—** Cronograma de desarrollo

```
Semanas Actividad principal
```
#### 1 – 2

```
Servidor Node.js + Socket.io operativo. Creación de salas por código, múltiples salas en
paralelo, sincronización del tablero parametrizado entre clientes y chat por equipo.
Prototipo mínimo jugable al final de la semana 2.
```
#### 2 – 3

```
Lógica completa del juego: colocación colaborativa con tiempo límite, turnos alternos,
validación de disparos y detección de hundimientos. Modo 1 vs 1 vs máquina con bot
básico de «cazar y rematar».
```
#### 3 – 4

```
Los cinco poderes con lógica de energía, la ventana de Contramedida de 5 s y la Salva
Simultánea con resolución atómica de disparos. Pruebas de integración con varias salas
y jugadores reales.
```
```
4 Interfaz completa con animaciones, pantalla de victoria, deploy en Render + Upstash, corrección de bugs y preparación de la demostración final.
```
## 7. Riesgos y Mitigaciones


```
Riesgo Nivel Consecuencia Mitigación
```
```
Disparos concurrentes en
la Salva sobre la misma
casilla
```
```
Alto Doble conteo o estado inconsistente del tablero
```
```
El servidor serializa los eventos
y valida cada disparo de forma
atómica; rechaza casillas ya
resueltas.
```
```
El cliente recibe eventos
de socket en orden
incorrecto
```
```
Alto Tablero desfasado entre jugadores del mismo equipo
```
```
El servidor envía snapshots
completos del estado; el cliente
descarta su estado local y pinta
lo recibido.
```
```
El free tier de Render
duerme el servidor entre
pruebas
```
```
Medio Cold start de 30demo – 60 s en la
```
```
Request de calentamiento antes
de la presentación. Para
producción, plan Starter de
Render.
```
```
La ventana de la Salva o
la Contramedida se
desincroniza
```
```
Medio Un jugador actúa fuera del tiempo permitido
```
```
Los timers viven solo en el
servidor; los clientes reciben
ticks por socket y muestran el
valor recibido.
```
```
El alcance crece con
modos o mecánicas extra
antes de terminar el MVP
```
```
Alto
```
```
El juego base no queda
funcional para la
presentación
```
```
La semana 1 fija el corte del
MVP. El bot y la Salva solo
entran tras tener el core jugable
con salas reales.
Fallo de autenticación
Google (credenciales
vencidas o cambio de
política)
```
```
Bajo Jugador desconectado sin poder reconectarse
```
```
Validar tokens de Google en
cada evento de socket.
Implementar refresh automático
de tokens.
```
```
Trabajo en solitario con
tiempo limitado Medio^
```
```
Funcionalidades sin terminar
a tiempo
```
```
Priorización estricta: core
jugable primero, poderes
después, pulido al final. Fase 2
explícitamente fuera del MVP.
```
## 8. Conclusión y Recomendación

Battleship lleva décadas siendo uno de los juegos de estrategia más reconocidos del mundo y, sin
embargo, ninguna plataforma digital ha apostado por llevarlo más allá del duelo individual.
BattleCaos-ship llena ese vacío de forma concreta: convierte una mecánica que todos dominan
en una experiencia con modos individuales y por equipos, sabotaje, coordinación en tiempo real
y acceso en segundos desde cualquier navegador.

El diferenciador no está en reinventar el juego, sino en añadir exactamente lo que le faltaba: más
de un formato de juego, comunicación entre jugadores, decisiones tácticas compartidas y
momentos de acción simultánea que hacen cada partida distinta. La Salva Simultánea, en
particular, convierte el proyecto en un caso claro de concurrencia real — múltiples clientes


compitiendo por los mismos recursos en la misma ventana de tiempo — y no solo de
sincronización rápida.

Desde la perspectiva de Arquitecturas de Software, el proyecto es un caso de estudio completo y
demostrable: un servidor autoritativo en Node.js gestionando estado compartido entre múltiples
clientes concurrentes mediante WebSockets, múltiples salas aisladas corriendo en paralelo,
snapshots de estado para garantizar consistencia, y resolución atómica de condiciones de carrera
visible y medible en la presentación.

La viabilidad económica está verificada con costos reales a junio de 2026: el costo total del
proyecto va de USD 0 a USD 12 dependiendo de si se necesita un dominio propio. La
recomendación es empezar por el servidor, el aislamiento de salas y la sincronización del tablero
antes de cualquier otra funcionalidad. Una vez que el tablero funciona en tiempo real con varias
salas en paralelo, todo lo demás — poderes, Salva, chat, modos — se construye sobre esa base
sin modificar lo que ya funciona.

La arquitectura está preparada para escalar horizontalmente: el monolito modular, el estado
centralizado en Redis y el patrón Timer Master documentado en `docs/ESCALABILIDAD.md`
permiten replicar el servidor a N instancias sin rediseñar la lógica de negocio.


