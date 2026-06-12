# Staging Manual Test Execution Results

- **Target:** https://staging.autopilotee.com
- **Driver:** Playwright MCP (Chrome)
- **Scope:** P0 full (Bands 0–6), all mutations allowed
- **Banner observed:** "Test mode — staging environment. Bookings here are not real."

> NOTE: The live staging build diverges from the repo source the test cases were
> extracted from (redesigned home/search UI, updated hero copy). Cases are executed
> against the **live UI**, with divergences from documented (repo-based) expectations
> recorded as findings.

| Case | Band | Result | Notes |
| --- | --- | --- | --- |
| GSRCH-01 | 0 | ✅ PASS* | Home loads, title "Autopilotee Cars", hero + search card + featured cars present. *Divergence: hero reads "Find Your Perfect Ride" (source expected "Rent The Best Qulity Car's"); search is an "Open search options" button opening a modal with "Pickup Location"/"Enter Address" tabs, not inline `booking-card-container`. App is up. |
| GSRCH-03 | 0 | ✅ PASS | Search modal: selected San Jose + default future dates → navigated to `/cars/guest/search?searchMode=location&locationId=1&startDate=...&returnDate=...&driverAge=25&type=CAR`. All expected params present. |
| GSRCH-04 | 0 | ✅ PASS | Seattle (locationId=6, weekday dates) → "Found 2 cars available": CHEVROLET (New badge) + CADILLAC, both "Seattle", price shown, Google map renders with markers. |
| GSRCH-05 | 0 | ✅ PASS | Clicking CHEVROLET card → `/cars/guest/view/1c282407-7f24-4d5e-977e-7ebc83ca2cff?...` carrying forward all date/location params. Real CAR_ID obtained. |
| GSRCH-11 | 0 | ✅ PASS (bonus) | San Jose (locationId=1) → "Found 0 cars available" + "No Cars Available" empty-state card with correct copy + map. |
| GSRCH-13 | 0 | ✅ PASS (bonus) | San Diego (locationId=4) with Saturday return → "Location Closed During Selected Hours" error card + toast "Location is not available for return on Saturday". Operating-hours logic works (time dropdown later limited to 8AM–5PM). |

**Band 0 finding:** Daily-search inventory is sparse — San Jose, Sunnyvale(? untested), San Diego return 0 cars; only Seattle (loc 6) had 2 cars for the tested window. Not a defect per se (seed data), but worth flagging for QA: most seeded locations show no daily availability.

**Reusable data:** CAR_ID=`1c282407-7f24-4d5e-977e-7ebc83ca2cff` (CHEVROLET Trax 2024, Seattle, $150→$135 total) | Seattle locationId=6 | window 2026-06-22→2026-06-25 12PM | Host = `autopiloteecars+206503` (→ USER2 is HOST; USER1 likely GUEST).

| GVCO-01 | 2 | ✅ PASS* | Car detail loads: photo gallery+thumbnails, "CHEVROLET Trax 2024" + New badge, specs (2 seats/REGULAR/automatic), Hosted by autopiloteecars+206503 (5.0), Included-in-price, description, "No ratings yet", price $150→$135 "Before taxes and fees", trip date/time selectors, "On-site at Seattle", 3+ day $15 discount, Driver Age 25+, free-cancellation/flexible-payment/600mi badges. *Divergence: checkout CTA is "Continue" (source said "Book this trip"). |
| AUTH-01 | 1 | ✅ PASS | `?signin=true` opened "Log in with email" modal. USER1 (jidosoju+5) login succeeded: modal closed, returned to `/`, chat notification appeared, nav switched to authenticated (Switch to host/admin + menu items). |
| AUTH-02 | 1 | ✅ PASS | Wrong password → modal stayed open, no session created. (Toast auto-dismissed before capture; modal-remains + no-auth observed.) |
| AUTH-12 | 1 | ✅ PASS | Full page reload → still authenticated (Switch to host/admin buttons persist). |
| AUTH-15 | 1 | ✅ PASS | Guest menu = Home/My Bookings/Cancelled Bookings/Damage Claims/Chats(1)/Allowed to drive/Account. "Switch to host" → label flips to "Switch to guest", host menu = Locations/Insurance/Product Types/Tripfee/Coupons/Damage Claims/Trips(`/bookedmycars`)/Cancelled Trips/Chats/Refund Invoices/Manage ShortID/Account. Role toggle works both ways. USER1 is host+admin capable. |

