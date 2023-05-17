# Tercera Práctica

## Resolución con paso de mensajes

Para la resolución de análisis y diseño se deberán utilizar paso de mensajes asíncronos como herramienta para la
programación concurrente.

Para la implementación de la práctica se utilizará como herramienta de concurrencia JMS (Java Message Service). Esta
práctica es una práctica en grupo de hasta dos alumnos y cada grupo deberá crear en el _broker_ sus propios _destinos_
para sus mensajes. Los miembros del grupo no tienen que pertenecer al mismo grupo de prácticas. Cada destino deberá
definirse siguiendo la siguiente estructura:

```
// En la interface Constantes del proyecto
public static final String DESTINO = "ssccdd.curso2023.NOMBRE_GRUPO.BUZON";
```

El nombre del grupo tiene que ser único para los grupos, por lo que se recomienda usar alguna combinación de los nombres
de los integrantes del grupo.

## Problema a resolver: Empresa de Logística

En una empresa de logística, hay cuatro tipos de procesos diferentes que participan en la cadena de suministro: Proceso
de Pedido, Proceso de Almacén, Proceso de Transporte y Proceso de Entrega. Estos procesos deben comunicarse entre sí
mediante el uso de paso de mensajes asíncronos y se implementará mediante JMS (Java Message Service).

### Objetivo:

El objetivo de este ejercicio es diseñar e implementar una solución que permita a estos cuatro procesos colaborar de
manera efectiva utilizando el paso de mensajes como herramienta de concurrencia. Deben asegurarse de que los mensajes se
envíen y reciban de manera asíncrona y que los procesos se sincronicen de forma correcta.

### Restricciones:

1. El Proceso de Pedido es el que recibe pedidos de los clientes y envía un mensaje asincrónico al Proceso de Almacén
   con la información del pedido (ID de pedido, productos y cantidades).

2. El Proceso de Almacén tiene que recibir mensajes del Proceso de Pedido y verifica si los productos del pedido están
   disponibles en el almacén. Si los productos están disponibles, envía un mensaje asincrónico al Proceso de Transporte
   con la información del pedido y la ubicación del almacén. Si no hay suficientes productos en el almacén, envía un
   mensaje asincrónico de vuelta al Proceso Pedido notificando la falta de inventario.

3. El Proceso de Transporte recibe mensajes del Proceso de Almacén y organiza el transporte del pedido. Una vez que el
   transporte está listo, envía un mensaje asincrónico al Proceso de Entrega con la información del pedido y la hora
   estimada de entrega.

4. El Proceso de Entrega recibe el mensajes del Proceso de Transporte y realiza la entrega del pedido al cliente. Una
   vez que la entrega ha sido completada, envía un mensaje asincrónico de vuelta al Proceso Pedido confirmando la
   entrega exitosa del pedido.

5. Hay que justificar adecuadamente las decisiones adoptadas en la solución propuesta. Cada miembro del grupo deberá
   encargarse de dos de los procesos implicados en el problema.

6. Asegurar la sincronización correcta de los procesos para completar los pedidos que se solicienten por parte del
   usuario.

---

# Análisis del problema:

## Contexto:

El problema plantea una situación en la que se debe simular el funcionamiento de una empresa de logística utilizando la
programación concurrente y el paso de mensajes asíncronos como herramienta. Se utilizará Java Message Service (JMS) para
la implementación de la solución.

## Objetivos:

- Diseñar e implementar una solución que permita la colaboración efectiva entre los cuatro procesos de la cadena de
  suministro.
- Asegurar la comunicación asíncrona y la sincronización correcta de los procesos.

## Procesos involucrados:

1. Proceso de Pedido
2. Proceso de Almacén
3. Proceso de Transporte
4. Proceso de Entrega

## Restricciones y requerimientos de cada proceso:

1. Proceso de Pedido:
    - Recibir pedidos de los clientes.
    - Enviar un mensaje asíncrono al Proceso de Almacén con la información del pedido (ID de pedido, productos y
      cantidades).
    - Recibir mensajes de falta de inventario del Proceso de Almacén.
    - Recibir mensajes de confirmación de entrega exitosa del Proceso de Entrega.

2. Proceso de Almacén:
    - Recibir mensajes del Proceso de Pedido.
    - Verificar la disponibilidad de productos en el almacén.
    - Si hay suficiente inventario, enviar un mensaje asíncrono al Proceso de Transporte con la información del pedido y
      la ubicación del almacén.
    - Si no hay suficiente inventario, enviar un mensaje asíncrono al Proceso de Pedido notificando la falta de
      inventario.

3. Proceso de Transporte:
    - Recibir mensajes del Proceso de Almacén.
    - Organizar el transporte del pedido.
    - Enviar un mensaje asíncrono al Proceso de Entrega con la información del pedido y la hora estimada de entrega.

4. Proceso de Entrega:
    - Recibir mensajes del Proceso de Transporte.
    - Realizar la entrega del pedido al cliente.
    - Enviar un mensaje asíncrono al Proceso de Pedido confirmando la entrega exitosa del pedido.

Cada miembro del grupo debe encargarse de dos de los procesos involucrados en el problema. Además, se debe justificar
adecuadamente las decisiones adoptadas en la solución propuesta.

En resumen, este problema implica el diseño e implementación de una solución que permita la colaboración eficiente entre
cuatro procesos de una empresa de logística utilizando la programación concurrente y paso de mensajes asíncronos con
JMS. Es crucial garantizar la comunicación asíncrona y la sincronización adecuada de los procesos para completar los
pedidos solicitados por el usuario.

