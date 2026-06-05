# RabbitMQ

## 1. Executive Summary

RabbitMQ is an open-source message broker that implements the Advanced Message Queuing Protocol (AMQP). Written in Erlang, it provides reliable messaging, flexible routing, and supports multiple messaging protocols. RabbitMQ is known for its robustness, ease of use, and extensive routing capabilities through exchanges and bindings. It is widely used for task distribution, decoupling services, and building reliable distributed systems.

## 2. Core Theory

### AMQP Model

- **Producer**: Publishes messages to an exchange.
- **Exchange**: Receives messages and routes them to queues based on bindings.
- **Binding**: A link between an exchange and a queue with a routing key.
- **Queue**: Stores messages until consumed.
- **Consumer**: Receives messages from a queue.

### Exchange Types

| Type | Routing Logic | Example Use Case |
|------|--------------|------------------|
| Direct | Matches routing key exactly | Point-to-point messaging |
| Topic | Matches routing key pattern (wildcards: `*`, `#`) | Pub-sub with filtering |
| Fanout | Routes to all bound queues (ignores routing key) | Broadcast |
| Headers | Matches message headers | Content-based routing |

### Message Lifecycle

```
Producer -> Exchange -> (Binding match) -> Queue -> Consumer (ACK)
                                                  -> DLQ (on failure)
                                                  -> TTL expiry
```

## 3. Under-the-Hood Deep Dive

### AMQP Frame Structure

```
AMQP Frame:
| Frame Type (1 byte) | Channel (2 bytes) | Size (4 bytes) | Payload | End (1 byte) |

Frame Types:
- TYPE_METHOD (1): Method invocation
- TYPE_HEADER (2): Content header
- TYPE_BODY (3): Message body
- TYPE_HEARTBEAT (4): Heartbeat
```

### RabbitMQ Internals

- **Erlang Process per Queue**: Each queue runs as a separate Erlang process.
- **Message Store**: Messages are persisted to a write-ahead log (WAL) and message store.
- **Lazy Queues**: Messages are stored on disk immediately (reduces RAM usage).
- **Quorum Queues**: Raft-based replicated queues for high availability.
- **Streams**: Append-only log for replayable message consumption.

### Confirms and Transactions

```java
// Publisher Confirms (recommended)
channel.confirmSelect();
channel.basicPublish(exchange, routingKey, null, message.getBytes());
channel.waitForConfirmsOrDie(5000);

// Transactions (slower, not recommended for throughput)
channel.txSelect();
channel.basicPublish(exchange, routingKey, null, message.getBytes());
channel.txCommit();
```

## 4. Production Code Examples

### Spring Boot RabbitMQ Configuration

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    template:
      retry:
        enabled: true
        initial-interval: 1000ms
        max-attempts: 3
        multiplier: 2.0
    listener:
      simple:
        concurrency: 5
        max-concurrency: 10
        acknowledge-mode: manual
        prefetch: 10
        retry:
          enabled: true
          initial-interval: 1000ms
          max-attempts: 3
          multiplier: 2.0
```

### Declarative Queue Configuration

```java
@Configuration
public class RabbitMQConfig {

