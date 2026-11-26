# Phase 0 Architecture Specification - QueueLess

**Status:** Approved by Project Owner (Phase 0 Complete).

---

## 1. Goals and Non-Goals

### Goals
- Design a portfolio-grade full-stack appointment and live-queue management platform.
- Demonstrate deep backend Node.js engineering, TypeScript, PostgreSQL relational modeling, concurrency control, and clean architecture.
- Establish robust domain invariants at both application logic and PostgreSQL constraint levels.
- Design a clean modular monolith in Phase 1 that prepares for event-driven architecture (Kafka/CDC) and cloud deployment in later phases.
- Build a responsive, accessible React frontend from Day 1 to provide a demonstrable product layer.

### Non-Goals (Strict Phase & Project Boundaries)
- **Clinical Records**: Not a medical record or Electronic Health Record (EHR) system. No diagnoses, prescriptions, lab results, or clinical documentation.
- **Payments & Billing**: Permanently out of scope. No payments, invoicing, subscriptions, or monetization.
- **SMS Gateways**: Not implemented due to per-message carrier costs and lack of durable free tiers.
- **Microservices from Day 1**: No microservices, service meshes, or distributed transactions in Phase 1.
- **Premature Infrastructure**: No Redis, WebSockets, Kafka, Docker, Kubernetes, or cloud resources in Phase 1.

---

## 2. System Context and Actors

| Actor | Access Mode | Primary Responsibilities |
| :--- | :--- | :--- |
| **Patient (Authenticated)** | JWT Session | Registers universally; books appointments across clinics; checks in $\le 15$ min prior; tracks queue position via private patient dashboard. |
| **Guest Walk-In (Unauthenticated)** | Header Auth (`X-Queue-Tracking-Token`) | Joins live queue at kiosk/desk; receives token number and 256-bit CSPRNG tracking secret; tracks queue position via PII-free mobile tracking endpoint. |
| **Provider (Clinician)** | Scoped Staff (`PROVIDER`) | Manages daily shift queue; calls next patient; starts consultations; skips absent patients; recalls eligible patients; completes consultations. |
| **Organization Administrator** | Scoped Staff (`ORG_ADMIN`) | Configures locations, services, and provider profiles; schedules queue shifts; voids pre-operational accidental shifts; executes administrative token reinstatements. |

---

## 3. Container / Component Architecture (Modular Monolith)

Phase 1 operates as a single deployable modular monolith:

```
+------------------------------------------------------------------------+
|                        QueueLess Web (apps/web)                        |
|   React, TypeScript, Vite, Tailwind CSS, shadcn/ui, TanStack Query     |
+-----------------------------------+------------------------------------+
                                    | HTTP / REST (/api/v1/)
                                    v
+------------------------------------------------------------------------+
|                        QueueLess API (apps/api)                        |
|   Node.js, Express, TypeScript, Zod Schema Validation, Winston/Pino    |
|                                                                        |
|   +----------------------------------------------------------------+   |
|   | Modules: Auth, Users, Orgs, Locations, Providers, Services,    |   |
|   |          Appointments, Queue, AuditLog                         |   |
|   +-------------------------------+--------------------------------+   |
|                                   |                                    |
|   +-------------------------------v--------------------------------+   |
|   | Persistence Layer: Repositories & Prisma ORM Client            |   |
|   +-------------------------------+--------------------------------+   |
+-----------------------------------+------------------------------------+
                                    | SQL (TCP 5432)
                                    v
+------------------------------------------------------------------------+
|                  PostgreSQL Database Engine (v16+)                     |
|   - Relational Tables & B-Tree Indexes                                 |
|   - PostgreSQL btree_gist Extension & Range Exclusion Constraints      |
|   - Partial Unique Indexes (single active call per session)            |
|   - Composite Foreign Keys & Restrictive Cascades                      |
|   - Immutable Append-Only Audit Trigger (trg_audit_events_immutable)   |
+------------------------------------------------------------------------+
```

---

## 4. Monorepo and Module Boundaries

### Monorepo Layout (`pnpm` Workspaces)
```
queueless/
|-- apps/
|   |-- web/          # React SPA
|   \-- api/          # Express Modular Monolith API
|-- packages/
|   |-- shared/       # Shared types, Zod schemas, error codes, constants
|   \-- config/       # Shared TypeScript, ESLint, Prettier configs
|-- infrastructure/   # Docker, Terraform, Kubernetes (Phase-gated)
\-- docs/             # Canonical project architecture, domain specs, ADRs
```