**Account roles discovered:** USER1 (jidosoju+5) = guest by default, can switch to **host** and **admin**. USER2 (autopilotee+206503) = host of test cars (`autopiloteecars+206503`). USER1 used for both guest & host flows.

| GVCO-14 | 2 | ✅ PASS | "Continue" on detail page (sidebar) → created booking intent, navigated to `/cars/guest/pay/<carId>/e1b076e8-d9f3-44cf-aa5e-e2dd1daf1cb5`. Checkout page rendered. |
| CHK-01 | 2 | ✅ PASS | Checkout renders all sections: "Checkout" heading, Protection options (STANDARD checked / MINIMUM / decline), free-cancellation + mileage notes, terms checkbox (links to /policies/terms, /cancellation, /privacy), car summary card, price breakdown, coupon input, "Book this trip" button. |
| CHK-02 | 2 | ✅ PASS | Totals consistent: Trip Fee $36 (3d@12) + Rental $135 (3d@50, 10% off $15 applied) + Insurance $60 (3d@20) = Total $231.00. "Total Price With Post Trip Charges" $231.00; Security Deposit $500 held until trip completion + 80h. Arithmetic checks out. |
| CHK-03 | 2 | ✅ PASS | "Book this trip" without terms checked → inline error "Please agree to the terms and conditions before proceeding", no Stripe modal, no navigation. Terms gate works. |
| CHK-06 | 2 | ✅ PASS | Terms checked → "Book this trip" opens Stripe Payment Element dialog (card number/expiry/CVC/country=US/ZIP + Link), "Pay now" disabled until card valid. No charge yet. |
| CHK-09 | 3 | ✅ PASS [MUTATING — CHARGED] | Filled success card 4242·4242·4242·4242 exp 04/42 CVC 424 ZIP 24242 → "Pay now" → redirected to `/cars/guest/booking/e1b076e8-...`. Status **BOOKED**. Total $231, card xxxx-4242, Standard Protection, Security Deposit $500 **AUTHORIZED**, timeline: Booking Created → Booking Confirmed $231. Real booking created. |

**Reusable booking:** BOOKING_ID=`e1b076e8-d9f3-44cf-aa5e-e2dd1daf1cb5` (BOOKED, CHEVROLET Trax, 06/22–06/25 Seattle, $231, deposit $500 authorized). Candidate for GBOOK-02/03 (appears in My Bookings) and GBOOK-04 cancel (>24h → full refund) cleanup.

| CHK-10 | 3 | ✅ PASS | `/guest/mybookings` lists the new booking **ID: E1B076E8** (2024 CHEVROLET Trax, 06/22–06/25 12PM, Seattle 840 E Union St, host autopiloteecars+206503) under Upcoming. Page has tabs Upcoming/Ongoing/History/Cancelled; 13 BOOKED bookings present. |
| GBOOK-02 | 5 | ✅ PASS (bonus) | My Bookings list renders with all 4 tabs (Upcoming/Ongoing/History/Cancelled); each card shows status badge, short ID, dates, car, pickup/return location, host. |
| GVCO-05 | 2 | ✅ PASS | Date picker on detail page opens a multi-month calendar modal (Dates/Months toggle, Reset/Save). Selected Jul 13 → Jul 15; Save → URL `startDate`/`returnDate` updated to 2026-07-13/07-15, sidebar shows "07/13/2026 - 07/15/2026", price recalculated. |
| CHK-11 | 3 | ✅ PASS [MUTATING — decline card] | New booking intent (`92e3f343-...`, 07/13–07/15 to avoid conflict). Decline card 4000·0000·0000·0002 → "Pay now" → Stripe SetupIntent confirm returned **HTTP 402** (console error), modal closed, stayed on `/cars/guest/pay/...` (no redirect to confirmation). No BOOKED booking created. Decline handled correctly. |

**Band 2/3 finding (booking conflict detection):** Re-opening the same car for the same already-booked window (06/22–06/25) shows "total: Unable to calculate" + "These dates have a conflict with another trip, please change dates before continue" and disables Continue. Overlap guard works. Changing to free dates (07/13–07/15) cleared it.

**Bands 0–3 status: COMPLETE.** All P0 anonymous, auth, guest-view, and checkout/charge cases pass (with live-UI divergences noted). Orphan unpaid booking intent `92e3f343-...` left from CHK-11 (never charged, not BOOKED).

---

## Band 4 — Host flows (USER1 switched to host)

