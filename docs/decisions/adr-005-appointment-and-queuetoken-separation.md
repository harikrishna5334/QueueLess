# ADR-005: Separation of Appointment and QueueToken

## Status
**Accepted** (Approved by Project Owner - 2026-11-02)

## Context
In healthcare outpatient clinics and walk-in practices, two distinct operational concepts govern patient flow:
1. **Scheduled Bookings (Appointments):** Planned future reservations where a patient commits to arrive at a specified date and time window with a specific provider and service.
2. **Physical Queue Participation (Queue Tokens):** Real-time queue participation on the day of service, representing arrival, waiting room presence, and position in line.

In real-world clinic workflows:
- Walk-in patients arrive without advance appointments.
- Booked patients may cancel hours or days ahead.
- Booked patients may fail to show up (`NO_SHOW`) without ever entering the waiting room.
- Booked patients may arrive early or late, entering the queue at their actual physical check-in time.
- Patients in the live queue may be temporarily skipped if unavailable when called, recalled later, or cancelled if they depart the waiting room.

Collapsing `Appointment` and `QueueToken` into a single entity (e.g., a unified "Booking" table) creates severe structural defects:
- Walk-ins require dummy scheduled timestamps or nullable appointment fields throughout the schema.
- Scheduled appointment statuses (`BOOKED`, `NO_SHOW`) conflict with live queue statuses (`WAITING`, `CALLED`, `IN_SERVICE`).
- The system cannot record no-show analytics separately from live queue delays.
- Auditing the patient journey becomes convoluted.

## Decision
We model `Appointment` and `QueueToken` as separate aggregate roots with independent lifecycles, states, and business rules:

1. **Independent Lifecycles:**
   - `Appointment` state machine:
     ```
     BOOKED --+--> CHECKED_IN --> COMPLETED
              +--> CANCELLED
              \--> NO_SHOW
     ```
   - `QueueToken` state machine:
     ```
     WAITING --+--> CALLED --> IN_SERVICE --> COMPLETED
               +--> SKIPPED --> WAITING (on recall)
               \--> CANCELLED
     ```
2. **1-to-(0..1) Association:**
   - An appointment yields at most one queue token upon physical check-in.
   - Enforced by a database partial unique constraint:
     ```sql
     CREATE UNIQUE INDEX uq_queue_tokens_appointment_id 
     ON queue_tokens (appointment_id) 
     WHERE appointment_id IS NOT NULL;
     ```
3. **Walk-in and Guest Support:**
   - Walk-in tokens have `appointment_id = NULL`.
   - Guest walk-ins (patients without an account) have `user_id = NULL` and are issued a cryptographically random 256-bit tracking secret whose SHA-256 hash is persisted in `guest_tracking_secret_hash`.
4. **Lifecycle Coordination:**
   - The check-in window opens at `scheduledStartTime - 15 minutes` and closes at `scheduledStartTime + 15 minutes` (`APPOINTMENT_CHECKIN_GRACE_MINUTES = 15`). The check-in window does not extend to `scheduledEndTime`. An appointment that has not checked in by the deadline may transition to `NO_SHOW`.
   - Within this window, check-in creates a `QueueToken` (`status = WAITING`) and updates `Appointment.status = CHECKED_IN` within the same database transaction.
   - When a `QueueToken` transitions to `COMPLETED`, if `appointment_id IS NOT NULL`, the associated `Appointment.status` is atomically updated to `COMPLETED`.
   - **Cancellation Boundary Before vs. After Check-In**:
     - Direct appointment cancellation (`POST /api/v1/appointments/:id/cancel`) is permitted strictly before check-in while `Appointment.status = 'BOOKED'`.
     - After check-in, the live `QueueToken` becomes the primary operational control for queue participation, and direct appointment cancellation is barred (`400 Bad Request`, `APPOINTMENT_ALREADY_CHECKED_IN`).
     - If a checked-in patient voluntarily departs the waiting room, cancellation must be executed on the live `QueueToken` (`POST /api/v1/queue-tokens/:id/cancel` with reason `PATIENT_DEPARTED`), which atomically synchronizes the linked `Appointment.status` to `CANCELLED` (reason `PATIENT_DEPARTED`).
     - Existing audit history (`TOKEN_CANCELLED`, `APPOINTMENT_CANCELLED`) and reinstatement rules are preserved: the token remains eligible for staff reinstatement if the patient returns while the session is still `OPEN` (up to 2 reinstatements). Once the session is closed, reinstatement is barred.
5. **No Duplicate Check-in:**
   - Once an appointment has checked in and created a `QueueToken`, it can never create another queue token. If a patient leaves and returns, queue reinstatement rules apply to the existing `QueueToken` (up to 2 reinstatements for eligible reasons), preserving audit continuity.

## Alternatives Considered
- **Single Unified "Booking" Entity:** Rejected because it forces nullable scheduling columns onto walk-ins, conflates scheduled time with check-in time, and creates ambiguity between `NO_SHOW` (never arrived) and `SKIPPED` (arrived but unseated).
- **Allowing Re-Check-in (New Token per Check-in):** Rejected because it violates the 1-to-1 integrity invariant, complicates wait time calculation, and allows malicious duplicate queue entry.

## Consequences
### Positive
- Strict separation between scheduling logic and real-time queue orchestration.
- First-class support for both pre-booked patients and walk-in / guest patients.
- Crystal-clear analytics for clinic operations (no-show rate vs. average queue wait time).
- Invariant integrity guaranteed at the PostgreSQL constraint level.

### Liabilities
- Requires cross-aggregate coordination inside application services and database transactions to synchronize completion states.
- Client applications must distinguish between appointment details and live queue status.

