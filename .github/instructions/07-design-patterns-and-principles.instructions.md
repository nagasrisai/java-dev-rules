---
applyTo: "**/*.java"
---

# Java Design Patterns and Principles Rules

## Foundational Principles

### SOLID Principles — Applied

#### Single Responsibility Principle (SRP)

Every class has exactly one reason to change.

```java
// Wrong — UserService does too many things
class UserService {
    public void createUser(User user) { ... }
    public void sendWelcomeEmail(User user) { ... }   // Email concern
    public void logAuditEvent(User user) { ... }       // Audit concern
}

// Right — each class has one responsibility
class UserService {
    public void createUser(User user) { ... }
}
class EmailNotificationService {
    public void sendWelcomeEmail(User user) { ... }
}
class AuditService {
    public void logUserCreated(User user) { ... }
}
```

#### Open/Closed Principle (OCP)

Open for extension, closed for modification. Use interfaces and polymorphism — not `if/else` chains.

```java
// Wrong — add a new payment type = modify existing code
if (type.equals("CARD")) { processCard(); }
else if (type.equals("PAYPAL")) { processPaypal(); }

// Right — add a new type = add a new class
interface PaymentProcessor {
    void process(Payment payment);
}
class CardPaymentProcessor implements PaymentProcessor { ... }
class PaypalPaymentProcessor implements PaymentProcessor { ... }
```

#### Liskov Substitution Principle (LSP)

Subclasses must be substitutable for their parent types without breaking correctness.

- Never throw new checked exceptions in overriding methods.
- Never weaken preconditions or strengthen postconditions in subclasses.
- Prefer composition over inheritance when LSP would be violated.

#### Interface Segregation Principle (ISP)

Many specific interfaces are better than one large general interface.

```java
// Wrong — clients that only read are forced to implement write methods
interface UserRepository {
    User findById(Long id);
    void save(User user);
    void delete(Long id);
}

// Right — split by client need
interface UserReadRepository { User findById(Long id); }
interface UserWriteRepository { void save(User user); void delete(Long id); }
```

#### Dependency Inversion Principle (DIP)

High-level modules depend on abstractions, not concretions.

```java
// Wrong
class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository();
}

// Right — depend on the interface, inject the implementation
class OrderService {
    private final OrderRepository repository;
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

## Design Patterns — When and How to Use Them

### Creational Patterns

#### Builder Pattern

Use when constructing objects with many optional parameters.

```java
@Builder
@Value
public class CreateUserRequest {
    String firstName;
    String lastName;
    String email;
    String phoneNumber;  // optional
    LocalDate dateOfBirth;  // optional
}

// Usage
CreateUserRequest request = CreateUserRequest.builder()
    .firstName("Alice")
    .lastName("Smith")
    .email("alice@example.com")
    .build();
```

#### Factory / Factory Method Pattern

Use when creation logic is complex or varies by type.

```java
public interface NotificationSender {
    void send(Notification notification);
}

public class NotificationSenderFactory {
    public NotificationSender create(NotificationType type) {
        return switch (type) {
            case EMAIL -> new EmailNotificationSender();
            case SMS -> new SmsNotificationSender();
            case PUSH -> new PushNotificationSender();
        };
    }
}
```

### Structural Patterns

#### Adapter Pattern

Use to integrate with an external system whose interface does not match your domain.

```java
// External library interface
class LegacyPaymentGateway {
    String executePayment(String amount, String currency, String cardToken) { ... }
}

// Your domain interface
interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}

// Adapter bridges them
class LegacyPaymentGatewayAdapter implements PaymentGateway {
    private final LegacyPaymentGateway legacy;

    @Override
    public PaymentResult charge(Money amount, PaymentMethod method) {
        String result = legacy.executePayment(
            amount.getValue().toString(),
            amount.getCurrency().code(),
            method.getToken()
        );
        return PaymentResult.fromLegacyResponse(result);
    }
}
```

#### Decorator Pattern

Use to add behaviour to objects dynamically without modifying them.

```java
interface OrderRepository {
    Optional<Order> findById(Long id);
}

