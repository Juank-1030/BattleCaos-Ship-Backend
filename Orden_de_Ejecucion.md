# Orden de Ejecución — BattleCaos-Ship

Orden basado en dependencias entre historias de usuario. Cada fase requiere que la anterior esté completada.

**Total: 33 historias de usuario | 51 tareas técnicas**

---

## SPRINT 1 — Cimientos, Conexión y Core

---

### Fase 0: Fundación

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 1 | **DOMH01 — Estado centralizado como fuente única de verdad** | Must |

**Tareas:**
- DOMF001: Inicialización de cada servicio y conexión al almacenamiento de estado compartido
- DOMF002: Enrutamiento de eventos del Gateway y control de tráfico

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 2 | **UIUXH01 — Identidad visual coherente** | Must |

**Tareas:**
- UIUXF001: Pantallas principales de la aplicación cliente
- UIUXF002: Tablero visual parametrizable y lenguaje de colores

---

### Fase 1: Identidad y Salas

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 3 | **DOMH02 — Identificación segura del usuario** | Must |

**Tareas:**
- DOMF101: Identificación delegada y emisión de credencial de sesión
- DOMF102: Verificación de la credencial en cada conexión

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 4 | **UIUXH02 — Inicio de sesión sin fricción** | Must |

**Tareas:**
- UIUXF101: Pantalla de inicio de sesión con estados de carga y error

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 5 | **DOMH03 — Creación y unión a salas mediante código** | Must |

**Tareas:**
- DOMF201: Creación y unión a salas con asignación de equipo
- DOMF202: Aislamiento entre salas concurrentes
- DOMF203: Publicación de eventos de ciclo de vida de sala

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 6 | **UIUXH03 — Lobby para crear o unirse a salas** | Must |

**Tareas:**
- UIUXF201: Lobby con selector de modo, creación y unión a salas

---

### Fase 2: Loop Principal del Juego

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 7 | **DOMH04 — Estado de partida siempre actualizado** | Must |

**Tareas:**
- DOMF301: Distribución del estado completo tras cada cambio
- DOMF302: Definición de fases válidas de una partida

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 8 | **UIUXH04 — Tablero siempre fiel al estado real** | Must |

**Tareas:**
- UIUXF301: Render del tablero a partir del estado recibido

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 9 | **DOMH06 — Colocación de la flota propia** | Must |

**Tareas:**
- DOMF501: Validación y registro de la colocación de flota

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 10 | **UIUXH06 — Colocación de flota cómoda e intuitiva** | Must |

**Tareas:**
- UIUXF501: Colocación interactiva de barcos con vista previa

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 11 | **DOMH07 — Colocación coordinada en equipo** | Must |

**Tareas:**
- DOMF502: Coordinación de colocación en equipo con límite de tiempo

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 12 | **DOMH08 — Disparos por turnos con resultado visible** | Must |

**Tareas:**
- DOMF601: Validación y resolución de disparos por turno
- DOMF602: Detección de hundimientos y condición de victoria

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 13 | **UIUXH07 — Turno y resultado de disparos visibles** | Must |

**Tareas:**
- UIUXF601: Indicador de turno y retroalimentación visual de disparos

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 14 | **DOMH09 — Acumulación de energía de equipo** | Must |

**Tareas:**
- DOMF701: Acumulación de energía por equipo sin condiciones de carrera

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 15 | **UIUXH08 — Energía de equipo visible** | Must |

**Tareas:**
- UIUXF701: Barra de energía sincronizada con el estado de la partida

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 16 | **DOMH05 — Recuperación de partida tras una desconexión** | Must |

**Tareas:**
- DOMF401: Recuperación de estado al reconectar
- DOMF402: El usuario desconectado conserva su lugar (no se expulsa)
- DOMF403: Publicación de eventos de desconexión y reconexión desde la sala

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 17 | **UIUXH05 — Chat de equipo en tiempo real** | Should |

