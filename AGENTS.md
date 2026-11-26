# QueueLess - Project Constitution


## 1. Project Identity


QueueLess is a portfolio-grade full-stack appointment and live queue management platform.


The project is primarily designed to teach and demonstrate:


- Backend Node.js engineering

- TypeScript

- PostgreSQL and relational database design

- API design

- Authentication and authorization

- Real-time systems

- Event-driven architecture

- Apache Kafka

- Kafka Connect and CDC

- Reliability and idempotency

- Docker

- CI/CD

- AWS

- Terraform

- Observability

- Kubernetes


React is included from Day 1 so the project is a complete, demonstrable product rather than a backend-only CRUD application.


The primary engineering focus remains:


1\. Backend

2\. Event-driven architecture

3\. DevOps/cloud

4\. Frontend as a supporting product layer


---


# 2. Golden Rule


Every technology must have a reason.


Do not introduce a technology merely because it is popular, modern, or commonly seen in job descriptions.


The project must evolve in phases.


AI accelerates implementation and learning but does not replace understanding.


The developer must be able to explain the important architecture, business rules, database relationships, API contracts, infrastructure decisions, and code produced by AI.


---


# 3. AI / Antigravity Operating Rules


Antigravity acts as:


- Senior software engineer

- Solution architect

- Backend mentor

- Full-stack pair programmer

- DevOps mentor

- Code reviewer

- Testing assistant


Antigravity must NOT act as an autonomous developer that silently changes architecture.


Before significant implementation:


1\. Understand the requirement.

2\. Read this file.

3\. Identify the current phase.

4\. Explain the proposed design.

5\. Explain affected files/modules.

6\. Explain dependencies.

7\. Explain database/API implications.

8\. Identify edge cases.

9\. Identify tests.

10\. Wait for approval when a major architectural decision is involved.


Never silently:


- change the database model

- add a dependency

- introduce a new framework

- introduce a future-phase technology

- change API contracts

- change authentication strategy

- change architectural boundaries

- introduce microservices

- add infrastructure

- modify the phase sequence


If a change is necessary, explain why and request approval.


Do not consider "the code compiles" equivalent to "the feature is complete."


---


# 4. Development Philosophy


Use this progression:


Requirement

-> Architecture

-> Review

-> Small implementation increment

-> Tests

-> Manual verification

-> Code review

-> Documentation

-> Git commit


Do not ask AI to build the entire project in one step.


Do not generate the entire application before validating the first vertical slice.


---


# 5. Current Development Phase


Current phase status:


## Phase 0 - Architecture and Planning (Approved and Complete)


Phase 0 was formally reviewed and approved by the project owner on 2026-11-04. All architecture decision records (ADR-001 through ADR-009) are accepted, with ADR-004 adopting Option 1 (In-Memory Access Token + HttpOnly Refresh Cookie).


Phase 1 is not started. Phase 1 implementation is awaiting explicit project-owner authorization before any application code, package manifests, database schemas, migrations, or infrastructure are created.


Next phase:


## Phase 1 - Full-Stack Foundation


---


# 6. Target Technology Stack


## Frontend


- React

- TypeScript

- Vite

- Tailwind CSS

- shadcn/ui

- React Router

- TanStack Query

- React Hook Form

- Zod

- Vitest

- React Testing Library

- WebSocket client later


## Backend


- Node.js

- TypeScript

- Express

- Zod

- Prisma

- PostgreSQL

- JWT

- structured logging

- OpenAPI/Swagger

- Vitest

- Supertest


## Event-driven


- Apache Kafka

- KafkaJS initially

- Kafka Connect

- PostgreSQL CDC

- source/sink connectors

- retry strategies

- DLQ

- idempotency


## Infrastructure


- Docker

- Docker Compose

- GitHub Actions

- AWS

- Terraform

- Kubernetes

- Helm

- optional GitOps later


## Observability


- OpenTelemetry

- Prometheus

- Grafana

- Loki


Do not introduce these technologies before their designated phase unless explicitly approved.


---


# 7. Repository Structure


Use a monorepo.


Preferred structure:


queueless/


|-- apps/

|   |-- web/

|   \-- api/

|

|-- packages/

|   |-- shared/

|   \-- config/

|

|-- infrastructure/

|   |-- docker/

|   |-- terraform/

|   |-- kubernetes/

|   \-- monitoring/

|

|-- docs/

|   |-- api/

|   |-- architecture/

|   |-- decisions/

|   |-- diagrams/

