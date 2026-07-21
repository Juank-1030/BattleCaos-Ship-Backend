# Análisis SOLID — BattleCaos-Ship

Revisión de los 9 microservicios contra los cinco principios SOLID, **con evidencia del
código**. Incluye la violación que se encontró y cómo se corrigió.

## Advertencia metodológica

SOLID nace de la orientación a objetos y este backend es **JavaScript modular**: casi todo
son funciones exportadas y hay **una sola `class` real** en todo el proyecto
(`DomainError`, en `game/src/domain/errors.js`). Se evalúa por su espíritu, tomando el
**módulo** como unidad de encapsulamiento — que es como está estructurado el código.

| Principio | Veredicto |
|---|---|
| **S** — Responsabilidad única | ⚠️ Parcial |
| **O** — Abierto/cerrado | ✅ Cumple |
| **L** — Sustitución de Liskov | ✅ Cumple (poca herencia, bien hecha) |
| **I** — Segregación de interfaces | ✅ Cumple |
| **D** — Inversión de dependencias | ✅ Cumple **tras la corrección** (antes ❌) |

---

## S — Responsabilidad única · ⚠️ Parcial

### Lo que está bien

La separación **dominio / orquestación** en `game` es de manual:

- `src/domain/` — 8 módulos **puros**: `board`, `fleet`, `engine`, `powers`, `energy`,
  `salvo`, `countermeasure`, `autoFleet`. No tocan Redis ni Kafka.
- `src/handlers/` — 9 módulos que orquestan: leen de Redis, aplican las reglas, publican.

Esa frontera es la razón de que las reglas se puedan probar sin levantar infraestructura.
`room` sigue el mismo patrón: `domain/room.js` recibe la sala, la muta y la devuelve.

### Lo que no

Los `index.js` acumulan responsabilidades:

| Servicio | Líneas de `index.js` |
|---|---|
| **gateway** | **429** |
| **room** | **330** |
| timer | 193 |
| game | **25** |

`gateway/src/index.js` hace, en un solo archivo: autenticación del handshake, **dos** rate
limits distintos (por IP y por jugador), enrutado de 15 eventos, consumer de Kafka con
backoff, relevo de señalización WebRTC, `/health`, `/metrics`, `/kpis` y la lógica de
reconexión. Son al menos seis responsabilidades.

**El contraste es la mejor prueba:** `game/src/index.js` tiene 25 líneas y solo cablea. El
equipo sabe hacerlo; simplemente no se aplicó en todos los servicios.

> **Recomendación:** extraer de `gateway/index.js` al menos `rateLimit.js`, `sockets.js` y
> `consumer.js`. No es urgente —funciona y está probado— pero es donde más cuesta entrar
> hoy.

---

## O — Abierto/cerrado · ✅ Cumple

Dos ejemplos genuinos de extensión sin modificación:

**`gateway/src/domain/router.js`** — la tabla `ROUTES` mapea evento → topic:

```js
export const ROUTES = {
  'room:create':      'cmd.room',
  'disparo:realizar': 'cmd.game',
  'chat:mensaje':     'cmd.chat',
  ...
};
```

Añadir un evento nuevo es **añadir una entrada**. El bucle que despacha no se toca.

**`room/src/index.js` y `game/src/handlers/broker.js`** — mapa `HANDLERS` de tipo → función.
Mismo patrón: registrar un handler nuevo no modifica el despachador.

### La excepción

`game/src/handlers/handlePower.js` despacha los poderes con condicionales:

```js
if (powerType === 'escudo')   result = applyEscudo(sala, jugador.equipo);
if (powerType === 'tormenta') result = applyTormenta(sala, playerId);
```

Añadir un quinto poder obliga a **modificar** ese archivo. Con `POWER_COSTS` y
`OFFENSIVE_POWERS` ya definidos como tablas en `domain/powers.js`, el paso natural sería un
mapa `poder → función`, igual que `ROUTES`.

---

## L — Sustitución de Liskov · ✅ Cumple

Apenas hay herencia, y la que hay está bien construida:

```js
export class DomainError extends Error {
  constructor(code, message) {
    super(message);
    this.code = code;
    this.name = 'DomainError';
  }
}
```

Llama a `super(message)`, fija `name` y añade `code`. Es sustituible por `Error` en
cualquier `catch` sin sorpresas: `err.message` y `err.stack` funcionan como se espera.

Caso más interesante: el **`Proxy`** de `makeResilient()` en `redis.js`. Imita la interfaz
de ioredis, y los servicios no distinguen entre cliente simple y resiliente — cumple el
contrato del tipo que sustituye.

---

## I — Segregación de interfaces · ✅ Cumple

El ejemplo lo documenta el propio código, en `lock.js`:

