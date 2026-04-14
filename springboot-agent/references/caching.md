# Caching Reference — Redis & Caffeine

## Spring Cache Abstraction

```xml
<!-- Redis -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- Caffeine (local) -->
<dependency>
  <groupId>com.github.ben-manes.caffeine</groupId>
  <artifactId>caffeine</artifactId>
</dependency>
```

---

## Redis Cache Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    RedisCacheManager cacheManager(RedisConnectionFactory cf) {
        var defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        return RedisCacheManager.builder(cf)
            .cacheDefaults(defaultConfig)
            .withCacheConfiguration("products",
                defaultConfig.entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("userSessions",
                defaultConfig.entryTtl(Duration.ofMinutes(30)))
            .transactionAware()   // syncs with @Transactional
            .build();
    }
}
```

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
```

---

## Caffeine (local, single-node)

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    CacheManager cacheManager() {
        var spec = CaffeineSpec.parse("maximumSize=1000,expireAfterWrite=5m,recordStats");
        return CaffeineCacheManager.builder()
            .caffeineSpec(spec)
            .build();
    }
}
```

---

## Cache Annotations

```java
@Service
@RequiredArgsConstructor
public class ProductCatalogService {

    // Cache on read — key defaults to method args
    @Cacheable(value = "products", key = "#id",
               condition = "#id != null",
               unless = "#result == null")
    @Transactional(readOnly = true)
    public Optional<Product> findById(UUID id) {
        return productRepository.findById(id);
    }

    // Evict on update
    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public Product update(Product product) {
        return productRepository.save(product);
    }

    // Update cache after write
    @CachePut(value = "products", key = "#result.id")
    @Transactional
    public Product create(CreateProductCmd cmd) {
        return productRepository.save(Product.from(cmd));
    }

    // Evict multiple caches
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productList", allEntries = true)
    })
    @Transactional
    public void delete(UUID id) {
        productRepository.deleteById(id);
    }
}
```

---

## Programmatic Cache (cache-aside pattern)

```java
@Service
@RequiredArgsConstructor
public class InventoryCache {

    private final RedisTemplate<String, StockLevel> redisTemplate;
    private final InventoryRepository inventoryRepository;

    private static final Duration TTL = Duration.ofMinutes(5);
    private static final String KEY_PREFIX = "inventory:stock:";

    public StockLevel getStock(UUID productId) {
        String key = KEY_PREFIX + productId;
        StockLevel cached = redisTemplate.opsForValue().get(key);
        if (cached != null) return cached;

        StockLevel fresh = inventoryRepository.getStockLevel(productId);
        redisTemplate.opsForValue().set(key, fresh, TTL);
        return fresh;
    }

    public void invalidate(UUID productId) {
        redisTemplate.delete(KEY_PREFIX + productId);
    }
}
```

---

## Distributed Lock (Redis)

```java
// Use Redisson for production-grade distributed locks
@Bean
RedissonClient redissonClient(@Value("${spring.data.redis.host}") String host,
                               @Value("${spring.data.redis.port}") int port) {
    Config config = new Config();
    config.useSingleServer().setAddress("redis://" + host + ":" + port);
    return Redisson.create(config);
}

@Service
@RequiredArgsConstructor
public class StockReservationService {

    private final RedissonClient redissonClient;

    public void reserve(UUID productId, int quantity) {
        RLock lock = redissonClient.getLock("lock:product:" + productId);
        try {
            if (!lock.tryLock(5, 10, TimeUnit.SECONDS))
                throw new ConcurrentModificationException("Could not acquire lock");
            // critical section
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }
    }
}
```

---

## Cache Warming

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheWarmingRunner implements ApplicationRunner {

    private final ProductCatalogService catalogService;
    private final CacheManager cacheManager;

    @Override
    public void run(ApplicationArguments args) {
        log.info("Warming product cache...");
        catalogService.findTopProducts(100)
            .forEach(p -> catalogService.findById(p.getId())); // populates cache
        log.info("Cache warmed");
    }
}
```

---

## Caching Best Practices

| Practice | Why |
|---|---|
| Always set a TTL | Prevents stale data permanently residing in cache |
| Cache at service layer, not repository | Repository results may include unpopulated lazy fields |
| Null-safe: `unless="#result==null"` | Avoids caching empty results (negative caching requires explicit TTL) |
| Use structured cache keys | `"entity:type:id"` prevents collisions across services |
| Monitor hit rate via Actuator | Low hit rate may indicate too-short TTL or poor key design |
| Avoid caching mutable collections | Race conditions; cache individual entities instead |
| Redis for cluster/distributed | Caffeine for single-node or L1 cache in front of Redis |

---

## application.yml — cache metrics

```yaml
management:
  metrics:
    cache:
      instrument:
        auto: true    # auto-instruments all CacheManager beans
```
