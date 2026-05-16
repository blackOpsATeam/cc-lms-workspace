# docs/

Source of truth for the CC-LMS project. Both tracks consume these files.

## Layout

- `srs/` — one file per module. Requirements only.
- `design/` — design system and per-screen design specs.
- `api-contracts/` — shared FE/BE contracts (REST endpoints, request/response shapes).
- `decisions/` — Architecture Decision Records (ADRs).

## Why this lives in the repo

Anything that would normally go in a wiki or Notion doc lives here, because:
1. Claude Code reads from the filesystem — having docs in the repo means every Claude session has the latest context automatically.
2. Docs version with the code. The brief that produced a feature is in git history alongside the feature itself.
3. PRs can update docs in the same commit as code, so contracts and implementations don't drift.

## Editing rules

- **`srs/`** — manager-owned. Devs read; they don't edit.
- **`design/`** — manager + design-validator subagent. Devs reference; they don't edit.
- **`api-contracts/`** — manager owns the spec; devs update when implementing a new endpoint defined by the spec (a PR may add to the contract if a new endpoint is implemented).
- **`decisions/`** — anyone can propose an ADR; manager approves by merging.
