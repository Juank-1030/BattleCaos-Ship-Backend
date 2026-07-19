# Informe: Atributos de Calidad del Software

---

## 1. Definición de Atributo de Calidad

Un atributo de calidad es una **propiedad medible de un sistema**, que indica qué tan bien el sistema satisface las necesidades de las partes interesadas. También se conoce como:

- Requerimientos no funcionales
- Características de arquitectura
- Propiedades de calidad

---

## 2. Clasificación General de Atributos de Calidad

Los atributos de calidad se organizan en categorías principales dentro de la utilidad del sistema (**Utility**):

| Categoría | Subcategorías | Ejemplos de escenarios |
|-----------|--------------|----------------------|
| **Performance** | Data Latency, Transaction Throughput | Reducir latencia de almacenamiento a < 200ms; entregar video en tiempo real |
| **Modifiability** | New products, Change COTS | Agregar middleware CORBA en < 20 meses-persona |
| **Availability** | H/W failure, COTS S/W failures | Redirigir tráfico en < 3 segundos ante fallo de energía |
| **Security** | Data confidentiality, Data integrity | Transacciones seguras el 99.999% del tiempo |

---

## 3. Estándar de Calidad del Software (ISO 25010)

Según el estándar ISO 25010, la calidad del producto software se organiza en las siguientes características principales:

### 3.1 Adecuación Funcional
- Completitud funcional
- Corrección funcional
- Pertinencia funcional

### 3.2 Eficiencia de Desempeño
- Comportamiento temporal
- Utilización de recursos
- Capacidad

### 3.3 Compatibilidad
- Coexistencia
- Interoperabilidad

### 3.4 Usabilidad
- Inteligibilidad
- Aprendizaje
- Operabilidad
- Protección frente a errores de usuario
- Estética
- Accesibilidad

### 3.5 Fiabilidad
- Madurez
- Disponibilidad
- Tolerancia a fallos
- Capacidad de recuperación

### 3.6 Seguridad
- Confidencialidad
- Integridad
- No repudio
- Autenticidad
- Responsabilidad

### 3.7 Mantenibilidad
- Modularidad
- Reusabilidad
- Analizabilidad
- Capacidad de ser modificado
- Capacidad de ser probado

### 3.8 Portabilidad
- Adaptabilidad
- Facilidad de instalación
- Capacidad de ser reemplazado

---

## 4. Escenarios de Calidad

Los escenarios de calidad son la forma en que se **especifican los requerimientos** de atributos de calidad. Se componen de los siguientes elementos:

| Elemento | Descripción |
|----------|-------------|
| **Fuente** | Origen del estímulo |
| **Estímulo** | Evento dentro del sistema o dentro del proceso de desarrollo |
| **Entorno** | Contexto/estado en el que se presenta el estímulo |
| **Respuesta** | Cómo el sistema maneja (o permite manejar) el evento |
| **Métrica de la respuesta** | Resultados esperados (o encontrados) tras la aplicación de la respuesta |

---

## 5. Disponibilidad

### Definición
La disponibilidad es la **capacidad del sistema o componente de estar operativo y accesible para su uso cuando se requiere**.

- Está asociada a las fallas del sistema y sus consecuencias.
- Dicha falla es observable por los usuarios del sistema (humanos u otros sistemas).
- También se define como la capacidad de un servicio, datos o sistema de ser accesible y utilizable por usuarios o procesos autorizados cuando estos lo requieran.

### Escenario de Disponibilidad (Ejemplo)
- **Fuente:** Externa al sistema
- **Estímulo:** Mensaje inesperado (Unanticipated Message)
- **Artefacto:** Proceso
- **Entorno:** Operación normal
- **Respuesta:** Informar al operador, continuar operando
- **Métrica:** Sin tiempo de inactividad (No Downtime)

### Escenario General de Disponibilidad

