# Project Structure & Clean Architecture Reference

## Multi-Module Maven Layout (preferred for microservices)

```
my-service/
├── pom.xml                        # Parent BOM
├── domain/                        # Zero framework deps
│   ├── pom.xml
│   └── src/main/java/.../domain/
│       ├── model/                 # Entities, Value Objects, Aggregates
│       ├── repository/            # Repository interfaces (ports)
│       ├── service/               # Domain services
│       └── exception/             # Domain exceptions
├── application/                   # Use-case orchestration
│   ├── pom.xml                    # depends on :domain
│   └── src/main/java/.../application/
│       ├── usecase/               # One class per use case
│       ├── port/in/               # Inbound port interfaces
│       ├── port/out/              # Outbound port interfaces
│       └── dto/                   # Commands, Queries, Results
├── infrastructure/                # Spring, JPA, Kafka, HTTP clients
│   ├── pom.xml                    # depends on :application
│   └── src/main/java/.../infrastructure/
│       ├── persistence/           # JPA entities, repos, mappers
│       ├── messaging/             # Kafka/RabbitMQ adapters
│       ├── client/                # Feign / WebClient adapters
│       └── config/                # Spring @Configuration classes
└── presentation/                  # Controllers, resolvers, security filters
    ├── pom.xml                    # depends on :application
    └── src/main/java/.../presentation/
        ├── rest/
        ├── graphql/
        └── security/
```

## Parent pom.xml (key sections)

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.3.4</version>
</parent>

<properties>
  <java.version>21</java.version>
  <spring-cloud.version>2023.0.3</spring-cloud.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## Domain Model Patterns

### Aggregate Root
```java
@Entity
@Table(name = "orders")
public class Order {  // Aggregate root

    @Id
    @UuidGenerator
    private UUID id;

    @Embedded
    private CustomerId customerId;          // Value Object

    @OneToMany(cascade = ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    @Enumerated(STRING)
    private OrderStatus status = OrderStatus.DRAFT;

    @Version
    private Long version;                   // Optimistic locking

    // Business key equals/hashCode
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Order other)) return false;
        return id != null && id.equals(other.id);
    }

    @Override public int hashCode() { return getClass().hashCode(); }

    // Domain behaviour — no setters for state machine transitions
    public void confirm() {
        if (status != OrderStatus.DRAFT)
            throw new IllegalStateException("Order already confirmed");
        this.status = OrderStatus.CONFIRMED;
    }
}
```

### Value Object
```java
@Embeddable
public record CustomerId(@NotBlank String value) {
    public CustomerId {
        Objects.requireNonNull(value);
        if (value.isBlank()) throw new IllegalArgumentException("customerId blank");
    }
}
```

### Repository Port (domain layer)
```java
public interface OrderRepository {
    Optional<Order> findById(UUID id);
    Order save(Order order);
    Page<Order> findByCustomerId(CustomerId customerId, Pageable pageable);
}
```

## Use Case Pattern

```java
// application/usecase/PlaceOrderUseCase.java
@UseCase   // custom stereotype — @Service + @Transactional
@RequiredArgsConstructor
public class PlaceOrderUseCase implements PlaceOrderPort {

    private final OrderRepository orderRepository;
    private final ProductPort productPort;
    private final EventPublisher eventPublisher;

    @Override
    public OrderResult execute(PlaceOrderCommand cmd) {
        var order = Order.create(new CustomerId(cmd.customerId()), cmd.items()
            .stream().map(this::toLine).toList());

        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order.getId()));
        return OrderResult.from(order);
    }
}
```

## Custom Stereotypes

```java
// Cleaner than raw @Service
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Service
@Transactional
public @interface UseCase {}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface Adapter {}
```

## application.yml structure

```yaml
spring:
  application:
    name: order-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
  threads:
    virtual:
      enabled: true   # Java 21 virtual threads

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/orders}
    username: ${DB_USER:orders}
    password: ${DB_PASSWORD:secret}

  jpa:
    open-in-view: false          # ALWAYS disable
    hibernate:
      ddl-auto: validate         # Flyway manages schema

server:
  port: ${PORT:8080}
  shutdown: graceful             # Drain in-flight requests on SIGTERM

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      probes:
        enabled: true            # /actuator/health/liveness, /readiness
```

## Docker Compose (local dev)

```yaml
# compose.yaml (Spring Boot 3.1+ auto-detects this)
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 1
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    ports: ["9092:9092"]
```