|   |-- domain/

|   |-- phases/

|   |-- project-context/

|   \-- tasks/

|

|-- .github/

|   \-- workflows/

|

|-- AGENTS.md

|-- README.md

|-- LICENSE

|-- SECURITY.md

|-- package.json

|-- pnpm-workspace.yaml

|-- .gitignore

\-- docker-compose.yml


---


# 8. Frontend Architecture


apps/web/src/


|-- app/

|   |-- router/

|   |-- providers/

|   \-- layouts/

|

|-- features/

|   |-- auth/

|   |-- appointments/

|   |-- queue/

|   |-- providers/

|   |-- services/

|   |-- notifications/

|   \-- dashboard/

|

|-- components/

|   |-- ui/

|   |-- forms/

|   |-- navigation/

|   \-- feedback/

|

|-- hooks/

|-- lib/

|   |-- api/

|   |-- websocket/

|   \-- utils/

|-- types/

|-- assets/

\-- main.tsx


Use TanStack Query for server state.


Do not introduce Redux, Zustand, MobX, or another state-management framework unless explicitly justified later.


Use React/local state for UI state.


Frontend must include:


- loading states

- error states

- empty states

- form validation

- responsive design

- accessible interaction patterns

- meaningful status indicators


---


# 9. Backend Architecture


apps/api/src/


|-- config/

|-- middleware/

|-- modules/

|   |-- auth/

|   |-- users/

|   |-- organizations/

|   |-- locations/

|   |-- providers/

|   |-- services/

|   |-- appointments/

|   |-- queue/

|   \-- notifications/

|

|-- database/

|   |-- prisma/

|   \-- seed/

|

|-- shared/

|   |-- errors/

|   |-- logger/

|   |-- types/

|   \-- utils/

|

|-- app.ts

\-- server.ts


Request flow:


Route

-> Middleware

-> Controller

-> Service / Use Case

-> Repository

-> Prisma

-> PostgreSQL


Controllers remain thin.


Business rules belong in application/domain logic.


Repositories isolate persistence concerns.


Avoid unnecessary abstraction and generic-framework patterns.


---


# 10. Domain Model


Core entities:


- User

- Organization

- OrganizationMembership

- Location

- Provider

- ProviderService

- Service

- ProviderAvailability

- Appointment

- QueueSession

- QueueToken

- Notification

- AuditEvent


Relationships:


Organization

-> Location

-> Provider

-> ProviderService

-> Service


User

-> OrganizationMembership

-> Organization


Provider

-> ProviderAvailability


Appointment

-> User

-> Provider

-> Service

-> Location


Appointment and Walk-in

-> QueueToken


QueueSession

-> Provider

-> Service

-> Location

-> Date/session


QueueSession

-> QueueToken


---


# 11. Organization Model


QueueLess supports multiple organizations structurally.


The initial example domain may use fictional healthcare organizations, but the model must not be hardcoded around one real hospital.


Example:


CarePoint Health Center


with:


- Gachibowli location

- Kondapur location

- fictional providers

- fictional services


Do not implement advanced enterprise multi-tenancy.


Do not implement:


- tenant databases

- tenant-specific infrastructure

- tenant billing

- tenant schemas

- complex tenant routing


The organization model exists primarily to provide realistic ownership and authorization boundaries.


---


# 12. User and Role Model


Authentication identity belongs to User.


Roles:


- PATIENT

- PROVIDER

- ORG_ADMIN


Provider is an organization-scoped profile associated with a User rather than a separate authentication system. A user may hold a Provider profile across multiple distinct organizations (@@unique([userId, organizationId])).


OrganizationMembership determines the user's organization-scoped staff authorization. A user may hold both ORG_ADMIN and PROVIDER memberships within the same organization (@@unique([userId, organizationId, role])).


Patient:


- self-registers with universal platform identity; does not require organization membership to discover clinics or book appointments.


Provider:


- created by an organization admin within that organization; Provider record is anchored to an active PROVIDER membership via composite foreign key.


Organization admin:


- initially seeded/platform-controlled


Do not create a super-admin or complex enterprise RBAC system in Phase 1.


---


# 13. Authentication


Use JWT-based authentication.


Passwords must use an appropriate password hashing algorithm such as Argon2id.


Never:


- store plaintext passwords

- hash passwords with plain SHA-256

- hardcode secrets


Authentication must include:


- registration

- login

- logout behavior

- protected routes

- authorization

- forgot password

- reset password


Forgot-password email infrastructure is deferred until the notification phase.


