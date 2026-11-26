# First Antigravity Prompt - QueueLess Phase 0

We are starting QueueLess today. This workspace contains an existing project constitution and starter context. Your job is to inspect and validate this material, not redesign the project from scratch.

## Before you act
1. Read `AGENTS.md`, `PROJECT_CONTEXT.md`, `README.md`, `docs/project-context/current-status.md`, `docs/tasks/current-task.md`, and the architecture/domain/roadmap documents.
2. Inspect the actual directory tree and Git status if Git exists.
3. Do not assume application code or infrastructure exists.
4. Do not modify any files during this first response.

## Project direction
QueueLess is a portfolio-grade appointment and provider-specific live-queue platform. The initial application is a React + TypeScript frontend and Node.js + TypeScript + Express modular monolith backed by PostgreSQL and Prisma. Later phases add Redis/WebSockets, Kafka, Kafka Connect/CDC, workers/notifications, application containers, GitHub Actions, AWS, Terraform, observability, and local Kubernetes/Helm.

`AGENTS.md` contains the detailed constitution and domain rules. Treat it as the baseline. If a summary conflicts with it, report the conflict instead of silently deciding which wins.

## Phase 0 boundary
This is a READ-ONLY audit. Do not:
- edit, delete, rename, or create files;
- generate app code or package manifests;
- install dependencies;
- create Prisma schemas/migrations or initialize a database;
- start services or provision infrastructure;
- advance to Phase 1.

## Report
Return:
1. Workspace inventory (actual files found).
2. Existing state: documented, implemented, and unknown.
3. Requirements and architecture understood.
4. Contradictions, gaps, and risks, with severity and affected files.
5. Whether the starter documents duplicate or conflict with `AGENTS.md`.
6. A minimal, file-by-file remediation plan.
7. Phase 0 exit checklist.
8. One recommended next action.

Wait for my explicit approval before changing any files. Do not claim tests or commands were run unless they were actually run.
