# ADR-0005: Single-track development — Track B only

- **Status:** accepted (supersedes ADR-0001)
- **Date:** 2026-05-18
- **Deciders:** Manager

## Context

ADR-0001 chose a two-track strategy: Track A (refine the existing legacy codebase to match `DESIGN.md` and ship demos) and Track B (fresh, ground-up build serving as the lab for a fully Claude-driven workflow). The workspace was structured to host both — shared `docs/`, shared `.claude/`, and two sibling code folders (`track-a/`, `track-b/`).

In practice, the two-track setup has not held up:

- Manager attention is one person split across two unrelated codebases, two stacks, and two sets of patterns. Throughput on either is worse than focused work on one.
- The legacy codebase's patterns make it costly to retrofit `DESIGN.md` and the new SRS conventions; meaningful changes there compete with new-build work.
- Track B is already the long-term product per ADR-0002. Effort spent on Track A delays the product rather than complementing it.
- The "specs/tooling battle-tested in B flow back to A" benefit assumed parallel velocity on both. With one manager, that benefit doesn't materialize.

A simpler structure is needed: one codebase, module-by-module, refined → designed → contracted → built.

## Decision

Run the project as a **single track** — the fresh build previously called Track B, now simply "the CC-LMS app" — and pause Track A indefinitely.

- Application code lives in `codebases/cc-lms/` (a nested git repo). No `track-a/` or `track-b/` folders exist at the workspace root.
- All workflow tooling (skills, agents, briefs, ADR, SRS, design system) targets this one codebase.
- The legacy codebase is **paused / out of scope**, not deleted. If it is reintroduced later, it lands as a sibling folder under `codebases/` (e.g. `codebases/cc-lms-legacy/`) with its own `CLAUDE.md`. No promise is made that it will be.

The stack (ADR-0002), multi-tenancy decision (ADR-0003), and build order (ADR-0004) remain as-is — they were always written for Track B.

## Consequences

**Good**

- Manager attention is undivided
- One codebase to maintain, one set of patterns, one CI
- The workspace structure now matches reality — root `CLAUDE.md` and `README.md` describe one track, not two
- Workflow experiments don't have to be "kept safe" from a demo product; the experiment *is* the product

**Bad / trade-offs**

- Existing investment in the legacy codebase is set aside; if it ships features the new build doesn't yet, that gap stays open until the new build catches up
- Loses the original "two tracks, ideas flow between them" feedback loop — the new build has to test its own conventions in isolation

**Follow-ups**

- Workspace root `CLAUDE.md` and `README.md` updated to remove two-track framing and `track-a/` / `track-b/` references (done 2026-05-18)
- `codebases/CLAUDE.md` already reflects the single-track reality and needs no change
- ADR-0001 stays in the log as the historical record but is marked superseded by this ADR