Phase 1 may use a development-only email/log mechanism.


Follow the accepted ADR-004 decision (Option 1: In-Memory Access Token + HttpOnly Refresh Cookie) and its security controls: short-lived in-memory access tokens (15 minutes), database-backed rotating refresh tokens (7 days) in HttpOnly cookies with Path=/api/v1/auth, server-side Origin/Referer allowlist validation, SameSite defense in depth, and anti-CSRF token verification on cookie-authenticated refresh and logout endpoints.


---

# 14. Appointment Rules


Appointment belongs to:


- patient/user

- provider

- service

- location

- scheduled start/end time


Service duration is defined per service.


Example:


General Consultation = 30 minutes

Follow-up = 15 minutes

Specialist Consultation = 45 minutes


A provider cannot have overlapping appointments.


A patient cannot have overlapping appointments.


A patient may hold multiple future appointments if they do not overlap.


Appointment states:


- BOOKED

- CHECKED_IN

- CANCELLED

- NO_SHOW

- COMPLETED


Appointment check-in window opens at scheduledStartTime - 15 minutes and closes at scheduledStartTime + 15 minutes (APPOINTMENT_CHECKIN_GRACE_MINUTES = 15). If a patient has not checked in by this deadline, the appointment may transition to NO_SHOW. The check-in window does not extend to scheduledEndTime.


If an active consultation is aborted mid-service, the appointment transitions from CHECKED_IN to CANCELLED with reason CONSULTATION_ABORTED.


Do not add unnecessary appointment states.


---


# 15. Provider Availability


Providers have recurring weekly working hours.


Schedules may differ between providers.


Example:


Provider A:

Monday-Friday, 09:00-17:00


Provider B:

Monday-Saturday, 10:00-18:00


Model basic recurring weekly availability.


Do not implement complex holiday/leave/exception scheduling in Phase 1.


---


# 16. Services


A service can be offered by multiple providers.


Use:


Provider

<->

ProviderService

<->

Service


Service contains its duration.


Do not create a one-provider-per-service limitation.


---


# 17. Queue Model


A QueueSession represents a provider's queue session at a specific location for a designated operational window [startTime, endTime) on a calendar date.


Phase 1 queue is provider-specific. A provider's queue session serves all services that the provider offers at that location. Each QueueToken snapshots the service's duration at token creation (durationMinutesSnapshot).


Example:


Dr. A + CarePoint Gachibowli + 2026-09-19 (09:00-13:00)

-> QueueSession A (serves both General Consultation and Follow-up tokens)


Dr. B + CarePoint Gachibowli + 2026-09-19 (10:00-14:00)

-> QueueSession B


Do not implement pooled multi-provider queues in Phase 1.


A pooled queue may be documented as a future enhancement.


---


# 18. Walk-ins


QueueLess supports both:


- appointments

- walk-ins


Both can ultimately produce a QueueToken.


Appointment flow:


Appointment

-> Check-in

-> QueueToken


Walk-in flow:


Walk-in

-> QueueToken


Do not require an appointment before joining a queue. Guest walk-ins receive a cryptographically random 256-bit tracking secret whose SHA-256 hash is stored at rest.


---


# 19. Queue Ordering


Queue ordering uses a hybrid FIFO policy:


1. One-time recalled priority tokens first (priority_next = TRUE).

2. Among priority tokens, oldest recalled_at first (recalled_at ASC NULLS LAST).

3. Ordinary waiting tokens follow chronological arrival via queue_order_time ASC.


When a token is called, its priority is immediately consumed (priority_next = FALSE).


When an eligible cancelled token is reinstated by staff, its queue_order_time is reset to NOW(), placing it fairly at the tail of the ordinary waiting queue.


---


# 20. Check-in


Appointment patients may check in starting 15 minutes before their scheduled start time up to 15 minutes after (scheduledStartTime - 15 minutes to scheduledStartTime + 15 minutes). If a patient has not checked in by this deadline, the appointment transitions to NO_SHOW. The check-in window does not extend to scheduledEndTime.


Phase 1 should keep the check-in rules simple.


Walk-ins may join while the relevant queue session is open.


Do not build an elaborate appointment-window engine.


---


# 21. Queue State


QueueSession states:


- SCHEDULED

- OPEN

- PAUSED

- CLOSED

- VOIDED


Lifecycle:


SCHEDULED -> OPEN <-> PAUSED -> CLOSED

SCHEDULED -> VOIDED


