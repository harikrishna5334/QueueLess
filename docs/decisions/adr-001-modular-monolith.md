# ADR-001: Modular Monolith Architecture

- **Status:** Accepted
- **Date:** 2026-11-02
- **Approval:** Owner Approved

## Context
QueueLess is designed to demonstrate full-stack engineering, complex domain logic, relational modeling, and concurrency controls. A common architectural anti-pattern in early-stage projects is premature microservices decomposition, which introduces distributed transactions, network latency, deployment overhead, and premature operational complexity before domain boundaries are stable.

## Decision
QueueLess Phase 1 will be built as a **Modular Monolith** using Node.js, Express, and TypeScript within a `pnpm` monorepo (`apps/api`). 

The application is structured into clear vertical domain modules (`auth`, `users`, `organizations`, `locations`, `providers`, `services`, `appointments`, `queue`, `audit`). Each module isolates its controllers, use-case domain services, and database repositories. Communication between modules occurs via in-process TypeScript interfaces and atomic PostgreSQL transactions.

## Alternatives Considered
- **Microservices from Day 1**: Rejected due to high operational complexity, dual-write hazards, distributed tracing requirements, and network serialization overhead before business logic is validated.
- **Unstructured / Layer-Only Monolith ("Big Ball of Mud")**: Rejected because organizing solely by technical layers (controllers, services, models) obscures business domain boundaries and makes later event-driven extraction difficult.

## Consequences
- **Positive**: Direct ACID database transactions; zero inter-service network latency; simple local developer setup; clear domain boundaries that simplify extracting event producers in Phase 4.
- **Negative**: Requires strict discipline to prevent tight coupling between module repositories; all modules share the same runtime process and database connection pool in Phase 1.