### Backend Internal Architecture (`apps/api/src/`)
```
src/
|-- config/           # Environment variables, startup validation
|-- middleware/       # Auth JWT, error handler, rate limiters, tracking header
|-- modules/          # Vertical domain slices (Controller -> Service -> Repository)
|   |-- auth/
|   |-- users/
|   |-- organizations/
|   |-- locations/
|   |-- providers/
|   |-- services/
|   |-- appointments/
|   |-- queue/
|   \-- audit/
|-- database/         # Prisma schema, custom raw migrations, seed data
|-- shared/           # Error classes, logger, types, utils
|-- app.ts            # Express application configuration
\-- server.ts         # HTTP server entrypoint
```
**Request Flow Rule**:
`Route -> Middleware -> Controller (thin) -> Service (domain rules) -> Repository -> Prisma/PostgreSQL`.

---

## 5. Domain Model and Relational Integrity

Refer to [docs/domain/domain-model.md](../domain/domain-model.md) for full entity field listings and Mermaid ERD.

### Key Database Integrity Constraints
1. **Provider Membership Integrity**: `Provider` includes fixed discriminator column `CHECK (role = 'PROVIDER')` and a composite foreign key referencing `OrganizationMembership(userId, organizationId, role) ON DELETE RESTRICT`.
2. **Provider Shift Overlap Prevention**: `QueueSession` enforces a PostgreSQL `btree_gist` range exclusion on `(provider_id WITH =, tstzrange(start_time, end_time, '[)') WITH &&) WHERE (status != 'VOIDED')`.
3. **Session Timing Validity**: `CHECK (start_time < end_time)`.
4. **Single Active Call Invariant**: `CREATE UNIQUE INDEX single_active_call_per_session ON queue_tokens(queue_session_id) WHERE status IN ('CALLED', 'IN_SERVICE')`.
5. **Token Multiplicity 1:1 Invariant**: `@@unique([appointmentId])` on `QueueToken`.
6. **Immutable Audit Ledger**: `trg_audit_events_immutable` PostgreSQL trigger blocking any `UPDATE` or `DELETE` on `audit_events` via SQLSTATE `P0001` (`raise_exception`). Triggers protect against application logic bugs, but do not prevent alterations by database superusers or database owners.

---

## 6. Appointment Lifecycle

### State Machine
`BOOKED -> CHECKED_IN -> COMPLETED | CANCELLED | NO_SHOW`.

### Rules
- **Advance Cancellation**: Patient or staff cancels before check-in (`BOOKED -> CANCELLED`). Slot is freed; no token exists.
- **Check-In Window**: Patient checks in within the window `[scheduledStartTime - 15 minutes, scheduledStartTime + 15 minutes]` (`APPOINTMENT_CHECKIN_GRACE_MINUTES = 15`). Transitions `BOOKED -> CHECKED_IN` and transactionally creates `QueueToken` in `WAITING`. The check-in window does not extend to `scheduledEndTime`.
- **Pre-Check-In Forfeit**: If 15 minutes pass after scheduled start time without check-in (`NOW() > scheduledStartTime + 15 minutes`), appointment may transition `BOOKED -> NO_SHOW`.
- **Consultation Completion**: Synchronized atomically when active token transitions `IN_SERVICE -> COMPLETED`.
- **Aborted Consultation**: If a consultation is aborted mid-service, appointment transitions `CHECKED_IN -> CANCELLED` with reason `CONSULTATION_ABORTED`.
- **Unresponsive Resolution**: If a checked-in patient fails to enter when called or fails to return after being skipped, linked appointment transitions `CHECKED_IN -> NO_SHOW`.

---

## 7. QueueSession and QueueToken Lifecycle

### QueueSession Lifecycle
`SCHEDULED -> OPEN <-> PAUSED -> CLOSED`  
`SCHEDULED -> VOIDED`

- **`SCHEDULED`**: Shift is planned on calendar. No tokens exist; doors not open.
- **`OPEN`**: Shift active. Check-ins, walk-ins, and patient calls permitted.
- **`PAUSED`**: Calling suspended; tokens preserved; queue visible to patients as paused.
- **`CLOSED`**: Shift ended. Unresolved tokens resolved. Blocked if any token is `IN_SERVICE`.
- **`VOIDED`**: Accidental session eliminated before opening. Permitted strictly for `ORG_ADMIN`, strictly while `SCHEDULED`, and strictly when zero tokens were created (`COUNT = 0`).

### QueueToken Lifecycle
`WAITING -> CALLED -> IN_SERVICE -> COMPLETED`  
`CALLED -> SKIPPED -> WAITING (Recall)`  
`CALLED / SKIPPED / WAITING -> CANCELLED`  
`CANCELLED -> WAITING (Reinstatement)`