OPEN is the active operational state in which check-ins and patient calls take place.


Provider can pause an open queue.


When paused:


- existing tokens remain

- provider cannot call the next patient

- patients see that the queue is temporarily paused


A session may be VOIDED only while in SCHEDULED status, only if zero tokens were ever created or served, and only by an ORG_ADMIN. VOIDED and CLOSED are terminal.


---


# 22. QueueToken State


QueueToken states:


- WAITING

- CALLED

- IN_SERVICE

- COMPLETED

- SKIPPED

- CANCELLED


Normal flow:


WAITING

-> CALLED

-> IN_SERVICE

-> COMPLETED


Timing and grace period rules:


- CALLED patient response window: 5 minutes (CALLED_RESPONSE_WINDOW_MINUTES = 5). Provider must wait the full 5 minutes before marking a patient SKIPPED. Cannot be marked no-show before this window expires.

- SKIPPED patient return window: 30 minutes (SKIPPED_RETURN_WINDOW_MINUTES = 30). Provider may execute a one-time priority recall (recallCount <= 1) through NOW() <= skippedAt + 30 minutes while the session is OPEN. Transition to NO_SHOW_UNRESPONSIVE is permitted only when NOW() > skippedAt + 30 minutes. If the session closes while NOW() <= skippedAt + 30 minutes, the token defaults to UNSERVED_CLINIC_CLOSED.

- WAITING tokens are never automatically marked as no-show.

- Reinstatement: Permitted at most twice per token (reinstatementCount <= 2) by authorized staff while the session is OPEN. Eligible for ACCIDENTAL_CANCELLATION and PATIENT_DEPARTED. NO_SHOW_UNRESPONSIVE is permanently ineligible for reinstatement. Reinstated tokens rejoin at the end of the ordinary queue.


Alternative transitions must be explicitly validated.


---


# 23. Appointment / Queue Separation


Appointment and QueueToken are separate domain concepts.


An appointment represents a scheduled booking.


A queue token represents actual participation in the live queue.


This allows:


- no-show appointments

- walk-ins

- delayed check-ins

- queue history

- real-time queue updates

- future analytics


Do not collapse Appointment and QueueToken into one entity.


---


# 24. Queue Wait Estimate


Wait time is calculated dynamically per-token using each token's service duration snapshot (durationMinutesSnapshot) to support mixed-service queues:


Estimated Wait = Remaining Active Estimate + Sum of durationMinutesSnapshot for all WAITING tokens ahead


Tokens ahead include priority recalled tokens (priority_next = TRUE, recalled_at ASC NULLS LAST) followed by ordinary waiting tokens (queue_order_time ASC).


Active consultation remaining estimate:
- For IN_SERVICE, elapsed consultation time starts at serviceStartedAt: MAX(0, activeToken.durationMinutesSnapshot - elapsedMinutesSinceStart). If a consultation exceeds its scheduled duration, remaining time evaluates to 0 minutes, acknowledging the approximation limitation that in-progress consultations cannot be predicted with certainty.
- For CALLED, the consultation may not have started yet (patient is responding or walking in). Elapsed time since calledAt is not subtracted as though consultation had begun. The estimate is less certain while a token is CALLED (evaluating to the full durationMinutesSnapshot) and may be recalculated when service starts.


The result must be presented as an estimate, not a guaranteed appointment time.


Later phases may improve this using historical service-time data.


---


# 25. Cancellation


Patients may cancel appointments before check-in while in BOOKED status.


After check-in, the live QueueToken becomes the operational control for queue participation. If a patient voluntarily departs after check-in, QueueToken cancellation with reason PATIENT_DEPARTED synchronizes the linked Appointment to CANCELLED, preserving audit events and approved reinstatement rules.


Do not implement complex cancellation policies in Phase 1.


---


# 26. Provider Account Creation


Organization admin creates provider accounts.


The operation conceptually creates:


- User

- Provider profile

- OrganizationMembership


with:


role = PROVIDER


Real email invitations are deferred.


Seed data can create provider accounts during development.


---


# 27. Admin UI


Phase 1 includes a minimal organization-admin interface.


Admin can:


- view/manage locations

- create/edit providers

- activate/deactivate providers

- create/edit services


Do not build:


- billing

- advanced analytics

- enterprise permission management

- complex organization settings


---


# 28. IDs and Time


Use UUIDs consistently for primary identifiers.


Human-friendly queue display numbers may exist separately from UUIDs.


Store timestamps in UTC.


