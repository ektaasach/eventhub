# EventHub — Booking Management Test Scenarios

Generated: 2026-11-17
Scope: Booking Management: view list, view detail, cancel (from list, detail page, and admin table), clear all, refund eligibility, cross-user isolation, FIFO booking cap, per-user seat restoration, and booking reference rules.
Sources: `eventhub-domain` skill (business-rules.md, user-flows.md), `frontend/app/bookings/**`, `frontend/app/admin/bookings/page.tsx`, `frontend/components/bookings/BookingCard.jsx`, `backend/src/services/bookingService.js`, `backend/src/repositories/bookingRepository.js`, `backend/src/validators/bookingValidator.js`.

> **Code facts that shape these scenarios**
> - Cancelling is a **hard delete** (`DELETE /api/bookings/:id`). No booking ever has `status = "cancelled"`.
> - `GET /api/bookings/:id` returns **403** for another user's booking, but `DELETE /api/bookings/:id` returns **404** for the same booking, because the repository filters by `userId` before the ownership check.
> - FIFO pruning in `createBooking` runs **before** the event-exists and seat-availability checks.
> - "Clear all bookings" uses a native `window.confirm()` dialog, not the app's ConfirmDialog.
> - The domain skill describes a sandbox banner on the bookings page. The current `app/bookings/page.tsx` does not render one.

---

## Happy Path (TC-001 – TC-099)

### TC-001: View bookings list with existing bookings
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in and has at least one booking
**Steps**:
1. Navigate to `/bookings`
**Expected Results**: Heading "My Bookings" is visible. One `booking-card` appears per booking, and each card shows the booking ref, a "confirmed" badge, `#<id>` (`booking-id`), the event title, event date, ticket count, city, booked date, total price, "View Details", and "Cancel Booking"
**Business Rule**: Flow 4 — Manage Bookings
**Suggested Layer**: E2E

---

### TC-002: Bookings list is ordered newest first
**Category**: Happy Path
**Priority**: P2
**Preconditions**: User has at least 2 bookings created at different times
**Steps**:
1. Create booking A, then booking B
2. Navigate to `/bookings`
**Expected Results**: Booking B's card appears above booking A's
**Business Rule**: Repository orders by `createdAt desc`
**Suggested Layer**: API

---

### TC-003: View booking detail page
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has a booking
**Steps**:
1. On `/bookings`, click "View Details" on a card
**Expected Results**: URL is `/bookings/<id>`. The breadcrumb shows "My Bookings / <ref>". The page shows the ref, status badge, and event title, plus Event Details (event, category, date, venue, city), Customer Details (name, email, phone), Payment Summary (tickets, price per ticket, Total Paid), a Refund section, and Booking Information (booked on, `#<id>`)
**Business Rule**: Flow 4 step 3–4
**Suggested Layer**: E2E

---

### TC-004: Booking detail data matches what was submitted
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User is logged in
**Steps**:
1. Book an event with name "Test User", email "test.user@example.com", phone "9876543210", quantity 2
2. Open the booking's detail page
**Expected Results**: Customer name, email, and phone match the input. Tickets = 2. Total Paid = event price × 2
**Business Rule**: §9 Price Calculation
**Suggested Layer**: E2E

---

### TC-005: Cancel a booking from the bookings list
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has at least one booking
**Steps**:
1. On `/bookings`, click `cancel-booking-btn` on a card
2. In the dialog "Cancel this booking?", confirm with `confirm-dialog-yes` ("Yes, cancel it")
**Expected Results**: Toast "Booking cancelled successfully" appears, and the card is removed from the list
**Business Rule**: Flow 4 step 6; §4 cancellation frees seats
**Suggested Layer**: E2E

---

### TC-006: Cancel a booking from the detail page
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has a booking
**Steps**:
1. Open `/bookings/<id>`
2. Click "Cancel Booking" in the header, then confirm
**Expected Results**: Toast "Booking cancelled successfully" appears, the user is redirected to `/bookings`, and the booking is no longer listed
**Business Rule**: Flow 4 step 6
**Suggested Layer**: E2E

---

### TC-007: Clear all bookings
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has 2 or more bookings
**Steps**:
1. On `/bookings`, click "Clear all bookings"
2. Accept the browser confirm "Clear all your bookings? This cannot be undone."
**Expected Results**: The list refreshes to the empty state "No bookings yet". `DELETE /api/bookings` responds 200 with message "`N` booking(s) cleared", where N is the prior count
**Business Rule**: §4 "Clear All Bookings removes all bookings in one go"
**Suggested Layer**: E2E

