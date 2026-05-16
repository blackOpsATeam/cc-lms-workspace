---
name: cc-lms-brief-author
description: Use when writing or refining a task brief in briefs/active/ for the CC-LMS project. A brief bundles an SRS section, design spec, API contract, and acceptance criteria into a dev-ready unit of work.
---

# CC-LMS Brief Authoring

## When to use

Activate whenever creating or editing files in `briefs/active/` or `briefs/templates/`. Use the templates in `briefs/templates/` as starting points.

## Brief filename convention

```
<module>-<short-task>-<track>.md
```

Examples:
- `notice-board-list-view-a.md`
- `dashboard-student-shell-b.md`
- `attendance-roll-call-bulk-a.md`

## A complete brief must include

1. **Header metadata** — Track (A or B), Layer (FE/BE/Both), Owner, Complexity
2. **Context** — one paragraph: why this task exists and how it fits the module
3. **Inputs** — explicit file paths the dev needs to read first:
   - SRS section: `docs/srs/<module>.md § <section>`
   - Design spec: `docs/design/screens/<module>/<screen>.md`
   - API contract: `docs/api-contracts/<module>.md § <endpoint>`
4. **Scope** — In Scope / Out of Scope, both explicit
5. **Acceptance criteria** — checklist tied to FR IDs (each criterion testable)
6. **Outputs** — expected file paths to be created or modified
7. **Skills/subagents to use** — name them explicitly so the dev's Claude session loads them
8. **Definition of done** — tests pass, lint clean, design-validator PASS, PR opened

## Quality rules

- Never invent requirements the SRS doesn't state. If the SRS is unclear, return the brief as a DRAFT and list ambiguities at the top.
- Every acceptance criterion must map to an FR ID (or a stated business rule).
- Scope statements must be exhaustive — if it's not in "In Scope" or explicitly "Out of Scope", the brief is incomplete.
- Briefs should be readable in 5 minutes. If yours runs longer, split it.

## When to split a brief

A single brief should be:
- Completable in 1–3 days by one dev
- Mergeable as a single PR
- Reviewable in under 30 minutes

If a task is larger, split into sub-briefs with explicit ordering:
- `notice-board-list-view-a.md`
- `notice-board-detail-view-a.md`
- `notice-board-create-flow-a.md`
- `notice-board-audience-targeting-a.md`

## What not to do

- Don't include implementation details ("use a useState hook"). The brief states what; the dev decides how.
- Don't reference closed/archived briefs without explaining the link.
- Don't write briefs that span both tracks. One brief = one track.
- Don't omit "Out of Scope" — it's the most useful section for keeping work focused.
