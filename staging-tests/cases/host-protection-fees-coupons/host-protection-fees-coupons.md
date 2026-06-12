# Host Protection Plans, Trip Fees, and Coupons

Manual test cases for the Host-facing configuration pages: protection plans, miscellaneous trip fees, and coupons, plus the guest-side coupon application flow at checkout.

Base URL: https://staging.autopilotee.com

Key routes (from `src/App.tsx`):
- Protection Plans: `/cars/host/protection-plans`
- Trip Fees ("miscellaneous fees"): `/cars/host/miscellaneous-fees`
- Coupons: `/cars/host/coupons`

Role note: These pages render under the authenticated host routes. The account with host role must be used. USER1 or USER2 has host role (executor discovers which). If a non-host account lands here, GraphQL queries (GetProtectionPlans / GetTripFees / GetCoupons) may error or return empty.

GraphQL operations referenced:
- Protection: `GetProtectionPlans`, `CreateProtectionPlan`, `UpdateProtectionPlan`
- Trip fees: `GetTripFees`, `CreateTripFee`, `UpdateTripFee`
- Coupons: `GetCoupons`, `CreateCoupon`, `UpdateCoupon`, `DeleteCoupon`, `SearchGuests`, `ValidateCoupon`

---

### PLAN-01: Protection Plans page loads with all four plan types
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as the host account.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/protection-plans
  2. Wait for the page to finish loading (skeleton disappears).
- **Expected:**
  - Heading "Protection Plan Set up" is visible.
  - Exactly four plan cards are listed in order: MINIMUM, STANDARD, PRO, PREMIUM.
  - Each card shows a `$<n>/day` price, an "updated on: <date>" line, and an "Active" or "Inactive" status.
  - Active plans show a green check-circle icon; inactive plans show a red minus-circle icon.
  - `GetProtectionPlans` GraphQL query returns 200 with no errors.
- **Source:** src/app/containers/ProtectPlanPage/index.tsx; e2e/host-protection-plans.spec.ts; src/App.tsx

### PLAN-02: Expand a protection plan to reveal the edit form
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On the Protection Plans page (PLAN-01 passed).
- **Steps:**
  1. Click the MINIMUM plan card.
- **Expected:**
  - An inline form expands directly under the card.
  - Form contains: "Active" toggle switch (`button[role="switch"]`), "Daily plan price" numeric input, "Out of Pocket Maximum" numeric input, "Description" textarea, "Plan Details" textarea, and a "Save" button.
  - A collapse chevron button (chevron-up) appears at the top of the form.
- **Source:** src/app/containers/ProtectPlanPage/index.tsx (EachPlan, lines 340-394)

### PLAN-03: Create/save a protection plan with active toggle and price
- **Priority:** P0  **Persona:** Host
- **Preconditions:** On the Protection Plans page; MINIMUM plan form expanded (PLAN-02).
- **Steps:**
  1. Note the current state of the "Active" switch (aria-checked value).
  2. [MUTATING] Click the "Active" toggle switch so it flips state.
  3. [MUTATING] Clear "Daily plan price" and type `15`.
  4. [MUTATING] Type `500` into "Out of Pocket Maximum".
  5. [MUTATING] Type a description, e.g. `Test minimum plan` into the Description textarea.
  6. [MUTATING] Click "Save".
- **Expected:**
  - "Active" switch `aria-checked` changes from its initial value after step 2.
  - A success toast appears: "Protection plan created successfully" (first save for a plan type that has no DB row yet) or "Protection plan updated successfully" (plan already existed).
  - The `CreateProtectionPlan` or `UpdateProtectionPlan` GraphQL mutation is sent and succeeds.
  - After the automatic refetch, the MINIMUM card reflects `$15/day` and the new Active/Inactive status.
- **Source:** src/app/containers/ProtectPlanPage/index.tsx (handleSave lines 279-304, mutation callbacks lines 242-277); e2e/host-protection-plans.spec.ts