---

# Diseño de la solución

## 1. Estructura de mensajes

Primero, debemos definir las estructuras de mensajes que se utilizarán para la comunicación entre los procesos. Estos
mensajes serán objetos serializables y se pueden definir como clases en Java.

```java
class PedidoMessage implements Serializable {
    private int idPedido;
    private Map<String, Integer> productos;
    // Constructor, getters y setters
}

class AlmacenMessage implements Serializable {
    private int idPedido;
    private boolean hayStock;
    private String ubicacionAlmacen;
    // Constructor, getters y setters
}

class TransporteMessage implements Serializable {
    private int idPedido;
    private Date horaEstimadaEntrega;
    // Constructor, getters y setters
}

class EntregaMessage implements Serializable {
    private int idPedido;
    private boolean entregaExitosa;
    // Constructor, getters y setters
}
```

## 2. Implementación de los procesos

Cada proceso se implementará como una clase en Java. Los cuatro procesos
son `ProcesoPedido`, `ProcesoAlmacen`, `ProcesoTransporte` y `ProcesoEntrega`. Cada proceso tendrá una conexión JMS y un
buzón (destino) para recibir mensajes. Además, se encargará de enviar mensajes a los destinos correspondientes de otros
procesos.

### 2.1. Proceso de Pedido

El `ProcesoPedido` será responsable de recibir pedidos de los clientes y enviar mensajes al `ProcesoAlmacen`. Además,
debe recibir mensajes del `ProcesoAlmacen` sobre la falta de inventario y mensajes del `ProcesoEntrega` confirmando la
entrega exitosa del pedido.

```java
class ProcesoPedido {
    private ConnectionFactory connectionFactory;
    private Destination destinationAlmacen;
    private Destination destinationPedido;
    // Constructor e inicialización de la conexión JMS y destinos

    public void recibirPedido(int idPedido, Map<String, Integer> productos) {
        // Crear mensaje de pedido y enviar al ProcesoAlmacen
    }

    public void onMessage(Message message) {
        // Procesar mensajes de falta de inventario y confirmación de entrega
    }
}
```

### 2.2. Proceso de Almacén

El `ProcesoAlmacen` debe recibir mensajes del `ProcesoPedido`, verificar la disponibilidad de productos en el almacén y
enviar mensajes al `ProcesoTransporte` o al `ProcesoPedido` según corresponda.

```java
class ProcesoAlmacen {
    private ConnectionFactory connectionFactory;
    private Destination destinationPedido;
    private Destination destinationAlmacen;
    private Destination destinationTransporte;
    // Constructor e inicialización de la conexión JMS y destinos

    public void onMessage(Message message) {
        // Procesar mensajes del ProcesoPedido y verificar disponibilidad de productos
        // Enviar mensajes al ProcesoTransporte o ProcesoPedido según corresponda
    }
}
```

### 2.3. Proceso de Transporte

El `ProcesoTransporte` debe recibir mensajes del `ProcesoAlmacen`, organizar el transporte del pedido y enviar mensajes
al `ProcesoEntrega`.

```java
class ProcesoTransporte {
    private ConnectionFactory connectionFactory;
    private Destination destinationTransporte;
    private Destination destinationEntrega;
    // Constructor e inicialización de la conexión JMS y destinos

    public void onMessage(Message message) {
        // Procesar mensajes del ProcesoAlmacen y organizar el transporte
        // Enviar mensajes al ProcesoEntrega
    }
}
```

### 2.4. Proceso de Entrega

El `ProcesoEntrega` debe recibir mensajes del `ProcesoTransporte`, realizar la entrega del pedido al cliente y enviar
mensajes al `ProcesoPedido` confirmando la entrega exitosa del pedido.

```java
class ProcesoEntrega {
    private ConnectionFactory connectionFactory;
    private Destination destinationEntrega;
    private Destination destinationPedido;
    // Constructor e inicialización de la conexión JMS y destinos

    public void onMessage(Message message) {
        // Procesar mensajes del ProcesoTransporte y realizar la entrega del pedido
        // Enviar mensajes al ProcesoPedido confirmando la entrega exitosa
    }
}
```

## 3. Integración y sincronización de procesos

Para garantizar la correcta sincronización de los procesos y completar los pedidos solicitados por el usuario, es
necesario implementar los siguientes pasos:

1. Crear instancias de `ProcesoPedido`, `ProcesoAlmacen`, `ProcesoTransporte` y `ProcesoEntrega`.
2. Iniciar las conexiones JMS y destinos en cada proceso.
3. Configurar los procesos para que se suscriban a sus respectivos destinos y escuchen los mensajes entrantes.
4. Implementar la lógica para procesar los mensajes entrantes en cada proceso y enviar mensajes a los destinos
   correspondientes.
5. Asegurar que los mensajes se envíen y reciban de manera asíncrona y que los procesos se sincronicen de forma
   correcta.

## 4. Justificación de las decisiones

- Se implementaron cuatro clases de mensajes serializables para facilitar la comunicación entre los procesos,
  permitiendo así una mayor flexibilidad y reutilización del código.
- La sincronización de los procesos se garantiza mediante el uso de mensajes asíncronos y la suscripción a los destinos
  correspondientes, lo que evita el bloqueo de los procesos mientras esperan mensajes entrantes.
