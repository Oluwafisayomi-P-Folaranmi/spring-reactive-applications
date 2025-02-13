# spring-reactive-applications

When you move from a monolithic or single-application context to a distributed microservices architecture, **in-process Spring events** are no longer enough because they can’t cross application boundaries. This is where **external messaging systems**—like **Apache Kafka**, **RabbitMQ**, or frameworks such as **Spring Cloud Bus**—come into play. They allow you to propagate domain events across services reliably, decoupling your microservices even when they’re deployed separately.

Below, we’ll walk through a real-world example of an e-commerce system where the **Order Service** publishes an event (e.g., an order is placed) that is consumed by several other services (like **Inventory**, **Payment**, and **Notification** services) using external messaging.

---

## **Scenario Overview**

Imagine an e-commerce platform with the following microservices:

- **Order Service:** Creates orders and publishes an event when an order is placed.
- **Inventory Service:** Updates stock based on new orders.
- **Payment Service:** Processes payment for the order.
- **Notification Service:** Sends confirmation emails to customers.

In a distributed environment, each service runs as a separate application. To propagate the event across services, you have two common approaches:

1. **Direct Integration with a Messaging Broker (e.g., Kafka or RabbitMQ)**
2. **Using Spring Cloud Bus** to abstract the messaging layer and automatically broadcast events across services.

---

## **1. Direct Integration with Apache Kafka**

### **a. Order Service (Event Publisher)**

The Order Service publishes an event to Kafka when an order is placed.

#### **Domain Event**
```java
public class OrderPlacedEvent {
    private final String orderId;
    private final String userId;
    private final double total;

    public OrderPlacedEvent(String orderId, String userId, double total) {
        this.orderId = orderId;
        this.userId = userId;
        this.total = total;
    }

    // Getters...
}
```

#### **Publishing the Event with KafkaTemplate**
```java
@Component
public class OrderService {

    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    public OrderService(KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void placeOrder(Order order) {
        // Persist order logic here...
        System.out.println("Order placed: " + order.getId());

        // Create and publish the event to Kafka
        OrderPlacedEvent event = new OrderPlacedEvent(order.getId(), order.getUserId(), order.getTotal());
        kafkaTemplate.send("order-events", event);
    }
}
```

#### **Kafka Producer Configuration (application.yml)**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### **b. Inventory Service (Event Consumer)**

The Inventory Service listens to the `order-events` topic to update its stock.

#### **Consuming the Event with @KafkaListener**
```java
@Component
public class InventoryService {

    @KafkaListener(topics = "order-events", groupId = "inventory-group")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Inventory updated for order: " + event.getOrderId());
        // Update inventory logic...
    }
}
```

#### **Kafka Consumer Configuration (application.yml)**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: inventory-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: '*'
```

*Similarly, the Payment and Notification Services can have their own Kafka consumers to process the event according to their responsibilities.*

---

## **2. Using Spring Cloud Bus**

**Spring Cloud Bus** leverages a common messaging system (such as Kafka or RabbitMQ) to broadcast state changes or events across all connected services. This is particularly useful for propagating configuration changes or custom application events.

### **a. Setup Spring Cloud Bus**

1. **Include the Dependency:**  
   Add the Spring Cloud Bus dependency to each microservice. For example, using Maven:
   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-bus-kafka</artifactId>
   </dependency>
   ```
   *Or, if you prefer RabbitMQ: use `spring-cloud-starter-bus-amqp`.*

2. **Configure the Broker (application.yml):**
   ```yaml
   spring:
     cloud:
       stream:
         bindings:
           input:
             destination: bus
           output:
             destination: bus
     kafka:
       bootstrap-servers: localhost:9092
   ```

### **b. Publishing and Listening with Spring Cloud Bus**

With Spring Cloud Bus, you can publish a custom event that will be automatically broadcasted to all connected services.

#### **Custom Application Event**
```java
public class CustomOrderEvent extends ApplicationEvent {
    private final String orderId;

    public CustomOrderEvent(Object source, String orderId) {
        super(source);
        this.orderId = orderId;
    }

    public String getOrderId() {
        return orderId;
    }
}
```

#### **Order Service (Publishing via Spring ApplicationEventPublisher)**
```java
@Component
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void placeOrder(Order order) {
        // Save order...
        System.out.println("Order placed: " + order.getId());
        // Publish event on the application context (Spring Cloud Bus will pick this up)
        eventPublisher.publishEvent(new CustomOrderEvent(this, order.getId()));
    }
}
```

#### **Other Services (Listening for the Bus Event)**
Any service connected to the bus can listen for the `CustomOrderEvent`:
```java
@Component
public class NotificationService {

    @EventListener
    public void handleCustomOrderEvent(CustomOrderEvent event) {
        System.out.println("Notification Service received order event for order: " + event.getOrderId());
        // Send notification email logic...
    }
}
```

*Spring Cloud Bus automatically propagates the event to all services connected to the same messaging broker, ensuring that each microservice receives and processes the event accordingly.*

---

## **Key Benefits of External Messaging in Microservices**

- **Decoupling Across Process Boundaries:**  
  Microservices can be deployed independently and communicate asynchronously through a robust messaging system.

- **Scalability:**  
  Brokers like Kafka are designed to handle high throughput, making it easier to scale as your system grows.

- **Reliability & Durability:**  
  External brokers provide persistent storage for messages and features like replay and error handling.

- **Dynamic Event Propagation with Spring Cloud Bus:**  
  Spring Cloud Bus abstracts the messaging layer, enabling automatic broadcast of events (such as configuration changes or domain events) to all connected services.

---

## **Conclusion**

By integrating external messaging systems (Kafka or RabbitMQ) or using **Spring Cloud Bus**, you can build a **robust, event-driven microservices architecture** where services remain decoupled and scalable. The Order Service can publish an event (whether via Kafka directly or through Spring Cloud Bus), and multiple services—Inventory, Payment, Notification, etc.—can listen and react accordingly, even when deployed on separate servers.

Would you like to dive deeper into advanced topics like **error handling & retries**, **message serialization strategies**, or **security considerations** when propagating events across microservices?