    // Exchange
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange("order.exchange")
            .durable(true)
            .build();
    }

    @Bean
    public TopicExchange notificationExchange() {
        return ExchangeBuilder.topicExchange("notification.exchange")
            .durable(true)
            .build();
    }

    @Bean
    public FanoutExchange broadcastExchange() {
        return ExchangeBuilder.fanoutExchange("broadcast.exchange")
            .durable(true)
            .build();
    }

    // Queues
    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable("order.created.queue")
            .deadLetterExchange("order.dlx")
            .deadLetterRoutingKey("order.dead")
            .messageTtl(86400000) // 24 hours
            .maxLength(100000)
            .build();
    }

    @Bean
    public Queue orderPaymentQueue() {
        return QueueBuilder.durable("order.payment.queue")
            .lazy() // keep messages on disk
            .build();
    }

    @Bean
    public Queue notificationEmailQueue() {
        return QueueBuilder.durable("notification.email.queue")
            .build();
    }

    // Bindings
    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder.bind(orderCreatedQueue())
            .to(orderExchange())
            .with("order.created");
    }

    @Bean
    public Binding orderPaymentBinding() {
        return BindingBuilder.bind(orderPaymentQueue())
            .to(orderExchange())
            .with("order.payment");
    }

    @Bean
    public Binding emailNotificationBinding() {
        return BindingBuilder.bind(notificationEmailQueue())
            .to(notificationExchange())
            .with("notification.email.*");
    }

    // Dead Letter Queue
    @Bean
    public DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange("order.dlx").build();
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("order.dlq").build();
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange())
            .with("order.dead");
    }
}
```

### Producer

```java
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishOrderCreated(OrderEvent event) {
        CorrelationData correlationData = new CorrelationData(
            UUID.randomUUID().toString());

        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.created",
            event,
            message -> {
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                message.getMessageProperties().setPriority(event.getPriority());
                message.getMessageProperties().setHeader("eventType", event.getType());
                message.getMessageProperties().setExpiration("3600000"); // 1 hour TTL
                return message;
            },
            correlationData);

        correlationData.getFuture().whenComplete((confirm, ex) -> {
            if (confirm != null && confirm.isAck()) {
                log.info("Message confirmed: {}", correlationData.getId());
            } else {
                log.error("Message not confirmed: {}", correlationData.getId());
                // Handle failure
                handlePublishFailure(event);
            }
        });
    }

    public void publishOrderPayment(OrderEvent event) {
        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.payment",
            event);
    }

    public void publishNotification(String type, NotificationEvent event) {
        rabbitTemplate.convertAndSend(
            "notification.exchange",
            "notification.email." + type,
            event);
    }

    // Batch publish
    public void publishBatch(List<OrderEvent> events) {
        rabbitTemplate.invoke(operations -> {
            events.forEach(event ->
                operations.convertAndSend(
                    "order.exchange", "order.created", event));
            return null;
        });
    }
}
```

### Consumer with Manual Acknowledgment

```java
@Component
public class OrderCreatedConsumer {

    private final OrderService orderService;

    @RabbitListener(queues = "order.created.queue",
                    containerFactory = "rabbitListenerContainerFactory")
    public void handleOrderCreated(
            OrderEvent event,
            Message message,
            Channel channel) throws IOException {

        long deliveryTag = message.getMessageProperties().getDeliveryTag();

        try {
            log.info("Processing order: {}", event.getOrderId());
            orderService.processNewOrder(event);
            channel.basicAck(deliveryTag, false);
        } catch (RetryableException e) {
            log.warn("Retryable error for order: {}", event.getOrderId());
            channel.basicNack(deliveryTag, false, true); // requeue true
        } catch (FatalException e) {
            log.error("Fatal error for order: {}", event.getOrderId());
            channel.basicNack(deliveryTag, false, false); // requeue false -> DLQ
        }
    }
}
```

### Custom Retry with Dead Lettering

```java
@Configuration
public class RetryConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setConcurrentConsumers(5);
        factory.setMaxConcurrentConsumers(10);
        factory.setPrefetchCount(10);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        factory.setDefaultRequeueRejected(false);
        factory.setAdviceChain(RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000) // 1s, 2s, 4s, max 10s
            .recoverer(new RejectAndDontRequeueRecoverer())
            .build());
        return factory;
    }
}
```

### Request-Reply Pattern

```java
@Service
public class OrderStatusClient {

    private final RabbitTemplate rabbitTemplate;

    public OrderStatus requestOrderStatus(String orderId) {
        OrderStatusRequest request = new OrderStatusRequest(orderId);
        OrderStatus response = (OrderStatus) rabbitTemplate.convertSendAndReceive(
            "order.exchange",
            "order.status.request",
            request,
            message -> {
                message.getMessageProperties().setReplyTo("order.status.reply.queue");
                message.getMessageProperties().setCorrelationId(UUID.randomUUID().toString());
                return message;
            });
        return response;
    }
}

@Component
public class OrderStatusConsumer {

