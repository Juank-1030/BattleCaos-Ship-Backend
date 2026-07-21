# Guía de pruebas de observabilidad — BattleCaos

Cómo levantar Prometheus, Grafana y Loki, generar tráfico real y **comprobar** que las
métricas, los logs y las alertas funcionan.

Este documento no describe lo que *debería* pasar: describe una ejecución real hecha el
**20 de julio de 2026**, con los números que salieron. Si sigues los pasos deberías obtener
resultados equivalentes.

> El código y los scripts viven en el repositorio de trabajo (`E:\ARSW\Proyecto`), que
> contiene los 11 repos de servicios. Este documento es la guía; la referencia técnica
> completa está en `deploy/monitoring/GUIA-OBSERVABILIDAD.md` de ese repositorio.

---

## 0. Qué vas a probar

| Herramienta | Señal | Qué demuestra |
|---|---|---|
| **Prometheus** | Métricas | Cuántos eventos, con qué latencia, qué está caído |
| **Loki** | Logs | Qué pasó exactamente, y el rastreo de un evento por todos los servicios |
| **Grafana** | Ambas | El panel único donde se ve todo junto |

Complementos: **Alertmanager** (alertas), **Jaeger** (trazas, opcional) y
**kafka-exporter** (retraso de los consumidores).

### Todas las direcciones, de un vistazo

| Herramienta | URL | Login | ¿Dónde se explica? |
|---|---|---|---|
| **Grafana** | http://localhost:3030 | `admin` / `battlecaos` | §5 |
| **Prometheus** | http://localhost:9090 | — | §3 y §6.1 |
| **Alertmanager** | http://localhost:9093 | — | §6.2 |
| **Jaeger** | http://localhost:16686 | — | §6.3 |
| **Kafka UI** | http://localhost:8080 | — | §6.4 (hay que arrancarlo) |
| **Loki** | http://localhost:3100 | — | §6.5 (sin interfaz propia) |
| **Balanceador** | http://localhost:8090/lb-health | — | §7.1 |

**Empieza siempre por Grafana.** Es el único que presenta las cosas ya montadas; los demás
son crudos y sirven para comprobaciones puntuales.

---

### Cómo usar los comandos de esta guía

**Todos los comandos son de PowerShell**, listos para copiar y pegar en una ventana de
**PowerShell 7** situada en `E:\ARSW\Proyecto`:

```powershell
cd E:\ARSW\Proyecto
```

Dos cosas que conviene tener claras si has visto ejemplos de Linux por ahí:

- **Los comandos partidos en varias líneas con `\` al final son de bash y aquí NO funcionan.**
  Si pegas solo la primera línea, el comando se ejecuta incompleto (típico:
  `parse error: no expression found in input`). Los de esta guía van en una sola línea o en
  bloques que se pegan enteros.
- **Las variables de entorno se asignan antes**, con `$env:NOMBRE='valor'`. Ponerlas delante
  del comando (`RONDAS=80 node …`) es sintaxis de Linux y da error.

Los pocos comandos que no son de PowerShell (`docker`, `node`, `gh`) funcionan igual porque
son programas externos.

---

## 1. Requisitos

- Docker Desktop **abierto y corriendo** (si no, todo lo demás falla).
- Node.js 20+.
- PowerShell 7 (`pwsh`).
- Los `.env` de `battlecaos-gateway` y `battlecaos-auth` presentes (`JWT_SECRET`, `MONGO_URL`).

---

## 2. Levantar el entorno (5 comandos)

Desde la raíz del repositorio de trabajo, **en este orden**:

```powershell
# 1. Infraestructura: Redis + Zookeeper + Kafka
docker compose -f docker-compose.yml up -d zookeeper kafka redis

# 2. Observabilidad: Prometheus, Grafana, Loki, promtail, Jaeger, Alertmanager, kafka-exporter
docker compose -f deploy/monitoring/docker-compose.monitoring.yml up -d

# 3. Balanceador de los 3 gateways (nginx en contenedor; no hay que instalar nada)
docker compose -f deploy/docker-compose.lb.yml up -d

# 4. Los 9 microservicios, con 3 réplicas de gateway
pwsh ./levantar-todo.ps1 -Gateways 3