- **`CALLED` Response Window**: 5 minutes (`CALLED_RESPONSE_WINDOW_MINUTES = 5`). Provider must wait the full 5 minutes before skipping.
- **`SKIPPED` Return Window**: 30 minutes (`SKIPPED_RETURN_WINDOW_MINUTES = 30`). One-time priority recall (`recallCount <= 1`) is allowed through `NOW() <= skippedAt + 30 minutes` while the session is `OPEN`. Transition to `NO_SHOW_UNRESPONSIVE` is allowed only when `NOW() > skippedAt + 30 minutes`. If the session closes while `NOW() <= skippedAt + 30 minutes`, the token defaults to `UNSERVED_CLINIC_CLOSED`.
- **`WAITING` Protection**: Waiting tokens can never be marked no-show (`NO_SHOW_UNRESPONSIVE` is strictly forbidden).
- **Reinstatement Rules**: Permitted at most twice (`reinstatementCount <= 2`) for `ACCIDENTAL_CANCELLATION` and `PATIENT_DEPARTED` while session is `OPEN`. Reinstated tokens rejoin at the tail of the ordinary queue (`queue_order_time = NOW()`). `NO_SHOW_UNRESPONSIVE` is permanently ineligible.

---

## 8. Authentication and Authorization

### User & Identity Model
- Universal `User` identity: all registered users possess patient access globally without tenant boundaries.
- Organization-scoped authorization: `OrganizationMembership` grants `ORG_ADMIN` or `PROVIDER` roles for a specific clinic.
- Dual memberships permitted: a user may hold both `ORG_ADMIN` and `PROVIDER` in the same organization.

### Token Strategy (ADR-004 Accepted - Option 1 Selected)
- **Access Tokens**: Short-lived JWT (15-minute expiration) containing `userId`, `email`, and active membership roles. Held in client memory and passed via `Authorization: Bearer <jwt_access_token>`.
- **Refresh Tokens**: Long-lived token (7-day expiration) stored in database with rotation and revocation; transmitted via `HttpOnly`, `Secure` cookie with `Path=/api/v1/auth`.
- **Client Storage Choice**: Accepted on 2026-11-04 in ADR-004 selecting Option 1 (In-Memory Access Token + HttpOnly Refresh Cookie) with mandatory Origin allowlisting, SameSite defense in depth, and anti-CSRF verification.

---

## 9. API Conventions & Error Mapping

- Base path `/api/v1/`. REST endpoints returning RFC 7807 aligned JSON errors.
- Comprehensive SQLSTATE error mapping intercepting PostgreSQL codes:
  - `23505`: Unique constraint violation (`DUPLICATE_CHECK_IN`, `ACTIVE_CALL_IN_PROGRESS`).
  - `23P01`: Range exclusion violation (`PROVIDER_SESSION_OVERLAP`).
  - `23503`: Foreign key violation (`INVALID_PROVIDER_MEMBERSHIP`).
  - `23514`: Check constraint violation (`INVALID_SESSION_TIMES`).
  - `P0001` (with message contract): PostgreSQL trigger exception (`trg_audit_events_immutable`, matching message `'Audit records are immutable...'` -> `AUDIT_LOG_IMMUTABLE`). Unmatched `P0001` errors map to `INTERNAL_SERVER_ERROR`.
- Detailed conventions and endpoint catalog documented in [docs/api/api-conventions.md](../api/api-conventions.md).

---

## 10. Consistency, Concurrency, and Locking Architecture

### 10.1 Global Lock Acquisition Hierarchy
All multi-table transactions must acquire row locks in this exact hierarchy:
$$\text{Organization} \longrightarrow \text{Provider} \longrightarrow \text{OrganizationMembership} \longrightarrow \text{QueueSession} \longrightarrow \text{Appointment} \longrightarrow \text{QueueToken}$$
Whenever multiple rows of the same entity are locked within a transaction, they must be acquired in deterministic primary-key order (`ORDER BY id ASC`).

### 10.2 Provider Coordination Locking
- **Deactivation**: Acquires exclusive row lock `SELECT ... FROM providers WHERE id = $id FOR UPDATE`. Deactivates provider and membership atomically; logs audit event.
- **Booking & Session Creation**: Acquires shared lock `SELECT ... FROM providers WHERE id = $id FOR SHARE`. Waits if deactivation is in-flight; aborts with `PROVIDER_INACTIVE` if `is_active === false`.

### 10.3 QueueSession Call-Next Serialization
- Call Next executes within an interactive transaction following the global lock hierarchy:
  1. The application first locks the parent `QueueSession` row with `FOR UPDATE`:
     ```sql
     SELECT id, status FROM queue_sessions WHERE id = :sessionId FOR UPDATE;
     ```
  2. Verifies that `QueueSession.status === 'OPEN'`.
  3. Verifies the single active call invariant: ensures zero tokens currently exist with `status IN ('CALLED', 'IN_SERVICE')`. If a patient is already called or in consultation, aborts with `ACTIVE_CALL_IN_PROGRESS`.
  4. Selects the next eligible token using deterministic hybrid FIFO ordering and locks it:
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
  5. Consumes recall priority (`priority_next = FALSE`), sets `status = 'CALLED'`, `called_at = NOW()`, and records `TOKEN_CALLED` in `audit_events`.