    @RabbitListener(queues = "order.status.request.queue")
    public OrderStatus handleStatusRequest(OrderStatusRequest request) {
        return orderService.getStatus(request.getOrderId());
    }
}
```

### Topic Exchange with Wildcards

```java
// Binding patterns
// *.critical  -> production.critical, staging.critical
// order.*     -> order.created, order.payment
// #.error     -> production.service.error, dev.database.error

@Component
public class TopicConsumer {

    @RabbitListener(queues = "critical.alerts.queue")
    public void handleCriticalAlert(AlertEvent event) {
        // Only receives *.critical messages
    }

    @RabbitListener(queues = "all.order.events.queue")
    public void handleAllOrderEvents(OrderEvent event) {
        // Receives order.# messages
    }
}
```

### Quorum Queue Configuration

```java
@Bean
public Queue quorumOrderQueue() {
    return QueueBuilder.durable("order.quorum.queue")
        .quorum() // Raft-based replication
        .deliveryLimit(3) // Max delivery attempts
        .deadLetterExchange("order.dlx")
        .build();
}
```

### Stream Queue

```java
@Bean
public Queue orderStreamQueue() {
    return QueueBuilder.durable("order.stream.queue")
        .stream() // Append-only log
        .maxLengthBytes(1000000000) // 1GB
        .build();
}
```

### Message Converter with JSON

```java
@Configuration
public class MessageConverterConfig {

    @Bean
    public MessageConverter jsonMessageConverter() {
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        converter.setCreateMessageIds(true);
        converter.setClassMapper(classMapper());
        return converter;
    }

    @Bean
    public DefaultClassMapper classMapper() {
        DefaultClassMapper classMapper = new DefaultClassMapper();
        Map<String, Class<?>> mappings = new HashMap<>();
        mappings.put("com.example.event.OrderEvent", OrderEvent.class);
        mappings.put("com.example.event.NotificationEvent", NotificationEvent.class);
        classMapper.setIdClassMapping(mappings);
        return classMapper;
    }
}
```

### Custom Retry Template

```java
@Component
public class RetryableProducer {

    private final RabbitTemplate rabbitTemplate;
    private final RetryTemplate retryTemplate;

