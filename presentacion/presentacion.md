# BattleCaos-Ship
## Arquitectura Híbrida para un Juego Naval Multijugador en Tiempo Real

**Autor:** Juan Carlos Bohorquez Monroy
**Materia:** Arquitecturas y Sistemas de Software (ARSW) — 2026
**Fecha de presentación:** 8 de julio de 2026

---

## Índice

1. Contexto del proyecto
2. Mecánicas — qué lo diferencia
3. Catálogo de poderes
4. Estado de implementación actual
5. Escenarios de calidad
6. Requisitos de calidad seleccionados
7. Argumentación de la arquitectura
8. Los tres paradigmas
9. Justificación tecnológica
10. Testing y calidad del código
11. Tensiones reconocidas
12. Historias de usuario y tareas técnicas
13. Conclusiones

---

## 1. Contexto del proyecto

**BattleCaos-Ship** — batalla naval multijugador en tiempo real

- Modos: **1v1 · 1v1 vs Bot · 2v2 colaborativo**
- El servidor es la única fuente de verdad — el cliente solo renderiza
- Diseñado para poner a prueba **concurrencia y comunicación en tiempo real**

### Mecánicas diferenciadoras

- **Salva Simultánea** — todos disparan a la vez (ventana de 8s)
- **Poderes + Contramedida** — energía acumulable, ventana de reacción de 5s
- **2v2 colaborativo** — flota compartida + chat privado de equipo

### KPIs de negocio

| KPI | Meta |
|-----|------|
| % partidas completadas | > 80% |
| Latencia P95 por disparo | < 200ms |
| % reconexiones exitosas | > 95% |
| Salas concurrentes sin fuga | 0 eventos cruzados |

---

## 2. Mecánicas — qué lo diferencia de un Battleship clásico

| Mecánica | Qué aporta |
|----------|------------|
| **Turnos clásicos** | Disparo por turno, agua / impacto / hundido |
| **Energía por equipo** | +1 por impacto, +3 por hundimiento — recurso estratégico |
| **Salva Simultánea** | Cada 3 rondas, todos disparan a la vez durante 8s |
| **Poderes** | 5 habilidades que se compran con energía acumulada |
| **Contramedida** | Ventana de 5s para anular un poder enemigo |
| **2v2 colaborativo** | Flota y energía en equipo, chat privado |

Un Battleship clásico es azar por turnos.
BattleCaos-Ship agrega **estrategia, economía y concurrencia real**.

---

## 3. Catálogo de poderes

| Poder | Costo | Tipo | Efecto |
|-------|:-----:|------|--------|
| **Bombardeo de área** | 2 | Ofensivo | Impacta una zona de 3×3 casillas |
| **Sonar** | 2 | Inteligencia | Revela si hay barco en una fila/columna |
| **Escudo** | 1 | Defensivo | Protege una casilla propia por 2 turnos |
| **Tormenta** | 3 | Sabotaje (1 vez) | Anula el siguiente turno del rival |
| **Contramedida** | — | Reactivo | Anula un poder ofensivo activado en <5s |

Agregar un sexto poder = **máximo 2 archivos nuevos, menos de 3 horas** *(EC-05 — Modificabilidad)*

---

## 4. Estado de implementación actual

### Microservicios completados ✅

| Servicio | Puerto | Tests unitarios | Cobertura |
|---|:---:|:---:|:---:|
| **Gateway** | :3000 | 20 tests | 100% |
| **Auth Service** | :3001 | 13 tests | 100% |
| **Room Service** | Redis | 39 tests | 100% |

### En construcción ⬜

Game Service · Timer Service · Chat Service · Bot Service · Observability

### Infraestructura operativa

- Redis en la nube — Upstash (`rediss://:6379` TLS)
- Logger estructurado — pino (JSON en producción)
- Tests de integración Room — **13/13 escenarios pasando**
- **Total: 72 tests unitarios + 13 de integración — 0 fallos**

---

## 5. Escenarios de calidad

### EC-01 — Desempeño en tiempo real *(deriva de DOMH04)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario |
| Estímulo | Realiza una acción que cambia el estado de la partida (disparo, turno, fase) |
| Artefacto | Distribución del estado de la partida |
| Entorno | Operación normal, hasta 5 salas concurrentes |
| Respuesta | El sistema resuelve la acción y entrega el estado actualizado a todos los participantes de la sala |
| Medida de Respuesta | **Latencia P95 < 200 ms** entre la acción y la actualización recibida — estimado actual: **~35 ms** |

---