Locations specify their IANA timezone (e.g. "Asia/Kolkata"). Frontend displays timestamps in the appropriate user or organization timezone.


QueueSession operational windows are modeled as half-open UTC timestamp intervals [startTime, endTime) with a database check constraint requiring startTime < endTime.


Do not store business timestamps as arbitrary local-time strings.


---


# 29. Database Integrity


Important business invariants must be protected at both:


1. application/business logic level

2. PostgreSQL constraint/transaction level where practical


Examples:


- unique email

- foreign keys with restrictive deletion (do not cascade-delete appointments, sessions, or audit history)

- composite foreign key anchoring Provider to OrganizationMembership with role = 'PROVIDER'

- strict 1:1 database constraint between appointment and token (@@unique([appointmentId]))

- single active call per session partial unique index (status IN ('CALLED', 'IN_SERVICE'))

- PostgreSQL GiST exclusion constraint on QueueSession preventing provider shift overlap WHERE (status != 'VOIDED')

- check constraint requiring QueueSession startTime < endTime

- immutable AuditEvent ledger with ON DELETE RESTRICT on organization_id and a PostgreSQL trigger (trg_audit_events_immutable) rejecting UPDATE and DELETE via exception with stable message 'Audit records are immutable and cannot be modified or deleted' (SQLSTATE P0001). Application maps to AUDIT_LOG_IMMUTABLE only when both SQLSTATE P0001 and this message contract match, avoiding false-positive mapping of generic P0001 errors. Database triggers protect against application logic bugs and operational errors, but do not protect against database superusers or database owners (who can bypass triggers via session_replication_role or table alteration).


Do not rely solely on frontend validation.


---


# 30. Transactions and Concurrency


Concurrency is an explicit learning objective.


Important operations such as booking/check-in/calling the next patient must be designed with appropriate transactional behavior.


The system must prevent:


- duplicate queue tokens

- duplicate check-ins

- conflicting appointments

- two simultaneous "call next" operations selecting the same patient

- booking or session-creation racing with provider deactivation


Coordination and locking rules:


- Provider row acts as the canonical coordination lock: deactivation acquires FOR UPDATE; appointment booking and session creation acquire FOR SHARE.

- Global lock acquisition hierarchy: Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken. When multiple rows of the same entity are locked, always acquire in deterministic primary-key order (ORDER BY id ASC).

- Multi-table transaction lock consistency: The analyzed queue lifecycle paths follow their documented lock acquisition order and deterministic primary-key ordering, subject to the stated transaction assumptions. This is not a proof of application-wide deadlock freedom. Operations that do not lock QueueSession (Start Service, Skip Patient, Complete Service, Cancellation) operate on an active called/in-service slot protected by unique constraints, or affect only single token/appointment sub-trees without inverting lock order with closure or reinstatement. While Start Service can race with Session Closure between closure's pre-lock check and token lock acquisition, both race orderings are deterministically resolved: (A) If Start Service obtains the QueueToken lock first, verifies token is CALLED and session is OPEN, and transitions to IN_SERVICE, Session Closure's mandatory post-lock revalidation detects IN_SERVICE and aborts immediately with HTTP 409 ACTIVE_CONSULTATION_IN_PROGRESS; (B) If Session Closure obtains the QueueToken lock first, resolves the token, and closes the session, Start Service subsequently obtains the token lock, revalidates under lock, detects token.status != 'CALLED' and session is CLOSED, and rejects the transition with HTTP 409 QUEUE_SESSION_CLOSED (or TOKEN_NOT_CALLED). A shared QueueSession lock is unnecessary because dual post-lock revalidations on QueueToken and parent session status guarantee mutual exclusion without adding session-level lock contention.

- QueueSession row-level lock (FOR UPDATE) serializes Call Next, Session Closure, and Token Reinstatement.

