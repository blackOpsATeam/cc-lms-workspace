# docs/decisions/

Architecture Decision Records (ADRs). One file per decision, numbered sequentially: `0001-track-strategy.md`, `0002-track-b-orm.md`, etc.

## When to write an ADR

- Choosing a library, framework, or service (Prisma vs TypeORM, R2 vs S3)
- Diverging from a convention stated elsewhere (e.g. a track using a different ORM)
- A decision the team will likely re-argue if it isn't written down
- Anything in a brief that introduces a new dependency

## When NOT to write an ADR

- Day-to-day code choices (variable names, function structure)
- Anything covered by `docs/design/DESIGN.md` or an existing SRS
- Reversible micro-decisions

## Template

Use `_template.md`. ADRs are short — 1 page is usually enough.

## States

- **proposed** — written, not yet decided
- **accepted** — in effect
- **superseded** — replaced by a later ADR (link to the replacement)

Don't delete superseded ADRs. They're history.
