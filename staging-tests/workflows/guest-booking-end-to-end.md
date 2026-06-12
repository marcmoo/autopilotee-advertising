# Workflow: Guest Booking End-to-End

Chains discovery -> car view -> date selection -> checkout -> (careful) payment -> confirmation. This is the core revenue path.

- **Env:** https://staging.autopilotee.com (Chrome via Playwright MCP)
- **Persona:** Guest (signed-in non-host account; discover via auth workflow)
- **Routes:** results `/cars/guest/search`; detail `/cars/guest/view/:id`; checkout `/cars/guest/pay/:id/:bookingId`; confirmation `/cars/guest/booking/:bookingId`; list `/guest/mybookings`.
- **Search selectors (BookCard):** `booking-card-container`, `booking-pickup-location-input`, `booking-pickup-date-input`, `booking-return-date-input`, `booking-pickup-time-select`, `booking-return-time-select`, `booking-age-select`, `booking-search-button`.
- **Card selectors:** `search-car-card-<id>`, `search-car-title-<id>`, `search-car-price-<id>`, `search-car-favorite-<id>`.
- **Checkout selectors:** `checkout-title`, `checkout-protection-section`, `free-cancellation-desktop`, `terms-agreement-desktop`, `checkout-coupon-input`, `checkout-coupon-apply-button`, `checkout-coupon-success`/`error`, `checkout-book-button-desktop`/`-mobile`. Stripe iframe title "Secure payment input frame"; "Pay now" disabled until Element complete.

## Setup
1. Sign in as the guest account (`staging_credentials.local`); confirm role = guest.

## Flow — Discovery
1. **GSRCH-01** — Home loads (hero + `booking-card-container`).
2. **GSRCH-03** — Fill pickup location + future dates/times, click `booking-search-button`; land on `/cars/guest/search` with "Found N cars available".
3. **GSRCH-05** — Click a `search-car-card-<id>`; capture the real `CAR_ID` and navigate to `/cars/guest/view/<CAR_ID>`.

## Flow — Car view & intent (non-charging)
4. **GVCO-01** — Detail page shows title, rating, specs, pre-tax "$N total / Before taxes and fees".
5. **GVCO-05** — Open the date-range modal; select valid future dates + times.
6. **GVCO-07** — Confirm driver-age selector defaults to 25+.
7. **GVCO-10** — [MUTATING] Click "Continue" -> creates a booking intent, navigates to `/cars/guest/pay/<CAR_ID>/<bookingId>`. Capture `bookingId`.

## Flow — Checkout (non-charging)
8. **CHK-01** — All sections render (title, protection, price breakdown, deposit, coupon, terms, book button).
9. **CHK-02** — Price totals consistent (rental + tax = total before deposit).
10. **CHK-03** — With terms unchecked, click book; confirm "Please agree to the terms and conditions" blocks payment.
11. **CHK-04** — Toggle `terms-agreement-desktop` checked.
12. **CHK-13** — [MUTATING-intent] Apply a valid staging coupon (if one exists); confirm `checkout-coupon-success` and updated totals. Else CHK-14 invalid coupon shows error.
13. **CHK-06** — With terms accepted, click `checkout-book-button-desktop`; confirm the Stripe payment modal opens (iframe "Secure payment input frame"). **STOP here unless charge approved.**

## Flow — Payment (EXPLICIT APPROVAL ONLY)
14. **CHK-07** — "Pay now" stays disabled until all Stripe fields valid.
15. **CHK-09** — [MUTATING — CHARGES CARD] Enter test card `4242...`, submit; confirm BOOKED booking + redirect to `/cars/guest/booking/<bookingId>`.
16. **CHK-10** — Booking appears in `/guest/mybookings`.

Alternative negative (no money moves):
- **CHK-11** — [MUTATING] Enter decline card `4000 0000 0000 0002`; confirm error and no booking created.

## Teardown
- If a real booking was created (CHK-09), cancel it via guest cancel (GBOOK-04, >24h before start) to clean up and exercise the refund path, or leave it as a known disposable test booking.
- Remove any applied coupon / favorite toggled during the run.