# 5. Frontend (opcional, solo si quieres jugar a mano)
cd battlecaos-frontend ; npm run dev
```

El orden importa: el paso 2 se conecta a la red de Docker que crea el paso 1, y el paso 3
necesita que los gateways del paso 4 existan (aunque los levante después, nginx los
reintenta solo).

**Comprobación inmediata** — deben salir 11 contenedores:

```powershell
docker ps --format "{{.Names}}" | Sort-Object
```

```
battlecaos-alertmanager     battlecaos-loki
battlecaos-gateway-lb       battlecaos-prometheus
battlecaos-grafana          battlecaos-promtail
battlecaos-jaeger           battlecaos-redis
battlecaos-kafka            battlecaos-zookeeper
battlecaos-kafka-exporter
```

Y los 11 servicios respondiendo:

```powershell
3000,3010,3020,3001,9101,9102,9103,9104,9105,9106,9107 | ForEach-Object {
  $c = try { (Invoke-WebRequest "http://localhost:$_/health" -TimeoutSec 5).StatusCode } catch { 'X' }
  "$_`:$c"
} | Join-String -Separator '  '
```

Resultado esperado (y obtenido): `3000:200 3010:200 3020:200 3001:200 9101:200 … 9107:200`

### Si las 11 ventanas se cierran con `node: The term 'node' is not recognized`

Node sí está instalado; lo que falla es el PATH. `Start-Process` abre cada ventana heredando
el entorno de **la terminal desde la que lanzaste el script**, no el PATH del registro. Si
esa terminal se abrió antes de instalar Node (o alguien tocó el PATH), las ventanas nacen
sin `node` y mueren al instante.

Ya está mitigado: `levantar-todo.ps1` resuelve la ruta completa a `node.exe` al arrancar,
la imprime (`Node: C:\Program Files\nodejs\node.exe`) y la usa explícitamente. Si aun así
no lo encuentra, aborta con un mensaje claro en vez de dejar 11 ventanas muertas.

Si el mensaje de error aparece, **cierra y reabre la terminal** — eso recarga el PATH del
registro. Lo mismo aplica al `npm run dev` del frontend, que lo ejecutas tú a mano.

### Si Kafka no arranca

Síntoma en `docker logs battlecaos-kafka`: `NodeExistsException … /brokers/ids/1`.
Zookeeper guardó el registro de un broker de una sesión anterior. Se recrean los dos:

```powershell
docker compose -f docker-compose.yml rm -sf zookeeper kafka
docker compose -f docker-compose.yml up -d zookeeper kafka
```

---

## 3. Comprobar que Prometheus recoge todo

> **Lee esto antes de abrir Prometheus.** Prometheus **no tiene paneles ni gráficas
> preparadas**: es una base de datos con una caja de búsqueda. Si entras esperando ver el
> estado del sistema, te vas a encontrar una pantalla vacía y vas a pensar que algo falla.
>
> **El sitio para mirar es Grafana** (§5). Prometheus sirve para dos cosas concretas:
> comprobar que los servicios están siendo leídos (§3.1) y lanzar una consulta suelta
> cuando dudas de un número (§3.2).

### 3.1 Ver que los 12 servicios están siendo leídos

1. Abre **http://localhost:9090**
2. En el menú de arriba: **Status** → **Targets**
   (la versión aquí es Prometheus 2.53; en Prometheus 3.x ese menú se llama **Target health**)
3. Verás una lista agrupada por *job*: `auth`, `bot`, `chat`, `game`, `gateway`, `kafka`,
   `observability`, `room`, `timer`, `voice-channel`

Cada fila debe tener una etiqueta verde **UP**. Deben ser **12 en total** (el job `gateway`
tiene 3 filas, una por réplica). Si alguna sale roja **DOWN**, ese servicio está caído o no
arrancó.

Lo mismo desde la terminal:

```powershell
$t = (Invoke-RestMethod "http://localhost:9090/api/v1/targets").data.activeTargets
"$(($t | Where-Object health -eq 'up').Count) / $($t.Count) UP"
```

Resultado obtenido: **`12 / 12 UP`**.

### 3.2 Lanzar una consulta

1. Pestaña **Graph**, la primera del menú (es la pantalla que sale al entrar a
   http://localhost:9090)
2. Escribe la consulta en la caja grande de arriba
3. Pulsa el botón azul **Execute**
4. Debajo aparecen dos pestañas: **Table** (el valor actual, un número) y **Graph** (su
   evolución en el tiempo). Si usas **Graph**, sube el rango de tiempo (el control `1h`
   a la izquierda del gráfico) hasta que abarque el momento en que jugaste.

Consultas para empezar — cópialas tal cual:

| Consulta | Qué responde |
|---|---|
| `up` | Qué servicios están vivos (1) o caídos (0) |
| `sum(gateway_active_sockets)` | Cuántos jugadores hay conectados ahora |
| `gateway_active_sockets` | Lo mismo, desglosado por réplica de gateway |
| `sum(gateway_events_routed_total)` | Total de eventos enviados a Kafka |
| `sum by (job) (events_processed_total)` | Eventos procesados por cada microservicio |
| `sum(kafka_consumergroup_lag)` | Retraso de los consumidores (0 = al día) |

> **Los nombres importan.** Las métricas del gateway llevan prefijo `gateway_`: son
> `gateway_active_sockets`, `gateway_events_routed_total`. Si escribes `active_sockets` a
> secas no sale nada y parece roto cuando no lo está. La caja de consulta **autocompleta**:
> escribe `gateway_` y espera a que aparezca la lista.

### 3.3 Si sale "Empty query result"

Casi siempre es una de estas tres, en este orden de probabilidad:

1. **No hay tráfico todavía.** Es lo más común. Un contador con etiquetas (`events_processed_total`)
   **no existe hasta que se incrementa por primera vez**: sin partidas, no hay serie que
   mostrar. Solución: genera tráfico (§4).
2. **Reiniciaste los servicios.** Los contadores viven en memoria y **vuelven a cero** al
   reiniciar. Si jugaste y luego relanzaste `levantar-todo.ps1`, los totales se perdieron.
   La historia anterior sigue en Prometheus, pero hay que consultarla con un rango de tiempo,
   no con el valor actual.
3. **El nombre está mal escrito.** Usa el autocompletado.

Para distinguir el caso 1 del caso 2, mira si hubo actividad en las últimas 2 horas:

```powershell
$fin=[int][double]::Parse((Get-Date -UFormat %s)); $ini=$fin-7200
$r=(Invoke-RestMethod "http://localhost:9090/api/v1/query_range" -Body @{query='sum(gateway_active_sockets)';start=$ini;end=$fin;step=120}).data.result
if($r){ "momentos con jugadores conectados: " + ($r[0].values | Where-Object { [int]$_[1] -gt 0 }).Count } else { 'sin serie' }
```

Si devuelve `0`, nadie estuvo conectado (o estuvo menos de los 2 minutos del `step`).

---

## 4. Generar tráfico real

Sin tráfico los paneles están vacíos y no se prueba nada. Dos opciones.

### Opción A — jugar a mano (más realista)

#### A.1 Apuntar el frontend al balanceador

⚠️ **El fallo más fácil de cometer.** Vite tiene **dos** archivos de configuración y
`.env.local` **gana sobre** `.env`. Si editas `.env` mientras existe `.env.local`, tu cambio
**no surte ningún efecto** y el frontend sigue conectándose a una sola réplica, sin pasar por
el balanceador — todo parecerá funcionar, pero no estarás probando el balanceo.

Edita **`battlecaos-frontend/.env.local`** (no `.env`):

```
VITE_GATEWAY_URL=http://localhost:8090
```

Comprueba que no quede la variable definida dos veces:

```powershell
Select-String -Path .\battlecaos-frontend\.env, .\battlecaos-frontend\.env.local -Pattern "VITE_GATEWAY_URL"
```

Debe salir **una sola línea con `8090`** (la de `.env` puede quedar en `:3000`, pero
entonces `.env.local` debe existir y llevar el `8090`).

**Vite lee los `.env` solo al arrancar**: si ya tenías `npm run dev` corriendo, párralo con
`Ctrl+C` y vuelve a lanzarlo. Recargar el navegador no basta.

#### A.2 Comprobar que de verdad entra por el balanceador

Con el frontend abierto y sesión iniciada, en otra terminal:

```powershell
(Invoke-RestMethod "http://localhost:9090/api/v1/query" -Body @{ query = 'gateway_active_sockets' }).data.result | ForEach-Object { "{0,-32} {1}" -f $_.metric.instance, $_.value[1] }
```

Debe aparecer **`1`** en una de las tres instancias (`:3000`, `:3010` o `:3020`). Si sale
en `:3000` siempre y nunca en las otras, sospecha del `.env.local`.

#### A.3 Generar tráfico suficiente

Un login suelto **no basta**: deja un único punto en las gráficas y Prometheus parecerá
vacío. Para que los paneles tengan forma, haz una partida completa:

1. Abre **dos** pestañas (o dos navegadores distintos, o una en incógnito — dos sesiones).
2. En la primera: inicia sesión y **crea una sala**. Anota el código.
3. En la segunda: inicia sesión con **otra cuenta** y **únete** con ese código.
4. Pulsa **comenzar** y **coloca las flotas** en ambas.
5. **Dispara 20–30 veces** alternando, y escribe algún mensaje en el chat.
6. **Deja las pestañas abiertas** un par de minutos: `gateway_active_sockets` es un valor
   instantáneo, y si cierras todo se va a 0 y las gráficas quedan planas.

Cuantas más jugadas, más claras salen las gráficas de latencia y de eventos por segundo.

### Opción B — script automatizado (recomendado para probar rápido)

Dos jugadores simulados juegan una partida completa a través del balanceador: crean sala,
se unen, colocan flota, cambian de fase, disparan y chatean. No necesitas abrir el
navegador ni tener dos cuentas.

```powershell
node tools/generar-trafico.mjs                          # partida de 40 jugadas

$env:RONDAS=80; node tools/generar-trafico.mjs          # más larga, gráficas más claras
$env:RONDAS=$null                                       # (deja la variable como estaba)

