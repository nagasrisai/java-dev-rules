---
applyTo: "**/*.java"
---

# Java Architect Rules

## Role and Responsibilities

As a Java architect, your responsibilities extend beyond writing code. You own:

- The overall system architecture and its evolution.
- Technology selection and dependency governance.
- Cross-cutting concerns: security, performance, observability, scalability.
- Architecture Decision Records (ADRs).
- Code review standards and enforcement.
- Enabling the team through standards, patterns, and mentoring.

## Architecture Principles

### Design for Change

- Keep modules loosely coupled and highly cohesive.
- Hide implementation details behind well-defined interfaces.
- Favour a hexagonal (ports and adapters) architecture to isolate the domain from infrastructure concerns.
- Use anti-corruption layers when integrating with external systems.

### Package Structure for Larger Services

```
com.example.service/
  domain/
    model/          # Pure domain entities (no framework annotations)
    service/        # Domain services (business logic)
    port/
      in/           # Inbound ports (use case interfaces)
      out/          # Outbound ports (repository/external service interfaces)
  application/
    usecase/        # Use case implementations (orchestrate domain services)
    dto/            # Application-level DTOs
  adapter/
    in/
      rest/         # REST controllers (Spring MVC)
      messaging/    # Message consumers (Kafka, SQS, etc.)
    out/
      persistence/  # JPA repositories and entity mappers
      external/     # External service clients
  config/           # Spring configuration
```

### API Versioning Strategy

- Version all public APIs from day one: `/api/v1/`, `/api/v2/`.
- Never remove a version without a deprecation period (minimum 3 months).
- Communicate deprecations in response headers: `Deprecation: true`, `Sunset: <date>`.
- Internal service-to-service APIs follow the same versioning rules.

### Microservice Principles (if applicable)

- Each service owns its own data — no shared database schemas between services.
- Communicate via well-defined API contracts (REST + OpenAPI or event schema registry).
- Design for failure — every service call may fail; implement retries, timeouts, and circuit breakers (e.g., Resilience4j).
- Services must be independently deployable.
- Keep services appropriately sized — each should map to a bounded context.

## Technology Governance

### Adding a New Dependency

Before introducing any new library or framework:

1. Confirm no existing library in the codebase already covers the need.
2. Check the library's licence compatibility (prefer Apache 2.0, MIT, or similar permissive licences).
3. Review the library's security vulnerability history (use OWASP Dependency Check or Snyk).
4. Confirm the library is actively maintained (check last release date and open issues).
5. Get architect approval for new dependencies in core or shared modules.
6. Document the decision in an ADR.

### Approved Technology Stack

Document your team's approved stack here and keep it updated:

| Category | Approved Library/Tool | Notes |
|---|---|---|
| Framework | Spring Boot 3.x | Production default |
| ORM | Spring Data JPA + Hibernate | Prefer JPQL; native SQL only when needed |
| Validation | Jakarta Validation (Bean Validation) | |
| Mapping | MapStruct | Preferred over ModelMapper |
| Resilience | Resilience4j | Circuit breaker, retry, rate limiter |
| Messaging | Spring Kafka / AWS SQS | Team decision per project |
| Caching | Spring Cache + Caffeine (local) / Redis (distributed) | |
| Testing | JUnit 5, Mockito, AssertJ, TestContainers | |
| Coverage | JaCoCo | Minimum 100% line coverage |
| Static Analysis | SonarQube | Zero blocker/critical issues policy |
| API Documentation | SpringDoc OpenAPI (Swagger UI) | |
| Observability | Micrometer + Prometheus + Grafana | |
| Logging | SLF4J + Logback | Structured JSON in production |

## Cross-Cutting Concerns

### Observability Standards

Every service must implement:

1. **Health checks** — Spring Actuator `/actuator/health` (include readiness + liveness probes).
2. **Metrics** — Micrometer counters and timers on all business-critical paths.
3. **Distributed tracing** — Micrometer Tracing (Brave/OpenTelemetry) with trace/span IDs in all logs.
4. **Structured logging** — JSON format in production; include: service name, version, environment, trace ID, span ID, user ID (if available), Jira ticket for any temporary debug logs.

### Performance Standards

- All public API endpoints must respond in under 500ms at the 95th percentile under expected load.
- List endpoints must be paginated — never return an unbounded collection.
- Database queries must be reviewed with `EXPLAIN ANALYZE` for any query operating on tables with >10,000 rows.
- Identify and resolve N+1 query problems using `@EntityGraph` or `JOIN FETCH`.
- Async processing (Spring `@Async`, Kafka consumers) must be used for operations >2 seconds.

### Security Architecture

- All services must be protected by the organisation's authentication gateway (OAuth2/JWT).
- Implement defence-in-depth: gateway authentication + service-level authorisation checks.
- Secrets must be managed by a secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.) — never in environment variables embedded in Docker images.
- Apply the principle of least privilege to all service accounts and database users.
- Run OWASP Dependency Check as part of every build pipeline.
- Conduct threat modelling (STRIDE) for any feature involving sensitive data.

## Architecture Review Process

### When an Architecture Review Is Required

- New service or module creation.
- Adoption of a new technology or framework.
- API contract changes that affect more than one consuming service.
- Schema changes affecting production data at scale.
- Changes to authentication or authorisation mechanisms.
- Major performance optimisations (indexing strategy changes, caching layers).

### Architecture Review Output

Produce a brief review document covering:

1. **Problem statement** — what needs to be solved and why.
2. **Proposed solution** — technical design with diagrams (sequence diagrams, component diagrams).
3. **Alternatives considered** — at least 2 alternatives and why they were rejected.
4. **Impact analysis** — what existing services, APIs, or data are affected.
5. **Risk assessment** — technical, operational, and security risks.
6. **Migration/rollout plan** — how to deploy without downtime.
7. **Decision** — the approved approach and who approved it.
8. **ADR reference** — link to the ADR created for this decision.

## Architect Code Review Focus Areas

When reviewing PRs as an architect, focus on:

- Architecture alignment: does the change fit the agreed architecture?
- Abstraction level: are the right things abstracted behind interfaces?
- Dependency direction: do dependencies point in the correct direction?
- Scalability: will this design hold at 10x current load?
- Resilience: what happens when each external dependency fails?
- Security: are all trust boundaries correctly enforced?
- Observability: will we be able to debug this in production?
- Test strategy: are the tests testing the right things at the right level?