| Case | Band | Result | Notes |
| --- | --- | --- | --- |
| HLOC-01 | 4 | ✅ PASS | My Listings (`/cars/host`) → "Add location" opened add-location form at `/cars/host/add-location` with name/address/city/state/zip/tax-rate/phone/description fields + Save. |
| HLOC-02 | 4 | ✅ PASS [MUTATING] | Created location **"QA Test Location 0612A"** (id=8). Google Places address autocomplete selected via keyboard (ArrowDown+Enter; suggestions render in `.pac-container`, not a11y tree). Toast "Location created successfully". Minor UX divergence: selecting an address overwrote the Name field, re-typed unique name before save. |
| HLOC-07 | 4 | ✅ PASS [MUTATING] | `/operating-hours/8`. "Edit operating hours" modal: toggled **Saturday** ON (flipped Unavailable→Available 12:00 AM–11:59 PM) → Save. Toast "Operating hours updated successfully" + green banner "Operating hours saved successfully!". Current Operating Hours panel now shows Saturday 12:00 AM–11:59 PM (persisted). Note: per-day toggle pill is not exposed as an a11y node; clicked via positional DOM locator. |
| HCAR-01 | 4 | ✅ PASS [MUTATING] | `/cars/host/add?type=CAR`: title "List your car", location select lists existing locations + "+ Create new location". Selected loc id=8 ("100 Market St, San Francisco") → Next → draft car created, navigated to `/cars/host/add-vin?locationId=8`. |
| HCAR-03 | 4 | ✅ PASS [MUTATING] | VIN `1HGCM82633A004352` → Next → car-info modal opened with NHTSA decode: Make=HONDA, Model=Accord, Year=2003, Gear=Automatic, Fuel=Gasoline, Gas Grade=Regular prefilled (Seats blank). |
| HCAR-05 | 4 | ✅ PASS [MUTATING] | Filled Seats=5, Plate=QA12345, State=CA, City MPG=24, Hwy MPG=34, Doors=4, Category=Sedan, Description. Next button **disabled until all required filled**, then enabled → Next → navigated to `/cars/host/add-price`. Validation gate confirmed. |
| HCAR-06 | 4 | ✅ PASS [MUTATING] | Daily pricing Weekday=50/Weekend=60/Holiday=75 → Next. *Divergence: flow now inserts a `/cars/host/add-monthly-price` step (heading "Set Monthly Car Pricing", optional, off by default) before photos; source docs went straight to add-photos. Skipped monthly (left off) → Next → `/cars/host/add-photos`. |
| HCAR-08 | 4 | ✅ PASS (bonus) | On add-photos with zero photos, "Next" is disabled. |
| HCAR-07 | 4 | ✅ PASS [MUTATING] | Uploaded 3 JPEGs via dropzone file chooser; first photo got "MAIN PHOTO" badge, others got remove (×) buttons, Next enabled → Next → toast **"Car created successfully!"**, navigated to `/cars/host` My Listings. |
| HCAR-09 | 4 | ✅ PASS (bonus) | New listing **"HONDA Accord 2003"** (Base · plate QA12345, "Listed" badge, "No trips yet") appears on My Listings alongside the pre-existing MERCEDES-BENZ G-Class 2023. |

**Reusable data (Band 4):** New host car **HONDA Accord 2003** (plate QA12345) created at location id=8 ("100 Market St, San Francisco"), status Listed. Created via USER1 in host role.

