# EventHub — Booking Management Test Strategy

Generated: 2026-11-17
Input: `docs/test-scenarios.md` (75 scenarios, TC-001 – TC-509)
Sources scanned: `backend/src/services/bookingService.js`, `backend/src/controllers/bookingController.js`, `backend/src/routes/bookingRoutes.js`, `backend/src/validators/bookingValidator.js`, `backend/src/repositories/bookingRepository.js`, `backend/src/middleware/{authMiddleware,errorHandler}.js`, `frontend/app/bookings/**`, `frontend/app/admin/bookings/page.tsx`, `frontend/components/bookings/BookingCard.jsx`, `frontend/lib/hooks/useBookings.ts`, `frontend/lib/api/client.js`, `tests/booking-management.spec.js`

---

## 1. Layer Distribution

| Layer | Primary TCs | Incl. defense-in-depth | Focus | Approx. run time |
|---|---|---|---|---|
| **Unit** | 10 | 18 | Pure helpers (`randomRef`, refund rule), validator chain, `bookingService` branches with mocked repositories | < 2 s |
| **API / Integration** | 28 | 31 | HTTP contract, auth/ownership, FIFO cap, per-user seats, cascade deletes, real DB state | 30–60 s (live API) |
| **Component** | 26 (+2 merged) | 26 | Single-page UI states with `page.route` mocks: loading/empty/error, dialogs, refund timer, toasts | 1–2 min |
| **E2E** | 9 | 9 | Critical real-stack journeys: book → list → detail → cancel/clear, refund, cross-user Access Denied | 2–4 min |
| **Total** | **75** | | | |

**Shape:** at the primary level the distribution is more of a diamond than a pyramid. The Component layer is wide because booking management is mostly UI state, and the frontend has no unit runner today. Section 5 recommends two small extractions that move rules out of components into Unit tests. Counting defense-in-depth instances, the shape is Unit 18 → API 31 → Component 26 → E2E 9. E2E stays narrow, which is the main goal.

### Layer definitions in this repo

| Layer | Tooling (see §6 for prerequisites) | Target |
|---|---|---|
| Unit | Jest in `backend/` (CJS + `jest.mock` for repositories). Frontend pure helpers need a runner such as Vitest | Functions in isolation |
| API | Playwright `request` fixture, token from `POST /api/auth/login` | Live backend API |
| Component | Playwright page tests with `page.route('**/api/bookings**', …)` and token injected via `addInitScript`. No real backend dependency for the asserted state | Frontend pages, mocked API |
| E2E | Playwright browser, real backend, no mocks | `baseURL` in `playwright.config.ts` |

---

## 2. Layer Assignments

### 2.1 Unit (10 primary + 8 defense-in-depth)

| TC | Title | Target (source) | Why Unit |
|---|---|---|---|
| TC-100 | Ref prefix = title first letter | `randomRef(eventTitle)` — `bookingService.js` | Pure string function. Also asserted at API + E2E (critical rule) |
| TC-101 | Prefix uppercased | `randomRef` | Pure |
| TC-411 | Title starts with digit/symbol | `randomRef` | Pure. Cheap to enumerate inputs (`"2026 Expo"`, `"#Hack"`, `""` → `"E"`) |
| TC-106 | FIFO prefers a different event | `createBooking` with mocked `bookingRepository.findOldestUserBookingExcludingEvent` / `findOldestUserBooking` | Branch selection. API setup would need 9 crafted bookings per case |
| TC-107 | Same-event fallback burns seats | `createBooking` → asserts `eventRepository.decrementSeats` called when `sameEventFallback` | Side-effect branch that isn't observable without DB inspection |
| TC-311 | Validation error matrix | `validateCreateBooking` chain, run against a mock `req` | 9+ input permutations. Validation must not be tested at E2E. One API contract test is kept |
| TC-409 | Email normalization | `body('customerEmail').normalizeEmail()` | Validator behavior |
| TC-410 | Phone with no digits accepted | `body('customerPhone')` regex + length | Validator behavior. Reproduces a suspected defect cheaply |
| TC-401 | Refund boundary qty 2 | `RefundEligibility` rule (`quantity === 1`) — `app/bookings/[id]/page.tsx` | Pure rule. **Requires extraction** to `getRefundStatus(qty)` (§5) |
| TC-402 | Refund qty 10 | same | same |

