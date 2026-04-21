---
applyTo: "**/*.java,**/pom.xml,**/build.gradle,**/build.gradle.kts,**/*.properties,**/*.yml,**/*.yaml"
---

# Java and Spring Boot Upgrade Rule (Self-Contained — No Web Search Needed)

## When This Rule Activates

The developer says any of:
- "Upgrade Java from X to Y"
- "Upgrade Spring Boot from X to Y"
- "Migrate to Java 17 / 21"
- "Migrate to Spring Boot 3.x"
- "Update our project to use the latest Spring Boot"
- "Move from javax to jakarta"
- "Help me upgrade my Spring app"
- Mentions any version transition like "2.7 to 3.2", "Java 11 to 21"

**You must ask the developer for the source and target versions if they did not provide both.**

---

## Step 1 — Gather Inputs

Before proposing any change, confirm:

```
1. Current Java version:          (8 / 11 / 17 / 21)
2. Target Java version:           (11 / 17 / 21)
3. Current Spring Boot version:   (e.g., 2.5.4, 2.7.18, 3.0.6, 3.2.0)
4. Target Spring Boot version:    (e.g., 3.0.x, 3.2.x, 3.3.x, 3.4.x)
5. Build tool:                    (Maven / Gradle)
6. Database (if Spring Data JPA):  (Hibernate version locked? Custom dialect?)
7. Spring Cloud (if used):        (current and target)
8. Spring Security (if used):     (yes/no — major changes in 6.x)
9. Major libraries used:          (Lombok, MapStruct, Resilience4j, etc.)
10. Test frameworks:              (JUnit 4 or 5? Mockito version?)
```

If any of these are missing from the developer's request, ask in ONE consolidated message — do not ask one at a time.

---

## Step 2 — Run Pre-Upgrade Audit

Before making any change, instruct the developer to run these commands and share the output:

```bash
# Maven
mvn -version
mvn dependency:tree > dependency-tree.txt
mvn dependency:analyze

# Gradle
./gradlew --version
./gradlew dependencies > dependency-tree.txt
./gradlew dependencyInsight --dependency <name>
```

This tells you exactly what versions are in use and what will break.

---

## JAVA VERSION UPGRADES — Embedded Knowledge

### Java 8 → Java 11 — Breaking Changes and Migration Steps

#### Removed / Replaced APIs
| What was removed | Replacement |
|---|---|
| Nashorn JavaScript engine | Use external engine (GraalJS) or remove |
| Java EE modules (JAXB, JAX-WS) | Add as explicit dependencies |
| CORBA modules | Use external CORBA implementation |
| `sun.misc.*` internal APIs | Use public APIs (`MethodHandles`, `VarHandle`) |
| `Thread.destroy()`, `Thread.stop(Throwable)` | Use interrupt mechanism |

#### Build File Changes (Maven)
```xml
<properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
    <maven.compiler.release>11</maven.compiler.release>
</properties>
```

#### Build File Changes (Gradle)
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(11)
    }
}
```

#### Dependencies to Add (if previously used Java EE)
```xml
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
    <version>2.3.1</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.9</version>
</dependency>
```

#### Code Changes
```java
// OLD (Java 8) — verbose HTTP call with HttpURLConnection
URL url = new URL("https://api.example.com/data");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();

// NEW (Java 11) — built-in HttpClient
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .build();
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
```

```java
// NEW (Java 11) — local variable type inference with var
var users = userRepository.findAll();        // List<User>
var counts = new HashMap<String, Integer>(); // HashMap<String, Integer>
```

---

### Java 11 → Java 17 (LTS) — Breaking Changes and Migration Steps

#### Removed / Replaced APIs
| What was removed | Replacement |
|---|---|
| RMI Activation | None — refactor to non-RMI approach |
| `Pack200` tool | Use jlink / jpackage |
| Strong encapsulation of JDK internals | Use `--add-opens` only as a temporary fix |
| Applet API (deprecated) | Migrate to web alternative |
| `SecurityManager` (deprecated) | Plan to remove SecurityManager usage |

#### New Features You Should Use
- **Records** (Java 14+) — for DTOs and value objects
- **Sealed classes** (Java 17) — for controlled type hierarchies
- **Pattern matching for instanceof** (Java 16+) — cleaner type checks
- **Text blocks** (Java 15+) — multi-line strings
- **Switch expressions** (Java 14+) — concise switch with return value

#### Build File Changes (Maven)
```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <maven.compiler.release>17</maven.compiler.release>
</properties>
```

#### Build File Changes (Gradle)
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

#### Compiler Plugin Versions (must be recent enough to support Java 17)
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>  <!-- Or newer -->
</plugin>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>   <!-- Required for proper Java 17 module handling -->
</plugin>
```

