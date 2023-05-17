## 1. Estructura de mensajes

```java
package org.example;

import java.io.Serializable;
import java.util.Date;
import java.util.Map;

public class Messages {
    public static class PedidoMessage implements Serializable {
        private int idPedido;
        private Map<String, Integer> productos;

        public PedidoMessage(int idPedido, Map<String, Integer> productos) {
            this.idPedido = idPedido;
            this.productos = productos;
        }

        public int getIdPedido() {
            return idPedido;
        }

        public Map<String, Integer> getProductos() {
            return productos;
        }
    }

    public static class AlmacenMessage implements Serializable {
        private int idPedido;
        private boolean hayStock;
        private String ubicacionAlmacen;

        public AlmacenMessage() {
        }

        public AlmacenMessage(int idPedido, boolean hayStock, String ubicacionAlmacen) {
            this.idPedido = idPedido;
            this.hayStock = hayStock;
            this.ubicacionAlmacen = ubicacionAlmacen;
        }

        public int getIdPedido() {
            return idPedido;
        }

        public void setIdPedido(int idPedido) {
            this.idPedido = idPedido;
        }

        public boolean isHayStock() {
            return hayStock;
        }

        public void setHayStock(boolean hayStock) {
            this.hayStock = hayStock;
        }

        public String getUbicacionAlmacen() {
            return ubicacionAlmacen;
        }

        public void setUbicacionAlmacen(String ubicacionAlmacen) {
            this.ubicacionAlmacen = ubicacionAlmacen;
        }
    }

    public static class TransporteMessage implements Serializable {
        private int idPedido;
        private Date horaEstimadaEntrega;

        public TransporteMessage(int idPedido, Date horaEstimadaEntrega) {
            this.idPedido = idPedido;
            this.horaEstimadaEntrega = horaEstimadaEntrega;
        }

        public int getIdPedido() {
            return idPedido;
        }

        public Date getHoraEstimadaEntrega() {
            return horaEstimadaEntrega;
        }
    }

    public static class EntregaMessage implements Serializable {
        private int idPedido;
        private boolean entregaExitosa;

        public EntregaMessage(int idPedido, boolean entregaExitosa) {
            this.idPedido = idPedido;
            this.entregaExitosa = entregaExitosa;
        }

        public int getIdPedido() {
            return idPedido;
        }

        public boolean isEntregaExitosa() {
            return entregaExitosa;
        }
    }
}
```

## 2. Procesos

### 2.1. Proceso de Pedido


```java
package org.example;

import javax.jms.*;
import java.util.Map;
import org.example.Messages.*;
public class ProcesoPedido implements MessageListener {

    private ConnectionFactory connectionFactory;
    private Destination destinationAlmacen;
    private Destination destinationPedido;

    public ProcesoPedido(ConnectionFactory connectionFactory, Destination destinationAlmacen, Destination destinationPedido) {
        this.connectionFactory = connectionFactory;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationPedido = destinationPedido;
    }

    public void iniciar() throws JMSException {
        Connection connection = connectionFactory.createConnection();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(destinationPedido);
        consumer.setMessageListener(this);
        connection.start();
    }

    public void recibirPedido(int idPedido, Map<String, Integer> productos) throws JMSException {
        Connection connection = connectionFactory.createConnection();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageProducer producer = session.createProducer(destinationAlmacen);

        PedidoMessage pedido = new PedidoMessage(idPedido, productos);
        ObjectMessage objectMessage = session.createObjectMessage(pedido);
        producer.send(objectMessage);
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                Object obj = objectMessage.getObject();

                if (obj instanceof AlmacenMessage) {
                    AlmacenMessage almacenMessage = (AlmacenMessage) obj;
                    // Procesar mensaje de falta de inventario
                } else if (obj instanceof EntregaMessage) {
                    EntregaMessage entregaMessage = (EntregaMessage) obj;
                    // Procesar confirmación de entrega
                }
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}

```

### 2.2. Proceso de Almacén