**Defense-in-depth Unit tests** (primary layer is API):
TC-102 price (`parseFloat(event.price) * quantity`), TC-105 FIFO trigger at `count >= MAX_USER_BOOKINGS`, TC-201 / TC-202 / TC-203 ownership checks in `getBookingById` / `getBookingByRef` / `cancelBooking`, TC-310 call-order (prune happens before `eventRepository.findById`), TC-312 `personalAvailable` calculation and `InsufficientSeatsError` message, TC-413 pagination defaults (`Number(limit) || 10`).

### 2.2 API / Integration (28 primary + 3 defense-in-depth)

| TC | Title | Endpoint → service fn | Why API |
|---|---|---|---|
| TC-002 | List ordered newest first | `GET /api/bookings` → `getBookings` → `findAll` (`orderBy createdAt desc`) | Data contract, no UI needed |
| TC-012 | Lookup by ref | `GET /api/bookings/ref/:ref` → `getBookingByRef` | No UI exposes this endpoint |
| TC-102 | totalPrice = price × qty | `POST /api/bookings` → `createBooking` | Persisted value. Card/detail rendering is covered in E2E TC-004 |
| TC-103 | Cancel restores seats | `DELETE /api/bookings/:id` + `GET /api/events/:id` (`withPersonalSeats`) | Backend rule. Also E2E (critical) |
| TC-104 | Clear all restores seats | `DELETE /api/bookings` → `clearAllBookings` | Multi-event state check is cheap via API |
| TC-105 | 10th booking prunes oldest | `POST /api/bookings` ×10 | Needs 10 creations, which is far too slow through the UI |
| TC-108 | Same event booked twice | `POST` ×2, `GET /api/events/:id` | Backend rule |
| TC-109 | Status always "confirmed"; `status=cancelled` empty | `GET /api/bookings?status=` | Contract |
| TC-110 | Count never exceeds 9 | `GET /api/bookings` → `pagination.total` | The cap is server-side. UI rendering of ≤9 cards is covered in Component TC-505 |
| TC-201 | Cross-user GET by id → 403 | `GET /api/bookings/:id` → `getBookingById` | API error code. The UI message is E2E TC-200 |
| TC-202 | Cross-user GET by ref → 403 | `getBookingByRef` | API error code |
| TC-203 | Cross-user DELETE → 404, booking survives | `cancelBooking` → `findById(id, userId)` | Two-user state check. Documents the 403/404 inconsistency |
| TC-204 | Clear all scoped to caller | `deleteAllForUser(userId)` | Two-user state |
| TC-205 | List scoped to caller | `findAll({ … }, userId)` | Two-user state |
| TC-206 | No token → 401 on all 5 routes | `authMiddleware` | Pure HTTP contract |
| TC-207 | Bad token → 401 | `authMiddleware` | Pure HTTP contract |
| TC-209 | Book another user's private event → 404 | `createBooking` → `eventRepository.findById(id, userId)` | Two-user state |
| TC-302 | Non-numeric id | `GET` / `DELETE /api/bookings/abc` | Checks for a 500 response, which only shows up at this layer |
| TC-303 | Double cancel → 404 | `cancelBooking` | Error code |
| TC-304 | Unknown ref → 404 | `getBookingByRef` | Error code |
| TC-310 | FIFO prunes even when the booking fails | `POST /api/bookings` with bad `eventId` at the 9-booking cap | Suspected defect. Must be shown on real data (also Unit for root cause) |
| TC-312 | Insufficient seats → 400 + message | `createBooking` | Error contract |
| TC-400 | Exactly 9 → no pruning | `POST` ×9 | Boundary that needs many creations |
| TC-407 | Clear all at zero | `DELETE /api/bookings` | Contract (`"0 booking(s) cleared"`) |
| TC-412 | Seats isolated across users | 2 tokens, `GET /api/events/:id` | Two-user state |
| TC-413 | Pagination params / negative page | `GET /api/bookings?page=&limit=` | Checks for a 500 response. Defaults are also Unit-tested |
| TC-415 | Event delete cascades bookings | `DELETE /api/events/:id` → booking 404 | Prisma `onDelete: Cascade`, DB-level |
| TC-416 | Event FIFO prune cascades bookings | `POST /api/events` ×7 | Cross-feature DB rule |

**Defense-in-depth API tests:** TC-100 (ref regex against the real title), TC-311 (one invalid payload asserting the `{ error: "Validation failed", details[] }` shape), TC-106 (one end-to-end check of the preferred-prune behavior).

### 2.3 Component (26 primary + 2 merged)

