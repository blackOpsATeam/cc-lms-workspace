# ADR-0001: Two-track development strategy

- **Status:** superseded by ADR-0005 (2026-05-18)
- **Date:** 2026-05-15
- **Deciders:** Manager

> ⚠️ This decision has been **superseded**. The project now runs as a single track (the fresh build, in `codebases/cc-lms/`); Track A is paused. See [ADR-0005](0005-single-track-track-b-only.md). The content below is preserved as historical record.

## Context

The CC-LMS project has an existing codebase (built before the design system was finalized) that already covers most backend modules and ~12 of 17 frontend modules. Several modules remain unbuilt (Notice Board, Notification, Dashboard, parts of Assessment) and existing frontend doesn't match the new `DESIGN.md`.

The team wants to (a) ship a working product on the existing codebase, and (b) experiment with a fully automated, Claude-driven development workflow that can scale across many modules.

Trying to do both in the same codebase risks compromising the demo product with experimental tooling, or compromising the experiment by overlaying it on legacy patterns.

## Options considered

### Option A: One track, refine existing codebase only

- Pros: Simple, single deliverable, less coordination.
- Cons: No greenfield testbed for the automation experiment; existing patterns lock in choices.

### Option B: One track, abandon existing codebase, rebuild from scratch

- Pros: Clean slate; full benefit of new design system and automated workflow.
- Cons: Throws away substantial existing work; pushes out demo timeline.

### Option C: Two parallel tracks (chosen)

- **Track A:** Refine existing codebase — match DESIGN.md, finish unbuilt modules, ship demos.
- **Track B:** Fresh build — used as the lab for full Claude-driven workflow automation; built gradually.
- Shared workspace: SRS, design system, API contracts, briefs, Claude Code skills/agents all live at the workspace root and serve both tracks.

## Decision

Option C. Two tracks, two devs each (1 FE + 1 BE), manager (one person) oversees both. Shared workspace with one `docs/` and `.claude/` source of truth.

## Consequences

**Good**

- Existing investment in Track A is preserved
- Track B can experiment freely without risking the demo product
- Shared SRS/design/contracts means decisions only need to be made once
- Specs and tooling battle-tested in Track B can flow back to Track A

**Bad / trade-offs**

- Manager attention is split across two tracks
- Two codebases to maintain
- Shared workspace requires discipline (no cross-track edits)

**Follow-ups**

- Lock Track B stack in ADR-0002 (Prisma vs TypeORM, monorepo tooling)
- Document Track A's current stack in `track-a/CLAUDE.md` (replace the question marks)
- Decide multi-tenancy model in ADR-0003 (affects every table in Track B)
