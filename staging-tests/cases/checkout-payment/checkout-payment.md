# Checkout & Stripe Payment — Manual Test Cases

App: https://staging.autopilotee.com
Route: `/cars/guest/pay/:id/:bookingId` (CheckoutPage)
Confirmation route: `/cars/guest/booking/:bookingId`

## Selectors and operations reference (from source)
- Page container: `data-testid="checkout-page-container"`
- Title: `data-testid="checkout-title"` (text "Checkout")
- Protection section: `data-testid="checkout-protection-section"`
- Free cancellation (desktop): `data-testid="free-cancellation-desktop"`
- Terms checkbox (desktop): `data-testid="terms-agreement-desktop"` (role="checkbox", aria-checked)
- Terms checkbox (mobile): `data-testid="terms-agreement-mobile"`
- Coupon section: `data-testid="checkout-coupon-section"`
- Coupon input: `data-testid="checkout-coupon-input"` (placeholder "Enter coupon code", maxLength 7, auto-uppercases)
- Coupon apply: `data-testid="checkout-coupon-apply-button"` (text "Apply" / "Applying...")
- Coupon error: `data-testid="checkout-coupon-error"`
- Coupon success: `data-testid="checkout-coupon-success"`
- Book button (desktop): `data-testid="checkout-book-button-desktop"` (text "Book this trip")
- Book button (mobile): `data-testid="checkout-book-button-mobile"` (text "Book Now")
- Stripe modal Pay button: `<button type="submit">Pay now` (purple; disabled until Stripe Element `complete`)
- Stripe Payment Element iframe title: "Secure payment input frame"
- Terms error text: "Please agree to the terms and conditions before proceeding"
- GraphQL ops: `GetBooking`, `GetCar`, `UpdateBookingProtectionPlan`, `CreateSetupIntentForBooking`, `CreateDepositPaymentIntent`, `CreateRentalPaymentIntent`, `CreatePaymentIntent`, `VerifyBookingPayment`, `CheckCarAvailability`, `ValidateCoupon`, `CreateBookingIntent`
- Payment flow: if `securityDeposit > 0` -> deposit-first flow (SetupIntent to collect card, then deposit PI + rental PI). If deposit == 0 -> legacy single PaymentIntent.

Note on preconditions: every checkout case needs an existing booking intent (a `bookingId`) for the signed-in guest. On staging this is produced by going through car search -> car view -> selecting dates -> "Continue/Book", which lands on `/cars/guest/pay/:carId/:bookingId`. The executor must create that booking intent first (non-mutating until "Pay now" is clicked), or reuse a known staging bookingId.

---

### CHK-01: Checkout page renders all required sections for a valid booking
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in as the guest account. A booking intent exists; you are on `/cars/guest/pay/:carId/:bookingId` (reach it from a car detail page by selecting dates and clicking the book/continue CTA).
- **Steps:**
  1. Navigate to the checkout URL `/cars/guest/pay/:carId/:bookingId`.
  2. Wait for the skeleton to disappear (page content loads).
  3. Observe the page.
- **Expected:**
  - URL matches `/cars/guest/pay/<carId>/<bookingId>`.
  - "Checkout" title visible (`data-testid="checkout-title"`).
  - "Protection options" section visible with description starting "Choose a protection plan to protect you in the event of damage...".
  - Car name and date range (`MMM dd yyyy HH:mm → MMM dd yyyy HH:mm`) visible in the right summary.
  - Price breakdown shows a "Rental Price (N days @ X/day)" row, a "Tax (..%)" row, and a "Total" row with a dollar amount.
  - "Security Deposit ... held until trip completion" text visible.
  - Coupon section ("Have a coupon code?") with input and "Apply" button visible.
  - Terms agreement checkbox visible (`data-testid="terms-agreement-desktop"`).
  - "Book this trip" button visible and enabled (`data-testid="checkout-book-button-desktop"`).
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATIONS 1-10); src/app/containers/GuestCarPage/CheckoutPage/index.tsx

### CHK-02: Price totals are mathematically consistent (rental + tax = total before deposit)
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** On the checkout page for a valid booking (CHK-01 passes).
- **Steps:**
  1. Read the "Rental Price" amount.
  2. Read the "Tax (..%)" amount.
  3. Read the "Total" amount.
- **Expected:**
  - Rental Price + Tax equals the displayed Total (e.g. seed data: $82.50 + $7.42 = $89.93).
  - The "Rental Price" small label "(N days @ price/day)" matches price/day x N days = Rental Price.
  - The mobile fixed footer "total:" value (if visible at <768px) equals the desktop Total, labeled "After taxes and fees".
  - Security Deposit amount is shown separately and is NOT added into the Total line (it is held, not charged here).
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATIONS 4-6); src/app/components/chargingDetails/index.tsx

