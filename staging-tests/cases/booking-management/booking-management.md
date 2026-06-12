# Booking Management Lifecycle - Manual Test Cases

App: https://staging.autopilotee.com (AutoPilotee Cars React frontend)

## Lifecycle / refund rules extracted from backend (ground truth)

Source: `backend-autopilotee-cars/src/components/bookings/enums/booking.enums.ts`, `bookings.service.ts`, `bookings.resolver.ts`.

- `BookingStatus` values: `pending`, `booked`, `completed`, `card_declined`, `shortened`, `cancelled_by_guest_more_than_24_hours`, `cancelled_by_guest_less_than_24_hours_before_trip_start`, `cancelled_by_guest_after_trip_start`, `cancelled_by_host`, `cancelled_by_admin`.
- There is NO host approve/decline of a booking itself. A booking goes `pending` -> `booked` automatically after Stripe payment confirms (webhook `charge.succeeded` -> `verifyBookingPayment`). "Approve/Decline" in this codebase exist only for: (a) Car Swap (`approveCarSwap` / `rejectCarSwap` / `forceApproveCarSwap` / `forceDeclineCarSwap`), and (b) DL verification (`rejectDLAndCancelBooking`). Those are covered as the host-side decision flows below.
- Frontend list tabs (Upcoming / Ongoing / History) are derived from dates, NOT status: Upcoming = `startAt >= now`; Ongoing = `startAt <= now <= endAt`; History = `endAt < now`. Source: `MyBookingsPage/index.tsx`, `BookedMyCarsPage/index.tsx`.
- Cancel + refund logic (`bookings.service.ts cancelBooking`):
  - Guest cancel > 24h before trip start -> `FULL_REFUND`, refund auto-executed immediately, status `cancelled_by_guest_more_than_24_hours`.
  - Guest cancel < 24h before trip start -> `PARTIAL_REFUND`, invoice set to `refundStatus = pending_host_review`, host gets a "Refund Needs Review" notification, status `cancelled_by_guest_less_than_24_hours_before_trip_start`.
  - Host cancel -> always full refund, status `cancelled_by_host`, `cancellationReason = HOST_CANCELLED`.
- `RefundStatus`: `NONE`, `pending_host_review`, `processing`, `completed`, `failed`, `cancelled`.
- Invoice refund actions (`InvoiceDetailsPage`): "Create Refund Now" is shown only when `!stripeRefundId && stripePaymentIntentId` (i.e. paid invoice with no refund yet). "Refresh Refund Status" auto-fires once on load.

## Routes

| Page | Route |
|------|-------|
| My Bookings (guest) | `/guest/mybookings` |
| Bookings For My Cars (host) | `/bookedmycars` |
| My Cancelled Bookings | `/cancelled-bookings` |
| Booking detail (guest) | `/cars/guest/booking/:bookingId` |
| Booking detail (host) | `/cars/host/booking/:bookingId` |
| Invoices by refund status | `/invoices/refund-status` |
| Invoice detail | `/invoice/:id` |

Credentials: USER1 `jidosoju+5@gmail.com` / `<password: see staging_credentials.local>`; USER2 `autopilotee+206503@gmail.com` / `<password: see staging_credentials.local>`. Stripe test card `4242 4242 4242 4242`.

---

### GBOOK-01: My Bookings page requires authentication
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Not signed in (clear cookies / fresh session).
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/guest/mybookings
  2. Wait for the page network to settle.
- **Expected:** A login prompt is shown (email input `input[type="email"]` visible, or text matching `sign in`/`log in`). The My Bookings list is NOT rendered for an anonymous user.
- **Source:** `e2e/booking-management.spec.ts` (`my bookings page requires authentication`); `App.tsx` route `/guest/mybookings`.

### GBOOK-02: View My Bookings list and tab filtering (guest)
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as the account that has at least one booking as guest (discover which of USER1/USER2 has guest bookings).
- **Steps:**
  1. Sign in.
  2. Navigate to https://staging.autopilotee.com/guest/mybookings
  3. Observe the title and the three tabs.
  4. Click the `Upcoming` tab.
  5. Click the `Ongoing` tab.
  6. Click the `History` tab.
- **Expected:**
  - Page title "My Bookings" is shown.
  - Three tabs render: `Upcoming`, `Ongoing`, `History`.
  - Each booking card shows: a status badge (e.g. `booked`), an `ID:` short-id badge, a date range `MM/dd/yyyy hh:mm a - ...`, car image, `year make model`, pickup location, return location, and `Host:` name.
  - Upcoming shows only bookings with start date >= now (sorted earliest first); Ongoing shows in-progress; History shows ended bookings (sorted most recently finished first). If a tab has none, "No bookings found." is displayed.
