# ADR-0002: Track B technology stack

- **Status:** accepted
- **Date:** 2026-05-15
- **Deciders:** Manager

## Context

Track B is a fresh, ground-up build of CC-LMS, intended both as the long-term product and as the lab for a fully Claude-driven development workflow. The stack must be modern, type-safe end to end, and friendly to Claude Code automation (declarative schemas, generated types, minimal hand-written boilerplate).

The stack family was constrained up front: Next.js frontend, NestJS backend, PostgreSQL database. The decisions below fill in the remaining choices.

## Decision

| Concern | Choice | Reason |
|---|---|---|
| ORM | **Prisma** | Declarative `schema.prisma` is one of the most Claude-friendly formats; migrations are first-class; generated client is fully typed. |
| Validation | **Zod**, shared FE/BE via `packages/types` | One schema, two consumers — eliminates type drift. |
| API style | **REST + NestJS OpenAPI**; FE consumes a generated typed client | Cuts integration bugs; contract is machine-checkable. |
| Queues | **BullMQ + Redis** | Billing cycles, notification fan-out, payroll runs, schedule reminders are all queue work per the SRS. |
| Cache | **Redis** | Dashboard snapshot cache is explicit in the SRS; also session + rate limiting. |
| File storage | **S3-compatible — Cloudflare R2** | SRS requires cloud storage with signed URLs; R2 is cost-effective for the Bangladesh market. |
| Auth | **NestJS Passport + JWT** (access + refresh) | SRS specifies JWT-based auth. |
| Monorepo tooling | **pnpm workspaces + Turborepo** | Shared `types`, `ui`, `config` packages; fast incremental builds. |
| Tests | **Jest** (unit) + **Playwright** (E2E) | Both well-supported and Claude-friendly. |
| Observability | **Pino** logging, OpenTelemetry-ready | Structured logs; APM can be added later without rework. |

## Repository layout

```
codebases/cc-lms/
├── apps/
│   ├── web/          # Next.js (App Router)
│   └── api/          # NestJS
├── packages/
│   ├── types/        # Shared Zod schemas + inferred types
│   ├── ui/           # Shared React components on docs/design/DESIGN.md
│   └── config/       # Shared eslint, tsconfig, prettier
├── prisma/
│   └── schema.prisma
└── docker-compose.yml
```

## Consequences

**Good**

- End-to-end type safety: Prisma → Zod → DTO → generated client → React
- Declarative formats (Prisma schema, Zod) maximize Claude Code reliability
- Shared `packages/` prevents duplicate type definitions

**Bad / trade-offs**

- Generated FE client adds a build step that must stay in CI
- Monorepo tooling has a learning curve for devs new to Turborepo

**Follow-ups**

- ADR-0003 must resolve multi-tenancy before `prisma/schema.prisma` is designed
- ADR-0004 records the build-order (phased) plan
