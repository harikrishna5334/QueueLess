# QueueLess API Conventions & Endpoint Catalog

**Status:** Approved for Phase 0.

All QueueLess REST APIs follow consistent design conventions, standard error structures, explicit PostgreSQL SQLSTATE mappings, and strict security/privacy boundaries.

---

## 1. Core API Conventions

- **Base Prefix**: `/api/v1/`
- **Transport**: HTTPS required in all environments.
- **Content-Type**: `application/json` for all request and response bodies.
- **Character Encoding**: UTF-8.
- **Date/Time Format**: ISO 8601 UTC strings (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **Input Validation**: Strictly validated at the boundary using Zod before controller execution.

---

## 2. Standard Response Envelopes

### 2.1 Standard Success Envelope
```json
{
  "data": { ... },
  "meta": { ... } // Optional: pagination, timestamps, or disclaimers
}
```

### 2.2 Standard Error Envelope (RFC 7807 Aligned)
```json
{
  "error": {
    "code": "ERROR_CODE_STRING",
    "message": "Human-readable explanation of the error.",
    "details": [ ... ] // Optional: validation issues or field-specific errors
  }
}
```

---

## 3. PostgreSQL SQLSTATE Error Mapping

The API error middleware inspects the database error `SQLSTATE` and specific constraint name, translating engine violations into clean HTTP responses:

| SQLSTATE | Constraint / Condition Name | Application Error Code | HTTP Status | Response Message |
| :--- | :--- | :--- | :--- | :--- |
| **`23505`** | `queue_tokens_appointment_id_key` | `DUPLICATE_CHECK_IN` | `409 Conflict` | "This appointment has already been checked in." |
| **`23505`** | `single_active_call_per_session` | `ACTIVE_CALL_IN_PROGRESS` | `409 Conflict` | "The provider already has an active patient called or in service." |
| **`23505`** | `org_memberships_user_org_role_key` | `MEMBERSHIP_ALREADY_EXISTS` | `409 Conflict` | "User already holds this role within the organization." |
| **`23505`** | `users_email_key` | `EMAIL_ALREADY_REGISTERED` | `409 Conflict` | "An account with this email address already exists." |
| **`23P01`** | `no_overlapping_provider_sessions` | `PROVIDER_SESSION_OVERLAP` | `409 Conflict` | "Provider already has a queue session scheduled in this time window." |
| **`23503`** | `fk_provider_membership` | `INVALID_PROVIDER_MEMBERSHIP` | `400 Bad Request` | "User does not hold a PROVIDER membership in this organization." |
| **`23503`** | *(General foreign key violation)* | `RESOURCE_NOT_FOUND` | `404 Not Found` | "Referenced resource does not exist." |
| **`23514`** | `queue_sessions_start_time_end_time_check` | `INVALID_SESSION_TIMES` | `400 Bad Request` | "Session start time must be chronologically earlier than end time." |
| **`23514`** | `queue_tokens_walkin_or_patient_check` | `INVALID_TOKEN_IDENTITY` | `400 Bad Request` | "A token must belong to either a registered patient or a guest walk-in, not both." |
| **`P0001`** (with message contract) | `trg_audit_events_immutable` | `AUDIT_LOG_IMMUTABLE` | `403 Forbidden` | "Audit records cannot be modified or deleted." |
| **`40001`** / **`40P01`** | Serialization failure or deadlock | `TRANSACTION_RETRY_EXHAUSTED` | `503 Service Unavailable` | "System is experiencing high concurrency; please retry your request." |

> [!IMPORTANT]
> **Audit Immutability Trigger Error Identification (`P0001`)**:
> `P0001` is PostgreSQL's generic PL/pgSQL error code for `RAISE EXCEPTION`. The application error middleware must **not** map every `P0001` error to `AUDIT_LOG_IMMUTABLE`. Instead, the application checks both the SQLSTATE and matches the trigger's stable contract error message:
> `error.code === 'P0001' && error.message.includes('Audit records are immutable')`
> Only when both the SQLSTATE and message contract match is the error translated to `403 Forbidden` (`AUDIT_LOG_IMMUTABLE`). Any other `P0001` error from unrelated database triggers or functions is mapped to `500 Internal Server Error` (`INTERNAL_SERVER_ERROR`).

---

## 4. Authentication & Security Headers

### 4.1 Authenticated Users (Patients & Staff)
- Authenticated requests pass credentials via the standard HTTP header:
  `Authorization: Bearer <jwt_access_token>`
  *(Per ADR-004 Option 1 accepted resolution: short-lived access tokens stored in memory and sent via Authorization Bearer header; refresh tokens stored in HttpOnly cookies)*.
- Scoped staff operations inspect the user's `OrganizationMembership` for the target clinic.

### 4.2 Guest Walk-Ins (Private Token Tracking)
- Guest walk-in tracking requests pass the 256-bit CSPRNG tracking secret via a dedicated header:
  `X-Queue-Tracking-Token: <plaintext_secret>`
- **Security & Hygiene Rules**:
  - The secret is **never accepted via URL query parameters** (`?token=...`), preventing credential leakage into server access logs, proxy logs, and browser referrers.
  - Application loggers (Pino/Winston) and infrastructure proxies must explicitly redact `req.headers['x-queue-tracking-token']`.
  - Hashed at rest using SHA-256 (`trackingTokenHash`).
  - Terminal states (`COMPLETED`, `CANCELLED`) permanently revoke the secret (`trackingTokenHash = NULL`).
  - Responses to `GET /api/v1/queue-tokens/track` are **strictly PII-free** (`tokenNumber`, `positionInQueue`, `estimatedWaitMinutes`, `assignedRoom`, `sessionStatus`).

---

## 5. Pagination Conventions

For list endpoints returning multiple rows:
- **Query Parameters**: `?page=1&limit=20` (Default `limit=20`, maximum `limit=100`).
- **Response Shape**:
```json
{
  "data": [ ... ],
  "meta": {
    "page": 1,
    "limit": 20,
    "totalCount": 85,
    "totalPages": 5
  }
}
```

---

## 6. Phase 1 Endpoint Catalog

### 6.1 Authentication (`/api/v1/auth`)
- `POST /api/v1/auth/register`: Public. Registers a new universal `PATIENT` user.
- `POST /api/v1/auth/login`: Public. Validates credentials (Argon2id), issues JWT.
- `POST /api/v1/auth/logout`: Authenticated. Terminates session.
- `POST /api/v1/auth/refresh`: Refreshes short-lived access token.
- `POST /api/v1/auth/forgot-password`: Public. Development-only log mechanism in Phase 1.
- `POST /api/v1/auth/reset-password`: Public. Consumes reset token and updates password.

### 6.2 Users & Profiles (`/api/v1/users`)
- `GET /api/v1/users/me`: Authenticated. Returns current user profile and active memberships.
- `PATCH /api/v1/users/me`: Authenticated. Updates current user personal info.

### 6.3 Organizations & Locations (`/api/v1/organizations`, `/api/v1/locations`)
- `GET /api/v1/organizations`: Public. Lists active clinics and health centers.
- `GET /api/v1/organizations/:id`: Public. Retrieves organization details.
- `GET /api/v1/organizations/:id/locations`: Public. Lists locations for an organization.
- `POST /api/v1/organizations/:id/locations`: `ORG_ADMIN` only. Creates a clinic location.
- `PATCH /api/v1/locations/:id`: `ORG_ADMIN` only. Updates location details (e.g. timezone).

### 6.4 Services & Providers (`/api/v1/services`, `/api/v1/providers`)
- `GET /api/v1/organizations/:id/services`: Public. Lists services offered by an organization.
- `POST /api/v1/organizations/:id/services`: `ORG_ADMIN` only. Creates a service with duration.
- `GET /api/v1/organizations/:id/providers`: Public. Lists active providers in an organization.
- `POST /api/v1/organizations/:id/providers`: `ORG_ADMIN` only. Coordinates atomic creation of `OrganizationMembership` and `Provider` profile.
- `DELETE /api/v1/organizations/:id/providers/:providerId`: `ORG_ADMIN` only. Coordinated soft-deactivation. Locks provider `FOR UPDATE`, deactivates membership and provider.
- `POST /api/v1/providers/:id/services`: `ORG_ADMIN` only. Maps a provider to a service.

### 6.5 Appointments (`/api/v1/appointments`)
- `POST /api/v1/appointments`: Patient only. Books an appointment. Coordinates `FOR SHARE` on Provider; checks non-overlapping intervals for provider and patient.
- `GET /api/v1/appointments/my-appointments`: Patient only. Lists user's future and historical bookings.
- `GET /api/v1/appointments/:id`: Authenticated (Patient owner, assigned Provider, or `ORG_ADMIN`).
- `POST /api/v1/appointments/:id/cancel`: Patient owner or Staff. Advance cancellation strictly permitted before check-in while in `BOOKED` status. Once checked in (`status = 'CHECKED_IN'`), the live `QueueToken` becomes the operational control; direct cancellation via this endpoint is barred (`400 Bad Request`, `APPOINTMENT_ALREADY_CHECKED_IN`). A checked-in patient who leaves cancels via `POST /api/v1/queue-tokens/:id/cancel` (reason `PATIENT_DEPARTED`), which atomically synchronizes the linked appointment to `CANCELLED`.
- `POST /api/v1/appointments/:id/check-in`: Patient owner or Staff. Executes check-in within the window `[scheduledStartTime - 15 minutes, scheduledStartTime + 15 minutes]`. Creates `QueueToken` in `WAITING` status atomically. Rejects requests after the 15-minute grace deadline (`scheduledStartTime + 15 minutes`) with `400 Bad Request` (`CHECKIN_WINDOW_EXPIRED`), permitting transition to `NO_SHOW`. Check-in window does not extend to `scheduledEndTime`.

### 6.6 Queue Sessions (`/api/v1/queue-sessions`)
- `POST /api/v1/organizations/:id/queue-sessions`: Staff only. Schedules a session (`SCHEDULED`). Coordinates `FOR SHARE` on Provider; validates half-open GiST exclusion (`WHERE (status != 'VOIDED')`).
- `PATCH /api/v1/queue-sessions/:id/times`: `ORG_ADMIN` or Provider. Updates `startTime` and `endTime` before session opening.
- `POST /api/v1/queue-sessions/:id/open`: Staff or Provider. Transitions session `SCHEDULED -> OPEN`.
- `POST /api/v1/queue-sessions/:id/pause`: Provider only. Transitions session `OPEN -> PAUSED`.
- `POST /api/v1/queue-sessions/:id/resume`: Provider only. Transitions session `PAUSED -> OPEN`.
- `POST /api/v1/queue-sessions/:id/close`: Staff or Provider. Authorizes session closure. Follows the global lock hierarchy: acquires `QueueSession FOR UPDATE`, verifies session is `OPEN` or `PAUSED` and no token is `IN_SERVICE` (`409 Conflict`, `ACTIVE_CONSULTATION_IN_PROGRESS`), identifies affected records without prematurely locking tokens, acquires locks on linked `Appointment` rows in deterministic primary-key order (`ORDER BY id ASC FOR UPDATE`), acquires locks on affected `QueueToken` rows in deterministic primary-key order (`ORDER BY id ASC FOR UPDATE`), revalidates states under lock (specifically, if ANY affected token has transitioned to `IN_SERVICE` concurrently, the entire transaction aborts immediately with HTTP 409 and error code `ACTIVE_CONSULTATION_IN_PROGRESS`, making zero updates, because Start Service locks QueueToken without locking QueueSession and can race with closure), and deterministically resolves all unresolved tokens and linked appointments according to the closure matrix (staff cannot arbitrarily choose a no-show outcome): `WAITING` tokens deterministically resolve to `CANCELLED` with reason `UNSERVED_CLINIC_CLOSED` (never `NO_SHOW_UNRESPONSIVE`; voluntary departures `PATIENT_DEPARTED` are recorded during open sessions via normal cancellation endpoints); `CALLED` tokens become `NO_SHOW_UNRESPONSIVE` only after the 5-minute response window (`NOW() >= calledAt + 5 min`); `SKIPPED` tokens become `NO_SHOW_UNRESPONSIVE` only strictly after the 30-minute return window (`NOW() > skippedAt + 30 min`); unexpired tokens default to `UNSERVED_CLINIC_CLOSED`. Appointment-linked tokens update their appointments atomically (to `NO_SHOW` or `CANCELLED`); walk-in tokens resolve independently (with guest tracking secret revocation). Updates the session to `CLOSED` and commits atomically.
- `POST /api/v1/queue-sessions/:id/void`: `ORG_ADMIN` only. Voids a pre-operational session (`SCHEDULED -> VOIDED`). Guarded by `COUNT(queue_tokens) === 0`.
- `GET /api/v1/queue-sessions/:id/public-display`: Public. Returns anonymized display board data (`tokenNumber`, `status`, room number). Zero PII.

### 6.7 Queue Tokens (`/api/v1/queue-tokens`)
- `POST /api/v1/queue-sessions/:id/walk-in`: Staff/Kiosk. Creates a guest walk-in token (`WAITING`). Returns token data and plaintext 256-bit `trackingSecret`.
- `GET /api/v1/queue-tokens/track`: Public (Header Auth via `X-Queue-Tracking-Token`). Returns PII-free operational status and dynamic wait estimate.
  Wait time is calculated dynamically per-token for mixed-service queues:
  Estimated Wait = Active Consultation Remaining + Sum of durationMinutesSnapshot for all WAITING tokens ahead.
  Tokens ahead include priority recalled tokens (priorityNext = TRUE, ordered by recalledAt ASC NULLS LAST) followed by ordinary waiting tokens (queueOrderTime ASC).
  Active consultation remaining estimate calculation:
  - For `IN_SERVICE`, elapsed consultation time starts at `serviceStartedAt`: `MAX(0, activeToken.durationMinutesSnapshot - elapsedMinutesSinceStart)`. If the consultation exceeds scheduled duration, remaining time evaluates to 0 minutes, acknowledging that overrunning consultations cannot be predicted with certainty.
  - For `CALLED`, the consultation may not have started yet (patient is responding or entering the room); elapsed time since `calledAt` is not subtracted as though consultation had begun. The estimate is less certain while a token is `CALLED` (evaluating to the full `durationMinutesSnapshot`) and may be recalculated when service starts (`IN_SERVICE`).
  Presented explicitly as a dynamic estimate, never a guaranteed consultation or appointment time.
- `GET /api/v1/queue-tokens/my-active`: Authenticated Patient. Returns full queue position and appointment details for the logged-in patient.
- `POST /api/v1/queue-sessions/:id/call-next`: Provider only. Serializes on `QueueSession FOR UPDATE`, verifies zero tokens in `CALLED` or `IN_SERVICE`, then selects next eligible token via:
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
  `QueueSession FOR UPDATE` serves as the primary call-next serializer. `SKIP LOCKED` skips rows locked by other concurrent transactions, preventing lock wait contention or deadlocks if another transaction concurrently holds a lock on a token row. It does not affect ordinary non-locking inspection queries. Consumes priority (`priority_next = FALSE`), transitions token to `CALLED`, sets `called_at = NOW()`.
- `POST /api/v1/queue-tokens/:id/start-service`: Provider only. Transitions token `CALLED -> IN_SERVICE`. Acquires `QueueToken FOR UPDATE`, then performs mandatory post-lock revalidation: verifies `token.status === 'CALLED'` and parent `QueueSession.status === 'OPEN'`. If `token.status !== 'CALLED'`, rejects with `409 Conflict` (`TOKEN_NOT_CALLED`). If the parent session is no longer `OPEN`, rejects with `409 Conflict` (`QUEUE_SESSION_CLOSED` or `QUEUE_SESSION_PAUSED`). Handles both race orderings with Session Closure: in Race Ordering A, Start Service locks `QueueToken` first and transitions to `IN_SERVICE`, causing Session Closure's post-lock check to abort with `409 Conflict` (`ACTIVE_CONSULTATION_IN_PROGRESS`); in Race Ordering B, Session Closure locks `QueueToken` first and closes the session, causing Start Service to reject the transition with `409 Conflict` (`QUEUE_SESSION_CLOSED`). Sets `service_started_at = NOW()`.
- `POST /api/v1/queue-tokens/:id/complete`: Provider only. Transitions token `IN_SERVICE -> COMPLETED` and linked `Appointment -> COMPLETED`. Vacates active call slot. Revokes guest tracking secret.
- `POST /api/v1/queue-tokens/:id/skip`: Provider only. Patient absent after 5 min response window (`CALLED_RESPONSE_WINDOW_MINUTES = 5`). Transitions token `CALLED -> SKIPPED`. Vacates active call slot.
- `POST /api/v1/queue-tokens/:id/recall`: Provider only. Allowed through `NOW() <= skippedAt + 30 minutes` (`SKIPPED_RETURN_WINDOW_MINUTES = 30`) and `recallCount < 1`. Transitions token `SKIPPED -> WAITING` with `priorityNext = TRUE`. Beyond 30 minutes (`NOW() > skippedAt + 30 minutes`), recall is rejected and token may transition to `CANCELLED` (`NO_SHOW_UNRESPONSIVE`).
- `POST /api/v1/queue-tokens/:id/cancel`: Patient or Staff. Cancels active token (`WAITING`, `CALLED`, or `SKIPPED`). Follows lock order: if appointment-linked, acquires `Appointment FOR UPDATE` then `QueueToken FOR UPDATE`. Atomically synchronizes linked appointment to `CANCELLED` (reason: `PATIENT_DEPARTED`). Preserves audit history (`TOKEN_CANCELLED`, `APPOINTMENT_CANCELLED`). Reinstatement remains eligible while session is `OPEN` (up to 2 times).
- `POST /api/v1/queue-tokens/:id/reinstate`: Staff only. Follows the global lock hierarchy:
  1. Acquires `QueueSession FOR UPDATE`.
  2. Re-checks that `QueueSession.status === 'OPEN'` after acquiring lock (rejects with `400 Bad Request`, `QUEUE_SESSION_NOT_OPEN` if closed or paused).
  3. If appointment-linked (`appointmentId != NULL`), acquires `Appointment FOR UPDATE`.
  4. Acquires `QueueToken FOR UPDATE`.
  (Deterministic primary-key ordering `ORDER BY id ASC` is enforced if multiple rows of any entity are locked).
  Verifies guards: `status = 'CANCELLED'`, eligible reason (`ACCIDENTAL_CANCELLATION`, `PATIENT_DEPARTED`), and `reinstatementCount < 2`. Restores token to `WAITING` at queue tail (`queue_order_time = NOW()`), increments `reinstatementCount`, and synchronizes linked appointment back to `CHECKED_IN`.