#### Code Modernisation Examples
```java
// OLD — verbose POJO
public class UserDto {
    private final Long id;
    private final String name;
    public UserDto(Long id, String name) { this.id = id; this.name = name; }
    public Long getId() { return id; }
    public String getName() { return name; }
    @Override public boolean equals(Object o) { /* ... */ }
    @Override public int hashCode() { /* ... */ }
}

// NEW — Record (Java 17)
public record UserDto(Long id, String name) {}
```

```java
// OLD — instanceof with cast
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.toUpperCase());
}

// NEW — pattern matching for instanceof
if (obj instanceof String s) {
    System.out.println(s.toUpperCase());
}
```

```java
// OLD — switch statement
String result;
switch (status) {
    case PENDING: result = "Wait"; break;
    case APPROVED: result = "Go"; break;
    default: result = "Unknown";
}

// NEW — switch expression
String result = switch (status) {
    case PENDING -> "Wait";
    case APPROVED -> "Go";
    default -> "Unknown";
};
```

---

### Java 17 → Java 21 (LTS) — Breaking Changes and Migration Steps

#### New Features You Should Use
- **Virtual threads** (`Thread.ofVirtual()`) — massive concurrency without thread pool tuning
- **Pattern matching for switch** (final in 21) — exhaustive type switches
- **Sequenced collections** — `getFirst()`, `getLast()`, `reversed()` on ordered collections
- **Record patterns** — destructuring records in switch and instanceof

#### Removed / Deprecated
| What | Status |
|---|---|
| `SecurityManager` | Deprecated for removal — plan migration |
| `finalize()` method | Deprecated for removal — use cleaners or try-with-resources |
| Thread suspension/resumption methods | Already removed |

#### Build File Changes (Maven)
```xml
<properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <maven.compiler.release>21</maven.compiler.release>
</properties>
```

#### Build File Changes (Gradle)
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

#### Code Examples — Java 21 Features
```java
// Virtual threads — ideal for I/O-bound workloads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> {
            // Each task on its own virtual thread — cheap to create
            return performHttpCall(i);
        })
    );
}

// Pattern matching for switch
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive integer: " + i;
        case Integer i -> "non-positive integer: " + i;
        case String s -> "string of length: " + s.length();
        case null -> "null value";
        default -> "unknown type";
    };
}

// Record patterns
record Point(int x, int y) {}
record Line(Point start, Point end) {}

double length(Line line) {
    return switch (line) {
        case Line(Point(var x1, var y1), Point(var x2, var y2)) ->
            Math.hypot(x2 - x1, y2 - y1);
    };
}

// Sequenced collections
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
String first = list.getFirst();    // "a"
String last = list.getLast();      // "c"
List<String> reversed = list.reversed();
```

---

## SPRING BOOT VERSION UPGRADES — Embedded Knowledge

### Spring Boot 2.x → 3.x — THE BIGGEST UPGRADE (Mandatory Reading)

This is the largest Spring Boot upgrade in years. Plan for significant work.

#### Mandatory Prerequisites
- **Java 17 minimum** — Spring Boot 3.x requires Java 17+
- **Hibernate 6.x** — significant query and entity changes
- **Jakarta EE 9+** — `javax.*` packages renamed to `jakarta.*` (universal)

#### Step-by-Step Upgrade Plan

**1. Upgrade Java first to 17 (or 21) — see Java sections above**

**2. Update parent / dependency in pom.xml**
```xml
<!-- OLD -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>  <!-- last 2.x release -->
</parent>

<!-- NEW -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>  <!-- target version -->
</parent>
```

