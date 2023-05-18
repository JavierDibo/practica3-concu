## 1. Main
```java
public class Main {
    public static void main(String[] args) {
        System.setProperty("org.apache.activemq.SERIALIZABLE_PACKAGES","*");

        try {
            // Crear la ConnectionFactory y los Destinations
            ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://suleiman.ujaen.es:8018");
            Destination destinationPedido = new ActiveMQQueue("ssccdd.curso2023.DiboFernandez.Pedido");
            Destination destinationAlmacen = new ActiveMQQueue("ssccdd.curso2023.DiboFernandez.Almacen");
            Destination destinationTransporte = new ActiveMQQueue("ssccdd.curso2023.DiboFernandez.Transporte");
            Destination destinationEntrega = new ActiveMQQueue("ssccdd.curso2023.DiboFernandez.Entrega");

            // Crear instancias de los procesos
            ProcesoPedido procesoPedido = new ProcesoPedido(connectionFactory, destinationAlmacen, destinationPedido);
            ProcesoAlmacen procesoAlmacen = new ProcesoAlmacen(connectionFactory, destinationPedido, destinationAlmacen, destinationTransporte);
            ProcesoTransporte procesoTransporte = new ProcesoTransporte(connectionFactory, destinationAlmacen, destinationEntrega);
            ProcesoEntrega procesoEntrega = new ProcesoEntrega(connectionFactory, destinationTransporte, destinationPedido);

            // Iniciar los procesos
            System.out.println("Starting processes...");
            procesoPedido.iniciar();
            procesoAlmacen.iniciar();
            procesoTransporte.iniciar();
            procesoEntrega.iniciar();

            // Simular un pedido
            System.out.println("Simulating order...");
            Map<String, Integer> productos = new HashMap<>();
            productos.put("Producto1", 10);
            procesoPedido.recibirPedido(1, productos);

            // Wait for the processes to finish
            System.out.println("Waiting for processes to finish...");
            Thread.sleep(500);

            procesoPedido.terminar();
            procesoAlmacen.terminar();
            procesoTransporte.terminar();
            procesoEntrega.terminar();

            System.out.println("Main process finished.");

        } catch (JMSException e) {
            System.out.println("Caught a JMSException: " + e.getMessage());
            e.printStackTrace();
        } catch (InterruptedException e) {
            System.out.println("Caught an InterruptedException: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

## 2. Procesos

### 2.1. Proceso de Pedido

```java
public class ProcesoPedido implements MessageListener {

    private ConnectionFactory connectionFactory;
    private Destination destinationAlmacen;
    private Destination destinationPedido;
    private Connection connection;
    private Session session;
    private MessageConsumer consumer;
    private MessageProducer producer;

    public ProcesoPedido(ConnectionFactory connectionFactory, Destination destinationAlmacen, Destination destinationPedido) {
        this.connectionFactory = connectionFactory;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationPedido = destinationPedido;
    }