class CachedOrderRepository implements OrderRepository {
    private final OrderRepository delegate;
    private final Cache<Long, Order> cache;

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(cache.getIfPresent(id))
            .or(() -> {
                Optional<Order> order = delegate.findById(id);
                order.ifPresent(o -> cache.put(id, o));
                return order;
            });
    }
}
```

### Behavioural Patterns

#### Strategy Pattern

Use when an algorithm or behaviour varies and the variation should be encapsulated.

```java
interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

class PercentageDiscount implements DiscountStrategy {
    private final BigDecimal percentage;
    @Override
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(BigDecimal.ONE.subtract(percentage));
    }
}

class FlatDiscount implements DiscountStrategy {
    private final BigDecimal amount;
    @Override
    public BigDecimal apply(BigDecimal price) {
        return price.subtract(amount).max(BigDecimal.ZERO);
    }
}
```

#### Observer / Event-Driven Pattern

Use for loosely coupled communication between components.

```java
// Spring Application Events
@Component
class UserService {
    private final ApplicationEventPublisher eventPublisher;

    public User createUser(CreateUserRequest request) {
        User user = userRepository.save(map(request));
        eventPublisher.publishEvent(new UserCreatedEvent(this, user));
        return user;
    }
}

@Component
class WelcomeEmailListener {
    @EventListener
    public void onUserCreated(UserCreatedEvent event) {
        emailService.sendWelcomeEmail(event.getUser());
    }
}
```

#### Template Method Pattern

Use when an algorithm has a fixed skeleton but individual steps vary.

```java
abstract class ReportGenerator {
    // Template method — defines the algorithm skeleton
    public final Report generate(ReportRequest request) {
        List<ReportRow> data = fetchData(request);
        List<ReportRow> processed = processData(data);
        return formatReport(processed);
    }

    protected abstract List<ReportRow> fetchData(ReportRequest request);
    protected abstract List<ReportRow> processData(List<ReportRow> data);
    protected abstract Report formatReport(List<ReportRow> data);
}
```

## Java-Specific Best Practices

### Use Records for Pure Data Carriers (Java 16+)

```java
public record UserDto(Long id, String name, String email) {}
```

### Use Sealed Classes for Controlled Hierarchies (Java 17+)

```java
public sealed interface PaymentResult permits SuccessResult, FailureResult, PendingResult {}
public record SuccessResult(String transactionId) implements PaymentResult {}
public record FailureResult(String errorCode, String message) implements PaymentResult {}
public record PendingResult(String referenceId) implements PaymentResult {}
```

### Use Switch Expressions for Pattern Matching (Java 14+)

```java
String describe(PaymentResult result) {
    return switch (result) {
        case SuccessResult s -> "Paid: " + s.transactionId();
        case FailureResult f -> "Failed: " + f.message();
        case PendingResult p -> "Pending: " + p.referenceId();
    };
}
```

### Prefer Streams for Collection Processing

```java
// Wrong
List<String> names = new ArrayList<>();
for (User user : users) {
    if (user.isActive()) {
        names.add(user.getName().toUpperCase());
    }
}

// Right
List<String> names = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .map(String::toUpperCase)
    .toList();
```

## Anti-Patterns — Never Use These

| Anti-Pattern | Problem | Solution |
|---|---|---|
| God Class | One class does everything | Split by SRP |
| Service Locator | Hidden dependencies, untestable | Use dependency injection |
| Anemic Domain Model | Business logic in services, dumb entities | Move logic into domain objects |
| Magic Numbers/Strings | Unreadable and error-prone | Use named constants or enums |
| Singleton via static | Hidden global state, untestable | Use Spring-managed beans |
| Catch-all Exception handler | Hides bugs | Catch specific exception types |
| Premature Optimisation | Complexity without proven need | Profile first, optimise second |
| Copy-Paste Code | Maintenance nightmare | Extract to shared method or class |
