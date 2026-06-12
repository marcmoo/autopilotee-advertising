# Workflow: Booking Management Lifecycle

Covers viewing bookings (guest + host), the cancel/refund rules, invoices/refunds, and the only host decision flows that exist (car-swap, DL-verification rejection). There is NO host approve/decline of bookings — a booking goes pending -> booked automatically after the Stripe `charge.succeeded` webhook (`verifyBookingPayment`).

- **Env:** https://staging.autopilotee.com (Chrome via Playwright MCP)
- **Personas:** Guest and Host (discover which account is which).
- **Routes:** guest list `/guest/mybookings`; host list "Bookings For My Cars"; checkout `/cars/guest/pay/:id/:bookingId`.
- **GraphQL ops:** `GetMyBookings`, `GetBookingsForMyListings`, `GetCancelledBookingsAsGuest`/`Host`, `GetInvoiceById`, `RefreshInvoiceRefundStatus`, `CreateRefundForInvoice`, `GetInvoicesByRefundStatus`.
- **Cancel/refund rules (bookings.service.ts):**
  - Guest cancel **>24h** before trip = FULL refund, auto-executed (`cancelled_by_guest_more_than_24_hours`).
  - Guest cancel **<24h** = PARTIAL refund, set to `pending_host_review` (host must Create Refund Now).
  - Host cancel = always full refund.
  - List tabs (Upcoming/Ongoing/History) are date-derived, not status-derived.
  - Invoice "Create Refund Now" appears only when `stripePaymentIntentId` set and `stripeRefundId` empty.

## Setup
- Have at least one disposable BOOKED booking for the guest account (created via the guest-booking workflow CHK-09). Identify the host account that owns the booked car.

## Flow — Read-only viewing
1. **GBOOK-01** — `/guest/mybookings` requires auth (logged-out shows no data / prompts sign-in).
2. **GBOOK-02** — Signed in as guest: view list + Upcoming/Ongoing/History tab filtering.
3. **GBOOK-03** — Open a booking -> guest booking detail.
4. **HBOOK-01** — Signed in as host: view "Bookings For My Cars" list + tabs.
5. **HBOOK-02** — Open a booking -> host booking detail.
6. **INV-01** — View invoices by refund status (host).
7. **INV-02** — View invoice details.
8. **INV-03** — Refresh invoice refund status.
9. **CBOOK-01** — Guest: view My Cancelled Bookings list.
10. **CBOOK-04** — Host: view cancelled bookings.

## Flow — Cancel / refund (MUTATING, disposable bookings only)
11. **GBOOK-04** — [MUTATING] Guest cancel **>24h** before trip -> confirm FULL refund auto-executed, status `cancelled_by_guest_more_than_24_hours`.
12. **GBOOK-05** — [MUTATING] Guest cancel **<24h** before trip -> confirm PARTIAL refund, status `pending_host_review`. (Needs a booking starting within 24h.)
13. **INV-04** — [MUTATING] Host: Create Refund Now for a paid invoice that is `pending_host_review` (from GBOOK-05).
14. **HBOOK-03** — [MUTATING] Host "Cancel Trip Without Penalty" -> confirm guest cancellation penalty waived.

## Flow — Host decision flows (MUTATING)
15. **HSWAP-01** — [MUTATING] Host requests a car swap.
16. **GSWAP-01** — [MUTATING] Guest approves or rejects the pending swap.
17. **HDL-01** — [MUTATING] Host rejects DL verification -> booking cancelled (`rejectDLAndCancelBooking`).

## Teardown
- Cancellations/refunds are terminal; ensure only disposable test bookings were used.
- Record refund IDs and final statuses for the report. Do not re-run mutations against the same booking.
