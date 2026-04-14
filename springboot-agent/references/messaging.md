# Messaging Reference — Kafka & RabbitMQ

## Kafka

### Dependencies
```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

### application.yml
```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all            # strongest durability
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 1
    consumer:
      group-id: ${spring.application.name}
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false      # manual ack for at-least-once
      properties:
        spring.json.trusted.packages: "com.example.*"
    listener:
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 3               # threads per consumer
```

### Producer
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void publish(OrderPlacedEvent event) {
        kafkaTemplate.send(
                KafkaTopics.ORDER_PLACED,
                event.orderId().toString(),   // key = orderId → same partition
                event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish OrderPlacedEvent {}", event.orderId(), ex);
                } else {
                    log.debug("Published to {}-{} offset {}",
                        result.getRecordMetadata().topic(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### Consumer
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderPlacedConsumer {

    private final ProcessOrderUseCase processOrderUseCase;

    @KafkaListener(
        topics = KafkaTopics.ORDER_PLACED,
        groupId = "${spring.application.name}",
        containerFactory = "kafkaListenerContainerFactory")
    public void onOrderPlaced(
        @Payload OrderPlacedEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack) {

        log.info("Received OrderPlacedEvent id={} partition={} offset={}",
            event.orderId(), partition, offset);
        try {
            processOrderUseCase.execute(event);
            ack.acknowledge();
        } catch (NonRetryableException ex) {
            log.error("Non-retryable failure for order {}", event.orderId(), ex);
            ack.acknowledge();  // don't reprocess; route to DLT separately
        }
        // Retryable exceptions → Spring Kafka retries, then DLT
    }
}
```

### Dead Letter Topic
```java
@Bean
CommonErrorHandler errorHandler(KafkaOperations<Object, Object> template) {
    var recover = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", -1));

    var backOff = new ExponentialBackOffWithMaxRetries(3);
    backOff.setInitialInterval(1_000);
    backOff.setMultiplier(2.0);

    return new DefaultErrorHandler(recover, backOff);
}
```

### Transactional Outbox Pattern (recommended for consistency)
```java
// 1. Write event to outbox table in same DB transaction as domain change
@Transactional
public Order createOrder(CreateOrderCmd cmd) {
    var order = orderRepository.save(Order.create(cmd));
    outboxRepository.save(new OutboxEvent("order.placed", order.getId(),
        objectMapper.writeValueAsString(new OrderPlacedEvent(order))));
    return order;
}

// 2. Separate scheduled job reads outbox and publishes to Kafka
@Scheduled(fixedDelay = 1000)
@Transactional
public void relayOutboxEvents() {
    outboxRepository.findUnpublishedBatch(50).forEach(event -> {
        kafkaTemplate.send(event.getTopic(), event.getAggregateId(), event.getPayload());
        outboxRepository.markPublished(event.getId());
    });
}
```

---

## RabbitMQ

### Dependencies
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### application.yml
```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: 5672
    username: ${RABBITMQ_USER:guest}
    password: ${RABBITMQ_PASS:guest}
    virtual-host: /
    listener:
      simple:
        acknowledge-mode: MANUAL
        prefetch: 10
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
```

### Exchange / Queue / Binding Config
```java
@Configuration
public class RabbitConfig {

    public static final String ORDER_EXCHANGE  = "orders";
    public static final String ORDER_QUEUE     = "orders.processing";
    public static final String ORDER_DLX       = "orders.dlx";
    public static final String ORDER_DLQ       = "orders.dlq";

    @Bean TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE).durable(true).build();
    }

    @Bean Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", ORDER_DLX)
            .withArgument("x-dead-letter-routing-key", "dlq")
            .withArgument("x-message-ttl", 300_000)
            .build();
    }

    @Bean DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange(ORDER_DLX).durable(true).build();
    }

    @Bean Queue deadLetterQueue() {
        return QueueBuilder.durable(ORDER_DLQ).build();
    }

    @Bean Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.*");
    }
}
```

### Producer
```java
@Component
@RequiredArgsConstructor
public class OrderMessagePublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderPlaced(OrderPlacedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.ORDER_EXCHANGE,
            "order.placed",
            event,
            message -> {
                message.getMessageProperties().setCorrelationId(event.orderId().toString());
                message.getMessageProperties().setContentType("application/json");
                return message;
            });
    }
}
```

### Consumer
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderMessageConsumer {

    @RabbitListener(queues = RabbitConfig.ORDER_QUEUE)
    public void onOrderPlaced(
        OrderPlacedEvent event,
        Message message,
        Channel channel) throws IOException {

        long tag = message.getMessageProperties().getDeliveryTag();
        try {
            processOrderUseCase.execute(event);
            channel.basicAck(tag, false);
        } catch (RecoverableException ex) {
            log.warn("Retrying order {}", event.orderId());
            channel.basicNack(tag, false, true);   // requeue
        } catch (Exception ex) {
            log.error("Rejecting order {} to DLQ", event.orderId(), ex);
            channel.basicNack(tag, false, false);  // → DLX
        }
    }
}
```

---

## Event Schema Conventions

```java
// Base event
public record DomainEvent(
    UUID eventId,
    String eventType,
    UUID aggregateId,
    String aggregateType,
    Instant occurredAt,
    int version
) {}

// Specific event
public record OrderPlacedEvent(
    UUID eventId,
    UUID aggregateId,
    String aggregateType,
    Instant occurredAt,
    int version,
    // Payload
    UUID customerId,
    BigDecimal total,
    String currency
) implements DomainEventPayload {

    public static OrderPlacedEvent from(Order order) {
        return new OrderPlacedEvent(
            UUID.randomUUID(), order.getId(), "Order",
            Instant.now(), 1,
            order.getCustomerId(), order.getTotal(), "USD");
    }
}
```
