# ADR-006: Provider-Specific Queues and Hybrid FIFO Ordering

## Status
**Accepted** (Approved by Project Owner - 2026-11-02)

## Context
In outpatient medical clinics, patient consultations are conducted by specific doctors working in assigned consultation rooms during defined shift hours.
Queue management in this domain requires solving three specific challenges:
1. **Queue Scope:** Should a queue be shared across all doctors in a department (pooled queue) or tied to an individual healthcare provider?
2. **Multi-Service Queues:** How can a single provider handle multiple consultation types (e.g., 15-minute routine follow-up vs. 45-minute new patient consultation) within the same operating session?
3. **Queue Fairness and Operational Realities:** How do we maintain first-in, first-out (FIFO) fairness based on check-in arrival while allowing doctors to temporarily skip an unresponsive patient and recall them with immediate priority when they return, or reinstate an accidentally cancelled patient?

## Decision
We establish provider-scoped queue sessions combined with a hybrid FIFO queue ordering algorithm:

1. **Provider-Specific Sessions:**
   - Every live queue is represented by a `QueueSession` aggregate tied to a single `Provider`, `Location`, and session operating window `[startTime, endTime)`.
   - Providers cannot have overlapping active sessions. This invariant is enforced at the database level using a PostgreSQL `btree_gist` exclusion constraint:
     ```sql
     ALTER TABLE queue_sessions 
     ADD CONSTRAINT uq_provider_session_time_exclusion 
     EXCLUDE USING gist (
       provider_id WITH =,
       tstzrange(start_time, end_time, '[)') WITH &&
     ) WHERE (status != 'VOIDED');
     ```
   - All sessions except `VOIDED` reserve their provider's time interval. `CLOSED` sessions do not silently release the exclusion window, preventing accidental retroactive overlap; only `VOIDED` releases the reserved interval.
2. **Multi-Service Support via Token Snapshotting:**
   - Rather than scoping the session to a single service, a provider session admits any service that the provider is authorized to perform (via `ProviderService`).
   - When a `QueueToken` is issued, it snapshots the service duration and service name (`service_duration_minutes`, `service_name`).
   - Wait time estimation for any token in a mixed-service queue is calculated dynamically from per-token duration snapshots:
     $$\text{Estimated Wait} = \text{Remaining Active Estimate} + \sum_{T \in S_{\text{ahead}}} T.\text{durationMinutesSnapshot}$$
     where tokens ahead ($S_{\text{ahead}}$) follow hybrid FIFO ordering (priority recalled tokens first, then chronological waiting tokens).
     If an active consultation exists:
     - For `IN_SERVICE`, elapsed consultation time starts at `serviceStartedAt`:
       $$\text{Remaining Active Estimate} = \max\left(0, \text{activeToken.durationMinutesSnapshot} - \lfloor(\text{NOW} - \text{serviceStartedAt}) / 60\text{s}\rfloor\right)$$
       If the consultation exceeds its scheduled duration, the remaining estimate evaluates to 0 minutes, acknowledging that overrunning consultations cannot be predicted with certainty.
     - For `CALLED`, the consultation may not have started yet; elapsed time since `calledAt` is not subtracted as though consultation had begun. The estimate is less certain while a token is `CALLED` (evaluating to the full `durationMinutesSnapshot`) and may be recalculated when service starts (`IN_SERVICE`).
     Presented as a dynamic estimate, not a guaranteed consultation or appointment time.
3. **Hybrid FIFO Queue Ordering:**
   - The application first locks the `QueueSession` row with `FOR UPDATE`:
     ```sql
     SELECT id, status FROM queue_sessions WHERE id = :sessionId FOR UPDATE;
     ```
   - Checks that the session is `OPEN` and that no active `CALLED` or `IN_SERVICE` token exists.
   - Calling the next patient is driven by a deterministic hybrid ordering query with valid PostgreSQL ordering (`LIMIT 1 FOR UPDATE SKIP LOCKED`):
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
   - `QueueSession FOR UPDATE` serves as the primary call-next serializer. `SKIP LOCKED` skips rows that are locked by other concurrent transactions, preventing lock wait contention or deadlocks if another transaction concurrently holds a lock on a token row. It does not affect ordinary non-locking inspection queries. The partial unique index `single_active_call_per_session` guarantees the single active call invariant at the engine level as defense in depth.
4. **State and Priority Transitions:**
   - **Ordinary Check-in:** Issued with `priority_next = FALSE`, `queue_order_time = NOW()`. Ordered strictly by check-in timestamp.
   - **Skip & Recall:** If a patient does not answer within `CALLED_RESPONSE_WINDOW_MINUTES` (5 min), they are marked `SKIPPED`. One-time priority recall (`recallCount <= 1`) is allowed through `NOW() <= skippedAt + 30 minutes` while the session is `OPEN`. Recalling sets `priority_next = TRUE`, `status = WAITING`, and `recalled_at = NOW()`. Transition to `NO_SHOW_UNRESPONSIVE` is permitted only when `NOW() > skippedAt + 30 minutes`. If the session closes while `NOW() <= skippedAt + 30 minutes`, the token defaults to `UNSERVED_CLINIC_CLOSED`.
   - **Atomic Priority Consumption:** When `callNext` executes, `priority_next` is immediately reset to `FALSE`, ensuring the recalled patient is served next without permanently dominating future queue calls.
   - **Reinstatement at Queue Tail:** If an accidentally cancelled or departed patient is reinstated (up to 2 times), `queue_order_time` is updated to `NOW()`, placing them at the back of the ordinary waiting line while retaining their original check-in timestamp for auditing.

## Alternatives Considered
- **Centralized Pooled Queues:** A single queue where patients are called by the next available provider. Rejected for Phase 1 because outpatient healthcare typically requires patient-doctor continuity; pooled queues will be considered as an optional future enhancement for urgent care scenarios.
- **Strict Timestamp-Only FIFO:** Rejected because it makes skipping impossible: if an unresponsive patient remains at the head of the queue, the entire clinic stops. If moved back without recall priority, a patient who stepped out to use the restroom would be forced to wait behind 20 new arrivals.

## Consequences
### Positive
- Matches real-world outpatient clinical workflows where doctors manage their own patient lists.
- Deterministic `callNext` execution using `QueueSession FOR UPDATE` and `FOR UPDATE SKIP LOCKED` without lock wait contention.
- Resilient queue operation: no single absent patient can block the clinic.
- Accurate wait time estimates reflecting varying consultation lengths.

### Liabilities
- Load imbalance across providers cannot be automatically rebalanced by the system in Phase 1; staff must manually rebook or reassign patients if one doctor falls behind.

