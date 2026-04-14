---
name: springboot-agent
description: >
  Expert Spring Boot engineering skill covering the full production stack. Use this skill for ANY
  Spring Boot task — generating project scaffolding, writing service/controller/repository layers,
  configuring Spring Security + OAuth2, designing REST or GraphQL APIs, wiring Spring Cloud
  microservices, adding distributed tracing and observability, integrating message queues (Kafka,
  RabbitMQ), caching (Redis, Caffeine), and applying Clean Architecture / DDD patterns.
  Trigger this skill whenever the user mentions: Spring Boot, Spring Data, Spring Security, OAuth,
  JWT, Spring Cloud, Eureka, Gateway, Config Server, Feign, REST, GraphQL, microservices,
  Kafka, RabbitMQ, Redis, Micrometer, OpenTelemetry, Zipkin, Sleuth, Actuator, JPA, Hibernate,
  Flyway/Liquibase, Docker Compose for Spring, Testcontainers, or requests help with a Java
  backend service of any complexity. Always consult this skill before generating Spring Boot code.
---

# Spring Boot Agent Skill

A comprehensive guide for producing idiomatic, production-grade Spring Boot applications.
Always read the relevant reference file(s) before generating code for a specific domain.

---

## Reference Files — load the ones relevant to the task

| File | When to load |
|---|---|
| `references/project-structure.md` | Scaffolding, module layout, Clean Architecture, DDD |
| `references/data.md` | Spring Data JPA/MongoDB/Redis, Flyway, Querydsl, transactions |
| `references/security.md` | Spring Security, OAuth2, JWT, method security, CORS |
| `references/rest-graphql.md` | REST controllers, validation, error handling, GraphQL |
| `references/cloud-microservices.md` | Spring Cloud, Gateway, Eureka, Config, Feign, Resilience4j |
| `references/messaging.md` | Kafka, RabbitMQ, transactional outbox, dead-letter queues |
| `references/caching.md` | Redis, Caffeine, cache-aside, TTL strategies |
| `references/observability.md` | Actuator, Micrometer, OpenTelemetry, Zipkin, Loki, Grafana |
| `references/testing.md` | Unit, slice (@WebMvcTest etc.), Testcontainers, contract tests |

---

## Core Principles (always apply)

### 1. Layered / Clean Architecture default
```
presentation/   → Controllers, GraphQL resolvers, DTOs
application/    → Use-case services, command/query objects
domain/         → Entities, value objects, domain services, repository interfaces
infrastructure/ → JPA repos, external clients, messaging adapters, config
```
- Domain layer has **zero** Spring/framework dependencies.
- Application layer orchestrates; it may use Spring `@Transactional` but no HTTP types.
- Infrastructure wires everything together via Spring DI.

### 2. Dependency direction
```
presentation → application → domain ← infrastructure
```
Infrastructure *implements* domain interfaces (Ports & Adapters). Never import infrastructure types in domain.

### 3. Package naming convention
```
com.{company}.{service}/
  presentation/rest/
  presentation/graphql/
  application/usecase/
  application/dto/
  domain/model/
  domain/repository/
  domain/service/
  infrastructure/persistence/
  infrastructure/messaging/
  infrastructure/client/
  config/
```

### 4. Always use records for DTOs (Java 16+)
```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemDto> items
) {}
```

### 5. Constructor injection only — no `@Autowired` on fields
```java
@Service
@RequiredArgsConstructor   // Lombok
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;
}
```

### 6. Fail-fast configuration
- Use `@ConfigurationProperties` + `@Validated` beans — never raw `@Value` for structured config.
- Bind secrets via environment variables; never commit secrets.

### 7. Exception handling
- Custom exception hierarchy: `DomainException` → `NotFoundException`, `ConflictException`, etc.
- Single `@RestControllerAdvice` maps to RFC 7807 `ProblemDetail` (Spring 6+).

---

## Quick-Start Checklist

When the user asks to create a new service, follow this order:

1. **Read** `references/project-structure.md`
2. Emit `pom.xml` / `build.gradle` with correct BOM (`spring-boot-dependencies`)
3. Create package skeleton per §3 above
4. Add `application.yml` with environment-variable placeholders
5. Wire security (read `references/security.md`)
6. Add observability (read `references/observability.md`)
7. Add persistence layer (read `references/data.md`)
8. Expose API — REST or GraphQL (read `references/rest-graphql.md`)
9. Add tests (read `references/testing.md`)

---

## Spring Boot Version Policy

- **Default**: Spring Boot **3.3.x** (Spring Framework 6.1, Jakarta EE 10)
- Java **21** (virtual threads enabled via `spring.threads.virtual.enabled=true`)
- Only suggest Boot 2.x if the user explicitly requests it; note the Jakarta vs javax namespace difference.

---

## Common Dependency Block (Gradle Kotlin DSL)

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.25"          // omit for pure Java
    kotlin("plugin.spring") version "1.9.25"
}

dependencies {
    // Core
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    // Data
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")

    // Security
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")

    // Observability
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
}
```

---

## Decision Trees

### "Should I use REST or GraphQL?"
- Multiple clients with different field-shape needs → **GraphQL**
- Simple CRUD service, internal only → **REST**
- Mobile-first, bandwidth-sensitive → **GraphQL**
- File upload/download involved → **REST** (GraphQL handles it poorly)
- Both? Mount REST at `/api/**` and GraphQL at `/graphql` — they coexist fine.

### "Which cache?"
- Session/distributed, cluster → **Redis**
- Single-node, high-throughput, no TTL complexity → **Caffeine**
- Need pub/sub invalidation → **Redis**

### "Which message broker?"
- Need ordering, replay, high throughput → **Kafka**
- Need routing, dead-letter, RPC-style → **RabbitMQ**
- Both available? Default to Kafka for events, RabbitMQ for commands.

---

## Code Generation Rules

1. Always include `@Slf4j` (Lombok) on service classes; use `log.debug()`/`log.info()` with structured args.
2. Never swallow exceptions silently.
3. Paginate all collection endpoints: `Page<T>` + `Pageable` parameter.
4. Use `Optional` returns from repositories; never return `null`.
5. All `@Entity` classes: use `UUID` PK (`@UuidGenerator`), define `equals`/`hashCode` on business key.
6. Async methods annotated with `@Async`; always return `CompletableFuture` or `void`.
7. Mark read-only transactions with `@Transactional(readOnly = true)`.
8. Expose health + metrics via Actuator; never disable in production profiles.
