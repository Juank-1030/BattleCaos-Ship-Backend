# Informe Detallado: Escenarios de Atributos de Calidad

**Documento analizado:** *Escenarios de Calidad* 

---

# Tabla de Contenido

1. [Introducción](#introducción)
2. [¿Qué son los Escenarios de Calidad?](#qué-son-los-escenarios-de-calidad)
3. [Las 6 partes de un Escenario de Calidad](#las-6-partes-de-un-escenario-de-calidad)
4. [Escenarios como Casos de Prueba](#escenarios-como-casos-de-prueba)
5. [Escenario de Desempeño](#escenario-de-desempeño)
6. [Escenario de Seguridad](#escenario-de-seguridad)
7. [Escenario de Disponibilidad](#escenario-de-disponibilidad)
8. [Escenario de Modificabilidad](#escenario-de-modificabilidad)
9. [Escalas de Medición](#escalas-de-medición)
10. [Métricas de Disponibilidad](#métricas-de-disponibilidad)
11. [Métricas de Eficiencia](#métricas-de-eficiencia)
12. [Métricas de Seguridad](#métricas-de-seguridad)
13. [Métricas de Usabilidad](#métricas-de-usabilidad)
14. [Mínimo, Objetivo y Excepcional](#mínimo-objetivo-y-excepcional)
15. [Glosario](#glosario)

---

# Introducción

En Arquitectura de Software no basta con que un sistema funcione correctamente. También debe ser:

* Rápido
* Seguro
* Fácil de modificar
* Disponible
* Escalable
* Fácil de usar

Estas características reciben el nombre de **atributos de calidad**.

El documento presenta una técnica para especificarlos mediante **Escenarios de Calidad**, permitiendo convertir requisitos no funcionales en especificaciones concretas y medibles. 

---

# ¿Qué son los Escenarios de Calidad?

Un escenario de calidad es una descripción estructurada de cómo debe reaccionar un sistema ante un evento determinado.

Según el documento, cada escenario está compuesto por **6 elementos fundamentales**. 

La idea principal es responder:

> ¿Qué ocurre?
>
> ¿Quién lo provoca?
>
> ¿Cómo responde el sistema?
>
> ¿Cómo medimos si respondió correctamente?

---

# Las 6 partes de un Escenario de Calidad

## 1. Fuente de Estímulo

Es la entidad que genera el evento.

Puede ser:

* Un usuario
* Otro sistema
* Un atacante
* Un dispositivo externo
* Un desarrollador



### Ejemplo

```text
Un usuario realiza una búsqueda.
```

Fuente de estímulo:

```text
Usuario
```

---

## 2. Estímulo

Es el evento que llega al sistema. 

### Ejemplo

```text
Realiza una búsqueda de productos.
```

---

## 3. Artefacto

Es la parte del sistema afectada por el estímulo. 

### Ejemplos

```text
Base de datos
API REST
Pantalla Login
Servidor Web
Microservicio
```

---

## 4. Entorno

Condiciones bajo las cuales ocurre el evento. 

### Ejemplos

```text
Operación normal
Alta carga
Modo mantenimiento
Producción
Desarrollo
```

---

## 5. Respuesta

Acción realizada por el sistema después del estímulo. 

### Ejemplo

```text
Muestra resultados al usuario.
```

---

## 6. Medida de Respuesta

Valor cuantificable para verificar el cumplimiento. 

### Ejemplos

```text
Menos de 3 segundos
99.9% de disponibilidad
Máximo 2 clases modificadas
```

---

# Escenarios como Casos de Prueba

El documento menciona que un escenario puede verse como un caso de prueba. 

| Caso de Prueba       | Escenario            |
| -------------------- | -------------------- |
| Entradas             | Fuente + Estímulo    |
| Condiciones          | Entorno              |
| Resultados Esperados | Respuesta + Medición |

---

# Escenario de Desempeño

## Definición

Mide la rapidez con la que responde el sistema.

Ejemplo del documento:

> Un usuario consulta el catálogo y el resultado debe mostrarse en menos de 3 segundos. 

---

## Implementación en Código

### Mala práctica

```java
@GetMapping("/productos")
public List<Producto> obtenerProductos() {

    try {
        Thread.sleep(5000);
    } catch(Exception e){}

    return productoService.findAll();
}
```

Tiempo:

```text
5 segundos
```

No cumple.

---

## Buena práctica

```java
@GetMapping("/productos")
public List<Producto> obtenerProductos() {

    return productoRepository.findTop100();
}
```

---

## Uso de Caché

```java
@Cacheable("productos")
public List<Producto> obtenerProductos() {

    return productoRepository.findAll();
}
```

### ¿Qué hace?

Primera petición:

```text
Consulta BD
```

Segunda petición:

```text
Consulta caché
```

Resultado:

```text
Respuesta mucho más rápida
```

---

## Medición

```java
long inicio = System.currentTimeMillis();

productoService.obtenerProductos();

long fin = System.currentTimeMillis();

System.out.println(fin - inicio);
```

Salida:

```text
285 ms
```

Cumple el escenario.

---

# Escenario de Seguridad

El documento presenta un escenario donde un usuario realiza varios intentos fallidos y después de tres intentos se bloquea la IP. 

---

## Implementación Básica

```java
public class LoginService {

    private Map<String, Integer> intentos = new HashMap<>();

    public boolean login(
        String ip,
        String usuario,
        String password
    ) {

        if(esBloqueada(ip)) {
            throw new RuntimeException(
                "IP bloqueada"
            );
        }

        if(validar(usuario,password)) {
            intentos.remove(ip);
            return true;
        }

        intentos.put(
            ip,
            intentos.getOrDefault(ip,0)+1
        );

        return false;
    }

    private boolean esBloqueada(String ip){

        return intentos.getOrDefault(ip,0) >= 3;
    }
}
```

---

## Registro en Bitácora

```java
logger.warn(
    "Intento fallido desde IP {}",
    ip
);
```

---

## Resultado esperado

```text
Intento 1 -> Permitido
Intento 2 -> Permitido
Intento 3 -> Permitido
Intento 4 -> Bloqueado
```

---

# Escenario de Disponibilidad

El documento plantea una falla en un dispositivo I/O donde el sistema debe continuar funcionando. 

---

## Concepto

Disponibilidad significa:

```text
El sistema sigue funcionando
aunque fallen algunos componentes.
```

---

## Ejemplo de Código

### Sin tolerancia a fallos

```java
public void imprimirFactura() {

    impresora.imprimir();
}
```

Si la impresora falla:

```text
Toda la operación falla.
```

---

### Con tolerancia a fallos

```java
public void imprimirFactura() {

    try {

        impresora.imprimir();

    } catch(Exception e){

        logger.error(
            "Impresora no disponible"
        );

        guardarPDF();
    }
}
```

Ahora:

```text
Si falla la impresora
se genera PDF.
```

El sistema sigue operativo.

---

# Escenario de Modificabilidad

El documento plantea:

> Un desarrollador agrega un nuevo caso de uso modificando máximo dos clases. 

---

## Mala Arquitectura

```java
public class Sistema {

    public void procesarPago() {}

    public void generarFactura() {}

    public void generarReporte() {}

    public void exportarExcel() {}

    public void enviarCorreo() {}

}
```

Problema:

```text
Cada nueva funcionalidad
modifica la misma clase.
```

---

## Aplicando SOLID

```java
public interface Reporte {

    void generar();
}
```

---

```java
public class ReportePDF
implements Reporte {

    @Override
    public void generar() {

        System.out.println(
            "PDF"
        );
    }
}
```

---

```java
public class ReporteExcel
implements Reporte {

    @Override
    public void generar() {

        System.out.println(
            "Excel"
        );
    }
}
```

Agregar un nuevo reporte:

```java
public class ReporteWord
implements Reporte {

    @Override
    public void generar() {

        System.out.println(
            "Word"
        );
    }
}
```

Solo se agrega una clase.

La arquitectura es modificable.

---

# Escalas de Medición

El documento menciona tres tipos de escalas. 

---

## Escala Natural

Medidas directas.

Ejemplos:

```text
Tiempo en segundos
Cantidad de usuarios
Memoria usada
```

Código:

```java
long tiempo = 2500;
```

---

## Escala Construida

Se crea una escala artificial.

Ejemplo:

```text
Satisfacción:
1 a 10
```

Código:

```java
int satisfaccion = 8;
```

---

## Escala Proxy

Se mide algo indirectamente.

Ejemplo:

```text
IMC
MTTF
```

Código:

```java
double imc = peso / (altura * altura);
```

---

# Métricas de Disponibilidad

Según el documento: 

## Tiempo de reparación

```text
Máximo 5 minutos
```

---

## Disponibilidad

Fórmula:

Disponibilidad=\frac{Tiempo\ Operativo}{Tiempo\ Total}\times100

Ejemplo:

```text
Sistema operativo:
99 horas

Tiempo total:
100 horas
```

Resultado:

```text
99%
```

---

# Métricas de Eficiencia

Miden consumo de recursos. 

---

## Memoria

```java
Runtime runtime =
Runtime.getRuntime();

long memoria =
runtime.totalMemory()
- runtime.freeMemory();
```

---

## CPU

```java
OperatingSystemMXBean os =
ManagementFactory
.getOperatingSystemMXBean();
```

---

## Throughput

```text
500 peticiones/segundo
```

---

# Métricas de Seguridad

El documento menciona: 

* Probabilidad de detectar ataques
* Tiempo para recuperarse
* Número de ataques resistidos
* Recursos disponibles tras un ataque

---

## Ejemplo

```java
if(intentos > 3){

    bloquearUsuario();
}
```

---

## Detección de Ataques

```java
if(numeroPeticiones > 1000){

    activarAlarma();
}
```

---

# Métricas de Usabilidad

El documento propone medir: 

* Tiempo de ejecución
* Errores reportados
* Problemas resueltos
* Satisfacción

---

## Ejemplo

```java
long inicio = System.nanoTime();

realizarCompra();

long fin = System.nanoTime();
```

---

## Encuesta

```java
int satisfaccionUsuario = 9;
```

Escala:

```text
1 = Muy Malo
10 = Excelente
```

---

# Mínimo, Objetivo y Excepcional

El documento define tres niveles de calidad. 

| Nivel       | Significado      |
| ----------- | ---------------- |
| Mínimo      | Lo indispensable |
| Objetivo    | Lo esperado      |
| Excepcional | Lo ideal         |

---

## Ejemplo

Tiempo de respuesta:

```text
Mínimo: 10 segundos
Objetivo: 9 segundos
Excepcional: 7 segundos
```

Interpretación:

```text
>10 segundos = Falla

7-10 segundos = Cumple

<7 segundos = Excelente
```

---

# Glosario

## A

### [Atributo de Calidad](#qué-son-los-escenarios-de-calidad)

Característica no funcional que define la calidad del sistema.

Ejemplos:

* Seguridad
* Rendimiento
* Disponibilidad

---

## D

### [Desempeño](#escenario-de-desempeño)

Capacidad del sistema para responder rápidamente.

---

### [Disponibilidad](#escenario-de-disponibilidad)

Capacidad de mantenerse operativo ante fallos.

---

## E

### [Escenario de Calidad](#qué-son-los-escenarios-de-calidad)

Descripción estructurada de un atributo de calidad.

---

### [Escala Natural](#escalas-de-medición)

Medida directa como tiempo o memoria.

---

### [Escala Construida](#escalas-de-medición)

Escala diseñada específicamente para una medición.

---

### [Escala Proxy](#escalas-de-medición)

Medición indirecta de una característica.

---

## M

### [Modificabilidad](#escenario-de-modificabilidad)

Facilidad para agregar cambios al sistema.

---

### [Métrica](#métricas)

Valor cuantificable para verificar calidad.

---

## S

### [Seguridad](#escenario-de-seguridad)

Protección frente a accesos o acciones no autorizadas.

---

### [Fuente de Estímulo](#las-6-partes-de-un-escenario-de-calidad)

Entidad que genera el evento.

---

### [Estímulo](#las-6-partes-de-un-escenario-de-calidad)

Evento que recibe el sistema.

---

### [Respuesta](#las-6-partes-de-un-escenario-de-calidad)

Acción ejecutada por el sistema.

---

# Conclusión

La idea central del documento es que los **atributos de calidad deben expresarse como escenarios medibles**. Un requisito como:

> "El sistema debe ser rápido"

es ambiguo.

Mientras que:

> "Un usuario realiza una búsqueda y el sistema responde en menos de 3 segundos"

es verificable, medible y puede convertirse directamente en pruebas automáticas, métricas y decisiones arquitectónicas. 

Por eso los escenarios de calidad son una de las herramientas más utilizadas en Arquitectura de Software para definir requisitos no funcionales de forma precisa.
