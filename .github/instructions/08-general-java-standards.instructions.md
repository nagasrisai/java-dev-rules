---
applyTo: "**/*.java"
---

# General Java Standards (Applies to All Roles)

## Code Style and Formatting

- Use the project's code formatter — all code must pass formatting checks in CI.
  - Maven: Spotless or Checkstyle
  - Gradle: Spotless plugin
- Indent with 4 spaces — never tabs.
- Line length: max 120 characters.
- One blank line between methods; two blank lines between top-level declarations in a class.
- Import order: static imports first, then standard library, then third-party, then internal — no wildcard imports.

## Javadoc Standards

All public classes, interfaces, and methods must have Javadoc:

```java
/**
 * Retrieves a user by their unique identifier.
 *
 * @param userId the unique identifier of the user; must not be null
 * @return the user matching the given ID
 * @throws ResourceNotFoundException if no user exists with the given ID
 */
public User findById(Long userId) { ... }
```

- Private methods: Javadoc optional but encouraged for complex logic.
- DTOs and records: document the purpose of the class; individual field comments for non-obvious fields.

## Version Control Rules

### Branching Strategy (GitFlow or Trunk-Based — pick one per project)

GitFlow:
```
main        — production releases only
develop     — integration branch
feature/*   — new features
bugfix/*    — bug fixes
hotfix/*    — emergency production fixes
release/*   — release preparation
```

Trunk-Based:
```
main           — always releasable
feature/*      — short-lived (max 2 days), merged frequently
```

### Commit Rules

- Commits must be atomic: one logical change per commit.
- Commit messages must follow Conventional Commits:
  ```
  <type>(<scope>): <subject>

  [optional body]

  [optional footer: Jira ticket reference]
  ```
  Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`

- No commented-out code in commits.
- No debug logging committed (no `System.out.println`, no temporary `logger.debug` left in production paths).

### Pull Request Rules

- PR must target the correct base branch.
- PR title must reference the Jira ticket: `[PROJ-1234] Add user registration endpoint`.
- PR description must follow the team PR template.
- All CI checks must pass before review is requested.
- Minimum 1 approved review required before merge (2 for core/shared modules).
- Resolve all review comments before merging — do not merge with unresolved threads.
- Squash-merge feature branches; preserve merge commits for release branches.

## Continuous Integration

Every PR triggers the CI pipeline which must include:

| Stage | Tool | Pass Criteria |
|---|---|---|
| Compile | Maven / Gradle | Zero errors |
| Unit Tests | JUnit 5 | All tests green |
| Integration Tests | JUnit 5 + TestContainers | All tests green |
| Code Coverage | JaCoCo | 100% line coverage on new code |
| Static Analysis | SonarQube | Zero new Critical/Blocker issues |
| Dependency Check | OWASP Dependency Check | Zero high/critical CVEs in direct deps |
| Code Style | Checkstyle / Spotless | Zero violations |
| Build | Maven / Gradle | Successful artifact build |

CI must pass on every commit pushed to a PR branch. The main/develop branch is protected and can only be updated via merged PRs.

## Documentation Standards

### Code-Level Documentation

- Javadoc on all public API surface.
- Inline comments for non-obvious business logic — explain WHY, not WHAT.
- Reference Jira ticket numbers for business rules that came from a specific requirement.

### API Documentation

- All REST APIs documented via SpringDoc OpenAPI annotations.
- Swagger UI accessible at `/swagger-ui.html` in development.
- OpenAPI spec exported as `openapi.yaml` and committed to the repository.
- Each endpoint must document: description, all parameters, all response codes, and example request/response.

### Architecture Documentation

- `README.md` at the service root: purpose, quick start, environment variables, API overview.
- `docs/adr/` directory for Architecture Decision Records.
- `docs/runbooks/` for operational runbooks (deployment, incident response, rollback procedure).
- System diagram in `docs/architecture/` updated whenever the architecture changes.

## Security Rules (All Developers)

- Never commit secrets, API keys, passwords, or tokens. Use `.gitignore` and environment variables.
- Run `git-secrets` or a pre-commit hook to prevent accidental secret commits.
- Review OWASP Top 10 annually and check your code against it.
- Never trust user input — validate all inputs server-side even if validated client-side.
- Use prepared statements / parameterised queries — never string-concatenate SQL.
- Audit all third-party dependencies for known CVEs before adding them.

## Performance Rules (All Developers)

- Never load all records from a database without pagination.
- Do not call external services (REST, database) inside a loop — batch where possible.
- Use connection pooling for all database connections (HikariCP is the Spring Boot default).
- Set explicit timeouts on all HTTP client calls (read timeout, connect timeout).
- Profile before optimising — use Spring Boot Actuator metrics and a profiler (JProfiler, async-profiler).

## Environment and Configuration

- No environment-specific logic in code — use Spring profiles (`@Profile`) and `application-{profile}.yml`.
- All configuration properties must be documented with `@ConfigurationProperties` + validation annotations.
- Required properties must fail fast on startup if missing (use `@NotNull`, `@NotBlank` on config classes).
- Keep `application.yml` minimal in `src/main/resources`; use environment-specific overrides.
- Never use `application-prod.yml` committed to source control — production config comes from the environment or secrets manager.
