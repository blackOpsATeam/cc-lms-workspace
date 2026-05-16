# Brief: Refine SRS for <module>

**Track:** N/A (workspace docs — both tracks consume)
**Owner:** Manager + Claude
**Goal:** Take the existing draft in `docs/srs/<module>.md` to a state where design and API contracts can begin.

## Inputs

- `docs/srs/<module>.md` — current draft (or empty placeholder)
- `docs/design/DESIGN.md` — design system constraints
- Original SRS PDF — reference only, not authoritative anymore
- Related module SRS files: `docs/srs/<related>.md`

## Workflow

1. Read current draft
2. Run the `srs-reviewer` subagent → review output
3. Address gaps, ambiguities, missing edge cases
4. Lock open questions (decide or defer with reason)
5. Re-run `srs-reviewer` until it returns `PASS` or `PASS WITH NOTES`

## Skills/subagents to use

- Skill: `cc-lms-srs-author` (for section structure and style)
- Subagent: `srs-reviewer` (for completeness checks)

## Definition of done

- All 16 sections present and substantive
- Every FR is testable (clear pass/fail criterion)
- Every entity in the data model is referenced by at least one FR
- Every endpoint in API Contracts is implied by at least one FR
- No more than 3 unresolved Open Questions
- `srs-reviewer` returns `PASS` or `PASS WITH NOTES`
- Manager signs off (commit message: `docs(srs): finalize <module> SRS`)
