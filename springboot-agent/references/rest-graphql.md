# REST & GraphQL Reference

## REST Controllers

### Standard Controller Pattern
```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Orders", description = "Order management")  // Swagger
public class OrderController {

    private final PlaceOrderPort placeOrderPort;
    private final GetOrderPort getOrderPort;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Place a new order")
    public OrderResponse placeOrder(
        @Valid @RequestBody PlaceOrderRequest request,
        Authentication auth) {

        log.info("Placing order for customer={}", auth.getName());
        var cmd = PlaceOrderCommand.from(request, auth.getName());
        return OrderResponse.from(placeOrderPort.execute(cmd));
    }

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable UUID id) {
        return getOrderPort.execute(id)
            .map(OrderResponse::from)
            .orElseThrow(() -> new NotFoundException("Order not found: " + id));
    }

    @GetMapping
    public Page<OrderSummaryResponse> listOrders(
        @RequestParam(required = false) OrderStatus status,
        @PageableDefault(size = 20, sort = "createdAt", direction = DESC) Pageable pageable) {

        return getOrderPort.list(status, pageable).map(OrderSummaryResponse::from);
    }

    @PatchMapping("/{id}/cancel")
    public OrderResponse cancelOrder(@PathVariable UUID id) {
        return OrderResponse.from(cancelOrderPort.execute(id));
    }
}
```

### Global Exception Handler (RFC 7807 ProblemDetail)
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    ProblemDetail handleNotFound(NotFoundException ex) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        return pd;
    }

    @ExceptionHandler(ConflictException.class)
    ProblemDetail handleConflict(ConflictException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY,
            "Validation failed");
        pd.setProperty("errors", ex.getFieldErrors().stream()
            .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
            .toList());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail handleGeneric(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception for {}", req.getRequestURI(), ex);
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred");
    }
}
```

### Bean Validation on DTOs
```java
public record PlaceOrderRequest(
    @NotBlank(message = "customerId is required") String customerId,
    @NotEmpty @Size(max = 50) List<@Valid OrderItemRequest> items,
    @NotNull ShippingAddressRequest shippingAddress
) {}

public record OrderItemRequest(
    @NotBlank String productId,
    @Min(1) @Max(100) int quantity
) {}

// Custom validator
@Target(FIELD) @Retention(RUNTIME) @Constraint(validatedBy = IsoDateValidator.class)
public @interface IsoDate {
    String message() default "Must be ISO-8601 date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### Versioning Strategy

Option A — URI versioning (simpler, preferred):
```
/api/v1/orders
/api/v2/orders
```

Option B — Header versioning:
```java
@GetMapping(value = "/orders", headers = "X-API-Version=2")
```

### Hypermedia (HAL) — when needed
```java
// Add spring-boot-starter-hateoas
@GetMapping("/{id}")
EntityModel<OrderResponse> getOrder(@PathVariable UUID id) {
    var order = OrderResponse.from(...);
    return EntityModel.of(order,
        linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel(),
        linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
}
```

---

## GraphQL (Spring for GraphQL)

### Schema Definition
```graphql
# resources/graphql/schema.graphqls
type Query {
    order(id: ID!): Order
    orders(filter: OrderFilter, page: Int = 0, size: Int = 20): OrderPage!
}

type Mutation {
    placeOrder(input: PlaceOrderInput!): Order!
    cancelOrder(id: ID!): Order!
}

type Subscription {
    orderStatusChanged(orderId: ID!): OrderStatusEvent!
}

type Order {
    id: ID!
    status: OrderStatus!
    lines: [OrderLine!]!
    total: Float!
    customer: Customer     # resolved via DataLoader
    createdAt: String!
}

enum OrderStatus { DRAFT CONFIRMED SHIPPED DELIVERED CANCELLED }

input PlaceOrderInput {
    customerId: ID!
    items: [OrderItemInput!]!
}
```

### Controller
```java
@Controller
@RequiredArgsConstructor
public class OrderGraphQlController {

    private final PlaceOrderPort placeOrderPort;
    private final GetOrderPort getOrderPort;

    @QueryMapping
    public Optional<Order> order(@Argument UUID id) {
        return getOrderPort.execute(id);
    }

    @QueryMapping
    public Connection<Order> orders(@Argument OrderFilter filter,
                                    ScrollPosition scrollPosition) {
        // Cursor-based pagination via Spring GraphQL ScrollPosition
        return getOrderPort.listCursor(filter, scrollPosition);
    }

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Order placeOrder(@Argument PlaceOrderInput input, Authentication auth) {
        return placeOrderPort.execute(PlaceOrderCommand.from(input, auth.getName()));
    }

    @SubscriptionMapping
    public Flux<OrderStatusEvent> orderStatusChanged(@Argument UUID orderId) {
        return orderEventPublisher.subscribe(orderId);
    }

    // DataLoader — prevents N+1
    @SchemaMapping(typeName = "Order", field = "customer")
    public CompletableFuture<Customer> customer(Order order, DataLoader<UUID, Customer> loader) {
        return loader.load(order.getCustomerId());
    }
}
```

### DataLoader Registration
```java
@Configuration
public class DataLoaderConfig {

    @Bean
    BatchLoaderRegistry batchLoaderRegistry(CustomerService customerService) {
        return registrar -> registrar
            .forTypePair(UUID.class, Customer.class)
            .withName("customerLoader")
            .registerBatchLoader((ids, env) ->
                Mono.fromCallable(() -> customerService.findAllByIds(ids)));
    }
}
```

### GraphQL application.yml
```yaml
spring:
  graphql:
    graphiql:
      enabled: true             # disable in prod
      path: /graphiql
    websocket:
      path: /graphql            # subscriptions
    schema:
      printer:
        enabled: true           # log schema on startup
```

### GraphQL Security
```java
// Per-field security
@SchemaMapping(typeName = "Order", field = "internalNotes")
@PreAuthorize("hasRole('SUPPORT')")
public String internalNotes(Order order) { ... }
```

---

## OpenAPI / Swagger (springdoc)

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.6.0</version>
</dependency>
```

```java
@OpenAPIDefinition(
    info = @Info(title = "Order Service API", version = "v1"),
    security = @SecurityRequirement(name = "bearerAuth"))
@SecurityScheme(name = "bearerAuth", type = HTTP, scheme = "bearer", bearerFormat = "JWT")
@Configuration
public class OpenApiConfig {}
```

```yaml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
```