All of these run against the frontend with `page.route` mocks, so the asserted UI state does not depend on live data.

| TC | Title | Component / page | Why Component |
|---|---|---|---|
| TC-009 | Refund ineligible (qty 3) | `RefundEligibility` | Client-only logic. E2E TC-008 already proves the real path once |
| TC-010 | Back / breadcrumb links | `app/bookings/[id]/page.tsx` | Single-page links |
| TC-013 | Admin view modal | `app/admin/bookings/page.tsx` → `BookingModal` | Rendering of mocked row data |
| TC-014 | Admin cancel row | `AdminBookingsPage` → `useCancelBooking` | Same hook and endpoint as E2E TC-005, so only the admin UI wiring needs checking |
| TC-208 | Unauthenticated redirect to /login | `lib/api/client.js` 401 interceptor | Mock 401 → assert redirect and token removal |
| TC-300 | "Booking not found" state | detail page `isError` branch (status ≠ 403) | Mock 404 |
| ~~TC-301~~ | Cancelled booking → not found | *merged into TC-300 + API TC-303* | Hard delete is proven at API, and the UI branch is identical to TC-300 |
| TC-305 | Dismiss cancel dialog | `ConfirmDialog` in `BookingCard` | Assert no DELETE request (`page.on('request')`) |
| TC-306 | Dismiss "Clear all" confirm | `BookingsContent.handleClearAll` | `page.once('dialog', d => d.dismiss())` and assert no DELETE |
| TC-307 | Cancel → 500 → error toast | `BookingCard.onError` | Requires an injected server error |
| TC-308 | Clear all → 500 → no feedback | `handleClearAll` (no `catch`) | Requires an injected server error. UX defect |
| TC-309 | List load error + Retry | `BookingsContent` `isError` | Requires an injected failure |
| TC-403 | Refund spinner ~4 s | `RefundEligibility` `setTimeout(4000)` | Timed UI. Uses `not.toBeVisible({ timeout: 6000 })`, never `waitForTimeout` |
| TC-404 | Refund resets on reload | same | Client state |
| TC-405 | Navigate away mid-spinner | same | Console-error check (`page.on('console')`) |
| TC-406 | Cancel last booking → empty state | `BookingsContent` | Mock list [1] → after DELETE [0] |
| TC-408 | "1 ticket" / "2 tickets" | `BookingCard` | Rendering |
| TC-414 | Double-click confirm | `ConfirmDialog isLoading` | Delayed mock DELETE, count requests = 1 |
| TC-500 | Empty state + Browse Events | `BookingsContent` | Mock `data: []`. The real empty state also appears at the end of E2E TC-007 |
| TC-501 | Loading skeletons | `BookingCardSkeleton` | Delayed mock |
| TC-502 | "Clearing…" disabled state | `handleClearAll` | Delayed mock |
| TC-503 | Cancel dialog copy (card vs detail) | `BookingCard`, detail page | Text rendering |
| TC-504 | Refund state transitions | `RefundEligibility` | UI state machine |
| TC-505 | Pagination hidden ≤ 1 page | `components/ui/Pagination.jsx` (`totalPages <= 1 → null`) | Rendering |
| TC-506 | Sandbox banner on bookings page | `app/bookings/page.tsx` | Mock 9 bookings. **Currently expected to fail** (banner not implemented). Mark `test.fixme` until the requirement is clarified |
| TC-507 | Events query refetched after cancel | `useCancelBooking` → `invalidateQueries(['events'])` | Cache behavior: assert `GET /api/events` fires after DELETE. No real seats needed |
| TC-508 | Admin status filter / total | `AdminBookingsPage` `STATUS_OPTIONS` | Mocked responses per `status` param |
| ~~TC-509~~ | Admin pagination at 15 | *merged into TC-508* | Same mocked page, one extra assertion |

### 2.4 E2E (9)

Real backend, no mocks. Each test is one complete user journey.

