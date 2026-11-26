# QueueLess - Project Context

## Purpose

Build a portfolio-grade appointment-booking and provider-specific live-queue platform. Healthcare is fictional demonstration domain only; this is not a medical-record system.

## Learning priorities

1. Backend Node.js and TypeScript.
2. PostgreSQL, relational modelling, API design, auth and concurrency.
3. Event-driven architecture with Kafka and Kafka Connect/CDC.
4. DevOps: Docker, CI/CD, AWS, Terraform, observability, Kubernetes.
5. React frontend as a polished supporting product layer.

## Architecture progression

Modular monolith -> real-time modular monolith -> Kafka/event-driven processing -> workers and notifications -> application containers -> CI/CD -> AWS -> Terraform -> observability -> Kubernetes/Helm.

## Initial stack (proposed direction; phase-gated)

- Web: React, TypeScript, Vite, Tailwind CSS, shadcn/ui, React Router, TanStack Query, React Hook Form, Zod, Vitest, React Testing Library.
- API: Node.js, TypeScript, Express, Zod, Prisma, PostgreSQL, JWT, structured logging, OpenAPI/Swagger, Vitest, Supertest.
- Later: Redis, WebSockets, Kafka/KafkaJS, Kafka Connect, PostgreSQL CDC, workers, retries, DLQ, Docker, GitHub Actions, AWS, Terraform, OpenTelemetry, Prometheus, Grafana, Loki, Kubernetes, Helm.

## Core domain decisions

