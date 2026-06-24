# Escalabilidad Horizontal — BattleCaos-Ship

## Visión general

El servidor está diseñado como un **monolito modular** que se replica horizontalmente sin cambios en la lógica de negocio. El estado de juego vive en Redis (centralizado), los timers se gestionan con un patrón **Timer Master**, y Socket.io usa Redis como bus de mensajes entre instancias.

---

## MVP (Sprint 1-2): Una sola instancia

- Timers en memoria (`setTimeout`)
- Redis para estado de juego, energía, locks y métricas
- Socket.io sin adapter (broadcast local)

Capacidad estimada: **5+ salas concurrentes / ~20 jugadores simultáneos** en plan F1 de Azure.

---

## Post-MVP: N instancias

```
[Azure Load Balancer]
    ↙        ↓        ↘
[Node App 1] [Node App 2] [Node App 3]
    ↖        ↓        ↗
[Redis Upstash — estado + locks + pub/sub]
    ↖        ↓        ↗
[Timer Master — solo 1 instancia con el lease]
```

### Requisitos técnicos

| # | Cambio | Código |
|---|--------|--------|
| 1 | Socket.io Redis adapter | `npm install @socket.io/redis-adapter` + 5 líneas en server.js |
| 2 | Timer Master pattern | ~50 líneas: SETNX + heartbeat + Pub/Sub |
| 3 | Scripts Lua (opcional) | Atomicidad compuesta para operaciones multi-key |

### Lo que NO cambia

| Componente | Razón |
|------------|-------|
| Energía (INCRBY/DECRBY) | Ya es atómico en Redis |
| Locks de Salva (GETSET) | Ya es atómico en Redis |
| Estado de salas | Ya vive en Redis |
| Snapshots | Socket.io adapter propaga el broadcast |
| Lógica de juego | No depende de la instancia |

---

## Timer Master

Ver [`docs/TIMER_MASTER.md`](./TIMER_MASTER.md) para la implementación detallada.

### Scope

| Timer | Duración | ¿Timer Master? |
|-------|----------|----------------|
| Colocación | 60s | ✅ |
| Turno | 30s | ✅ |
| Salva | 8s | ✅ |
| Contramedida | 5s | ✅ |
| Energía | — | ❌ (vive en Redis) |
| Locks | — | ❌ (vive en Redis) |
| Snapshots | — | ❌ (vive en Redis) |

### Ventana de downtime máximo: ~1.5s

Aceptable para todos los timers del MVP. La Contramedida incluye un margen de gracia de 1s post-failover.

---

## Documentación relacionada

- [`docs/TIMER_MASTER.md`](./TIMER_MASTER.md) — Implementación detallada
- `contexto.md` — Sección 6.1 (viabilidad técnica)
- `BattleCaos-Ship_Inception.md` — Sección de escalabilidad horizontal