### CHK-03: Terms validation blocks payment when checkbox is unchecked
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** On the checkout page; terms checkbox is unchecked.
- **Steps:**
  1. Ensure the terms checkbox (`data-testid="terms-agreement-desktop"`) is unchecked (aria-checked="false").
  2. Click "Book this trip" (`data-testid="checkout-book-button-desktop"`).
- **Expected:**
  - No Stripe modal opens.
  - Error message "Please agree to the terms and conditions before proceeding" appears (inline under the checkbox and as a toast).
  - The terms checkbox container highlights red and shake-animates.
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATION 11); CheckoutPage/index.tsx handleMakePayment

### CHK-04: Terms checkbox toggles checked/unchecked state
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On the checkout page.
- **Steps:**
  1. Click the terms checkbox (`data-testid="terms-agreement-desktop"`).
  2. Observe state.
  3. Click it again.
- **Expected:**
  - After first click: aria-checked="true" and a white check icon appears in the box.
  - After second click: aria-checked="false" and the check icon is removed.
  - Keyboard: focusing the checkbox and pressing Space/Enter also toggles it.
- **Source:** CheckoutPage/index.tsx (terms-agreement-desktop onClick/onKeyDown)

### CHK-05: Terms policy links navigate to correct policy pages
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On the checkout page.
- **Steps:**
  1. Click the "terms of service" link in the terms label.
  2. Go back; click "cancellation policy".
  3. Go back; click "privacy policy".
- **Expected:**
  - "terms of service" navigates to `/policies/terms`.
  - "cancellation policy" navigates to `/policies/cancellation`.
  - "privacy policy" navigates to `/policies/privacy`.
  - Clicking a link does NOT toggle the checkbox (stopPropagation).
- **Source:** CheckoutPage/index.tsx (TermsLink to="/policies/...")

### CHK-06: Opening the Stripe payment modal (deposit-first flow) with terms accepted
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** On the checkout page. Guest phone is already verified (otherwise the PhoneVerification modal appears first — see CHK-12). Booking has a security deposit > 0.
- **Steps:**
  1. Check the terms checkbox so aria-checked="true".
  2. Click "Book this trip".
  3. Wait for the payment modal.
- **Expected:**
  - A "Preparing payment form..." loading overlay may briefly show (CreateSetupIntentForBooking runs).
  - A modal opens containing the Stripe Payment Element (iframe title "Secure payment input frame") with Card number / Expiration / CVC / ZIP fields.
  - A "Pay now" button is present, initially disabled (gray) until the card form is complete.
  - No charge has occurred yet.
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATIONS 12-13); CheckoutPage/index.tsx handleMakePayment; PaymentForm/index.tsx

### CHK-07: Pay now stays disabled until all Stripe fields are valid
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Stripe payment modal open (CHK-06).
- **Steps:**
  1. Observe "Pay now" before entering anything.
  2. Inside the iframe enter only card number 4242 4242 4242 4242, leave expiry/CVC/ZIP empty.
  3. Fill expiry (e.g. 04/42), CVC (e.g. 424), ZIP (e.g. 24242).
- **Expected:**
  - With empty/partial fields, "Pay now" is disabled (gray, `bg-gray-400 cursor-not-allowed`).
  - Once all fields are valid (Stripe `onChange complete: true`), "Pay now" becomes enabled (purple).
- **Source:** PaymentForm/index.tsx (BookButton disabled={!stripe || !isComplete}); e2e spec VERIFICATION 13

### CHK-08: Invalid / incomplete card shows Stripe inline validation
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Stripe payment modal open.
- **Steps:**
  1. In the card number field enter an invalid number, e.g. `4242 4242 4242 4241`.
  2. Move focus away from the field.
  3. Enter a past expiry date, e.g. `01/20`.
- **Expected:**
  - Stripe shows inline error "Your card number is invalid." under the card field.
  - Expiry shows "Your card's expiration date is in the past."
  - "Pay now" remains disabled while any field is invalid.
- **Source:** PaymentForm/index.tsx (Stripe PaymentElement validation); Stripe Elements behavior

### CHK-09: Successful payment with test card 4242 creates a BOOKED booking
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Stripe modal open, terms accepted, phone verified, deposit-first flow.
- **Steps:**
  1. In the Stripe iframe enter card number `4242 4242 4242 4242`. [MUTATING]
  2. Enter expiry `04/42`, CVC `424`, ZIP `24242`. [MUTATING]
  3. Wait for "Pay now" to enable, then click it. [MUTATING — charges the test card / authorizes deposit]
  4. Wait for processing overlays ("Authorizing $X security deposit...", then "Processing rental payment...", then "Payment successful, verifying booking...").