    public RetryableProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(5)
            .exponentialBackoff(1000, 2, 30000)
            .retryOn(ConnectionException.class)
            .build();
    }

    public void publishWithRetry(String exchange, String routingKey, Object message) {
        retryTemplate.execute(context -> {
            rabbitTemplate.convertAndSend(exchange, routingKey, message);
            return null;
        });
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: E-Commerce Order Processing

```
Order Service
  |-- (fanout) order.created --> Inventory Service
                              --> Payment Service
                              --> Notification Service
                              --> Analytics Service
```

### Scenario 2: Multi-Tenant Notification System

```java
@Configuration
public class MultiTenantNotificationConfig {

    // Each tenant gets its own queue
    @Bean
    public Queue tenant1Notifications() {
        return QueueBuilder.durable("tenant1.notifications")
            .maxPriority(10)
            .build();
    }

    @Bean
    public Queue tenant2Notifications() {
        return QueueBuilder.durable("tenant2.notifications")
            .maxPriority(10)
            .build();
    }

    @Bean
    public Binding tenant1Binding() {
        return BindingBuilder.bind(tenant1Notifications())
            .to(notificationExchange())
            .with("notification.tenant1.*");
    }
}
```

### Scenario 3: Delayed Message Processing

```java
@Configuration
public class DelayedMessagingConfig {

    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(
            "delayed.exchange",
            "x-delayed-message",
            true,
            false,
            args);
    }

    @Bean
    public Queue delayedOrderQueue() {
        return QueueBuilder.durable("delayed.order.queue").build();
    }

    @Bean
    public Binding delayedBinding() {
        return BindingBuilder.bind(delayedOrderQueue())
            .to(delayedExchange())
            .with("order.delayed")
            .noargs();
    }
}

@Service
public class DelayedPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishDelayed(OrderEvent event, int delayMs) {
        rabbitTemplate.convertAndSend(
            "delayed.exchange",
            "order.delayed",
            event,
            message -> {
                message.getMessageProperties()
                    .setHeader("x-delay", delayMs);
                return message;
            });
    }
}
```

## 6. Performance

### Performance Tuning

```yaml
# Producer performance
spring.rabbitmq.template:
  mandatory: true
  receive-timeout: 5000
  reply-timeout: 5000

# Consumer performance
spring.rabbitmq.listener.simple:
  concurrency: 10
  max-concurrency: 50
  prefetch: 50
  transaction-size: 10

# Performance tips:
# - Use publisher confirms for reliability
# - Batch publishes when possible
# - Set appropriate prefetch count
# - Use async consumers
# - Enable lazy queues for large queues
```

### Prefetch Optimization

- **Low prefetch (1-10)**: Fair distribution, good for heterogeneous consumers.
- **High prefetch (100-1000)**: Higher throughput, risk of uneven load.
- **Rule of thumb**: prefetch = 2 * average processing time (seconds) * desired throughput.

### Throughput Benchmarks

```
Direct Exchange:    ~50,000 msg/s (persistent)
Fanout Exchange:    ~45,000 msg/s (persistent)
Topic Exchange:     ~40,000 msg/s (persistent)
Non-persistent:     ~100,000 msg/s
```

## 7. Security

### TLS Configuration

```yaml
spring:
  rabbitmq:
    ssl:
      enabled: true
      key-store: classpath:client-key.p12
      key-store-password: changeit
      trust-store: classpath:truststore.jks
      trust-store-password: changeit
      algorithm: TLSv1.2
```

### Authentication and Authorization

```yaml
spring:
  rabbitmq:
    username: application-user
    password: strong-password
    virtual-host: /myapp

# RabbitMQ permission commands:
# rabbitmqctl add_user app-user strong-password
# rabbitmqctl set_permissions -p /myapp app-user ".*" ".*" ".*"
# rabbitmqctl set_topic_permissions -p /myapp app-user "amq.topic" "order.*" "order.*"
```

## 8. Common Mistakes

- **No dead letter configuration**: Failed messages remain in the queue, blocking processing.
- **Forgetting manual ACK**: Auto-ACK can lose messages on consumer crash.
- **Too many queues on one node**: Each queue consumes resources.
- **Unlimited queue growth**: Set max length and TTL to prevent unbounded growth.
- **Synchronous publishing**: Blocks producer thread; use async confirms.
- **Not handling poison messages**: Messages that consistently fail need to be moved to DLQ.
- **Incorrect exchange type**: Using direct when topic or fanout would be more appropriate.
- **No monitoring**: Queue depth, consumer lag, and message rates must be tracked.
- **Binding mismatches**: Ensure routing keys match between publisher bindings and consumer bindings.

## 9. Senior Engineer Perspective

### Operational Best Practices

- **Cluster size**: 3 nodes minimum, odd number for quorum queues.
- **Memory high watermark**: Set `vm_memory_high_watermark` to 0.4-0.6.
- **Disk free limit**: Set `disk_free_limit` to adequate space for message persistence.
- **Monitoring**: Queue depth, message rates, consumer count, file descriptors, memory, disk.
- **Backup**: Regular backups of definitions (rabbitmqadmin export) and message store.

### High Availability

```java
@Configuration
public class HaConfig {

    @Bean
    public Queue haOrderQueue() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-queue-type", "quorum");
        args.put("x-quorum-initial-group-size", 3);
        args.put("x-delivery-limit", 3);
        return new Queue("order.ha.queue", true, false, false, args);
    }
}
```

### Architectural Patterns

```
Competing Consumers: Multiple consumers on one queue (load balancing)
Work Queues: Task distribution with fair dispatch
RPC: Request-reply via callback queues
Dead Lettering: Failed messages routed to DLQ
Priority Queues: High priority messages processed first
Transaction: Atomic enqueue/dequeue operations
```

## 10. Interview Questions (Easy)

1. What is RabbitMQ and what protocol does it implement?
2. What is an exchange in RabbitMQ?
3. Name the four exchange types in RabbitMQ.
4. What is a binding in RabbitMQ?
5. What is the difference between a queue and an exchange?
6. What is message acknowledgment?
7. What is a dead letter queue?
8. What is the purpose of a routing key?
9. What is a virtual host in RabbitMQ?
10. What port does RabbitMQ use for AMQP?

## Medium

1. How do publisher confirms work in RabbitMQ?
2. What is the difference between direct and topic exchanges?
3. How do you implement retry with dead lettering in RabbitMQ?
4. What is prefetch count and how does it affect performance?
5. How do quorum queues differ from classic queues?
6. What is lazy queue mode and when to use it?
7. How do you implement request-reply with RabbitMQ?
8. What is the difference between fanout and direct exchanges?
9. How do you configure RabbitMQ for high availability?
10. What is the shovel plugin and when would you use it?

## 11. Advanced Interview Questions (Hard)

1. Design a system that guarantees exactly-once delivery with RabbitMQ.
2. How would you implement a priority queue system with multiple levels?
3. Design a RabbitMQ-based saga coordinator for distributed transactions.
4. How do you handle message ordering guarantees across multiple queues?
5. Implement a custom exchange type for content-based routing.
6. Design a RabbitMQ cluster with cross-datacenter replication.
7. How would you implement a delayed retry mechanism with exponential backoff?
8. Design a RabbitMQ-based event sourcing system.
9. How do you handle large messages (>100MB) in RabbitMQ?
10. Implement a circuit breaker pattern with RabbitMQ consumers.

## System Design

1. Design a RabbitMQ-based order processing system for an e-commerce platform.
2. Design a multi-tenant notification system with RabbitMQ.
3. Design a RabbitMQ-based microservices event bus.
4. Design a real-time logging and monitoring system with RabbitMQ.
5. Design a distributed job scheduler using RabbitMQ.
6. Design a RabbitMQ-based payment processing pipeline.
7. Design a real-time chat system using RabbitMQ.
8. Design a RabbitMQ-based CI/CD pipeline event system.
9. Design a RabbitMQ-based inventory management system.
10. Design a RabbitMQ-based analytics data pipeline.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a global RabbitMQ mesh network connecting 10 data centers with automatic failover, message replication, and exactly-once delivery guarantees.
2. How would you build a RabbitMQ-based event store that supports event sourcing, snapshots, and replay for a financial trading system?
3. Design a multi-tenant RabbitMQ platform with per-tenant quotas, rate limiting, and resource isolation for a SaaS provider.
4. How would you implement a RabbitMQ migration from classic mirrored queues to quorum queues across 500+ queues with zero downtime?
5. Design a RabbitMQ monitoring and auto-remediation system that detects anomalies (partitioned clusters, high memory, queue pileup) and triggers automated recovery.
6. How would you build a RabbitMQ-based distributed transaction coordinator using the saga pattern with automatic compensation and recovery?
7. Design a RabbitMQ plugin that adds custom exchange types for advanced routing (e.g., geospatial, time-based, probabilistic).
8. How would you implement a RabbitMQ schema registry for message type evolution and compatibility checking?
9. Design a RabbitMQ capacity planning system that predicts resource needs based on traffic patterns and automatically scales the cluster.
10. How would you build a multi-region RabbitMQ solution with active-active topology and conflict resolution?

## 13. Debugging & Troubleshooting

### Common Issues

- **Unroutable messages**: Check exchange and binding configuration.
- **Queue pileup**: Check consumer health, processing speed, prefetch settings.
- **Connection refused**: Check port, firewall, user credentials.
- **Message lost**: Verify persistence settings, ACK mode, publisher confirms.
- **High memory usage**: Check queue depth, lazy queues, memory alarms.
- **Consumer cancellation**: Check queue existence, permissions.

### Management Commands

```bash
# List queues
rabbitmqctl list_queues name messages consumers

# List exchanges
rabbitmqctl list_exchanges name type

# List bindings
rabbitmqctl list_bindings

# Check status
rabbitmqctl status

# Check connections
rabbitmqctl list_connections

# Trace messages
rabbitmqctl trace_on
```

### Monitoring with Spring Boot Actuator

```yaml
management:
  metrics:
    export:
      rabbitmq:
        enabled: true
  endpoint:
    rabbitmq:
      enabled: true
```

## 14. Comparison Section

### RabbitMQ vs Kafka

| Aspect | RabbitMQ | Kafka |
|--------|----------|-------|
| Model | Exchange/Queue | Log/Topic/Partition |
| Performance | ~50K msg/s | ~1M msg/s |
| Message Routing | Flexible (4 exchange types) | Topic-based only |
| Message Retention | Deleted after ACK | Configurable retention |
| Protocol | AMQP, MQTT, STOMP | Custom binary protocol |
| Message Size | No hard limit (practical <100MB) | 1MB default |
| Complex Routing | Excellent | Limited |
| Learning Curve | Moderate | Moderate-High |

### RabbitMQ vs ActiveMQ

| Aspect | RabbitMQ | ActiveMQ |
|--------|----------|----------|
| Language | Erlang | Java |
| Protocol | AMQP (primary) | JMS, AMQP, MQTT |
| Performance | Higher | Lower |
| Cluster | Native Erlang clustering | Shared DB or network |
| Management | Excellent UI | Good |
| JMS Support | Via plug-in | Native |
| Community | Large | Large |

## 15. Revision Notes

- RabbitMQ: AMQP broker, Erlang-based, flexible routing
- 4 exchange types: Direct, Topic, Fanout, Headers
- Message lifecycle: Producer -> Exchange -> Binding -> Queue -> Consumer
- ACK modes: auto, manual (recommended), none
- Quorum queues for HA (Raft consensus)
- Lazy queues for large queues (disk-based)
- Dead letter queues for failed messages
- Publisher confirms for reliable publishing
- Prefetch count: balance throughput and fairness
- `@RabbitListener`, `RabbitTemplate`, `@EnableRabbit`

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| RABBITMQ CHEAT SHEET                                             |
+------------------------------------------------------------------+
| EXCHANGE TYPES                                                   |
|   Direct: routing_key = queue_name (exact match)                 |
|   Topic:  routing_key matches pattern (*, #)                     |
|   Fanout: routes to ALL bound queues (broadcast)                 |
|   Headers: matches on message headers (not routing key)          |
+------------------------------------------------------------------+
| EXCHANGE PATTERNS                                                |
|   direct:     order.created -> [order.created.queue]             |
|   topic:      order.*       -> [order.created, order.updated]    |
|   fanout:     broadcast     -> [queue1, queue2, queue3]          |
|   headers:    headers match -> [queue]                           |
+------------------------------------------------------------------+
| WILDCARD RULES                                                   |
|   * (star):  matches exactly one word                            |
|   # (hash):  matches zero or more words                          |
|   Example:   order.*.completed -> order.payment.completed        |
|              order.#           -> order.created, order.payment.ok|
+------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                          |
|   @EnableRabbit                -- Enable RabbitMQ support        |
|   @RabbitListener(queues=...) -- Message listener                |
|   @RabbitHandler               -- Method-level handler           |
|   RabbitTemplate               -- Send/receive messages          |
+------------------------------------------------------------------+
| QUEUE PROPERTIES                                                 |
|   durable:    survive broker restart                             |
|   exclusive:  auto-delete when connection closes                 |
|   auto-delete: delete when last consumer unsubscribes             |
|   arguments:  x-message-ttl, x-dead-letter-exchange, x-max-length|
+------------------------------------------------------------------+
| RELIABILITY PATTERNS                                             |
|   Publisher Confirm  -> async ACK from broker                    |
|   Consumer ACK       -> manual basicAck/basicNack                |
|   Dead Letter Queue  -> x-dead-letter-exchange property          |
|   Quorum Queue       -> Raft-based HA                            |
|   Lazy Queue         -> Always on disk                           |
+------------------------------------------------------------------+
| CLI COMMANDS                                                     |
|   rabbitmqctl list_queues                                        |
|   rabbitmqctl list_exchanges                                     |
|   rabbitmqctl list_bindings                                      |
|   rabbitmqadmin get queue <name>                                 |
|   rabbitmqadmin list queues                                      |
+------------------------------------------------------------------+
```