---

### TC-008: Refund check — single ticket is eligible
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has a booking with quantity = 1
**Steps**:
1. Open the booking detail page
2. Click `check-refund-btn` ("Check eligibility for refund?")
**Expected Results**: `refund-spinner` with "Checking your refund eligibility…" appears, then `refund-result` shows "Eligible for refund. Single-ticket bookings qualify for a full refund."
**Business Rule**: §8 Refund Eligibility
**Suggested Layer**: E2E

---

### TC-009: Refund check — multi-ticket is not eligible
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has a booking with quantity = 3
**Steps**:
1. Open the detail page and click `check-refund-btn`
**Expected Results**: After the spinner, `refund-result` shows "Not eligible for refund. Group bookings (3 tickets) are non-refundable."
**Business Rule**: §8 Refund Eligibility
**Suggested Layer**: E2E

---

### TC-010: Navigate back from detail to list
**Category**: Happy Path
**Priority**: P3
**Preconditions**: User is on a booking detail page
**Steps**:
1. Click "← Back to My Bookings" (and, separately, the "My Bookings" breadcrumb)
**Expected Results**: Both links navigate to `/bookings`
**Business Rule**: Flow 4 navigation
**Suggested Layer**: E2E

---

### TC-011: Reach bookings from booking confirmation
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User is logged in
**Steps**:
1. Book an event and wait for the confirmation card
2. Note the `booking-ref`, then click "View My Bookings"
**Expected Results**: The user lands on `/bookings`, and the first card's ref equals the noted ref
**Business Rule**: §1 "After booking, user can view bookings via View My Bookings"
**Suggested Layer**: E2E

---

### TC-012: Look up a booking by reference (API)
**Category**: Happy Path
**Priority**: P2
**Preconditions**: User owns a booking with ref `R`
**Steps**:
1. `GET /api/bookings/ref/R` with the owner's token
**Expected Results**: 200, `data.bookingRef === R`, and `data.event` is included
**Business Rule**: API — get booking by reference
**Suggested Layer**: API

---

### TC-013: Admin bookings table — view booking in modal
**Category**: Happy Path
**Priority**: P2
**Preconditions**: User has at least one booking
**Steps**:
1. Navigate to `/admin/bookings`
2. Click "View" on a row
**Expected Results**: A modal titled "Booking — <ref>" shows the reference, status, event title/date/city, customer name/email/phone, tickets, total, and booked date. Closing the modal (Escape or close) dismisses it
**Business Rule**: Admin — view bookings
**Suggested Layer**: E2E

---

### TC-014: Admin bookings table — cancel a booking
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User has at least one booking
**Steps**:
1. On `/admin/bookings`, click "Cancel" on a row, then confirm
**Expected Results**: Toast "Booking cancelled" appears, the row disappears, and the "`N` total bookings" count drops by 1
**Business Rule**: Admin — cancel bookings
**Suggested Layer**: E2E

---

## Business Rules (TC-100 – TC-199)

### TC-100: Booking reference starts with the event title's first letter
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User is logged in
**Steps**:
1. Book an event titled e.g. "Tech Conference Bangalore"
2. Read the ref on the confirmation card and on `/bookings`
**Expected Results**: Ref matches `^T-[A-Z0-9]{6}$`, and the first character equals the uppercased first character of the title
**Business Rule**: §7 Booking Reference Format
**Suggested Layer**: E2E

---

### TC-101: Booking reference prefix is uppercased for lowercase titles
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has created an event titled "yoga retreat"
**Steps**:
1. Book "yoga retreat"
**Expected Results**: Ref starts with "Y-" (uppercase)
**Business Rule**: §7 "First letter comes from the event title (uppercase)"
**Suggested Layer**: API

---

### TC-102: Total price = event price × quantity
**Category**: Business Rule
**Priority**: P0
**Preconditions**: Event with known price P
**Steps**:
1. Book quantity Q
2. Check the card total, the detail-page "Total Paid", and the API `totalPrice`
**Expected Results**: All three equal P × Q
**Business Rule**: §9 Price Calculation
**Suggested Layer**: API

---

### TC-103: Cancelling a booking restores seats for that user
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User is logged in; event E has S available seats for this user
**Steps**:
1. Book E with quantity 3, then open `/events/<E>`
2. Cancel the booking, then open `/events/<E>` again
**Expected Results**: After booking, seats = S − 3. After cancelling, seats = S
**Business Rule**: §4 "Booking deletion immediately frees seats"; §6 Per-user seat availability
**Suggested Layer**: E2E

