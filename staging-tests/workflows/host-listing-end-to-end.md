# Workflow: Host Listing End-to-End

Chains host setup (location -> operating hours -> multi-step add-car -> pricing -> photos) plus protection plans, trip fees, and coupons. Every step after location persists via GraphQL, so the bulk of this workflow is MUTATING.

- **Env:** https://staging.autopilotee.com (Chrome via Playwright MCP)
- **Persona:** Host (account discovered via auth workflow; toggle "Switch to host" in hamburger menu)
- **Routes (from App.tsx):** add-location `/cars/host/add-location`; add-car `/cars/host/add` -> `add-vin` -> (car-info modal) -> `add-price` -> `add-photos` -> listings `/cars/host`; protection plans `/cars/host/protection-plans`; trip fees `/cars/host/miscellaneous-fees` (title "Trip Fees Set up"); coupons `/cars/host/coupons`.
- **VIN regex:** `/^[A-HJ-NPR-Z0-9]{17}$/`.

## Setup
1. Sign in as the host account (`staging_credentials.local`).
2. Open hamburger -> "Switch to host"; confirm host nav links and `localStorage.role = host`.

## Flow — Location & hours
1. **HLOC-01** — Open the add-location modal from My Locations.
2. **HLOC-03** — Confirm required-field validation blocks empty submit.
3. **HLOC-02** — [MUTATING] Create a location (happy path); confirm it appears in My Locations.
4. **HLOC-07** — [MUTATING] Set operating hours for the new location; save and confirm persisted.
5. **HLOC-08** — [MUTATING] Toggle "Always available"; confirm state persists.
6. **HLOC-06** — Open delete confirmation, click Cancel only (no destructive delete).

## Flow — Add a car (multi-step, all MUTATING after step 1)
7. **HCAR-02** — If no location exists, confirm add-car requires a location first.
8. **HCAR-01** — [MUTATING] Start add-car: select item type + the location from step 3.
9. **HCAR-04** — Confirm VIN format validation rejects bad input.
10. **HCAR-03** — [MUTATING] Enter a valid 17-char VIN; confirm decode populates car info.
11. **HCAR-05** — [MUTATING] Complete car details form and submit.
12. **HCAR-06** — [MUTATING] Set car pricing.
13. **HCAR-08** — Confirm Next disabled with zero photos.
14. **HCAR-07** — [MUTATING] Upload photos and finish; lands on `/cars/host`.
15. **HCAR-09** — New listing appears on My Listings.
16. **HCAR-10** — Open the listing detail / edit view.

## Flow — Protection plans, trip fees, coupons
17. **PLAN-01** — Protection Plans page loads with all four plan types.
18. **PLAN-02** — Expand a plan to reveal the edit form.
19. **PLAN-03** — [MUTATING] Create/save a plan (active toggle + price).
20. **PLAN-05** — [MUTATING] Toggle a plan OFF and persist.
21. **TFEE-01** — Trip Fees page loads with default fee rows.
22. **TFEE-03** — Expand a charge-rule fee; verify chargeBy selector.
23. **TFEE-05** — [MUTATING] Configure and save a trip fee (amount + active + chargeBy).
24. **COUP-01** — Coupons page loads, lists existing coupons.
25. **COUP-02** — Open the Create Coupon form.
26. **COUP-05** — Confirm percentage validation rejects out-of-range value.
27. **COUP-03** — [MUTATING] Create a percentage coupon (capture the server-generated code for use in CHK-13 / COUP-12).
28. **COUP-09** — [MUTATING] Edit the coupon.
29. **COUP-10** — [MUTATING] Delete the coupon (cleanup; uses native `window.confirm`).

## Teardown
- Delete the coupon created (COUP-10) if not already.
- The test car/location created via HCAR/HLOC are persistent; either delete via the host UI where allowed or record them as known disposable test fixtures.
- Switch role back to guest if proceeding to guest workflows.
