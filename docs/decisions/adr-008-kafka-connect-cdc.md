# ADR-008: Kafka Connect and PostgreSQL Change Data Capture (CDC)

## Status
**Accepted** (Approved by Project Owner - 2026-11-02)

## Context
In Phase 4, QueueLess establishes the Transactional Outbox pattern where application code writes events to an `OutboxEvent` table and a worker relays them to Kafka.
In Phase 5, the architectural focus advances to automated, high-throughput data streaming and Change Data Capture (CDC):
1. **Application Polling Bottlenecks:** Having application workers poll an outbox table (`SELECT ... FOR UPDATE SKIP LOCKED` or `WHERE published = false`) introduces polling latency, continuous database CPU overhead, and potential table bloat.
2. **Pedagogical Goals:** A key portfolio objective is mastering Kafka Connect, distributed connector architecture, source and sink connectors, WAL-level capture, schema registries, and Debezium.
3. **Database Mutation Capture:** Demonstrating how PostgreSQL Write-Ahead Logs (WAL) can be tapped directly to stream database changes downstream for analytics, search indexing, or read replicas without touching application code.

## Decision
We introduce **Kafka Connect with PostgreSQL CDC via Debezium** in **Phase 5**:

1. **Infrastructure Sequencing:**
   - Kafka Connect is deployed via local Docker Compose alongside the Phase 4 Kafka broker.
   - Debezium PostgreSQL Connector is used as the primary source connector.
2. **WAL-Based Logical Replication:**
   - PostgreSQL is configured for logical decoding:
     ```ini
     wal_level = logical
     max_replication_slots = 4
     max_wal_senders = 4
     ```
   - Debezium taps into PostgreSQL logical replication streams via the `pgoutput` plugin.
3. **Outbox and CDC Integration:**
   - Rather than relying on custom application polling, Debezium streams events directly from the database outbox table or designated domain tables into Kafka topics with millisecond latency.
   - Tombstone events and event routing are managed declaratively using Debezium Single Message Transforms (SMTs), specifically the Debezium Outbox Event Router.
4. **Failure Handling and Schema Management:**
   - Connector configurations, replication slot monitoring, offset tracking, and Dead Letter Queues (DLQ) for malformed events are formally implemented and documented as core Phase 5 learning deliverables.

## Alternatives Considered
- **Permanent Application-Level Polling Worker:** While simple in Phase 4, relying on polling in production-scale systems introduces query overhead and table vacuum pressure. Retaining only polling misses the chance to learn enterprise CDC patterns.
- **PostgreSQL Triggers with External HTTP / Webhook Calls:** Rejected because executing network I/O inside database triggers holds locks open, severely impairs transaction throughput, and crashes the database if the endpoint is unresponsive.

## Consequences
### Positive
- True zero-polling CDC directly from the PostgreSQL WAL log.
- Teaches enterprise data engineering tools: Kafka Connect, Debezium, replication slots, and CDC sinks.
- Near-instantaneous event propagation to downstream Kafka topics.
- Decouples downstream event distribution entirely from the core Node.js application process.

### Liabilities
- Requires PostgreSQL logical replication configuration and dedicated replication slots.
- Unconsumed replication slots can prevent PostgreSQL from truncating WAL segments, necessitating careful disk space monitoring.
- Docker Compose resource footprint increases (Kafka broker + Kafka Connect worker).