**3. Update Gradle plugin**
```groovy
plugins {
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.5'
}
```

**4. javax → jakarta package rename (UNIVERSAL CHANGE)**

This is mechanical but touches every file. Use IDE-wide find/replace.

| OLD package (Spring Boot 2.x) | NEW package (Spring Boot 3.x) |
|---|---|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.ws.rs.*` | `jakarta.ws.rs.*` |
| `javax.transaction.Transactional` | `jakarta.transaction.Transactional` |
| `javax.annotation.PostConstruct` | `jakarta.annotation.PostConstruct` |
| `javax.annotation.PreDestroy` | `jakarta.annotation.PreDestroy` |
| `javax.mail.*` | `jakarta.mail.*` |
| `javax.jms.*` | `jakarta.jms.*` |
| `javax.xml.bind.*` | `jakarta.xml.bind.*` |

**Important exceptions — these stay javax:**
- `javax.sql.*` (JDBC) — stays javax
- `javax.crypto.*` — stays javax
- `javax.net.ssl.*` — stays javax
- `javax.security.auth.*` — stays javax

**Find & replace command (run from project root):**
```bash
# Java files
find src -name "*.java" -exec sed -i \
  -e 's/javax\.persistence/jakarta.persistence/g' \
  -e 's/javax\.validation/jakarta.validation/g' \
  -e 's/javax\.servlet/jakarta.servlet/g' \
  -e 's/javax\.transaction\.Transactional/jakarta.transaction.Transactional/g' \
  -e 's/javax\.annotation\.PostConstruct/jakarta.annotation.PostConstruct/g' \
  -e 's/javax\.annotation\.PreDestroy/jakarta.annotation.PreDestroy/g' \
  {} +
```

**5. Spring Security 6 changes (if used)**

The most disruptive change in 3.x. `WebSecurityConfigurerAdapter` is REMOVED.

```java
// OLD — Spring Security 5 (Spring Boot 2.x)
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .formLogin();
    }
}

// NEW — Spring Security 6 (Spring Boot 3.x) — lambda DSL, no adapter
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()  // antMatchers → requestMatchers
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    // If you had a custom UserDetailsService bean:
    @Bean
    public UserDetailsService userDetailsService() {
        return new CustomUserDetailsService();
    }

    // Replace AuthenticationManager exposure:
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

**6. Spring Data JPA — Hibernate 6 changes**

Hibernate 6 brings several breaking changes:

```java
// OLD — Hibernate 5 (Spring Boot 2.x)
@Type(type = "jsonb")  // hibernate-types library
private Map<String, Object> metadata;

// NEW — Hibernate 6 native JSON support (Spring Boot 3.x)
@JdbcTypeCode(SqlTypes.JSON)
private Map<String, Object> metadata;
```

Common Hibernate 6 issues:
- `Query.uniqueResult()` returns `Object` — cast or use typed queries
- `Session.load()` is deprecated — use `Session.byId()` or `Session.find()`
- Entity proxies enabled by default — explicit `OPEN_SESSION_IN_VIEW` may be needed
- Custom dialects must extend new dialect classes

**7. Configuration property changes**

```properties
# OLD (Spring Boot 2.x)
spring.datasource.initialization-mode=always
spring.jpa.hibernate.naming.implicit-strategy=...
server.servlet.context-path=/api

# NEW (Spring Boot 3.x)
spring.sql.init.mode=always                        # renamed
spring.jpa.hibernate.naming.implicit-strategy=...  # unchanged
server.servlet.context-path=/api                   # unchanged
spring.threads.virtual.enabled=true                # NEW in 3.2 — enable virtual threads
```

**8. Removed / changed properties (key ones)**

| OLD property | NEW property |
|---|---|
| `spring.datasource.initialization-mode` | `spring.sql.init.mode` |
| `spring.datasource.schema` | `spring.sql.init.schema-locations` |
| `spring.datasource.data` | `spring.sql.init.data-locations` |
| `management.metrics.export.*` | `management.<exporter>.metrics.export.*` |
| `spring.security.oauth2.client.provider.<id>.user-info-authentication-method` | (removed — use default) |
| `server.max-http-header-size` | `server.max-http-request-header-size` |
| `logging.file` | `logging.file.name` |
| `logging.path` | `logging.file.path` |

