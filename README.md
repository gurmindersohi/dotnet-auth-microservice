# 🚀 Auth Service (ASP.NET Core, .NET 8) — Production-Grade Microservice

 A cloud-native authentication & RBAC microservice with clean architecture, provable reliability, and operational excellence. Ships with RSA/JWKS key rotation, refresh-token rotation & revocation, idempotency, outbox/inbox, OpenTelemetry, Grafana/Prometheus dashboards, and canary-ready Helm.

Default mode: Resource Server (JWT validation, token issuing for first-party services).
Optional profile: Lightweight IdP/OIDC via OpenIddict (publish discovery & JWKS).

### Why this exists

Most samples stop at “it runs in Docker.” This template is opinionated about security, failure semantics, and day-2 ops so teams can scale safely.

---

### 📂 Repository Structure

```bash
auth-service/
├── src/
│   ├── AuthService.Api/                      # HTTP boundary
│   │   ├── Controllers/ or Endpoints/        # Choose one (Minimal APIs or MVC)
│   │   ├── Filters/                          # ProblemDetails (RFC 7807)
│   │   ├── Middlewares/                      # CorrelationId, Idempotency, Logging
│   │   ├── SecurityHeaders/                  # HSTS/CSP/CORS baseline
│   │   ├── appsettings*.json
│   │   └── Program.cs                        # DI, composition, OpenTelemetry
│   │
│   ├── AuthService.Application/              # Use-cases (CQRS), ports, validators
│   │   ├── Abstractions/                     # IUserRepository, IJwtService, IEmailSender
│   │   ├── CommandsQueries/                  # MediatR handlers
│   │   ├── DTOs/
│   │   └── Validators/                       # FluentValidation
│   │
│   ├── AuthService.Domain/                   # Entities, ValueObjects, DomainEvents, Policies
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   └── Policies/                         # Password/lockout/uniqueness invariants
│   │
│   └── AuthService.Infrastructure/           # EF Core, Identity/OpenIddict, messaging
│       ├── Persistence/
│       │   ├── AuthDbContext.cs
│       │   ├── Configurations/
│       │   └── Migrations/
│       ├── Identity/                         # Optional: OpenIddict profile
│       ├── Messaging/                        # Outbox dispatcher, Kafka/RabbitMQ adapters
│       ├── Idempotency/                      # Store + provider
│       ├── Security/                         # RSA key mgmt, JWKS publisher
│       └── Services/                         # JwtService, EmailSender, PasswordHasher
│
├── tests/
│   ├── AuthService.Tests.Unit/               # xUnit + FluentAssertions
│   └── AuthService.Tests.Integration/        # Testcontainers (Postgres, Broker)
│
├── ops/
│   ├── helm/                                 # Helm chart (canary-ready)
│   ├── dashboards/                           # Grafana JSON
│   ├── alerts/                               # Prometheus alert rules
│   └── k6/                                   # Load test scripts
│
├── Dockerfile
├── docker-compose.yml
├── Directory.Packages.props                  # Centralized NuGet versions
├── Directory.Build.props                     # Analyzers, warnings-as-errors
└── README.md
```

---

### ✨ Key Capabilities

- Clean architecture: Domain/Application/Infrastructure/Api with strict boundaries.

- Security hardening:
    - RSA signing + JWKS with key rotation (kid header, dual-key overlap).
    - Refresh-token rotation & revocation (device binding, jti, IP, expiry).
    - Security headers: HSTS, CSP (report-only toggle), strict CORS defaults.
    - Secrets from Key Vault / Secrets Manager; no secrets in appsettings.

- Reliability patterns:
    - Outbox/Inbox for exactly-once effects over at-least-once delivery.
    - Idempotency middleware (Idempotency-Key) for POST/PUT/DELETE.
    - Polly policies for outbound I/O (retry/backoff/circuit-breaker/bulkhead).