---

### TC-104: Clear all restores seats for every affected event
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has bookings on events E1 (qty 2) and E2 (qty 4)
**Steps**:
1. Record the available seats on E1 and E2
2. Clear all bookings
3. Re-check E1 and E2
**Expected Results**: E1 increases by 2 and E2 increases by 4
**Business Rule**: §4, §6
**Suggested Layer**: API

---

### TC-105: FIFO — 10th booking prunes the oldest booking
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has cleared all bookings
**Steps**:
1. Create 9 bookings on distinct events (or alternating events), recording the ref of booking #1
2. Create a 10th booking on an event different from booking #1's
3. `GET /api/bookings`
**Expected Results**: `pagination.total` = 9. Booking #1's ref is gone, and the 10th booking is present
**Business Rule**: §4 Max 9 bookings, FIFO pruning
**Suggested Layer**: API

---

### TC-106: FIFO prefers pruning a booking for a different event
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has 9 bookings; the oldest is for event A, and the second oldest is for event B
**Steps**:
1. Create a new booking for event A
**Expected Results**: The oldest booking **not** for event A (the B booking) is deleted. The original A booking survives
**Business Rule**: `findOldestUserBookingExcludingEvent` (code behavior)
**Suggested Layer**: API

---

### TC-107: FIFO same-event fallback permanently consumes seats
**Category**: Business Rule
**Priority**: P2
**Preconditions**: All 9 of the user's bookings are for event A; A's DB `availableSeats` = S
**Steps**:
1. Create a 10th booking for A with quantity q
2. Check the event's seats and the bookings list
**Expected Results**: The oldest A booking is deleted, the new booking exists, and A's DB `availableSeats` drops to S − q permanently (it is not restored when bookings are later cancelled)
**Business Rule**: `sameEventFallback` → `decrementSeats` (code behavior)
**Suggested Layer**: API

---

### TC-108: Same user can book the same event multiple times
**Category**: Business Rule
**Priority**: P1
**Preconditions**: Event has enough seats
**Steps**:
1. Book event E twice (qty 1, then qty 2)
**Expected Results**: Two distinct bookings with different refs are listed. Seats for E drop by 3 in total
**Business Rule**: §6 "allows the same user to book the same event multiple times"
**Suggested Layer**: API

---

### TC-109: All listed bookings have status "confirmed"
**Category**: Business Rule
**Priority**: P2
**Preconditions**: User has bookings, some of which have been cancelled
**Steps**:
1. `GET /api/bookings` and `GET /api/bookings?status=cancelled`
**Expected Results**: Every returned booking has `status: "confirmed"`. The `status=cancelled` query returns `total: 0`, because cancelled bookings are deleted
**Business Rule**: Data model — status always "confirmed"
**Suggested Layer**: API

---

### TC-110: Booking count limit enforced on the bookings page
**Category**: Business Rule
**Priority**: P2
**Preconditions**: User has made more than 9 bookings over time
**Steps**:
1. Navigate to `/bookings`
**Expected Results**: At most 9 `booking-card` elements appear and no pagination control is shown (limit = 10 > 9)
**Business Rule**: §4 "Bookings page shows max 9 bookings at a time"
**Suggested Layer**: E2E

---

## Security (TC-200 – TC-299)

### TC-200: User B cannot view User A's booking detail (UI)
**Category**: Security
**Priority**: P0
**Preconditions**: User A (gmail account) has booking id X
**Steps**:
1. As User A, capture booking id X from `booking-id`
2. Log out (clear `eventhub_token`) and log in as User B (yahoo account)
3. Navigate to `/bookings/X`
**Expected Results**: "Access Denied" with "You are not authorized to view this booking." and a "View My Bookings" button. No customer data from A is rendered
**Business Rule**: §2 / Flow 6 Cross-User Security
**Suggested Layer**: E2E

---

### TC-201: User B cannot fetch User A's booking by id (API)
**Category**: Security
**Priority**: P0
**Preconditions**: User A owns booking X
**Steps**:
1. `GET /api/bookings/X` with User B's token
**Expected Results**: 403 `{ success: false, error: "You are not authorized to view this booking" }`
**Business Rule**: §2 Cross-user → 403
**Suggested Layer**: API

---