### PLAN-04: Edit an existing protection plan's description and details
- **Priority:** P1  **Persona:** Host
- **Preconditions:** A protection plan already saved (PLAN-03). On Protection Plans page.
- **Steps:**
  1. Click the previously-saved plan card to expand it.
  2. [MUTATING] Change the "Description" textarea to `Updated description`.
  3. [MUTATING] Change the "Plan Details" textarea to `Updated policy details`.
  4. [MUTATING] Click "Save".
  5. Reload the page and re-expand the same plan card.
- **Expected:**
  - Success toast "Protection plan updated successfully" appears.
  - `UpdateProtectionPlan` mutation succeeds (plan has an id, so update branch is taken).
  - After reload, the Description and Plan Details fields retain the updated text.
- **Source:** src/app/containers/ProtectPlanPage/index.tsx (handleSave update branch lines 294-303)

### PLAN-05: Toggle a protection plan OFF (deactivate) and persist
- **Priority:** P1  **Persona:** Host
- **Preconditions:** An Active protection plan exists. On Protection Plans page.
- **Steps:**
  1. Click an Active plan card (status shows "Active", green check icon).
  2. [MUTATING] Click the "Active" toggle switch to set it to off (aria-checked false).
  3. [MUTATING] Click "Save".
  4. Reload the page.
- **Expected:**
  - Success toast "Protection plan updated successfully".
  - After reload, the plan card status shows "Inactive" with the red minus-circle icon.
- **Source:** src/app/containers/ProtectPlanPage/index.tsx; e2e/host-protection-plans.spec.ts (Step 7)

### PLAN-06: Collapse the plan form via chevron / click outside
- **Priority:** P2  **Persona:** Host
- **Preconditions:** A protection plan card is expanded.
- **Steps:**
  1. Click the chevron-up (collapse) button at the top of the expanded form.
  2. Re-expand a different plan card, then click empty space outside the radio group.
- **Expected:**
  - Clicking the chevron collapses the form (form fields no longer visible).
  - Clicking outside the radio group also collapses any open form (selected resets to null).
- **Source:** src/app/containers/ProtectPlanPage/index.tsx (onDeselect lines 342-344; handleClickOutside lines 172-186)

---

### TFEE-01: Trip Fees page loads with all default fee rows
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host (userId present in authStore, required for GetTripFees).
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/miscellaneous-fees
  2. Wait for the loader (green MoonLoader) to finish.
- **Expected:**
  - Heading "Trip Fees Set up" is visible.
  - The following fee rows are listed: "Trip Fee", "<21 Youngster Fee", "21-25 Youngster Fee", "free cancellation grace period", "free cancellation advance in hours", "Cancellation Maximum Penalty", "shorten booking max penalty", "Free Cancellation or Shorten Service Fee".
  - Each row shows charging text, "updated on: <date>", and Active/Inactive status with the matching icon (green check / red minus).
  - Each row also shows a default hint (e.g. "default: $0/day", "default: $50", "default: 1 hour").
  - `GetTripFees` GraphQL query (variable `userId`) succeeds.
- **Source:** src/app/containers/TripFeePage/index.tsx (defaultTripFees lines 127-214, render lines 257-277); src/App.tsx line 251

### TFEE-02: Service-fee row shows the Stripe-fee explanatory subtitle
- **Priority:** P2  **Persona:** Host
- **Preconditions:** On the Trip Fees page.
- **Steps:**
  1. Locate the "Free Cancellation or Shorten Service Fee" row.
- **Expected:**
  - A subtitle is shown: "E.g., 3% means a $3000 refund becomes $2910 (to recover Stripe processing fees)".
- **Source:** src/app/containers/TripFeePage/index.tsx (lines 441-445)

### TFEE-03: Expand a Charge-rule fee and verify chargeBy selector
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On the Trip Fees page.
- **Steps:**
  1. Click the "Trip Fee" row to expand it.
- **Expected:**
  - Inline form expands with: "Active" toggle switch, a charge-by dropdown (`select` with options Daily Rate / One Time / Percentage), an "amount:" labeled text input, a "Plan Details" textarea, and a "Save" button.
  - The chargeBy `select` is present because this fee's rule is CHARGE (not ADVANCE_RULES).