### EC-02 — Concurrencia sin condiciones de carrera *(deriva de DOMH12)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Dos o más usuarios |
| Estímulo | Disparan a la misma casilla casi en el mismo instante, durante la ventana de disparo simultáneo |
| Artefacto | Resolución exclusiva por casilla (lock `SETNX` en Redis) |
| Entorno | Ventana de disparo simultáneo activa, con disparos concurrentes de prueba |
| Respuesta | Solo el disparo que llega primero resuelve la casilla; los demás se rechazan como repetidos |
| Medida de Respuesta | **0% de casillas resueltas dos veces** en pruebas con disparos concurrentes simulados |

---

### EC-03 — Disponibilidad ante caída de red *(deriva de DOMH05)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario |
| Estímulo | Pierde la conexión durante una partida activa y vuelve a conectarse |
| Artefacto | Recuperación de estado tras reconexión |
| Entorno | Operación normal, partida en curso |
| Respuesta | El sistema reasocia al usuario y le entrega el estado completo y actual de su partida |
| Medida de Respuesta | **> 95% de reconexiones exitosas**, recuperación en **menos de 3 segundos** |

---

### EC-04 — Seguridad de acceso *(deriva de DOMH02)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuario sin credencial válida (sesión expirada, manipulada o inexistente) |
| Estímulo | Intenta establecer conexión o realizar una acción sobre una partida |
| Artefacto | Verificación de identidad en cada conexión Socket.io (middleware JWT local) |
| Entorno | Operación normal |
| Respuesta | El sistema rechaza la conexión o la acción y no expone ningún estado de partida |
| Medida de Respuesta | **0% de accesos no autorizados aceptados** en pruebas con credenciales inválidas o expiradas |

---

### EC-05 — Modificabilidad del catálogo de poderes *(deriva de DOMH10)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Desarrollador |
| Estímulo | Agrega un nuevo poder al catálogo existente |
| Artefacto | Módulo de poderes del motor de juego (Domain Kernel) |
| Entorno | Tiempo de diseño |
| Respuesta | El nuevo poder se integra sin modificar la validación de turnos, energía ni disparos |
| Medida de Respuesta | **Máximo 2 archivos** modificados o agregados; implementado y probado en **menos de 3 horas** |

---

### EC-06 — Escalabilidad por aislamiento de salas *(deriva de DOMH03)*

| Elemento | Valor |
|----------|-------|
| Fuente de Estímulo | Usuarios de distintas partidas |
| Estímulo | Crean y juegan simultáneamente en múltiples salas |
| Artefacto | Aislamiento entre salas concurrentes (clave Redis `sala:{codigo}`) |
| Entorno | Carga sostenida de al menos 5 salas activas |
| Respuesta | Cada sala opera de forma aislada; ningún evento de una sala afecta a otra |
| Medida de Respuesta | **0 fugas de eventos** entre salas; sin degradación perceptible de latencia con **5 o más salas concurrentes** |

---

## 6. Requisitos de calidad seleccionados

| Atributo | Por qué este proyecto lo necesita | Táctica implementada |
|----------|-----------------------------------|----------------------|
| **Desempeño** | La Salva exige resolver disparos concurrentes en milisegundos | Redis Pub/Sub + estado externo |
| **Disponibilidad** | Multijugador real-time sobre redes inestables | `conectado: false` sin eliminar jugador |
| **Seguridad** | El servidor autoritativo solo vale con identidad verificada | JWT + Google OAuth en cada conexión |
| **Modificabilidad** | El catálogo de poderes debe evolucionar fácilmente | Domain Kernel — dominio puro sin dependencias |
| **Escalabilidad** | "Salas concurrentes" es un KPI de negocio declarado | Clave Redis `sala:{codigo}` por sala |

---

## 7. Argumentación de la arquitectura

### 5 principios que gobiernan toda decisión

1. **Servidor autoritativo** — el backend decide, nunca el cliente
2. **Estado en Redis** — externo a todos los servicios, nunca en memoria del proceso
3. **Comunicación asíncrona** — sin llamadas HTTP directas entre servicios de dominio
4. **Consistencia atómica** — `INCRBY`, `SETNX`, `MULTI`/`EXEC`
5. **Gateway único** — único punto de entrada público

### Por qué una arquitectura híbrida

| Opción | Resultado |
|--------|-----------|
| Monolito puro | No testeable sin infraestructura, no escala por dominio |
| Microservicios puros desde el día 1 | Riesgo alto para 2 sprints |
| **Híbrida (adoptada)** | Microservicios ya separados + Domain Kernel en los servicios complejos |

---

## 8. Los tres paradigmas

### ① Domain Kernel

```
DOMAIN   — funciones puras, 0 imports externos, 100% testeables sin mocks
   ↑
HANDLERS — leen Redis → llaman domain → escriben Redis → publican eventos
   ↑
REDIS    — única fuente de verdad
```

Aplicado en **Room Service** (implementado) y **Game Service** (en construcción)