| PLAN-01 | 4 | ✅ PASS | `/cars/host/protection-plans`: heading "Protection Plan Set up", exactly 4 cards in order MINIMUM/STANDARD/PRO/PREMIUM, each with $n/day, "updated on:" date, Active/Inactive. Active=green check, Inactive=red minus icons confirmed. (Divergence: a Daily/Monthly tab toggle is present, not in source docs.) |
| PLAN-02 | 4 | ✅ PASS (bonus) | Clicking MINIMUM card expands inline form: Active toggle (chevron-collapse + switch), Daily plan price, Out of Pocket Maximum, Description, Plan Details, Save. |
| PLAN-03 | 4 | ✅ PASS [MUTATING] | Changed MINIMUM Daily plan price 12.37→**15**, kept Active, Save → toast "Protection plan updated successfully"; card reflects $15/day. (Note: left MINIMUM at $15/day; no delete available for plans.) |
| TFEE-01 | 4 | ✅ PASS | `/cars/host/miscellaneous-fees`: heading "Trip Fees Set up", all 8 rows present (Trip Fee, <21 Youngster, 21-25 Youngster, free cancellation grace period, free cancellation advance in hours, Cancellation Maximum Penalty, shorten booking max penalty, Free Cancellation or Shorten Service Fee), each with charging text, updated date, Active/Inactive icon, and default hint ($0/day, $15/day, $25/day, 1 hour, 24 hours, $50, 50% per day, 0%). |
| TFEE-02 | 4 | ✅ PASS (bonus) | "Free Cancellation or Shorten Service Fee" row shows subtitle "E.g., 3% means a $3000 refund becomes $2910 (to recover Stripe processing fees)". |
| TFEE-03 | 4 | ✅ PASS (bonus) | Trip Fee row expands: Active toggle, charge-by dropdown (Daily Rate/One Time/Percentage), "amount:" input, Plan Details textarea, Save. |
| TFEE-05 | 4 | ✅ PASS [MUTATING] | Trip Fee: toggled Active ON, chargeBy=Daily Rate, amount 12→**20**, Plan Details "QA trip fee", Save → row updates to "20/day · Active" (green check), persisted. |
| COUP-01 | 4 | ✅ PASS | `/cars/host/coupons`: heading "Coupons" + "Add Coupon" button. Existing coupon card: code 0C228Y2 (green mono), name "5offnow", Active badge, Category HOST_CREATED, Algorithm PERCENTAGE (5%), "Valid from: No limit to No limit", Edit + Expire buttons. *Divergence: destructive action labeled "Expire" (not "Delete").* |
| COUP-02 | 4 | ✅ PASS (bonus) | "Add Coupon" → "Create Coupon" form with Name*, Start/End Date, Guest search, Product Type, Category, Minimum Days, Active (checked default), Apply By (Trip default), Price Algorithm (Percentage default), Percentage Off*, Category (EVERYDAYMARKETING default), Apply To Price Of (Daily Price default), Save, Cancel. |
| COUP-03 | 4 | ✅ PASS [MUTATING] | Created percentage coupon Name="QA-PCT-0612A", Percentage Off=10, Active → Save → toast "Coupon created successfully". New card: code **IKN72JU**, Active, EVERYDAYMARKETING, PERCENTAGE (10%). |
| COUP-10 | 4 | ✅ PASS [MUTATING — cleanup] | Clicked "Expire" on QA-PCT-0612A → native confirm "Are you sure you want to expire this coupon? It will be set to inactive." → Accept → toast "Coupon expired successfully"; coupon now **Inactive**. *Divergence from source: action expires (deactivates) the coupon rather than deleting it ("DeleteCoupon" → expire semantics); card remains in list as Inactive.* |

**Band 4 status: COMPLETE.** All P0 host cases pass: HLOC-01/02/07, HCAR-01/03/05/06/07, PLAN-01/03, TFEE-01/05, COUP-01/03/10 (plus bonus HCAR-08/09, PLAN-02, TFEE-02/03, COUP-02). Live-UI divergences noted (extra monthly-pricing step in add-car; Daily/Monthly tabs on plans/fees; "Expire" vs "Delete" coupon).

---

## Band 5 — Booking management