- Observability:
    - OpenTelemetry traces/metrics/logs; W3C context propagation.
    - Prometheus scraping, Grafana dashboards + alert rules included.
    - Serilog structured logs with PII redaction & correlation enrichment.

- Data:
    - PostgreSQL, EF Core 8 with compiled queries, proper indexes & constraints.
      One schema per service; migrations gated and audited.

- Delivery:
    - Azure DevOps pipeline: SAST, format/analyzers, SBOM, container scan, gated DB migration
      Helm canary rollout.

- Testing:
    - Unit + Integration (Testcontainers), sample contract tests (Pact) ready.

---

### 🌐 API Surface (v1)

```bash
POST /api/v1/auth/register – create user

POST /api/v1/auth/login – issue access + refresh

POST /api/v1/auth/refresh – rotate refresh token

POST /api/v1/auth/logout – revoke current refresh token

POST /api/v1/auth/revoke – admin/device-wide revocation

GET /api/v1/users/me – current user

POST /api/v1/users/{id}/roles – assign roles (admin)

GET /health/live / GET /health/ready – probes

GET /.well-known/jwks.json – JWKS (when RSA enabled; always in this template)

GET /.well-known/openid-configuration – OIDC discovery (if OpenIddict profile enabled)
```

API versioning uses ASP.NET API Versioning with [ApiVersion] attributes.

---

### 🔐 Token & Key Management (built-in)

**Access tokens:** default 15m. Refresh tokens: default 7d, rotated on every use.
Storage: refresh tokens persisted with user_id, device_id, jti, ip, created_at, expires_at, revoked_at, replaced_by.

**Signing:** RSA keys resolved from secrets manager; JWKS published with kid.
Rotation plan: keep two active keys overlapping; signer picks the newest; validators accept both until cutoff.

Sample strongly-typed options
```bash
public sealed class JwtOptions
{
    [Required] public string Issuer { get; init; } = default!;
    [Required] public string Audience { get; init; } = default!;
    [Range(5,180)] public int AccessTokenMinutes { get; init; } = 15;
    [Range(1,30)]  public int RefreshTokenDays { get; init; } = 7;
    // Key material is resolved via IKeyMaterialProvider (RSA) – not from config.
}
```

---

### 🗃 Data Model (core tables)

- Users (Id PK, Email UNIQUE, PasswordHash, CreatedAt, LockoutUntil, ... )

- Roles (Id PK, Name UNIQUE) and UserRoles (UserId, RoleId, UNIQUE)

- RefreshTokens (Id PK, UserId, DeviceId, Jti, Ip, ExpiresAt, RevokedAt, ReplacedBy)

- OutboxMessages (Id PK, Type, Payload JSONB, OccurredAt, ProcessedAt NULL, Attempts, UniqueKey)

- InboxMessages (Id PK, Source, MessageId UNIQUE, ReceivedAt)

Indexes: IX_Users_Email, IX_RefreshTokens_UserId_DeviceId_Jti, FK indexes, outbox/inbox uniqueness.

---

### ♻️ Reliability Patterns

#### Outbox & Dispatcher
- Save domain event + outbox record in same transaction.

- Background dispatcher (hosted service) publishes with exponential backoff.

- Consumers dedupe with Inbox (MessageId).

#### Idempotency

- Middleware checks Idempotency-Key + user context.

- For qualifying routes, previous successful responses are replayed; in-flight requests coalesce.

---

### 🛡️ Security Baseline
- HSTS enabled (non-dev).
- CSP default denies; allowlists Swagger in dev; report-only flag for rollout.
- CORS deny by default; per-env allowlists.
- PII redaction: email/subject claims scrubbed via Serilog destructuring policy.
- Validation: FluentValidation integrated; all errors surfaced via ProblemDetails with machine-readable type URIs.

---

