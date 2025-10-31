# CLAUDE.md
# Purpose
This document guides Claude Code when working in this repository. It defines architecture, conventions, commands, testing, permissions, and unsafe areas. Claude: follow this strictly, ask for confirmation if in doubt, and prefer making incrementally testable changes.

# Architecture Overview
- Stack: .NET 8 (ASP.NET Core), C#, EF Core, MediatR, FluentValidation
- Style: Hexagonal/Clean Architecture per service
- Services:
  - IdentityService (AuthN/AuthZ, JWT)
  - CatalogService (Products, Categories)
  - OrderingService (Orders, Sagas)
  - PaymentService (Payments, Webhooks)
  - NotificationService (Email/SMS, Outbox)
- Cross-cutting:
  - API Gateways: YARP reverse proxy, BFF where needed
  - Messaging: RabbitMQ (AMQP) for events/commands
  - Persistence: PostgreSQL (primary), Redis (cache), MongoDB (read models optional)
  - Observability: OpenTelemetry -> OTLP -> Grafana Tempo/Prometheus/Loki; Serilog to console/Seq
  - Config: appsettings.json + env vars; Azure Key Vault (prod)
  - Containers: Docker, docker-compose; k8s manifests under /deploy/k8s
  - CI/CD: GitHub Actions under /.github/workflows

# Repo Structure
- /services/<ServiceName>/
  - src/<ServiceName>.Api/
  - src/<ServiceName>.Application/
  - src/<ServiceName>.Domain/
  - src/<ServiceName>.Infrastructure/
  - tests/<ServiceName>.*Tests/
- /building-blocks/
  - Messaging/, Persistence/, Observability/, Web/
- /gateway/
  - YarpGateway/
- /deploy/
  - docker-compose.yml, k8s/, helm/
- /.claude/commands/ (custom slash commands)
- /.editorconfig, .globalconfig, Directory.Build.props, Directory.Build.targets

# Coding Conventions
- C# 12, .NET 8 SDK
- Nullable reference types enabled
- Async all the way, no .Result/.Wait()
- Use records for immutable DTOs, readonly structs for perf-sensitive value types
- Controllers minimal; business logic via MediatR handlers
- Validation via FluentValidation in Application layer
- Domain events raised in Aggregate roots; use Outbox pattern for reliable publishing
- Strongly-typed IDs (e.g., ProductId) to avoid primitive obsession
- Logging: structured; no log PII. Use Serilog with enrichment.
- Configuration: IOptions<T>; never read environment directly in business code
- Security: JWT bearer; validate scopes/roles; input sanitization; output encoding
- Error handling: ProblemDetails for APIs, consistent error codes
- API versioning with Microsoft.AspNetCore.Mvc.Versioning
- HTTP clients: Refit or HttpClientFactory; set reasonable timeouts/retries with Polly

# Messaging & Transactions
- Commands: queue-per-service, events: topic-per-aggregate
- Use message contracts in /building-blocks/Messaging.Contracts
- Idempotency via message dedup + Outbox
- Sagas/Process Managers in OrderingService for long-running workflows
- Never publish domain events directly from controllers; use Application layer

# Testing Strategy
- Unit: xUnit, FluentAssertions
- Integration: WebApplicationFactory + Testcontainers (Postgres, RabbitMQ)
- Contract: Pact (consumer/provider) for cross-service HTTP APIs
- End-to-end: docker-compose + Playwright for BFF/UI
- Performance: k6 scripts in /tests/Perf
- Security: Dependency-check, Snyk (CI); middleware tests for authz/authn

# Observability
- OpenTelemetry:
  - Traces: ASP.NET Core, HTTP client, EF Core, RabbitMQ
  - Metrics: request duration, DB duration, message latency
  - Logs: JSON, CorrelationId, TraceId
- Propagate trace context across services; use W3C traceparent headers

# Local Development
- Prereqs: .NET 8 SDK, Docker, Node 20 (if UI), RabbitMQ, Postgres
- Quick start:
  - docker-compose up -d
  - dotnet build
  - dotnet test
  - Run a service:
    - cd services/CatalogService/src/CatalogService.Api
    - dotnet run

# Security & Secrets
- Do NOT commit secrets. Use .env and user-secrets for local.
- Keys: store in Azure Key Vault (prod) or dotenv (local).
- Rotate tokens regularly; short-lived JWTs with refresh via IdentityService.
- Validate all inbound events; use schema validation.

# Dangerous Areas (ask before changing)
- Migrations in production folders
- Message contracts in /building-blocks/Messaging.Contracts
- Public API models under /services/*/src/*/Api/Contracts
- CI workflows in /.github/workflows
- Kubernetes manifests under /deploy/k8s (cluster-impacting)
- Global props/targets files

# Performance Guidelines
- Prefer async streaming for large payloads
- Paginate all list endpoints; max page size 100
- Cache hot reads (Redis) with sensible TTL; cache aside pattern
- EF Core: NoTracking for queries; compiled queries for hot paths
- Batch DB ops; avoid N+1; use Include judiciously

