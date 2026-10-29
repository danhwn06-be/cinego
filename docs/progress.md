# Progress

Agents read this file at the start of every session and update it at the end.

## Current phase

**Phase 0: Foundation.** Domain specification v0.1 is written. Next is the database schema.

## Done

- Researched booking platforms (flows, data model, seat hold mechanism).
- Chose lightweight full-stack direction (see `decisions.md`).
- Created folder tree, `AGENTS.md` (root, backend, frontend), `.gitignore`, `.env.example`.
- Switched all project files to English; added bilingual README (`README.md`, `README.vi.md`).
- Wrote `docs/01-domain-spec.md` v0.1: roles, booking flow, business rules with stable IDs, entities, lifecycles, glossary, sources. Facts are tagged Verified, Decision or Assumption.
- Recorded D-006 to D-008 in `decisions.md`.

## Next

1. Design the MySQL schema in `docs/02-database-schema.md`. It must cover BR-SEAT-1 to BR-SEAT-8 (derived seat state, all-or-nothing hold, double-booking prevention with a transaction and a database constraint).
2. Write the API contract in `docs/03-api-contract.md`.
3. Initialize `backend/package.json` and fill in the Commands section of `AGENTS.md`.
4. Build the core in the backend first: hold seats, release, confirm, cancel, with tests for the concurrency rules.
5. Build the frontend against the API contract.

## Blocked

Nothing.

## Open questions

See section 8 of `01-domain-spec.md` (six questions, each with a default).

Not verified at source: CGV's real hold time, refund terms and ticket rules (CGV's own pages block automated access). The spec marks these as Decision or Assumption.

Still to do later: concrete `.agents/rules/` and `.agents/workflows/` once the working process is stable.