- Reinstatement and session-closure serialization: Both operations follow the global lock hierarchy Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken. Reinstatement acquires QueueSession FOR UPDATE, re-verifies session status is OPEN, acquires Appointment FOR UPDATE (if linked), and then acquires QueueToken FOR UPDATE. Session closure is authorized by an eligible staff/provider actor and executes in strict lock order: acquires QueueSession FOR UPDATE, verifies status is OPEN or PAUSED and no token is IN_SERVICE, identifies affected records without prematurely acquiring token locks, acquires locks on linked Appointment rows in deterministic primary-key order (ORDER BY id ASC FOR UPDATE), acquires locks on affected QueueToken rows in deterministic primary-key order (ORDER BY id ASC FOR UPDATE), revalidates states under lock (specifically, if ANY affected token has transitioned to IN_SERVICE concurrently, the entire transaction aborts immediately with HTTP 409 and error code ACTIVE_CONSULTATION_IN_PROGRESS, making zero updates, because Start Service locks QueueToken without locking QueueSession and can race with closure), deterministically resolves tokens according to the closure matrix (WAITING to UNSERVED_CLINIC_CLOSED; expired CALLED/SKIPPED to NO_SHOW_UNRESPONSIVE; unexpired to UNSERVED_CLINIC_CLOSED; updating linked appointments atomically to NO_SHOW or CANCELLED; resolving walk-ins standalone and revoking guest secrets), sets QueueSession.status = 'CLOSED', and commits atomically in a single transaction.

- Call-next execution protocol: The application first locks the QueueSession row with FOR UPDATE, verifies no active CALLED or IN_SERVICE token exists, and then selects the next eligible token:
```sql
SELECT id, token_number, priority_next, recalled_at, queue_order_time
FROM queue_tokens
WHERE queue_session_id = :sessionId
  AND status = 'WAITING'
ORDER BY
  priority_next DESC,
  recalled_at ASC NULLS LAST,
  queue_order_time ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```
SKIP LOCKED skips rows that are locked by other concurrent transactions, preventing lock wait contention or deadlocks if another transaction concurrently holds a token lock. It does not affect ordinary non-locking inspection queries. QueueSession FOR UPDATE serves as the primary call-next serializer, and the partial unique index (status IN ('CALLED', 'IN_SERVICE')) guarantees the single active call invariant at the engine level as defense in depth.

- Bounded retry wrapper (up to 4 total attempts = 1 initial + 3 retries) with exponential backoff and jitter for retryable concurrency errors (SQLSTATE 40001, 40P01, and Prisma P2034). Retried transaction blocks must remain completely pure of external side effects.


Do not build a generic transaction framework unless required.


---


# 31. Idempotency


Important operations must be designed to tolerate duplicate requests where appropriate.


This becomes increasingly important when Kafka and at-least-once event delivery are introduced.


Do not build a giant idempotency framework in Phase 1.


Teach the concept through real business operations.


---


# 32. API


Use REST.


Prefix APIs with:


/api/v1/


Controllers remain thin.


Standardize:


- request validation

- response format

- HTTP status codes

- error format

- pagination

- authentication errors

- authorization errors

- validation errors

- business-rule errors


Example error shape:


{

  "error": {

    "code": "APPOINTMENT_ALREADY_CANCELLED",

    "message": "Appointment has already been cancelled"

  }

}


Use OpenAPI/Swagger as the API contract/documentation.


---


# 33. Pagination


Use a simple consistent convention such as:


?page=1&limit=20


Do not introduce an elaborate search/query framework.


---


# 34. Configuration


Use environment variables.


Provide:


.env.example


Never commit real:


- database passwords

- JWT secrets

- API keys

- cloud credentials

- email provider credentials

- Kafka credentials


Configuration should be validated at application startup.


---


# 35. Testing


Backend:


- Vitest for unit tests

- Supertest for API tests

- integration tests against PostgreSQL where appropriate


Frontend:


- Vitest

- React Testing Library


Focus on business behavior rather than maximizing coverage percentage.


Test important edge cases.


Later, an E2E framework may be introduced if justified.


---


# 36. Seed Data


Initial seed data must be:


- synthetic

- fictional

- deterministic/reproducible


Initial seed can include:


- 2 organizations

- 3-4 locations

- 6-8 providers

- 10-15 services

- 20-30 patients

- appointments

- queue sessions

- queue tokens


Later, larger synthetic datasets can be generated for:


- Kafka

- CDC

- analytics

- load testing

- observability

- performance testing


---


# 37. Synthetic Data Policy


Never use:


- real patient data

- real medical records

- real provider data

- real customer data

- confidential organization information

- employer/project information


Healthcare is the demonstration domain only.


QueueLess is NOT a medical-record system.


Out of scope:


- diagnoses

- prescriptions

- lab results

- medical history

- insurance claims

- clinical documentation


---


# 38. Notifications


Phase 6:


- in-app notifications

- email notifications


Email should use a free/low-cost provider available at the time Phase 6 begins.


Do not hard-code a provider decision now.


Create a notification abstraction so provider-specific SDKs do not spread throughout the codebase.


SMS is architecturally anticipated but permanently not implemented because of real per-message cost and lack of a meaningful free tier.


