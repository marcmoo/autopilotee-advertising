# Guest Car Detail View & Checkout Entry

Area: Guest car detail page (`/cars/guest/view/:id`) and checkout entry (`/cars/guest/pay/:id/:bookingId`).

Source repo: `/Users/chao/marcmoo/autopilotee-cars`
Primary sources:
- `e2e/guest-car-view-checkout.spec.ts`
- `src/app/containers/GuestCarPage/ViewCarPage/index.tsx`
- `src/app/containers/GuestCarPage/CheckoutPage/index.tsx`
- `src/app/components/chargingDetails/index.tsx`
- `src/app/components/protectionPlan/chooseProtectionPlan/index.tsx`
- `src/app/components/protectionPlan/planRadioList/index.tsx`
- `src/App.tsx` (routes, lines 197-241)

## Conventions for executing these cases

- A reference car-view URL with all query params (from the e2e spec, rewritten to staging) is:
  `https://staging.autopilotee.com/cars/guest/view/<CAR_ID>?driverAge=25&locationId=1&returnDate=2026-10-03T07%3A00%3A00.000Z&returnLocationId=1&returnTime=12%3A00%20PM&searchMode=location&startDate=2026-10-01T07%3A00%3A00.000Z&startTime=12%3A00%20PM&timezone=America%2FLos_Angeles`
- The e2e seed car id `aaaaaaaa-bbbb-4ccc-8ddd-eeeeeeeeeeee` is local-only. On staging, obtain a real `CAR_ID` by browsing a guest search result and opening a car (the URL becomes `/cars/guest/view/<CAR_ID>`).
- The guest user must NOT be the owner of the car (the page blocks "You cannot book your own item").
- Desktop (>=768px wide) and mobile (<768px) render different DOM. Most P0 cases below are written for desktop viewport unless marked mobile.
- The Continue/Book flow MUTATES staging data: `createBookingIntent` creates a booking-intent record. Stop before payment to avoid charging.

---