- **Source:** `MyBookingsPage/index.tsx` (`useGetMyBookingsQuery`, operation `GetMyBookings`).

### GBOOK-03: Open a booking from My Bookings navigates to guest booking detail
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as a guest with at least one booking.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/guest/mybookings
  2. Click on the first booking card.
- **Expected:** URL changes to `/cars/guest/booking/<bookingId>`. The booking detail page renders (car info, dates, status, payment summary). For a non-cancelled future booking, an "Actions" card with guest-facing buttons appears.
- **Source:** `MyBookingsPage/index.tsx` (`navigate('/cars/guest/booking/' + booking.id)`); `App.tsx`; `BookingDetails/index.tsx`.

### HBOOK-01: View Bookings For My Cars list and tabs (host)
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Signed in as the account that owns at least one car with bookings (discover which of USER1/USER2 is a host).
- **Steps:**
  1. Sign in.
  2. Navigate to https://staging.autopilotee.com/bookedmycars
  3. Observe the title and tabs.
  4. Click `Upcoming`, then `Ongoing`, then `History`.
- **Expected:**
  - Page title "Bookings For My Cars".
  - Three tabs: `Upcoming`, `Ongoing`, `History`.
  - Each card shows status badge, `ID:` badge, date range, car image, `year make model`, pickup/return location, and `Guest:` name (not Host).
  - Tab filtering by date works as in GBOOK-02. Empty tab shows "No bookings found."
- **Source:** `BookedMyCarsPage/index.tsx` (`useGetBookingsForMyListingsQuery`, operation `GetBookingsForMyListings`).

### HBOOK-02: Open a booking from Bookings For My Cars navigates to host booking detail
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Signed in as a host with at least one booking on a listing.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/bookedmycars
  2. Click the first booking card.
- **Expected:** URL changes to `/cars/host/booking/<bookingId>`. Host booking detail renders. For a future, non-cancelled booking, the host "Actions" card shows host buttons such as "Cancel Trip Without Penalty", "Modify Trip Without Penalty", and (when status is `booked` with no active swap) "Swap Car".
- **Source:** `BookedMyCarsPage/index.tsx` (`navigate('/cars/host/booking/' + booking.id)`); `BookingDetails/index.tsx` lines 1267-1316.

### HBOOK-03: Host "Cancel Trip Without Penalty" waives guest cancellation penalty
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Signed in as host. Open a host booking detail for a future, non-cancelled booking where the penalty has NOT already been waived (`isCancellationPenaltyWaivedByHost` false). USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to the host booking detail `/cars/host/booking/<bookingId>`.
  2. In the Actions card, click the "Cancel Trip Without Penalty" button.
  3. [MUTATING] Confirm the action in the dialog ("This will allow the guest to cancel this booking without any penalty. Continue?").
- **Expected:** Success indication. The "Cancel Trip Without Penalty" button no longer appears (penalty now waived). The guest can subsequently cancel with a full refund regardless of the 24h window.
- **Source:** `BookingDetails/index.tsx` `handleRemoveCancellationPenalty` (confirm text line ~921), button line 1270-1276.

### GBOOK-04: Guest cancel > 24h before trip => full refund auto-executed
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as guest with a confirmed (`booked`) booking whose `startAt` is more than 24 hours in the future and not yet cancelled. USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to the guest booking detail `/cars/guest/booking/<bookingId>`.
  2. In the Actions card, click the "Cancel the Trip" button.
  3. Read the "Confirm Refund" toast/alert (shows refund type + amount, plus deposit refund note if a deposit exists).
  4. [MUTATING] Click OK / confirm to proceed with cancellation.
- **Expected:**
  - A confirmation toast titled "Confirm Refund" appears before mutation; clicking Cancel instead aborts and shows "Booking not cancelled" (the just-created refund invoice is removed).
  - After confirming: toast "Booking cancelled successfully". Booking status becomes `cancelled_by_guest_more_than_24_hours`. Refund is auto-executed (FULL_REFUND) without host review. Booking disappears from active My Bookings tabs and appears in `/cancelled-bookings`.
- **Source:** `BookingDetails/index.tsx` lines 753-794 (`cancelBooking` with `CancelledByGuestMoreThan_24Hours`); backend `bookings.service.ts` lines 1106-1133 (auto full refund).

