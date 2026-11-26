# ADR-003: React Single-Page Application from Day 1

- **Status:** Accepted
- **Date:** 2026-11-02
- **Approval:** Owner Approved

## Context
QueueLess is fundamentally a backend-focused portfolio project emphasizing Node.js, PostgreSQL concurrency, Kafka event-driven architectures, and cloud DevOps. However, a backend-only CRUD API without a visible interface makes demonstrating live queue progression, patient waiting room states, and clinical call workflows difficult for non-technical stakeholders and interviewers.

## Decision
QueueLess will include a **React Single-Page Application (SPA)** from Day 1 (`apps/web`) within the monorepo, developed using:
- **Build Tool**: Vite with TypeScript.
- **Styling & UI**: Tailwind CSS and shadcn/ui.
- **Routing**: React Router (v6+).
- **Server State Management**: TanStack Query (React Query).
- **Forms & Validation**: React Hook Form with Zod schemas.

The frontend serves as a supporting product layer. Its primary role is to visualize backend capabilities, demonstrate real-time states (via polling in Phase 1, WebSockets in Phase 3), and provide an intuitive demonstration interface for Patients, Providers, and Administrators.

## Alternatives Considered
- **Backend-Only API (Postman / Swagger UI)**: Simple to build, but fails to demonstrate a complete, demonstrable end-to-end product.
- **Server-Side Rendered (Next.js / Remix)**: Adds unnecessary full-stack framework complexity (SSR, server actions) when the focus is mastering pure Node.js/Express backend engineering.
- **Global State Libraries (Redux, Zustand, MobX)**: Unnecessary complexity. Server state is managed cleanly with TanStack Query; local UI state is managed with native React hooks.

## Consequences
- **Positive**: Immediate full-stack portfolio demonstration capability; clear separation of client and server concerns; TanStack Query handles caching, loading states, and automatic refetching cleanly.
- **Negative**: Increased frontend maintenance overhead; requires paired TypeScript schema sharing across monorepo packages.

