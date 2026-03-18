---
applyTo: "**/*.java"
---

# Java New Development Rules

## Before Writing Any Code

- Understand the full requirement before starting. If the requirement is unclear, ask clarifying questions.
- Identify all impacted layers: controller, service, repository, model, config, tests.
- Check for existing utilities, base classes, or shared libraries that can be reused.
- Confirm the target Java version (Java 11+ preferred, Java 17+ LTS recommended).
- Check `pom.xml` or `build.gradle` for existing dependencies before adding new ones.

## Project Structure

Follow standard layered architecture:

```
src/
  main/
    java/
      com.example.project/
        controller/      # REST controllers (@RestController)
        service/         # Business logic (@Service)
        repository/      # Data access (@Repository)
        model/           # Entities / domain objects
        dto/             # Data Transfer Objects
        config/          # Spring configuration classes
        exception/       # Custom exceptions and handlers
        util/            # Utility/helper classes
        mapper/          # Object mappers (MapStruct preferred)
  test/
    java/
      com.example.project/
        (mirrors main structure)
```

## Coding Standards

- Follow Java naming conventions strictly:
  - Classes: `PascalCase`
  - Methods and variables: `camelCase`
  - Constants: `UPPER_SNAKE_CASE`
  - Packages: `lowercase`
- Every public class, method, and field must have Javadoc.
- Keep methods short and focused — ideally under 20 lines, never exceed 40.
- Apply SOLID principles at all times:
  - **S** — Single Responsibility
  - **O** — Open/Closed
  - **L** — Liskov Substitution
  - **I** — Interface Segregation
  - **D** — Dependency Inversion
- Prefer composition over inheritance.
- Use `final` for fields that should not be reassigned.
- Never use raw types with generics (`List` instead of `List<String>` is forbidden).
- Use `Optional<T>` to represent nullable return values from service/repository methods.

## REST API Standards

- Use `@RestController` + `@RequestMapping` with versioned paths (e.g., `/api/v1/...`).
- HTTP method semantics:
  - `GET` — read only, idempotent
  - `POST` — create resource
  - `PUT` — full update, idempotent
  - `PATCH` — partial update
  - `DELETE` — remove resource
- Return proper HTTP status codes:
  - `200 OK` — success
  - `201 Created` — resource created
  - `204 No Content` — successful delete
  - `400 Bad Request` — validation failure
  - `401 Unauthorized` — authentication required
  - `403 Forbidden` — insufficient permissions
  - `404 Not Found` — resource not found
  - `500 Internal Server Error` — unexpected failure
- Always use a standard response wrapper (e.g., `ApiResponse<T>`).
- Validate all incoming request DTOs using `javax.validation` / `jakarta.validation` annotations.

## Data Access

- Use Spring Data JPA repositories.
- Never use raw SQL unless performance demands it — in that case, use `@Query` with named parameters.
- Never expose JPA entities directly via REST — always map to DTOs.
- Prefer lazy loading; understand N+1 and resolve it with `JOIN FETCH` or `@EntityGraph`.

## Exception Handling

- Define a global exception handler using `@RestControllerAdvice`.
- Create typed custom exceptions (e.g., `ResourceNotFoundException`, `ValidationException`).
- Never swallow exceptions silently — always log or rethrow.
- Never expose internal stack traces to the API consumer.

## Logging

- Use SLF4J with Logback (or Log4j2) — never use `System.out.println`.
- Use the appropriate log level:
  - `DEBUG` — detailed development info
  - `INFO` — significant business events
  - `WARN` — recoverable unexpected situations
  - `ERROR` — failures requiring immediate attention
- Always include contextual data (e.g., entity ID, username) in log messages.
- Never log sensitive data (passwords, tokens, PII).

## Security

- Never hard-code credentials, secrets, or API keys in source code.
- Use environment variables or Spring's `@ConfigurationProperties`.
- Validate and sanitize all user input.
- Protect endpoints with Spring Security roles/authorities.
- Apply CSRF protection for state-changing requests where applicable.

## Testing Requirements

- Write unit tests for every service method — minimum **100% line coverage**.
- Use JUnit 5 (`@ExtendWith(MockitoExtension.class)`) and Mockito.
- Write integration tests for all controller endpoints using `@SpringBootTest` + `MockMvc` or `WebTestClient`.
- Test file naming: `<ClassName>Test.java` for unit, `<ClassName>IT.java` for integration.
- Every test must follow the AAA pattern:
  - **Arrange** — set up test data and mocks
  - **Act** — invoke the method under test
  - **Assert** — verify the outcome

## Dependency Management

- Pin all dependency versions explicitly in `pom.xml` or `build.gradle`.
- Run `mvn dependency:analyze` or `gradle dependencies` to check for unused/missing deps.
- Avoid snapshot versions in production code.

## Code Review Checklist Before Committing

- [ ] All new methods have Javadoc
- [ ] No commented-out code committed
- [ ] No TODOs left unresolved without a Jira ticket reference
- [ ] Tests written and passing
- [ ] 100% line coverage on new code
- [ ] No SonarQube critical or blocker issues introduced
- [ ] No hardcoded configuration values
- [ ] API contracts are documented (Swagger/OpenAPI)
