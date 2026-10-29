# Decision Log

For each decision record: context, choice, reason. Never delete an old decision. If it changes, add a new entry and mark the old one as superseded.

## D-001: Lightweight full-stack

- **Decision:** vanilla HTML/CSS/JS frontend, Node.js backend, MySQL database.
- **Reason:** preventing double-booked seats is a server-side job. The owner already knows Node.js and MySQL. One language (JavaScript) on both sides.

## D-002: Monorepo split by area

- **Decision:** `frontend/`, `backend/`, `database/`, `docs/` in one repository, each area with its own `AGENTS.md`.
- **Reason:** clear boundaries reduce agent mistakes and context cost. Frontend and backend communicate only through `docs/03-api-contract.md`.

## D-003: Agent configuration for Antigravity

- **Decision:** root `AGENTS.md` is the main file (cross-tool format). The `.agents/` folder holds `rules/` and `workflows/` (recommended by the official docs; `.agent/` also works, and `_agents` can replace the dot for tools that avoid hidden folders). `GEMINI.md` is added only when an Antigravity-specific override is needed.
- **Verified:** there is no `.antigravityignore`. To stop an agent from reading specific files, set access restrictions in Antigravity's settings or `settings.json`, or declare the restriction in `AGENTS.md` / `GEMINI.md`.
- **Reason:** keep rule files short because they are loaded into every prompt.

## D-004: All project files in English

- **Decision:** `AGENTS.md`, everything in `docs/`, code, identifiers, comments and commit messages are in English. The only Vietnamese file is `README.vi.md`, kept in sync with `README.md`. Conversation with the owner stays in Vietnamese.
- **Reason:** one language across all agent-facing files avoids drift, lowers token cost, and matches code identifiers. A glossary in `docs/01-domain-spec.md` maps Vietnamese business terms to English identifiers.
- **Supersedes:** the earlier plan to write `docs/` in Vietnamese.

## D-005 (provisional, to be confirmed during frontend design)

- **Proposal:** frontend is an SPA using ES modules, hash-based routing, no bundler.
- **Status:** not final. Revisit when frontend work begins. The domain specification v0.1 does not change it.

## D-006: Seat state is derived, not stored as a flag

- **Decision:** a seat in a showtime is `held` only while an unexpired hold covers it; an expired hold is treated as `available` at read time. No background job is required for correctness (a cleanup job may exist only for tidiness).
- **Reason:** a flag that a job must flip creates a window where a seat is wrongly blocked or wrongly free. Deriving state from expiry timestamps removes that window. See BR-SEAT-2.
- **Consequence:** `02-database-schema.md` must make this derivable and must prevent double booking with a transaction plus a database-level constraint.

## D-007: Booking model

- **Decision:** one booking = one showtime, one ticket per seat, one booking code and QR per booking. Statuses: `pending`, `confirmed`, `expired`, `cancelled`, `checked_in`. Prices are computed by the server and frozen at payment confirmation.
- **Reason:** matches how real platforms issue one code per order, and keeps cancellation whole-booking only. See BR-BOOK-1..4, BR-PRICE-1, BR-PRICE-7.

## D-008: MVP simplifications that differ from real CGV behavior

- **Decision:** refunds are simulated back to the original payment method (CGV issues a one-month gift card and limits 2 refunds per month per account); no printed-ticket collection (QR on screen is the ticket); base price stored on the showtime instead of a pricing engine.
- **Reason:** gift cards and per-account limits need accounts and wallets, which are out of MVP. Revisit when accounts are added. See BR-CANCEL-4, BR-BOOK-5, BR-PRICE-3.