# Spring Cloud & Microservices Reference

## Dependency Versions (Spring Cloud 2023.0.x → Boot 3.3.x)

```xml
<spring-cloud.version>2023.0.3</spring-cloud.version>
```

---

## Spring Cloud Gateway

```yaml
# gateway/application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service       # lb:// = Eureka load-balanced
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"
            - AddRequestHeader=X-Request-Id, ${spring.application.name}-${random.uuid}
            - TokenRelay=    # forward JWT to downstream
```

```java
@Configuration
public class GatewayConfig {

    @Bean
    KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    }

    @Bean
    RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("product-rewrite",
                r -> r.path("/products/**")
                      .filters(f -> f.rewritePath("/products/(?<seg>.*)", "/api/v1/products/${seg}"))
                      .uri("lb://product-service"))
            .build();
    }
}
```

---

## Service Discovery (Eureka)

### Server
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication {}
```
```yaml
# eureka-server/application.yml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

### Client
```yaml
# any microservice
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
```

> **Kubernetes alternative**: Use `spring-cloud-kubernetes` + native k8s Service discovery instead of Eureka.

---

## Spring Cloud Config Server

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {}
```
```yaml
# config-server/application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: ${CONFIG_REPO_URI}
          default-label: main
          search-paths: '{application}'
          clone-on-start: true
```

### Client
```yaml
# bootstrap.yml OR application.yml (with spring-cloud-starter-config)
spring:
  config:
    import: optional:configserver:${CONFIG_SERVER_URI:http://localhost:8888}
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
```

---

## OpenFeign (declarative HTTP client)

```java
@FeignClient(
    name = "inventory-service",
    fallbackFactory = InventoryClientFallback.class,
    configuration = FeignConfig.class)
public interface InventoryClient {

    @GetMapping("/api/v1/inventory/{productId}")
    InventoryResponse getInventory(@PathVariable UUID productId);

    @PostMapping("/api/v1/inventory/reserve")
    ReservationResponse reserve(@RequestBody ReserveRequest request);
}

// Fallback
@Component
public class InventoryClientFallback
    implements FallbackFactory<InventoryClient> {

    @Override
    public InventoryClient create(Throwable cause) {
        return new InventoryClient() {
            @Override
            public InventoryResponse getInventory(UUID productId) {
                throw new ServiceUnavailableException("Inventory service unavailable", cause);
            }
            @Override
            public ReservationResponse reserve(ReserveRequest r) {
                throw new ServiceUnavailableException("Inventory service unavailable", cause);
            }
        };
    }
}
```

```java
// Feign config — propagate tracing + auth headers
@Configuration
public class FeignConfig {

    @Bean
    RequestInterceptor tracingInterceptor() {
        return template -> {
            // Micrometer Tracing injects trace headers automatically
            // Add any custom headers here
        };
    }

    @Bean
    Logger.Level feignLogLevel() {
        return Logger.Level.BASIC;  // FULL in dev only
    }
}
```

---

## Resilience4j

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true

  retry:
    instances:
      inventory-service:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - feign.FeignException.ServiceUnavailable

  ratelimiter:
    instances:
      inventory-service:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 0s

  bulkhead:
    instances:
      inventory-service:
        max-concurrent-calls: 20
        max-wait-duration: 100ms
```

```java
@Service
public class InventoryServiceAdapter {

    @CircuitBreaker(name = "inventory-service", fallbackMethod = "fallbackStock")
    @Retry(name = "inventory-service")
    @Bulkhead(name = "inventory-service")
    public StockLevel getStock(UUID productId) {
        return inventoryClient.getInventory(productId).toStockLevel();
    }

    private StockLevel fallbackStock(UUID productId, CallNotPermittedException ex) {
        log.warn("Circuit open for inventory-service, returning UNKNOWN stock for {}", productId);
        return StockLevel.UNKNOWN;
    }
}
```

---

## Distributed Configuration Refresh

```java
// Refresh config without restart (with Spring Cloud Bus + Kafka/RabbitMQ)
@RefreshScope
@ConfigurationProperties(prefix = "feature")
public class FeatureFlags {
    private boolean newCheckoutEnabled;
}
```

```bash
# Trigger refresh on all instances via Bus
POST /actuator/busrefresh
```

---

## Service-to-Service Auth Pattern

```
Client → Gateway (validates JWT) → Service A (client_credentials) → Service B
```

1. Gateway validates end-user JWT (`TokenRelay` filter).
2. Service A calls Service B using a machine-to-machine `client_credentials` token (see security.md).
3. Service B validates the client token + checks `sub` claim for service identity.