**9. Logback configuration**

If you have a `logback.xml` or `logback-spring.xml`, check for:
- `<springProperty>` still works
- Property file format unchanged
- Pattern layouts work the same

**10. Actuator changes**

```properties
# Spring Boot 3.x — /actuator endpoints
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.health.show-details=when-authorized
management.endpoint.health.probes.enabled=true   # for k8s readiness/liveness
```

**11. Test changes**

```java
// OLD (Spring Boot 2.x)
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.mock.mockito.MockBean;  // still works in 3.x

// NEW (Spring Boot 3.4+) — MockBean is deprecated, use:
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@SpringBootTest
class UserServiceTest {
    @MockitoBean   // replaces @MockBean from Spring Boot 3.4+
    UserRepository userRepository;
}
```

---

### Spring Boot 3.0 → 3.1 — Lighter Upgrade

Key additions:
- Docker Compose support (`spring-boot-docker-compose` starter)
- Testcontainers `@ServiceConnection` support
- SSL bundles for centralised SSL configuration
- Improved CRaC (Coordinated Restore at Checkpoint) support

Update: simple parent version bump.

---

### Spring Boot 3.1 → 3.2 — Important Upgrade (Java 21 Support)

Key changes:
- Full Java 21 support
- Virtual threads (enable with `spring.threads.virtual.enabled=true`)
- Spring Framework 6.1
- RestClient (modern alternative to RestTemplate)
- JdbcClient (modern alternative to JdbcTemplate)

```java
// NEW in 3.2 — RestClient (replaces RestTemplate going forward)
RestClient client = RestClient.create();
User user = client.get()
    .uri("https://api.example.com/users/{id}", userId)
    .retrieve()
    .body(User.class);

// NEW in 3.2 — JdbcClient
List<User> users = jdbcClient.sql("SELECT * FROM users WHERE active = ?")
    .param(true)
    .query(User.class)
    .list();
```

---

### Spring Boot 3.2 → 3.3 — Steady Improvements

Key changes:
- Java 22 support added
- Improved CDS (Class Data Sharing) support
- Class data sharing during build for faster startup
- Property `spring.application.version` for versioning

Update: simple parent version bump. Few breaking changes.

---

### Spring Boot 3.3 → 3.4 — Recent Upgrades

Key changes:
- `@MockBean` and `@SpyBean` deprecated → use `@MockitoBean` and `@MockitoSpyBean`
- Improved Docker Compose / Testcontainers integration
- Structured logging support (built-in JSON logging)

```properties
# NEW in 3.4 — built-in structured logging
logging.structured.format.console=ecs    # or 'logstash', 'gelf'
logging.structured.format.file=ecs
```

---

## DEPENDENCY UPGRADE MATRIX (When Upgrading Spring Boot 3.x)

Update these dependencies in lock-step with Spring Boot:

| Library | Spring Boot 2.7 | Spring Boot 3.0 | Spring Boot 3.2 | Spring Boot 3.4 |
|---|---|---|---|---|
| Java (min) | 8 | 17 | 17 | 17 (21 recommended) |
| Spring Framework | 5.3.x | 6.0.x | 6.1.x | 6.2.x |
| Spring Security | 5.7.x | 6.0.x | 6.2.x | 6.4.x |
| Spring Data | 2021.x | 2022.x | 2023.x | 2024.x |
| Hibernate ORM | 5.6.x | 6.1.x | 6.4.x | 6.6.x |
| Jakarta EE | 8 (javax) | 9+ (jakarta) | 10 (jakarta) | 10+ (jakarta) |
| JUnit Platform | 1.8.x | 1.9.x | 1.10.x | 1.11.x |
| Mockito | 4.x | 5.x | 5.x | 5.x |
| MapStruct | 1.5.x | 1.5.x+ | 1.6.x | 1.6.x |
| Lombok | 1.18.22+ | 1.18.30+ | 1.18.30+ | 1.18.34+ |

---

## SPRING CLOUD COMPATIBILITY

Spring Cloud versions are tied to Spring Boot:

| Spring Boot | Spring Cloud Release Train |
|---|---|
| 2.7.x | 2021.0.x (Jubilee) |
| 3.0.x | 2022.0.x (Kilburn) |
| 3.1.x | 2022.0.x (Kilburn) |
| 3.2.x | 2023.0.x (Leyton) |
| 3.3.x | 2023.0.x (Leyton) |
| 3.4.x | 2024.0.x (Moorgate) |

Update in pom.xml:
```xml
<properties>
    <spring-cloud.version>2024.0.0</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## STRUCTURED UPGRADE EXECUTION PLAN

When the developer asks to upgrade, ALWAYS follow this exact sequence and present it back to them as a numbered plan first:

```markdown
## Upgrade Plan: Spring Boot {FROM} → {TO}, Java {FROM} → {TO}

### Phase 1: Pre-Upgrade Audit (no code changes yet)
1. Run `mvn -version` and `mvn dependency:tree` — share output
2. List custom dependencies and their versions
3. Identify Spring Security usage (yes/no)
4. Identify Spring Cloud usage and version
5. Confirm test framework versions

### Phase 2: Java Upgrade (do this first)
1. Update `<maven.compiler.release>` or Gradle toolchain
2. Update Maven/Gradle plugin versions for Java compatibility
3. Build and run all tests — fix any compile errors
4. Commit: "chore: upgrade to Java {VERSION}"

### Phase 3: Spring Boot Parent Upgrade
1. Update `<parent>` version in pom.xml or Gradle plugin version
2. Update Spring Cloud version (if used)
3. Build (DO NOT run yet) — expect compile errors

### Phase 4: javax → jakarta Migration (if 2.x → 3.x)
1. Run global find/replace for `javax.*` → `jakarta.*` (with exceptions)
2. Update third-party dependencies that ship `javax.*` to their `jakarta.*` versions:
   - `javax.servlet:javax.servlet-api` → `jakarta.servlet:jakarta.servlet-api`
   - `javax.validation:validation-api` → `jakarta.validation:jakarta.validation-api`
   - `javax.persistence:javax.persistence-api` → `jakarta.persistence:jakarta.persistence-api`
3. Build — fix remaining import errors

### Phase 5: Spring Security Migration (if used and 2.x → 3.x)
1. Replace `WebSecurityConfigurerAdapter` with `SecurityFilterChain` bean
2. Replace `antMatchers` with `requestMatchers`
3. Convert chained config to lambda DSL
4. Update authentication manager exposure

### Phase 6: Hibernate 6 Migration (if 2.x → 3.x)
1. Replace `@Type(type="jsonb")` with `@JdbcTypeCode(SqlTypes.JSON)`
2. Update custom dialects to extend new dialect classes
3. Review and fix any HQL/JPQL queries that use changed syntax
4. Replace `Session.load()` with `Session.byId()` or `Session.find()`

### Phase 7: Configuration Property Updates
1. Update `application.properties` / `application.yml` for renamed properties
2. Add `spring.threads.virtual.enabled=true` if upgrading to 3.2+ on Java 21

### Phase 8: Test Migration
1. Update JUnit and Mockito versions
2. If on Spring Boot 3.4+, replace `@MockBean` with `@MockitoBean`
3. Run all tests — fix failures

### Phase 9: Verification
1. Build: `mvn clean package`
2. Run all unit tests
3. Run all integration tests
4. Start the application — verify clean startup
5. Run smoke tests against key endpoints
6. Run SonarQube scan — fix any new issues
7. Verify dependency tree has no version conflicts