| Elemento | Opciones |
|----------|----------|
| **Source** | Internal / External to system |
| **Stimulus** | Crash, Omission, Timing, No response, Incorrect response |
| **Artifact** | System's processors, Communication channels, Persistent storage |
| **Environment** | Normal operation, Startup, Shutdown, Repair mode, Degraded mode, Overloaded operation |
| **Response** | Prevent failure, Log failure, Notify users/operators, Disable source of failure, Temporarily unavailable, Continue (normal/degraded) |
| **Measure** | Time interval available, Availability %, Detection time, Repair time, Degraded mode time interval, Unavailability time interval |

### Tácticas de Disponibilidad

Las tácticas de disponibilidad se dividen en tres grandes grupos:

**1. Detectar Fallas (Detect Faults)**
- Ping/Echo
- Monitor
- Heartbeat
- Timestamp
- Sanity Checking
- Condition Monitoring
- Voting
- Exception Detection
- Self-Test

**2. Recuperarse de Fallas (Recover from Faults)**

*Preparación y Reparación:*
- Active Redundancy
- Passive Redundancy
- Spare
- Exception Handling
- Rollback
- Software Upgrade
- Retry
- Ignore Faulty Behavior
- Degradation
- Reconfiguration

*Reintroducción:*
- Shadow
- State Resynchronization
- Escalating Restart
- Non-Stop Forwarding

**3. Prevenir Fallas (Prevent Faults)**
- Removal from Service
- Transactions
- Predictive Model
- Exception Prevention
- Increase Competence Set

---

## 6. Modificabilidad

### Definición
La modificabilidad representa la **capacidad del producto software para ser modificado efectiva y eficientemente**, debido a necesidades evolutivas, correctivas o perfectivas.

Está asociada al **costo del cambio**, considerando:
- ¿Qué puede cambiar?
- ¿Cuándo se realiza el cambio y quién lo puede realizar?

### Escenario de Modificabilidad (Ejemplo)
- **Fuente:** Developer (Desarrollador)
- **Estímulo:** Desea cambiar la UI
- **Artefacto:** Código
- **Entorno:** Tiempo de diseño (Design Time)
- **Respuesta:** Cambio realizado y probado con pruebas unitarias
- **Métrica:** En tres horas

### Escenario General de Modificabilidad

| Elemento | Opciones |
|----------|----------|
| **Source** | End-user, Developer, System-administrator |
| **Stimulus** | Add/delete/modify functionality, quality attribute, capacity or technology |
| **Artifact** | Code, Data, Interfaces, Components, Resources, Configurations |
| **Environment** | Runtime, Compile time, Build time, Initiation time, Design time |
| **Response** | Make modification, Test modification, Deploy modification |
| **Measure** | Cost in effort, Cost in money, Cost in time, Cost in number/size/complexity of affected artifacts, Extent affects other system functions or qualities, New defects introduced |

### Tácticas de Modificabilidad

| Táctica Principal | Sub-tácticas |
|------------------|-------------|
| **Reduce Size of a Module** | Split Module |
| **Increase Cohesion** | Increase Semantic Coherence |
| **Reduce Coupling** | Encapsulate, Use an Intermediary, Restrict Dependencies, Refactor, Abstract Common Services |
| **Defer Binding** | — |

---

## 7. Desempeño (Performance)

### Definición
El desempeño representa el **desempeño relativo a la cantidad de recursos utilizados cuando el software lleva a cabo su función bajo condiciones determinadas**.

- Referente a tasas de respuesta, latencia, tiempos límite, tasa de pérdidas de datos, etc., frente a eventos ocurridos en el sistema.
- Los eventos pueden ser **periódicos, esporádicos, o de carácter estocástico**.

### Escenario de Desempeño (Ejemplo)
- **Fuente:** Usuarios
- **Estímulo:** Iniciar transacciones
- **Artefacto:** Sistema
- **Entorno:** Operación normal
- **Respuesta:** Transacciones procesadas
- **Métrica:** Latencia promedio de dos segundos

### Escenario General de Desempeño