### TC-202: User B cannot fetch User A's booking by reference
**Category**: Security
**Priority**: P0
**Preconditions**: User A owns ref R
**Steps**:
1. `GET /api/bookings/ref/R` with User B's token
**Expected Results**: 403 "You do not own this booking"
**Business Rule**: §2
**Suggested Layer**: API

---

### TC-203: User B cannot cancel User A's booking
**Category**: Security
**Priority**: P0
**Preconditions**: User A owns booking X
**Steps**:
1. `DELETE /api/bookings/X` with User B's token
2. `GET /api/bookings/X` with User A's token
**Expected Results**: Step 1 is rejected with 404 "Booking with id X not found" (note: 404, not 403, which is inconsistent with GET). Step 2 returns 200, so the booking still exists
**Business Rule**: §2; `bookingRepository.findById` filters by userId
**Suggested Layer**: API

---

### TC-204: Clear all only affects the caller's bookings
**Category**: Security
**Priority**: P0
**Preconditions**: Users A and B both have bookings
**Steps**:
1. As User B, `DELETE /api/bookings`
2. As User A, `GET /api/bookings`
**Expected Results**: B's bookings are gone, and A's count is unchanged
**Business Rule**: §2 Sandbox isolation; `deleteMany({ where: { userId } })`
**Suggested Layer**: API

---

### TC-205: Bookings list shows only the caller's bookings
**Category**: Security
**Priority**: P0
**Preconditions**: Users A and B each have bookings
**Steps**:
1. As B, `GET /api/bookings` and `GET /api/bookings?eventId=<event A booked>`
**Expected Results**: Only B's bookings are returned. None of A's refs appear even when filtering by the shared event
**Business Rule**: §2
**Suggested Layer**: API

---

### TC-206: Booking endpoints require authentication
**Category**: Security
**Priority**: P0
**Preconditions**: None
**Steps**:
1. Call `GET /api/bookings`, `GET /api/bookings/1`, `GET /api/bookings/ref/X-ABCDEF`, `DELETE /api/bookings/1`, and `DELETE /api/bookings` with no `Authorization` header
**Expected Results**: Each returns 401 `{ success: false, error: "Unauthorized" }`
**Business Rule**: authMiddleware on all booking routes
**Suggested Layer**: API

---

### TC-207: Invalid or expired token is rejected
**Category**: Security
**Priority**: P1
**Preconditions**: None
**Steps**:
1. `GET /api/bookings` with `Authorization: Bearer garbage` and with a token signed by a different secret
**Expected Results**: 401 "Invalid or expired token"
**Business Rule**: JWT auth (7-day expiry)
**Suggested Layer**: API

---

### TC-208: Unauthenticated user visiting /bookings is redirected to login
**Category**: Security
**Priority**: P1
**Preconditions**: No `eventhub_token` in localStorage, or an invalid one
**Steps**:
1. Navigate directly to `/bookings` and to `/bookings/1`
**Expected Results**: The user ends on `/login`, and no booking data is displayed. An invalid token is removed from localStorage
**Business Rule**: API client 401 interceptor
**Suggested Layer**: E2E

---

### TC-209: Booking cannot be created on another user's private event
**Category**: Security
**Priority**: P1
**Preconditions**: User A created dynamic event E
**Steps**:
1. As User B, `POST /api/bookings` with `eventId = E`
**Expected Results**: 404 "Event with id E not found", and no booking is created for B
**Business Rule**: §2 dynamic events are private
**Suggested Layer**: API

---

## Negative / Error (TC-300 – TC-399)

### TC-300: Detail page for a non-existent booking id
**Category**: Negative
**Priority**: P1
**Preconditions**: User is logged in
**Steps**:
1. Navigate to `/bookings/99999999`
**Expected Results**: "Booking not found" with "This booking doesn't exist or may have been cancelled." and a "View My Bookings" button. The API returns 404
**Business Rule**: NotFoundError → 404
**Suggested Layer**: E2E

---

### TC-301: Detail page for a just-cancelled booking
**Category**: Negative
**Priority**: P1
**Preconditions**: User cancelled booking X
**Steps**:
1. Navigate to `/bookings/X`
**Expected Results**: The "Booking not found" state is shown (not "Access Denied", and not a cancelled-status booking)
**Business Rule**: Cancellation is a hard delete
**Suggested Layer**: E2E

---

