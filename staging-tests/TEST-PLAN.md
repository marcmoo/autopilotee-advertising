# AutoPilotee Cars — Master Manual Test Plan (Staging)

## 1. Scope & Environment

- **Application under test:** AutoPilotee Cars React frontend (title "AutoPilotee Cars").
- **Base URL:** https://staging.autopilotee.com
- **Browser / driver:** Chrome via Playwright MCP.
- **Backend:** NestJS GraphQL (operation names referenced per case). Auth via AWS Cognito. Payments via Stripe (test mode).
- **Out of scope on this host:**
  - The dealership microsite (DealershipLanding, `/cars_on_sale`, `/contact`, CarFax viewer) is domain-gated to `autopiloteecars.com` / `jidosojuholding.com` and the surrogate page to `handysurrogate.com`. It will NOT render on `staging.autopilotee.com` (see DEAL-00 negative case).
  - The marketing landing site (Next.js `autopilotee-landing`) is a separate app, likely on a different host. Executor must discover its URL or mark `LAND-*` not applicable.
  - `/account-deletion` is a content markdown page only, not the real delete-account action (which lives in profile/settings, out of scope here).

## 2. Test Data & Credentials

- Credentials live in `staging_credentials.local` (do not paste passwords into reports or commits — reference the file).
  - **USER1:** `jidosoju+5@gmail.com` (password: see `staging_credentials.local`)
  - **USER2:** `autopilotee+206503@gmail.com` (password: see `staging_credentials.local`)
  - **Stripe test card (success):** `4242 4242 4242 4242`, any future expiry, any CVC, any ZIP (see `staging_credentials.local`).
  - **Stripe decline card (CHK-11):** `4000 0000 0000 0002`.
- **Role discovery:** The guest-vs-host role of USER1 / USER2 is unknown. Both accounts can toggle role via the "Switch to host/guest" button in the hamburger menu (authenticated only). The executor must discover at runtime which account owns cars (host) and which has bookings (guest). Record the mapping once discovered and reuse it.
- **Runtime IDs:** Seeded e2e IDs (e.g. car `aaaaaaaa-...`) are local-only and will NOT exist on staging. The executor must obtain real `CAR_ID` / `bookingId` values by going through guest search -> car view -> date select.

## 3. Personas

| Persona | Description | How to enter |
| --- | --- | --- |
| Anonymous | Logged-out visitor | Default state; clear `localStorage` keys `tokens` / `user` |
| Guest | Authenticated renter | Sign in; role defaults to guest |
| Host | Authenticated lister/owner | Sign in, then "Switch to host" in hamburger menu (persists to `localStorage.role`) |
| Dealer | Host who owns a dealer entity | Host account that qualifies for `/dealership` (DealerPage); discover at runtime |

## 4. Coverage Matrix (area × priority)

| Area | P0 | P1 | P2 | Total |
| --- | --- | --- | --- | --- |
| Authentication (AUTH) | 4 | 7 | 7 | 18 |
| Guest search & discovery (GSRCH) | 4 | 7 | 9 | 20 |
| Guest car view & checkout entry (GVCO) | 5 | 7 | 8 | 20 |
| Checkout & Stripe payment (CHK) | 6 | 8 | 7 | 21 |
| Host listing & location (HLOC/HCAR/HADM) | 7 | 6 | 9 | 22 |
| Host plans/fees/coupons (PLAN/TFEE/COUP) | 6 | 9 | 11 | 26 |
| Booking management (GBOOK/HBOOK/CBOOK/INV/HSWAP/GSWAP/HDL) | 4 | 8 | 9 | 21 |
| Chat (CHAT) | 3 | 5 | 6 | 14 |
| Secondary pages / policies / dealer (HELP/ABOUT/POL/ACCT/DEAL) | 0 | 4 | 9 | 13 |
| Landing (LAND) | 0 | 4 | 5 | 9 |
| **Total** | **39** | **65** | **80** | **184** |

(Counts reflect priority tags in the per-area case files; totals are approximate where a case spans personas.)

## 5. Risk Notes & Mutation Caution Strategy

**Default posture: read-only verification on staging.** Prefer cases that assert page render, validation, and disabled-state behavior over cases that write data.

