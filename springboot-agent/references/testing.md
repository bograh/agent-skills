# Testing Reference — Unit, Slice, Integration, Contract

## Test Pyramid Strategy

```
         /\
        /E2E\           few — Playwright / RestAssured against deployed env
       /------\
      /Contract\        per service — Spring Cloud Contract / Pact
     /----------\
    / Integration\      per feature — @SpringBootTest + Testcontainers
   /--------------\
  /   Slice Tests   \   per layer — @WebMvcTest, @DataJpaTest, @JsonTest
 /------------------\
/ Unit Tests (most)  \  per class — JUnit 5 + Mockito; no Spring context
```

---

## Unit Tests

```java
@ExtendWith(MockitoExtension.class)
class PlaceOrderUseCaseTest {

    @Mock OrderRepository orderRepository;
    @Mock InventoryPort inventoryPort;
    @Mock EventPublisher eventPublisher;
    @InjectMocks PlaceOrderUseCase useCase;

    @Test
    void shouldPlaceOrderSuccessfully() {
        // Arrange
        var cmd = new PlaceOrderCommand("cust-1", List.of(new OrderItem("prod-1", 2)));
        when(inventoryPort.isAvailable("prod-1", 2)).thenReturn(true);
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        var result = useCase.execute(cmd);

        // Assert
        assertThat(result.status()).isEqualTo(OrderStatus.CONFIRMED);
        verify(eventPublisher).publish(argThat(e ->
            e instanceof OrderPlacedEvent && ((OrderPlacedEvent)e).customerId().equals("cust-1")));
    }

    @Test
    void shouldThrowWhenItemOutOfStock() {
        when(inventoryPort.isAvailable(any(), anyInt())).thenReturn(false);
        assertThatThrownBy(() -> useCase.execute(validCmd()))
            .isInstanceOf(OutOfStockException.class);
    }
}
```

---

## Slice Tests

### @WebMvcTest — Controller only
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean PlaceOrderPort placeOrderPort;
    @MockBean GetOrderPort getOrderPort;
    @Autowired ObjectMapper objectMapper;

    @Test
    @WithMockUser(roles = "USER")
    void shouldReturn201WhenOrderPlaced() throws Exception {
        var request = new PlaceOrderRequest("cust-1",
            List.of(new OrderItemRequest("prod-1", 2)), validAddress());
        var response = new OrderResponse(UUID.randomUUID(), "CONFIRMED", ...);

        when(placeOrderPort.execute(any())).thenReturn(response);

        mockMvc.perform(post("/api/v1/orders")
                .contentType(APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("CONFIRMED"))
            .andExpect(header().exists("Location"));
    }

    @Test
    void shouldReturn401WhenUnauthenticated() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .contentType(APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser
    void shouldReturn422WhenInvalidRequest() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .contentType(APPLICATION_JSON)
                .content("{\"customerId\":\"\"}"))  // blank customerId
            .andExpect(status().isUnprocessableEntity())
            .andExpect(jsonPath("$.errors[0].field").value("customerId"));
    }
}
```

### @DataJpaTest — Repository only
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)  // use real DB via Testcontainers
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired JpaOrderRepository repository;

    @Test
    void shouldFindOrdersByCustomerId() {
        var order = Order.create(new CustomerId("cust-1"), List.of(validLine()));
        repository.save(order);

        var found = repository.findByCustomerId(new CustomerId("cust-1"), Pageable.ofSize(10));
        assertThat(found).hasSize(1);
    }
}
```

---

## Integration Tests (Testcontainers)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired TestRestTemplate restTemplate;

    @Test
    void fullOrderFlow() {
        var request = new PlaceOrderRequest(...);
        var response = restTemplate.withBasicAuth("user", "pass")
            .postForEntity("/api/v1/orders", request, OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().status()).isEqualTo("CONFIRMED");
    }
}
```

---

## Spring Cloud Contract (consumer-driven)

```groovy
// contracts/order-service/placeOrder.groovy
Contract.make {
    request {
        method POST()
        url '/api/v1/orders'
        headers { contentType applicationJson() }
        body([customerId: 'cust-1', items: [[productId: 'prod-1', quantity: 2]]])
    }
    response {
        status CREATED()
        headers { contentType applicationJson() }
        body([status: 'CONFIRMED'])
    }
}
```

```java
// Provider test (auto-generated base class)
@SpringBootTest(webEnvironment = RANDOM_PORT)
public abstract class OrderContractBase {
    @Autowired WebApplicationContext context;

    @BeforeEach
    void setup() { RestAssuredMockMvc.webAppContextSetup(context); }
}
```

---

## Test Utilities

```java
// application-test.yml
spring:
  jpa:
    hibernate.ddl-auto: create-drop  # fresh schema per test run
  kafka:
    consumer.auto-offset-reset: earliest
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://mock-auth-server  # Wiremock or Spring Authorization Server test

// Custom @WithMockJwt annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = MockJwtSecurityContextFactory.class)
public @interface WithMockJwt {
    String sub() default "test-user";
    String[] roles() default {"USER"};
}
```

---

## Coverage & Quality Gates

```kotlin
// build.gradle.kts
tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    reports { xml.required = true }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit { minimum = "0.80".toBigDecimal() }  // 80% line coverage
        }
    }
}
```