### GVCO-01: Car detail page loads with title, rating, specs and pre-tax price
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as a guest (USER1 or USER2, whichever is not the car's host). A valid `CAR_ID` is known. Desktop viewport.
- **Steps:**
  1. Navigate to the reference car-view URL with a real `CAR_ID` substituted (see Conventions).
  2. Wait for the page skeleton (`ViewCarPageSkeleton`) to disappear and content to render.
  3. Observe the car title heading.
  4. Observe the rating row next to the star icon.
  5. Observe the vehicle specs row (seats, gas, transmission).
  6. Observe the desktop price block in the right-hand booking column.
- **Expected:**
  - URL stays on `/cars/guest/view/<CAR_ID>`.
  - A car title `<h1>` shows make/model/year (e.g. "FORD Escape ...").
  - A rating value (numeric e.g. "5.0") or the text "New" appears next to the star.
  - Specs show "<n> seats", a gas/fuel label, and "<type> transmission".
  - The price block shows "$<n> total" and the line "Before taxes and fees".
- **Source:** ViewCarPage/index.tsx (CarTitle 1692-1697, RatingText 1700-1703, CarSpecs 1719-1734, PriceInfoComponent 2195-2206), guest-car-view-checkout.spec.ts (203-213)

### GVCO-02: Pickup location displays on the car detail page
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Same as GVCO-01.
- **Steps:**
  1. Navigate to the reference car-view URL with a real `CAR_ID`.
  2. Wait for content to load (depends on GET_LOCATIONS + GET_CAR queries).
  3. Locate the trip-location block in the booking column (`TripLocation` component).
- **Expected:**
  - A location line is shown (the e2e baseline shows "On-site at San Francisco"; on staging it reflects the car's actual pickup location/city).
  - When the pickup location differs from the car's home location (airport), an airport-style label is shown.
- **Source:** ViewCarPage/index.tsx (TripLocation 1926-1934, getLocationById 1410-1438), guest-car-view-checkout.spec.ts (219-233)

### GVCO-03: Photo gallery — thumbnails and "View N photos" open the full-screen modal
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` for a car with more than one image. Desktop viewport.
- **Steps:**
  1. Navigate to the car-view URL.
  2. In the desktop image gallery, click a thumbnail (the first 3 thumbnails are shown).
  3. Click the "View N photos" button (camera icon) overlaid on the last thumbnail.
  4. In the opened modal, click the close (X) button in the header.
- **Expected:**
  - Clicking a thumbnail swaps the main image and the selected thumbnail gets a purple border (`#593CFB`).
  - The "View N photos" button is visible only when the car has more than 3 photos (`totalPhotos > 3`).
  - Clicking it opens a full-screen photo modal showing all images in a grid; page body scroll is locked.
  - Clicking the close button dismisses the modal and restores scroll.
- **Source:** ViewCarPage/index.tsx (ThumbnailContainer 1658-1687, ViewPhotosButton 1672-1683, PhotoModal 2088-2110, scroll lock 1253-1262)

### GVCO-04: Photo gallery on mobile — single thumbnail opens modal
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` for a car with more than one image. Mobile viewport (<768px).
- **Steps:**
  1. Resize to mobile width, navigate to the car-view URL.
  2. Tap the single mobile thumbnail image (or the "View N photos" button if shown).
- **Expected:**
  - The full-screen photo modal opens showing all car images stacked/grid.
  - A heart (favorite) button is shown over the thumbnail.
- **Source:** ViewCarPage/index.tsx (MobileThumbnailWrapper 1494-1537)

### GVCO-05: Date selection via the date-range modal and time selectors
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` known. Desktop viewport. Navigate WITHOUT pre-filled date params (use `/cars/guest/view/<CAR_ID>`).
- **Steps:**
  1. Navigate to `/cars/guest/view/<CAR_ID>` (no date query params).
  2. In the "Your trip" section, click the date-range trigger button (shows "Select dates").
  3. In the opened `DateRangeModal`, pick a future start date and a later end date, then save.
  4. Change the pickup time select (left) and the return time select (right).
- **Expected:**
  - The trigger initially shows "Select dates" with sub-text "Select pickup and return dates".
  - After saving, the trigger shows the selected range formatted "MM/DD/YYYY - MM/DD/YYYY".
  - If the chosen range is shorter than the car's minimum rental days, the end date auto-adjusts forward to satisfy the minimum.
  - Both time selects update and the price recalculates ("$<n> total" or a calculating spinner appears).
  - The URL query string updates (replace) with startDate/returnDate/startTime/returnTime/driverAge/timezone.
- **Source:** ViewCarPage/index.tsx (DesktopTripSection 1878-1923, DateRangeModal onSave 2065-2085, URL sync effect 899-930)

### GVCO-06: Date conflict / unavailable dates show an inline error and block continue
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` for a car that has at least one existing booked or host-blocked date range. Desktop viewport.
- **Steps:**
  1. Navigate to `/cars/guest/view/<CAR_ID>`.
  2. Open the date modal and select a range that overlaps a booked/unavailable period (blocked tiles are visually disabled).
  3. Observe the trip section.
  4. Click "Continue".
- **Expected:**
  - An inline red error box appears with either "These dates have a conflict with another trip, please change dates before continue" or "This car is not available for the selected dates, please choose different dates".
  - The date wrapper shows the error styling and shakes; a toast with the same message appears.
  - No navigation to the checkout page occurs.
- **Source:** ViewCarPage/index.tsx (checkDateConflict 1045-1098, conflict effect 1100-1162, handleContinueBooking 1164-1174, DateErrorContainer 1909-1922)

### GVCO-07: Driver age selector defaults to 25+ and offers three brackets
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` known. Desktop viewport. Use the reference URL with `driverAge=25`.
- **Steps:**
  1. Navigate to the reference car-view URL (driverAge=25) with a real `CAR_ID`.
  2. Locate the "Driver Age" select in the booking column.
  3. Open the select and inspect options.
  4. Change selection to "18-20".
- **Expected:**
  - The age select has exactly three options: "18-20" (value 18), "21-24" (value 21), "25+" (value 25).
  - With `driverAge=25` in the URL, the "25+" option (value 25) is selected by default.
  - Changing the age triggers a price recalculation (a youngster fee may apply for younger brackets, visible later on checkout).
- **Source:** ViewCarPage/index.tsx (StyledAgeSelect 1950-1960, driverAge state 848), guest-car-view-checkout.spec.ts (241-244)

### GVCO-08: Pre-tax pricing breakdown shows discount when a multi-day discount applies
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` for a car/host that offers a multi-day discount tier. Desktop viewport.
- **Steps:**
  1. Navigate to `/cars/guest/view/<CAR_ID>`.
  2. Select a date range long enough to trigger a discount tier (e.g. a week).
  3. Observe the price block and the "Trip Savings" section.
- **Expected:**
  - When `discountAmount > 0`, the price shows the original price struck through followed by the discounted "$<n> total".
  - A "Trip Savings" box shows the discount tier label and the discount amount "$<n>".
  - "Before taxes and fees" caption remains.
- **Source:** ViewCarPage/index.tsx (PriceInfoComponent discount 2193-2206, TripSavingsSection 1936-1947)

### GVCO-09: Cancellation policy, flexible payment, and distance-included info render
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` known with selected dates. Desktop viewport.
- **Steps:**
  1. Navigate to the reference car-view URL with dates.
  2. Scroll the booking column to the policy section.
- **Expected:**
  - "Free cancellation" with "Full refund before <date>, <time>" (deadline = trip start minus the host's FREE_CANCELLATION_ADVANCE hours, default 24h).
  - "Flexible payment" note: "$0 due now when you choose the Refundable option at checkout." (desktop only).
  - If the car has a distance limit: "Distance included" shows "Unlimited" or "<n> total miles • <n> miles per day" (desktop only).
- **Source:** ViewCarPage/index.tsx (cancellationDeadline 1285-1332, PolicySection 1970-1989, PaymentOptionsSection 1992-2004, DistanceSection 2007-2042)

### GVCO-10: Continue creates a booking intent and navigates to checkout (happy path)
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as guest who is NOT the car's host. `CAR_ID` known. Valid future date range with no conflicts and a successfully calculated price. Desktop viewport.
- **Steps:**
  1. Navigate to the reference car-view URL with valid dates and `CAR_ID`.
  2. Wait for the price to show "$<n> total" (not "Calculating..." / "Unable to calculate").
  3. Confirm the "Continue" button is enabled (not "Loading...").
  4. [MUTATING] Click "Continue".
- **Expected:**
  - `CreateBookingIntent` GraphQL mutation succeeds.
  - URL navigates to `/cars/guest/pay/<CAR_ID>/<bookingId>` where `<bookingId>` is a UUID.
  - The checkout page renders with the "Checkout" title.
- **Source:** ViewCarPage/index.tsx (handleContinueBooking 1164-1238, ContinueButton 1963-1967), guest-car-view-checkout.spec.ts (246-269), App.tsx (240-241)

### GVCO-11: Guest cannot book their own car
- **Priority:** P1  **Persona:** Host (acting as guest on own listing)
- **Preconditions:** Signed in as the user who OWNS the target car. `CAR_ID` = a car owned by that user. Desktop viewport.
- **Steps:**
  1. Navigate to `/cars/guest/view/<OWN_CAR_ID>`.
  2. Select valid dates.
  3. Click "Continue".
- **Expected:**
  - A toast error appears: "You cannot book your own car" (or "...your own item" for non-car product types).
  - No navigation to the checkout page; no booking intent created.
- **Source:** ViewCarPage/index.tsx (handleContinueBooking owner check 1182-1186)

### GVCO-12: Continue while unauthenticated prompts sign-in
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** NOT signed in (no tokens in storage). `CAR_ID` known. Desktop viewport.
- **Steps:**
  1. Navigate to `/cars/guest/view/<CAR_ID>` while logged out.
  2. Select valid dates.
  3. Click "Continue".
- **Expected:**
  - The booking-intent call fails with `NoTokenProvided` and the sign-in modal opens (instead of navigating to checkout).
- **Source:** ViewCarPage/index.tsx (handleContinueBooking catch / openSignInModal 1231-1237)

### GVCO-13: Mobile fixed price bar shows price and continues
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest; `CAR_ID` known. Mobile viewport (<768px).
- **Steps:**
  1. Resize to mobile width, navigate to `/cars/guest/view/<CAR_ID>`.
  2. Before selecting dates, observe the bottom fixed bar.
  3. Select valid dates via the mobile trip section.
  4. [MUTATING] Tap "Continue" in the fixed bottom bar.
- **Expected:**
  - Before dates: bottom bar shows "Select dates to see price".
  - After valid dates: bar shows "$<n> total" and "Before taxes and fees" with a Continue button.
  - Tapping Continue performs the same booking-intent + navigation as desktop (GVCO-10).
- **Source:** ViewCarPage/index.tsx (MobileFixedPriceBar 2047-2063, MobilePriceInfoComponent 2210-2289)

### GVCO-14: Checkout page renders title, protection options, car summary and price breakdown
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Arrived on checkout via GVCO-10 (URL `/cars/guest/pay/<CAR_ID>/<bookingId>`). Desktop viewport.
- **Steps:**
  1. From GVCO-10, land on the checkout page (or navigate directly to a valid `/cars/guest/pay/<CAR_ID>/<bookingId>`).
  2. Wait for `CheckoutPageSkeleton` to disappear.
  3. Observe the title, protection section, car summary, and price breakdown.
- **Expected:**
  - `data-testid="checkout-page-container"` present.
  - `data-testid="checkout-title"` shows "Checkout".
  - `data-testid="checkout-protection-section"` shows "Protection options" heading and a description mentioning Travelers insurance.
  - Right column shows car image, car name, "5.0 ★", date range ("MMM dd yyyy HH:mm → ..."), and pickup location/city.
  - A price breakdown (ChargingDetails) lists rows such as Rental Price (n days @ $/day), Insurance, Trip Fee, Youngster Fee (if young driver), Tax (%), and a "Total" row, with a security-deposit note when applicable.
- **Source:** CheckoutPage/index.tsx (container/title 1184-1197, CarInfoWrapper 1259-1295, ChargingDetails 1298-1300), chargingDetails/index.tsx (getRenderingByKey 89-150, Total 249-263)

### GVCO-15: Protection plan selection updates the booking
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On the checkout page for a valid booking intent. The host has more than one active protection plan. Desktop viewport.
- **Steps:**
  1. On checkout, locate the protection plan radio list inside the protection section.
  2. A non-declined plan is auto-selected by default; note the selected plan's green-tinted style.
  3. [MUTATING] Click a different (non-declined) protection plan radio.
  4. (Optional) Click "No thanks, I'd like to decline damage protection".
- **Expected:**
  - Selecting a plan calls `UpdateBookingProtectionPlan` and refetches `GetBooking`; the price breakdown / insurance row updates accordingly.
  - The selected radio shows the checked (green) style.
  - Clicking the decline link opens a "Decline protection plan" dialog with "Decline" and "Choose Minimum Coverage" actions; choosing one sets the corresponding plan.
- **Source:** CheckoutPage/index.tsx (ChooseProtectionPlan 1196, setAndUpdateSelectedPlan 1125-1136), chooseProtectionPlan/index.tsx (default select 51-65, decline dialog 68-91), planRadioList/index.tsx (39-90)

### GVCO-16: Coupon code application updates the receipt
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On the checkout page for a valid booking intent. A valid coupon code is known (ask executor / staging data). Desktop viewport.
- **Steps:**
  1. Locate `data-testid="checkout-coupon-section"`.
  2. Type a coupon into `data-testid="checkout-coupon-input"` (auto-uppercased, max 7 chars).
  3. [MUTATING] Click `data-testid="checkout-coupon-apply-button"` ("Apply").
- **Expected:**
  - Valid coupon: shows `data-testid="checkout-coupon-success"` "Coupon <CODE> applied successfully!"; a new booking intent is created with the coupon and the receipt shows a "Coupon Discount (<CODE>)" row with a negative green amount; URL bookingId may change (replace).
  - Invalid/empty coupon: shows `data-testid="checkout-coupon-error"` with the validation message; no discount applied.
- **Source:** CheckoutPage/index.tsx (validateCoupon 671-759, CouponSection 1302-1333), chargingDetails/index.tsx (Coupon Discount row 234-247)

### GVCO-17: Terms agreement is required before Book this trip
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On the checkout page for a valid booking intent. Desktop viewport.
- **Steps:**
  1. With the terms checkbox UNCHECKED, click `data-testid="checkout-book-button-desktop"` ("Book this trip").
  2. Observe the terms section.
  3. Click `data-testid="terms-agreement-desktop"` to check the box.
- **Expected:**
  - With terms unchecked: a toast "Please agree to the terms and conditions before proceeding" appears, the terms box gets red error styling and shakes, the page scrolls to it, and no payment form opens.
  - The terms label links to /policies/terms, /policies/cancellation, /policies/privacy.
  - After checking the box (and only then), proceeding moves to phone verification or the payment form.
- **Source:** CheckoutPage/index.tsx (handleMakePayment terms check 860-869, TermsSectionDesktop 1213-1255, scrollToTermsAndShake 843-858)

### GVCO-18: Book this trip with verified phone opens the Stripe payment form (stop before charge)
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** On the checkout page for a valid booking intent. Terms checkbox CHECKED. Guest phone already verified (`isPhoneVerified=true`). Desktop viewport.
- **Steps:**
  1. Check the terms agreement box.
  2. [MUTATING] Click `data-testid="checkout-book-button-desktop"` ("Book this trip").
  3. Observe the loading overlay then the Stripe payment form.
  4. STOP. Do not submit the Stripe form (do not charge the card).
- **Expected:**
  - A "Preparing payment form..." loading overlay appears.
  - If a security deposit > 0, a Stripe SetupIntent is created and the Stripe Elements payment form opens (deposit step); otherwise a PaymentIntent is created (rental step).
  - The Stripe card form is visible and ready for input. No charge occurs until the form is submitted.
- **Source:** CheckoutPage/index.tsx (handleMakePayment 860-934, renderPaymentSection 1142-1180, Elements/PaymentForm 1150-1178)

### GVCO-19: Unverified phone triggers phone-verification modal before payment
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On the checkout page. Terms checked. Guest phone NOT verified (`isPhoneVerified=false`). Desktop viewport.
- **Steps:**
  1. Check the terms box.
  2. Click "Book this trip".
- **Expected:**
  - The `PhoneVerification` modal opens (no payment form yet).
  - After successful verification, the flow continues to the payment form (GVCO-18) automatically.
- **Source:** CheckoutPage/index.tsx (phone check 871-875, PhoneVerification 1396-1405)

### GVCO-20: Invalid/expired booking intent on checkout redirects home
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest. Use a `/cars/guest/pay/<CAR_ID>/<bogusBookingId>` URL where the booking does not exist.
- **Steps:**
  1. Navigate directly to the checkout URL with a non-existent `bookingId` (use a random UUID).
- **Expected:**
  - `GetBooking` errors; a toast alert "Booking not found" appears with an action to navigate to the home page; choosing it (or auto) navigates to `/`.
- **Source:** CheckoutPage/index.tsx (useGetBookingQuery onError 783-796)