### TC-302: Non-numeric booking id
**Category**: Negative
**Priority**: P2
**Preconditions**: User is logged in
**Steps**:
1. `GET /api/bookings/abc` and `DELETE /api/bookings/abc`
2. Navigate to `/bookings/abc`
**Expected Results**: The API returns a 4xx client error rather than 500 or a stack trace, and the UI shows "Booking not found". (Risk: `Number('abc')` = NaN is passed to Prisma, so a 500 is possible. Log it as a defect if so.)
**Business Rule**: Discovered code behavior — no id validation on routes
**Suggested Layer**: API

---

### TC-303: Cancelling an already-cancelled booking
**Category**: Negative
**Priority**: P2
**Preconditions**: Booking X was just cancelled
**Steps**:
1. `DELETE /api/bookings/X` again
**Expected Results**: 404 "Booking with id X not found", and no change to seats
**Business Rule**: NotFoundError
**Suggested Layer**: API

---

### TC-304: Unknown booking reference
**Category**: Negative
**Priority**: P2
**Preconditions**: User is logged in
**Steps**:
1. `GET /api/bookings/ref/Z-000000`
**Expected Results**: 404 `Booking with reference "Z-000000" not found`
**Business Rule**: NotFoundError
**Suggested Layer**: API

---

### TC-305: Dismissing the cancel dialog keeps the booking
**Category**: Negative
**Priority**: P1
**Preconditions**: User has a booking
**Steps**:
1. Click "Cancel Booking", then close the dialog (Escape or close) without confirming
**Expected Results**: The dialog closes, no DELETE request is sent, and the card is still present
**Business Rule**: Flow 4 confirmation step
**Suggested Layer**: E2E

---

### TC-306: Dismissing the "Clear all" confirm keeps all bookings
**Category**: Negative
**Priority**: P1
**Preconditions**: User has bookings
**Steps**:
1. Click "Clear all bookings" and dismiss the native confirm (`page.once('dialog', d => d.dismiss())`)
**Expected Results**: No `DELETE /api/bookings` request is sent, and the list is unchanged
**Business Rule**: §4 Clear all requires confirmation
**Suggested Layer**: E2E

---

### TC-307: Cancel fails on the server
**Category**: Negative
**Priority**: P2
**Preconditions**: Mock `DELETE /api/bookings/:id` to return 500 `{ error: "Internal server error" }`
**Steps**:
1. Cancel a booking from the list and confirm
**Expected Results**: An error toast with the server message appears, the dialog closes, and the card remains
**Business Rule**: BookingCard `onError` handling
**Suggested Layer**: Component

---

### TC-308: Clear all fails on the server
**Category**: Negative
**Priority**: P2
**Preconditions**: Mock `DELETE /api/bookings` to return 500
**Steps**:
1. Click "Clear all bookings" and accept
**Expected Results**: The button resets from "Clearing…" to "Clear all bookings" and the list is unchanged. (Gap: no error toast is shown and the rejection is unhandled, so log it as a UX defect.)
**Business Rule**: `handleClearAll` has `finally` but no `catch`
**Suggested Layer**: Component

---

### TC-309: Bookings list fails to load
**Category**: Negative
**Priority**: P2
**Preconditions**: Mock `GET /api/bookings` to return 500, or abort it
**Steps**:
1. Navigate to `/bookings`, then click "Retry" after restoring the API
**Expected Results**: "Couldn't load bookings" with "Failed to connect to the server. Please try again." appears. After Retry, the list loads
**Business Rule**: Error state in BookingsContent
**Suggested Layer**: Component

---

### TC-310: FIFO pruning happens even when the new booking fails
**Category**: Negative
**Priority**: P1
**Preconditions**: User has exactly 9 bookings
**Steps**:
1. `POST /api/bookings` with a non-existent `eventId` (e.g. 99999999) and otherwise valid data
2. `GET /api/bookings`
**Expected Results**: Expected: 404 and still 9 bookings. **Suspected defect:** the request returns 404, but the oldest booking was already deleted, leaving 8. The same applies when the request fails on insufficient seats
**Business Rule**: §4 FIFO only on a successful new booking; the code prunes before validating
**Suggested Layer**: API

---

### TC-311: Booking validation errors (create input to management)
**Category**: Negative
**Priority**: P1
**Preconditions**: User is logged in
**Steps**:
1. `POST /api/bookings` with each invalid field in turn: missing `eventId`; `eventId: 0`; `customerName: "A"`; `customerEmail: "bad"`; `customerPhone: "12345"`; `customerPhone: "98765abcde"`; `quantity: 0`; `quantity: 11`; `quantity: 2.5`
**Expected Results**: 400 `{ error: "Validation failed", details: [{ field, message }] }` with the matching message (e.g. "Quantity must be an integer between 1 and 10"). No booking is created and the booking count is unchanged
**Business Rule**: bookingValidator; Data model constraints
**Suggested Layer**: API