- **Expected:**
  - SetupIntent confirms, then CreateDepositPaymentIntent and CreateRentalPaymentIntent fire, then VerifyBookingPayment / GetBooking polling confirms status `BOOKED`.
  - Browser navigates to `/cars/guest/booking/<bookingId>` (booking confirmation page).
  - Status badge "BOOKED" visible.
  - "Trip Details" section, "Total Price:" line, and "Security Deposit" section visible.
  - Card last-4 shown as `...4242`.
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATIONS 13-14); CheckoutPage/index.tsx onPaymentSuccessCallback / handleRentalPayment

### CHK-10: Booking confirmation appears in My Bookings list after payment
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** CHK-09 completed successfully; a freshly BOOKED booking exists.
- **Steps:**
  1. Navigate to `/guest/mybookings`.
  2. Ensure the "Upcoming" tab is selected.
  3. Locate the new booking card.
- **Expected:**
  - "My Bookings" heading (h1) visible; "Upcoming" tab visible.
  - The booking shows a "BOOKED" badge and an "ID: <SHORTID>" badge (first 8 chars of bookingId, uppercase).
  - Date range, car name, host name, pickup location and return location are displayed.
- **Source:** e2e/guest-checkout-payment.spec.ts (VERIFICATION 15)

### CHK-11: Declined card shows error and does not create a booking
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Stripe modal open, terms accepted, phone verified.
- **Steps:**
  1. In the Stripe iframe enter the Stripe decline test card `4000 0000 0000 0002`. [MUTATING — attempts a charge]
  2. Enter any future expiry, any CVC, any ZIP.
  3. Click "Pay now". [MUTATING]
- **Expected:**
  - Stripe rejects with an error toast such as "Your card was declined." (error.message surfaced via toast).
  - The payment modal closes (onClose runs in finally).
  - No navigation to `/cars/guest/booking/...`; the booking remains unconfirmed (not BOOKED).
  - If a deposit was authorized first, backend auto-refunds it (toast "Rental payment failed. Your deposit will be automatically refunded." may appear when rental leg fails).
- **Source:** PaymentForm/index.tsx (error -> toast -> onPaymentFailedCallback); CheckoutPage/index.tsx handleRentalPayment catch

### CHK-12: Unverified phone triggers phone verification before payment
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Signed in as a guest whose `isPhoneVerified` is false. On checkout page with terms accepted.
- **Steps:**
  1. Check the terms checkbox.
  2. Click "Book this trip".
- **Expected:**
  - Instead of the Stripe modal, the PhoneVerification modal opens.
  - After completing phone verification, payment preparation continues automatically (handleMakePayment is re-invoked with skipPhoneCheck) and the Stripe modal opens.
  - If phone is already verified, this modal is skipped.
- **Source:** CheckoutPage/index.tsx handleMakePayment (showPhoneModal), PhoneVerification

### CHK-13: Apply a valid coupon updates the receipt totals
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On checkout page. A known-valid staging coupon code exists (ask executor to discover one; codes are <=7 chars, auto-uppercased).
- **Steps:**
  1. Type the coupon code into `data-testid="checkout-coupon-input"`.
  2. Click "Apply" (`data-testid="checkout-coupon-apply-button"`).
  3. Wait for validation (button shows "Applying...").
- **Expected:**
  - ValidateCoupon query returns isValid=true.
  - Success message "Coupon applied successfully!" then the section shows "Coupon <CODE> applied successfully!" (`data-testid="checkout-coupon-success"`).
  - A new booking intent is created with the coupon (CreateBookingIntent) and the URL replaces to `/cars/guest/pay/<carId>/<newBookingId>`.
  - The price breakdown reflects a discount line (e.g. "X% off of ... Applied") and a reduced Total/Tax.
- **Source:** CheckoutPage/index.tsx (handleApplyCoupon, validateCoupon onCompleted, createBookingIntent); couponService/queries.ts ValidateCoupon

### CHK-14: Apply an invalid coupon shows an error and does not change totals
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On checkout page.
- **Steps:**
  1. Type a clearly invalid code, e.g. `ZZZZZ`, into the coupon input.
  2. Click "Apply".
- **Expected:**
  - ValidateCoupon returns isValid=false; error text shown in `data-testid="checkout-coupon-error"` (the backend errorMessage or "Invalid coupon").
  - No success message; appliedCouponCode stays null.
  - Total/Tax unchanged; URL unchanged (no new booking intent).
