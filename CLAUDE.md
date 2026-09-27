# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
EventHub is a full-stack event ticket booking platform built for QA training. Users can browse events, book tickets, manage bookings, and create events. Each user operates in an isolated sandbox.

- **Frontend**: Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS, React Query v5 (port 3000)
- **Backend**: Express.js, Prisma 5, MySQL 8+, JWT auth (7-day expiry), bcryptjs (port 3001, Swagger at `/api/docs`)
- **Testing**: Playwright E2E, Chromium only

## Commands
Run from the repo root unless noted.

```bash
npm run setup        # Install deps in backend/ and frontend/ (root deps: npm install)
npm run dev          # Frontend + backend concurrently
npm run db:push      # Push Prisma schema to DB (non-interactive)
npm run migrate      # prisma migrate dev (interactive, creates migration files)
npm run seed         # Seed 10 static events
npm run build        # Build the Next.js frontend
npm run lint         # next lint (frontend only)
(cd frontend && npx tsc --noEmit)   # Type-check; CI runs this
(cd backend && npx prisma format)   # CI fails if schema.prisma is not formatted

npm run test         # All Playwright tests
npm run test:ui      # Playwright UI mode
npx playwright test tests/<file>.spec.js --reporter=line   # Single file
npx playwright test -g "TC-001" --reporter=line            # Single test by title
docker compose run --rm tests                              # Run the suite in the Playwright image
```

Local setup needs `backend/.env` (`DATABASE_URL`, `PORT`, `CORS_ORIGIN`, `JWT_SECRET`, `SHOW_EXPLORE_LINKS`; see `backend/.env.example`) and `frontend/.env.local` (`NEXT_PUBLIC_API_URL=http://localhost:3001/api`). README.md has the full setup walkthrough. The backend has no lint or unit-test setup.

## Architecture

### Backend (`backend/`, plain JavaScript/CommonJS)
`server.js` connects Prisma and calls `app.listen`; `app.js` wires middleware and routes (`/api/auth`, `/api/events`, `/api/bookings`, plus `/api/health` and `/api/config` feature flags). Layering is Routes → Controllers → Services → Repositories (Prisma). Things that span multiple files:

- **Response envelope**: `{ success, data | error }`. Services throw the typed errors in `src/utils/errors.js` (`NotFoundError`, `ForbiddenError`, `ValidationError`, `InsufficientSeatsError`); `middleware/errorHandler.js` maps them and Prisma codes (`P2002`/`P2025`/`P2003`) to HTTP statuses. New error types must be registered in that handler.
- **Per-user sandboxing lives in the repositories/services, not in the DB**: event queries return `isStatic: true` rows plus rows where `userId` matches. Static (seeded) events are immutable. Bookings are scoped by `userId`, and cross-user access throws `ForbiddenError`.
- **Seat counts are computed per user**: `availableSeats` in the DB is not decremented on booking. `eventService.withPersonalSeats` subtracts the current user's booked quantity on read, and `bookingService.createBooking` checks against the same computed value. Cancelling just deletes the booking. The one exception is the FIFO same-event fallback, which permanently burns seats via `eventRepository.decrementSeats`.
- **FIFO pruning**: `eventService` caps user-created events at 6, and `bookingService` caps bookings at 9 (prefers pruning a booking for a *different* event).
- **Booking ref** is `<first letter of event title uppercased>-<6 random chars>`, generated in `bookingService`.
- Auth: `middleware/authMiddleware.js` verifies the Bearer JWT and sets `req.user` (`userId`, `email`). `JWT_SECRET` has a hard-coded dev fallback in `config/env.js`.

### Frontend (`frontend/`)
App Router pages in `app/` (`events`, `bookings`, `admin/events`, `admin/bookings`, `login`, `register`); components grouped by domain in `components/`. There are **two parallel API client layers** in `lib/api/`:
- `client.js` + `authApi.js`/`eventsApi.js`/`bookingsApi.js`: axios with a request interceptor that attaches the JWT from `localStorage['eventhub_token']`. Used by the React Query hooks in `lib/hooks/` and by `useAuth`.
- `client.ts` + `events.ts`/`bookings.ts`/`index.ts`: typed `fetch` wrapper with **no auth header**. Used directly by `app/bookings/page.tsx`. Check which layer a page uses before changing an endpoint call, since a change to one may not affect the other.

`client.js` and `client.ts` both export `BASE_URL` from `NEXT_PUBLIC_API_URL`.

### Playwright tests (`tests/`)
Tests run against the **live site** (`https://eventhub.rahulshettyacademy.com`), not localhost, so backend or frontend changes are not exercised locally unless you edit the URL. `playwright.config.ts` sets `baseURL` to that site, but the existing spec hardcodes its own `BASE_URL`. Config: sequential (`fullyParallel: false`), no retries, 30s test / 5s expect timeouts, HTML reporter, screenshot/video kept on failure. Tests share one account, so each begins by logging in and clearing that user's bookings via the "Clear all bookings" button.

### CI/CD (`.github/workflows/`)
- `ci.yml` (PRs to `main`, and reusable): backend syntax check, `prisma validate`/`format`/`generate`, a schema-drift check over SSH against the production DB, and frontend `tsc --noEmit` + `next build`.
- `deploy.yml` (push to `main`): runs `ci.yml` as a gate, then SSHes to the server, writes `.env` files from secrets, pulls, builds, and reloads pm2. **Pushing to `main` deploys to production.**
- `playwright.yml` (push to `main`): runs the suite against the live site with `continue-on-error`, so a red test does not fail the workflow. Check the uploaded HTML report.

## Claude Code Skills (`.claude/skills/`)
Slash-invoked agents (each has `disable-model-invocation`):
- `/generate-tests <feature>`: writes and validates Playwright tests
- `/review-tests <file>`: reviews test code quality
- `/create-scenarios <area>`: creates test scenario documents (output in `docs/test-scenarios.md`)
- `/test-strategy <scenarios>`: assigns tests to pyramid layers (output in `docs/test-strategy.md`)

Reference skills loaded automatically when relevant (`user-invocable: false`): `eventhub-domain` (business rules, API reference, UI selectors, user flows) and `playwright-best-practices`. Consult them before writing or reviewing tests.

## Testing Conventions
- Test files: `tests/<feature-name>.spec.js` (JavaScript, not TypeScript)
- Locator priority: `data-testid` > role > label/placeholder > ID > CSS class
- No `page.waitForTimeout()`; use `expect().toBeVisible()`
- Tests must be self-contained (login → action → assert), with step comments
- Test account: `rahulshetty1@gmail.com` / `Magiclife1!`

## Key Business Rules
- Max 6 user-created events and 9 bookings per user (FIFO pruning on overflow)
- Booking ref first character = event title first character (uppercase)
- Refund eligibility: 1 ticket = eligible, >1 ticket = not eligible (client-side only)
- Cross-user booking access returns "Access Denied" in the UI (backend responds 403)
- Static (seeded) events are immutable

## Code Style
- Backend: JavaScript with JSDoc, Express patterns
- Frontend: TypeScript, React hooks, Tailwind utility classes