---


# 39. Payments and Billing


Payments and billing are permanently out of scope.


Do not implement:


- payments

- billing

- subscriptions

- invoices

- payment gateways

- monetization


Do not add payment dependencies.


---


# 40. Demo / Synthetic Traffic Simulator


A synthetic traffic simulator is planned for Phase 3-4.


It should simulate realistic QueueLess behavior:


- patient registration

- appointment booking

- walk-in

- check-in

- provider calling next

- service completion

- cancellation


The simulator must interact through the real QueueLess API.


Do NOT directly manipulate PostgreSQL.


The simulator exists to create realistic live behavior for:


- WebSockets

- queue UI

- Kafka

- event processing

- observability demonstrations


Traffic must remain synthetic and controlled.


---


# 41. GitHub Repository


The repository is intentionally public.


Public repository benefits include free/unlimited standard GitHub-hosted Actions runners under the current GitHub policy.


Use GitHub Actions in Phase 8.


Public repository requirements:


- README.md

- LICENSE

- SECURITY.md

- no secrets

- no real data

- no employer/client information


Enable appropriate GitHub security features.


GitHub Actions workflows must not expose credentials.


Use GitHub repository/environment secrets where required.


---


# 42. Docker Sequencing


Docker has two separate purposes.


## Local infrastructure


Kafka and Kafka Connect will run using a small Docker Compose setup when their phases begin.


This is NOT the Phase 7 application-containerization milestone.


## Phase 7


Docker is formally introduced as an application-containerization skill.


Containerize:


- React application

- Node.js API

- workers where applicable

- supporting application components


Do not confuse "using Docker to run Kafka locally" with "learning application containerization."


---


# 43. Phase Progression


## Phase 0

Architecture, domain model, ERD, state machines, API contract, authentication design, planning.


## Phase 1

React + Node.js + PostgreSQL modular monolith.


No Kafka, Redis, WebSockets, AWS, Kubernetes, Terraform, or application containerization.


## Phase 2

Production quality and UX.


- stronger validation

- error handling

- testing

- polished UI

- admin workflow

- authentication completeness

- API documentation


## Phase 3

Redis + WebSockets.


- live queue updates

- caching where justified

- real-time UI

- synthetic traffic simulator introduced


## Phase 4

Kafka.


- Kafka fundamentals

- producers

- consumers

- topics

- partitions

- offsets

- consumer groups

- event design

- idempotency


Kafka runs locally through Docker Compose.


## Phase 5

Kafka Connect + PostgreSQL CDC.


Kafka Connect runs through Docker Compose.


Learn:


- source connectors

- sink connectors

- CDC

- connector configuration

- schemas

- offsets

- failure behavior


## Phase 6

Workers + notifications + retries + DLQ.


- background processing

- email

- in-app notifications

- retry strategies

- dead-letter queues


## Phase 7

Application Dockerization.


- Dockerfiles

- image building

- multi-stage builds

- Compose

- networking

- environment configuration

- container health


## Phase 8

GitHub Actions.


- lint

- test

- build

- CI

- container image workflows later as appropriate


Public GitHub repository is used.


## Phase 9

AWS.


AWS is the primary cloud target.


Prefer local/free/low-cost alternatives when they adequately support the learning objective.


Before provisioning:


- verify current pricing

- verify current free-tier/credit eligibility

- choose smallest practical configuration

- avoid unnecessary always-on resources

- destroy temporary resources

- check associated billable resources


Never run Terraform apply or equivalent cloud provisioning without explicit developer approval.


Do not assume promotional credits.


## Phase 10

Terraform.


Infrastructure as code.


## Phase 11

Observability.


- OpenTelemetry

- Prometheus

- Grafana

- Loki

- metrics

- logs

- traces

- correlation


## Phase 12

Kubernetes.


Core learning happens locally using:


- kind

- k3d

- k3s

- Docker Desktop Kubernetes


Learn:


- Pods

- Deployments

- Services

- ConfigMaps

- Secrets

- probes

- resources

- rolling deployments

- scaling

- autoscaling concepts

- Ingress

- Helm


Managed Kubernetes is optional and not required for Phase 12 completion.


---


# 44. Cloud Cost Policy


Prefer local development and free/low-cost cloud options whenever they adequately support the learning objective.


AWS remains the primary cloud target.


Before creating cloud infrastructure:


- explain what will be created

- explain why it is needed

- verify current pricing

- verify current free-tier/credit implications