### Phase 10: Commit Strategy
Commit each phase separately so any phase can be reverted independently:
- chore: upgrade Java to {VERSION}
- chore: upgrade Spring Boot parent to {VERSION}
- refactor: migrate javax to jakarta packages
- refactor: migrate Spring Security to 6.x lambda DSL
- chore: upgrade Hibernate to 6.x and migrate JSON types
- chore: update configuration properties for Spring Boot {VERSION}
- chore: update test framework dependencies
```

---

## COMMON UPGRADE PROBLEMS AND SOLUTIONS (No Web Search Needed)

### Problem: "Cannot find symbol javax.persistence.Entity"
**Cause:** Spring Boot 3.x renamed packages.
**Fix:** Replace `javax.persistence.*` with `jakarta.persistence.*` everywhere.

### Problem: "WebSecurityConfigurerAdapter is deprecated/removed"
**Cause:** Removed in Spring Security 6 (Spring Boot 3.x).
**Fix:** Use `SecurityFilterChain` bean — see Phase 5 above.

### Problem: "antMatchers is not a method"
**Cause:** Renamed in Spring Security 6.
**Fix:** Replace `antMatchers(...)` with `requestMatchers(...)`.

### Problem: "Cannot resolve symbol javax.validation.constraints.NotNull"
**Cause:** javax.validation moved to jakarta.validation.
**Fix:** Replace `javax.validation.*` with `jakarta.validation.*`.

### Problem: "ClassCastException in Hibernate query"
**Cause:** Hibernate 6 changed query result handling.
**Fix:** Use typed queries or explicit casts; check for HQL changes.

### Problem: "java.lang.NoClassDefFoundError: javax/servlet/Filter"
**Cause:** Library still on javax.servlet but Spring Boot 3.x is on jakarta.servlet.
**Fix:** Upgrade the library to a Jakarta-compatible version, or replace it.

### Problem: "Unknown property spring.datasource.initialization-mode"
**Cause:** Property renamed in Spring Boot 3.
**Fix:** Use `spring.sql.init.mode` instead.

### Problem: Tests fail with "MockBean is deprecated"
**Cause:** Spring Boot 3.4+ deprecated MockBean.
**Fix:** Replace with `@MockitoBean`.

### Problem: Application starts but immediately exits
**Cause:** Missing or wrong starter dependency.
**Fix:** Verify `spring-boot-starter-web` (or appropriate starter) is present and at correct version.

### Problem: "Failed to determine a suitable driver class"
**Cause:** Datasource auto-config changed or driver not on classpath.
**Fix:** Add explicit JDBC driver dependency and `spring.datasource.url` property.

### Problem: Lombok stops generating code after Java upgrade
**Cause:** Old Lombok version not compatible with new Java.
**Fix:** Upgrade Lombok to 1.18.30+ for Java 21, 1.18.34+ for newer.

### Problem: Maven Compiler "release version not supported"
**Cause:** Old maven-compiler-plugin version.
**Fix:** Upgrade `maven-compiler-plugin` to 3.13.0+.

### Problem: Mockito fails with InvalidUseOfMatchersException after Spring Boot 3 upgrade
**Cause:** Mockito 4 not compatible with Java 17/Spring Boot 3.
**Fix:** Upgrade Mockito to 5.x.

---

## RESPONSE STYLE FOR UPGRADE REQUESTS

When the developer requests an upgrade, your response MUST follow this structure:

```
## Upgrade Plan: [FROM versions] → [TO versions]

### What Will Change
[Brief summary of major changes — Java APIs, Spring features, dependencies]

### Risk Assessment
- Estimated effort: [hours/days]
- Risk level: [Low / Medium / High]
- Most likely problem areas: [list]

### Step-by-Step Plan
[Reference the 10 phases above — only include phases that apply]

### Files I Will Change
[List every file that needs modification]

### Code Changes (per file)
[Full corrected file content for each file, in order of phase]

### Verification Commands to Run
[Maven/Gradle commands the developer should run after each phase]

### Rollback Plan
[Per-phase rollback — git revert commands]
```

---

## NEVER DO THESE THINGS DURING AN UPGRADE

- Do NOT skip the Java upgrade step before the Spring Boot upgrade
- Do NOT change application logic during the upgrade — keep it mechanical
- Do NOT update dependencies one at a time without testing in between
- Do NOT do a 2.x → 3.x upgrade without first reaching the latest 2.7.x
- Do NOT remove tests because they break — fix them
- Do NOT ignore deprecation warnings — fix them in a follow-up commit
- Do NOT upgrade across multiple major Spring Boot versions in one go (e.g., 2.5 → 3.2 in one shot) — do it stepwise: 2.5 → 2.7 → 3.0 → 3.2
- Do NOT rewrite from scratch — upgrade in place