```java
package org.example;

import javax.jms.*;
import java.util.Map;
import org.example.Messages.*;

public class ProcesoAlmacen implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationPedido;
    private Destination destinationAlmacen;
    private Destination destinationTransporte;
    private Connection connection;
    private Session session;
    private MessageProducer producerPedido;
    private MessageProducer producerTransporte;

    public ProcesoAlmacen(ConnectionFactory connectionFactory, Destination destinationPedido, Destination destinationAlmacen, Destination destinationTransporte) {
        this.connectionFactory = connectionFactory;
        this.destinationPedido = destinationPedido;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationTransporte = destinationTransporte;
    }

    public void init() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(destinationAlmacen);
        consumer.setMessageListener(this);
        producerPedido = session.createProducer(destinationPedido);
        producerTransporte = session.createProducer(destinationTransporte);
        connection.start();
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                PedidoMessage pedidoMessage = (PedidoMessage) objectMessage.getObject();
                Map<String, Integer> productosPedido = pedidoMessage.getProductos();

                if (verificarDisponibilidad(productosPedido)) {
                    AlmacenMessage almacenMessage = new AlmacenMessage();
                    almacenMessage.setIdPedido(pedidoMessage.getIdPedido());
                    almacenMessage.setHayStock(true);
                    almacenMessage.setUbicacionAlmacen("Almacén Principal"); // Aquí se debe poner la ubicación real del almacén

                    ObjectMessage respuestaMessage = session.createObjectMessage(almacenMessage);
                    producerTransporte.send(respuestaMessage);
                } else {
                    AlmacenMessage almacenMessage = new AlmacenMessage();
                    almacenMessage.setIdPedido(pedidoMessage.getIdPedido());
                    almacenMessage.setHayStock(false);

                    ObjectMessage respuestaMessage = session.createObjectMessage(almacenMessage);
                    producerPedido.send(respuestaMessage);
                }
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    private boolean verificarDisponibilidad(Map<String, Integer> productosPedido) {
        // Aquí debes implementar la lógica para verificar la disponibilidad de los productos en el almacén
        // Esta es una implementación simplificada y siempre devuelve true
        return true;
    }
}

```

### 2.3. Proceso de Transporte

```java
package org.example;

import javax.jms.*;
import org.example.Messages.*;
public class ProcesoTransporte implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationAlmacen;
    private Destination destinationEntrega;
    private Connection connection;
    private Session session;
    private MessageProducer producerEntrega;

    public ProcesoTransporte(ConnectionFactory connectionFactory, Destination destinationAlmacen, Destination destinationEntrega) {
        this.connectionFactory = connectionFactory;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationEntrega = destinationEntrega;
    }

    public void init() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(destinationAlmacen);
        consumer.setMessageListener(this);
        producerEntrega = session.createProducer(destinationEntrega);
        connection.start();
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                AlmacenMessage almacenMessage = (AlmacenMessage) objectMessage.getObject();
                if (almacenMessage.isHayStock()) {
                    // Aquí debes implementar la lógica para calcular la hora estimada de entrega
                    Date horaEstimadaEntrega = new Date();
                    TransporteMessage transporteMessage = new TransporteMessage(almacenMessage.getIdPedido(), horaEstimadaEntrega);
                    ObjectMessage respuestaMessage = session.createObjectMessage(transporteMessage);
                    producerEntrega.send(respuestaMessage);
                }
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}


```

### 2.4. Proceso de Entrega

```java
package org.example;

import javax.jms.*;
import org.example.Messages.*;

public class ProcesoEntrega implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationTransporte;
    private Destination destinationPedido;
    private Connection connection;
    private Session session;
    private MessageProducer producerPedido;

    public ProcesoEntrega(ConnectionFactory connectionFactory, Destination destinationTransporte, Destination destinationPedido) {
        this.connectionFactory = connectionFactory;
        this.destinationTransporte = destinationTransporte;
        this.destinationPedido = destinationPedido;
    }

    public void init() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(destinationTransporte);
        consumer.setMessageListener(this);
        producerPedido = session.createProducer(destinationPedido);
        connection.start();
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                TransporteMessage transporteMessage = (TransporteMessage) objectMessage.getObject();
                // Aquí debes implementar la lógica para realizar la entrega del pedido
                boolean entregaExitosa = true;
                EntregaMessage entregaMessage = new EntregaMessage(transporteMessage.getIdPedido(), entregaExitosa);
                ObjectMessage respuestaMessage = session.createObjectMessage(entregaMessage);
                producerPedido.send(respuestaMessage);
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}

```