### 📊 Observability
- OpenTelemetry for traces/metrics/logs → OTLP exporter.
- Correlation: CorrelationId middleware maps to W3C traceparent; Serilog enrichers add CorrelationId, UserId, ClientId, DeviceId.
- Prometheus: /metrics endpoint.
- Dashboards (in ops/dashboards):
    - Golden signals (RPS, p95 latency, error rate, saturation)
    - DB pool exhaustion, migration duration
    - Outbox backlog, consumer lag
- Alerts (in ops/alerts):
    - 5xx error rate spike
    - Readiness probe failures
    - Outbox age > threshold
    - DB connections > 80%

---

### 🧰 Tech Stack
- Runtime: .NET 8, ASP.NET Core (Minimal APIs or Controllers—pick one)
- Data: PostgreSQL, EF Core 8 (compiled queries, NodaTime recommended)
- Auth: JWT Bearer, RSA/JWKS, optional OpenIddict (OIDC)
- Messaging: MassTransit + RabbitMQ or Confluent.Kafka
- Resilience: Polly
- Observability: OpenTelemetry, Prometheus, Grafana, Serilog
- Testing: xUnit, FluentAssertions, Testcontainers, optional Pact
- Delivery: Docker, Helm, Azure DevOps pipeline (canary rollout)

---

### ⚙️ Getting Started
1) Prerequisites
    - .NET 8 SDK
    - Docker & Docker Compose

2) Local (Docker Compose)
    ```bash
    docker compose up --build
    # API:       http://localhost:8080
    # Swagger:   http://localhost:8080/swagger
    # Metrics:   http://localhost:8080/metrics
    # Postgres:  localhost:5432
    ```

3) Local (without Docker)
    Provision Postgres, set connection string via env, then:
    ```bash
    dotnet tool restore
    dotnet ef database update \
    --project src/AuthService.Infrastructure \
    --startup-project src/AuthService.Api

    dotnet run --project src/AuthService.Api
    ```

---


### 🔧 Configuration (12-factor)

Never store secrets/keys in files. Use env + secret provider.

```bash
// src/AuthService.Api/appsettings.Development.json (dev-safe only)
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=authdb;Username=auth;Password=authpwd"
  },
  "Jwt": {
    "Issuer": "auth-service",
    "Audience": "internal-clients",
    "AccessTokenMinutes": 15,
    "RefreshTokenDays": 7
  },
  "Serilog": { "MinimumLevel": "Information" }
}
```

Env vars
```bash
ASPNETCORE_ENVIRONMENT=Development
ConnectionStrings__Default=...
Jwt__Issuer=...
Jwt__Audience=...
# RSA keys are resolved via IKeyMaterialProvider (Key Vault / Secrets Manager)
```
---

### 🗂️ Docker & Health

Dockerfile
```bash
# Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

# Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ./src ./src
COPY Directory.*.props ./
RUN dotnet restore src/AuthService.Api/AuthService.Api.csproj
RUN dotnet publish src/AuthService.Api/AuthService.Api.csproj -c Release -o /out \
    /p:PublishReadyToRun=true /p:PublishTrimmed=false

# Final
FROM base AS final
WORKDIR /app
COPY --from=build /out .
HEALTHCHECK --interval=10s --timeout=3s --retries=12 CMD curl -f http://localhost:8080/health/ready || exit 1
ENTRYPOINT ["dotnet", "AuthService.Api.dll"]
```

docker-compose.yml
```bash
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: auth
      POSTGRES_PASSWORD: authpwd
      POSTGRES_DB: authdb
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U auth -d authdb"]
      interval: 5s
      timeout: 3s
      retries: 10

  auth-service:
    build: { context: ., dockerfile: Dockerfile }
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Default: Host=db;Port=5432;Database=authdb;Username=auth;Password=authpwd
      Jwt__Issuer: auth-service
      Jwt__Audience: internal-clients
    ports: ["8080:8080"]
    depends_on:
      db: { condition: service_healthy }
```
---

### 🧪 Tests
```bash
dotnet test tests/AuthService.Tests.Unit
dotnet test tests/AuthService.Tests.Integration   # spins up Postgres via Testcontainers
```

