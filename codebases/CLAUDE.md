# Track B — Fresh Build

You are working in Track B: a fresh CC-LMS build, used as the lab for fully automated, Claude-driven workflows.

## Stack (proposed — lock during initial setup with an ADR)

- **Frontend:** Next.js (App Router)
- **Backend:** NestJS
- **Database:** PostgreSQL via Prisma
- **Validation:** Zod (shared between FE and BE)
- **API:** REST with NestJS OpenAPI; FE consumes a generated typed client
- **Queues:** BullMQ + Redis
- **Cache:** Redis
- **File storage:** S3-compatible (Cloudflare R2 first choice)
- **Auth:** NestJS Passport + JWT (access + refresh)
- **Monorepo:** pnpm workspaces + Turborepo
- **Tests:** Jest (unit) + Playwright (E2E)
- **Observability:** Pino + OpenTelemetry-ready

## Track-B operating rules

1. **This is the long-term product.** Take time for correctness over speed.
2. **Spec-driven only.** Never start code without a brief in `../briefs/active/`.
3. **Schema first.** Backend tasks start by updating `prisma/schema.prisma`. Then types regenerate, then service, then controller, then tests.
4. **Types flow one way:** Prisma → Zod → NestJS DTOs → generated FE client → React components. Never define a type twice.
5. **Tests are non-negotiable.** A task is not done without unit tests covering the brief's acceptance criteria.
6. **Use subagents proactively.** Run `design-validator` before opening any UI PR. Run `code-reviewer` before opening any PR.

## Conventions

- **Money:** `Decimal(12, 2)` in Prisma; currency stored as ISO 4217 string (default `BDT`)
- **IDs:** UUID v7 (time-ordered) on every table
- **Timestamps:** `createdAt`, `updatedAt`, plus `createdBy`/`updatedBy` where SRS calls for audit
- **Soft delete:** `deletedAt` (nullable) for entities with FK references; hard delete for ephemeral records
- **Audit:** one central `audit_log` table with `entityType`, `entityId`, `actorId`, `diff` (JSONB)
- **Enums:** Prisma `enum` for closed sets; string columns with check constraints when values may evolve
- **Time:** Bangladesh Standard Time (UTC+6) for display; ISO 8601 with timezone for transport

## Folder structure (proposed)

```
track-b/
├── apps/
│   ├── web/          # Next.js
│   └── api/          # NestJS
├── packages/
│   ├── types/        # Shared Zod schemas + inferred types
│   ├── ui/           # Shared React components built on DESIGN.md
│   └── config/       # Shared eslint, tsconfig, prettier
├── prisma/
│   └── schema.prisma
└── docker-compose.yml  # Postgres + Redis for local
```

## Shared context

The workspace root has its own `CLAUDE.md` and the `docs/` and `briefs/` folders. Read those for cross-cutting context. This file is only for Track-B-specific conventions.
