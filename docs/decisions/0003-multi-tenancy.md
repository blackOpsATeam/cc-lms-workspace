# ADR-0003: Multi-tenancy model

- **Status:** accepted
- **Date:** 2026-05-18
- **Deciders:** Manager

## Context

CC-LMS could be deployed in two fundamentally different ways, and the choice touches every table in the database. It must be decided before any schema work.

The question: does one running instance of CC-LMS serve **one** coaching center, or **many**?

## Options considered

### Option A: Single-tenant (one deployment per coaching center)

- Each coaching center gets its own deployment and its own database.
- No `tenant_id` columns anywhere.
- Pros: simplest schema; strongest data isolation; no risk of cross-tenant leakage; simplest queries.
- Cons: operational overhead grows linearly with customers; no shared analytics; updates must roll out per deployment.

### Option B: Shared-schema multi-tenant (one deployment, many centers, `tenant_id` everywhere) — CHOSEN

- One database; every domain table carries `tenant_id`; the application enforces isolation on every query.
- Pros: one deployment to operate; easy onboarding of new centers; cross-tenant reporting possible; lowest infra cost for early customers.
- Cons: `tenant_id` discipline required on every query (a missed filter = data leak); largest blast radius if isolation fails; noisy-neighbor performance risk.

### Option C: Schema-per-tenant (one database, one Postgres schema per center)

- One database; each coaching center gets its own Postgres schema.
- Pros: middle ground — strong isolation, single deployment, per-tenant backup/restore.
- Cons: schema migrations must run across N schemas; connection routing logic needed.

## Decision

**Option B: Shared-schema multi-tenant.**

CC-LMS is shipping as a hosted SaaS where many coaching centers sign up and share infrastructure. Shared-schema is the standard SaaS choice for this model — lowest infra cost, fastest onboarding, single migration path. The leak risk is real but mitigated by Prisma middleware that injects `tenant_id` into every query plus integration tests that verify no cross-tenant access path.

## Consequences

**Schema**

- A new top-level entity: `Tenant` (one row per coaching center). Owns `id` (UUID), `name`, `slug` (URL-safe), `status` (`ACTIVE`, `SUSPENDED`, `ARCHIVED`), branding fields (logo, primary colour, contact info), `created_at`, `updated_at`. A dedicated **Tenant Management** SRS is a follow-up.
- Every domain table in every SRS carries `tenant_id` (`UUID NOT NULL`, FK → `Tenant.id`). Every uniqueness constraint that was previously "unique within the institution" becomes a partial unique index scoped by `tenant_id` (e.g. `(tenant_id, code)` on Class).
- Indexes for hot-path queries include `tenant_id` as the leading column.

**Auth**

- The JWT access token carries a `tenant_id` claim alongside `user_id`, `role`, `session_id`, `iat`, `exp` (see `auth.md` §4).
- `User` belongs to a tenant. A user cannot belong to multiple tenants in v1; multi-tenant individual accounts are a future enhancement.
- Login resolves the user's tenant from the identifier (email/phone is unique within a tenant); the tenant context is captured in the issued token and never crosses sessions.
- A future "platform super-admin" role for cross-tenant operations is **out of v1 scope**; tenant onboarding and cross-tenant reporting are operational tasks handled outside the user-facing app for now.

**Query layer**

- Prisma middleware injects `WHERE tenant_id = $caller.tenant_id` on every read and write. Direct `prisma.user.findMany()` without middleware is forbidden in application code; lints enforce.
- Every API guard verifies the caller's `tenant_id` claim matches the resource's `tenant_id`; mismatches return `404` (existence-hiding) rather than `403`.
- Internal service-to-service calls (e.g. Student Workspace resolver → Scheduling) pass the caller's `tenant_id` explicitly; service tokens encode a `tenant_id` claim too.

**Audit log**

- The central `audit_log` schema (TBD) carries `tenant_id` on every row so per-tenant audit retrieval is a single-column filter.

**Operations**

- Per-tenant retention policy (e.g. how long previous classes remain visible) is a `Tenant`-level config field.
- Backups are global (single database); per-tenant logical restore requires the `tenant_id` filter on every restore step — documented in the runbook.
- Noisy-neighbor protection: per-tenant rate limits on heavy endpoints (resolution API, schedule range queries, performance summaries) layered on top of the global rate limits already defined in each SRS.

**Testing**

- Every integration test runs a "tenant-leak" assertion: create two tenants, attempt cross-tenant reads as each tenant's users, verify all attempts return `404` or empty results.
- Prisma middleware has its own unit tests covering every model.

**Follow-ups**

- A new SRS for **Tenant Management** (creation, suspension, archival, branding, per-tenant config). Out of the current six-module batch; lands when the platform-onboarding flow is designed.
- All six SRSs in `docs/srs/` updated to remove the "single-tenant assumed" caveat and explicitly state that every table carries `tenant_id` per this ADR.
- A new ADR may be needed to capture the **Prisma tenant middleware** design choice in more depth once the implementation is drafted.