- **Source:** src/app/containers/TripFeePage/index.tsx (lines 480-516, getChargeLabel lines 367-378)

### TFEE-04: Expand an Advance-rule fee hides the chargeBy selector
- **Priority:** P2  **Persona:** Host
- **Preconditions:** On the Trip Fees page.
- **Steps:**
  1. Click the "free cancellation advance in hours" row to expand it.
- **Expected:**
  - The inline form shows the "Active" toggle, an amount input labeled "should be ___ hours before trip?", a "Plan Details" textarea, and "Save".
  - No charge-by dropdown is shown (rule is ADVANCE_RULES).
- **Source:** src/app/containers/TripFeePage/index.tsx (lines 480-491; getChargeLabel ADVANCE_RULES branch lines 368-370)

### TFEE-05: Configure and save a trip fee (amount + active + chargeBy)
- **Priority:** P0  **Persona:** Host
- **Preconditions:** On the Trip Fees page; "Trip Fee" row expanded (TFEE-03).
- **Steps:**
  1. [MUTATING] Toggle "Active" to on.
  2. [MUTATING] Select "Daily Rate" in the charge-by dropdown.
  3. [MUTATING] Set the "amount:" input to `20`.
  4. [MUTATING] Type `E2E trip fee` into the Plan Details textarea.
  5. [MUTATING] Click "Save".
  6. Reload the page.
- **Expected:**
  - Success toast "Trip fee created successfully" (first save, no id) or "Trip fee updated successfully" (existing fee).
  - `CreateTripFee` or `UpdateTripFee` mutation succeeds; input includes `userId`, `uniqName`, `name`, `amount`, `chargeBy`, `rule`, `isActive`.
  - After reload, the "Trip Fee" row shows the new charging text (e.g. `20/day` per renderChargingText) and "Active" status.
- **Source:** src/app/containers/TripFeePage/index.tsx (handleSave lines 332-365, mutations lines 307-331, renderChargingText lines 380-404)

### TFEE-06: Save without a userId surfaces an error
- **Priority:** P2  **Persona:** Host
- **Preconditions:** A session where `authStore.user.id` is not available (edge case; e.g. partially loaded auth state).
- **Steps:**
  1. Open any fee row and click "Save" before user data is loaded.
- **Expected:**
  - An error toast "User ID is required" appears and no mutation is sent.
- **Source:** src/app/containers/TripFeePage/index.tsx (handleSave guard lines 333-336)

---

### COUP-01: Coupons page loads and lists existing coupons
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/coupons
  2. Wait for the skeleton to disappear.
- **Expected:**
  - Heading "Coupons" and an "Add Coupon" button are visible.
  - If coupons exist, each is shown as a card with code (green mono font), name, an Active/Inactive status badge, category, algorithm (with % or $ value), valid-from/to dates, and Edit / Delete buttons.
  - If none exist, message "No coupons found. Click \"Add Coupon\" to create one." is shown.
  - `GetCoupons` query succeeds.
- **Source:** src/app/containers/CouponsPage/index.tsx; src/app/containers/CouponsPage/CouponList.tsx; src/App.tsx line 252

### COUP-02: Open the Create Coupon form
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On the Coupons page.
- **Steps:**
  1. Click "Add Coupon".
- **Expected:**
  - Form titled "Create Coupon" replaces the list.
  - Fields visible: Name * (text, max 40 chars), Start Date (datetime-local), End Date (datetime-local), Guest search (text), Product Type select, Category select, Minimum Days numeric, Active checkbox (checked by default), Apply By select (Day/Trip, default Trip), Price Algorithm select (Percentage/Fixed Amount, default Percentage), Percentage Off * numeric, Category select, Apply To Price Of select (Daily Price/Total Price, default Daily Price), Save button, Cancel button.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (lines 268-461)

### COUP-03: Create a percentage coupon (happy path)
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Create Coupon form open (COUP-02).
- **Steps:**
  1. [MUTATING] Type a unique Name, e.g. `E2E-PCT-<timestamp>`.
  2. Leave Price Algorithm as "Percentage".
  3. [MUTATING] Enter `10` in "Percentage Off".
  4. Leave Active checked.
  5. [MUTATING] Click "Save".
