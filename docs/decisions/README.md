# Architecture Decision Records (ADRs)

This directory documents the foundational architectural decisions for QueueLess.

## ADR Template

Each ADR follows this structure:
- **Status:** `Proposed` | `Accepted` | `Superseded`
- **Context:** What problem, requirement, or constraint prompted this decision?
- **Decision:** What architectural choice is made?
- **Alternatives Considered:** What other options were evaluated and why were they rejected?
- **Consequences:** What benefits, liabilities, and technical trade-offs result from this choice?
- **Approval:** Record of owner decision and date.

## ADR Index

| ADR ID | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [ADR-001](adr-001-modular-monolith.md) | Modular Monolith Architecture | **Accepted** | 2026-11-02 |
| [ADR-002](adr-002-postgresql-and-prisma.md) | PostgreSQL, Prisma, and Raw SQL Migrations | **Accepted** | 2026-11-02 |
| [ADR-003](adr-003-react-from-day-one.md) | React Single-Page Application from Day 1 | **Accepted** | 2026-11-02 |
| [ADR-004](adr-004-jwt-authentication.md) | JWT Authentication and Client Token Storage | **Accepted** | 2026-11-04 |
| [ADR-005](adr-005-appointment-and-queuetoken-separation.md) | Separation of Appointment and QueueToken | **Accepted** | 2026-11-02 |
| [ADR-006](adr-006-provider-specific-queues.md) | Provider-Specific Queues and Hybrid FIFO Ordering | **Accepted** | 2026-11-02 |
| [ADR-007](adr-007-kafka-introduction.md) | Deferred Kafka Introduction and Transactional Outbox | **Accepted** | 2026-11-02 |
| [ADR-008](adr-008-kafka-connect-cdc.md) | Kafka Connect and PostgreSQL Change Data Capture (CDC) | **Accepted** | 2026-11-02 |
| [ADR-009](adr-009-local-kubernetes-first.md) | Local Kubernetes First (Kind / k3s) | **Accepted** | 2026-11-02 |
