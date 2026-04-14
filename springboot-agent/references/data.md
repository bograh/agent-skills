# Spring Data Reference

## Spring Data JPA

### Entity Best Practices
```java
@Entity
@Table(name = "products",
       indexes = @Index(name = "idx_product_sku", columnList = "sku"))
@EntityListeners(AuditingEntityListener.class)
public class Product {

    @Id @UuidGenerator
    private UUID id;

    @Column(nullable = false, unique = true, length = 64)
    private String sku;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    // Auditing (enable via @EnableJpaAuditing)
    @CreatedDate   private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy     private String createdBy;
    @LastModifiedBy private String lastModifiedBy;

    @Version private Long version;   // optimistic locking

    // equals/hashCode on business key, not surrogate id
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Product p)) return false;
        return id != null && id.equals(p.id);
    }
    @Override public int hashCode() { return getClass().hashCode(); }
}
```

### Repository (infrastructure implements domain interface)
```java
// Infrastructure adapter
@Repository
public interface JpaOrderRepository
    extends JpaRepository<Order, UUID>, OrderRepository {

    // Projection — avoid loading full entity
    List<OrderSummary> findByCustomerId(UUID customerId);

    // Derived query with Pageable
    Page<Order> findByStatusAndCreatedAtAfter(
        OrderStatus status, Instant since, Pageable pageable);

    // JPQL — prefer over native when possible
    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") UUID id);

    // Native SQL (use sparingly, document why)
    @Query(value = "SELECT * FROM orders WHERE status = ?1 FOR UPDATE SKIP LOCKED LIMIT ?2",
           nativeQuery = true)
    List<Order> claimForProcessing(String status, int limit);

    // Modifying (always add @Transactional in service layer, not here)
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    int updateStatus(@Param("id") UUID id, @Param("status") OrderStatus status);
}
```

### Specification (dynamic queries)
```java
public class OrderSpecifications {
    public static Specification<Order> hasStatus(OrderStatus s) {
        return (root, q, cb) -> s == null ? null : cb.equal(root.get("status"), s);
    }
    public static Specification<Order> createdAfter(Instant t) {
        return (root, q, cb) -> t == null ? null : cb.greaterThan(root.get("createdAt"), t);
    }
}

// Usage in service
orderRepository.findAll(
    Specification.where(hasStatus(CONFIRMED)).and(createdAfter(since)), pageable);
```

### Projections
```java
// Interface projection (Spring generates proxy)
public interface OrderSummary {
    UUID getId();
    String getStatus();
    @Value("#{target.total + ' ' + target.currency}")
    String getFormattedTotal();
}

// Class projection (DTO — better performance)
public record OrderSummaryDto(UUID id, String status, BigDecimal total) {}
```

### Auditing Config
```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

## Flyway Migrations

```
resources/
  db/migration/
    V1__create_schema.sql
    V2__add_orders_table.sql
    R__refresh_order_summary_view.sql   # repeatable
```

```sql
-- V2__add_orders_table.sql
CREATE TABLE orders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id VARCHAR(36) NOT NULL,
    status      VARCHAR(32) NOT NULL DEFAULT 'DRAFT',
    version     BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status   ON orders(status) WHERE status <> 'COMPLETED';
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    out-of-order: false
    validate-on-migrate: true
```

## Spring Data MongoDB

```java
@Document(collection = "events")
public class DomainEvent {

    @Id private String id;          // String maps to ObjectId
    @Field("aggregate_id") private UUID aggregateId;
    @Field("event_type")   private String eventType;
    @Field("payload")      private Map<String, Object> payload;
    @CreatedDate           private Instant occurredAt;
}

@Repository
public interface EventRepository extends MongoRepository<DomainEvent, String> {
    List<DomainEvent> findByAggregateIdOrderByOccurredAtAsc(UUID aggregateId);
}
```

## Spring Data Redis

```java
// Keyspace + TTL
@RedisHash(value = "sessions", timeToLive = 3600)
public class UserSession {
    @Id private String sessionId;
    private String userId;
    private Set<String> roles;
}

@Repository
public interface UserSessionRepository
    extends CrudRepository<UserSession, String> {}
```

## Transactions

```java
// Always annotate at service layer (not repository)
@Transactional                          // readOnly = false by default
public Order createOrder(CreateOrderCmd cmd) { ... }

@Transactional(readOnly = true)         // performance boost for reads
public Page<Order> listOrders(Pageable p) { ... }

@Transactional(propagation = REQUIRES_NEW)   // separate tx (e.g. audit log)
public void recordAuditEvent(AuditEvent e) { ... }

// Programmatic (for conditional commits)
@Autowired TransactionTemplate tx;
tx.execute(status -> {
    // ... do work ...
    if (problem) status.setRollbackOnly();
    return result;
});
```

## Common Pitfalls

| Problem | Fix |
|---|---|
| `open-in-view = true` (default) | Set `spring.jpa.open-in-view: false` |
| N+1 selects | `JOIN FETCH` or `@EntityGraph` |
| `LazyInitializationException` | Fetch in repository, or use projection |
| `equals`/`hashCode` on mutable fields | Use business key or `@NaturalId` |
| Long-running tx holding DB connection | Break into smaller txs or use async |