- **Primary Serialization vs. Defense in Depth**: `SKIP LOCKED` is **not** the primary serialization mechanism when calls already serialize sequentially on the `QueueSession` row lock. Rather, `QueueSession FOR UPDATE` serves as the primary call-next serializer. `SKIP LOCKED` skips rows that are locked by other concurrent transactions, preventing lock wait contention or deadlocks if another transaction concurrently holds a lock on a token. It does not affect ordinary non-locking inspection queries. In addition, the partial unique index `single_active_call_per_session` acts as engine-level defense in depth.

### 10.4 Reinstatement and Session-Closure Concurrency Protocol
Both token reinstatement and session closure serialize on the parent `QueueSession` row to prevent race conditions:
1. **Locking Order**: Must strictly respect the global hierarchy: `QueueSession -> Appointment -> QueueToken`.
2. **Reinstatement Protocol**:
   - Acquires `SELECT id, status FROM queue_sessions WHERE id = :sessionId FOR UPDATE`.
   - **Re-checks Session Status**: The application must re-verify that `QueueSession.status === 'OPEN'` after acquiring the lock. If the session was closed or paused concurrently, the operation is immediately rejected (`QUEUE_SESSION_NOT_OPEN`).
   - If linked to an appointment, acquires `SELECT id, status FROM appointments WHERE id = :appointmentId FOR UPDATE` first.
   - Acquires `SELECT * FROM queue_tokens WHERE id = :tokenId FOR UPDATE`.
   - Validates guards: `status === 'CANCELLED'`, `cancellationReason IN ('ACCIDENTAL_CANCELLATION', 'PATIENT_DEPARTED')`, `reinstatementCount < 2`.
   - Restores token to `WAITING` at queue tail (`queue_order_time = NOW()`), increments `reinstatementCount`, synchronizes linked appointment back to `CHECKED_IN`, and records `TOKEN_REINSTATED`.
3. **Session Closure Protocol**:
   - Session closure is authorized by an eligible staff or provider actor (`POST /api/v1/queue-sessions/:id/close`).
   - Follows the global lock hierarchy (`QueueSession -> Appointment -> QueueToken`) and executes in the following sequence:
     1. **Acquire Session Lock**: Acquires `SELECT id, status FROM queue_sessions WHERE id = :sessionId FOR UPDATE`.
     2. **Pre-check Status & Active Consultations**: Verifies session is `OPEN` or `PAUSED`. If already `CLOSED`, returns idempotent success. Verifies zero tokens are `IN_SERVICE`. If a consultation is actively in progress, closure is blocked (`409 Conflict`, `ACTIVE_CONSULTATION_IN_PROGRESS`).
     3. **Identify Affected Records**: Identifies all affected unresolved tokens (`status IN ('WAITING', 'CALLED', 'SKIPPED')`) and their distinct linked `appointment_id` values (where not null) via an inspection query without prematurely acquiring token row locks.
     4. **Lock Appointments in Deterministic PK Order**: If any affected tokens are linked to appointments, acquires row locks on those appointment rows in deterministic primary-key order:
        ```sql
        SELECT id, status FROM appointments
        WHERE id IN (:appointmentIds)
        ORDER BY id ASC
        FOR UPDATE;
        ```
     5. **Lock QueueTokens in Deterministic PK Order**: Acquires row locks on all affected token rows in deterministic primary-key order:
        ```sql
        SELECT id, status, appointment_id, called_at, skipped_at FROM queue_tokens
        WHERE id IN (:tokenIds)
        ORDER BY id ASC
        FOR UPDATE;
        ```
     6. **Mandatory Post-Lock Token State Revalidation**: Revalidates token and appointment states after acquiring locks. If ANY affected token is now `IN_SERVICE`, the entire transaction must immediately abort with HTTP 409 and error code `ACTIVE_CONSULTATION_IN_PROGRESS`. The transaction makes no appointment, token, or session updates in this case. This second check is mandatory because Start Service locks `QueueToken` without locking `QueueSession` and can race with closure.
     7. **Deterministic Resolution According to Closure Matrix** (staff cannot arbitrarily choose a no-show outcome):
        - `WAITING` tokens deterministically resolve to `CANCELLED` with reason `UNSERVED_CLINIC_CLOSED`.
        - `CALLED` tokens become `NO_SHOW_UNRESPONSIVE` only after the 5-minute response window (`NOW() >= calledAt + 5 min`).
        - `SKIPPED` tokens become `NO_SHOW_UNRESPONSIVE` only strictly after the 30-minute return window (`NOW() > skippedAt + 30 min`).
        - Unexpired `CALLED` and `SKIPPED` tokens default to `UNSERVED_CLINIC_CLOSED`.
        - **Distinguishes Appointment vs. Walk-In Outcomes**: For appointment-linked tokens (`appointmentId != NULL`), atomically synchronizes `Appointment.status` in the same transaction to `NO_SHOW` (if reason is `NO_SHOW_UNRESPONSIVE`) or `CANCELLED` (if reason is `UNSERVED_CLINIC_CLOSED`). For walk-in tokens (`appointmentId == NULL`), resolves independently to `CANCELLED` with the designated reason and permanently revokes any guest tracking secret.
     8. **Set Session Closed**: Sets `QueueSession.status = 'CLOSED'` and `closed_at = NOW()`.
     9. **Atomic Commit**: Commits the entire operation atomically in one transaction, ensuring no token can be reinstated, checked in, or called concurrently during closure.