$env:URL='http://localhost:3000'; node tools/generar-trafico.mjs   # sin pasar por el balanceador
$env:URL=$null
```

> En PowerShell las variables de entorno **no** se ponen delante del comando (`RONDAS=80 node …`
> es sintaxis de Linux y da error): se asignan antes con `$env:NOMBRE='valor'`.

Tarda menos de un minuto y deja el sistema con datos suficientes para que todos los paneles
tengan forma. Es lo que se usó para medir los números de este documento.

> Requiere `battlecaos-frontend/node_modules` instalado (el script reutiliza el
> `socket.io-client` del frontend en vez de duplicar el paquete). Si falla, ejecuta
> `cd battlecaos-frontend ; npm install`.

### Comprobar que el tráfico llegó

#### La forma fácil: un solo comando

```powershell
pwsh ./tools/ver-metricas.ps1
```

Imprime de un golpe: servicios arriba, alertas activas, tráfico, eventos por servicio de
dominio, reparto entre réplicas de gateway y líneas de log en Loki. Es lo más cómodo y
evita por completo los problemas de comillas.

#### A mano, desde PowerShell

⚠️ **Los ejemplos de una sola línea, siempre.** En esta guía algunos comandos aparecen
partidos en dos líneas con `\` al final — eso es sintaxis de **bash**. Si copias solo la
primera línea en PowerShell, la consulta se envía **vacía** y Prometheus responde:

```
{"status":"error","errorType":"bad_data","error":"invalid parameter \"query\": ...
 parse error: no expression found in input"}
```

No es que Prometheus esté mal: es que le pediste una consulta vacía. Usa cualquiera de estas
tres formas, **todas en una sola línea**:

```powershell
# 1. Nativo de PowerShell — el más limpio, codifica la consulta solo
(Invoke-RestMethod "http://localhost:9090/api/v1/query" -Body @{ query = 'sum by (job) (events_processed_total)' }).data.result | ForEach-Object { "{0,-16} {1}" -f $_.metric.job, $_.value[1] }

# 2. curl, todo en una línea
curl -s -G http://localhost:9090/api/v1/query --data-urlencode "query=sum by (job) (events_processed_total)"

# 3. curl con la URL ya codificada
curl -s "http://localhost:9090/api/v1/query?query=sum%20by%20(job)%20(events_processed_total)"
```

#### Desde la interfaz de Prometheus

Si prefieres verlo en el navegador: **http://localhost:9090** → pestaña **Graph** → pega
`sum by (job) (events_processed_total)` en la caja → botón **Execute** → pestaña **Table**.
Saldrá una fila por servicio.

Resultado obtenido tras `RONDAS=30` (los números crecen con las jugadas; lo importante es
que **aparezcan los 7 servicios**, no la cifra exacta):

| servicio | eventos |
|---|---|
| observability | 295 |
| timer | 32 |
| game | 32 |
| bot | 15 |
| chat | 7 |
| room | 5 |
| voice-channel | 3 |

Y el reparto entre réplicas de gateway:

```powershell
(Invoke-RestMethod "http://localhost:9090/api/v1/query" -Body @{ query = 'sum by (instance) (gateway_events_routed_total)' }).data.result | ForEach-Object { "{0,-32} {1}" -f $_.metric.instance, $_.value[1] }
```

Obtenido: `:3000 → 21` y `:3020 → 19`. Los dos jugadores cayeron en réplicas distintas —
eso es el balanceador haciendo su trabajo.

---

## 5. Grafana: el panel y los logs

**Este es el sitio donde hay que mirar.** Grafana lee de Prometheus y de Loki y lo presenta
ya montado; no hay que escribir consultas.

Los pasos de abajo están verificados sobre **Grafana 11.1.0**, que es la versión que levanta
el `docker-compose`. Si ves otros nombres, mira la versión abajo del todo en la pantalla de
login.

### 5.1 Entrar y abrir el panel

1. Abre **http://localhost:3030**
2. Pantalla *Welcome to Grafana*: escribe **`admin`** en «Email or username» y
   **`battlecaos`** en «Password» → botón azul **Log in**.
   (No pide cambiar la contraseña; si en tu caso lo pidiera, puedes saltarlo.)
3. Arriba a la izquierda, el icono de **☰** (tres rayas) abre el menú lateral. Las opciones
   son: *Home · Starred · Dashboards · Explore · Alerting · Connections · Administration*.
4. Pulsa **Dashboards**.
5. Entra en la carpeta **BattleCaos** y abre **BattleCaos — Operación**.
   Viene pre-cargado: no hay que importar ni crear nada.

> **Atajo:** puedes ir directo con
> `http://localhost:3030/d/battlecaos-ops?from=now-15m&to=now&refresh=5s`
> — esa URL ya trae el rango y el refresco puestos.

### 5.2 Ajustar el rango de tiempo

⚠️ **La segunda causa de "no veo nada".** En la barra superior, a la derecha, hay un botón
con un **icono de reloj** que pone `Last 6 hours` por defecto. Si tu partida duró dos minutos,
queda como un pico invisible dentro de seis horas.

1. Pulsa el botón del reloj → se abre una lista de rangos.
2. Elige **`Last 15 minutes`** justo después de jugar.
3. A su derecha hay un **icono de recarga circular** con una flechita: despliégala y elige
   **`5s`**. Así el panel se actualiza solo y ves las gráficas moverse mientras juegas.

### 5.3 Los 20 paneles, ordenados de "¿está vivo?" a "¿por qué falló?"

1. **Fila superior** — sockets activos, conexiones, reconexiones, crashes del consumer.
2. **Sockets por instancia de gateway** — las 3 líneas (`gw1`, `gw2`, `gw3`) se mueven
   juntas. Una plana en 0 = réplica fuera.
3. **Eventos ruteados a Kafka** — el caudal real del juego.
4. **Latencia HTTP p95** y **Logins**.
5. **Servicios arriba (up)** — debe marcar **12**.
6. **Salud de Redis por servicio** — sin respaldo configurado solo aparece `primary`;
   **que no aparezca `secondary` es lo correcto**, no un fallo.
7. **Eventos / p95 / tasa de error por servicio** — aquí se ve qué servicio se degrada.
8. **Error budget (SLO)** — disponibilidad del gateway (99.5%) y éxito de eventos (99%).
   100% = presupuesto intacto.
9. **Kafka consumer lag** — 10 grupos, todos en 0 si nadie va retrasado.
10. **Dos paneles de logs** — bajando del todo. Vienen de Loki, no de Prometheus.

### 5.4 Trucos dentro de un panel

- **Ampliar uno solo:** pasa el ratón por encima y pulsa la tecla **`v`** (o menú `⋮` del
  panel → *View*). Vuelve con **Esc**.
- **Ver la consulta que usa:** menú `⋮` → *Edit*. Abajo aparece la expresión PromQL/LogQL
  exacta, que puedes copiar a Prometheus o a Explore.
- **Aislar una serie:** haz clic en su nombre en la leyenda. Con `Ctrl`+clic añades varias.
- **Acercar un intervalo:** arrastra sobre la gráfica seleccionando de izquierda a derecha.

### 5.5 Cómo se lee una línea de log (léelo antes de usar Explore)

Una línea cruda de Loki se ve así:

```json
{"level":30,"time":1784572658062,"pid":20668,"hostname":"DESKTOP-URK5QH8",
 "name":"room","cid":"c8c85a6f-f376-4492-910b-e2867dc432ea",
 "msg":"jugador \"Capitana Ana\" (ana-real) unido a sala 233380 — equipo B"}
```

Campo por campo:

| Campo | Qué es | Ojo |
|---|---|---|
| `level` | Nivel numérico: **30**=info, **40**=warn, **50**=error | En Grafana tienes la etiqueta `level_str` ya legible |
| `time` | Momento del evento, en milisegundos | Grafana ya lo muestra como hora |
| `pid` | Id del proceso del sistema operativo | Ruido; sirve para distinguir dos procesos del mismo servicio |
| `hostname` | **El nombre de TU PC** (o del contenedor) | ⚠️ **No es un jugador.** `DESKTOP-...` es tu máquina |
| `name` | **Qué microservicio** escribió la línea | Es la etiqueta principal para filtrar |
| `cid` | **correlationId**: identifica UN evento a través de todos los servicios | La clave del rastreo (§5.7) |
| `instancia` | Solo en gateways: `gw1`/`gw2`/`gw3` | Para ver qué réplica atendió |
| `msg` | **El mensaje.** Aquí está lo que pasó y quién lo hizo | Es lo que quieres leer |

> **Si viste `DESKTOP-...` y pensaste que era el jugador**: eso es el nombre de tu equipo,
> que la librería de logs añade a todas las líneas. **El jugador está en `msg`**, con este
> formato: `jugador "Capitana Ana" (ana-real) unido a sala 233380 — equipo B` — primero el
> apodo entre comillas, luego su id entre paréntesis.

#### ¿Veo también lo que hace el otro jugador?

**Sí, todo.** Los logs los escribe el **servidor**, no tu navegador: recogen lo que hicieron
**todos** los jugadores de todas las partidas. Si jugaste con un amigo, sus acciones están
ahí igual que las tuyas. Para verlo:

```logql
{name="room"}
```

Verás una línea por cada jugador que entra y sale, con su apodo. Por ejemplo, la salida real
de una prueba con dos jugadores:

```
sala creada: 233380 (“Sala de Capitana Ana”) — modo: 1v1
jugador "Beto" (beto-real) unido a sala 233380 — equipo B
jugador "Capitana Ana" (ana-real) desconectado de sala 233380
jugador "Beto" (beto-real) desconectado de sala 233380
sala 233380 sin jugadores — destruyendo
```

#### Ver solo el mensaje, sin el ruido

Para quitar `pid`, `hostname` y demás y quedarte con lo legible, añade `line_format`:

```logql
{name=~".+"} | json | line_format "{{.name}} | {{.msg}}"
```

Es exactamente lo que hacen los dos paneles de logs del dashboard.

### 5.6 Explore: consultar los logs a mano (Loki)

Los paneles del dashboard traen dos vistas fijas de logs. Para **buscar** hace falta Explore:

1. Menú **☰** → **Explore**.
2. Arriba a la izquierda hay un **desplegable de fuente de datos**: elígelo y selecciona
   **Loki** (por defecto puede venir Prometheus).
3. A la derecha de la caja de consulta hay dos botones: **Builder** y **Code**.
   Pulsa **Code** para poder escribir la consulta directamente.
4. Escribe la consulta y pulsa **Shift+Enter** (o el botón azul *Run query*, arriba a la
   derecha).
5. Ajusta el rango de tiempo igual que en §5.2.

Lo que verás:

- **Logs volume** — un histograma de cuántas líneas hay por instante. Sirve para localizar
  de un vistazo *cuándo* pasó algo.
- **Logs** — las líneas. Cada una se despliega con la **flecha ▸** de la izquierda, que
  muestra los campos (`name`, `level_str`, `cid`, `instancia`…) ya separados.
- Interruptores útiles: **Wrap lines** (para que no se corte el texto) y **Prettify JSON**
  (formatea el JSON crudo). **Newest first / Oldest first** cambia el orden.
- Botón **Live** (arriba a la derecha) — muestra los logs en tiempo real, como un `tail -f`.

Comprobado en esta máquina: la consulta `{name=~".+"}` devolvió **141 líneas** en 15 minutos.

#### Consultas de ejemplo

```logql
{name="game"}                             # todo lo de un servicio
{name=~".+", level_str="error"}           # solo errores, de todos los servicios
{name="gateway", instancia="gw2"}         # una réplica concreta del gateway
{name=~".+"} |= "37e3680b-7f45-45ac"      # UN evento por TODOS los servicios
```

Etiquetas para filtrar: `name` (servicio), `level_str` (`info`/`warn`/`error`),
`instancia` (solo gateways), `origen`.

> Si no te sabes las etiquetas, pulsa **Label browser** (junto a la caja de consulta): lista
> las disponibles y sus valores, y construye la consulta por ti.

De dónde salen: cada servicio escribe su log JSON en `logs/<servicio>.log`; promtail los
sigue y los manda a Loki. Los gateways escriben **uno por instancia**
(`gateway-gw1.log`, …). Esa carpeta **se vacía en cada arranque** de `levantar-todo.ps1`,
para que cada sesión de pruebas empiece limpia.

### 5.7 La prueba que hay que enseñar: rastrear un evento

Cada línea lleva un **correlationId** (`cid`) que acompaña al evento por todo el sistema.

1. En Explore con Loki, lanza `{name=~".+"}`.
2. Despliega cualquier línea con la flecha **▸** y copia el valor del campo **`cid`**.
3. Cambia la consulta a `{name=~".+"} |= "<pega-aquí-el-cid>"` y ejecuta.
4. Pon el orden en **Oldest first** para leerlo como una línea de tiempo.

Resultado real obtenido al rastrear la desconexión de un jugador:

```
15:14:08.021  room           jugador jugador-beto desconectado de sala 499236
15:14:08.024  room           sala 499236 sin jugadores — destruyendo
15:14:08.025  timer          sala 499236 — timer pausado por desconexión
15:14:08.028  game           sala 499236 — jugador-beto desconectado, turno pausado: false
15:14:08.043  voice-channel  sala 499236 — canal de voz cerrado (sala destruida)
```

**Un evento, 4 servicios, 22 milisegundos, en orden.** Eso es lo que resuelve la
observabilidad: sin esto habría que abrir 9 ventanas y cuadrar las horas a mano.

---

## 6. Las otras interfaces

### 6.1 Prometheus — http://localhost:9090

Sin login. Menú superior: **Alerts · Graph · Status ▾ · Help**.

| Quiero… | Dónde |
|---|---|
| Ver si los servicios responden | **Status → Targets** |
| Lanzar una consulta | **Graph** (§3.2) |
| Ver qué alertas hay definidas y cuáles están disparadas | **Alerts** |
| Ver la configuración cargada | **Status → Configuration** |
| Ver las reglas de alerta y de SLO | **Status → Rules** |

En **Targets** los servicios salen agrupados por *job* con su contador: `auth (1/1 up)`,
`gateway (3/3 up)`… y una etiqueta verde **UP** por fila. Si algo falla, la columna
**Error** dice por qué (por ejemplo `connection refused`).

En **Alerts** hay tres contadores arriba: **Inactive** (verde, todo bien), **Pending**
(ámbar: la condición se cumple pero aún no ha pasado el `for:`) y **Firing** (rojo,
disparada). Debajo salen agrupadas por bloque de reglas: *disponibilidad*, *rendimiento*
y *slo*.

Con el sistema sano el estado es **`Inactive (13) · Pending (0) · Firing (0)`** — las 13
reglas definidas, ninguna activa. Son:

- *disponibilidad*: `ServicioCaido`, `ConsumerKafkaCaido`, `RedisSinNodos`,
  `RedisNodoDegradado`, `MongoCaido`
- *rendimiento*: `TasaErrorAlta`, `LatenciaP95Alta`, `RateLimitDisparado`, `KafkaLagAlto`,
  `MensajesEnDLQ`
- *slo*: `RedisConsultasLentas`, `SLOEventosExitoAgotado`, `SLOGatewayDisponibilidadAgotado`

Haz clic en el nombre de cualquiera para desplegar su expresión y su umbral.

### 6.2 Alertmanager — http://localhost:9093

Sin login. Menú superior: **Alerts · Silences · Status · Settings · Help**, y un botón
**New Silence** arriba a la derecha.

**Qué hace y en qué se diferencia de Prometheus.** Prometheus *evalúa* las reglas y decide
si una alerta se dispara; Alertmanager las *recibe*, las agrupa y las enviaría a un destino
(correo, Slack, ntfy…). Por eso hay dos pantallas de alertas y **muestran cosas distintas**:

| | Prometheus → Alerts | Alertmanager → Alerts |
|---|---|---|
| Muestra | **Todas** las reglas definidas (13), con su estado | **Solo** las que están disparadas ahora |
| Con el sistema sano | 13 en verde `Inactive` | **Vacío** |

#### Paso a paso

1. Abre **http://localhost:9093**. Entra directo a la pestaña **Alerts**.
2. **Con el sistema sano verás un recuadro amarillo que dice `No alert groups found`.**
   ⚠️ Eso **no es un error ni un fallo de configuración**: significa que no hay ninguna
   alerta disparada, que es justo lo que quieres. Si entraste aquí y viste eso y pensaste
   que no funcionaba, era esto.
3. Para verlo **con contenido**, provoca un fallo. Mata una réplica de gateway:
   ```powershell
   (Get-NetTCPConnection -LocalPort 3010 -State Listen).OwningProcess | ForEach-Object { Stop-Process -Id $_ -Force }
   ```
4. Espera entre **30 y 60 segundos** (las reglas tienen un `for:` que evita avisar por un
   parpadeo) y **recarga** la página.
5. Ahora aparecen las alertas agrupadas. Resultado real de esta prueba:
   ```
   ntfy-critico   alertname="ServicioCaido"  job="gateway"                  1 alert
   ntfy-critico   alertname="SLOGatewayDisponibilidadAgotado" job="gateway" 3 alerts
   ```
   - `ntfy-critico` (en azul) es el **receptor** al que se enviarían.
   - Las cajas son las **etiquetas** por las que se agrupó.
   - `1 alert` / `3 alerts` es cuántas hay dentro del grupo.
6. Pulsa el **`+` azul** a la izquierda del grupo (o **Expand all groups**) para desplegar
   cada alerta y ver sus etiquetas y anotaciones completas.
7. **Restaura el gateway** relanzando `pwsh ./levantar-todo.ps1 -Gateways 3`. La alerta
   desaparece sola en cuanto Prometheus vuelve a ver el servicio.

#### Silences: callar una alerta a propósito

Si vas a hacer mantenimiento y no quieres ruido:

1. Botón **New Silence** (arriba a la derecha).
2. Rellena **Start**/**End** (o una duración), y un **matcher** — por ejemplo
   `alertname="ServicioCaido"`.
3. Pon tu nombre en **Creator** y un motivo en **Comment** (son obligatorios).
4. **Create**. La alerta deja de notificarse hasta que expire; la verás en la pestaña
   **Silences**.

#### Status

Muestra la configuración cargada y los receptores definidos. Útil para comprobar a dónde
irían las notificaciones.

### 6.3 Jaeger — http://localhost:16686

Trazas distribuidas: el recorrido de una petición por los servicios, con su duración.

⚠️ **Sale vacío salvo que actives el trazado a propósito.** Es *opt-in* para no pagar su
coste en desarrollo: sin la variable de entorno, el código de trazado no hace nada. Si abres
Jaeger y el desplegable **Service** aparece **sin ninguna opción**, **no está roto** — es que
ningún servicio le está enviando trazas. Comprobado en esta máquina: la API
`http://localhost:16686/api/services` devuelve `null`, es decir, cero servicios registrados.

#### Paso a paso para activarlo

1. Para el servicio que quieras trazar (por ejemplo `game`, en el puerto 9102):
   ```powershell
   (Get-NetTCPConnection -LocalPort 9102 -State Listen).OwningProcess | ForEach-Object { Stop-Process -Id $_ -Force }
   ```
2. Relánzalo con el trazado encendido:
   ```powershell
   cd E:\ARSW\Proyecto\battlecaos-game
   $env:OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
   $env:SERVICE_NAME="game"; $env:OBS_PORT="9102"; $env:REDIS_URL="redis://localhost:6379"
   node --import ./src/tracing.js src/index.js
   ```
3. **Genera tráfico** (`node tools/generar-trafico.mjs`) — sin tráfico no hay trazas.
4. En Jaeger: desplegable **Service** → elige **`game`** → botón azul **Find Traces**.
5. Sale una lista de trazas. Haz clic en una para ver la cascada de *spans*: cada barra es
   una operación, y su ancho es lo que tardó.

#### Alternativa sin Jaeger (más simple)

El servicio `observability` reconstruye la línea de tiempo de un evento por su
correlationId, sin necesidad de activar nada:

```powershell
Invoke-RestMethod "http://localhost:9106/trace/<pega-aqui-el-cid>"
```

### 6.4 Kafka UI — http://localhost:8080

**No se levanta por defecto.** Si abres esa URL sin arrancarlo, el navegador dirá que no
puede conectar — es normal. Para arrancarlo:

```powershell
docker compose -f docker-compose.yml up -d kafka-ui
```

Tarda ~30 s en estar listo. Luego:

1. Abre **http://localhost:8080** → verás el clúster **`local`**.
2. Menú lateral → **Topics**. Están los del juego: `cmd.room`, `cmd.game`, `cmd.chat`,
   `evt.room`, `evt.game`, `evt.timer`, `gw.broadcast` y `dlq`.
3. Entra en un topic → pestaña **Messages** para ver los mensajes reales que circulan
   (el JSON con `type`, `correlationId` y `data`). Es la forma de comprobar con tus ojos
   que el gateway publica y los servicios consumen.
4. Menú lateral → **Consumers** para ver los consumer groups y su retraso. Es la vista cruda
   de lo que el panel *Kafka consumer lag* resume.

> **`dlq` es el topic importante.** Ahí acaban los mensajes que fallaron al procesarse. Si
> tiene mensajes, algo se rompió: mira su contenido para saber qué evento fue. Con el
> sistema sano debe estar vacío (y hay una alerta, `MensajesEnDLQ`, que avisa).

### 6.5 Loki — http://localhost:3100

**No tiene interfaz propia.** Es solo una API: si abres esa URL en el navegador verás un
**404** y es lo esperado, no un fallo. Los logs se consultan desde Grafana (§5.6).

Si quieres comprobar que está vivo:

```powershell
Invoke-RestMethod "http://localhost:3100/ready"          # responde: ready
(Invoke-RestMethod "http://localhost:3100/loki/api/v1/label/name/values").data   # lista los servicios
```

> Recién arrancado, `/ready` puede responder `Ingester not ready: waiting for 15s after
> being ready`. Es normal: espera unos segundos y vuelve a probar.

Lo segundo debe devolver los 9 servicios: `auth, bot, chat, game, gateway, observability,
room, timer, voice-channel`. Si devuelve vacío, promtail no está enviando nada.

---

## 7. Pruebas de disponibilidad

Cinco experimentos. Todos los resultados están **medidos**, no supuestos.

### ⚠️ Antes de nada: dónde se escriben estos comandos

Al levantar el entorno se te abrieron **11 ventanas de PowerShell**, una por servicio
(`auth`, `gateway:3000`, `room`…). **En esas NO se escribe nada**: son la salida de cada
servicio y deben quedarse ahí corriendo. Si escribes en ellas, no pasará nada útil; si las
cierras, matas ese servicio.

**Abre UNA ventana nueva y aparte, solo para las pruebas:**

1. Menú Inicio → escribe **PowerShell** → abre **PowerShell 7** (o Windows Terminal).
2. Sitúate en la carpeta del proyecto:
   ```powershell
   cd E:\ARSW\Proyecto
   ```
3. Toda la sección 7 se escribe **en esa ventana**.

> **Todos los comandos de esta sección son de PowerShell**, no de bash. Están pensados para
> copiarlos y pegarlos tal cual en esa ventana. (Si en otras secciones ves comandos que
> empiezan por `for i in $(seq ...)`, eso es sintaxis de Linux y **no funciona aquí**.)

**Comprobaciones previas:**

1. Parte de un estado limpio:
   ```powershell
   pwsh ./tools/ver-metricas.ps1
   ```
   Debe decir **`12/12 UP`** y **`alertas activas 0 (limpio)`**. Si hay alertas de una
   prueba anterior, espera ~5 minutos a que se cierren solas.
2. Ten **Grafana abierto en el navegador** (rango `Last 15 minutes`, refresco `5s`) para ver
   el efecto en vivo mientras lanzas los comandos.
3. Haz las pruebas **en orden**: la 7.3 continúa donde acaba la 7.2.

---

### 7.1 Balanceo entre los 3 gateways

**Qué demuestra:** que el balanceador reparte de verdad y no manda todo a una sola réplica.

**Paso 1.** En tu ventana de pruebas, pega **esta única línea** y pulsa Enter:

```powershell
1..30 | ForEach-Object { (Invoke-RestMethod "http://localhost:8090/health").instance } | Group-Object | Select-Object Count, Name
```

Qué hace, por partes: `1..30` repite 30 veces · `Invoke-RestMethod` pregunta al balanceador
(`:8090`) · `.instance` se queda con el nombre de la réplica que respondió · `Group-Object`
cuenta cuántas veces respondió cada una.

**Paso 2.** Compara con el resultado medido:

```
Count Name
----- ----
   10 gw1
    9 gw2
   11 gw3
```

**Cómo saber si va bien:** aparecen **las tres** con un reparto más o menos parejo.

**Si algo falla:**

| Lo que ves | Qué significa |
|---|---|
| Solo aparece **una** réplica | El balanceo no funciona (revisa §10.2 y `.env.local`) |
| Error *No se puede conectar con el servidor remoto* | El balanceador no está arriba: `docker compose -f deploy/docker-compose.lb.yml up -d` |
| Salen menos de 30 en total | Alguna petición falló; repite y mira si es constante |

**Paso 3 (opcional, visual).** En Grafana, panel *Sockets activos por instancia de gateway
(balanceo)*: con jugadores conectados deben moverse las tres líneas.

---

### 7.2 Failover: matar un gateway sin tumbar el servicio

**Qué demuestra:** que perder una réplica **no** corta el servicio.

**Paso 1.** Mata la réplica `gw2` (la que escucha en el puerto 3010):

```powershell
(Get-NetTCPConnection -LocalPort 3010 -State Listen).OwningProcess |
  ForEach-Object { Stop-Process -Id $_ -Force }
```

Verás cerrarse la ventana titulada `gateway:3010`. Eso es lo esperado.

**Paso 2.** *Inmediatamente*, en la misma ventana de pruebas, lanza 25 peticiones:

```powershell
1..25 | ForEach-Object { (Invoke-RestMethod "http://localhost:8090/health").instance } | Group-Object | Select-Object Count, Name
```

**Resultado medido:**

```
Count Name
----- ----
   13 gw1
   12 gw3
```

**25 de 25 atendidas, 0 errores**, y `gw2` ya no aparece. Ni una sola petición se perdió.
Lo consiguen dos ajustes del nginx: `max_fails=3 fail_timeout=15s` (saca la instancia muerta
del reparto) y `proxy_next_upstream` (reintenta en otra si una muere a mitad de petición).

> Si sale algún error suelto en las primeras peticiones, es normal: nginx necesita unos
> intentos fallidos (`max_fails=3`) para darse cuenta de que esa réplica murió. Repite el
> comando y ya deberían ir todas a las dos supervivientes.

**Paso 3.** Espera ~20 segundos y mira qué detectó Prometheus:

```powershell
pwsh ./tools/ver-metricas.ps1
```

Debe decir **`11/12 UP`** y listar `ServicioCaido` en alertas activas.

**Resultado medido a los 20 s:**

| Dónde | Qué se ve |
|---|---|
| Prometheus → Status → Targets | `host.docker.internal:3010` en rojo **DOWN** |
| Grafana, panel *Servicios arriba* | baja de **12** a **11** |
| Prometheus → Alerts | `ServicioCaido` en **FIRING** |
| Alertmanager → Alerts | aparece el grupo `ntfy-critico / ServicioCaido` |

**Paso 4.** Revive **solo esa réplica** (no hace falta reiniciar los 11 servicios):

```powershell
pwsh ./tools/levantar-gateway.ps1
```

Se abrirá una ventana nueva `gateway:3010`. Déjala abierta.

> **Si vas a hacer también la prueba §7.3, NO ejecutes esto todavía.** La 7.3 continúa con el
> gateway caído y te dice cuándo revivirlo. Salta directamente a §7.3.

---

### 7.3 El SLO se consume y se recupera solo

**Qué demuestra:** que el sistema mide su propio cumplimiento y se recupera sin intervención.
Es la prueba más vistosa para una sustentación.

⚠️ **Esta prueba CONTINÚA la §7.2: empieza con `gw2` todavía muerto.** Si ya lo reviviste,
vuelve a matarlo con el comando del paso 1 de §7.2 y sigue aquí.

**La idea en una frase:** rompiste algo (§7.2), ahora vas a ver cómo el sistema *cuantifica*
ese daño contra un objetivo declarado (pasos 1-3), lo reparas (paso 4) y compruebas que la
alerta se cierra **sola** (paso 5).

Un poco de vocabulario, que si no los pasos no se entienden:

- **SLO** = el objetivo que te comprometes a cumplir. Aquí: *el gateway responde el 99,5% del
  tiempo*.
- **SLI** = la medición real de si lo estás cumpliendo. Aquí: qué fracción de las
  comprobaciones de los últimos 5 minutos salieron bien.
- **Error budget** = cuánto margen de fallo te queda. 100% = intacto; 0% = agotaste el
  0,5% de caída que te permitía el objetivo; negativo = te pasaste.

**Paso 1.** Mira el **SLI** (fracción de comprobaciones correctas en los últimos 5 min):

```powershell
(Invoke-RestMethod "http://localhost:9090/api/v1/query" -Body @{ query = 'sli:gateway_disponibilidad:ratio5m' }).data.result | ForEach-Object { "{0,-32} {1:P1}" -f $_.metric.instance, [double]$_.value[1] }
```

**Medido al minuto de la caída:** `gw1` y `gw3` al **100%**, `gw2` al **93,3%**.

**Paso 2.** Mira el **error budget** (100% = presupuesto intacto):

```powershell
(Invoke-RestMethod "http://localhost:9090/api/v1/query" -Body @{ query = 'slo:gateway_disponibilidad:budget_restante' }).data.result | ForEach-Object { "{0,-32} {1:P0}" -f $_.metric.instance, [double]$_.value[1] }
```

**Medido:** `gw2` cae a **−1233%**. El objetivo es 99,5% de disponibilidad, o sea un
presupuesto de solo 0,5%: un 6,7% de caída se lo come doce veces.

> ⚠️ **El budget tarda ~30 s más que el SLI en moverse.** El SLI se recalcula cada 15 s y el
> budget cada 30 s, así que es normal ver el SLI ya caído y el budget todavía en 100%. No es
> un error: espera medio minuto.

**Paso 3.** Mira las alertas. Verás **dos estados distintos**, y la diferencia importa:

```powershell
(Invoke-RestMethod "http://localhost:9090/api/v1/alerts").data.alerts | ForEach-Object { "{0,-10} {1}" -f $_.state, $_.labels.alertname }
```

**Medido:**

```
pending    SLOGatewayDisponibilidadAgotado
firing     ServicioCaido
```

- **PENDING** = la condición ya se cumple, pero aún no ha pasado el tiempo mínimo (`for:`)
  que exige la regla. Sirve para no avisar por un parpadeo de dos segundos.
- **FIRING** = confirmada; es la que llega a Alertmanager.

**Paso 4 — vuelve a levantar la réplica que mataste.**

Esta es la parte que cierra el experimento: has provocado el daño, ahora reparas la causa y
observas si el sistema se recupera **solo**, sin que tú toques nada más.

Escribe esto (una línea):

```powershell
pwsh ./tools/levantar-gateway.ps1
```

Eso levanta `gw2` en el puerto 3010, que es justo la réplica que mataste en §7.2. Se abrirá
una ventana nueva titulada `gateway:3010`; **déjala abierta**, es el servicio corriendo.

> **¿Por qué este script y no `levantar-todo.ps1`?** Porque `levantar-todo.ps1` reinicia los
> **11** servicios, incluidas las otras dos réplicas de gateway que ahora mismo están sanas.
> Eso las tumbaría un momento, dañaría también su SLO y ensuciaría la medición. Aquí solo
> quieres reparar lo que rompiste.
>
> Si mataste otra réplica, indícale el puerto:
> `pwsh ./tools/levantar-gateway.ps1 -Puerto 3020`

Comprueba a los ~15 segundos que volvió:

```powershell
(Invoke-RestMethod "http://localhost:3010/health").instance
```

Debe responder `gw2`. (Si el script dice *"el puerto 3010 ya está en uso"*, es que esa
réplica ya estaba viva: no hace falta hacer nada.)

**Paso 5 — ahora NO toques nada y observa.** Aquí está lo que demuestra la prueba: no hay
que reparar el SLO a mano, se repara solo cuando el servicio vuelve a estar sano.

Cada minuto, lanza:

```powershell
pwsh ./tools/ver-metricas.ps1
```

Verás esta secuencia:

| Momento | `alertas activas` |
|---|---|
| Justo tras revivir `gw2` | 1 (`SLOGatewayDisponibilidadAgotado`) — el budget aún recuerda la caída |
| A los ~3 min | sigue 1, pero el budget ya sube |
| A los ~5-6 min | **0 (limpio)** |

**Resultado medido: todo se cerró solo en 337 segundos (5 min 37 s)** y el budget volvió a
**100%** en las tres réplicas. Encaja con la ventana móvil de 5 minutos del SLI más el
retardo de evaluación.

Para verlo en números, repite el comando del paso 2: las tres réplicas deben acabar en
`100 %`.

**La conclusión que hay que contar:** el sistema detectó la caída solo, la cuantificó contra
un objetivo declarado (99,5%), avisó, y cerró la alerta solo al recuperarse. Nadie tuvo que
tocar el monitoreo.

---

### 7.4 Caída de Redis (prueba de caos)

**Qué demuestra:** que una caída del almacén de estado **degrada** el sistema pero no lo
destruye, y que la observabilidad sigue viéndose durante el incidente.

**Paso 1 — averigua qué contenedor de Redis estás usando.** ⚠️ Puede haber **dos**, y parar
el que no es no hace absolutamente nada (me pasó a mí montando esto):

```powershell
docker ps --filter "name=redis" --format "{{.Names}}`t{{.Status}}"
```

- `battlecaos-redis` → el del `docker-compose.yml` de la raíz.
- `battlecaos-local-redis` → el que crea `levantar-todo.ps1` **si el otro no estaba
  levantado**.

**El que salga en esa lista es el que está en uso.** Apunta su nombre; lo llamaremos
`<REDIS>` en los pasos siguientes.

**Paso 2 — tíralo:**

```powershell
docker stop battlecaos-local-redis
```

(Cambia el nombre si el tuyo era `battlecaos-redis`.)

**Paso 3 — espera ~20 segundos y comprueba que los servicios SIGUEN VIVOS.** Pega este
bloque completo:

```powershell
@{gateway=3000; auth=3001; room=9101; game=9102; chat=9103; timer=9104; bot=9105; observability=9106; 'voice-channel'=9107}.GetEnumerator() | Sort-Object Value | ForEach-Object {
  $estado = try { (Invoke-WebRequest "http://localhost:$($_.Value)/health" -TimeoutSec 5).StatusCode } catch { 'SIN RESPUESTA' }
  "{0,-15} {1}" -f $_.Key, $estado
}
```

**Resultado medido — los 9 siguieron respondiendo `200`:**

```
gateway         200
auth            200
room            200
game            200
chat            200
timer           200
bot             200
observability   200
voice-channel   200
```

**Lo importante es que ninguno diga `SIN RESPUESTA`.** Si alguno se cayó, ese servicio no
tolera la pérdida de Redis (es exactamente el fallo que se encontró en `timer`, punto 10 del
anexo).

**Paso 4 — comprueba que la observabilidad NO se queda ciega.** Esto es lo que falla en
muchos sistemas y es lo más valioso de esta prueba:

```powershell
(Invoke-WebRequest "http://localhost:9101/metrics" -TimeoutSec 8).Content -split "`n" | Select-String "^redis_node_up"
```

**Resultado medido:** `redis_node_up{node="primary"} 0`.

Fíjate en el matiz: la métrica dice **0 = caído**, no desaparece. Y `/metrics` responde en
**~1,5 s**, no se cuelga. Si en tu caso los targets se van a `DOWN` y el panel se queda en
blanco, es que estás con una versión anterior al arreglo (ver el punto 10 del anexo).

En Grafana, panel *Salud de Redis por servicio*: la franja pasa a rojo.
En Prometheus → Alerts: saltan `RedisNodoDegradado` y `RedisSinNodos`.

**Paso 5 — recupera:**

```powershell
docker start battlecaos-local-redis
```

**Resultado medido:** en ~15 s `redis_node_up` vuelve a **1** (repite el comando del paso 4
para verlo), y en el log del `timer` aparece `redis respondiendo de nuevo — volviendo a
competir por el lease`. Las alertas se cierran solas.

Para comprobar el mensaje de recuperación en el log:

```powershell
Get-Content .\logs\timer.log -Tail 5 | ForEach-Object { ($_ | ConvertFrom-Json).msg }
```

> **Ojo con reiniciar durante la caída.** Con Redis caído, un servicio que arranca de cero
> **no puede conectar y sale** (el `await redis.connect()` inicial falla). Es correcto —en
> Azure el orquestador lo reintenta hasta que Redis vuelve— pero significa que **no puedes
> relanzar servicios mientras Redis esté parado**. Primero levanta Redis, luego los servicios.

**Variante: modo resiliente (sin perder partidas).** Con un Redis de respaldo, la caída del
primario no interrumpe el juego:

```powershell
pwsh ./levantar-todo.ps1 -Gateways 3 -Resiliente
```

Ahí sí aparece la serie `secondary` en el panel de Redis, y matar el primario **no** corta la
partida: las escrituras iban a ambos nodos.

---

### 7.5 Backup de la base de datos (DR)

**Qué demuestra:** que hay copia de seguridad automática de los datos durables (cuentas e
historial), con un RPO de 24 h.

Mongo vive en Atlas, así que el respaldo no es local: lo hace un workflow de GitHub Actions
en el repositorio `battlecaos-infra`.

> Esta prueba usa la herramienta `gh` (GitHub CLI). Comprueba que la tienes con
> `gh auth status`; si no, puedes hacerlo todo desde la web (se indica en cada paso).

**Paso 0 — cámbiate al repositorio de infraestructura.** Los comandos `gh` actúan sobre el
repositorio de la carpeta en la que estés, así que esto es imprescindible:

```powershell
cd E:\ARSW\Proyecto\battlecaos-infra
```

**Paso 1 — comprueba que el workflow existe y está activo:**

```powershell
gh workflow list
```

Debe aparecer `Backup diario de Mongo (DR)   active`.
*En la web:* pestaña **Actions**, en la lista de la izquierda.

**Paso 2 — comprueba que el secreto está configurado:**

```powershell
gh secret list
```

Debe aparecer `MONGO_URL`. Sin él, el workflow falla a propósito con un mensaje claro.
*En la web:* **Settings → Secrets and variables → Actions**.

**Paso 3 — mira la última ejecución:**

```powershell
gh run list --workflow="Backup diario de Mongo (DR)" --limit 5
```

**Resultado medido:**

```
completed  success  Backup diario de Mongo (DR)  main  schedule  29731223667  11s  2026-07-20T09:22:57Z
```

Fíjate en `success` y en `schedule` (se disparó solo, por horario). **Copia el número largo
(`29731223667`): es el ID de la ejecución**, y lo necesitas en los pasos siguientes.

**Paso 4 — comprueba que el backup tiene contenido de verdad.** Que el workflow diga
`success` no basta: podría estar copiando una base vacía.

```powershell
gh run view 29731223667 --log | Select-String "done dumping"
```

(Sustituye el número por el ID de tu ejecución.)

**Resultado medido:**

```
done dumping battlecaos.usuarios (27 documents)
done dumping battlecaos.partidas (36 documents)
```

**Eso es la prueba real:** 27 cuentas y 36 partidas respaldadas.

**Paso 5 — comprueba el artifact y su retención:**

```powershell
gh api repos/:owner/:repo/actions/runs/29731223667/artifacts --jq '.artifacts[] | "\(.name) \(.size_in_bytes) bytes expira \(.expires_at[:10])"'
```

**Resultado medido:** `mongo-backup-29731223667 5809 bytes expira 2026-08-19` — 30 días de
retención, como exige el RPO de 24 h.

**Paso 6 — lanzarlo a mano** (no hay que esperar a las 06:00 UTC):

```powershell
gh workflow run "Backup diario de Mongo (DR)"
```

*En la web:* **Actions → "Backup diario de Mongo (DR)" → botón `Run workflow`**.

Espera ~30 s y vuelve al paso 3 para ver la ejecución nueva.

**Paso 7 — descargar un backup** (para probar la restauración):

```powershell
gh run download 29731223667
```

Te deja la carpeta `mongo-backup-.../battlecaos/` con los `.bson`. El procedimiento completo
de restauración con `mongorestore` está en `deploy/DR.md`.

**Al terminar, vuelve a la carpeta del proyecto** para el resto de comandos:

```powershell
cd E:\ARSW\Proyecto
```

El procedimiento completo de restauración con `mongorestore` está en `deploy/DR.md`.

---

## 8. Estado final esperado

Con todo levantado y sin fallos inducidos:

| Comprobación | Valor |
|---|---|
| Contenedores | 11 |
| Endpoints `/health` | 11 en 200 |
| Targets de Prometheus | 12/12 UP |
| **Alertas activas** | **0** |
| Error budget (ambos SLO) | 100% |
| Servicios en Loki | 9 + 3 instancias de gateway |
| Grupos en consumer lag | 10, todos en 0 |

**Si ves alertas activas sin haber provocado ningún fallo, algo va mal.** Un panel que
grita siempre no sirve: entre el ruido no se distingue una caída real.

---

## 9. Bajar todo

```powershell
pwsh ./bajar-todo.ps1
docker compose -f deploy/docker-compose.lb.yml down
docker compose -f deploy/monitoring/docker-compose.monitoring.yml down
docker compose -f docker-compose.yml down
```

---

## 10. ¿Solo se puede probar desde este PC?

No. "Local" quiere decir que los servicios **corren** en tu PC, no que solo se puedan
**mirar** desde él. Hay tres alcances distintos:

| Desde dónde | ¿Funciona? | Qué hace falta |
|---|---|---|
| El mismo PC | Sí | Nada, `localhost` |
| Otro equipo en la **misma red Wi-Fi** | Sí | Usar la IP del PC + §10.2 para jugar |
| Desde **internet** (otra red) | No directamente | Un túnel (§10.3) o el despliegue en Azure |

### 10.1 Desde otro equipo de la misma red

Averigua la IP de tu PC:

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object { $_.IPAddress -notmatch '^(127\.|169\.254\.|172\.)' } |
  Select-Object IPAddress, InterfaceAlias
```

En esta máquina salió **`192.168.1.11`** (interfaz Wi-Fi). Sustituye esa IP por la tuya —
**cambia cada vez que te reconectas a la red**, no la des por fija.

Desde el otro equipo, en el navegador:

| | URL |
|---|---|
| Grafana | `http://192.168.1.11:3030` |
| Prometheus | `http://192.168.1.11:9090` |
| Alertmanager | `http://192.168.1.11:9093` |
| Jaeger | `http://192.168.1.11:16686` |

Esto funciona sin tocar nada porque los contenedores publican sus puertos en **todas** las
interfaces (`0.0.0.0`), no solo en loopback.

**Si no carga, es el firewall de Windows.** Comprueba en qué categoría está tu red y si hay
regla que permita el tráfico:

```powershell
Get-NetConnectionProfile | Select-Object Name, NetworkCategory
```

En esta máquina la Wi-Fi está como **Public**, y tanto *Docker Desktop Backend* como
*Node.js JavaScript Runtime* tienen regla de entrada permitida en ese perfil — por eso
funciona. Si tu red está como **Private** y Docker solo tiene regla en Public (o al revés),
los puertos de los contenedores quedarán bloqueados aunque el servicio esté vivo. Para
verlo:

```powershell
Get-NetFirewallRule -Direction Inbound -Enabled True -Action Allow |
  ForEach-Object { $r=$_; $a=$_ | Get-NetFirewallApplicationFilter
    if ($a.Program -match 'docker|node\.exe') { [PSCustomObject]@{ Regla=$r.DisplayName; Perfil=$r.Profile } } } |
  Format-Table -AutoSize
```

> Ojo con el diagnóstico: hacer `curl http://192.168.1.11:3030` **desde el propio PC** no
> prueba nada, porque ese tráfico no atraviesa el firewall. Hay que probarlo desde el otro
> equipo.

### 10.2 Jugar desde otro equipo (dos cosas que cambiar)

Ver los paneles funciona solo con la IP, pero **jugar** necesita dos ajustes más, porque el
frontend está configurado para `localhost` y un navegador en otro equipo resolvería
`localhost` como *sí mismo*:

**1. Que Vite escuche en la red.** Por defecto solo atiende en `localhost`. Usa el script
añadido para esto:

```powershell
cd battlecaos-frontend
npm run dev:lan          # equivale a: vite --host
```

Al arrancar imprime la dirección de red (`Network: http://192.168.1.11:5173/`).

**2. Que el frontend apunte a la IP, no a `localhost`.** En
`battlecaos-frontend/.env.local` (recuerda: este gana sobre `.env`):

```
VITE_GATEWAY_URL=http://192.168.1.11:8090
VITE_AUTH_URL=http://192.168.1.11:3001
```

Para y relanza `npm run dev:lan` después de cambiarlo — Vite lee los `.env` solo al arrancar.

> Si usas login de Google, la nueva URL (`http://192.168.1.11:5173`) tiene que estar dada de
> alta como origen autorizado en la consola de Google Cloud, o el login fallará. Para una
> prueba rápida es más cómodo usar cuentas locales (email + contraseña).

### 10.3 Desde fuera de la red

Los puertos de tu PC no son accesibles desde internet (estás detrás del router). Dos
caminos:

- **Túnel temporal** — `cloudflared tunnel --url http://localhost:3030` o `ngrok http 3030`
  te dan una URL pública provisional. Útil para enseñarle el panel a alguien un rato.
  ⚠️ Eso **expone tu Grafana a internet**: cambia la contraseña `admin/battlecaos` antes, y
  cierra el túnel al terminar.
- **El despliegue en Azure** — accesible desde cualquier sitio, pero con la observabilidad
  limitada a los logs (§11).

---

## 11. ¿Y en la versión desplegada en Azure?

| | Local | Desplegado |
|---|---|---|
| **Logs** | Loki + Grafana | ✅ **Sí, ya funciona** — Azure Log Analytics |
| **Métricas** | Prometheus | ⚠️ Los `/metrics` existen, pero nadie los recoge |
| **Dashboards / Alertas** | Grafana + Alertmanager | ❌ No desplegado |

### Logs desplegados: ya los tienes

Terraform crea un `azurerm_log_analytics_workspace` unido al Container App Environment, así
que todo lo que los servicios escriben queda guardado 30 días. Desde el portal:
**Container App → Monitoring → Logs**. Por CLI:

```powershell
az extension add -n log-analytics     # solo la primera vez

# Guarda el id del workspace en una variable (en PowerShell NO se usa WS=$(...) )
$WS = az monitor log-analytics workspace list -g battlecaosdev-rg --query "[0].customerId" -o tsv
```

```powershell
# Volumen de log por app en las últimas 6 h
az monitor log-analytics query -w $WS --analytics-query "ContainerAppConsoleLogs_CL | where TimeGenerated > ago(6h) | summarize lineas=count() by ContainerAppName_s | order by lineas desc | take 10" -o table
```

```powershell
# Seguir un correlationId por TODOS los servicios (equivalente desplegado de §5.7)
az monitor log-analytics query -w $WS --analytics-query "ContainerAppConsoleLogs_CL | where Log_s contains 'PEGA-AQUI-EL-CID' | project TimeGenerated, ContainerAppName_s, Log_s | order by TimeGenerated asc" -o table
```

**Resultado medido** con el primer comando:

```
ContainerAppName_s           Lineas
---------------------------  ------
battlecaosdev-timer           20712
battlecaosdev-game              600
battlecaosdev-gateway           577
battlecaosdev-redis             360
battlecaosdev-observability     360
```

Funciona igual de bien que Loki porque los servicios logean JSON estructurado con el `cid`;
solo cambia el lenguaje de consulta (KQL en vez de LogQL).

### Métricas desplegadas: lo que faltaría

Los servicios ya exponen `/metrics` en el clúster, pero nada los scrapea, y **no se pueden
scrapear desde tu máquina** porque su ingress es interno — justamente el hallazgo `S6382`
que se aceptó en SonarCloud, y está bien que siga así.

La opción realista es **Grafana Cloud (capa gratuita)**: en vez de que Prometheus entre a
buscar las métricas, un agente dentro del clúster las **empuja** hacia afuera. Encaja con
Container Apps porque no hay que exponer nada, y el plan gratuito (10k series, 50 GB de
logs) sobra. Requiere desplegar un Container App con Grafana Alloy y configurar
`remote_write`.

Descartadas: desplegar Prometheus+Grafana como Container Apps exige almacenamiento
persistente y sube la factura; Azure Managed Grafana es de pago.

**Esto no está hecho** — es un trabajo aparte.

---

## Anexo — Fallos reales encontrados al montar esto

Por si reaparecen, y como registro de por qué el stack no funcionaba antes:

1. **Loki estaba siempre vacío.** promtail solo leía contenedores Docker, pero los 9
   servicios corren como procesos `node` nativos. Sus logs morían en las ventanas de
   PowerShell. Solución: cada servicio escribe además una copia JSON en `logs/` que
   promtail sigue.
2. **18 alertas falsas permanentes.** `mongo_up 0` en servicios que no usan Mongo y
   `redis_node_up{secondary} 0` donde no hay respaldo. El código usaba `null` para "no
   aplica" pero lo publicaba como `0` = *caído*. Solución: omitir la serie. Ojo, un gauge
   sin etiquetas publica `0` aunque nunca se le asigne valor — hay que borrarlo explícitamente.
3. **Métricas de salud congeladas.** Se recalculaban solo al llamar a `/health`, pero
   Prometheus scrapea `/metrics`. En Azure colaba de casualidad porque la liveness probe
   llama a `/health` cada pocos segundos. Solución: refrescar también en `/metrics`.
4. **Consumer lag siempre vacío.** El exporter apuntaba a `host.docker.internal:9092`;
   Kafka le respondía con su listener anunciado `localhost:9092`, que dentro del contenedor
   del exporter es él mismo → `connection refused`. Solución: usar el listener interno
   `kafka:29092` y unir el exporter a la red de Kafka.
5. **`ip_hash` en el balanceador.** Detrás del NAT de Docker todas las conexiones llegan con
   la misma IP de origen, así que mandaba todo a un solo gateway. Solución: `least_conn`,
   viable porque el frontend negocia WebSocket de entrada (una conexión persistente, sin
   handshake de long-polling que deba volver a la misma instancia).
6. **`.env.local` pisando a `.env` en el frontend.** Cambiar `VITE_GATEWAY_URL` en `.env` no
   surtía efecto porque existía un `.env.local` con el valor viejo, y en Vite ese gana. El
   frontend seguía conectándose a una sola réplica: todo *parecía* funcionar, pero el
   balanceador nunca entraba en juego. Solución: la URL vive en `.env.local`, y ambos
   archivos llevan un comentario explicando la precedencia.
7. **`node` no reconocido al lanzar los servicios.** `Start-Process` hereda el entorno de la
   terminal que lanza el script, no el PATH del registro; una terminal abierta antes de
   instalar Node arrastra el PATH viejo y las 11 ventanas mueren al instante. Solución:
   `levantar-todo.ps1` resuelve la ruta completa a `node.exe` y la usa explícitamente.
8. **Retraso fantasma en el panel de Kafka.** El `timer` nombraba su consumer group con su
   PID (`timer-timer-22096`), así que **cada reinicio abandonaba un grupo** en Kafka. Los
   grupos huérfanos quedan registrados con su lag congelado, de modo que el panel *Kafka
   consumer lag* iba acumulando retrasos de procesos muertos y dejaba de servir para
   detectar un atasco real (se midió un lag de 32 sin que nada estuviera mal). Solución: usar
   un id **estable** por instancia (`INSTANCE_ID`, igual que el gateway) para el grupo, y
   dejar el PID solo en el lease de leader election, donde sí hace falta distinguir procesos.
   Si arrastras grupos huérfanos de antes, se borran así:
   ```bash
   docker exec battlecaos-kafka kafka-consumer-groups --bootstrap-server localhost:9092 --list
   docker exec battlecaos-kafka kafka-consumer-groups --bootstrap-server localhost:9092 --delete --group timer-timer-22096
   ```
9. **La observabilidad se quedaba CIEGA justo al caer Redis.** Encontrado haciendo la prueba
   de caos de §7.4: al parar Redis, **6 de los 12 targets de Prometheus pasaron a DOWN** con
   `context deadline exceeded`. La causa: `/metrics` y `/health` hacen un PING a Redis, y un
   PING contra un Redis caído no falla rápido —ioredis encola el comando y espera—, así que
   el endpoint se colgaba. Resultado: en el momento en que más falta hace ver el panel, el
   panel desaparecía en vez de mostrar "Redis caído". Solución: un límite de 1,5 s en el
   chequeo; ahora el endpoint **siempre** responde y publica `redis_node_up 0`.
10. **El servicio `timer` MORÍA al caer Redis.** Mismo experimento. El bucle de *leader
    election* es un `setInterval(async …)` **sin `try/catch`**: cuando Redis dejaba de
    responder, la promesa se rechazaba sin capturar y Node mataba el proceso (comportamiento
    por defecto desde Node 15). Se perdía el servicio entero —y con él los turnos de todas
    las partidas— por un fallo del que debería haberse recuperado. Solución: capturar el
    error, ceder el rol de leader (no se puede renovar el lease) y seguir reintentando, con
    un aviso **una sola vez** por caída para no inundar el log a 2 líneas por segundo.
    Verificado: ahora sobrevive y al volver Redis registra `redis respondiendo de nuevo —
    volviendo a competir por el lease` y recupera el liderazgo solo.
11. **Los logs no decían quién era el jugador.** `room` registraba solo el id
   (`jugador ana-real unido a sala…`). Con cuentas de Google ese id es un número largo, así
   que al revisar una partida era imposible saber quién hizo qué. El frontend ya enviaba el
   apodo y `room` ya lo guardaba: solo faltaba escribirlo. Ahora se registra
   `jugador "Capitana Ana" (ana-real)`. Como el apodo lo escribe el usuario, se añadió
   también al logger común el saneador anti **inyección de logs** que ya tenía `auth`
   (Sonar S5145): reemplaza saltos de línea y caracteres de control por espacios, de modo que
   un apodo malicioso no pueda fabricar una línea de log falsa. Verificado intentándolo: un
   apodo con `\n{"level":50,"msg":"LINEA FALSA"}` quedó **dentro** del mensaje legítimo, en
   una sola línea, sin crear ninguna entrada nueva.