- **Expected:**
  - Success toast "Coupon created successfully".
  - `CreateCoupon` mutation succeeds; the form closes and the list refetches.
  - The new coupon appears in the list as Active, algorithm PERCENTAGE (10%), with a generated `code`.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (handleSubmit lines 214-266, createCoupon lines 143-151)

### COUP-04: Create a fixed-amount coupon
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Create Coupon form open.
- **Steps:**
  1. [MUTATING] Type a unique Name.
  2. Change "Price Algorithm" to "Fixed Amount".
  3. [MUTATING] Enter `25` in "Fixed Amount Value".
  4. [MUTATING] Click "Save".
- **Expected:**
  - Selecting "Fixed Amount" replaces the Percentage Off field with a "Fixed Amount Value *" numeric field.
  - Success toast "Coupon created successfully".
  - New coupon listed with algorithm FIX_AMOUNT ($25).
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (lines 402-434, handleSubmit fixAmount branch lines 250-252)

### COUP-05: Percentage validation rejects out-of-range value
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Create Coupon form open, Price Algorithm = Percentage.
- **Steps:**
  1. Type a Name.
  2. Attempt to enter `150` in "Percentage Off".
  3. Click "Save".
- **Expected:**
  - The NumericInput clamps/blocks values >= 100 (maxValue 99.99), so 150 cannot be committed.
  - If a value >= 100 reaches submit, an error toast "Percentage off must be between 0 and 99.99" appears and no mutation is sent.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (NumericInput maxValue 99.99 lines 405-417; handleSubmit validation lines 218-224)

### COUP-06: Create coupon requires a Name
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Create Coupon form open.
- **Steps:**
  1. Leave Name blank.
  2. Click "Save".
- **Expected:**
  - Form does not submit; the Name field (required, HTML5) shows a native validation prompt and no `CreateCoupon` mutation fires.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (Name input `required` line 279)

### COUP-07: Restrict coupon to a specific guest via search
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Create Coupon form open. At least one guest exists whose first name is searchable.
- **Steps:**
  1. [MUTATING] Type a Name.
  2. In the "Guest (optional - search by first name)" field, type at least 2 characters of a known guest first name.
  3. Wait for the dropdown of matches to appear; click a guest.
  4. [MUTATING] Set Percentage Off to `5` and click "Save".
- **Expected:**
  - Typing >= 2 chars triggers the `SearchGuests` query; a dropdown lists "FirstName LastName (email)".
  - Selecting a guest adds a chip showing the guest name + email with an "✕" remove button.
  - Re-selecting the same guest shows an info toast "This guest is already added".
  - On Save, `CreateCoupon` input includes `applyToGuestIds` and the coupon is created.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (handleGuestSearch lines 184-193, handleSelectGuest lines 195-208, handleSubmit lines 241-243)

### COUP-08: Set optional fields (dates, minDays, productType, category, applyBy, priceOf)
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Create Coupon form open.
- **Steps:**
  1. [MUTATING] Enter a Name.
  2. Set Start Date and End Date via the datetime-local pickers.
  3. Select a Product Type and a Category (apply-to filters).
  4. Set Minimum Days to `2`.
  5. Set "Apply By" to "Day" and "Apply To Price Of" to "Total Price".
  6. Enter Percentage Off `10` and click "Save".
- **Expected:**
  - Success toast "Coupon created successfully".
  - The list card shows "Valid from: <start> to <end>" and "Minimum days: 3" (UI renders `minDays + 1`).
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (handleSubmit optional fields lines 235-246); CouponList.tsx (minDays display line 116)

### COUP-09: Edit an existing coupon
- **Priority:** P1  **Persona:** Host
- **Preconditions:** At least one coupon exists. On Coupons page list.
- **Steps:**
  1. Click "Edit" on a coupon card.
  2. [MUTATING] Change the Name and the Percentage Off value.
  3. [MUTATING] Click "Save".