---

### TC-312: Booking more seats than available
**Category**: Negative
**Priority**: P1
**Preconditions**: Event with fewer than 10 personal available seats (e.g. a user-created event with totalSeats 3)
**Steps**:
1. `POST /api/bookings` with quantity 4
**Expected Results**: 400 "Only 3 seat(s) available, but 4 requested"
**Business Rule**: InsufficientSeatsError
**Suggested Layer**: API

---

## Edge Cases (TC-400 – TC-499)

### TC-400: Exactly 9 bookings — no pruning
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User has cleared all bookings
**Steps**:
1. Create 9 bookings
**Expected Results**: All 9 exist (total = 9), none are pruned
**Business Rule**: §4 boundary (max = 9)
**Suggested Layer**: API

---

### TC-401: Refund boundary — quantity 2 is not eligible
**Category**: Edge Case
**Priority**: P1
**Preconditions**: Booking with quantity 2
**Steps**:
1. Check refund eligibility
**Expected Results**: "Not eligible… Group bookings (2 tickets) are non-refundable."
**Business Rule**: §8 boundary (1 eligible, >1 not)
**Suggested Layer**: Component

---

### TC-402: Refund for the maximum quantity (10)
**Category**: Edge Case
**Priority**: P3
**Preconditions**: Booking with quantity 10
**Steps**:
1. Check refund eligibility
**Expected Results**: "Group bookings (10 tickets) are non-refundable."
**Business Rule**: §8; quantity max 10
**Suggested Layer**: Component

---

### TC-403: Refund spinner lasts about 4 seconds
**Category**: Edge Case
**Priority**: P2
**Preconditions**: Booking detail page is open
**Steps**:
1. Click `check-refund-btn`
2. Assert `refund-spinner` is visible and `refund-result` is not
3. Wait for `refund-result` (use an expect timeout above 4s, not `waitForTimeout`)
**Expected Results**: The spinner shows for roughly 4s and is then replaced by the result. The check button is hidden from the moment it is clicked
**Business Rule**: §8 "4-second spinner"
**Suggested Layer**: Component

---

### TC-404: Refund check resets on page reload
**Category**: Edge Case
**Priority**: P3
**Preconditions**: Refund result is displayed
**Steps**:
1. Reload the page
**Expected Results**: The Refund section shows the "Check eligibility for refund?" button again (client-only state, nothing persisted)
**Business Rule**: §8 frontend-only logic
**Suggested Layer**: E2E

---

### TC-405: Navigating away during the refund spinner
**Category**: Edge Case
**Priority**: P3
**Preconditions**: Refund check in progress
**Steps**:
1. Click "← Back to My Bookings" within 4s
**Expected Results**: Navigation succeeds with no console errors (e.g. no React state update on an unmounted component)
**Business Rule**: §8 timer behavior
**Suggested Layer**: Component

---

### TC-406: Cancelling the only booking shows the empty state
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has exactly one booking
**Steps**:
1. Cancel it from `/bookings`
**Expected Results**: The empty state "No bookings yet" with a "Browse Events" button appears
**Business Rule**: UI empty-state transition
**Suggested Layer**: E2E

---

### TC-407: Clear all with zero bookings
**Category**: Edge Case
**Priority**: P3
**Preconditions**: User has no bookings
**Steps**:
1. `DELETE /api/bookings` (or click "Clear all bookings" on the empty page)
**Expected Results**: 200 "0 booking(s) cleared", and the empty state stays
**Business Rule**: clearAllBookings
**Suggested Layer**: API

---

### TC-408: Ticket label pluralization
**Category**: Edge Case
**Priority**: P3
**Preconditions**: Bookings with quantity 1 and quantity 2
**Steps**:
1. View `/bookings`
**Expected Results**: The cards show "1 ticket" and "2 tickets" respectively
**Business Rule**: BookingCard rendering
**Suggested Layer**: Component

---

### TC-409: Email is normalized on save
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User is logged in
**Steps**:
1. Book with customer email "Test.User@Gmail.com"
2. View the booking detail
**Expected Results**: Document the actual stored email. `normalizeEmail()` lowercases it and removes Gmail dots ("testuser@gmail.com"), so the displayed email may differ from what was typed. Confirm this is intended
**Business Rule**: bookingValidator `.normalizeEmail()`
**Suggested Layer**: API

