# CC-LMS Workspace — Claude Instructions

You are working in the CC-LMS (Coaching Center LMS) workspace. One codebase, module-by-module: refine the SRS, design the screens, agree on the API contract, then build.

## Project context

- **Domain:** Coaching center management for Bangladesh — students, teachers, admin, accountant roles
- **Market:** Bangladesh. Bilingual English + Bangla. Mobile-heavy for students/teachers (often low-end Android). Manual cash/MFS payments are normal, not edge cases.
- **Stack:** Next.js (frontend), NestJS (backend), PostgreSQL via Prisma — see `codebases/CLAUDE.md` and ADR-0002
- **Brand:** Institutional, trustworthy, bank-grade calm — see `docs/design/DESIGN.md`

## Workflow (per module)

```
SRS  →  Design  →  API contract  →  Brief  →  Build (in codebases/cc-lms/)  →  PR review  →  Merge
 |       |          |                |
docs/   docs/      docs/            briefs/
srs/    design/    api-contracts/   active/
```

One module at a time. SRS refinement happens just-in-time, one module ahead of its build phase. See ADR-0004 for build order.

## Source of truth

| Concern | Location |
|---|---|
| Requirements (per module) | `docs/srs/<module>.md` |
| Design system | `docs/design/DESIGN.md` |
| Per-screen design specs | `docs/design/screens/<module>/` |
| Component patterns | `docs/design/patterns/` |
| API contracts | `docs/api-contracts/<module>.md` |
| Decisions log | `docs/decisions/<NNNN>-<name>.md` |
| Current task briefs | `briefs/active/` |
| Brief templates | `briefs/templates/` |
| Application code | `codebases/cc-lms/` (separate git repo) |

Never duplicate or restate content from these files. Read them.

## Folder layout

```
/
├── CLAUDE.md                  # this file — workspace-level instructions
├── docs/                      # SRS, design, API contracts, ADRs
├── briefs/                    # templates + active task briefs
├── .claude/                   # skills, agents, settings
└── codebases/
    ├── CLAUDE.md              # code-level conventions (stack, schema, tests)
    └── cc-lms/                # the Next.js + NestJS + Prisma app (nested git repo)
```

Docs, briefs, and ADR commits happen from the workspace root. Code commits happen from inside `codebases/cc-lms/`.

> A second codebase (legacy Track A) was previously planned but is paused / out of scope. If reintroduced, it goes in a sibling folder under `codebases/` with its own `CLAUDE.md`. See ADR-0001 for the original rationale.

## Available skills

Skills auto-activate based on context (file paths, request content):

- `cc-lms-srs-author` — refining or authoring SRS documents in `docs/srs/`
- `cc-lms-brief-author` — writing task briefs in `briefs/active/`

More will be added as the project grows (component authoring, Prisma schema, etc.).

## Available subagents

Invoke explicitly: "Use the `<agent-name>` subagent to ..."

- `srs-reviewer` — read-only review of SRS files for completeness, clarity, ambiguity
- `design-validator` — checks design specs against `docs/design/DESIGN.md` rules
- `brief-author` — generates task briefs from SRS + design + API contract
- `code-reviewer` — read-only review of code changes against a brief

## Style and conventions

- Markdown is the project format for docs. Tables for matrices, prose for flow.
- **Money:** `৳` then single space then amount with thousands separator (`৳ 3,500.00`)
- **Dates:** `11 May 2026` format (day-month-year, no leading zero on day)
- **Numbers in BD content:** Western Arabic numerals (0-9), never Bengali numerals
- **Phone numbers:** `+880 1712 345 678` (space-separated after country code)
- **FR IDs:** `FR-<MODULE-ABBR>-NNN`, e.g. `FR-NB-001`, `FR-DB-001`

## What to do when you're unsure

1. Read the SRS for the module you're working on
2. Read DESIGN.md for any UI work
3. Check `docs/decisions/` for prior decisions
4. Ask the manager rather than guessing on scope
