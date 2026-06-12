# Host Listing & Location Creation — Manual Test Cases

Area: Host car listing, location creation, operating hours, edit/publish.
Base URL: https://staging.autopilotee.com

General preconditions for all cases:
- A logged-in account with the Host role. Either USER1 (jidosoju+5@gmail.com / <password: see staging_credentials.local>) or USER2 (autopilotee+206503@gmail.com / <password: see staging_credentials.local>) — executor must discover which account is a host. To get the host UI, open the hamburger menu (button with the `bars` icon) and click "Switch to host" (confirm the menu then shows "Switch to guest").
- Host navigation exposes a "Locations" link routing to `/cars/host/add-location` and a "My Listings" page at `/cars/host`.

Key routes:
- My Listings: `/cars/host`
- Add item (location step): `/cars/host/add` (accepts `?type=CAR` etc.)
- Add VIN: `/cars/host/add-vin?locationId=<id>`
- Add details (non-car): `/cars/host/add-details`
- Add price: `/cars/host/add-price`
- Add photos: `/cars/host/add-photos`
- Locations list / add: `/cars/host/add-location`
- Operating hours: `/operating-hours/:locationId`
- Admin public airports: `/admin/delivery-locations`

---

### HLOC-01: Open the add-location modal from My Locations
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host; on the Locations page.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/add-location
  2. Wait for the heading `My Locations` to be visible.
  3. If the add modal is not already open (it auto-opens when the host has zero locations), click the button `data-testid="add-location-button"` ("Add Location").
- **Expected:**
  - The page heading "My Locations" renders.
  - After clicking, the modal shows with the input `data-testid="location-name-input"` visible, plus address, city, state, zip, tax rate, phone, and description fields, and a `data-testid="location-save-button"` ("Save Location").
- **Source:** src/app/containers/HostCarPage/AddLocationPage/index.tsx; src/app/containers/HostCarPage/AddLocationPage/components/AddOrEditLocationModal.tsx; e2e/host-listing-creation.spec.ts

### HLOC-02: Create a new location (happy path) [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host; add-location modal open (HLOC-01).
- **Steps:**
  1. In `data-testid="location-name-input"` type a unique name, e.g. `QA Test Location <timestamp>`.
  2. Click `data-testid="location-address-input"` and type a street address (e.g. `100 Market St`); wait for the Google Places dropdown and select the first suggestion.
  3. Confirm `data-testid="location-city-input"` and `data-testid="location-state-input"` auto-populate (read-only).
  4. If `data-testid="location-zip-input"` is empty, type a valid ZIP (e.g. `94105`).
  5. In `data-testid="location-tax-rate-input"` type `8.5`.
  6. In the phone field (placeholder "Enter phone number", `data-testid="location-phone-input"`) type a valid US number, e.g. `+16505550123`.
  7. In `data-testid="location-description-input"` type `Created by QA`.
  8. [MUTATING] Click `data-testid="location-save-button"`.
- **Expected:**
  - Toast "Location created successfully" appears.
  - Modal closes; a new row `data-testid="location-row-<id>"` (desktop) or card `data-testid="location-card-<id>"` (mobile) appears containing the entered name and the selected formatted address.
- **Source:** AddOrEditLocationModal.tsx (createLocation mutation, success toast 'Location created successfully'); e2e/host-create-location.mocked.spec.ts

### HLOC-03: Location create — required-field validation
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Logged in as host; add-location modal open with all fields empty.
- **Steps:**
  1. Click `data-testid="location-save-button"` with no fields filled.
  2. Type only a name in `data-testid="location-name-input"`, then click save again.
  3. Add an address via autocomplete but leave the phone empty, then click save.
- **Expected:**
  - Step 1: error toast "Location name is required".
  - Step 2: error toast "Address is required" (then City/State/Zip in sequence as each gets satisfied).
  - Step 3: error toast "Phone number is required" and inline phone error text "Phone number is required".
  - No "Location created successfully" toast; modal stays open.
- **Source:** AddOrEditLocationModal.tsx onSubmitLocation validation chain

### HLOC-04: Location create — invalid phone format rejected
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Add-location modal open; name, address (autocomplete), city/state/zip populated.
- **Steps:**
  1. In the phone field type an invalid number, e.g. `123`.
  2. Blur the field (Tab away).
  3. Click `data-testid="location-save-button"`.