| Case | Band | Result | Notes |
| --- | --- | --- | --- |
| HBOOK-01 | 5 | ✅ PASS | `/bookedmycars`: title "Bookings For My Cars", tabs Upcoming/Ongoing/History/**Cancelled** (extra tab vs source's 3). Card: BOOKED, ID 11E0D608, 06/23–06/25, 2023 MERCEDES-BENZ G-Class, pickup/return 1588 Morris Ave Union NJ, **Guest:** autopiloteecars+10001 (shows Guest not Host). |
| HBOOK-02 | 5 | ✅ PASS | Card → `/cars/host/booking/11e0d608-...`. Host detail: status BOOKED, Trip Details, **Actions card with host buttons** "Cancel Trip Without Penalty", "Modify Trip Without Penalty", "Swap Car", "Contact Guest"; Financials (Total Earnings $452.74, card xxxx-4242), Security Deposit $500 AUTHORIZED, People (Guest+Host), Activity Timeline (Booking Created→Confirmed). |
| INV-01 | 5 | ✅ PASS | `/invoices/refund-status`: heading "Refund Invoices", status filter tabs Pending Review/Processing/Completed/Failed/Cancelled/All. "No refund invoices found for this status" (none for this host). Clicking "All" updates URL `?status=NONE` (filter switching works). |
| GBOOK-03 | 5 | ✅ PASS | First booking card (NISSAN Quest, ID 374197B8) → `/cars/guest/booking/374197b8-...`. Guest detail renders: status BOOKED, car header + "View car details" link, Trip Details (dates, pickup/return Google-maps link, "Free cancellation before"), **Actions card** (Cancel the Trip / Modify the Trip / Contact Host), Financials & Protection (Total $491.36, card xxxx-4242, View Receipt), Security Deposit $500 AUTHORIZED, Trip Photos, Add Additional Driver, Vehicle Documents, People (guest+host), Activity Timeline (4 events incl. Trip Extended). |
| GBOOK-04 | 5 | ✅ PASS [MUTATING — cleanup] | Opened disposable booking E1B076E8 (CHEVROLET Trax, 06/22–06/25, start >24h away) → "Cancel the Trip" → **Confirm Refund** dialog: "full refund of $231.00 + full $500.00 deposit auto-refunded" → Confirm. Toast **"Booking cancelled successfully"**. Page now shows **Refund Details** (Total Refund $231.00, Status REFUND PROCESSED, Stripe $231.00, Note "Automatic full refund - cancellation more than 24 hours before trip"), Security Deposit **RELEASED** ("Automatic release on cancellation"), Total Price $0.00, Timeline adds "Cancelled by Guest -$231.00 GUEST_REQUESTED". Full auto-refund rule confirmed; CHK-09 disposable booking cleaned up. |

**Band 5 status: COMPLETE.** All P0 booking-management cases pass: GBOOK-02/03/04, HBOOK-01/02, INV-01. Guest cancel >24h full-refund rule verified end-to-end; disposable booking from CHK-09 cancelled (cleanup done).

---

## Band 6 — Chat & secondary pages

| Case | Band | Result | Notes |
| --- | --- | --- | --- |
| CHAT-01 | 6 | ✅ PASS | `/chat`: "View Mode:" control with Guest/Host buttons; tab bar Messages (badge **1**, active) + Notifications (badge **5**). Conversation list: 4 cards each with "Booking #" + 8-char id, relative time, last-message preview ("Say hello!" for empty), participant + car make/model, avatar initial, pink unread badge on unread rows. |
| CHAT-04 | 6 | ✅ PASS | Clicked first card → `/chat/1ad168cc-...`. Header: back arrow, avatar "A", title "Booking #97ca1595", subtitle autopiloteecars+206503, **Report User** flag button. Booking card: CADILLAC XT5, Jun 19–24 2026, **BOOKED** (pink). Message bubbles with HH:mm timestamps + "Seen 4h ago". Input placeholder "Type a message..." + send button (disabled while empty). |
| CHAT-05 | 6 | ✅ PASS [MUTATING] | Send button disabled with empty input → typed "Test message from automated QA" → enabled → clicked send. New message appended as right-side bubble at **16:46**, input cleared, send button re-disabled. Message persisted to thread. |
| HELP-01 | 6 | ✅ PASS | `/help`: heading "Help & Support" + subtitle. FAQ section with **7** collapsible questions (incl. all four required). Contact Information: phone +1(866)217-2849 (tel link), email support@autopilotee.com (mailto), address "2150 N first Street STE 426, San Jose, CA 95131". "Call Support Now" → tel:+18662172849. Help Articles: **5** articles each with Read More (→ /, /cars/host, /policies/cancellation, /policies/terms, /policies/privacy). |
| POL-01 | 6 | ✅ PASS | `/policies/terms`: side nav "Terms and policies" (Terms of service / Cancellation policy / Privacy policy / Account deletion), title "Terms of Service" (Last Revised 12/01/2025), markdown body (~72KB) renders. **No literal "Turo"** in visible text (replaced). |
| POL-02 | 6 | ✅ PASS | `/policies/cancellation`: title "Cancellation Policy", body renders (Guest Cancellations / Free Cancellation Period), no "Turo". |
| POL-03 | 6 | ✅ PASS | `/policies/privacy`: title "Privacy Policy", body renders (Personal information we collect), no "Turo". |
| DEAL-00 | 6 | ✅ PASS (negative) | `/cars_on_sale` on staging.autopilotee.com → main app redirects to `/` (home "Find Your Perfect Ride"); **no dealership microsite content** (no cars-on-sale/CarFax). Domain-gating confirmed (microsite only on autopiloteecars.com / jidosojuholding.com). |
| LAND-01 | 6 | ⚠️ N/A | Marketing Next.js landing ("Ready to Own Your Platform?") not reachable in this environment. Attempted: staging.autopilotee.com/demo (→ car-app `/`), www.autopilotee.com & autopilotee.com (both = React car app, not landing), jidosoju.com / www.autopilotee.io / get.autopilotee.com (DNS NXDOMAIN). Marked not-applicable per case-file guidance; landing app is hosted elsewhere/undiscoverable from here. |

**Band 6 status: COMPLETE.** All reachable P0/P1 secondary cases pass: CHAT-01/04/05, HELP-01, POL-01/02/03, DEAL-00. LAND-01 not-applicable (marketing host not discoverable from this environment).

---

## Final — Sign out (run last, after all authenticated bands)

| Case | Band | Result | Notes |
| --- | --- | --- | --- |
| AUTH-13 | 1 | ✅ PASS [MUTATING — session] | `/profile` → "Sign out". Pre-logout: `localStorage.tokens`+`user` present, role=guest. After click: redirected to home `/`; `localStorage.tokens`/`user`/`role` all **removed** (null); header nav now shows **Home / Log in / Sign up**. Session fully cleared and reset to anonymous. |

---

## Execution Summary

**P0 scope (Bands 0–6) COMPLETE.** All P0 cases executed against the live staging UI; all pass except LAND-01 (environment-blocked, marked N/A).

| Band | Cases | Result |
| --- | --- | --- |
| 0 — Anonymous smoke | GSRCH-01/03/04/05 (+11/13 bonus) | ✅ all pass |
| 1 — Authentication | AUTH-01/02/12/15, AUTH-13 (sign out, last) | ✅ all pass |
| 2 — Guest view & checkout entry | GVCO-01/05/14, CHK-01/02/03/06 | ✅ all pass |
| 3 — Checkout charge | CHK-09 (charged), CHK-10, CHK-11 (decline) | ✅ all pass |
| 4 — Host flows | HLOC-01/02/07, HCAR-01/03/05/06/07 (+08/09), PLAN-01/03 (+02), TFEE-01/05 (+02/03), COUP-01/03/10 (+02) | ✅ all pass |
| 5 — Booking management | GBOOK-02/03/04, HBOOK-01/02, INV-01 | ✅ all pass |
| 6 — Chat & secondary | CHAT-01/04/05, HELP-01, POL-01/02/03, DEAL-00 | ✅ all pass; LAND-01 ⚠️ N/A |

**Mutations performed (all approved, cleaned up where possible):**
- CHK-09: charged $231 (success card) → BOOKED booking E1B076E8 → **cancelled in GBOOK-04** (full refund).
- CHK-11: decline card, no charge, no booking. Orphan unpaid intent `92e3f343-...` left (never BOOKED).
- HLOC-02: created location id=8 "QA Test Location 0612A" (no UI delete).
- HCAR-01–07: created car "HONDA Accord 2003" plate QA12345 (no UI delete from listings).
- PLAN-03: MINIMUM plan price set to $15/day (no delete for plans).
- TFEE-05: Trip Fee set to $20/day Active.
- COUP-03 → COUP-10: created coupon QA-PCT-0612A (IKN72JU) then **Expired** (cleanup; expire = deactivate, no hard delete).
- CHAT-05: sent one test chat message in booking #97ca1595 thread.
- AUTH-13: signed out (session cleared).

**Key live-UI divergences from repo-based case docs (findings for QA):**
1. Redesigned home/search: hero "Find Your Perfect Ride" (not "Rent The Best Qulity Car's"); search is a modal "Open search options", not inline booking card.
2. Detail-page checkout CTA is "Continue" (not "Book this trip"); checkout-page button is "Book this trip".
3. Add-car flow inserts an extra `/cars/host/add-monthly-price` step before photos.
4. Protection-plans & miscellaneous-fees pages have a Daily/Monthly tab toggle not in source docs.
5. Coupon destructive action is **"Expire"** (sets Inactive), not "Delete" (no hard removal).
6. Bookings lists have a 4th **"Cancelled"** tab (source documented 3 tabs).
7. Sparse daily inventory: only Seattle (loc 6) had cars for tested windows; San Jose/San Diego returned 0.

**Not-applicable / environment-blocked:** LAND-01 (separate Next.js marketing site, host not discoverable from staging).
