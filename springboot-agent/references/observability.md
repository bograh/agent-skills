# Observability Reference — Actuator, Micrometer, OpenTelemetry, Tracing

## The Three Pillars

| Pillar | Spring/Java tool | Backend |
|---|---|---|
| **Metrics** | Micrometer | Prometheus + Grafana |
| **Tracing** | Micrometer Tracing + OTel bridge | Zipkin / Jaeger / Tempo |
| **Logs** | Logback + structured JSON | Loki / ELK |

---

## Dependencies

```xml
<!-- Actuator -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Prometheus -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <scope>runtime</scope>
</dependency>

<!-- Tracing (OTel bridge) -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry.instrumentation</groupId>
  <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>

<!-- Export traces to Zipkin OR OTLP -->
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
  <scope>runtime</scope>
</dependency>
```

---

## application.yml

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,loggers,env,threaddump,heapdump
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true          # /actuator/health/liveness, /readiness
      group:
        readiness:
          include: db,redis,kafka
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s
    tags:
      application: ${spring.application.name}
      env: ${SPRING_PROFILES_ACTIVE:local}

  tracing:
    sampling:
      probability: ${TRACING_SAMPLE_RATE:0.1}   # 10% in prod; 1.0 in dev
    propagation:
      type: w3c    # W3C Trace Context (interoperable with OTel ecosystem)

  zipkin:
    tracing:
      endpoint: ${ZIPKIN_ENDPOINT:http://localhost:9411/api/v2/spans}
```

---

## Custom Metrics

```java
@Component
@RequiredArgsConstructor
public class OrderMetrics {

    private final MeterRegistry registry;
    private final Counter ordersPlaced;
    private final Timer orderProcessingTimer;

    public OrderMetrics(MeterRegistry registry) {
        this.registry = registry;
        this.ordersPlaced = Counter.builder("orders.placed")
            .description("Total orders placed")
            .tag("service", "order")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Order processing time")
            .publishPercentileHistogram()
            .register(registry);
    }

    public void recordOrderPlaced(String channel) {
        registry.counter("orders.placed", "channel", channel).increment();
    }

    public <T> T timeOrderProcessing(Supplier<T> fn) {
        return orderProcessingTimer.record(fn);
    }

    // Gauge — track in-flight
    @PostConstruct
    void registerGauges() {
        Gauge.builder("orders.in_flight", inFlightOrders, AtomicInteger::get)
            .register(registry);
    }
}
```

```java
// @Timed annotation (AOP-based)
@Service
public class OrderService {

    @Timed(value = "orders.place", percentiles = {0.5, 0.95, 0.99})
    public Order placeOrder(PlaceOrderCmd cmd) { ... }
}
```

---

## Structured Logging (Logback)

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="!local">
    <!-- JSON in non-local environments -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <customFields>{"service":"${spring.application.name}"}</customFields>
      </encoder>
    </appender>
    <root level="INFO"><appender-ref ref="JSON"/></root>
  </springProfile>

  <springProfile name="local">
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>%d{HH:mm:ss} %highlight(%-5level) [%X{traceId}] %cyan(%logger{36}) - %msg%n</pattern>
      </encoder>
    </appender>
    <root level="DEBUG"><appender-ref ref="CONSOLE"/></root>
  </springProfile>
</configuration>
```

```xml
<!-- pom.xml — logstash encoder -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>8.0</version>
</dependency>
```

---

## Tracing — propagation + custom spans

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {

    private final Tracer tracer;
    private final PaymentGatewayClient gatewayClient;

    public PaymentResult charge(ChargeRequest request) {
        // Create child span
        Span span = tracer.nextSpan().name("payment.charge")
            .tag("payment.method", request.method())
            .tag("payment.currency", request.currency())
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            log.info("Charging {} {}", request.amount(), request.currency());
            return gatewayClient.charge(request);
        } catch (Exception ex) {
            span.tag("error", ex.getMessage());
            throw ex;
        } finally {
            span.end();
        }
    }
}
```

---

## Health Indicators

```java
@Component
@RequiredArgsConstructor
public class KafkaHealthIndicator extends AbstractHealthIndicator {

    private final KafkaAdmin kafkaAdmin;

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            Map<String, Object> details = new LinkedHashMap<>();
            details.put("brokers", kafkaAdmin.describeCluster().nodes().get().size());
            builder.up().withDetails(details);
        } catch (Exception ex) {
            builder.down(ex);
        }
    }
}
```

---

## Grafana Dashboard Queries (PromQL examples)

```promql
# HTTP error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
  / rate(http_server_requests_seconds_count[5m])

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# Active DB connections
hikaricp_connections_active

# Cache hit rate
cache_gets_total{result="hit"} / cache_gets_total
```

---

## AlertManager Rules (example)

```yaml
groups:
  - name: order-service
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5..",application="order-service"}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for 2 minutes"

      - alert: SlowP99
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{application="order-service"}[5m])) > 1
        for: 5m
        annotations:
          summary: "P99 latency > 1s"
```
