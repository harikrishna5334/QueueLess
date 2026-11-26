# QueueLess Roadmap

## Phase 0 - Architecture and Planning (Approved & Complete)
Requirements analysis, domain modeling, entity relationship design (ERD), state machines, authentication design, API conventions, directory structures, testing plan, and foundational Architecture Decision Records (ADR-001 through ADR-009). Approved by project owner on 2026-11-04 (including ADR-004 Accepted with Option 1 selected). Phase 1 implementation remains awaiting explicit project-owner authorization; zero application code or database migrations.

## Phase 1 - Full-Stack Foundation
React + Node.js/Express + PostgreSQL/Prisma modular monolith.
- Core JWT authentication (registration, login, logout, password reset flow).
- Multi-organization structure with global patient identities and scoped staff memberships (`ORG_ADMIN`, `PROVIDER`).
- Location, provider, and service catalog management with composite foreign keys.
- Appointment booking, advance cancellation, check-in window enforcement (opens at `scheduledStartTime - 15 min`, closes at `scheduledStartTime + 15 min`; unserved forfeit to `NO_SHOW`; does not extend to `scheduledEndTime`).
- Provider-specific live queue operations: walk-ins, check-ins, hybrid FIFO queue calling (`LIMIT 1 FOR UPDATE SKIP LOCKED`), skips, recalls, and completions.
- Single active call invariant (`status IN ('CALLED', 'IN_SERVICE')`) enforced via PostgreSQL partial unique index.
- Provider overlap prevention via PostgreSQL `btree_gist` range exclusion (`WHERE (status != 'VOIDED')`).
- Immutable `AuditEvent` ledger with PostgreSQL trigger protection (`P0001`).
- No Redis, WebSockets, Kafka, Docker, or cloud infrastructure.

## Phase 2 - Production Quality and UX
Validation, robust error handling, testing, security hardening, API documentation, and polished responsive UX.
- Comprehensive input validation and sanitization using Zod.
- Standardized RFC 7807 error envelopes and PostgreSQL SQLSTATE error mapping.
- Vitest and Supertest backend test suite covering business edge cases and concurrency locks.
- React Testing Library component and flow tests.
- Polished responsive user interfaces with loading, empty, and error states.
- OpenAPI/Swagger API documentation.

## Phase 3 - Redis + WebSockets
Real-time live queue synchronization and traffic simulation.
- Redis caching for high-read endpoints (clinic directories, provider catalogs).
- WebSocket gateway for instant push notifications of queue position and calling alerts.
- Dedicated synthetic traffic simulator interacting strictly through the public REST API to simulate realistic clinic operations.

## Phase 4 - Kafka Fundamentals
Asynchronous event-driven architecture.
- Local Apache Kafka running via Docker Compose.
- Introduction of the transactional `OutboxEvent` pattern: domain changes write to `outbox_events` atomically.
- KafkaJS producers and consumer groups.
- Topic topology, partitioning strategies, consumer offsets, and at-least-once delivery idempotency.

## Phase 5 - Kafka Connect + PostgreSQL CDC
Change Data Capture (CDC) streaming.
- Local Kafka Connect running via Docker Compose.
- Debezium PostgreSQL connector streaming row changes from the database directly into Kafka topics without dual-write hazards.
- Connector configuration, schema registries, offset management, and failure recovery.

## Phase 6 - Workers and Notifications
Background processing and resilient messaging.
- Dedicated background worker service consuming domain events from Kafka.
- In-app notification delivery.
- Low-cost/free tier transactional email notifications for booking confirmations and password resets.
- Exponential backoff retry strategies and Dead Letter Queues (DLQ).

## Phase 7 - Application Containerization
Production container engineering.
- Multi-stage Dockerfiles for React frontend (Nginx/static) and Express API (Node.js).
- Container networking, environment variable injection, and non-root security.
- Unified local multi-service Docker Compose orchestration with container healthchecks.

## Phase 8 - CI/CD
Automated quality gates.
- Public GitHub Actions pipelines for automated linting, type-checking, unit testing, and integration testing.
- Automated container image building and vulnerability scanning.

## Phase 9 - AWS Deployment
Cost-verified, minimal cloud infrastructure.
- Deployment to Amazon Web Services using minimal, cost-controlled services (e.g. ECS/Fargate, RDS PostgreSQL, S3/CloudFront).
- Strict adherence to free-tier and low-cost boundaries; zero billable resources created without explicit owner approval.

## Phase 10 - Terraform
Infrastructure as Code (IaC).
- Declarative management of AWS cloud resources using Terraform.
- Modular code structure, state locking with S3 and DynamoDB, environment variable separation.

## Phase 11 - Observability
Full-stack telemetry and performance monitoring.
- OpenTelemetry instrumentation for distributed tracing across API and worker boundaries.
- Prometheus metrics collection (HTTP latency, queue depth, active consultations).
- Grafana dashboards and Loki centralized log aggregation.

## Phase 12 - Kubernetes & Helm
Container orchestration.
- Local Kubernetes learning using Kind, k3d, or k3s.
- Manifests for Deployments, Services, ConfigMaps, Secrets, Ingress, and Liveness/Readiness probes.
- Helm charts packaging the QueueLess platform for reproducible multi-environment deployment.

---
*Note: Every phase requires a strictly scoped task, explicit acceptance criteria, verification evidence, and owner review before progressing to the next phase.*
