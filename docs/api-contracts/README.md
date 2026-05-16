# docs/api-contracts/

Shared FE/BE contracts. One file per module. Frontend and backend devs both read the same file when implementing.

## Layout

- `_template.md` — starting structure for any new module
- `<module>.md` — endpoints for that module

## How this folder relates to code

Track B will generate an OpenAPI spec from NestJS controllers; the FE client is generated from that spec. The markdown files here are the **design-time** contract — what we agree should exist — before NestJS code is written. Once the NestJS controllers exist, they become the runtime source of truth, and these markdown files should be updated to match if they drift.

For Track A, where the codebase already exists, these markdown files document the intended contract so both tracks can stay in sync.

## Filling out the template

For each endpoint:

1. State which FR IDs it satisfies (from `docs/srs/<module>.md`)
2. Method + path
3. Auth requirements
4. Request body schema (JSON example)
5. Response body schema (JSON example) for the happy path
6. Error responses with status code + error code + meaning
7. Notes on rate limiting, idempotency, pagination if applicable

If a contract changes, update this file **in the same PR** as the code change.