```javascript
// DOMINIO — función pura, sin Redis ni Socket.io
export function agregarJugador(sala, playerId, name, socketId) {
  if (estaLlena(sala)) throw new Error('sala_llena');
  const equipo = asignarEquipo(sala);
  sala.jugadores.push({ id: playerId, name, socketId, equipo, conectado: true });
  return equipo;
}

// HANDLER — orquesta, sin lógica de negocio propia
async function handleJoin({ socketId, codigo, playerId, name }) {
  const sala = JSON.parse(await redis.get(`sala:${codigo}`));
  const equipo = agregarJugador(sala, playerId, name, socketId);
  await redis.set(`sala:${codigo}`, JSON.stringify(sala));
  await broadcast(codigo, 'room:joined', { jugadores: sala.jugadores });
}
```

### ② Event-Driven

- Los servicios **nunca se llaman directamente** — publican y consumen eventos en Redis Pub/Sub
- Si Chat Service cae, el juego continúa — ningún servicio crítico depende de uno no crítico
- 20+ eventos documentados con publicador y consumidores explícitos

### ③ Microservicios

- Solo **Gateway (:3000)** y **Auth (:3001)** tienen puerto público
- Room, Game, Chat, Timer, Bot, Observability: **workers Redis puros — sin puerto propio**
- Cada dominio escala de forma independiente según su carga

---

## 9. Justificación tecnológica

| Tecnología | Para qué | Por qué esta y no otra |
|------------|----------|------------------------|
| **Node.js 20 + ESM** | Runtime | I/O intensiva, módulos nativos, top-level await |
| **Socket.io 4** | Tiempo real | Rooms, reconexión automática, fallback long-polling |
| **Redis / Upstash** | Estado + Pub/Sub + locks | Única pieza con atomicidad sub-ms: `SETNX`, `INCRBY` |
| **ioredis 5** | Cliente Redis | ESM nativo, `lazyConnect`, 2 conexiones sub/pub |
| **JWT + Google OAuth** | Autenticación | Sin manejar contraseñas, verificación local con `JWT_SECRET` |
| **pino** | Logging estructurado | JSON en producción, pretty en desarrollo, nivel por env var |
| **Vitest + coverage-v8** | Testing | ESM nativo, cobertura V8, sin transpilación |
| **Render + Upstash + Vercel** | Hosting | Costo: **$0** para el MVP |

---

## 10. Testing y calidad del código

### Estrategia por capas

```
┌──────────────────────────────────────────────────────┐
│  Tests de integración — simulan el flujo real         │
│  "Gateway falso" publica en Redis y escucha eventos   │
│  Room Service: 13/13 escenarios OK                    │
├──────────────────────────────────────────────────────┤
│  Tests unitarios — solo la capa domain/               │
│  Sin Redis, sin mocks, sin infraestructura            │
│  72 tests en 3 servicios — 100% cobertura             │
└──────────────────────────────────────────────────────┘
```

### Cobertura por servicio

| Servicio | Módulo | Tests | Statements | Branches | Functions | Lines |
|---|---|:---:|:---:|:---:|:---:|:---:|
| Gateway | `domain/router.js` | 20 | 100% | 100% | 100% | 100% |
| Auth | `auth/jwt.js` | 13 | 100% | 100% | 100% | 100% |
| Room | `domain/room.js` | 39 | 100% | 100% | 100% | 100% |

**Branches** es la métrica más importante — verifica que todos los caminos del `if/else` estén cubiertos.
Equivalente al reporte HTML de JaCoCo: `npm run coverage` → `coverage/index.html`

---

## 11. Tensiones reconocidas

### Tensión 1 — Redis y Gateway son puntos únicos de falla

| Componente | Mitigación MVP | Mitigación futura |
|---|---|---|
| **Redis** | Backups automáticos Upstash | Redis Sentinel / Cluster |
| **Gateway** | Render reinicia en ~30s | 2+ réplicas + Load Balancer |
| **Timer master** | Leader election SETNX + failover <1.5s | Ya implementado |

### Tensión 2 — El desempeño empeora con la distribución

| Escenario | Latencia estimada |
|---|---|
| Monolito (sin red entre componentes) | ~8ms/disparo |
| Arquitectura híbrida actual | **~35ms/disparo** |
| Umbral máximo aceptable (EC-01) | 200ms |

**Aceptado conscientemente** — la ganancia en testeabilidad y modificabilidad lo justifica.
El overhead sigue siendo 5× menor que el umbral exigido.

---

## 12. Historias de usuario y tareas técnicas

### Épicas del proyecto

| Épica | Descripción | Historias | Tareas |
|-------|-------------|:---------:|:------:|
| **DOM-0** | Backend: autenticación, salas, motor de juego, observabilidad | 15 | 29 |
| **UIUX-1** | Frontend: pantallas, tablero, poderes, chat | 13 | 14 |
| **DOC-2** | Documentación: diagramas, contratos, concurrencia | 5 | 5 |
| **Total** | 2 sprints — entrega 8 julio 2026 | **33** | **48** |