---

### TC-410: Phone without digits passes validation
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User is logged in
**Steps**:
1. `POST /api/bookings` with `customerPhone: "----------"` (10 dashes) or `"(((((())))))"`
**Expected Results**: Expected: 400 (rule says "min 10 digits"). **Suspected defect:** accepted, because length and the character class are checked but the digit count is not
**Business Rule**: Data model — phone min 10 digits
**Suggested Layer**: Unit

---

### TC-411: Booking reference for a title starting with a digit or symbol
**Category**: Edge Case
**Priority**: P3
**Preconditions**: User creates an event titled "2026 Expo" (and one titled "#Hackathon")
**Steps**:
1. Book each event
**Expected Results**: The refs start with "2-" and "#-" respectively. Decide whether that is acceptable against the `[LETTER]-` format in the rule
**Business Rule**: §7 `randomRef` uses the first char as-is
**Suggested Layer**: Unit

---

### TC-412: Seat restoration does not leak across users
**Category**: Edge Case
**Priority**: P1
**Preconditions**: Users A and B; static event E
**Steps**:
1. Record E's seats as seen by B
2. A books E with quantity 5, then cancels
3. Re-check E's seats as seen by B at each step
**Expected Results**: B's seat count never changes
**Business Rule**: §6 per-user seat availability
**Suggested Layer**: API

---

### TC-413: Booking list pagination parameters
**Category**: Edge Case
**Priority**: P3
**Preconditions**: User has 9 bookings
**Steps**:
1. `GET /api/bookings?page=2&limit=5`, `?page=3&limit=5`, `?limit=0`, `?page=-1`
**Expected Results**: page 2 → 4 items, totalPages 2. page 3 → empty data. `limit=0` falls back to 10. Negative page returns a 4xx client error or page 1, not a 500 (risk: negative `skip` sent to Prisma)
**Business Rule**: getBookings pagination defaults
**Suggested Layer**: API

---

### TC-414: Double-clicking the confirm-cancel button
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has a booking
**Steps**:
1. Open the cancel dialog and double-click "Yes, cancel it" quickly
**Expected Results**: Exactly one success toast appears, with no error toast from a second DELETE (the button should be disabled while `isPending`)
**Business Rule**: ConfirmDialog `isLoading`
**Suggested Layer**: E2E

---

### TC-415: Deleting an event cascades its bookings
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User created event E and booked it
**Steps**:
1. Delete E from `/admin/events`
2. View `/bookings` and `GET /api/bookings/<bookingId>`
**Expected Results**: The booking for E is gone (404)
**Business Rule**: Event → Booking `onDelete: Cascade`
**Suggested Layer**: API

---

### TC-416: FIFO pruning of a user-created event removes its bookings
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has 6 created events, and the oldest has a booking
**Steps**:
1. Create a 7th event
2. Check the bookings list
**Expected Results**: The oldest event is pruned and its bookings disappear from `/bookings`
**Business Rule**: §3 FIFO event pruning + cascade
**Suggested Layer**: API

---

## UI State (TC-500 – TC-599)

### TC-500: Empty state when the user has no bookings
**Category**: UI State
**Priority**: P1
**Preconditions**: User has cleared all bookings
**Steps**:
1. Navigate to `/bookings`
2. Click "Browse Events"
**Expected Results**: "No bookings yet" with "You haven't booked any events yet. Browse upcoming events and grab your tickets!" is shown, and the button navigates to `/events`
**Business Rule**: Bookings page empty state
**Suggested Layer**: E2E

---

### TC-501: Loading skeletons while bookings load
**Category**: UI State
**Priority**: P3
**Preconditions**: Delay `GET /api/bookings` via route mocking
**Steps**:
1. Navigate to `/bookings`
**Expected Results**: 5 skeleton cards render until the data arrives. No empty state flashes before the data loads
**Business Rule**: BookingCardSkeleton
**Suggested Layer**: Component

---

### TC-502: "Clearing…" state on the Clear all button
**Category**: UI State
**Priority**: P3
**Preconditions**: Delay `DELETE /api/bookings`
**Steps**:
1. Click "Clear all bookings" and accept
**Expected Results**: The button text becomes "Clearing…" and is disabled until the response arrives
**Business Rule**: `clearing` state
**Suggested Layer**: Component