| TC | Journey | Why it must be E2E |
|---|---|---|
| TC-001 | Book → `/bookings` shows the card with correct data | Proves the frontend ↔ API ↔ DB loop |
| TC-003 + TC-004 | List → View Details → all sections show the submitted name/email/phone/qty and the correct total | Multi-page. Also covers TC-102 rendering |
| TC-005 | Cancel from list card → toast → card gone | P0 destructive flow through `BookingCard` |
| TC-006 | Cancel from detail → toast → redirect to `/bookings` | Different component and handler from TC-005 (`router.push`) |
| TC-007 | Clear all (native confirm) → empty state | P0. Native dialog plus real delete |
| TC-008 | Refund eligible for a real 1-ticket booking | One real check that the booked quantity reaches the refund widget |
| TC-011 | Book → confirmation ref → "View My Bookings" → same ref first in list | Cross-page. **Also asserts TC-100** (ref prefix) as defense-in-depth |
| TC-103 | Book qty N → seats −N on event page → cancel → seats restored | Critical rule across 3 pages (also tested at API) |
| TC-200 | User B opens User A's `/bookings/:id` → "Access Denied" | Security P0. Needs two real sessions and the 403 → UI mapping |

---

## 3. Decision Rationale for Contested Assignments

| TC | Decision | Rationale |
|---|---|---|
| TC-100 ref prefix | Unit + API + E2E | It is a headline business rule (§7). Unit covers the input space, API proves the service wires the real title in, and E2E piggybacks on TC-011 at no extra cost. The existing spec tests it **only** at E2E, which is the anti-pattern. |
| TC-103 seat restore | API primary + E2E | `withPersonalSeats` is pure backend logic, so API is adequate. It is also the rule users notice most, so one E2E journey confirms the UI shows the recomputed number. |
| TC-106 / TC-107 FIFO branches | Unit primary | Hitting each branch via API needs 9 carefully composed bookings and, for TC-107, DB-level seat inspection. Mocked repositories assert exactly which `delete` / `decrementSeats` calls happen. TC-105 (the user-visible cap) stays at API. |
| TC-310 prune-before-validate | API primary + Unit | This is a suspected defect. The API test shows the user-visible harm (9 → 8 bookings after a 404), and the Unit test pins the root cause (call order). Only API counts as proof for a bug report. |
| TC-311 validation | Unit matrix + one API contract test | The anti-pattern list forbids validation at E2E. The 9 permutations belong in a fast unit loop, and one API test guards the response shape that the frontend parses. |
| TC-200 vs TC-201 | Both | TC-201 owns the status code (API). TC-200 owns the mapping `error.status === 403 → "Access Denied"` in `app/bookings/[id]/page.tsx`, which only shows up with two real sessions. Keep only one E2E for this. |
| TC-008 vs TC-009 | E2E for eligible, Component for ineligible | The rule is client-only, so one E2E proves the real `quantity` flows in. Repeating it at E2E for every quantity adds 4 s each for no new integration coverage. |
| TC-401 / TC-402 | Unit (after extraction) | `quantity === 1` is pure logic currently embedded in JSX. Until it is extracted (§5), fall back to Component. |
| TC-005 vs TC-006 | Both E2E | They exercise different code (`BookingCard.handleCancel` vs detail-page `handleCancel` + `router.push`). Both are P0 destructive actions. |
| TC-014 admin cancel | Component | It uses the same `useCancelBooking` → `DELETE /api/bookings/:id` path already proven in E2E TC-005. Only the admin table wiring is new. |
| TC-507 seats refresh after cancel | Component, not E2E | What can break here is React Query invalidation, which shows up as a `GET /api/events` request after the DELETE. The seat arithmetic is covered by TC-103. |
| TC-208 auth redirect | Component | The behavior lives entirely in the axios interceptor. A mocked 401 is deterministic, while a real expired token is hard to produce. |
| TC-301, TC-509 | Merged | They duplicate TC-300 and TC-508 at the same layer. Keeping them separate adds runtime with no new coverage. |
| TC-506 banner | Component, `test.fixme` | The requirement (domain skill §5) conflicts with the code. Keep it visible in the suite without failing CI until a product decision is made. |

---

## 4. Anti-Patterns in Existing Tests (`tests/booking-management.spec.js`)