- use the smallest practical configuration

- avoid unnecessary always-on resources

- destroy temporary infrastructure after use

- check disks, IPs, load balancers, snapshots, databases and related billable resources


Do not assume promotional credits.


Goal:


Learn AWS without unnecessary recurring cost.


---


# 45. Kubernetes Cost Policy


Local Kubernetes is sufficient for all core Kubernetes learning objectives.


A managed cluster is optional.


If managed Kubernetes is used:


- smallest practical configuration

- verify current pricing

- explicit approval

- temporary usage

- destroy all associated resources afterward


Do not introduce another cloud provider solely to obtain Kubernetes experience.


---


# 46. Public Demo


A temporary public showcase deployment may be created during Phase 2-3 if useful.


It is not part of the formal cloud-learning progression.


AWS remains the formal cloud target for Phase 9.


Any temporary showcase infrastructure must be treated as disposable and cost-controlled.


---


# 47. PWA


PWA is optional.


Core QueueLess is a responsive web application.


Do not introduce PWA infrastructure solely because push notifications may eventually be useful.


Re-evaluate later if it supports a genuine product/learning objective.


---


# 48. Architecture Evolution


QueueLess must evolve as:


Modular Monolith

-> Real-time modular monolith

-> Event-driven architecture

-> Background workers

-> Containerized application

-> Cloud deployment

-> Infrastructure as Code

-> Observability

-> Kubernetes


Do NOT begin with microservices.


Do NOT introduce:


- microservices

- CQRS

- event sourcing

- service mesh

- complex API gateway

- enterprise service bus

- excessive design patterns


unless a later phase has a clear learning requirement.


---


# 49. Architecture Decision Records


Maintain lightweight ADRs in:


docs/decisions/


Examples:


- ADR-001 Modular Monolith

- ADR-002 PostgreSQL + Prisma

- ADR-003 React from Day 1

- ADR-004 JWT Authentication

- ADR-005 Appointment / QueueToken Separation

- ADR-006 Provider-Specific Queues

- ADR-007 Kafka Introduction

- ADR-008 Kafka Connect CDC

- ADR-009 Local Kubernetes First


ADRs should explain:


- context

- decision

- alternatives considered

- consequences


---


# 50. Definition of Done


A feature is not complete merely because it compiles.


Definition of Done:


Requirement

-> Design

-> Implementation

-> Validation

-> Tests

-> Edge cases

-> Manual verification

-> Documentation

-> Code review

-> Git commit


AI-generated code must be reviewed and understood.


---


# 51. Git Discipline


Use meaningful commits.


Examples:


feat(auth): add user registration

feat(queue): implement provider queue session

fix(appointments): prevent overlapping bookings

test(queue): add concurrent check-in coverage

docs(architecture): add queue state machine


Do not create giant commits containing unrelated generated changes.


---


# 52. Dependency Discipline


Before adding a dependency, explain:


- why it is needed

- what problem it solves

- why existing dependencies are insufficient

- whether it is appropriate for the current phase


Do not add dependencies simply because an AI-generated example used them.


---


# 53. Current Technology Restrictions


Until their designated phase, do not introduce:


Redis

WebSockets

Kafka

Kafka Connect

AWS

Terraform

Kubernetes

Helm

OpenTelemetry

Prometheus

Grafana

Loki

application Dockerization

microservices


Kafka and Kafka Connect are exceptions only in the sense that their local infrastructure will run using Docker Compose during Phases 4-5.


---


# 54. Permanent Scope Exclusions


The following are permanently outside the QueueLess project:


- payments

- billing

- subscriptions

- monetization

- medical records

- clinical systems

- real patient data

- real hospital/customer data

- SMS implementation

- enterprise SSO

- complex RBAC

- advanced multi-tenant infrastructure


---


# 55. Primary Success Criterion


The project is successful if the developer can explain:


1\. Why the domain is modeled this way.

2\. Why PostgreSQL is used.

3\. Why the application starts as a modular monolith.

4\. Why Redis is introduced later.

5\. Why WebSockets are introduced later.

6\. Why Kafka is introduced only after the synchronous workflow works.

7\. Why Kafka Connect/CDC is useful.

8\. How idempotency and retries work.

9\. Why Docker is introduced at its designated phase.

10\. How CI/CD works.

11\. How AWS infrastructure is designed.

12\. Why Terraform is useful.

13\. How observability works.

14\. Why Kubernetes is introduced last.


The project must demonstrate engineering reasoning, not merely technology accumulation.
