---
applyTo: "**/*.java,**/*.yml,**/*.yaml,**/Dockerfile,**/*.sh"
---

# CI/CD and DevOps Rules for Java Projects

## CI Pipeline — Required Stages

Every Java project must have a CI pipeline that runs on every push to a PR branch and on every merge to `main` / `develop`.

### Required Pipeline Stages (in order)

```
1. Checkout
2. Set up JDK (use the project's pinned Java version — e.g. Java 21 LTS)
3. Cache Maven/Gradle dependencies
4. Compile
5. Unit tests
6. Integration tests
7. Code coverage check (JaCoCo — fail if below threshold)
8. Static analysis (SonarQube / SonarCloud)
9. Dependency vulnerability scan (OWASP Dependency Check or Snyk)
10. Code style check (Checkstyle / Spotless)
11. Build artifact (JAR / WAR)
12. Docker image build (if containerised)
13. Push to registry (on main/develop only)
```

---

## GitHub Actions — Standard Java Pipeline

```yaml
# .github/workflows/ci.yml
name: Java CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout source
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarQube needs full history

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'  # or 'gradle'

      - name: Compile
        run: mvn --batch-mode compile

      - name: Run unit tests
        run: mvn --batch-mode test

      - name: Run integration tests
        run: mvn --batch-mode verify -P integration-tests

      - name: Check code coverage
        run: mvn --batch-mode jacoco:check
        # Fails if line coverage drops below 100% on new code

      - name: SonarQube analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: mvn --batch-mode sonar:sonar

      - name: OWASP Dependency Check
        run: mvn --batch-mode dependency-check:check
        continue-on-error: false  # Fail on HIGH/CRITICAL CVEs

      - name: Check code style
        run: mvn --batch-mode checkstyle:check

      - name: Build JAR
        run: mvn --batch-mode package -DskipTests
        # Tests already ran above; -DskipTests here saves time

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: target/surefire-reports/

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/
```

---

## Docker Standards

### Dockerfile Template (Spring Boot)

```dockerfile
# Stage 1 — Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

COPY pom.xml .
COPY src ./src

# Download deps first (cache layer)
RUN --mount=type=cache,target=/root/.m2 \
    mvn --batch-mode dependency:go-offline

RUN --mount=type=cache,target=/root/.m2 \
    mvn --batch-mode package -DskipTests

# Stage 2 — Runtime (minimal image)
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health | grep -q '"status":"UP"' || exit 1

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Rules

- Always use multi-stage builds — the final image must NOT contain the JDK, only the JRE.
- Always use an Alpine-based image for the runtime stage to minimise image size.
- Run the application as a non-root user — never run as `root`.
- Pin base image versions explicitly — never use `latest`.
- Include a `HEALTHCHECK` instruction.
- Do not `COPY` the entire project into the build stage — copy `pom.xml` and `src/` separately to maximise layer caching.
- Never put secrets in the `Dockerfile` or Docker build args — inject at runtime via environment variables.

---

## Environment Configuration

### Rules

- Never embed environment-specific config in the Docker image.
- All runtime configuration must come from environment variables or a secrets manager.
- Use Spring profiles activated by `SPRING_PROFILES_ACTIVE` environment variable.
- Provide a `.env.example` file in the repo documenting all required environment variables — never a real `.env` with actual values.

### Required Environment Variables Documentation (`.env.example`)

```
# Application
SPRING_PROFILES_ACTIVE=development
SERVER_PORT=8080

# Database
DATABASE_URL=jdbc:postgresql://localhost:5432/mydb
DATABASE_USERNAME=myuser
DATABASE_PASSWORD=changeme

# Security
JWT_SECRET=changeme-use-at-least-256-bit-secret
JWT_EXPIRATION_MS=3600000

# External APIs
PAYMENT_GATEWAY_URL=https://api.payment-provider.com
PAYMENT_GATEWAY_API_KEY=changeme

# Observability
MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,prometheus
```

---

## Kubernetes Deployment Standards (if applicable)

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: order-service
          image: my-registry/order-service:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: database-url
```

### Kubernetes Rules

- Always define `resources.requests` and `resources.limits` — never leave them undefined.
- Always configure both `livenessProbe` and `readinessProbe` using Spring Actuator endpoints.
- Never use `imagePullPolicy: Always` in production — use specific image tags.
- Store secrets in Kubernetes `Secret` objects — never in `ConfigMap` or environment variables embedded in manifests.
- Use `runAsNonRoot: true` in the pod security context.

---

## Release Management

- All releases are tagged in Git: `v{major}.{minor}.{patch}` (semantic versioning).
- `MAJOR` — breaking API or schema changes.
- `MINOR` — new backward-compatible features.
- `PATCH` — backward-compatible bug fixes.
- Every release tag triggers a release pipeline that builds, tests, and publishes the Docker image.
- Production deployments require at least 2 pipeline approvals.
- Maintain a `CHANGELOG.md` updated for every release with a reference to resolved Jira tickets.

---

## Secrets Management

- Never store secrets in Git — use a secrets manager (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, or Azure Key Vault).
- Rotate secrets on a defined schedule (at minimum: annually, or immediately on suspected compromise).
- Use short-lived credentials where possible (IRSA, Workload Identity, etc.).
- Add a pre-commit hook or CI step using `git-secrets` or `truffleHog` to detect accidentally committed secrets.