### GBOOK-05: Guest cancel < 24h before trip => partial refund, host review required
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as guest with a confirmed booking whose `startAt` is LESS than 24 hours away (and penalty not waived by host). USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to `/cars/guest/booking/<bookingId>`.
  2. Click "Cancel the Trip".
  3. Review the "Confirm Refund" alert: it should show a penalty/partial refund breakdown.
  4. [MUTATING] Confirm cancellation.
- **Expected:** Booking status becomes `cancelled_by_guest_less_than_24_hours_before_trip_start`. The refund invoice is set to `refundStatus = pending_host_review` (NOT auto-refunded). The host receives a "Refund Needs Review" notification. Booking now appears under `/cancelled-bookings`.
- **Source:** `BookingDetails/index.tsx` lines 762-766 (`CancelledByGuestLessThan_24HoursBeforeTripStart`); backend `bookings.service.ts` lines 1134-1158.

### CBOOK-01: View My Cancelled Bookings list (guest)
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as a guest with at least one cancelled booking (e.g. after GBOOK-04/05).
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cancelled-bookings
  2. Observe the title and tabs.
  3. Click a cancelled booking card.
- **Expected:**
  - Title "My Cancelled Bookings". Tabs `All`, `Pending Refund`, `Refunded`.
  - Each card shows a red status badge with the cancellation status (underscores replaced with spaces), the cancellation reason, date range, car, pickup/return location and `Host:` name.
  - Clicking a card opens a "Cancellation Details" modal showing Status, Reason, Cancelled at, and a "Refund Information" section (Refund Status, Refund Amount placeholder, Penalty Amount placeholder, Host Approved Refund Yes/No).
- **Source:** `CancelledBookingsPage/GuestCancelledBookingsPage.tsx` (`useGetCancelledBookingsAsGuestQuery`, operation `GetCancelledBookingsAsGuest`).

### CBOOK-02: Cancellation Details modal "View Full Details" navigates to booking
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest with at least one cancelled booking.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cancelled-bookings
  2. Click a cancelled booking card to open the modal.
  3. Click "View Full Details".
- **Expected:** Navigates to `/cars/guest/booking/<bookingId>` showing the full booking detail (including the cancellation banner and refund/payment summary).
- **Source:** `GuestCancelledBookingsPage.tsx` `navigateToBookingDetails` (line 230-232), button line 369-371.

### CBOOK-03: Filter cancelled bookings by cancellation reason (guest)
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest with multiple cancelled bookings having different reasons.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cancelled-bookings
  2. If a reason filter dropdown is present, select a specific cancellation reason.
- **Expected:** The query refetches with `filter.cancellationReason` and only matching cancelled bookings are listed; clearing the filter shows all.
- **Source:** `GuestCancelledBookingsPage.tsx` lines 179-187 (`filter: { cancellationReason }`).

### CBOOK-04: View cancelled bookings as host
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Signed in as host with at least one cancelled booking on a listing.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cancelled-bookings (host variant renders for host role).
  2. Observe the cancelled bookings list.
- **Expected:** Host cancelled bookings render (operation `GetCancelledBookingsAsHost`), each showing the guest, cancellation status and reason.
- **Source:** `CancelledBookingsPage/HostCancelledBookingsPage.tsx`; `bookingService/queries.ts` `GetCancelledBookingsAsHost`.

### INV-01: View invoices by refund status
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Signed in as a host/account that has refund invoices.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/invoices/refund-status
  2. Observe the list and any status filter (e.g. `pending_host_review`, `processing`, `completed`, `failed`, `cancelled`).
- **Expected:** Invoices grouped/filterable by refund status are listed; each row is clickable and links to `/invoice/<id>`.
- **Source:** `InvoicesByRefundStatusPage`; `invoiceServices/queries.ts` `GetInvoicesByRefundStatus` / `GetHostInvoicesByRefundStatus`.

