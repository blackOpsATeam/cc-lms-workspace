# ADR-0004: Track B build order and Dashboard strategy

- **Status:** accepted
- **Date:** 2026-05-15
- **Deciders:** Manager

## Context

Track B is a fresh build. Modules cannot be built in SRS document order — many depend on others. In particular, the Dashboard module is a *presentation layer* that aggregates data from other modules; it has nothing to display until those modules exist.

The team wanted to "do the Dashboard first." Taken literally in a fresh build, this is impossible — there is no auth, no users, no classes, no attendance to aggregate. A resolution was needed that honors the intent without the contradiction.

## Decision

### Build order — by dependency, not by SRS order

- **Phase 0 — Foundation:** monorepo scaffold (pnpm + Turborepo), Prisma + Postgres + Redis via Docker, CI pipeline, design tokens implemented in `packages/ui`.
- **Phase 1 — Auth + Dashboard shell:** Auth module (login, JWT, refresh, RBAC); Dashboard *shell only*.
- **Phase 2 — Core domain:** Class/Batch/Subject Context, User Management, Admission.
- **Phase 3 — Feature modules:** Attendance, Notice Board, Notification, Class Scheduling, Assessment, and the remaining modules — built in dependency order.

### Dashboard — shell first, widgets later

The Dashboard is **not** a single block of work. It is split:

- **Shell (Phase 1):** the empty role-based frame — login lands here, role-based layout, the widget framework, deep-link routing, empty states, and the "temporarily unavailable" state for absent data sources. This needs only Auth.
- **Widgets (Phase 3 onward):** each feature module ships *with its own dashboard widget*. Attendance ships → the attendance-percentage card appears. Notice Board ships → the communication-snapshot widget appears.

This matches the SRS principle "Dashboard owns presentation, source modules own truth" and the SRS requirement for empty / unavailable widget states.

## Consequences

**Good**

- No dependency contradictions — every module is built after what it needs
- The Dashboard shell appears early (so login has a destination) without waiting on every source module
- Each module's PR is self-contained: feature + its dashboard widget together

**Bad / trade-offs**

- The Dashboard is never "done" until the last feature module ships — progress on it is incremental and spread across many PRs
- Requires discipline: every feature brief must include "and its dashboard widget" in scope

**Follow-ups**

- SRS refinement happens just-in-time, one module ahead of its build phase
- The first brief to write is the Phase 0 scaffold brief (no SRS needed — driven by ADR-0002 and DESIGN.md)
