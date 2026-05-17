# codebases/ — Track B Application Code

This folder holds the CC-LMS application code. The project is currently running **Track B only** — the fresh, ground-up build used as the lab for a fully Claude-driven development workflow.

> Track A (the pre-existing codebase) is paused / out of scope for now. If it is reintroduced later, it goes in a sibling folder here (e.g. `codebases/cc-lms-legacy/`) with its own `CLAUDE.md`.

## Where the code lives

```
codebases/
├── CLAUDE.md          ← this file (tracked by the WORKSPACE repo)
└── cc-lms/            ← the Track B application (its OWN git repo, nested)
    ├── .git/
    ├── apps/
    │   ├── web/       # Next.js (App Router)
    │   └── api/       # NestJS
    ├── packages/
    │   ├── types/     # Shared Zod schemas + inferred types
    │   ├── ui/        # Shared React components built on docs/design/DESIGN.md
    │   └── config/    # Shared eslint, tsconfig, prettier
    ├── prisma/
    │   └── schema.prisma
    └── docker-compose.yml   # Postgres + Redis for local dev
```

The `cc-lms/` folder is a **separate git repository** from the workspace. The workspace repo does not track it (see the workspace `.gitignore`). Commit code changes from inside `codebases/cc-lms/`; commit docs/briefs/ADR changes from the workspace root.

## Stack (locked — see ADR-0002)

- **Frontend:** Next.js (App Router)
- **Backend:** NestJS
- **Database:** PostgreSQL via Prisma
- **Validation:** Zod, shared between FE and BE via `packages/types`
- **API:** REST with NestJS OpenAPI; FE consumes a generated typed client
- **Queues:** BullMQ + Redis
- **Cache:** Redis
- **File storage:** S3-compatible (Cloudflare R2 first choice)
- **Auth:** NestJS Passport + JWT (access + refresh)
- **Monorepo:** pnpm workspaces + Turborepo
- **Tests:** Jest (unit) + Playwright (E2E)
- **Observability:** Pino, OpenTelemetry-ready

## Operating rules

1. **This is the long-term product.** Prefer correctness over speed.
2. **Spec-driven only.** Never start code without a brief in `../briefs/active/`.
3. **Schema first.** Backend tasks start by updating `prisma/schema.prisma`. Then types regenerate, then service, then controller, then tests.
4. **Types flow one way:** Prisma → Zod → NestJS DTOs → generated FE client → React components. Never define a type twice.
5. **Tests are non-negotiable.** A task is not done without unit tests covering the brief's acceptance criteria.
6. **Use subagents proactively.** Run `design-validator` before any UI PR. Run `code-reviewer` before any PR.

## Conventions

- **Money:** `Decimal(12, 2)` in Prisma; currency stored as ISO 4217 string (default `BDT`)
- **IDs:** UUID v7 (time-ordered) on every table
- **Timestamps:** `createdAt`, `updatedAt`, plus `createdBy`/`updatedBy` where the SRS calls for audit
- **Soft delete:** `deletedAt` (nullable) for entities with FK references; hard delete for ephemeral records
- **Audit:** one central `audit_log` table — `entityType`, `entityId`, `actorId`, `diff` (JSONB)
- **Enums:** Prisma `enum` for closed sets; string columns with check constraints when values may evolve
- **Time:** Bangladesh Standard Time (UTC+6) for display; ISO 8601 with timezone for transport
- **Multi-tenancy:** see ADR-0003 — **decision pending**, do not design tenant-scoped tables until resolved

## Build order (see ADR-0004)

Track B is a fresh build, so modules land in dependency order, not SRS order:

- **Phase 0 — Foundation:** monorepo scaffold, Prisma + Docker, CI, design tokens in `packages/ui`
- **Phase 1 — Auth + Dashboard shell:** Auth module; Dashboard *shell* (empty role-based frame, widget framework, empty states)
- **Phase 2 — Core domain:** Class/Batch/Subject, User, Admission
- **Phase 3 — Feature modules:** Attendance, Notice Board, Notification, Scheduling, Assessment, … — **each ships with its own dashboard widget**

The Dashboard is never one big task: a shell in Phase 1, then a widget per module thereafter.

## Shared context

The workspace root has its own `CLAUDE.md` and the `docs/` and `briefs/` folders. Claude Code walks up the directory tree and reads those automatically. This file only covers code-specific conventions.
