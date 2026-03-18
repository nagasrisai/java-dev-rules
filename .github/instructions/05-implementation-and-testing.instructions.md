---
applyTo: "**/*.java"
---

# Implementation, Testing, and SonarQube Rules

## Implementation Standards

### Code Quality Non-Negotiables

- Every method must do one thing only (Single Responsibility).
- Cyclomatic complexity per method must be 10 or lower.
- No method should exceed 40 lines of code.
- No class should exceed 300 lines — split into focused classes.
- No duplicate code blocks — extract to shared utility or base class.
- Use `@lombok` (Lombok) annotations to reduce boilerplate where the team uses it.
- Always use `try-with-resources` for streams, connections, and other `Closeable` objects.

### Immutability and Thread Safety

- Prefer immutable objects (`final` fields, no setters).
- Document thread safety guarantees with `@ThreadSafe` or `@NotThreadSafe`.
- Avoid shared mutable state; prefer stateless Spring `@Service` beans.
- Use `ConcurrentHashMap`, `AtomicInteger`, etc. for shared state — never raw `HashMap` across threads.

### Error Handling

- Throw specific typed exceptions, not generic `Exception` or `RuntimeException`.
- Catch only exceptions you can meaningfully handle.
- Always log the exception with enough context to diagnose the failure.
- Never use exceptions for control flow.

## Testing Standards — 100% Line Coverage Required

### Unit Testing

Framework: JUnit 5 + Mockito

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("Should return user when found by ID")
    void shouldReturnUserWhenFoundById() {
        // Arrange
        Long userId = 1L;
        User expectedUser = User.builder().id(userId).name("Alice").build();
        when(userRepository.findById(userId)).thenReturn(Optional.of(expectedUser));

        // Act
        User result = userService.findById(userId);

        // Assert
        assertThat(result).isEqualTo(expectedUser);
        verify(userRepository).findById(userId);
    }

    @Test
    @DisplayName("Should throw ResourceNotFoundException when user not found")
    void shouldThrowWhenUserNotFound() {
        // Arrange
        Long userId = 999L;
        when(userRepository.findById(userId)).thenReturn(Optional.empty());

        // Act + Assert
        assertThatThrownBy(() -> userService.findById(userId))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("User not found: 999");
    }
}
```

### Rules for Unit Tests

- Test EVERY branch: happy path, sad path, edge cases, boundary values.
- Use `@DisplayName` with a human-readable description for every test.
- Use `@ParameterizedTest` and `@MethodSource` / `@CsvSource` for multiple input variations.
- Mock only direct dependencies — do not mock the class under test.
- Assert behaviour, not implementation (verify interactions only when necessary).
- Never use `Thread.sleep()` in tests — use `Awaitility` for async assertions.

### Integration Testing

Framework: `@SpringBootTest` + `MockMvc` (or `WebTestClient` for reactive)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class UserControllerIT {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("GET /api/v1/users/{id} should return 200 with user data")
    void shouldReturn200WithUserData() throws Exception {
        mockMvc.perform(get("/api/v1/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.id").value(1))
            .andExpect(jsonPath("$.data.name").value("Alice"));
    }

    @Test
    @DisplayName("GET /api/v1/users/{id} should return 404 when user not found")
    void shouldReturn404WhenUserNotFound() throws Exception {
        mockMvc.perform(get("/api/v1/users/999")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotFound());
    }
}
```

### Test Coverage Verification

Run coverage before every pull request:

```bash
# Maven (with JaCoCo)
mvn clean verify

# Gradle
./gradlew test jacocoTestReport
```

Configure JaCoCo to fail the build if coverage drops below 100% on new code:

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <rules>
            <rule>
                <element>BUNDLE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>1.0</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</plugin>
```

### Sample Input / Scenario Testing

Before closing a ticket, collect sample test scenarios:

1. Ask the business/QA: "What are the key real-world inputs I should test with?"
2. For every provided scenario, write a test case (unit or integration) that covers it.
3. Document scenarios in the PR description under a **Test Scenarios** section:

```markdown
## Test Scenarios Covered

| # | Scenario | Input | Expected Output | Test Class |
|---|---|---|---|---|
| 1 | Valid user login | email: alice@test.com, password: correct | 200 + JWT token | UserAuthIT |
| 2 | Wrong password | email: alice@test.com, password: wrong | 401 Unauthorized | UserAuthIT |
| 3 | Non-existent user | email: ghost@test.com | 404 Not Found | UserAuthIT |
| 4 | Empty email | email: "", password: any | 400 Bad Request | UserAuthControllerTest |
```

## SonarQube Rules

### Zero Tolerance for New Issues

New code must not introduce any SonarQube issues at **Critical** or **Blocker** severity.
New code must not reduce overall code coverage below the project's configured threshold.

### Categories and Actions

| SonarQube Category | Rule |
|---|---|
| Bugs | Fix immediately — no exceptions |
| Vulnerabilities | Fix immediately — treat as a security incident |
| Code Smells (Blocker / Critical) | Fix before merging |
| Code Smells (Major) | Fix before merging where possible; log a Jira task for the rest |
| Code Smells (Minor / Info) | Fix opportunistically |
| Security Hotspots | Review and mark as Safe or create a fix ticket |

### Common Java SonarQube Issues to Pre-empt

- Use `isEmpty()` instead of `.size() == 0`
- Use `equals()` for String comparison (never `==`)
- Close resources in `finally` blocks or use `try-with-resources`
- Do not catch `Exception` or `Throwable` generically
- Avoid `System.exit()` in application code
- Do not return `null` from public methods — use `Optional<T>`
- Avoid `instanceof` chains — use polymorphism or the visitor pattern
- Do not use `@SuppressWarnings("unchecked")` without justification

### Running SonarQube Locally

```bash
mvn sonar:sonar \
  -Dsonar.projectKey=your-project-key \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=your-token
```

Or with Gradle:

```bash
./gradlew sonar \
  -Dsonar.projectKey=your-project-key \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=your-token
```

## Final Pre-Merge Checklist

- [ ] All unit tests pass: `mvn test`
- [ ] All integration tests pass
- [ ] 100% line coverage on all new/changed classes (JaCoCo report verified)
- [ ] SonarQube — zero new Critical or Blocker issues
- [ ] SonarQube — all Security Hotspots reviewed
- [ ] Sample scenarios provided by business/QA all covered by automated tests
- [ ] Test scenarios table documented in PR description
- [ ] API documentation updated
