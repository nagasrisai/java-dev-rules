---
applyTo: "**/*.java"
---

# Java Performance Testing Rules

## When Performance Testing Is Required

Performance testing is mandatory when:

- A new public API endpoint is added.
- An existing endpoint's implementation is significantly changed.
- A new database table or query is introduced on a high-traffic path.
- A new caching layer is added or removed.
- A message consumer or batch job is introduced.
- The ticket explicitly mentions performance requirements.

---

## Performance Targets — Define Before Testing

Before running any test, document the required targets in the Jira ticket:

| Metric | Typical Target | Notes |
|---|---|---|
| p95 response time | < 500 ms | Under expected production load |
| p99 response time | < 1000 ms | Tail latency target |
| Throughput | > X req/s | Match expected peak load |
| Error rate | < 0.1% | Under load |
| Memory usage | Stable (no leak) | After sustained load |
| GC pause time | < 100 ms | No full GC under steady load |

If the ticket does not define targets, ask the product owner or tech lead before testing.

---

## Load Testing Tools

### Gatling (Recommended for Java projects)

Gatling tests are written in Java (or Scala/Kotlin) and integrate with Maven/Gradle.

```java
// src/gatling/java/simulations/OrderApiSimulation.java
public class OrderApiSimulation extends Simulation {

    private final HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");

    private final ScenarioBuilder createOrderScenario = scenario("Create Order")
        .exec(
            http("POST /api/v1/orders")
                .post("/api/v1/orders")
                .body(StringBody("""
                    {"customerId": 1, "items": [{"productId": 10, "quantity": 2}]}
                    """))
                .check(status().is(201))
                .check(responseTimeInMillis().lte(500))
        );

    {
        setUp(
            createOrderScenario.injectOpen(
                rampUsersPerSec(10).to(100).during(Duration.ofSeconds(60)),
                constantUsersPerSec(100).during(Duration.ofMinutes(5))
            )
        ).protocols(httpProtocol)
         .assertions(
             global().responseTime().percentile3().lte(500),   // p95 < 500ms
             global().responseTime().percentile4().lte(1000),  // p99 < 1000ms
             global().failedRequests().percent().lte(0.1)      // < 0.1% errors
         );
    }
}
```

### JMeter (Alternative — UI-driven)

Use for exploratory load testing or when a non-developer needs to run tests.
Store JMeter test plans (`.jmx` files) in `src/test/jmeter/`.
Run via Maven JMeter Plugin: `mvn jmeter:jmeter jmeter:results`.

---

## Profiling Java Applications

### When to Profile

Profile when:
- A test shows p95 > target and the cause is not obvious from metrics.
- Memory usage grows steadily (suspected leak).
- CPU usage is unexpectedly high under load.
- GC pauses are long and frequent.

### Profiling Tools

| Tool | Purpose |
|---|---|
| async-profiler | CPU and allocation profiling — minimal overhead, production-safe |
| JProfiler | Full IDE-integrated profiler — development use |
| VisualVM | Free, good for heap dump analysis |
| Flight Recorder (JFR) | Built into JVM — low overhead, production-safe |

### Running async-profiler

```bash
# Attach to running JVM process (find PID first)
jps -l

# CPU profile for 30 seconds, output as flame graph HTML
./asprof -d 30 -f /tmp/flamegraph.html <PID>

# Allocation profile
./asprof -e alloc -d 30 -f /tmp/alloc.html <PID>
```

### JVM Flags for Profiling

```bash
# Enable JFR for continuous low-overhead profiling
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=profile.jfr \
     -jar app.jar

# Heap dump on OutOfMemoryError (always enable in production)
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -jar app.jar
```

---

## Database Query Performance

### Analysing Slow Queries

Enable Hibernate statistics in development to identify slow / N+1 queries:

```yaml
# application-development.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100
logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

Run `EXPLAIN ANALYZE` for any query on tables with significant data:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'PENDING'
ORDER BY o.created_at DESC
LIMIT 20;
```

Look for:
- `Seq Scan` on large tables — add an index.
- `Nested Loop` with many loops — consider a hash join or a denormalized read model.
- High `actual rows` vs `estimated rows` — run `ANALYZE` to update statistics.

---

## Caching Strategy

Use caching to reduce load on the database or external services for frequently read, rarely changed data.

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    @Cacheable(value = "products", key = "#productId", unless = "#result == null")
    @Transactional(readOnly = true)
    public Optional<ProductDto> findById(Long productId) {
        return productRepository.findById(productId).map(this::toDto);
    }

    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public ProductDto update(Long id, UpdateProductRequest request) {
        // cache entry is evicted when the product is updated
        ...
    }
}
```

### Caching Rules

- Cache read results that are expensive to compute and change infrequently.
- Always evict or update cache entries on write operations.
- Set a TTL (time-to-live) on every cache entry — never cache indefinitely.
- Do not cache user-specific or sensitive data in shared caches without careful key isolation.
- Monitor cache hit/miss ratio with Micrometer metrics — low hit ratio means the cache is not helping.

---

## JVM Tuning (Production)

### Recommended JVM Flags for Containerised Spring Boot Applications

```bash
java \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:+UseStringDeduplication \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp \
  -XX:+ExitOnOutOfMemoryError \
  -Djava.security.egd=file:/dev/./urandom \
  -jar app.jar
```

### Container Memory Sizing

For containerised applications, always set JVM heap relative to container memory:

```bash
# Let JVM auto-size heap based on container limits (Java 10+)
java -XX:MaxRAMPercentage=75.0 -jar app.jar
```

Rule: Set `resources.limits.memory` in Kubernetes to 2x the expected heap size to account for off-heap memory (metaspace, thread stacks, native libraries).

---

## Performance Test Reporting

After every load test run, document the results in the Jira ticket or PR:

```markdown
## Performance Test Results

**Test Date:** 2026-03-18  
**Environment:** staging  
**Duration:** 10 minutes  
**Load Profile:** Ramp from 10 to 100 req/s over 2 min, then steady 100 req/s for 8 min

| Metric | Target | Result | Pass? |
|---|---|---|---|
| p95 response time | < 500 ms | 310 ms | ✅ |
| p99 response time | < 1000 ms | 480 ms | ✅ |
| Throughput | > 90 req/s | 98 req/s | ✅ |
| Error rate | < 0.1% | 0.0% | ✅ |
| Memory (heap) | Stable | 220 MB stable | ✅ |
| GC pause (p99) | < 100 ms | 42 ms | ✅ |

**Conclusion:** Passed all targets. Safe to deploy.
```
