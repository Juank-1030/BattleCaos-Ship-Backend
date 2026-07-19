# BattleCaos-Ship — Backend

Juego de batalla naval multijugador en tiempo real (1v1 · 1v1 vs Bot · 2v2), proyecto académico de Arquitecturas de Software (ARSW). Este README es el **mapa del repositorio**: qué documento leer según lo que necesites, y dónde vive cada cosa.

---

## ¿Por dónde empiezo?

| Si quieres... | Ve a |
|----------------|------|
| Entender el negocio y por qué existe el proyecto | [`desarrollo.md`](#-documentos-de-negocio) |
| Ver las historias de usuario y tareas técnicas | [`BattleCaos-Ship_Inception.md`](#-planeación-y-requisitos) |
| Entender la arquitectura técnica propuesta | [`Proyecto.md`](#-arquitectura) |
| Ver diagramas técnicos detallados (UML) | [`docs/diagramas/`](#-diagramas-técnicos-docsdiagramas) |
| Preparar la presentación del proyecto | [`presentacion/`](#-presentación-presentacion) |
| Repasar teoría de atributos de calidad | [`Criterios_de_calidad.md`](#-fundamentos-teóricos) / [`EscenariosCalidad.md`](#-fundamentos-teóricos) |

---

## 📋 Documentos de negocio

| Archivo | Contenido |
|---------|-----------|
| [`desarrollo.md`](desarrollo.md) | Business case completo: resumen ejecutivo, problema/oportunidad, objetivos, alcance de la solución (modos de juego, mecánicas, Salva Simultánea, Contramedida), beneficios esperados, viabilidad técnica/económica/operativa, riesgos, conclusión. |
| [`contexto.md`](contexto.md) | Versión equivalente del business case (mismo contenido y estructura que `desarrollo.md` — revisar cuál de los dos se mantiene como definitivo antes de la entrega). |
| [`BusinessCase_BattleCaos-ship_ARSW2026.pdf`](BusinessCase_BattleCaos-ship_ARSW2026.pdf) | Versión PDF / formato de entrega del business case. |

## 📐 Planeación y requisitos

| Archivo | Contenido |
|---------|-----------|
| [`BattleCaos-Ship_Inception.md`](BattleCaos-Ship_Inception.md) | Documento de Inception completo: actores, épicas, **historias de usuario y tareas técnicas** (Sprint 1 y Sprint 2), principios INVEST, DoR/DoD, **escenarios de calidad `EC-01`–`EC-06`**, reto de concurrencia, criterios de evaluación. Es el documento que rige *qué* se construye. |
| [`BattleCaos-ship_Inception.csv`](BattleCaos-ship_Inception.csv) | Las mismas historias de usuario en formato tabular (para importar a una herramienta de gestión de backlog). |

## 🏗️ Arquitectura

| Archivo | Contenido |
|---------|-----------|
| [`Proyecto.md`](Proyecto.md) | **Documento de arquitectura oficial.** Arquitectura híbrida (Hexagonal + Event-Driven + Microservicios): catálogo de microservicios, justificación tecnológica, mapa de eventos, despliegue y escalabilidad, **cumplimiento de atributos de calidad** (hoy vs. meta), roadmap de 2 sprints. Es el documento que rige *cómo* se construye. |

### 📊 Diagramas técnicos (`docs/diagramas/`)

14 diagramas PlantUML, cada uno en su propio archivo, con el detalle técnico completo (componentes, clases, despliegue, secuencia, estados, eventos). Índice y guía de visualización en [`docs/DIAGRAMAS.md`](docs/DIAGRAMAS.md).

| # | Archivo | Tipo |
|---|---------|------|
| 01 | `01-componentes-general.puml` | Vista general del sistema |
| 02 | `02-componentes-game-service.puml` | Game Service (hexagonal, detallado) |
| 03 | `03-componentes-room-service.puml` | Room Service (desconexión/reconexión) |
| 04 | `04-clases-dominio.puml` | Modelo de dominio |
| 05 | `05-despliegue-mvp.puml` | Despliegue actual ($0) |
| 06 | `06-despliegue-objetivo.puml` | Despliegue objetivo (post-MVP) |
| 07 | `07-secuencia-disparo-turnos.puml` | Secuencia: disparo en turnos |
| 08 | `08-secuencia-salva-simultanea.puml` | Secuencia: núcleo de concurrencia |
| 09 | `09-secuencia-desconexion-reconexion.puml` | Secuencia: desconexión/reconexión |
| 10 | `10-secuencia-acceso-identificacion.puml` | Secuencia: login y acceso |
| 11 | `11-estados-fases-partida.puml` | Estados: fases de la partida |
| 12 | `12-estados-conexion-jugador.puml` | Estados: conexión del jugador |
| 13 | `13-mapa-eventos.puml` | Catálogo de eventos |
| 14 | `14-comparativa-hoy-vs-meta.puml` | Monolito vs. arquitectura híbrida |

### Otros documentos de arquitectura en `docs/`

| Archivo | Contenido |
|---------|-----------|
| [`docs/ESCALABILIDAD.md`](docs/ESCALABILIDAD.md) | Visión de escalabilidad horizontal del sistema. |
| [`docs/TIMER_MASTER.md`](docs/TIMER_MASTER.md) | Patrón Timer Master (leader election) para temporizadores en múltiples instancias. |

## 🎙️ Presentación (`presentacion/`)

| Archivo | Contenido |
|---------|-----------|
| [`presentacion/presentacion.md`](presentacion/presentacion.md) | Guion listo para convertir en diapositivas: contexto del proyecto, mecánicas y poderes, escenarios de calidad (4 ejemplos), requisitos de calidad seleccionados, argumentación de la arquitectura, justificación tecnológica, tensiones reconocidas, conclusiones. Cada `##` es una diapositiva; los bloques **"Notas del orador"** llevan el detalle que no va en pantalla. |
| `presentacion/diagramas/` | 5 diagramas PlantUML simplificados, pensados para proyectar (versión resumida de los técnicos de `docs/diagramas/`). |

| # | Archivo |
|---|---------|
| 01 | `01-arquitectura-tres-paradigmas.puml` |
| 02 | `02-vision-general-arquitectura.puml` |
| 03 | `03-tacticas-por-atributo-calidad.puml` |
| 04 | `04-evolucion-monolito-a-microservicios.puml` |
| 05 | `05-componentes-por-servicio.puml` |

## 📚 Fundamentos teóricos

| Archivo | Contenido |
|---------|-----------|
| [`Criterios_de_calidad.md`](Criterios_de_calidad.md) | Teoría de atributos de calidad (ISO 25010) y sus tácticas: Disponibilidad, Modificabilidad, Desempeño, Seguridad, Testeabilidad, Escalabilidad. `Proyecto.md §9` evalúa la arquitectura contra este documento. |
| [`EscenariosCalidad.md`](EscenariosCalidad.md) | Plantilla del formato de 6 partes (Fuente, Estímulo, Artefacto, Entorno, Respuesta, Medida) usada para redactar los escenarios `EC-01`–`EC-06` en `Inception.md`. |

---

## Cómo se relacionan estos documentos

```
desarrollo.md / contexto.md  (¿por qué?)
        │
        ▼
BattleCaos-Ship_Inception.md  (¿qué? — historias, tareas, escenarios EC-01..EC-06)
        │
        ▼
Proyecto.md  (¿cómo? — arquitectura, cumplimiento de Criterios_de_calidad.md)
        │
        ├──▶ docs/diagramas/        (detalle técnico, 14 diagramas)
        └──▶ presentacion/          (síntesis para exponer, 5 diagramas)
```

## Visualizar los diagramas `.puml`

1. **VS Code:** extensión "PlantUML" (jebbs.plantuml) → `Alt+D` para vista previa.
2. **Online:** pegar el contenido en https://www.plantuml.com/plantuml/uml/
3. **CLI:** `java -jar plantuml.jar <archivo>.puml` (requiere Java + el JAR de PlantUML).