### Integration tests validate:

- Token flow (login → refresh rotation → revoke)
- Outbox dispatcher and consumer dedupe
- Idempotency replay on retried POSTs

---

### 🚀 CI/CD (Azure DevOps pipeline skeleton)
```bash
# .azure/azure-pipelines.yml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  imageName: 'auth-service'
  containerRegistry: '$(ACR_NAME).azurecr.io'
  helmChartPath: 'ops/helm/auth-service'

stages:
- stage: Build_Test_Scan
  jobs:
  - job: Build
    steps:
    - task: UseDotNet@2
      inputs: { packageType: 'sdk', version: '8.x' }

    - script: dotnet restore
      displayName: Restore

    - script: dotnet build -c $(buildConfiguration) -warnaserror
      displayName: Build

    - script: dotnet format --verify-no-changes
      displayName: Code style

    - script: dotnet test tests/AuthService.Tests.Unit -c $(buildConfiguration) --collect:"XPlat Code Coverage"
      displayName: Unit tests

    - script: dotnet test tests/AuthService.Tests.Integration -c $(buildConfiguration)
      displayName: Integration tests

    - script: |
        dotnet tool install --global CycloneDX
        ~/.dotnet/tools/cyclonedx . -o sbom.json
      displayName: SBOM

    - task: Docker@2
      displayName: Build image
      inputs:
        repository: $(imageName)
        command: buildAndPush
        Dockerfile: '**/Dockerfile'
        containerRegistry: '$(dockerServiceConnection)'
        tags: |
          $(Build.SourceVersion)
          latest

    - script: |
        trivy image --exit-code 1 $(containerRegistry)/$(imageName):$(Build.SourceVersion)
      displayName: Container scan

- stage: Migrate_DB
  dependsOn: Build_Test_Scan
  jobs:
  - deployment: Migrate
    environment: 'prod'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              dotnet ef database update \
                --project src/AuthService.Infrastructure \
                --startup-project src/AuthService.Api
            displayName: Apply migrations (gated)
          - script: echo "Create DB backup + capture schema hash here"
            displayName: Pre-migration backup

- stage: Deploy_Canary
  dependsOn: Migrate_DB
  jobs:
  - deployment: Deploy
    environment: 'prod'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              command: upgrade
              chartType: FilePath
              chartPath: $(helmChartPath)
              releaseName: auth-service
              namespace: platform
              overrideValues: |
                image.repository=$(containerRegistry)/$(imageName)
                image.tag=$(Build.SourceVersion)
                canary.enabled=true
```
Helm chart includes readiness probes, HPA, and (optional) Argo Rollouts for progressive traffic shifting.

---

### 🧭 Operational Runbook (quick checklist)
 - Keys in Key Vault; JWKS serving correct kid set; rotation schedule documented.
 - Refresh rotation verified; revoke on password reset & role change.
 - Outbox dispatcher running; outbox backlog alert in place.
 - Idempotency enabled on mutating endpoints; storage TTL configured.
 - Golden signals dashboards deployed; alerts wired to on-call.
 - DB migrations gated; backup/restore rehearsed.
 - CSP report-only → enforcing rollout plan completed.
 - Canary deploy tested; rollback is one command.

---

### ⚠️ Common Pitfalls (and how this template avoids them)
- Stale keys & invalid tokens → RSA keys rotate with dual acceptance windows; JWKS published.
- Double processing on retries → Idempotency + Outbox/Inbox dedupe.
- PII in logs → Redaction policies applied by default.
- “Works on laptop” migrations → Gated migration stage with backup step.
- Observability gaps → OTel + Prometheus + curated Grafana dashboards and alerts.

---

### 👤 Author
Gurminder Sohi — Senior Software Engineer | Full-stack | Cloud & AI
🔗 [LinkedIn](https://www.linkedin.com/in/gurmindersohi/)