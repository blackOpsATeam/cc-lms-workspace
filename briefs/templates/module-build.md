# Brief: <Module> — <Specific task>

**Track:** A | B
**Layer:** Frontend (Next.js) | Backend (NestJS) | Both
**Owner:** <dev name>
**Estimated complexity:** S (≤1 day) | M (1–3 days) | L (split this brief)

## Context

<One paragraph explaining why this task exists and how it fits into the module. What is the user-facing outcome?>

## Inputs

- SRS section: `docs/srs/<module>.md` § <section number>
- Design spec: `docs/design/screens/<module>/<screen>.md`
- API contract: `docs/api-contracts/<module>.md` § <endpoint name>
- Related code (if extending existing): `track-<x>/<path>`
- Related decisions: `docs/decisions/<NNNN>-<name>.md` (if applicable)

## Scope

**In scope**

- ...

**Out of scope** (explicit — don't touch)

- ...

## Acceptance criteria

- [ ] FR-XX-001 satisfied — <restated as a testable outcome>
- [ ] FR-XX-002 satisfied — ...
- [ ] `design-validator` subagent returns `PASS` on the new/changed screens
- [ ] All new endpoints documented in `docs/api-contracts/<module>.md`
- [ ] Unit tests cover happy path + at least 3 edge cases from SRS § 12
- [ ] Lint and typecheck clean
- [ ] No new dependencies (or new dep documented in `docs/decisions/`)

## Outputs (expected files)

- Code: `<paths>`
- Tests: `<paths>`
- Updated docs (if API contract changes): `<paths>`
- PR with description filled per project template

## Skills/subagents to use

- Skill: `cc-lms-component` (for any UI work) — *to be added*
- Subagent: `design-validator` (before opening PR — UI work)
- Subagent: `code-reviewer` (self-review pass before opening PR)

## Out-of-scope (reiterated as guardrails)

- Do not touch the other track
- Do not modify `docs/srs/` or `docs/design/`
- Do not add new dependencies without an ADR
- Do not change unrelated files in the same PR

## Definition of done

- All acceptance criteria checked
- `code-reviewer` returns `READY FOR PR` or `READY WITH NOTES`
- `design-validator` PASS (UI work)
- CI green
- PR opened with links to: this brief, the SRS section, the design spec, the API contract