---

### TC-503: Cancel dialog content reflects the booking
**Category**: UI State
**Priority**: P2
**Preconditions**: Booking with ref R and quantity Q
**Steps**:
1. Click "Cancel Booking" on the card, then on the detail page
**Expected Results**: Card dialog: "This will cancel R and release Q seat(s) back to the event. This action cannot be undone." Detail dialog: "Cancelling R will release Q seat(s) back to the event. This cannot be undone." Both have the title "Cancel this booking?" and the button "Yes, cancel it"
**Business Rule**: ConfirmDialog copy
**Suggested Layer**: Component

---

### TC-504: Refund section state transitions
**Category**: UI State
**Priority**: P2
**Preconditions**: Booking detail page is open
**Steps**:
1. Observe the initial state, click check, observe again, then observe after completion
**Expected Results**: idle → only the button is shown. checking → only the spinner and text are shown. result → only the green (eligible) or red (ineligible) panel is shown, and the button never reappears
**Business Rule**: §8
**Suggested Layer**: Component

---

### TC-505: Pagination hidden with 9 or fewer bookings
**Category**: UI State
**Priority**: P3
**Preconditions**: User has 1–9 bookings
**Steps**:
1. View `/bookings`
**Expected Results**: No pagination control is rendered (`totalPages <= 1`)
**Business Rule**: Pagination component
**Suggested Layer**: Component

---

### TC-506: Sandbox limit banner on bookings page
**Category**: UI State
**Priority**: P2
**Preconditions**: User has close to or exactly 9 bookings
**Steps**:
1. View `/bookings`
**Expected Results**: Per the domain skill, a heads-up banner about the 9-booking sandbox limit appears, and it is hidden when counts are low. **Note:** the current `app/bookings/page.tsx` renders no such banner. Either the domain doc is stale or the banner is missing, so flag it for clarification
**Business Rule**: §5 Sandbox Warning Banners
**Suggested Layer**: E2E

---

### TC-507: Events page reflects seats after cancel without manual reload
**Category**: UI State
**Priority**: P2
**Preconditions**: User booked event E (qty 2)
**Steps**:
1. Cancel the booking via the card, then client-side navigate to `/events` and open E
**Expected Results**: Seats include the 2 restored tickets (the `events` query was invalidated)
**Business Rule**: useCancelBooking invalidates `events`
**Suggested Layer**: E2E

---

### TC-508: Admin bookings — status filter and total count
**Category**: UI State
**Priority**: P3
**Preconditions**: User has bookings
**Steps**:
1. On `/admin/bookings`, select "Cancelled" and then "Confirmed" in the status filter
**Expected Results**: "Cancelled" shows "No bookings found", because cancelled bookings are deleted. "Confirmed" lists all bookings, and the header shows "`N` total bookings"
**Business Rule**: Hard-delete cancellation; admin filter
**Suggested Layer**: E2E

---

### TC-509: Admin bookings pagination at 15 per page
**Category**: UI State
**Priority**: P3
**Preconditions**: User has 9 or fewer bookings (max)
**Steps**:
1. View `/admin/bookings`
**Expected Results**: All bookings are on one page and no pagination is shown (limit 15 > max 9)
**Business Rule**: Admin page `limit: 15`; §4 max 9
**Suggested Layer**: Component

---

## Summary

| Category | Count | IDs |
|---|---|---|
| Happy Path | 14 | TC-001 – TC-014 |
| Business Rule | 11 | TC-100 – TC-110 |
| Security | 10 | TC-200 – TC-209 |
| Negative | 13 | TC-300 – TC-312 |
| Edge Case | 17 | TC-400 – TC-416 |
| UI State | 10 | TC-500 – TC-509 |
| **Total** | **75** | |

### Suspected defects / clarifications surfaced
1. **TC-310**: FIFO pruning deletes the oldest booking even when the new booking then fails (missing event, insufficient seats).
2. **TC-203**: Cross-user cancel returns 404, while cross-user GET returns 403. The response codes are inconsistent.
3. **TC-302 / TC-413**: There is no validation of `:id`, `page`, or `limit`, so non-numeric or negative values may produce a 500.
4. **TC-308**: A failed "Clear all" shows no error feedback (no `catch`).
5. **TC-410**: Phone validation accepts strings that contain no digits.
6. **TC-506**: The bookings-page sandbox banner described in the domain skill is not present in the code.
7. **TC-107**: The same-event FIFO fallback permanently decrements DB seats.
