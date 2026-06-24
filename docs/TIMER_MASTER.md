# Timer Master — Implementación

## Problema

Con múltiples instancias de Node.js detrás de un balanceador, los timers de juego (colocación 60s, turno 30s, Salva 8s, Contramedida 5s) no pueden vivir en `setTimeout` local porque:

1. Si la instancia dueña del timer falla, el timer se pierde
2. Si dos instancias ejecutan el mismo timer, hay comportamientos duplicados

## Solución: Timer Master con lease en Redis

### Elección del Master

Cada instancia intenta adquirir un lease en Redis al arrancar:

```javascript
const MASTER_KEY = 'redis:master:timer';
const INSTANCE_ID = crypto.randomUUID();
const TTL_SECONDS = 1;

async function tryBecomeMaster() {
    const acquired = await redis.set(MASTER_KEY, INSTANCE_ID, {
        NX: true,  // Only set if key does not exist
        EX: TTL_SECONDS
    });
    return acquired === 'OK';
}
```

### Heartbeat

La instancia que es master renueva su lease periódicamente:

```javascript
async function heartbeat() {
    if (!isMaster) return;

    const renewed = await redis.set(MASTER_KEY, INSTANCE_ID, {
        XX: true,  // Only set if key exists
        EX: TTL_SECONDS
    });

    if (!renewed) {
        isMaster = false;
        console.log('[TimerMaster] Lease expired — stepping down');
    }
}

// Ejecutar cada ~500ms
setInterval(heartbeat, 500);
```

### Failover

Si el master falla:
1. Su heartbeat se detiene
2. El TTL de 1s expira
3. Otra instancia ejecuta `SETNX` exitosamente
4. La nueva instancia toma el rol de master

**Downtime máximo:** ~1.5s (1s TTL + ~500ms detección).

### Gestión de timers

Solo el Timer Master ejecuta `setTimeout`/`setInterval`:

```javascript
const TIMERS = {
    COLOCACION: 60_000,
    TURNO: 30_000,
    SALVA: 8_000,
    CONTRAMEDIDA: 5_000
};

const activeTimers = new Map();

function startTimer(salaCodigo, tipo, onEnd) {
    if (!isMaster) return;  // Solo el master

    const duracion = TIMERS[tipo];
    if (!duracion) return;

    const timerId = setTimeout(async () => {
        activeTimers.delete(salaCodigo);
        await onEnd(salaCodigo);
    }, duracion);

    activeTimers.set(salaCodigo, { tipo, timerId });

    // Publicar ticks cada 100ms para el countdown del cliente
    startTicks(salaCodigo, tipo, duracion);
}
```

### Publicación de ticks

El master publica ticks del countdown vía Redis Pub/Sub (o Socket.io directo):

```javascript
function startTicks(salaCodigo, tipo, duracion) {
    const start = Date.now();
    const intervalId = setInterval(() => {
        if (!isMaster) {
            clearInterval(intervalId);
            return;
        }

        const elapsed = Date.now() - start;
        const remaining = Math.max(0, duracion - elapsed);

        // Publicar tick via Socket.io (Redis adapter lo propaga)
        io.to(salaCodigo).emit(`${tipo}:tick`, { remaining });

        if (remaining <= 0) clearInterval(intervalId);
    }, 100);
}
```

### Limpieza al perder el rol

```javascript
function onMasterLost() {
    isMaster = false;
    for (const [codigo, { timerId }] of activeTimers) {
        clearTimeout(timerId);
    }
    activeTimers.clear();
}
```

---

## Mitigación: Margen de gracia post-failover

Si el nuevo master toma el rol y detecta una Contramedida activa cuya ventana ya expiró durante el failover:

```javascript
const GRACE_PERIOD_MS = 1000;  // 1 segundo de gracia

function checkContramedidaExpired(salaCodigo, ventanaInicio) {
    const elapsed = Date.now() - ventanaInicio;
    const duracion = TIMERS.CONTRAMEDIDA + GRACE_PERIOD_MS;

    if (elapsed >= duracion) {
        // Cerrar ventana — el defensor no reaccionó
        endContramedida(salaCodigo, 'expired');
    } else {
        // Conceder el tiempo restante + gracia
        const remaining = duracion - elapsed;
        startTimer(salaCodigo, 'CONTRAMEDIDA', () => {
            endContramedida(salaCodigo, 'expired');
        }, remaining);
    }
}
```

---

## Integración con Socket.io Redis Adapter

```javascript
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));
```

Esto permite que `io.to(codigoSala).emit(...)` funcione entre instancias. El Timer Master usa el mismo mecanismo para publicar ticks y eventos de fin de timer.

---

## Diagrama de secuencia (failover)

```
Instancia A (Master)          Instancia B              Redis
       |                          |                      |
       |--- SETNX master:timer -- |                      |
       |                          |                      |--- OK (A es master)
       |--- heartbeat (c/500ms) - |                      |
       |                          |                      |
       |   [FALLA]                |                      |
       |                          |                      |
       |                          |--- SETNX master:timer |
       |                          |                      |--- OK (B es master)
       |                          |                      |
       |                          |--- [asume timers]    |
       |                          |--- checkContramedida |
       |                          |--- concede gracia 1s |
```

---

## Resumen

| Aspecto | Detalle |
|---------|---------|
| Elección de líder | `SETNX` + TTL de 1s |
| Heartbeat | Cada 500ms (renueva lease) |
| Failover | Automático al expirar TTL (~1.5s) |
| Scope | Solo 4 timers de juego |
| Ticks | Publicados cada 100ms vía Socket.io |
| Gracia post-failover | 1s para Contramedida |
| Líneas de código | ~50 |
