# AI Session Handoff - Phase 0 Approval Recorded

**Date:** 2026-11-04  
**Current phase:** Phase 0 Complete / Phase 1 Not Started  
**Active task:** Record Phase 0 Approval and ADR-004 Owner Decision  
**Model used:** Claude 3.5 Sonnet / Antigravity Agent  

## What changed

- **Final Consistency Pass Completed**:
  - Reinstatement lock protocol synchronized across all API and domain documentation: `QueueSession FOR UPDATE -> Appointment FOR UPDATE (if appointment-linked) -> QueueToken FOR UPDATE`, with post-lock verification that session status is `OPEN` and deterministic PK ordering.
  - Queue wait-time estimation refined: unified per-token `durationMinutesSnapshot` summation for mixed-service queues across tracking API and domain specs. For `IN_SERVICE`, elapsed consultation time starts at `serviceStartedAt` (clamped to 0 if overrunning); for `CALLED`, the consultation may not have started yet, so elapsed time since `calledAt` is not subtracted, and the estimate is documented as less certain while `CALLED` and recalculated when service starts.
  - Multi-table transaction lock consistency matrix authored: audited all 8 lifecycle operations (Call Next, Start Service, Skip, Recall, Complete Service, Cancellation, Reinstatement, Session Closure); documented that the analyzed queue lifecycle paths follow their documented lock acquisition order and deterministic primary-key ordering, subject to the stated transaction assumptions. This is not a proof of application-wide deadlock freedom. Documented safety rationale for operations not locking `QueueSession`.
  - Dual race ordering resolution between Start Service and Session Closure documented: (A) Start Service acquires `QueueToken FOR UPDATE` first, verifies `token.status === 'CALLED'` and parent session is `OPEN`, and transitions to `IN_SERVICE`, causing Session Closure's post-lock revalidation to detect `IN_SERVICE` and abort with `409 Conflict` (`ACTIVE_CONSULTATION_IN_PROGRESS`); (B) Session Closure acquires `QueueToken FOR UPDATE` first, resolves the token, and closes the session, causing Start Service's post-lock revalidation on `QueueToken` to detect that the token is no longer `CALLED` and/or the parent session is `CLOSED`, rejecting the transition with `409 Conflict` (`QUEUE_SESSION_CLOSED` or `TOKEN_NOT_CALLED`). A shared `QueueSession` lock was evaluated and avoided to eliminate unnecessary session-level lock contention.

- **Corrected Session-Closure Lock Ordering**:
  - Reconciled session-closure protocol across all architecture documents with the global lock hierarchy: `Organization -> Provider -> OrganizationMembership -> QueueSession -> Appointment -> QueueToken`.
  - Documented strict 9-step acquisition protocol: (1) acquire `QueueSession FOR UPDATE`; (2) verify `OPEN`/`PAUSED` and no `IN_SERVICE` token; (3) identify affected tokens and linked appointments without premature token locks; (4) lock linked `Appointment` rows in deterministic PK order (`ORDER BY id ASC FOR UPDATE`); (5) lock affected `QueueToken` rows in deterministic PK order (`ORDER BY id ASC FOR UPDATE`); (6) mandatory post-lock token revalidation under lock (aborts immediately with HTTP 409 `ACTIVE_CONSULTATION_IN_PROGRESS` if any token transitioned to `IN_SERVICE` concurrently); (7) deterministically resolve tokens and appointments according to closure matrix; (8) set `QueueSession.status = 'CLOSED'`; (9) commit atomically in a single transaction.
  - Verified lock ordering consistency with cancellation, reinstatement, and consultation completion.
- **Removed Duplicate Documentation**: Removed consecutive duplicate `Transaction Purity` statement from `phase-0-architecture-spec.md`.
- **Clarified Appointment Cancellation Boundary**: Explicitly distinguished that direct appointment cancellation is permitted before check-in (`status = BOOKED`), whereas after check-in, the live `QueueToken` is the operational control and departure is executed via token cancellation (`reason = PATIENT_DEPARTED`), synchronizing the linked appointment to `CANCELLED` and preserving reinstatement eligibility.

- **Clarified PATIENT_DEPARTED during Session Closure**:
  - Clarified that `PATIENT_DEPARTED` represents voluntary departures explicitly recorded during open sessions via normal token cancellation endpoints (`WAITING / CALLED / SKIPPED -> CANCELLED` with reason `PATIENT_DEPARTED`).
  - Removed redundant `PATIENT_DEPARTED` outcome rows from the session closure matrix. At closure, all remaining `WAITING` tokens deterministically resolve to `UNSERVED_CLINIC_CLOSED` because the clinic shift ended before they could be called.
  - Preserved owner-approved reinstatement rule: tokens cancelled with `PATIENT_DEPARTED` remain eligible for reinstatement if the patient returns while the session is `OPEN` (max 2 reinstatements). Once the session is `CLOSED`, reinstatement is barred (`SESSION_NOT_OPEN`).
- **Corrected Audit Trigger Error Mapping (`P0001`)**:
  - Documented that `P0001` is PostgreSQL's generic PL/pgSQL `raise_exception` code.
  - Mandated that the application error handler must NOT map every `P0001` to `AUDIT_LOG_IMMUTABLE`.
  - Established contract: application verifies `error.code === 'P0001' && error.message.includes('Audit records are immutable')` before returning `403 Forbidden` (`AUDIT_LOG_IMMUTABLE`). Unmatched `P0001` errors map to `500 Internal Server Error`.