### INV-02: View invoice details
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Signed in. Have a valid invoice id (from INV-01 or from a cancelled booking's refund).
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/invoice/<id>
  2. Wait for the page to load (refund status auto-refreshes once on load).
- **Expected:**
  - Title "Invoice Details" with a "Back to Invoices" link.
  - Basic Information section shows: Invoice ID, Type, Amount ($), Invoice Status, Created At, Approved At, Cancellation Reason, Requester (Email), Host (Email), Booking ID (links to `/cars/host/booking/<id>`), and a Refund Status badge (color-coded: PENDING_HOST_REVIEW yellow, PROCESSING blue, COMPLETED green, FAILED red, CANCELLED gray).
  - If `stripePaymentIntentId` exists, a Stripe Payment Intent link is shown; if `stripeRefundId` exists, a Stripe Refund link is shown.
  - If the invoice has original + new booking states (e.g. trip modification), Original Booking (red) and New Booking (green) sections plus a "Price Difference" banner appear.
- **Source:** `InvoiceDetailsPage/index.tsx` (`GET_INVOICE_BY_ID` / operation `GetInvoiceById`).

### INV-03: Refresh invoice refund status
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On an invoice detail page `/invoice/<id>`.
- **Steps:**
  1. Click the "Refresh Refund Status" button.
- **Expected:** Button shows "Refreshing..." while loading, then re-enables. The Refund Status badge reflects the latest status from Stripe after the refetch. (This is a read/sync action, no charge.)
- **Source:** `InvoiceDetailsPage/index.tsx` lines 244-252 (`REFRESH_INVOICE_REFUND_STATUS` / operation `RefreshInvoiceRefundStatus`).

### INV-04: Create refund for a paid invoice (host)
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Signed in as host. On an invoice `/invoice/<id>` where `stripePaymentIntentId` exists and `stripeRefundId` is empty, so the "Create Refund Now" button is visible (typically a `pending_host_review` partial refund). USE A DISPOSABLE/TEST INVOICE.
- **Steps:**
  1. Optionally type a note into the "Refund note (optional)" textarea.
  2. [MUTATING] Click "Create Refund Now".
- **Expected:** Button shows "Creating...". On success the page refetches and the Refund Status badge updates toward PROCESSING/COMPLETED and a Stripe Refund ID link appears; the "Create Refund Now" button disappears (since `stripeRefundId` is now set). On failure an alert "Refund cannot be executed." with the error message is shown.
- **Source:** `InvoiceDetailsPage/index.tsx` lines 186-199, 253-268 (`CREATE_REFUND_FOR_INVOICE` / operation `CreateRefundForInvoice`; `canCreateRefund` condition line 234).

### HSWAP-01: Host requests a car swap (host decision flow)
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Signed in as host. Host booking detail for a `booked` booking with no active swap. USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to `/cars/host/booking/<bookingId>`.
  2. In Actions, click "Swap Car".
  3. [MUTATING] Complete the swap modal and submit.
- **Expected:** A swap request is created; `swapStatus` becomes `pending_guest_approval`. The host detail shows the pending swap awaiting guest approval. The guest detail will show approve/reject controls.
- **Source:** `BookingDetails/index.tsx` lines 1296-1307 ("Swap Car"), 1375 (host pending swap UI).

### GSWAP-01: Guest approves or rejects a pending car swap
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest. The guest has a booking with `swapStatus = pending_guest_approval` (created via HSWAP-01). USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to `/cars/guest/booking/<bookingId>`.
  2. Observe the pending swap section (line 1440 in BookingDetails).
  3. [MUTATING] Click the approve control (and separately test reject on another swap).
- **Expected:** Approve -> toast "Car swap approved successfully", `swapStatus = approved`/`completed`, booking now references the new car (swap history shows "Swapped 1 time" with the original car). Reject -> toast "Car swap rejected", `swapStatus = rejected`.
- **Source:** `BookingDetails/index.tsx` `handleApproveSwap`/`handleRejectSwap` (lines 652-672); backend `approveCarSwap`/`rejectCarSwap` resolvers.

### HDL-01: Host rejects driver license verification, cancelling the booking
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Signed in as host. Booking with a submitted DL awaiting host review (`dlVerificationStatus = SUBMITTED`). USE A DISPOSABLE/TEST BOOKING.
- **Steps:**
  1. Navigate to `/cars/host/booking/<bookingId>`.
  2. Locate the DL verification review UI.
  3. [MUTATING] Reject the DL with a reason.
- **Expected:** DL status becomes `REJECTED`; the booking is cancelled with `cancellationReason = DL_VERIFICATION_FAILED`; the guest is refunded per policy. Booking appears in cancelled bookings.
- **Source:** backend `bookings.resolver.ts` line 819 (`rejectDLAndCancelBooking`); enums `DLVerificationStatus`, `CancellationReason.DL_VERIFICATION_FAILED`.

### GBOOK-06: Checkout page loads for a created booking
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest. Have a created (pending) booking id and its car id.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/guest/pay/<carId>/<bookingId>
  2. Wait for the page to settle.
- **Expected:** Either the checkout page renders (text matching `checkout`, Stripe payment elements) or, if session expired, a login prompt. (Do NOT submit payment unless explicitly testing a real charge.)
- **Source:** `e2e/booking-management.spec.ts` (`checkout page with booking`); `GuestCarPage/CheckoutPage/index.tsx`.