**Tareas:**
- UIUXF401: Chat de equipo y general en tiempo real
- DOMF1103: Servicio de mensajería: recepción, persistencia y enrutamiento

---

## SPRINT 2 — Mecánicas, Observabilidad y Cierre

---

### Fase 3: Calidad y Preparación

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| — | *Tarea transversal* (asociada a DOMH01) | Should |

**Tarea:**
- DOMF003: Suite de tests unitarios del dominio del Game Service

---

### Fase 4: Poderes y Contramedida

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 18 | **DOMH10 — Uso de poderes mediante energía acumulada** | Must |

**Tareas:**
- DOMF801: Poderes ofensivos: bombardeo de área y detección
- DOMF802: Poderes defensivos: protección y bloqueo de turno

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 19 | **UIUXH09 — Panel de poderes claro** | Must |

**Tareas:**
- UIUXF801: Panel de poderes con costo y selección de objetivo

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 20 | **DOMH11 — Ventana de reacción para anular un poder** | Must |

**Tareas:**
- DOMF901: Ventana de reacción, anulación y devolución de energía

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 21 | **UIUXH10 — Aviso visible de ventana de reacción** | Must |

**Tareas:**
- UIUXF901: Aviso de ventana de reacción con cuenta regresiva

---

### Fase 5: Salva, Bot y Timeouts

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 22 | **DOMH12 — Ronda de disparo simultáneo** | Must |

**Tareas:**
- DOMF1001: Ventana de disparo simultáneo
- DOMF1002: Resolución exclusiva por casilla y control de cadencia

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 23 | **UIUXH11 — Interfaz clara durante la ronda simultánea** | Must |

**Tareas:**
- UIUXF1001: Interfaz de la ronda simultánea con cadencia y retroalimentación

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 24 | **DOMH13 — Oponente automático para jugar en solitario** | Could |

**Tareas:**
- DOMF1101: Oponente automático con disparo y remate básico
- DOMF1102: Integración del oponente automático con el bus de eventos

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 25 | **DOMH15 — Continuidad ante inactividad o desconexión** | Should |

**Tareas:**
- DOMF1301: Límite de tiempo de turno y abandono definitivo por desconexión prolongada
- DOMF1302: Verificación de disponibilidad previa a la operación
- DOMF1303: Timer Service: elección de líder, heartbeat y gestión de temporizadores

---

### Fase 6: Observabilidad, Cierre y Documentación

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 26 | **DOMH14 — Visibilidad del comportamiento del sistema** | Must |

**Tareas:**
- DOMF1201: Registro estructurado de eventos clave
- DOMF1202: Métricas técnicas de concurrencia y rendimiento
- DOMF1203: Cálculo de los indicadores de negocio

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 27 | **UIUXH12 — Panel de indicadores para el operador** | Could |

**Tareas:**
- UIUXF1101: Panel mínimo de indicadores de negocio

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 28 | **UIUXH13 — Cierre de partida claro y manejo de errores** | Must |

**Tareas:**
- UIUXF1201: Pantalla de victoria y manejo visible de errores

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 29 | **DOCH01 — Arquitectura documentada** | Must |

**Tareas:**
- DOCF001: Diagramas de despliegue y componentes

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 30 | **DOCH02 — Acceso y comunicación documentados** | Must |

**Tareas:**
- DOCF101: Secuencia de acceso y contrato de eventos

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 31 | **DOCH03 — Motor del juego documentado** | Must |

**Tareas:**
- DOCF201: Diagrama de clases del motor y de fases

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 32 | **DOCH04 — Concurrencia documentada** | Must |

**Tareas:**
- DOCF301: Diagramas de secuencia de la ronda simultánea y la ventana de reacción

---

| # | Historia | Prioridad |
|:-:|----------|:---------:|
| 33 | **DOCH05 — Documentación de cierre y demo** | Must |

**Tareas:**
- DOCF401: Manual de demo y visión de evolución futura