4. **Start Service Concurrency Protocol and Dual Race Handling with Session Closure**:
   - `Start Service` is invoked by a provider (`POST /api/v1/queue-tokens/:id/start-service`) to begin consultation on a `CALLED` patient.
   - **Lock Acquisition**: Start Service acquires a row lock on the target token:
     ```sql
     SELECT id, status, queue_session_id FROM queue_tokens WHERE id = :id FOR UPDATE;
     ```
   - **Mandatory Post-Lock Revalidation**: After acquiring the `QueueToken FOR UPDATE` lock, Start Service revalidates both the token state and the parent `QueueSession` state before transitioning to `IN_SERVICE`:
     - If `token.status !== 'CALLED'`, the transition is invalid: Start Service aborts with `409 Conflict` (error code `INVALID_TOKEN_STATUS` or `TOKEN_NOT_CALLED`).
     - If the parent `QueueSession.status !== 'OPEN'`, the session is no longer operational: Start Service aborts with `409 Conflict` and error code `QUEUE_SESSION_CLOSED` (or `QUEUE_SESSION_PAUSED` if paused).
     - Only if `token.status === 'CALLED'` AND parent `QueueSession.status === 'OPEN'` does Start Service transition `token.status = 'IN_SERVICE'`, set `service_started_at = NOW()`, and commit.
   - **Dual Race Orderings with Session Closure**:
     Because Start Service locks `QueueToken` without acquiring `QueueSession FOR UPDATE`, two distinct race orderings can occur when Start Service and Session Closure execute concurrently:
     - **Race Ordering A (Start Service obtains `QueueToken` lock first)**:
       1. Start Service acquires `QueueToken FOR UPDATE` on the `CALLED` token, confirms `token.status === 'CALLED'` and parent `QueueSession.status === 'OPEN'`, transitions the token to `IN_SERVICE`, and commits.
       2. Session Closure (which holds `QueueSession FOR UPDATE`) subsequently acquires `QueueToken FOR UPDATE` in Step 5.
       3. In Step 6 (Mandatory Post-Lock Token State Revalidation), Session Closure inspects the locked token, detects that it is now `IN_SERVICE`, and **immediately aborts and rolls back completely** with `409 Conflict` and error code `ACTIVE_CONSULTATION_IN_PROGRESS`, making zero appointment, token, or session updates.
     - **Race Ordering B (Session Closure obtains `QueueToken` lock first)**:
       1. Session Closure holds `QueueSession FOR UPDATE`, verifies zero tokens are `IN_SERVICE` in Step 2, and acquires `QueueToken FOR UPDATE` on all affected tokens (including the `CALLED` token) in Step 5.
       2. In Step 6, Session Closure confirms the token is still `CALLED` (not `IN_SERVICE`).
       3. In Step 7, Session Closure deterministically resolves the token according to the closure matrix (to `CANCELLED` with reason `NO_SHOW_UNRESPONSIVE` if the 5-minute response window expired, or `UNSERVED_CLINIC_CLOSED` if unexpired), updates the linked appointment atomically, sets `QueueSession.status = 'CLOSED'` in Step 8, and commits in Step 9.
       4. Start Service (which was blocked waiting for `QueueToken FOR UPDATE`) now acquires the token lock.
       5. Start Service executes its mandatory post-lock revalidation: it inspects the locked token and discovers `token.status !== 'CALLED'` (now `CANCELLED`) and parent `QueueSession.status === 'CLOSED'`.
       6. Start Service **immediately rejects the transition** with `409 Conflict` and error code `QUEUE_SESSION_CLOSED` (or `TOKEN_NOT_CALLED`), aborting with zero updates and preserving the closed session state.
   - **Locking Evaluation (Why Shared `QueueSession` Lock Is NOT Recommended)**:
     If Start Service were instead to acquire `QueueSession FOR SHARE`, it would block Session Closure (`QueueSession FOR UPDATE`) until Start Service completes. However, adding `QueueSession FOR SHARE` introduces an unnecessary lock acquisition on the session row for every consultation start. Mutual exclusion and state integrity across both race orderings are already fully guaranteed by the post-lock revalidations on `QueueToken FOR UPDATE` and parent session state, making a shared session lock redundant overhead.


