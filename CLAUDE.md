# CC-LMS Workspace — Claude Instructions

You are working in a shared workspace for the CC-LMS (Coaching Center LMS) project. Two codebase tracks share documentation and tooling.

## Project context

- **Domain:** Coaching center management for Bangladesh — students, teachers, admin, accountant roles
- **Market:** Bangladesh. Bilingual English + Bangla. Mobile-heavy for students/teachers (often low-end Android). Manual cash/MFS payments are normal, not edge cases.
- **Stack:** Next.js (frontend), NestJS (backend), PostgreSQL (database)
- **Brand:** Institutional, trustworthy, bank-grade calm — see `docs/design/DESIGN.md`

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

Never duplicate or restate content from these files. Read them.

## Tracks

- `track-a/` — existing codebase being refined to match DESIGN.md
- `track-b/` — fresh build, used as the lab for automated workflows

**Never mix code across tracks.** If you're in `track-a/`, edits go to `track-a/`. Track-specific `CLAUDE.md` files exist in each track folder for stack details and conventions.

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