- **Pure ASCII Sanitization for Universal Portability**:
  - Replaced all non-ASCII typographical punctuation (em-dashes, en-dashes, directional arrows, box drawing characters) across all repository Markdown files with clean, standard ASCII equivalents (`-`, `->`, `<->`, `+`, `|`, `\`).
  - Completely eliminated any possibility of CP437/OEM mojibake (`mojibake-emdash`, `mojibake-arrow`, `mojibake-biarrow`) occurring in exported diffs or Windows console redirections.

- **Session Closure Resolution Reconciled**: Reconciled general session closure protocol across `phase-0-architecture-spec.md`, `domain-model.md`, `api-conventions.md`, `PROJECT_CONTEXT.md`, and `AGENTS.md` with the detailed closure matrix:
  - `WAITING` tokens are never marked `NO_SHOW_UNRESPONSIVE`; they transition to `CANCELLED` (`UNSERVED_CLINIC_CLOSED` or `PATIENT_DEPARTED`).
  - `CALLED` tokens with expired response window (`NOW() >= calledAt + 5 min`) and `SKIPPED` tokens with expired return window (`NOW() > skippedAt + 30 min`) may be classified as `NO_SHOW_UNRESPONSIVE`.
  - Unexpired tokens default to `UNSERVED_CLINIC_CLOSED`.
  - Linked appointments update in sync (`NO_SHOW` or `CANCELLED`), while walk-in tokens resolve standalone to `CANCELLED` with guest tracking secret revocation.
  - Entire closure runs atomically while holding `QueueSession FOR UPDATE`.
- **Skipped-Token Deadline Standardized**:
  - One-time priority recall (`recallCount <= 1`) allowed through `NOW() <= skippedAt + 30 minutes` while session is `OPEN`.
  - `NO_SHOW_UNRESPONSIVE` permitted only when `NOW() > skippedAt + 30 minutes`.
  - Fixed boundary condition in `domain-model.md` closure matrix (changed `>=` to `>`).
- **SKIP LOCKED Explanation Clarified**:
  - Clarified across `phase-0-architecture-spec.md`, `adr-006`, `api-conventions.md`, `PROJECT_CONTEXT.md`, and `AGENTS.md` that `SKIP LOCKED` skips rows locked by other concurrent transactions, preventing lock wait contention or deadlocks.
  - Explicitly removed claims that it protects against non-locking inspection queries.
  - Re-affirmed `QueueSession FOR UPDATE` as the primary call-next serializer and the partial unique index as engine-level defense in depth.
- **UTF-8 Encoding & Mojibake Verification**:
  - Verified all Markdown files in the repository are valid UTF-8 without BOM.
  - Confirmed zero mojibake byte sequences exist in repository Markdown files on disk.
  - Identified that the corrupted character sequences observed during diff review were artifacts of PowerShell 5.1's default OEM (CP437) redirection to UTF-16LE when exporting `git diff` on Windows.
- **Status & Task Trackers**: Updated `current-status.md`, `AI-HANDOFF.md`, and `current-task.md`.

## Verification evidence

- **Session Closure Verification**: Confirmed all documents state `WAITING` tokens cannot be marked `NO_SHOW_UNRESPONSIVE` and follow the closure matrix.
- **Skipped-Token Boundary Verification**: Confirmed exact alignment: recall through `NOW() <= skippedAt + 30 min`; no-show only when `NOW() > skippedAt + 30 min`.
- **SKIP LOCKED Verification**: Verified accurate database locking description (skips rows locked by other transactions; non-locking queries are unaffected).
- **Encoding Verification**: Full Python scan confirmed all markdown files on disk decode cleanly as valid UTF-8 without BOM or mojibake.
- **Repository State**: `git status` clean documentation; zero application code, zero packages, zero migrations, zero Git commits.
- **Phase Status**: Phase 0 formally reviewed, approved, and complete on 2026-11-04. Phase 1 is not started.

## Decisions

- **Approved by Owner**:
  1. `APPOINTMENT_CHECKIN_GRACE_MINUTES = 15` (Window: `[Start - 15m, Start + 15m]`, does not extend to `scheduledEndTime`).
  2. `CALLED_RESPONSE_WINDOW_MINUTES = 5`.
  3. `SKIPPED_RETURN_WINDOW_MINUTES = 30` (Recall: `NOW() <= skippedAt + 30m`; No-show: `NOW() > skippedAt + 30m`).
  4. Maximum 2 token reinstatements. Eligible: `ACCIDENTAL_CANCELLATION`, `PATIENT_DEPARTED` (while session is `OPEN`). Ineligible: `NO_SHOW_UNRESPONSIVE`.
  5. Session voiding permitted only for `ORG_ADMIN`, only while `SCHEDULED`, and only if zero queue tokens were created or served.
  6. ADR-001 through ADR-009 all marked `Accepted`.
  7. ADR-004 accepted on 2026-11-04 selecting Option 1: In-Memory Access Token + HttpOnly Refresh Cookie, with documented server-side Origin allowlisting, SameSite defense in depth, and anti-CSRF verification.
  8. Phase 0 Architecture and Planning approved and complete.
- **Pending Verification in Phase 1**:
  - Exact Prisma runtime driver SQLSTATE extraction details to be validated during Phase 1 database client setup.

## Blockers and risks

- None for Phase 0. Phase 1 implementation is awaiting explicit project-owner authorization before any code or infrastructure is created.

## Next exact step

Await explicit project-owner authorization to begin Phase 1 implementation.

## Owner approval needed?

- Owner authorization is required before beginning Phase 1 implementation. (Phase 0 and ADR-004 are fully approved).