### 10.5 Multi-Table Transaction Locking Consistency Matrix
To avoid lock-order inversion across concurrent queue lifecycle operations, transactions acquire row-level locks in accordance with the global lock hierarchy:
`Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken`
Multiple rows of the same entity are always locked in deterministic primary-key order (`ORDER BY id ASC`).

| Operation | Entity Rows Locked | Lock Acquisition Sequence | Hierarchy Conformance | Safety Rationale If `QueueSession` Is NOT Locked |
| :--- | :--- | :--- | :--- | :--- |
| **Call Next** | `QueueSession`, `QueueToken` | `QueueSession FOR UPDATE` -> `QueueToken FOR UPDATE SKIP LOCKED` | `QueueSession -> QueueToken` | Locks `QueueSession` to serialize candidate selection across concurrent staff/providers. |
| **Start Service** | `QueueToken` | `QueueToken FOR UPDATE` | `QueueToken` | **Safe without `QueueSession` lock via dual post-lock revalidation**: Only 1 active call exists per session, guaranteed by the partial unique index (`status IN ('CALLED', 'IN_SERVICE')`). Both race orderings with Session Closure are deterministically handled: in Race Ordering A, Start Service locks `QueueToken` first and transitions to `IN_SERVICE`, causing Session Closure's post-lock check to abort with `409 Conflict` (`ACTIVE_CONSULTATION_IN_PROGRESS`); in Race Ordering B, Session Closure locks `QueueToken` first, resolves the token, and closes the session, causing Start Service's post-lock check on `QueueToken` to detect `token.status !== 'CALLED'` and parent `QueueSession.status === 'CLOSED'`, rejecting the transition with `409 Conflict` (`QUEUE_SESSION_CLOSED`). |
| **Skip Patient** | `QueueToken` | `QueueToken FOR UPDATE` | `QueueToken` | **Safe without `QueueSession` lock**: Operates on an already `CALLED` token whose 5-minute response window has elapsed. Linked appointment remains `CHECKED_IN`. Vacates the active slot, allowing the provider to call the next patient. |
| **Recall Patient** | `QueueSession`, `QueueToken` | `QueueSession FOR UPDATE` -> `QueueToken FOR UPDATE` | `QueueSession -> QueueToken` | Locks `QueueSession` to coordinate queue re-entry with `priorityNext = TRUE` and re-verify session status is `OPEN`. |
| **Complete Service** | `Appointment` (if linked), `QueueToken` | `Appointment FOR UPDATE` -> `QueueToken FOR UPDATE` | `Appointment -> QueueToken` | **Safe without `QueueSession` lock**: Consultation is already `IN_SERVICE`. Session closure is strictly blocked while any token is `IN_SERVICE` (`409 Conflict`). Completing the consultation transitions token to `COMPLETED` and vacates the active call slot. Lock order (`Appointment -> QueueToken`) matches global hierarchy. |
| **Advance Cancellation** (before check-in) | `Appointment` | `Appointment FOR UPDATE` | `Appointment` | **Safe without `QueueSession` lock**: Token does not exist yet (`status = BOOKED`). Operates purely on the appointment schedule. |
| **Token Cancellation** (patient departs waiting room) | `Appointment` (if linked), `QueueToken` | `Appointment FOR UPDATE` -> `QueueToken FOR UPDATE` | `Appointment -> QueueToken` | **Safe without `QueueSession` lock**: Affects only this patient's queue token and linked appointment. Order is `Appointment -> QueueToken`. Because Session Closure ALSO locks `Appointment` before `QueueToken`, their lock acquisition order is identical (`Appointment -> QueueToken`), avoiding lock inversion between these operations. |
| **Reinstatement** | `QueueSession`, `Appointment` (if linked), `QueueToken` | `QueueSession FOR UPDATE` -> `Appointment FOR UPDATE` -> `QueueToken FOR UPDATE` | `QueueSession -> Appointment -> QueueToken` | Locks `QueueSession` to re-verify session is `OPEN` and serialize queue tail re-entry. Locks linked appointment before token. |
| **Session Closure** | `QueueSession`, `Appointment` (all linked, in PK order), `QueueToken` (all unresolved, in PK order) | `QueueSession FOR UPDATE` -> `Appointment FOR UPDATE` (PK order) -> `QueueToken FOR UPDATE` (PK order) | `QueueSession -> Appointment -> QueueToken` | Locks `QueueSession` to serialize closure. Gathers records without premature locks. Locks all linked `Appointment` rows in PK order, then all affected `QueueToken` rows in PK order. Mandatory post-lock revalidation aborts with `409 Conflict` (`ACTIVE_CONSULTATION_IN_PROGRESS`) without updates if any token transitioned to `IN_SERVICE` concurrently (Race Ordering A). In Race Ordering B, closure commits first and Start Service's post-lock check rejects on closed session. |