- **Source:** CheckoutPage/index.tsx (validateCoupon onCompleted else branch / onError)

### CHK-15: Empty coupon Apply is blocked
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On checkout page, coupon input empty.
- **Steps:**
  1. Leave the coupon input empty.
  2. Attempt to click "Apply".
- **Expected:**
  - The "Apply" button is disabled while the input is empty/whitespace (`disabled={validatingCoupon || !couponCode.trim()}`).
  - If somehow triggered, error "Please enter a coupon code" appears and no validation request is sent.
- **Source:** CheckoutPage/index.tsx (ApplyButton disabled, handleApplyCoupon guard)

### CHK-16: Coupon input enforces uppercase and 7-char max
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On checkout page.
- **Steps:**
  1. Type lowercase text `abcdefghij` into the coupon input.
- **Expected:**
  - Input value displays uppercase ("ABCDEFG") and is truncated to 7 characters (maxLength=7).
- **Source:** CheckoutPage/index.tsx (CouponInput onChange toUpperCase, maxLength={7})

### CHK-17: Selecting a protection plan updates the booking and receipt
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On checkout page; multiple protection plans available.
- **Steps:**
  1. In the "Protection options" section click a protection plan option.
  2. Wait for the receipt to update.
- **Expected:**
  - UpdateBookingProtectionPlan mutation fires; GetBooking refetches.
  - The selected plan card is highlighted (blue border).
  - The price breakdown updates to include the plan's insurance/protection cost and the Total recalculates accordingly.
- **Source:** CheckoutPage/index.tsx (setAndUpdateSelectedPlan, useUpdateBookingProtectionPlanMutation refetchQueries ['GetBooking']); components/protectionPlan/chooseProtectionPlan

### CHK-18: Free cancellation banner shows correct deadline (24h before start)
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On checkout page with a future booking start date.
- **Steps:**
  1. Read the green "Free cancellation before ..." banner (`data-testid="free-cancellation-desktop"`).
  2. Compare to the booking start date in the car summary.
- **Expected:**
  - Banner text "Free cancellation before <date>—if plans change, we've got your back." with date = start date minus 24 hours, formatted "MMM d, yyyy, h:mm a".
- **Source:** CheckoutPage/index.tsx (freeCancellationDate useMemo)

### CHK-19: Invalid / missing booking ID redirects with "Booking not found"
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Signed in as guest.
- **Steps:**
  1. Navigate to `/cars/guest/pay/<validCarId>/00000000-0000-0000-0000-000000000000` (a bookingId that does not exist).
- **Expected:**
  - GetBooking errors; a toast alert "Booking not found" appears.
  - Acknowledging the alert navigates to the home page `/`.
- **Source:** CheckoutPage/index.tsx (useGetBookingQuery onError -> toastActions.showToastAlert -> navigate('/'))

### CHK-20: Car becomes unavailable during checkout blocks confirmation
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On checkout page; the selected car/date range has been booked by someone else between intent creation and payment (hard to stage; may require coordination or a date that conflicts).
- **Steps:**
  1. Accept terms and open the Stripe modal.
  2. Enter the test card 4242 and click "Pay now". [MUTATING — attempts payment]
- **Expected:**
  - After Stripe confirms the SetupIntent, `beforeTransactionFinish` runs CheckCarAvailability; if isAvailable=false it rejects.
  - Error toast "This car is no more available for the selected dates" appears.
  - Modal closes; no BOOKED booking is created; any deposit hold is reversed by backend.
- **Source:** CheckoutPage/index.tsx (renderPaymentSection beforeTransactionFinish -> checkCarAvailability)

### CHK-21: Mobile layout — Book Now footer button and mobile terms
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Viewport width < 768px (mobile). On checkout page.
- **Steps:**
  1. Resize the browser to a mobile width (e.g. 390x844).
  2. Observe the fixed bottom bar.
  3. Check the mobile terms checkbox (`data-testid="terms-agreement-mobile"`) and tap "Book Now" (`data-testid="checkout-book-button-mobile"`).
- **Expected:**
  - A fixed bottom bar shows "total:", the total price (in red), and "After taxes and fees".
  - "Book Now" button visible in the footer; desktop "Book this trip" is hidden.
  - Tapping "Book Now" with mobile terms unchecked shows the same terms error; with terms checked it opens the Stripe modal (same flow as desktop).
- **Source:** CheckoutPage/index.tsx (FixedButtonContainer, MobilePriceSummary, terms-agreement-mobile)
