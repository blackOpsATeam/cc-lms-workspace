# CC-LMS Workspace

Workspace for the CC-LMS coaching center management platform. One codebase, one source of truth for SRS, design system, API contracts, briefs, and Claude Code tooling. Modules are refined → designed → contracted → built one at a time.

## For the manager

Day-to-day work happens here:

- Refining SRS in `docs/srs/<module>.md`
- Reviewing designs in `docs/design/screens/<module>/`
- Writing/reviewing task briefs in `briefs/active/`
- Logging decisions in `docs/decisions/`

Tools you'll use:

- **VS Code + Claude Code** in the integrated terminal — primary working environment
- **Claude.ai chat** (web/desktop) — strategy and planning conversations like this one
- **GitHub** — PR reviews from the devs

Open Claude Code at the workspace root for docs/specs/brief work. Open it inside `codebases/cc-lms/` when reviewing or editing application code.

## For devs

Application code lives in `codebases/cc-lms/` (its own git repo, nested inside this workspace). The workspace root `CLAUDE.md` plus `codebases/CLAUDE.md` tell Claude Code about shared docs, skills, and agents. Don't duplicate context — read from `docs/` and `briefs/active/`.

## The workflow

```
SRS  →  Design  →  API contract  →  Brief  →  Dev + Claude Code build  →  PR review  →  Merge
 |       |          |                |
 docs/   docs/      docs/            briefs/
 srs/    design/    api-contracts/   active/
```

1. Manager refines SRS in `docs/srs/<module>.md`
2. Manager + Claude generate design specs in `docs/design/screens/<module>/`
3. API contract drafted in `docs/api-contracts/<module>.md`
4. `brief-author` subagent writes a brief into `briefs/active/`
5. Dev opens Claude Code inside `codebases/cc-lms/`, points it at the brief
6. Dev opens PR; manager reviews; merges

## Folder map

```
/
├── CLAUDE.md                       # Workspace-level Claude instructions
├── docs/
│   ├── srs/                        # Source of truth: requirements per module
│   ├── design/
│   │   ├── DESIGN.md               # The design system
│   │   ├── patterns/               # Component-level specs (badge, KPI card, etc.)
│   │   └── screens/                # Per-screen design specs
│   ├── api-contracts/              # Shared FE/BE contracts
│   └── decisions/                  # ADRs (Architecture Decision Records)
├── briefs/
│   ├── templates/                  # Reusable brief templates
│   └── active/                     # Current task briefs (per dev/per task)
├── .claude/
│   ├── skills/                     # Project-specific skills
│   ├── agents/                     # Subagent personas
│   └── settings.json               # Permissions and config
└── codebases/
    ├── CLAUDE.md                   # Code-level conventions (stack, schema, tests)
    └── cc-lms/                     # The Next.js + NestJS + Prisma app (nested git repo)
```

> A previously planned second codebase (legacy Track A) is paused / out of scope. See ADR-0005 (supersedes ADR-0001). If reintroduced later, it goes in a sibling folder under `codebases/` with its own `CLAUDE.md`.

## Quick start

1. Install Claude Code: see `https://docs.claude.com/en/docs/claude-code/overview`
2. Open this folder in VS Code
3. In the integrated terminal, run `claude`
4. First thing it does: read `CLAUDE.md`. You're ready.