| Elemento | Opciones |
|----------|----------|
| **Source** | Internal / External to the system |
| **Stimulus** | Periodic events, Sporadic events, Bursty events, Stochastic events |
| **Artifact** | System, Component |
| **Environment** | Normal mode, Overload mode, Reduced capacity mode, Emergency mode, Peak mode |
| **Response** | Process events, Change level of service |
| **Measure** | Latency, Deadline, Throughput, Jitter, Miss rate, Data loss |

### Tácticas de Desempeño

**1. Control de la Demanda de Recursos (Control Resource Demand)**
- Manage Sampling Rate
- Limit Event Response
- Prioritize Events
- Reduce Overhead
- Bound Execution Times
- Increase Resource Efficiency

**2. Gestión de Recursos (Manage Resources)**
- Increase Resources
- Introduce Concurrency
- Maintain Multiple Copies of Computations
- Maintain Multiple Copies of Data
- Bound Queue Sizes
- Schedule Resources

---

## 8. Seguridad

### Definición
La seguridad es la **capacidad de protección de la información y los datos de manera que personas o sistemas no autorizados no puedan leerlos o modificarlos**.

- Medida de la habilidad del sistema para **resistir usos no autorizados**, sin dejar de prestar el servicio a los usuarios legítimos.
- Ejemplo de uso no autorizado: acceder a datos o servicios para modificarlos, o denegar el servicio a los usuarios legítimos.

### Seguridad – No Repudio (Ejemplo de Escenario)
- **Fuente:** Individuo correctamente identificado
- **Estímulo:** Intenta modificar información
- **Artefacto:** Datos dentro del sistema
- **Entorno:** Bajo operaciones normales
- **Respuesta:** El sistema mantiene una pista de auditoría (Audit Trail)
- **Métrica:** Los datos correctos son restaurados dentro de un día

### Escenario General de Seguridad

| Elemento | Opciones |
|----------|----------|
| **Source** | Identified user, Unknown user, Hacker from outside/inside the organization |
| **Stimulus** | Attempt to display data, Attempt to modify data, Attempt to delete data, Access system services, Change system's behavior, Reduce availability |
| **Artifact** | System services, Data within the system, Component/resource of the system, Data produced/consumed by the system |
| **Environment** | Normal mode, Overload mode, Reduced capacity mode, Emergency mode, Peak mode |
| **Response** | Process events, Change level of service |
| **Measure** | Latency, Deadline, Throughput, Jitter, Miss rate, Data loss |

### Tácticas de Seguridad

**1. Detectar Ataques (Detect Attacks)**
- Detect Intrusion
- Detect Service Denial
- Verify Message Integrity
- Detect Message Delay

**2. Resistir Ataques (Resist Attacks)**
- Identify Actors
- Authenticate Actors
- Authorize Actors
- Limit Access
- Limit Exposure
- Encrypt Data
- Separate Entities
- Change Default Settings

**3. Reaccionar ante Ataques (React to Attacks)**
- Revoke Access
- Lock Computer
- Inform Actors

**4. Recuperarse de Ataques (Recover from Attacks)**
- Maintain Audit Trail
- Restore
- (Ver también tácticas de Disponibilidad)

---

## 9. Testeabilidad

### Definición
La testeabilidad es la **facilidad con la que se pueden establecer criterios de prueba para un sistema o componente y con la que se pueden llevar a cabo las pruebas para determinar si se cumplen dichos criterios**.

- Facilidad con la cual se pueden ejecutar las pruebas en cada fase del desarrollo.
- Facilidad con la cual se pueden implementar pruebas que permitan detectar defectos.

### Escenario de Testeabilidad (Ejemplo)
- **Fuente:** Unit Tester
- **Estímulo:** Realiza prueba unitaria
- **Artefacto:** Componente del sistema
- **Entorno:** Al finalizar el componente
- **Respuesta:** El componente tiene interfaz para controlar comportamiento y salida del componente, siendo observable
- **Métrica:** Cobertura de caminos del 85% lograda en tres horas

### Escenario General de Testeabilidad

