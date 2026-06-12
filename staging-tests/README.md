# AutoPilotee Cars — Staging Manual Test Suite

Manual, browser-executable test cases for the live staging web app at **https://staging.autopilotee.com** (the `autopilotee-cars` React frontend). Cases are grounded in the actual source repos under `/Users/chao/marcmoo/` and use real `data-testid` selectors, route paths, button text, and GraphQL operation names so they can be driven by a human or a Playwright browser agent.

## What's here

- **`TEST-PLAN.md`** — the master plan: scope & environment, credentials reference, personas, coverage matrix, risk notes, and the prioritized execution order. **Start here.**
- **`cases/<area>/<area>.md`** — atomic test cases per area. Each case has Priority (P0/P1/P2), Persona, Preconditions, numbered Steps, Expected results, and Source files. Mutating steps are tagged `[MUTATING]`.
- **`workflows/*.md`** — end-to-end journeys that chain atomic cases into realistic sessions with setup/teardown:
  - `auth.md` — sign-in, persistence, role switch, sign-out.
  - `guest-booking-end-to-end.md` — search -> view -> checkout -> (careful) payment.
  - `host-listing-end-to-end.md` — location -> hours -> add-car -> pricing -> photos -> plans/fees/coupons.
  - `booking-management.md` — viewing, cancel/refund rules, invoices, swap/DL flows.
- **`results/`** — execution output (created during test runs).

## Directory layout

```
staging-tests/
  README.md            <- this file
  TEST-PLAN.md         <- master plan + execution order
  cases/
    authentication/authentication.md
    guest-search-discovery/guest-search-discovery.md
    guest-car-view-checkout/guest-car-view-checkout.md
    checkout-payment/checkout-payment.md
    host-listing/host-listing.md
    host-protection-fees-coupons/host-protection-fees-coupons.md
    booking-management/booking-management.md
    chat/chat.md
    secondary-pages/secondary-pages.md
    landing/landing.md
  workflows/
    auth.md
    guest-booking-end-to-end.md
    host-listing-end-to-end.md
    booking-management.md
  results/
```

## How to run (Playwright MCP against staging)

1. Use the **Playwright MCP** tools (Chrome) pointed at `https://staging.autopilotee.com`.
2. Read `TEST-PLAN.md` and follow the **execution order** (Band 0 anonymous smoke -> auth -> guest view/checkout -> careful payment -> host -> booking mgmt -> secondary).
3. For each case, drive the numbered Steps using the `data-testid` selectors / visible text in the case file. Assert each Expected result via `browser_snapshot` / `browser_wait_for`.
4. **Credentials:** see `staging_credentials.local` (USER1, USER2, Stripe test card). Never paste passwords into reports.
5. **Role discovery:** the guest-vs-host role of USER1/USER2 is unknown; discover it once (auth workflow) and reuse the mapping.
6. **Mutations:** prefer read-only verification. Run `[MUTATING]` steps only with explicit approval; never click "Pay now"/"Book" unless the charge is approved (use the decline card for negative cases). Clean up created data where the UI allows.

## Scope notes

- Auth flows are modals, not routes (open via hamburger menu or `?signin=true` / `?purpose=VERIFICATION` / `?purpose=RESET_PASSWORD`).
- The dealership microsite and surrogate page are domain-gated and will NOT render on staging (negative case DEAL-00).
- The marketing landing site is a separate Next.js app on a different host; discover its URL or mark `LAND-*` not applicable.
- `/account-deletion` is a content page, not the real delete-account flow.

See `TEST-PLAN.md` for full details.