**Deadlock Avoidance in Queue Lifecycle Paths**: The analyzed queue lifecycle paths follow their documented lock acquisition order and deterministic primary-key ordering, subject to the stated transaction assumptions. This is not a proof of application-wide deadlock freedom.

### 10.6 Dynamic Wait-Time Estimation Architecture
Wait time is calculated dynamically per-token using each token's snapshot service duration (`durationMinutesSnapshot`), providing accurate estimates for mixed-service queues (e.g. General Consultation = 30 min, Follow-up = 15 min, Specialist = 45 min):

$$\text{Estimated Wait (Minutes)} = \text{Remaining Active Estimate} + \sum_{T \in S_{\text{ahead}}} T.\text{durationMinutesSnapshot}$$

1. **Tokens Ahead ($S_{\text{ahead}}$)**:
   All `WAITING` tokens in the same `QueueSession` that precede the target token under the hybrid FIFO ordering:
   - Priority recalled tokens (`priorityNext = TRUE`) ordered by `recalledAt ASC NULLS LAST`.
   - Ordinary waiting tokens (`priorityNext = FALSE`) ordered by `queueOrderTime ASC`.
2. **Active Consultation Approximation & Known Limitations**:
   If a token in the session is currently active:
   - **`IN_SERVICE`**: Elapsed consultation time starts at `serviceStartedAt`. Let $\text{elapsedMinutes} = \lfloor(\text{NOW} - \text{serviceStartedAt}) / 60\text{s}\rfloor$. The remaining active estimate is:
     $$\text{Remaining Active Estimate} = \max\left(0, \text{activeToken.durationMinutesSnapshot} - \text{elapsedMinutes}\right)$$
     If the consultation overruns scheduled duration ($\text{elapsedMinutes} > \text{durationMinutesSnapshot}$), the remaining estimate clamps to 0 minutes, acknowledging that overrunning consultations cannot be predicted with certainty without clinical telemetry.
   - **`CALLED`**: The consultation may not have started yet (patient is responding to the call or walking into the consultation room). Elapsed time since `calledAt` is NOT subtracted as though consultation had begun. Instead, $\text{Remaining Active Estimate} = \text{activeToken.durationMinutesSnapshot}$. The estimate is less certain while a token is `CALLED` and may be recalculated when service starts (`IN_SERVICE`).
   - If no token is `CALLED` or `IN_SERVICE`, $\text{Remaining Active Estimate} = 0$.
3. **Operational Presentation**:
   - The result is explicitly presented as a dynamic **estimate**, never as a guaranteed consultation or appointment time.

### 10.7 Bounded Concurrency Retry Wrapper
Interactive transactions wrapped with bounded exponential backoff retries (up to 4 total attempts: 1 initial + 3 retries) for retryable concurrency errors (`40001`, `40P01`, and Prisma `P2034`):
```typescript
async function runWithRetry<T>(
  operation: (tx: Prisma.TransactionClient) => Promise<T>,
  maxRetries = 3,
  initialDelayMs = 50
): Promise<T> {
  let attempt = 0;
  while (true) {
    try {
      return await prisma.$transaction(operation, { isolationLevel: 'Serializable' });
    } catch (error: any) {
      const isRetryable = extractSqlState(error) === '40001' || 
                          extractSqlState(error) === '40P01' || 
                          error?.code === 'P2034';
      if (isRetryable && attempt < maxRetries) {
        attempt++;
        const jitter = Math.random() * 25;
        const delay = initialDelayMs * Math.pow(2, attempt - 1) + jitter;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}
```
**Transaction Purity**: Transaction blocks passed to `runWithRetry` are completely pure of external side effects.

---

## 11. Testing Strategy

- **Unit Testing (Vitest)**: Pure domain functions, state transition guards, wait-time estimation formulas.
- **Integration Testing (Supertest + Real PostgreSQL)**:
  - Multi-client concurrent check-ins on the same appointment asserting exactly 1 succeeds and duplicates fail with `409 Conflict`.
  - Concurrent Call Next calls asserting single active call enforcement.
  - Concurrent provider deactivation vs. booking asserting no bookings commit after deactivation.
  - Partial GiST exclusion testing asserting shift overlap prevention and voided vacancy.
- **Frontend Testing (Vitest + React Testing Library)**: UI state rendering (loading, empty, error), form validation, and accessible interaction patterns.

---

## 12. Security, Privacy, and Layered Audit Immutability

### Guest Walk-In Privacy & Secret Hygiene
- 256-bit CSPRNG tracking secret generated at creation; stored at rest strictly as SHA-256 hash.
- Header-based authentication (`X-Queue-Tracking-Token`) only. Query parameters prohibited.
- Application loggers explicitly redact `X-Queue-Tracking-Token`.
- Responses to tracking requests return strictly operational metrics; zero PII.
- Permanent secret revocation upon terminal states (`COMPLETED`, `CANCELLED`). Fresh secret issued if reinstated.