# Error & Retry
- Use Polly policies for HTTP: retry (exponential backoff), circuit breaker
- RabbitMQ consumers: retry with DLQ; poison message handling
- Never infinite retries; prefer deadletter with alert

# API Guidelines
- RESTful resource naming; plural nouns
- Consistent status codes; ProblemDetails on errors
- Request/response validation; model binding robustness
- Include ETags for cacheable resources
- Ensure idempotency for PUT/PATCH

# Versioning & Backward Compatibility
- Support at least N and N-1 API versions
- Event versioning with upcasters; no breaking changes to existing consumers
- Contracts change via additive fields; avoid removal/rename without adapter

# Database
- Postgres:
  - Naming: snake_case tables/columns
  - One schema per service
- EF Core migrations: under Infrastructure/Migrations
- Ensure transactional boundaries; no cross-service DB access

# Code Review Rules
- Focus: correctness, security, observability, test coverage
- Block changes that reduce telemetry, remove validation, or bypass auth
- Require tests for all non-trivial changes
- Ensure public contracts have samples and docs

# Release Process
- PR -> CI (build, tests, security scan) -> staging deploy -> canary -> prod
- Use semantic versioning
- Feature flags for risky changes

# Folder-Specific Notes
- /building-blocks/Observability: Otel setup, Serilog sinks, correlation middleware
- /building-blocks/Messaging: RabbitMQ connection factory, publisher/subscriber abstractions
- /building-blocks/Persistence: EF common, outbox, unit of work
- /gateway/YarpGateway: routes.json, clusters.json, auth policies

# Required Headers & Middleware
- CorrelationId propagation
- Request logging
- Global exception handling -> ProblemDetails
- Rate limiting for public APIs
- CORS configured per environment

# How Claude Should Work Here
- Always start with: readme, this CLAUDE.md, and the target service
- Generate a plan, ask for confirmation, then implement stepwise
- Prefer minimal edits; keep diffs small and well-tested
- Add/update tests with every change
- If changing contracts or messages, propose migration steps and adapters

# Example Tasks & Steps
- Add a GET /products search endpoint to CatalogService:
  1. Define query model + validator (Application)
  2. Handler: query DB (NoTracking), map to DTO
  3. Controller action + route + pagination
  4. Update OpenAPI annotations
  5. Tests: unit for validator/handler; integration for endpoint
- Publish ProductCreated event:
  1. Raise domain event in aggregate
  2. Outbox write in transaction
  3. Background publisher sends to RabbitMQ
  4. Consumer contract in interested services
  5. Tests: outbox -> publish -> consume flow

# NFRs (Non-Functional Requirements)
- Availability: 99.9%
- Latency: p95 < 200ms for simple reads
- Throughput: scale horizontally; autoscale on CPU/RPS
- Security: OWASP top 10 mitigations; regular dependency patching

# Tooling & Commands
- dotnet build/test/run; ef migrations; format/analyzers enabled
- Codegen:
  - Swagger client generation via NSwag in consumers
- Lint:
  - dotnet format; analyzers: Roslyn, Sonar (CI)
- Make targets (if present):
  - make up/down/build/test

# Slash Commands (Claude Code)
Describe-only commands; Claude should translate these to steps and ask before executing.
- /service:new <Name>
  - Scaffold a new service with Clean Architecture layers, default observability, Dockerfile, migration baseline, and tests.
- /api:add <Service> <Verb> <Route>
  - Add controller action, DTOs, validators, MediatR handler, OpenAPI doc, and tests.
- /event:publish <Service> <EventName>
  - Add domain event, outbox integration, publisher, contract, and tests.
- /consumer:add <Service> <EventName>
  - Add RabbitMQ consumer, handler, idempotency check, and tests.
- /http-client:add <Service> <TargetService>
  - Add Refit interface, Polly policies, tracing, and tests.
- /perf:test <Service> <Endpoint>
  - Scaffold k6 test with thresholds; add to CI if approved.

# Environments
- Local: docker-compose, dev secrets
- Staging: full stack, feature flags on
- Prod: Key Vault, autoscaling, high availability, backups

# OpenAPI & Docs
- Each service exposes /swagger
- Keep endpoints documented with examples
- Update changelog and API docs on contract changes

# Dependencies & Versions
- .NET SDK: 8.x
- EF Core: 8.x
- MediatR: 12.x
- FluentValidation: 11.x
- Serilog: 3.x
- OpenTelemetry: 1.x
- Polly: 8.x
- RabbitMQ.Client: 6.x
- Refit: 7.x
- Testcontainers: 3.x

# Edge Cases To Watch
- Partial failures in sagas -> compensation steps
- Duplicate messages -> ensure idempotency keys
- Large payloads -> streaming, chunking, size limits
- Clock skew -> use UTC everywhere
- Schema drift -> migration safety checks

# CI Gates
- Build + tests must pass
- Coverage >= 80% on Application/Domain
- No critical vulnerabilities
- Format and analyzers clean
- Pact contracts verified

# Final Notes
- Keep public APIs stable; prefer additive changes
- Preserve telemetry; never reduce visibility
- Ask before modifying dangerous areas
- Always add tests and docs with feature changes