    public void iniciar() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        consumer = session.createConsumer(destinationPedido);
        consumer.setMessageListener(this);
        connection.start();
    }

    public void terminar() throws JMSException {
        consumer.close();
        session.close();
        connection.close();
    }

    public void recibirPedido(int idPedido, Map<String, Integer> productos) throws JMSException {
        producer = session.createProducer(destinationAlmacen);
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
                    if (!almacenMessage.isHayStock()) {
                        System.out.println("No hay suficiente inventario para el pedido: " + almacenMessage.getIdPedido());
                    }
                } else if (obj instanceof EntregaMessage) {
                    EntregaMessage entregaMessage = (EntregaMessage) obj;
                    if (entregaMessage.isEntregaExitosa()) {
                        System.out.println("El pedido ha sido entregado exitosamente: " + entregaMessage.getIdPedido());
                    } else {
                        System.out.println("La entrega del pedido ha fallado: " + entregaMessage.getIdPedido());
                    }
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
public class ProcesoAlmacen implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationPedido;
    private Destination destinationAlmacen;
    private Destination destinationTransporte;
    private Connection connection;
    private Session session;
    private MessageProducer producerPedido;
    private MessageProducer producerTransporte;
    private MessageConsumer consumer;
    private  int cantitdadAlmacen = CANTIDAD_INICIAL;

    public ProcesoAlmacen(ConnectionFactory connectionFactory, Destination destinationPedido, Destination destinationAlmacen, Destination destinationTransporte) {
        this.connectionFactory = connectionFactory;
        this.destinationPedido = destinationPedido;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationTransporte = destinationTransporte;
    }

    public void iniciar() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        consumer = session.createConsumer(destinationPedido);
        consumer.setMessageListener(this);
        producerPedido = session.createProducer(destinationPedido);
        producerTransporte = session.createProducer(destinationTransporte);
        connection.start();
    }

    public void terminar() throws JMSException {
        consumer.close();
        producerPedido.close();
        producerTransporte.close();
        session.close();
        connection.close();
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
                    almacenMessage.setUbicacionAlmacen("Almacén Principal");

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
        for (Map.Entry<String, Integer> producto : productosPedido.entrySet()) {
            if (cantitdadAlmacen < producto.getValue()) {
                return false;
            }
        }
        return true;
    }

    private static final int CANTIDAD_INICIAL = 10;
}
```

### 2.3. Proceso de Transporte

```java
public class ProcesoTransporte implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationAlmacen;
    private Destination destinationEntrega;
    private Connection connection;
    private Session session;
    private MessageProducer producerEntrega;
    private MessageConsumer consumer;

    public ProcesoTransporte(ConnectionFactory connectionFactory, Destination destinationAlmacen, Destination destinationEntrega) {
        this.connectionFactory = connectionFactory;
        this.destinationAlmacen = destinationAlmacen;
        this.destinationEntrega = destinationEntrega;
    }

    public void iniciar() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        consumer = session.createConsumer(destinationAlmacen);
        consumer.setMessageListener(this);
        producerEntrega = session.createProducer(destinationEntrega);
        connection.start();
    }

    public void terminar() throws JMSException {
        consumer.close();
        producerEntrega.close();
        session.close();
        connection.close();
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                if (objectMessage.getObject() instanceof AlmacenMessage) {
                    AlmacenMessage almacenMessage = (AlmacenMessage) objectMessage.getObject();
                    if (almacenMessage.isHayStock()) {
                        Date horaEstimadaEntrega = calcularHoraEstimadaEntrega();
                        TransporteMessage transporteMessage = new TransporteMessage(almacenMessage.getIdPedido(), horaEstimadaEntrega);
                        ObjectMessage respuestaMessage = session.createObjectMessage(transporteMessage);
                        producerEntrega.send(respuestaMessage);
                    }
                }
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }


    private Date calcularHoraEstimadaEntrega() {
        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.HOUR, 1);
        return calendar.getTime();
    }
}
```

### 2.4. Proceso de Entrega

```java
public class ProcesoEntrega implements MessageListener {
    private ConnectionFactory connectionFactory;
    private Destination destinationTransporte;
    private Destination destinationPedido;
    private Connection connection;
    private Session session;
    private MessageProducer producerPedido;
    private MessageConsumer consumer;

    public ProcesoEntrega(ConnectionFactory connectionFactory, Destination destinationTransporte, Destination destinationPedido) {
        this.connectionFactory = connectionFactory;
        this.destinationTransporte = destinationTransporte;
        this.destinationPedido = destinationPedido;
    }

    public void iniciar() throws JMSException {
        connection = connectionFactory.createConnection();
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        consumer = session.createConsumer(destinationTransporte);
        consumer.setMessageListener(this);
        producerPedido = session.createProducer(destinationPedido);
        connection.start();
    }

    public void terminar() throws JMSException {
        consumer.close();
        producerPedido.close();
        session.close();
        connection.close();
    }

    @Override
    public void onMessage(Message message) {
        if (message instanceof ObjectMessage) {
            try {
                ObjectMessage objectMessage = (ObjectMessage) message;
                TransporteMessage transporteMessage = (TransporteMessage) objectMessage.getObject();
                boolean entregaExitosa = realizarEntrega(transporteMessage.getIdPedido());
                EntregaMessage entregaMessage = new EntregaMessage(transporteMessage.getIdPedido(), entregaExitosa);
                ObjectMessage respuestaMessage = session.createObjectMessage(entregaMessage);
                producerPedido.send(respuestaMessage);
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }

    private boolean realizarEntrega(int idPedido) {
       return true;
    }
}
```

## 3. Estructura de mensajes

```java
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
