# Backend rules

Applies in addition to the root `AGENTS.md`.

- Node.js. Keep dependencies minimal and record each one in `docs/decisions.md`.
- `src/routes/` only parses requests and returns responses. Business logic lives in `src/services/`. Database access lives in `src/db/`.
- Any operation that changes seat state (hold, release, confirm) must run inside a MySQL transaction.
- Use parameterized queries only. Never build SQL by string concatenation.
- Validate all input at the route boundary.
- Return errors in the shape defined in `docs/03-api-contract.md`.
- Every service function that changes data needs a test in `tests/`.