- **Expected:**
  - Form title shows "Edit Coupon" and fields are pre-populated from the coupon.
  - Success toast "Coupon updated successfully".
  - `UpdateCoupon` mutation succeeds (input includes the coupon id); list reflects the new name/value.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (edit branch lines 254-262, prefill lines 111-137); CouponsPage/index.tsx (handleEdit lines 62-65)

### COUP-10: Delete a coupon with confirm dialog
- **Priority:** P1  **Persona:** Host
- **Preconditions:** A disposable coupon exists (ideally one created in COUP-03).
- **Steps:**
  1. Click "Delete" on the target coupon card.
  2. A browser confirm dialog appears: "Are you sure you want to delete this coupon?". [MUTATING] Accept it.
- **Expected:**
  - On confirm, `DeleteCoupon` mutation runs; success toast "Coupon deleted successfully".
  - The coupon is removed from the list (GetCoupons refetched).
  - Dismissing the confirm dialog instead leaves the coupon untouched.
- **Source:** src/app/containers/CouponsPage/index.tsx (handleDelete lines 67-73, deleteCoupon lines 47-55)

### COUP-11: Cancel out of the coupon form
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Create or Edit Coupon form open.
- **Steps:**
  1. Click "Cancel".
- **Expected:**
  - Form closes and returns to the coupon list without sending any mutation.
- **Source:** src/app/containers/CouponsPage/CouponForm.tsx (Cancel onClick onClose line 457); CouponsPage/index.tsx (handleFormClose lines 75-79)

### COUP-12: Apply a valid coupon at guest checkout
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Logged in as a guest. A valid active coupon code exists (created in COUP-03) that applies to the selected car/dates/guest. Guest has progressed to the checkout page for a car (a booking intent exists). Do NOT complete the payment.
- **Steps:**
  1. On the checkout page (`data-testid="checkout-page-container"`), locate the coupon section (`data-testid="checkout-coupon-section"`).
  2. [MUTATING] Type the coupon code into `data-testid="checkout-coupon-input"`.
  3. [MUTATING] Click `data-testid="checkout-coupon-apply-button"`.
- **Expected:**
  - `ValidateCoupon` query runs with the entered code and booking context (carId, guestId, dates, totalDays).
  - On success: input + apply button are replaced by `data-testid="checkout-coupon-success"` reading "Coupon <CODE> applied successfully!".
  - The booking intent is recreated with the coupon and the price/receipt updates to reflect the discount.
  - Stop here; do not click the Book buttons (`checkout-book-button-desktop` / `checkout-book-button-mobile`) to avoid charging the card.
- **Source:** src/app/containers/GuestCarPage/CheckoutPage/index.tsx (coupon JSX lines 1302-1330, validateCoupon lines 671-728, handleApplyCoupon lines 732+); src/app/services/couponService/queries.ts (VALIDATE_COUPON)

### COUP-13: Invalid / expired coupon shows error at checkout
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** On the guest checkout page with a booking intent.
- **Steps:**
  1. [MUTATING] Type a bogus code (e.g. `NOTACOUPON`) into `data-testid="checkout-coupon-input"`.
  2. [MUTATING] Click `data-testid="checkout-coupon-apply-button"`.
- **Expected:**
  - `ValidateCoupon` returns `isValid: false` (or errors).
  - An error message appears in `data-testid="checkout-coupon-error"` showing the backend `errorMessage` or "Invalid coupon".
  - No success state is shown and no discount is applied.
- **Source:** src/app/containers/GuestCarPage/CheckoutPage/index.tsx (validateCoupon error branch lines 720-728, error JSX line 1329)

### COUP-14: Empty coupon code submission is blocked at checkout
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** On the guest checkout page with the coupon input empty.
- **Steps:**
  1. Click `data-testid="checkout-coupon-apply-button"` without entering a code.
- **Expected:**
  - Inline error "Please enter a coupon code" is shown; no `ValidateCoupon` query is sent.
- **Source:** src/app/containers/GuestCarPage/CheckoutPage/index.tsx (handleApplyCoupon empty guard lines 732-735)