---

### Sprint 1 — Cimientos, Conexión y Core

| Historia | Enunciado | Prioridad | Estado | Tareas técnicas |
|----------|-----------|:---------:|:------:|-----------------|
| **DOMH01** | Estado centralizado como fuente única de verdad | Must | ✅ | DOMF001 · DOMF002 |
| **DOMH02** | Identificación segura del usuario | Must | ✅ | DOMF101 · DOMF102 |
| **DOMH03** | Creación y unión a salas mediante código | Must | ✅ | DOMF201 · DOMF202 · DOMF203 |
| **DOMH04** | Estado de partida siempre actualizado | Must | ⬜ | DOMF301 · DOMF302 |
| **DOMH05** | Recuperación de partida tras desconexión | Must | 🔄 | DOMF401 · DOMF402 ✅ · DOMF403 ✅ |
| **DOMH06** | Colocación de la flota propia | Must | ⬜ | DOMF501 |
| **DOMH07** | Colocación coordinada en equipo (2v2) | Must | ⬜ | DOMF502 |
| **DOMH08** | Disparos por turnos con resultado visible | Must | ⬜ | DOMF601 · DOMF602 |
| **DOMH09** | Acumulación de energía por equipo | Must | ⬜ | DOMF701 |
| UIUXH01–08 | Identidad visual, login, lobby, tablero, chat, colocación, turno, energía | Must / Should | ⬜ | UIUXF001–701 |
| **DOCH01** | Arquitectura documentada | Must | ✅ | DOCF001 |
| **DOCH02** | Acceso y comunicación documentados | Must | ✅ | DOCF101 |
| **DOCH03** | Motor del juego documentado | Must | 🔄 | DOCF201 |

---

### Sprint 2 — Mecánicas, Observabilidad y Cierre

| Historia | Enunciado | Prioridad | Estado | Tareas técnicas |
|----------|-----------|:---------:|:------:|-----------------|
| **DOMH10** | Poderes mediante energía acumulada | Must | ⬜ | DOMF801 · DOMF802 |
| **DOMH11** | Ventana de reacción para anular un poder | Must | ⬜ | DOMF901 |
| **DOMH12** | Ronda de disparo simultáneo ← reto de concurrencia | Must | ⬜ | DOMF1001 · **DOMF1002** |
| **DOMH13** | Oponente automático (modo 1v1-Bot) | Could | ⬜ | DOMF1101 · DOMF1102 |
| **DOMH14** | Visibilidad del comportamiento del sistema | Must | ⬜ | DOMF1201 · DOMF1202 · DOMF1203 |
| **DOMH15** | Continuidad ante inactividad o desconexión | Should | ⬜ | DOMF1301 · DOMF1302 · DOMF1303 |
| UIUXH09–13 | Poderes, reacción, salva, indicadores, cierre | Must / Could | ⬜ | UIUXF801–1201 |
| **DOCH04** | Concurrencia documentada | Must | ⬜ | DOCF301 |
| **DOCH05** | Documentación de cierre y demo | Must | ⬜ | DOCF401 |

---

### Resumen de avance — Sprint 1

| Métrica | Valor |
|---------|-------|
| Historias de dominio completadas | 3 de 9 — DOMH01 · DOMH02 · DOMH03 |
| Historias de dominio en progreso | 1 de 9 — DOMH05 (desconexión 2/3 tareas ✅) |
| Microservicios operativos en producción | 3 de 8 — Gateway · Auth · Room |
| Tests unitarios — 100% cobertura | 72 tests |
| Tests de integración Room | 13/13 OK |
| Costo de infraestructura | $0 (Render + Upstash free tier) |

**DOMF1002** (resolución exclusiva por casilla + SETNX) es el núcleo del reto de concurrencia del curso — cubierto por EC-02.

---

## 13. Conclusiones

- **La arquitectura funciona hoy:** 3 microservicios operativos en producción, 72 tests unitarios al 100% de cobertura, 13 pruebas de integración end-to-end sin fallos

- **Los tres paradigmas se refuerzan mutuamente:**
  - Event-Driven desacopla los servicios sin que se conozcan entre sí
  - Domain Kernel hace el dominio testeable sin infraestructura
  - Microservicios permite escalar cada dominio de forma independiente

- **Las tensiones están cuantificadas:** +27ms de latencia adicional — documentado, aceptado, y 5× por debajo del umbral de 200ms

- **El camino al MVP está definido:** 5 servicios restantes + Frontend — entrega **8 de julio de 2026**

---

*Juan Carlos Bohorquez Monroy — ARSW 2026*
