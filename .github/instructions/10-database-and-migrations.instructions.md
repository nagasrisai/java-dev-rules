---
applyTo: "**/*.java"
---

# Java Database and Migration Rules

## Entity Design Standards

### JPA Entity Rules

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA requires no-arg; protect it
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "reference_number", nullable = false, unique = true, length = 50)
    private String referenceNumber;

    @Column(name = "total_amount", nullable = false, precision = 19, scale = 4)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 30)
    private OrderStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @Version
    private Long version;  // Optimistic locking — always include on entities that are updated

    // Factory method — protect the constructor
    public static Order create(String referenceNumber, Customer customer) {
        Order order = new Order();
        order.referenceNumber = referenceNumber;
        order.customer = customer;
        order.status = OrderStatus.PENDING;
        order.totalAmount = BigDecimal.ZERO;
        return order;
    }
}
```

### Entity Design Rules

- Always use `FetchType.LAZY` on `@ManyToOne` and `@OneToMany` — never `EAGER`.
- Use `@Version` for optimistic locking on all entities that are updated concurrently.
- Protect the JPA no-arg constructor (`ACCESS.PROTECTED`) — prevent direct instantiation.
- Use factory methods instead of public constructors for controlled creation.
- Map money as `BigDecimal` with `precision = 19, scale = 4` — never `double` or `float`.
- Map enums as `EnumType.STRING` — never `ORDINAL` (ordinal breaks when enum order changes).
- Use `@Column` on every field with explicit `name`, `nullable`, and `length`.
- Never use bidirectional relationships unless specifically required — prefer unidirectional.
- Include `createdAt` and `updatedAt` on every entity using `@CreationTimestamp` / `@UpdateTimestamp`.

### What Never to Do with Entities

- Never return an entity directly from a REST controller — always map to a DTO.
- Never call `save()` inside a loop — use `saveAll()` for batch operations.
- Never use `findAll()` without pagination on tables with significant data volume.

---

## Repository Standards

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Use Spring Data method names for simple queries
    Optional<Order> findByReferenceNumber(String referenceNumber);

    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);

    // Use @Query for anything complex — always with named parameters
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId AND o.status = :status")
    List<Order> findOrdersWithItemsByCustomerAndStatus(
        @Param("customerId") Long customerId,
        @Param("status") OrderStatus status
    );

    // Always paginate list queries
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);

    // Use Projections to load only needed fields
    @Query("SELECT o.id AS id, o.referenceNumber AS referenceNumber, o.status AS status FROM Order o WHERE o.customer.id = :customerId")
    List<OrderSummaryProjection> findOrderSummariesByCustomerId(@Param("customerId") Long customerId);
}
```

### Repository Rules

- Always return `Optional<T>` from single-result finder methods — never return `null`.
- Always paginate list results using `Pageable` — never return `List<T>` for unbounded queries.
- Use `JOIN FETCH` in `@Query` to avoid N+1 queries when associations are needed.
- Use projections (`interface` projections or `record` projections) to load partial data efficiently.
- Use `@Modifying` + `@Query` for bulk updates — never load all entities, update, and save in a loop.
- Always annotate bulk update / delete queries with `@Transactional` at the service layer.

---

## Database Migration Standards

### Migration Tool

Use **Flyway** (preferred) or **Liquibase**. The choice is per-project — do not mix them.

### Flyway File Naming Convention

```
src/main/resources/db/migration/
  V1__create_customers_table.sql
  V2__create_orders_table.sql
  V3__add_status_index_to_orders.sql
  V4__add_order_items_table.sql
  V5__add_email_unique_constraint_to_customers.sql
```

Rules:
- `V{number}__{description}.sql` — two underscores between version and description.
- Version numbers are sequential integers — never use decimals or dates as versions.
- Description uses underscores, all lowercase.
- Each file is immutable once applied — never modify an existing migration file.
- One logical change per migration file — do not bundle unrelated changes.

### Migration Script Standards

```sql
-- V3__add_status_index_to_orders.sql

-- Add index to support queries filtering by status and customer_id
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id_status
    ON orders (customer_id, status);
```

```sql
-- V4__add_order_items_table.sql

CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT        NOT NULL REFERENCES orders (id) ON DELETE CASCADE,
    product_id  BIGINT        NOT NULL REFERENCES products (id),
    quantity    INTEGER       NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(19,4) NOT NULL,
    created_at  TIMESTAMP     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

### Migration Rules

- [ ] Never use `DROP TABLE` or `DROP COLUMN` without an explicit data retention review.
- [ ] Use `CREATE INDEX CONCURRENTLY` on large tables to avoid locking.
- [ ] Always define foreign keys with explicit `ON DELETE` behaviour (`CASCADE`, `SET NULL`, `RESTRICT`).
- [ ] Include `NOT NULL` constraints on all columns that must always have a value.
- [ ] Test the migration on a copy of production data volume before deploying.
- [ ] Roll-forward approach only — fix a bad migration with a new forward migration, never edit the old one.
- [ ] For backward-incompatible schema changes (column rename, type change), use an expand-and-contract approach:
  1. **Expand**: Add the new column alongside the old one.
  2. **Migrate**: Backfill the new column; deploy application code that writes to both.
  3. **Contract**: Remove the old column in a separate migration once all reads are off it.

---

## Transaction Management

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    @Transactional                    // Read-write transaction
    public Order placeOrder(PlaceOrderRequest request) {
        Order order = Order.create(generateReference(), findCustomer(request.customerId()));
        inventoryService.reserve(request.items());
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)   // Read-only — Hibernate skips dirty check, faster
    public Page<OrderSummaryProjection> findCustomerOrders(Long customerId, Pageable pageable) {
        return orderRepository.findOrderSummariesByCustomerId(customerId, pageable);
    }
}
```

### Transaction Rules

- Annotate write operations with `@Transactional` at the service layer — not the repository or controller.
- Annotate read-only operations with `@Transactional(readOnly = true)` for performance.
- Keep transactions as short as possible — do not include HTTP calls or slow I/O inside a transaction.
- Never call a `@Transactional` method from within the same class — Spring AOP proxies do not intercept self-calls.
- For long-running batch operations, use pagination and process in chunks within separate transactions.

---

## Query Performance Rules

- Run `EXPLAIN ANALYZE` on any query that operates on a table with more than 10,000 rows.
- Every foreign key column must have an index unless there is a documented reason not to.
- Composite indexes — order columns by selectivity: most selective (highest cardinality) first.
- Avoid `SELECT *` in native queries — specify only the columns you need.
- Use `LIMIT` in any query that could return more than 100 rows.
- Batch inserts: use `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `@BatchSize`.
