# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EventHub is a full-stack event ticket booking app built as a QA/test-automation training target. The repo has three parts: a Next.js 14 frontend (`frontend/`, port 3000), an Express + Prisma + MySQL API (`backend/`, port 3001, Swagger at `/api/docs`), and a Playwright E2E suite (`tests/`) at the root.

## Commands

All from the repo root unless noted:

```bash
npm run setup        # npm install in backend/ and frontend/ (root deps: npm install)
npm run dev          # backend (nodemon) + frontend (next dev) via concurrently
npm run db:push      # push Prisma schema to MySQL (non-interactive)
npm run migrate      # prisma migrate dev (interactive, needs a TTY)
npm run seed         # seed the 10 static events
npm run build        # next build (frontend)
npm run lint         # next lint (frontend only; backend has no linter)

npm test                                                        # all Playwright tests
npx playwright test tests/booking-management.spec.js --reporter=line   # one file
npx playwright test -g "test name substring" --reporter=line           # one test
npm run test:ui      # Playwright UI mode
npm run test:report  # open last HTML report

docker compose up --build   # run the Playwright suite in the official Playwright image
```

Env setup: `backend/.env` needs `DATABASE_URL`, `PORT`, `NODE_ENV`, `CORS_ORIGIN`, and `JWT_SECRET` (see `backend/.env.example`); `frontend/.env.local` needs `NEXT_PUBLIC_API_URL=http://localhost:3001/api`.

## Important: tests run against the live site

`playwright.config.ts` sets `baseURL` to `https://eventhub.rahulshettyacademy.com`, and the existing spec also hardcodes `BASE_URL` to that site. Tests **do not** exercise the local app unless you change the config. Chromium only, `fullyParallel: false`, 0 retries, 30s test / 5s expect timeouts. The Docker image only copies `playwright.config.ts` and `tests/`, so it also targets the live site.

Test account: `rahulshetty1@gmail.com` / `Magiclife1!`.

## Architecture

**Backend** (`backend/`, CommonJS JavaScript): `server.js` boots `app.js`, which mounts `/api/auth`, `/api/events`, `/api/bookings` plus `/api/health` and Swagger. Requests flow through a strict layered structure: routes (with Swagger JSDoc and `express-validator` validators) → controllers (thin) → services (business rules, transactions) → repositories (Prisma only). Services throw domain errors from `src/utils/errors.js` (`NotFoundError`, `ForbiddenError`, `ValidationError`, `InsufficientSeatsError`, ...) and `middleware/errorHandler.js` maps them to HTTP responses. Event and booking routes are protected by `middleware/authMiddleware.js` (Bearer JWT, which puts `{ userId, email }` on `req.user`).

**Per-user sandbox.** Every user sees an isolated view of the data. This is implemented across several layers, so keep it in mind when changing queries:
- Repositories filter events by `isStatic: true OR userId = me`. Seeded events are static (shared and immutable); user-created events are private to their creator.
- `eventService.withPersonalSeats` recomputes `availableSeats` for each user by subtracting only *that user's* bookings, so one user's bookings never change the counts another user sees.
- Services enforce ownership: you can't update or delete static or other users' events, and you can't read other users' bookings (the UI shows this as "Access Denied").
- FIFO caps: at most 6 user-created events (`MAX_USER_DYNAMIC_EVENTS`) and 9 bookings (`MAX_USER_BOOKINGS`) per user. When a user goes over the cap, the oldest one is deleted. For bookings, the service prefers deleting a booking for a *different* event.

Data model (`backend/prisma/schema.prisma`): `User` 1–N `Event` (nullable `userId` for static events), `User` 1–N `Booking`, `Event` 1–N `Booking`. All relations cascade on delete. `bookingRef` is unique, and its first character matches the event title's first letter.

**Frontend** (`frontend/`, mixed TS/JSX): App Router pages under `app/` (events, bookings, admin, login, register). Data fetching uses React Query hooks in `lib/hooks/` wrapping axios API modules in `lib/api/`. `lib/api/client.js` attaches the JWT from `localStorage['eventhub_token']` and clears it on 401. Auth state lives in `lib/hooks/useAuth.tsx`. Key UI elements have `data-testid` attributes (the full list is in README and the `eventhub-domain` skill).

## CI/CD (`.github/workflows/`)

- `ci.yml` (PRs to main, also reused by deploy): backend `node --check` syntax pass, Prisma validate / format-check / generate, a schema-drift check against production over SSH, and frontend type-check + build.
- `deploy.yml` (push to main): runs ci.yml as a gate, then deploys over SSH with PM2.
- `playwright.yml` (push to main): runs the suite against the live site and uploads the HTML report. A failing test fails the job, but the report is still uploaded (30-day retention).

## Testing Conventions

- Specs go in `tests/<feature-name>.spec.js` (JavaScript).
- Locator priority: `data-testid` > role > label/placeholder > id > CSS. Never use `page.waitForTimeout()`; rely on web-first `expect` assertions.
- Each test is self-contained (login → act → assert) because tests share one live account and the FIFO caps prune old data.
- Project skills in `.claude/skills/` hold the detailed standards and domain reference: `playwright-best-practices`, `eventhub-domain` (business rules, API, selectors, user flows). There are also agent-style skills: `/generate-tests`, `/review-tests`, `/create-scenarios`, `/test-strategy`. Generated scenario and strategy docs live in `docs/`.