- **Expected:**
  - On blur: inline error "Invalid phone number format".
  - On save: error toast "Invalid phone number format" and inline "Please enter a valid phone number"; no success toast.
- **Source:** AddOrEditLocationModal.tsx (isValidPhoneNumber check)

### HLOC-05: Edit an existing location [MUTATING]
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Logged in as host with at least one existing location.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/add-location
  2. In the desktop table row (or mobile card) for a location, click its "Edit" button.
  3. Wait for the modal to load and prefill the existing values (name, address, city, state, zip, tax rate, phone, description).
  4. Change `data-testid="location-description-input"` to `Updated by QA <timestamp>`.
  5. [MUTATING] Click `data-testid="location-save-button"`.
- **Expected:**
  - Modal prefills with the location's current values.
  - Toast "Location updated successfully" appears; modal closes; the row reflects the change.
- **Source:** AddOrEditLocationModal.tsx (useGetLocationQuery prefill, updateLocation success toast 'Location updated successfully'); AddLocationPage/index.tsx (Edit button)

### HLOC-06: Delete location confirmation dialog (cancel, do not delete)
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Logged in as host with at least one location.
- **Steps:**
  1. Navigate to /cars/host/add-location
  2. Click "Delete" on a location row/card.
  3. In the dialog "Deleting a Location?" click "Cancel".
- **Expected:**
  - Dialog appears with title "Deleting a Location?" and the warning "delete location is NOT REVERSIBLE...".
  - After Cancel, the dialog closes and the location remains in the list (no delete performed).
  - NOTE: Do NOT click "Confirm" — that is a destructive [MUTATING] action; only exercise Cancel for routine runs.
- **Source:** AddLocationPage/index.tsx (BaseDialog, REMOVE_LOCATION mutation)

### HLOC-07: Set up operating hours for a location [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host with at least one location and its numeric id.
- **Steps:**
  1. From /cars/host/add-location click the "Hours" link for a location (`data-testid="location-table-hours-<id>"` desktop or `data-testid="location-card-hours-<id>"` mobile), OR navigate directly to https://staging.autopilotee.com/operating-hours/<locationId>
  2. Wait for the heading "Pickup & Return Hours" and the "Current Operating Hours" panel.
  3. Click `data-testid="operating-hours-edit-button"` (label "Setup operating hours" if none exist, else "Edit operating hours").
  4. In the modal, expand a day (click its row), set Start time and End time selects to specific values.
  5. [MUTATING] Click `data-testid="operating-hours-save-button"` ("Save").
- **Expected:**
  - Operating hours page renders with description "Please note that the hours you set apply to all of your vehicles."
  - After save: toast "Operating hours saved successfully" (or "updated successfully" if editing) and a green success banner "Operating hours saved successfully!".
- **Source:** OperatingHoursPage/index.tsx; OperatingHoursFormModal.tsx; AddLocationPage/index.tsx (hours testids)

### HLOC-08: Operating hours — "Always available" toggle [MUTATING]
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On /operating-hours/<locationId>, edit modal open.
- **Steps:**
  1. Toggle `data-testid="operating-hours-always-available-toggle"` ON.
  2. Verify the per-day list is hidden while Always available is ON.
  3. [MUTATING] Click `data-testid="operating-hours-save-button"`.
- **Expected:**
  - When the toggle is ON the per-day cards (Sunday..Saturday) are not shown.
  - After save, success toast and banner appear.
- **Source:** OperatingHoursFormModal.tsx (alwaysAvailable hides DaysList)

### HLOC-09: Operating hours — apply one day's hours to additional days
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Edit operating hours modal open, "Always available" OFF.
- **Steps:**
  1. Click a day row (e.g. Monday) to expand it.
  2. Set its Start/End times.
  3. In "Apply these hours to additional days", select two other day buttons (e.g. Tuesday, Wednesday).
  4. Click the "Apply to 2 days" button.
- **Expected:**
  - Toast "Hours applied to selected days".
  - The selected days' time summaries now match Monday's. (This is a client-side state change; the Save button is what persists — mark Save [MUTATING] if persisting.)
- **Source:** OperatingHoursPage/index.tsx (applyCurrentDayToSelected); OperatingHoursFormModal.tsx

### HLOC-10: Operating hours — unsaved-changes confirmation on close
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Edit operating hours modal open.
- **Steps:**
  1. Change a day's time (creating an unsaved change).
  2. Click "Cancel" (or the modal close).