> `redisLike` solo necesita: `set(key,val,'NX','PX',ms)` → `'OK'|null`, `get(key)`, `del(key)`

No exige un cliente Redis completo, sino los **tres comandos** que usa. Consecuencia
práctica: se prueba con un objeto simulado de 20 líneas.

`startObservability({ port, redis, mongo = null, traceLookup = null })` va en la misma línea:
`bot` no usa Redis y pasa `null` sin que nada se rompa; `observability` es el único que pasa
`traceLookup`.

---

## D — Inversión de dependencias · ✅ Cumple tras la corrección

### La violación encontrada

**9 archivos de `game` importaban Redis desde la raíz de composición**:

```js
import { redis } from '../index.js';   // broker · helpers · turnFlow · los 6 handleX
```

Eso creaba un **ciclo**: `index.js` → `broker.js` → `index.js`.

Las dos consecuencias, y la segunda es la grave:

1. **Fragilidad.** Funcionaba solo por cómo ESM resuelve los ciclos y porque `redis` quedaba
   asignado antes de llamar a `startBroker()`. Mover esa llamada dos líneas arriba rompía el
   servicio en tiempo de ejecución.
2. **Intestabilidad.** Importar cualquier handler arrastraba el arranque completo del
   servicio: conexión a Redis, productor de Kafka, servidor HTTP. **Por eso las reglas más
   críticas del juego —unas 700 líneas— estaban sin una sola prueba**, y la cobertura de
   `game` no pasaba del 28%.

### La corrección

Se añadió `game/src/contexto.js`, un módulo **hoja** (no importa nada, luego no puede formar
ciclos) que las capas superiores rellenan:

```js
export let redis = null;                          // enlace vivo de ESM
export function configurarContexto(deps = {}) { ... }
export function limpiarContexto() { redis = null; }  // para las pruebas
```

`index.js` lo rellena al arrancar:

```js
const redis = createRedis();
await redis.connect();
configurarContexto({ redis });
```

Y los 9 handlers solo cambian el origen del import — **ni una línea de uso**, porque
`export let` es un enlace vivo:

```js
import { redis } from '../contexto.js';
```

> **Honestidad sobre la solución:** esto es un *contexto de composición*, no inyección por
> constructor. La forma más pura sería pasar `redis` como parámetro a cada handler, pero eso
> tocaba 39 puntos de uso. Lo elegido rompe el ciclo, invierte la dependencia (el detalle se
> **inyecta**, no se importa) y desbloquea las pruebas con un cambio de 9 líneas y riesgo
> mínimo. Es el 90% del beneficio por el 10% del coste.

### Resultado medido

| | Antes | Después |
|---|---|---|
| Ciclos de dependencia en `game` | 1 (index ↔ handlers) | **0** |
| Pruebas de `game` | 194 | **258** |
| Cobertura de `game` (estimada Sonar) | 28,9% | **49,1%** |

Los 9 handlers ya no arrastran el arranque: `turnFlow`, `helpers`, `handlePlaceShips` y
`handleShot` tienen ahora **64 pruebas** que antes eran imposibles de escribir.

Verificado además end-to-end: el servicio arranca, responde en `/health` y procesa eventos
reales de una partida.

### Dónde ya se cumplía

Hay inyección real en los puntos que importan:

| Función | Firma | Qué inyecta |
|---|---|---|
| `makeResilient(primary, secondary, opts)` | `redis.js` | los dos clientes |
| `conLockSala(redisLike, codigo, fn, opts)` | `lock.js` | el cliente, con interfaz mínima |
| `persistirPartida(partidas, msg)` | `persistencia.js` | la colección de Mongo |
| `crearUsuario({ usuarios, redis }, datos)` | `auth/local.js` | ambos almacenes |
| `startObservability({ redis, mongo, traceLookup })` | `observability.js` | dependencias opcionales |

Todas se prueban sin infraestructura, y de hecho así están probadas.

---

## Conclusión

**La arquitectura de dominio está bien pensada.** La frontera entre reglas puras y
orquestación, las tablas de despacho y las interfaces mínimas son decisiones acertadas y
conscientes.

El único incumplimiento serio era el ciclo en `game/handlers`, y **no era un problema
teórico**: era la causa directa de que las reglas más críticas del juego no tuvieran
pruebas. Corregirlo mejoró el diseño **y** la cobertura a la vez, que es la señal de que el
principio estaba señalando algo real.

Queda como deuda menor, sin urgencia:

1. Trocear `gateway/index.js` (429 líneas, ~6 responsabilidades).
2. Convertir el despacho de poderes en tabla, como `ROUTES`.
3. Cubrir los 5 handlers de `game` que aún no tienen pruebas (`broker`, `handlePower`,
   `handleSalvo`, `handleDisconnect`, `handleCountermeasure`) — ahora ya es posible.
