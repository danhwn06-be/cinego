# CineGo

Movie ticket booking platform. Lightweight full-stack: vanilla HTML/CSS/JS frontend, Node.js + MySQL backend.

## Language
- Talk to the user in Vietnamese.
- Everything written to the project is in English: code, identifiers, comments, commit messages, `docs/`.
- The only Vietnamese file is `README.vi.md`. Keep it in sync with `README.md`.

## Layout
- `frontend/`  vanilla HTML/CSS/JS, ES modules, no framework, no build step
- `backend/`   Node.js API
- `database/`  MySQL migrations and seed data
- `docs/`      source of truth (read before coding)

## Read first
1. `docs/progress.md`        current state and next task
2. `docs/01-domain-spec.md`  business rules and flows
3. `docs/02-database-schema.md` and `docs/03-api-contract.md` when touching data or API

## Rules
- Frontend and backend talk only through the contract in `docs/03-api-contract.md`. Do not read backend code to build frontend, or the reverse.
- Never change the schema or the API without updating the matching doc in the same change.
- One small task per session. Update `docs/progress.md` at the end.
- Do not add a framework or dependency without recording it in `docs/decisions.md`.
- Seat availability and holds are enforced in the backend with MySQL transactions. The frontend never decides whether a seat is free.
- Prices are computed in the backend and stored on the ticket at booking time.
- Never commit secrets. Use `.env` (ignored) and keep `.env.example` current.

## Commands
Not set up yet. Add them here once `backend/package.json` exists (run server, migrate, seed, test).