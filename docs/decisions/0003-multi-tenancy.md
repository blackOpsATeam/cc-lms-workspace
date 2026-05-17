# ADR-0003: Multi-tenancy model

- **Status:** proposed — DECISION PENDING
- **Date:** 2026-05-15
- **Deciders:** Manager

> ⚠️ This decision is **not yet made**. It blocks the design of `prisma/schema.prisma`.
> Do not design tenant-scoped tables until this ADR is marked `accepted`.

## Context

CC-LMS could be deployed in two fundamentally different ways, and the choice touches every table in the database. It must be decided before any schema work.

The question: does one running instance of CC-LMS serve **one** coaching center, or **many**?

## Options considered

### Option A: Single-tenant (one deployment per coaching center)

- Each coaching center gets its own deployment and its own database.
- No `tenant_id` columns anywhere.
- Pros: simplest schema; strongest data isolation; no risk of cross-tenant leakage; simplest queries.
- Cons: operational overhead grows linearly with customers; no shared analytics; updates must roll out per deployment.

### Option B: Shared-schema multi-tenant (one deployment, many centers, `tenant_id` everywhere)

- One database; every table carries `tenant_id`; the application enforces isolation on every query.
- Pros: one deployment to operate; easy onboarding of new centers; cross-tenant reporting possible.
- Cons: `tenant_id` discipline required on every query (a missed filter = data leak); largest blast radius if isolation fails; noisy-neighbor performance risk.

### Option C: Schema-per-tenant (one database, one Postgres schema per center)

- One database; each coaching center gets its own Postgres schema.
- Pros: middle ground — strong isolation, single deployment, per-tenant backup/restore.
- Cons: schema migrations must run across N schemas; connection routing logic needed.

## Decision

**PENDING.** The manager must choose based on the business model:

- Selling CC-LMS as software each center self-hosts or gets a dedicated instance → **Option A**
- Selling CC-LMS as a hosted SaaS where centers sign up and share infrastructure → **Option B** or **C**
- Expecting tens of centers, want operational simplicity but strong isolation → **Option C**

## Consequences

To be completed once the decision is made. The chosen option determines:

- Whether `tenant_id` appears on every table
- Whether every Prisma query needs a tenant scope
- The shape of the auth token (does it carry a tenant claim?)
- Connection/routing logic in the NestJS data layer
