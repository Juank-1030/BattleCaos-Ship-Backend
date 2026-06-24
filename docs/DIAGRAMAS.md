# Diagramas de Arquitectura

Carpeta: `docs/diagramas/` — un archivo `.puml` independiente por diagrama, basados y corregidos contra `Proyecto.md`.

## Cómo visualizar los diagramas

### Opción 1: VS Code (recomendado)
1. Instala la extensión **"PlantUML"** (jebbs.plantuml)
2. Abre cualquier archivo dentro de `docs/diagramas/`
3. `Alt+D` para vista previa

### Opción 2: Online
1. Ve a https://www.plantuml.com/plantuml/uml/
2. Copia y pega el contenido completo de un archivo (`@startuml ... @enduml`)

### Opción 3: Línea de comandos
```bash
# Requiere Java + PlantUML JAR
java -jar plantuml.jar docs/diagramas/*.puml
```

## Contenido

| # | Archivo | Tipo | Descripción |
|---|---------|------|-------------|
| 01 | `01-componentes-general.puml` | Componentes | Vista general del sistema: Gateway, Broker, Redis State y los 7 servicios, con todas sus relaciones de publicación/consumo. |
| 02 | `02-componentes-game-service.puml` | Componentes (detalle) | Arquitectura hexagonal del Game Service: dominio puro, casos de uso, puertos y adaptadores. |
| 03 | `03-componentes-room-service.puml` | Componentes (detalle) | Room Service: ciclo de vida de salas y el flujo corregido de desconexión ≠ abandono. |
| 04 | `04-clases-dominio.puml` | Clases | Modelo de dominio del Game Service (GameEngine, Board, Fleet, Salvo, Powers, etc.) y los puertos que usa. |
| 05 | `05-despliegue-mvp.puml` | Despliegue | Topología real de hoy: 1 servicio web en Render + 1 Redis en Upstash, presupuesto $0. |
| 06 | `06-despliegue-objetivo.puml` | Despliegue | Topología objetivo post-MVP: microservicios independientes, Gateway replicado, Redis con HA. |
| 07 | `07-secuencia-disparo-turnos.puml` | Secuencia | Flujo completo de un disparo en fase de Turnos, incluyendo eventos a Observability y Bot. |
| 08 | `08-secuencia-salva-simultanea.puml` | Secuencia | El núcleo de concurrencia del proyecto: resolución exclusiva por casilla con locks atómicos en Redis. |
| 09 | `09-secuencia-desconexion-reconexion.puml` | Secuencia | Flujo corregido de desconexión/reconexión: caminos de éxito y de abandono definitivo. |
| 10 | `10-secuencia-acceso-identificacion.puml` | Secuencia | Identificación con proveedor externo, conexión en tiempo real, creación y unión a sala. |
| 11 | `11-estados-fases-partida.puml` | Estados | Máquina de fases de una partida: LOBBY → COLOCACION → TURNOS ↔ SALVA → FIN. |
| 12 | `12-estados-conexion-jugador.puml` | Estados | Estados de conexión de un jugador dentro de una partida: Conectado / Desconectado / Retirado. |
| 13 | `13-mapa-eventos.puml` | Componentes | Catálogo completo de eventos y qué servicio publica/consume cada uno. |
| 14 | `14-comparativa-hoy-vs-meta.puml` | Componentes | Comparación visual entre el monolito actual y la arquitectura híbrida objetivo. |

## Notas de consistencia con `Proyecto.md`

- El **broker de eventos es Redis Pub/Sub** en todos los diagramas; RabbitMQ aparece únicamente como evolución futura documentada (nunca como lo desplegado hoy).
- El despliegue actual usa **Render + Upstash + Vercel** (presupuesto $0), no Azure.
- El roadmap real es de **2 sprints**, alineados con las historias de `Inception.md` — la separación en microservicios como procesos independientes es post-MVP.
- El flujo de desconexión distingue explícitamente **"perder la conexión"** de **"abandonar la partida"** (diagramas 03, 09 y 12), reflejando la corrección aplicada a `DOMH05`/`DOMF402` en esta sesión de trabajo.