### Layered Audit Immutability Architecture
1. **Layer 1 (Least Privilege)**: Application role (`queueless_app`) granted `SELECT, INSERT` only. `REVOKE UPDATE, DELETE, TRUNCATE ON audit_events FROM queueless_app`.
2. **Layer 2 (PostgreSQL Trigger)**: `trg_audit_events_immutable` raises an exception with stable message `'Audit records are immutable and cannot be modified or deleted'` (SQLSTATE `P0001`, PostgreSQL default `raise_exception`) on any `UPDATE` or `DELETE`. The application maps to `AUDIT_LOG_IMMUTABLE` only when both SQLSTATE `P0001` and this message contract match, avoiding false-positive mapping of generic `P0001` errors.
3. **Layer 3 (Relational Retention)**: `organization_id` on `audit_events` uses `ON DELETE RESTRICT`.
4. **Honest Boundary**: Relational triggers prevent application logic bugs and operational mistakes. However, database triggers do not protect against database owners or superusers (who can disable triggers via `session_replication_role = replica` or `ALTER TABLE ... DISABLE TRIGGER`). Protection against superusers requires external infrastructure controls (e.g. S3 Object Lock WORM storage or remote append-only log streaming).

---

## 13. Event-Driven Evolution: Decoupling AuditEvent from OutboxEvent

- **Phase 1-3**: `audit_events` operates purely as an immutable business audit ledger. No outbox tables or Kafka dependencies are introduced.
- **Phase 4 (Kafka Introduction)**: Introduce a dedicated `outbox_events` table for asynchronous message relay. Domain mutations write to business tables, `audit_events`, and `outbox_events` in the same transaction.
- **Phase 5 (Kafka Connect CDC)**: Debezium captures inserts directly from PostgreSQL WAL logs, streaming events into Kafka topics reliably without dual-write hazards.

---

## 14. Architecture Decision Records (ADR) Index

| ADR ID | Title | Status | Primary Decision |
| :--- | :--- | :--- | :--- |
| **ADR-001** | Modular Monolith Architecture | **Accepted** | Single deployable Node.js/Express modular monolith for Phase 1. |
| **ADR-002** | PostgreSQL, Prisma, and Raw Migrations | **Accepted** | Prisma ORM augmented with custom raw PostgreSQL migrations for GiST and partial indexes. |
| **ADR-003** | React from Day 1 | **Accepted** | Responsive React SPA included from Day 1 as a supporting product layer. |
| **ADR-004** | JWT Authentication and Token Storage | **Accepted** | JWT auth with password hashing; Option 1 (In-Memory Access Token + HttpOnly Refresh Cookie) selected. |
| **ADR-005** | Appointment and QueueToken Separation | **Accepted** | Distinct entities for scheduled bookings vs. live queue participation. |
| **ADR-006** | Provider-Specific Hybrid FIFO Queues | **Accepted** | Provider-scoped sessions; hybrid FIFO ordering with one-time priority recall. |
| **ADR-007** | Deferred Kafka & Transactional Outbox | **Accepted** | Synchronous monolith first; OutboxEvent and Kafka introduced in Phase 4. |
| **ADR-008** | Kafka Connect and PostgreSQL CDC | **Accepted** | Debezium streaming from PostgreSQL introduced in Phase 5. |
| **ADR-009** | Local Kubernetes First | **Accepted** | Local Kind/k3s learning in Phase 12; cloud managed clusters optional. |

---

## 15. Phase 0 Exit Criteria Checklist

- [x] Complete domain model, attributes, and constraints defined.
- [x] Full Mermaid ERD documented.
- [x] Appointment, QueueSession, and QueueToken state transition matrices fully defined.
- [x] Session-closure resolution matrix and reinstatement eligibility matrix defined.
- [x] Provider deactivation row-locking coordination and global lock hierarchy specified.
- [x] Concurrency controls (call-next serialization, single active call partial index) detailed.
- [x] API conventions, SQLSTATE mapping table, and Phase 1 endpoint catalog authored.
- [x] Layered `AuditEvent` immutability and Phase 4 `OutboxEvent` separation documented.
- [x] ADR-001 through ADR-009 authored in `docs/decisions/`.
- [x] Owner explicitly reviews documentation and approves Phase 0 (approved 2026-11-04; Phase 1 implementation remains awaiting explicit authorization).

---

## 16. Approval Record

**Owner Decision:** Approved  
**Date:** 2026-11-04  
**Notes:** Phase 0 architecture specification fully reviewed and approved by the project owner. Phase 0 is complete. Phase 1 implementation remains awaiting explicit project-owner authorization.
