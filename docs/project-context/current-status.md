# QueueLess - Current Status

**Last updated:** 2026-11-04  
**Current phase:** Phase 0 - Complete (Phase 1 Not Started)  
**Status:** Phase 0 approved and complete on 2026-11-04. ADR-004 accepted with Option 1 selected (In-Memory Access Token + HttpOnly Refresh Cookie). Phase 1 is not started; Phase 1 implementation is awaiting explicit project-owner authorization.

## Verified facts

- The project constitution is preserved in `AGENTS.md` and updated with verified domain rules and locking protocols.
- Repository hygiene files (`.gitignore`, `LICENSE`, `SECURITY.md`) are present and verified; `Report/` and `*.diff` excluded in `.gitignore`.
- All Markdown files verified as valid UTF-8 without BOM or mojibake strings on disk.
- Appointment check-in window standardized everywhere: opens at `scheduledStartTime - 15 minutes`, closes at `scheduledStartTime + 15 minutes` (`APPOINTMENT_CHECKIN_GRACE_MINUTES = 15`). A BOOKED appointment not checked in by deadline may transition to `NO_SHOW`. Check-in window does not extend to `scheduledEndTime`.
- Skipped-token return window boundary standardized: recall allowed through `NOW() <= skippedAt + 30 minutes`; `NO_SHOW_UNRESPONSIVE` permitted only when `NOW() > skippedAt + 30 minutes`; defaults to `UNSERVED_CLINIC_CLOSED` if session closes before 30 minutes expire.
- Concurrency and data integrity designs are finalized:
  - Provider row canonical coordination lock (`FOR UPDATE` on deactivation vs. `FOR SHARE` on booking and session creation).
  - Global lock acquisition hierarchy: `Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken`. Deterministic primary-key ordering (`ORDER BY id ASC`) enforced for multi-row locks.
  - Multi-table transaction lock consistency: The analyzed queue lifecycle paths follow their documented lock acquisition order and deterministic primary-key ordering, subject to the stated transaction assumptions. This is not a proof of application-wide deadlock freedom. Start Service and Session Closure resolve both race orderings deterministically (Race A: Start Service transitions first, Closure aborts with `ACTIVE_CONSULTATION_IN_PROGRESS`; Race B: Closure closes session first, Start Service rejects on closed session with `QUEUE_SESSION_CLOSED`). Operations omitting `QueueSession` lock are documented and verified safe.
  - Dynamic wait-time estimation refined: per-token `durationMinutesSnapshot` summation for mixed-service queues. For `IN_SERVICE`, elapsed consultation time starts at `serviceStartedAt` (clamped to 0 if overrunning); for `CALLED`, the consultation may not have started, so elapsed time since `calledAt` is not subtracted, and the estimate is documented as less certain until service starts.
  - QueueSession row-level lock (`FOR UPDATE`) serializes Call Next, Session Closure, and Token Reinstatement.
  - Reinstatement protocol: acquires `QueueSession FOR UPDATE`, re-verifies session status is `OPEN`, then restores token at queue tail (`queue_order_time = NOW()`).
  - Session closure protocol: follows strict lock hierarchy `QueueSession -> Appointment (in PK order) -> QueueToken (in PK order)`. Acquires `QueueSession FOR UPDATE`, verifies no active consultation in progress, identifies records without premature token locks, locks linked `Appointment` rows in PK order, locks affected `QueueToken` rows in PK order, revalidates states under lock (mandatory post-lock check immediately aborts and rolls back with HTTP 409 and error code `ACTIVE_CONSULTATION_IN_PROGRESS` if ANY token transitioned to `IN_SERVICE` concurrently, because Start Service locks QueueToken without locking QueueSession and can race with closure), atomically resolves all unresolved tokens according to the closure matrix (`WAITING` tokens deterministically resolve to `UNSERVED_CLINIC_CLOSED` and never `NO_SHOW`; voluntary departures `PATIENT_DEPARTED` recorded during open session; expired `CALLED`/`SKIPPED` to `NO_SHOW_UNRESPONSIVE`; unexpired to `UNSERVED_CLINIC_CLOSED`; distinct appointment vs. walk-in outcomes), sets `CLOSED`, and commits atomically.
  - Call Next query uses valid PostgreSQL syntax (`ORDER BY ... LIMIT 1 FOR UPDATE SKIP LOCKED`) on `queue_session_id`. `QueueSession FOR UPDATE` serves as primary serializer; `SKIP LOCKED` skips rows locked by concurrent transactions without affecting non-locking queries; partial unique index (`status IN ('CALLED', 'IN_SERVICE')`) provides engine-level defense in depth.
  - QueueSession range exclusion predicate standardized to `WHERE (status != 'VOIDED')` using half-open `[start_time, end_time)` intervals.
  - Immutable `AuditEvent` ledger (`ON DELETE RESTRICT` on organization, database trigger `trg_audit_events_immutable` raises SQLSTATE `P0001` with message contract 'Audit records are immutable...'; application maps to `AUDIT_LOG_IMMUTABLE` only when both SQLSTATE and message contract match). Triggers protect against application logic bugs, but do not protect against database owners or superusers.
- No application implementation, database migrations, dependencies, or cloud resources have been created. Zero Git commits created.

## Decisions & Phase Status

- Phase 0: Formally reviewed, approved, and complete on 2026-11-04.
- ADR-004: Accepted on 2026-11-04 with Option 1 selected (In-Memory Access Token + HttpOnly Refresh Cookie). Documented server-side Origin allowlist validation, SameSite defense in depth, and anti-CSRF protections on both refresh and logout endpoints accepted.
- Phase 1: Not started.
- Phase 1 implementation: Awaiting explicit project-owner authorization before any application code, package manifests, database schemas, migrations, or infrastructure are created.
- Verification of exact driver/adapter SQLSTATE extraction for Prisma transaction retries will be confirmed during Phase 1 database client setup.

## Current task

Record Phase 0 approval and ADR-004 owner decision; maintain clean Phase 0 documentation state.

## Next step

Await explicit project-owner authorization to begin Phase 1 implementation.