| Elemento | Opciones |
|----------|----------|
| **Source** | Unit tester, Integration tester, System tester, Acceptance tester, End user, Automated testing tools |
| **Stimulus** | Execution of tests due to completion of code increment |
| **Artifact** | Portion of the system being tested |
| **Environment** | Design time, Development time, Compile time, Integration time, Deployment time, Run time |
| **Response** | Execute test suite & capture results, Capture cause of fault, Control & monitor state of the system |
| **Measure** | Effort to find fault, Effort to achieve coverage %, Probability of fault being revealed by next test, Time to perform tests, Effort to detect faults, Length of longest dependency chain, Time to prepare test environment, Reduction in risk exposure |

---

## 10. Escalabilidad

### Definición
La escalabilidad es la **propiedad de un sistema de manejar cantidades cada vez mayores de trabajo, o de ser fácilmente expandida, en respuesta a una demanda creciente de tráfico, procesamiento, acceso a base de datos, etc.**

### Conceptos clave: Capas (Layers) vs Niveles (Tiers)

- Los **Niveles (Tiers)** hacen referencia a la separación **física** de componentes del sistema (e.g., Client Tier, Business Tier, Storage Tier).
- Las **Capas (Layers)** hacen referencia a la separación **lógica** dentro de un mismo nivel (e.g., Layer 1, Layer 2, Layer N dentro del Business Tier).

### Tipos de Escalabilidad

#### Escalabilidad Vertical
Un sistema escala **verticalmente o hacia arriba**, cuando al añadir más recursos a un nodo particular del sistema, este mejora en conjunto. Por ejemplo, añadir memoria o un disco duro más rápido a una computadora puede mejorar el rendimiento del sistema global.

#### Escalabilidad Horizontal
Un sistema escala **horizontalmente**, cuando su capacidad puede ser extendida agregando nuevos nodos con una funcionalidad idéntica a la existente, redistribuyéndola entre todas estas.

### Tácticas de Escalabilidad Horizontal

Las principales tácticas para lograr la escalabilidad horizontal son:

#### 1. Balanceo de Carga
Mecanismo que distribuye las solicitudes entrantes entre múltiples nodos del sistema. Algunas estrategias son:
- **Round Robin:** Distribución equitativa y secuencial entre nodos.
- **Por número de conexiones:** Asigna al nodo con menos conexiones activas.
- **Por tiempos de respuesta:** Asigna al nodo con mejor tiempo de respuesta.
- **Por peso (prioridad):** Asigna mayor carga a nodos con mayor capacidad.

#### 2. Clustering
Un **Cluster** es un grupo de sistemas que trabaja de forma conjunta como una única unidad. Cada unidad está consciente de la existencia de las demás. Por lo general, maneja balanceo de carga.

#### 3. Paralelización / Procesamiento Asíncrono
- **Capacidad de cómputo:** Uso de colas de trabajo (procesamiento asíncrono), donde las tareas son enviadas por un productor (aplicación) y consumidas por múltiples consumidores de forma paralela.

### Disponibilidad vs Escalabilidad Horizontal
Al implementar escalabilidad horizontal (ej. balanceo de carga), surge el concepto de **SPA: Single Point of Failure** (Punto único de fallo). Si el balanceador de carga falla, todo el sistema queda inaccesible, lo que afecta directamente la disponibilidad. Por ello, ambos atributos deben considerarse de manera conjunta en el diseño arquitectónico.

---

## Resumen General

| Atributo | Enfoque Principal |
|----------|------------------|
| **Disponibilidad** | El sistema debe estar operativo y accesible cuando se necesite |
| **Modificabilidad** | El sistema debe poder ser cambiado con el menor costo posible |
| **Desempeño** | El sistema debe responder eficientemente bajo distintas condiciones de carga |
| **Seguridad** | El sistema debe proteger la información de accesos no autorizados |
| **Testeabilidad** | El sistema debe facilitar la verificación y detección de defectos |
| **Escalabilidad** | El sistema debe poder crecer para manejar mayor carga de trabajo |