| # | Anti-pattern | Where | Fix |
|---|---|---|---|
| 1 | **Ice cream cone**: 6/6 tests are E2E, with 0 Unit / API / Component | whole repo | Adopt the distribution in §1 |
| 2 | **Pure logic tested at E2E** | spec `TC-102` (ref prefix regex) | Move to Unit (`randomRef`) + API. Keep only an assertion inside the TC-011 E2E |
| 3 | **All setup done through the UI**: every test runs UI login → UI clear-all → UI booking before the step under test | `login()`, `clearBookings()`, `bookEvent()` | Log in once with `storageState` or a token in `addInitScript`. Seed and clean up via `request` (`DELETE /api/bookings`, `POST /api/bookings`). Use the UI only for the step under test |
| 4 | **Hardcoded `BASE_URL`** duplicates `baseURL` in config, and `toHaveURL(\`${BASE_URL}/bookings\`)` | top of spec, TC-003/006 | Use relative `page.goto('/bookings')` and `toHaveURL(/\/bookings$/)` so the suite can target localhost |
| 5 | **CSS / structural locators** despite the testid policy | `.confirm-booking-btn`, `.booking-ref`, `span.font-mono.font-bold`, `locator('h3')` | Add `data-testid="booking-ref"` to the confirmation ref (README claims it exists, but `app/events/[id]/page.tsx:40` has only a class) and `data-testid="confirm-booking-btn"`. Use `getByRole('heading')` for the card title |
| 6 | **Non-retrying state check** | `clearBookings`: `getByText('No bookings yet').isVisible().catch(() => false)` right after `goto`. This races the loading skeleton | Replace with an API `DELETE /api/bookings` |
| 7 | **Shared live account with destructive cleanup**: every test wipes all bookings for `rahulshetty1@gmail.com`, so concurrent runs (CI + local, or two CI refs) corrupt each other | all tests | Register a unique user per worker via `POST /api/auth/register` (fixture), or at least serialize with `workers: 1` |
| 8 | **Duplicated flow**: TC-006 re-implements `bookEvent` inline | spec TC-006 | Reuse the helper, or move it to a POM / fixture |
| 9 | **Stale TC IDs**: spec IDs follow the March scenario numbering | spec titles | Rename: spec TC-001→TC-001, TC-002→TC-003, TC-006→TC-011, TC-102→TC-100 (move down), TC-003→TC-006, TC-004→TC-007 |
| 10 | **Missing critical E2E** | — | Add TC-005 (cancel from list), TC-008 (refund), TC-103 (seat restore), and TC-200 (cross-user Access Denied) |
| 11 | **Weak, unscoped assertions** | `getByText('Test User')`, `getByText(eventTitle).first()` on the detail page | Scope to the section (e.g. a "Customer Details" region) and use unique data (`Test User ${Date.now()}`) |

---

## 5. Testability Changes Recommended Before Generation

1. **Export the booking-ref helpers**: in `backend/src/services/bookingService.js`, add `module.exports._internal = { randomRef, generateUniqueRef }` (or move them to `src/utils/bookingRef.js`). This is needed for Unit TC-100/101/411.
2. **Extract the refund rule**: create `frontend/lib/utils/refund.ts` exporting `getRefundStatus(quantity): 'eligible' | 'ineligible'` and use it in `RefundEligibility`. This moves TC-401/402 from Component to Unit.
3. **Add missing test IDs**: `booking-ref` and `confirm-booking-btn` on `app/events/[id]/page.tsx`, plus testids on the detail-page "Cancel Booking" button and the "Clear all bookings" button.

---

## 6. Prerequisites and Suggested File Layout (for `/generate-tests`)

| Layer | File | Prerequisite |
|---|---|---|
| Unit (backend) | `backend/__tests__/bookingService.test.js`, `backend/__tests__/bookingValidator.test.js` | Add Jest to `backend` devDependencies plus a `test` script. No unit runner exists today |
| Unit (frontend) | `frontend/lib/utils/refund.test.ts` | A frontend runner (e.g. Vitest), plus extraction §5.2 |
| API | `tests/api/bookings.api.spec.js` | **The live API base URL is not in the repo** (it lives in a deploy secret). Provide it as `API_URL` env var. Two test users (gmail + yahoo, see best-practices skill) or per-run registration |
| Component | `tests/component/bookings-ui.spec.js` | Token injected with `addInitScript(() => localStorage.setItem('eventhub_token', …))`. All `**/api/bookings**` routes mocked |
| E2E | `tests/e2e/booking-management.spec.js` (replaces the current spec) | API-based setup fixture (anti-pattern #3) |

`playwright.config.ts` `testDir: './tests'` already picks up the subfolders. Consider separate `projects` (`api`, `component`, `e2e`) so each layer can run on its own (`npx playwright test --project=api`).

### Suspected defects to track (from scenarios)
TC-310 (prune before validation), TC-203 (403/404 inconsistency), TC-302/413 (no id or pagination validation, possible 500), TC-308 (silent clear-all failure), TC-410 (digit-less phone accepted), TC-506 (missing banner), TC-107 (permanent seat burn).
