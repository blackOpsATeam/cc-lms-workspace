---
name: cc-lms-srs-author
description: Use when refining or authoring an SRS document in docs/srs/ for the CC-LMS project. Encodes the section structure, FR format, data model conventions, and depth expected so every module's SRS is consistent.
---

# CC-LMS SRS Authoring

## When to use

Activate whenever working with files in `docs/srs/`. This skill keeps every module's SRS structurally consistent so design, API contracts, and briefs can be derived predictably.

## Section structure (in order)

Every module SRS uses these 16 sections:

1. **Purpose** — one paragraph
2. **Scope** — In Scope / Out of Scope bullets
3. **Actors** — list of roles + System
4. **Core Concepts** — domain vocabulary specific to the module
5. **Functional Requirements** — `FR-<MODULE>-NNN: System shall ...`, grouped by sub-area
6. **Business Rules** — invariants and constraints
7. **User Flow / Process Flow** — numbered steps per scenario
8. **Data Model** — entities as markdown tables (Field | Type | Description)
9. **API Contracts** — endpoints with example request/response in code blocks
10. **UI Components** — screen-by-screen breakdown
11. **Validation Rules** — input and state validation
12. **Edge Cases** — known awkward scenarios (aim for 8+)
13. **Dependencies** — other modules this one consumes
14. **Non-Functional Requirements** — performance, accessibility, scale
15. **Assumptions** — explicit assumptions
16. **Open Questions** — questions still to resolve

Use `docs/srs/_template.md` as the starting structure.

## FR conventions

- IDs use the module abbreviation: `FR-NB-001` (Notice Board), `FR-DB-001` (Dashboard), `FR-AUTH-001` (Auth), `FR-ATT-001` (Attendance), etc.
- Every FR starts with "System shall …" or "<Actor> shall …"
- Each FR must be **testable** — a reviewer can write a pass/fail check from it
- No vague terms: avoid "etc.", "as needed", "where applicable" unless followed by a specific rule
- Group FRs into sub-areas (5.1, 5.2, ...) for readability

## Data model conventions

- Use markdown tables, not JSON schemas, for entity definitions
- Every field has: name, type, description
- Standard fields on most entities: `id (UUID)`, `created_at`, `updated_at`, `deleted_at (nullable)` where soft delete applies
- Money fields: `Decimal(12,2)`; currency stored as ISO code (default `BDT`)
- Status fields: enum with explicit value list

## API contract conventions

- Show the HTTP method and path in a code block
- Show request body as JSON example
- Show response body as JSON example
- List error responses with status codes and meaning
- Reference the FR IDs the endpoint satisfies

## What to flag proactively

When refining an SRS, surface these without being asked:

- Missing actors (FR references an actor not in section 3)
- FRs without clear acceptance criteria
- Data model fields not referenced in any FR
- Entities referenced in FRs but missing from the data model
- API endpoints implied by FRs but missing from section 9
- Dependencies on modules not yet present in `docs/srs/`
- Open questions left unanswered across multiple revisions
- Edge cases obvious for this domain but missing (network failure, concurrent edit, deleted referenced entity, batch change mid-flow)

## Bangladesh-context reminders

- Currency is BDT; format `৳ <amount>`
- Phone numbers `+880XXXXXXXXXX` (Bangladesh mobile starts with 1)
- Names and addresses may contain Bangla text; layout must allow expansion
- SMS is a primary communication channel, not a fallback
- Manual cash/MFS payments are standard, not edge cases