- **[MUTATING] cases write real staging data** (Cognito users, bookings, charges, refunds, messages, listings, coupons). Execute a mutation ONLY when explicitly approved. Clean up created data where the UI allows (delete coupon, cancel disposable booking, etc.).
- **Never charge the Stripe card unless explicitly approved.** Many flows are non-mutating right up to the final "Pay now"/"Book" click. Stop before that click (GVCO-18, CHK-06/07/08 verify the modal opens and validation without charging). CHK-09 (success charge) and CHK-11 (decline) are the only intentional charge cases; CHK-11 uses the decline card so no money moves.
- **Do not submit fake verify/reset codes** against real accounts (AUTH-11, AUTH-17) — verify the modal/validation only.
- **AUTH-08 (sign up)** needs a unique, never-registered email controlled by the executor.
- **AUTH-18 / Google OAuth** — only initiate the redirect to Cognito hosted UI; do not complete a Google login.
- **Destructive deletes** (HLOC-06 location delete, COUP-10 coupon delete) — HLOC-06 only exercises Cancel; COUP-10 deletes a coupon the executor created in COUP-03/04.
- **Host add-car flow** persists each step via GraphQL, so every "Next" after the location step is effectively mutating; treat the whole HCAR sequence as data-creating.
- **Cancel/refund rules** (booking management): guest cancel >24h = full auto-refund; <24h = partial, pending host review; host cancel = full refund. Use disposable test bookings only.
- **Data dependency / flakiness:** Some cases depend on specific seeded data that may be absent on staging (GSRCH-13 operating-hours error, GSRCH-20 airport tabs, DEAL-01 dealer-owning host, valid coupon code for CHK-13). Flag as not-applicable if unreachable rather than forcing.
- **No host approve/decline of bookings exists** — booking goes pending -> booked automatically after Stripe webhook. The only host decision flows are car-swap and DL-verification rejection.

## 6. Prioritized Execution Order

Run P0 smoke first, in dependency order (anonymous load -> auth -> guest search -> car view -> careful checkout -> host flows -> booking mgmt -> secondary). Within each band, non-mutating before mutating.

### Band 0 — Anonymous smoke (no auth, no mutation)
1. GSRCH-01 — Home page loads (hero + search card). Fastest signal the app is up.
2. GSRCH-03 — Location search navigates to results.
3. GSRCH-04 — Seeded location search shows results, cards, map.
4. GSRCH-05 — Car card click -> detail page.

### Band 1 — Authentication (gates everything authenticated)
5. AUTH-01 — Sign in valid (establishes session; discover role). [MUTATING-session]
6. AUTH-02 — Invalid password error.
7. AUTH-12 — Session persists across reload.
8. AUTH-15 — Role switch host/guest (confirms which account is host).
9. AUTH-13 — Sign out resets to guest. [MUTATING-session]

### Band 2 — Guest car view & checkout entry (non-charging)
10. GVCO-01 — Car detail loads.
11. GVCO-05 — Date selection via modal.
12. GVCO-14 — Checkout page renders.
13. CHK-01 — Checkout sections render.
14. CHK-02 — Price totals consistent.
15. CHK-03 — Terms blocks payment.
16. CHK-06 — Stripe modal opens (terms accepted, no charge).

### Band 3 — Checkout charge (explicit approval only)
17. CHK-09 — Successful payment creates BOOKED booking. [MUTATING — CHARGES CARD]
18. CHK-10 — Booking appears in My Bookings.
19. CHK-11 — Declined card shows error, no booking. [MUTATING — decline card]

### Band 4 — Host flows (host account)
20. HLOC-01 — Open add-location modal.
21. HLOC-02 — Create location. [MUTATING]
22. HLOC-07 — Set operating hours. [MUTATING]
23. HCAR-01 — Start add-car (type + location). [MUTATING]
24. HCAR-03 — VIN decode. [MUTATING]
25. HCAR-05 — Car details submit. [MUTATING]
26. HCAR-06 — Set pricing. [MUTATING]
27. HCAR-07 — Upload photos, finish. [MUTATING]
28. PLAN-01 — Protection Plans page loads.
29. PLAN-03 — Create/save plan. [MUTATING]
30. TFEE-01 — Trip Fees page loads.
31. TFEE-05 — Save a trip fee. [MUTATING]
32. COUP-01 — Coupons page loads.
33. COUP-03 — Create percentage coupon. [MUTATING]
34. COUP-10 — Delete coupon (cleanup). [MUTATING]

### Band 5 — Booking management
35. GBOOK-02 — My Bookings list + tabs.
36. GBOOK-03 — Open guest booking detail.
37. HBOOK-01 — Bookings For My Cars list.
38. HBOOK-02 — Open host booking detail.
39. INV-01 — Invoices by refund status.
40. GBOOK-04 — Guest cancel >24h full refund. [MUTATING — use disposable booking]

### Band 6 — Chat & secondary
41. CHAT-01 — Chat hub loads.
42. CHAT-04 — Open conversation.
43. CHAT-05 — Send message. [MUTATING]
44. HELP-01 — Help & Support renders.
45. POL-01/02/03 — Policy pages render.
46. DEAL-00 — Dealership microsite domain-gated (negative).
47. LAND-01 — Landing renders (if host discovered).

See `workflows/` for chained end-to-end journeys and `cases/` for the full atomic case list (184 cases). Remaining P1/P2 cases run after their P0 band passes.
