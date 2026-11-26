# ADR-007: Deferred Kafka Introduction and Transactional Outbox

## Status
**Accepted** (Approved by Project Owner - 2026-11-02)

## Context
QueueLess is designed as a portfolio-grade distributed systems learning and demonstration project. One of its primary pedagogical goals is demonstrating event-driven architecture, event streaming, consumer groups, partition keys, offsets, and idempotency using Apache Kafka.

However, introducing Kafka prematurely into Phase 1 creates serious architectural liabilities:
1. **Unjustified Operational Overhead:** Running Kafka brokers (or KRaft controllers), Zookeeper, and schema registries locally or in CI before basic CRUD, authentication, and core domain state machines are proven creates distraction and friction.
2. **Dual-Write Hazards:** Writing to both PostgreSQL and Kafka directly from application HTTP handlers without transactional outbox mechanisms results in partial failure modes (e.g., database commit succeeds but message publishing fails, or vice versa).
3. **Confusion Between Auditing and Streaming:** Developers often mistake audit logs for message queues. In QueueLess, `AuditEvent` is an immutable, write-once compliance ledger required for healthcare operations from Day 1, whereas Kafka is an asynchronous event streaming transport.

## Decision
We formally defer Apache Kafka to **Phase 4** and establish clear architectural boundaries between auditing and streaming:

1. **Phase 1-3 Architecture:**
   - All operations are executed as local PostgreSQL ACID transactions within the Node.js modular monolith.
   - Synchronous transactional updates guarantee immediate consistency for appointments, queue tokens, and sessions.
   - The system records compliance history in an append-only `AuditEvent` table, protected by a database trigger that rejects updates and deletions (SQLSTATE `P0001`, `raise_exception`; triggers protect against application bugs, but do not protect against database superusers). `AuditEvent` is strictly a compliance log, not a message queue.
2. **Phase 4 Kafka Introduction:**
   - In Phase 4, Apache Kafka is introduced using a local Docker Compose environment (configured with KRaft mode, eliminating Zookeeper dependency).
   - Domain events are NOT published directly over network calls during HTTP requests.
   - We implement the **Transactional Outbox Pattern**:
     - A dedicated `OutboxEvent` table (`id`, `aggregate_type`, `aggregate_id`, `event_type`, `payload`, `created_at`, `published_at`) is written in the exact same database transaction as the business entity changes.
     - An asynchronous worker process polls or tails the outbox and publishes events to Kafka with strict ordering based on partition keys (e.g., `sessionId`, `providerId`, or `patientId`).
3. **Idempotent Consumers:**
   - Downstream consumers in Phase 4 and beyond will track processed event IDs in an `IdempotencyKey` table to safely handle Kafka's at-least-once delivery guarantees.

## Alternatives Considered
- **Kafka from Day 1:** Rejected as a direct violation of Project Constitution Section 2 ("Every technology must have a reason") and Section 43 (Phase progression). It distracts from establishing sound relational modeling and authentication.
- **Direct Dual-Write (Publishing to Kafka from Express Controllers):** Rejected because network partitions between the API server and Kafka broker guarantee inconsistent state and data loss.
- **Using Redis Pub/Sub Instead of Kafka:** Redis Pub/Sub provides ephemeral messaging without persistent offsets or replayability, making it unsuitable for event streaming and auditing. Redis is introduced in Phase 3 solely for WebSocket presence and ephemeral cache.

## Consequences
### Positive
- Zero infrastructure friction in Phases 1-3: rapid local development using standard PostgreSQL and Node.js.
- Clean separation between compliance audit logging (`AuditEvent`) and asynchronous message streaming (`OutboxEvent`).
- Eliminates dual-write anomalies by design.
- Maximizes pedagogical value by introducing distributed streaming at the exact phase where developers are ready to learn it.

### Liabilities
- Asynchronous consumers in Phase 4 must handle eventual consistency and outbox lag.
- Requires introducing worker processes and outbox cleanup routines in Phase 4.