- **Expected:**
  - A confirmation dialog "Unsaved Changes" appears with "Discard" and "Save Changes" buttons.
  - Clicking "Discard" closes the modal without saving; "Save Changes" persists and closes.
- **Source:** OperatingHoursFormModal.tsx (handleClose / showConfirmation)

### HCAR-01: Start add-car flow — select item type and location
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Logged in as host with at least one location that has operating hours configured.
- **Steps:**
  1. From /cars/host click `data-testid="add-item-button"` ("Add Item"), then in the type menu click `data-testid="add-item-type-car"` (Car). (Alternatively navigate to https://staging.autopilotee.com/cars/host/add?type=CAR.)
  2. On the Add Car page, confirm the "Item Type" select shows "Car" and the "Location" select `data-testid="add-car-location-select"` is visible.
  3. Select an existing location from `data-testid="add-car-location-select"`.
  4. Click `data-testid="add-car-location-next"` ("Next").
- **Expected:**
  - Title reads "List your car".
  - Location select lists existing locations plus "+ Create new location".
  - [MUTATING] Clicking Next creates a draft car (addNewCar) and navigates to `/cars/host/add-vin?locationId=<id>`.
  - If the selected location has NO operating hours, an alert "Missing Operating Hours" appears prompting to add them instead of proceeding.
- **Source:** HostCarPage/index.tsx (Add Item menu); AddCarPage/index.tsx; e2e/host-listing-creation.spec.ts

### HCAR-02: Add-car requires a location when none exist
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Logged in as a host account that has zero locations.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host/add?type=CAR
- **Expected:**
  - Page shows "No locations found" and an "Add Location" button that routes to `/cars/host/add-location`.
- **Source:** AddCarPage/index.tsx (myLocations.length === 0 branch)

### HCAR-03: Enter VIN and decode car info [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** On `/cars/host/add-vin?locationId=<id>` with a valid draft car (reached via HCAR-01). The locationId in the URL must belong to the current user, otherwise the page redirects to `/cars/host/add`.
- **Steps:**
  1. In `data-testid="car-vin-input"` type a valid 17-char VIN (e.g. `1HGCM82633A004352`).
  2. Click `data-testid="car-vin-next"` ("Next").
  3. Wait for the car-info modal to open with decoded Make/Model/Year/Seats prefilled.
- **Expected:**
  - VIN input enforces 17 chars max and uppercases input.
  - On Next a VIN decode call runs (NHTSA); the modal opens with `data-testid="car-make-input"`, `car-model-input`, `car-year-input`, `car-seats-input` populated where available.
- **Source:** AddVinPage.tsx (VIN_REGEX, getCarInfoFromVin, modal)

### HCAR-04: VIN format validation
- **Priority:** P1  **Persona:** Host
- **Preconditions:** On `/cars/host/add-vin?locationId=<id>`.
- **Steps:**
  1. In `data-testid="car-vin-input"` type an invalid VIN, e.g. `ABC123` (too short / contains I,O,Q if used).
  2. Click `data-testid="car-vin-next"`.
- **Expected:**
  - Inline error "Invalid VIN format (must be 17 characters, excluding I, O, Q)".
  - No modal opens.
- **Source:** AddVinPage.tsx (VIN_REGEX = /^[A-HJ-NPR-Z0-9]{17}$/)

### HCAR-05: Complete car details form and submit [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** Car-info modal open (HCAR-03).
- **Steps:**
  1. Fill required fields: `car-make-input`, `car-model-input`, `car-year-input`, `car-seats-input`, `car-gear-type-select`, `car-fuel-type-select`, `car-license-plate-input`, `car-license-state-input`, `car-gas-grade-select`, `car-city-mpg-input`, `car-highway-mpg-input`, `car-doors-input`, `car-category-select`, `car-description-input`.
  2. Confirm `data-testid="car-info-next"` becomes enabled only when all required fields are set (Original Price is optional).
  3. [MUTATING] Click `data-testid="car-info-next"` ("Next").
- **Expected:**
  - Next button is disabled until all required fields are filled.
  - On Next the car is created/updated (addNewCar/updateCar) and the app navigates to `/cars/host/add-price`.
- **Source:** AddVinPage.tsx CarInfo (data-testids, isActivated validation, navigate add-price)

### HCAR-06: Set car pricing [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** On `/cars/host/add-price` with an active draft car (reached via HCAR-05).
- **Steps:**
  1. In `data-testid="car-weekday-price-input"` type `50`.
  2. In `data-testid="car-weekend-price-input"` type `60`.
  3. In `data-testid="car-holiday-price-input"` type `75`.
  4. (Optional) Tick "Enable hourly rental" and enter hourly prices.
  5. [MUTATING] Click `data-testid="car-price-next"` ("Next").
- **Expected:**
  - Title "Set Your Car Pricing".
  - Next is disabled until weekday, weekend, and holiday prices are all filled with numbers.
  - On Next the car prices (and hourly settings) are saved and the app navigates to `/cars/host/add-photos`.
- **Source:** AddPricePage.tsx (price testids, isFormValid, UPDATE_CAR + UPDATE_CAR_HOURLY_PRICING, navigate add-photos)

### HCAR-07: Upload car photos and finish listing [MUTATING]
- **Priority:** P0  **Persona:** Host
- **Preconditions:** On `/cars/host/add-photos` with an active draft car (reached via HCAR-06).
- **Steps:**
  1. Click the dropzone `data-testid="car-photos-dropzone"` and choose 1–3 image files (or set files on `data-testid="car-photos-input"`).
  2. Confirm thumbnails render; the first photo shows a "Main Photo" badge.
  3. [MUTATING] Click `data-testid="car-photos-next"` ("Next").
- **Expected:**
  - Up to 10 photos accepted; non-image or >10MB files show an error message; >10 total shows "You can only choose N more photos".
  - On Next photos upload to S3 (presigned URLs), the car is marked created (isCreated=true), toast "Car created successfully!" appears, and the app navigates to `/cars/host` (My Listings).
- **Source:** AddPhotosPage.tsx (dropzone/input testids, upload flow, 'Car created successfully!' toast, navigate /cars/host)

### HCAR-08: Photo upload — Next disabled with zero photos
- **Priority:** P2  **Persona:** Host
- **Preconditions:** On `/cars/host/add-photos` with no photos selected.
- **Steps:**
  1. Observe `data-testid="car-photos-next"` before adding any photo.
- **Expected:**
  - The Next button is disabled (greyed) until at least one photo is added.
- **Source:** AddPhotosPage.tsx (isActivated = photos.length > 0)

### HCAR-09: New listing appears on My Listings
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Completed HCAR-07 for a host account.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/cars/host
  2. Wait for the "My Listings" page.
- **Expected:**
  - Heading "My Listings" renders.
  - The just-created car appears as a card in the grid (make/model/year visible). If no cars, "No cars found." is shown instead.
- **Source:** HostCarPage/index.tsx (GetMyCars, HostCarCard)

### HCAR-10: Open a listing detail / edit view
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Host account with at least one listing.
- **Steps:**
  1. From /cars/host click a listing card.
- **Expected:**
  - App navigates to `/cars/host/view/<carId>` and the car detail/edit view loads.
- **Source:** HostCarPage/index.tsx (handleCarClick -> /cars/host/view/:id)

### HADM-01: Create a public airport location (admin) [MUTATING]
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Logged in as an account with admin access to the public locations page.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/admin/delivery-locations
  2. Wait for heading "Public Airport Locations" and the "Create public airport" form.
  3. Fill Name, Address (via autocomplete which fills City/State/Zip/Lat/Lng), optionally Tax Rate / Phone / Description.
  4. [MUTATING] Click "Create airport".
- **Expected:**
  - Toast "Public airport created"; the form clears; the new airport appears in the "Existing public airports" table.
- **Source:** AdminLocationsPage/index.tsx (CREATE_PUBLIC_LOCATION, 'Public airport created' toast)

### HADM-02: Edit a public airport location (admin) [MUTATING]
- **Priority:** P2  **Persona:** Host
- **Preconditions:** On /admin/delivery-locations with at least one existing public airport.
- **Steps:**
  1. Click "Edit" on an airport row.
  2. In the "Edit Location" modal change a field (e.g. Description).
  3. [MUTATING] Click "Save Changes".
- **Expected:**
  - Modal prefills the airport's values; on save, toast "Location updated successfully"; modal closes; table refreshes.
- **Source:** AdminLocationsPage/index.tsx (UPDATE_LOCATION, edit modal)