- Roles: PATIENT, PROVIDER, ORG_ADMIN. Universal User identity; patients self-register globally without organization binding.
- Organization owns locations and staff relationships; no advanced enterprise multi-tenancy.
- Provider is an organization-scoped profile linked to a User (`@@unique([userId, organizationId])`); a user may practice at multiple organizations.
- Dual memberships permitted: a user may hold both `ORG_ADMIN` and `PROVIDER` memberships in the same organization (`@@unique([userId, organizationId, role])`).
- A Service may be offered by multiple Providers through ProviderService; duration belongs to Service and is snapshotted onto QueueToken (`durationMinutesSnapshot`).
- Appointment and QueueToken are separate concepts.
- Appointment patients receive a QueueToken at check-in; walk-ins can join without an appointment.
- Guest walk-ins track status via 256-bit CSPRNG secrets (stored as SHA-256 hash at rest, submitted via `X-Queue-Tracking-Token` header, redacted from logs).
- Queues are provider-specific in the initial version.
- Queue ordering uses hybrid FIFO: one-time recalled priority tokens first (`priority_next = TRUE`), oldest `recalled_at` first, then FIFO by `queue_order_time`. Priority is consumed when called. Reinstated tokens rejoin at the ordinary queue tail. Wait time is estimated dynamically per-token using `durationMinutesSnapshot` plus remaining active consultation estimate (for `IN_SERVICE`, elapsed time starts at `serviceStartedAt` and clamps to 0 if overrunning; for `CALLED`, consultation may not have begun, so elapsed time since `calledAt` is not subtracted and the estimate is less certain until service starts).
- Appointment states: BOOKED, CHECKED_IN, CANCELLED, NO_SHOW, COMPLETED.
- QueueSession states: SCHEDULED, OPEN, PAUSED, CLOSED, VOIDED. `ORG_ADMIN` may void only while SCHEDULED and zero tokens created/served.
- QueueToken states: WAITING, CALLED, IN_SERVICE, COMPLETED, SKIPPED, CANCELLED.
- Approved timing rules: Check-in window opens at `scheduledStartTime - 15 min` and closes at `scheduledStartTime + 15 min` (`APPOINTMENT_CHECKIN_GRACE_MINUTES = 15`; a BOOKED appointment not checked in by this deadline transitions to `NO_SHOW`; window does not extend to `scheduledEndTime`); call response window is 5 min (`CALLED_RESPONSE_WINDOW_MINUTES = 5`); skipped return window is 30 min (`SKIPPED_RETURN_WINDOW_MINUTES = 30`: recall allowed through `NOW() <= skippedAt + 30 min`, `NO_SHOW_UNRESPONSIVE` permitted only when `NOW() > skippedAt + 30 min`, defaults to `UNSERVED_CLINIC_CLOSED` if session closes while `NOW() <= skippedAt + 30 min`).
- WAITING tokens are never marked `NO_SHOW_UNRESPONSIVE`. If a session closes before the skipped return window expires, the token defaults to `UNSERVED_CLINIC_CLOSED`.
- Reinstatement: Maximum 2 reinstatements per token (`reinstatementCount <= 2`) while session is OPEN. Eligible for `ACCIDENTAL_CANCELLATION` and `PATIENT_DEPARTED`. `NO_SHOW_UNRESPONSIVE` is permanently ineligible.
- Use UUID identifiers and UTC timestamps. Queue sessions use half-open intervals `[startTime, endTime)` with PostgreSQL GiST exclusion `WHERE (status != 'VOIDED')` and `CHECK (start_time < end_time)`.
- Concurrency & locking: Explicit hierarchy `Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken`. `QueueSession FOR UPDATE` serializes call-next, token reinstatement, and session closure. Reinstatement re-verifies session is `OPEN` and locks `Appointment` before `QueueToken`. Start Service and Session Closure resolve both race orderings deterministically: in Race A, Start Service locks `QueueToken` first and transitions to `IN_SERVICE`, causing Session Closure's post-lock check to abort with HTTP 409 `ACTIVE_CONSULTATION_IN_PROGRESS`; in Race B, Session Closure locks `QueueToken` first and closes the session, causing Start Service's post-lock check on `QueueToken` to detect the closed session and reject with HTTP 409 `QUEUE_SESSION_CLOSED`. Session closure executes in strict hierarchy order: acquires `QueueSession FOR UPDATE`, verifies no token is `IN_SERVICE`, identifies records without premature token locks, locks linked `Appointment` rows in PK order, locks affected `QueueToken` rows in PK order, revalidates states under lock (aborting with `ACTIVE_CONSULTATION_IN_PROGRESS` if any token transitioned to `IN_SERVICE`), deterministically resolves tokens (WAITING to UNSERVED_CLINIC_CLOSED and never NO_SHOW; voluntary departures PATIENT_DEPARTED recorded during open sessions; CALLED NO_SHOW only after 5 min; SKIPPED NO_SHOW only strictly after 30 min; unexpired default to UNSERVED_CLINIC_CLOSED; appointment vs. walk-in distinct; staff cannot arbitrarily choose no-show), sets session `CLOSED`, and commits atomically. `SKIP LOCKED` skips rows locked by concurrent transactions without affecting non-locking queries. The analyzed queue lifecycle paths follow their documented lock acquisition order and deterministic primary-key ordering, subject to the stated transaction assumptions. This is not a proof of application-wide deadlock freedom.
- `AuditEvent` is an immutable write-once ledger (`ON DELETE RESTRICT` on organization, trigger `trg_audit_events_immutable` raises an exception with stable message 'Audit records are immutable...' [SQLSTATE `P0001`]; application maps to `AUDIT_LOG_IMMUTABLE` only when both SQLSTATE and message contract match; triggers protect against application bugs but not database superusers). `OutboxEvent` is a separate mechanism introduced in Phase 4 for Kafka.
- Use synthetic fictional data only. No clinical records, real patient data, payments, billing, subscriptions or SMS implementation.

## Phase roadmap

0. Architecture and planning.
1. React + Node.js + PostgreSQL modular monolith.
2. Production quality, validation, security, testing, UX.
3. Redis + WebSockets + synthetic traffic simulator.
4. Kafka fundamentals using local Docker Compose (OutboxEvent introduced).
5. Kafka Connect + PostgreSQL CDC using local Docker Compose.
6. Workers, in-app/email notifications, retries and DLQ.
7. Application containerization.
8. GitHub Actions CI/CD.
9. AWS deployment.
10. Terraform.
11. Observability.
12. Local Kubernetes/Helm; managed Kubernetes optional.

## Rules

- Read `AGENTS.md` and the active task before work.
- Phase 0 is documentation only. Do not create app code, dependencies, Prisma schema/migrations, initialize databases, or provision infrastructure.
- Explain plans and acceptance criteria before changing files.
- Ask approval for architecture, security, dependency, cost, or scope changes.
- Never claim unverified work or tests.
- Keep the repository model-independent; files and Git are the source of truth.
- Do not run paid cloud provisioning or Terraform apply without explicit approval.

## Current state

See `docs/project-context/current-status.md`. If anything here conflicts with the constitution or a later owner-approved ADR, report the conflict instead of silently resolving it.
