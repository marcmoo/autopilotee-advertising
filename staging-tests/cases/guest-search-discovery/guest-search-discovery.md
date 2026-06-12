# Guest Car Search & Discovery — Manual Test Cases

Area: Guest car search and discovery on https://staging.autopilotee.com
Covers: landing/home load, location & address date search, filters, search results list, map markers, empty results, car card content/navigation.

Key facts from source:
- Home route: `/` renders `HomePage` (TopSection banner + CategoryFilter + TopCars).
- Search results route: `/cars/guest/search` with query string params.
- Car detail route: `/cars/guest/view/<carId>?<params>` (navigated from a car card click).
- Search form is the `BookCard` component (`data-testid="booking-card-container"`) with two tabs: "Pickup Location" (`data-testid="booking-location-tab"`) and "Enter Address" (`data-testid="booking-address-tab"`).
- BookCard selectors: `booking-pickup-location-input`, `booking-different-return-checkbox`, `booking-pickup-date-input`, `booking-return-date-input`, `booking-pickup-time-select`, `booking-return-time-select`, `booking-age-select`, `booking-search-button`.
- Search results card selectors (per car id): `search-car-card-<id>`, `search-car-image-<id>`, `search-car-title-<id>`, `search-car-location-<id>`, `search-car-price-<id>`, `search-car-favorite-<id>`.
- Results header text: `Found <N> cars available`. Car card title shows the car MAKE (e.g. "FORD"). Price shows `$<total>` followed by "total" and "before taxes and fees".
- A working example search URL (from e2e spec, dates rewritten as future): `/cars/guest/search?driverAge=25&locationId=1&returnDate=...&returnLocationId=1&returnTime=12%3A00%20PM&searchMode=location&startDate=...&startTime=12%3A00%20PM&timezone=America%2FLos_Angeles&type=CAR`.

Notes on auth: Search and discovery are mostly available to anonymous users. The favorite (heart) button on a card only renders when the user is authenticated (`onFavoriteClick` is passed only when `authStore.isAuthenticated`). Use a logged-in account (USER1 or USER2) for favorite cases; the executor will discover which credential logs in successfully.

---

### GSRCH-01: Home page loads with hero banner and search card
- **Priority:** P0  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/
  2. Wait for the page to finish loading.
  3. Observe the hero/top section text and the booking search card.
- **Expected:**
  - Page title is "AutoPilotee Cars".
  - Hero slogan text "Rent The Best Qulity Car's With Us" is visible (note: typo "Qulity" is in source).
  - The booking search card is present: element `data-testid="booking-card-container"` is visible.
  - Two search tabs are visible: "Pickup Location" (`data-testid="booking-location-tab"`) and "Enter Address" (`data-testid="booking-address-tab"`).
  - The search button `data-testid="booking-search-button"` is visible with text "Search".
- **Source:** src/app/containers/HomePage/index.tsx, src/app/containers/HomePage/topSection.tsx, src/app/components/bookCard/index.tsx

### GSRCH-02: Home page category filter and top cars render
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/
  2. Scroll to the category filter row below the hero section.
  3. Observe the category chips.
  4. Click the "SUV" category chip.
- **Expected:**
  - Category chips render in order: "All", "SUV", "Convertible", "Sedan", "Truck".
  - "All" is selected (highlighted) by default.
  - After clicking "SUV", the "SUV" chip becomes the active/highlighted chip (orange `#FF5A33` background, white text) and the Top Cars section updates to reflect the selected category.
- **Source:** src/app/containers/HomePage/index.tsx, src/app/components/categoryFilter/index.tsx

### GSRCH-03: Location-based search from home navigates to results
- **Priority:** P0  **Persona:** Anonymous
- **Preconditions:** Home page loaded; at least one pickup location seeded on staging.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/
  2. Ensure the "Pickup Location" tab (`data-testid="booking-location-tab"`) is active.
  3. Click the pickup location input `data-testid="booking-pickup-location-input"`.
  4. Select the first location from the dropdown list that appears.
  5. Click the pickup date input `data-testid="booking-pickup-date-input"`, pick a future start date in the date range modal, pick a future return date (after start), and save/confirm the modal.
  6. Confirm a pickup time is set via `data-testid="booking-pickup-time-select"` and a return time via `data-testid="booking-return-time-select"` (auto-selected to first available is acceptable).
  7. Confirm an age is selected in `data-testid="booking-age-select"` (defaults to "25+").
  8. Click the search button `data-testid="booking-search-button"`.
- **Expected:**
  - The URL changes to `/cars/guest/search?...` containing `searchMode=location`, a `locationId`, `startDate`, `returnDate`, `startTime`, `returnTime`, `driverAge`, and `type` params.
  - The search results page renders a results header reading "Found <N> cars available".
- **Source:** src/app/components/bookCard/index.tsx (handleSearch), src/app/containers/GuestCarPage/SearchResultPage/index.tsx

### GSRCH-04: Direct navigation to a seeded location search URL shows results, card content, and map
- **Priority:** P0  **Persona:** Anonymous
- **Preconditions:** Location id 1 seeded with at least one available car (San Francisco per e2e seed). Use future dates.
- **Steps:**
  1. Navigate to a results URL of the form: https://staging.autopilotee.com/cars/guest/search?driverAge=25&locationId=1&returnDate=<FUTURE_ISO>&returnLocationId=1&returnTime=12%3A00%20PM&searchMode=location&startDate=<FUTURE_ISO>&startTime=12%3A00%20PM&timezone=America%2FLos_Angeles&type=CAR (start/return dates a few days apart, both in the future).
  2. Wait for results to load (a skeleton may show first).
  3. Observe the results header, the first car card, and the map panel.
- **Expected:**
  - URL matches `/cars/guest/search`.
  - Results header "Found <N> cars available" is visible with N >= 1.
  - At least one car card is rendered: an element `data-testid="search-car-card-<id>"` exists.
  - The card shows a make title (`search-car-title-<id>`), a location/city (`search-car-location-<id>`), and a price (`search-car-price-<id>`) formatted as `$<amount>` followed by "total" and "before taxes and fees".
  - The map panel renders on the right side (Google Maps loads).
- **Source:** e2e/guest-booking-search.spec.ts, src/app/containers/GuestCarPage/SearchResultPage/index.tsx, src/app/components/car/searchResult/carCard/index.tsx

### GSRCH-05: Car card click navigates to the car detail (view) page
- **Priority:** P0  **Persona:** Anonymous
- **Preconditions:** On a search results page with at least one car card (e.g. via GSRCH-04).
- **Steps:**
  1. From a results page with cars, locate the first car card `data-testid="search-car-card-<id>"`.
  2. Click the car card (clicking the title `search-car-title-<id>` also works).
- **Expected:**
  - The URL changes to `/cars/guest/view/<id>?<params>` where `<id>` matches the clicked card's id and the query string carries forward `locationId`/`searchMode`, `startDate`, `startTime`, `returnDate`, `returnTime`, `driverAge`.
  - The car detail page loads.
- **Source:** src/app/components/car/searchResult/carCard/index.tsx (handleClick navigate), src/app/containers/GuestCarPage/SearchResultPage/index.tsx (urlParams)

### GSRCH-06: Car card displays price, "before taxes and fees" label, and optional New badge
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On a search results page with at least one car card.
- **Steps:**
  1. Inspect the first car card on the results list.
  2. Read the price block and image area.
- **Expected:**
  - Price element `search-car-price-<id>` shows `$<total>` (the total with delivery if present, else total price) styled green, followed by the word "total".
  - The label "before taxes and fees" appears under the price.
  - If the car is new (`isNew`), a "New" badge appears in the top-left of the image; otherwise no badge.
  - The car thumbnail image `search-car-image-<id>` renders with `alt` equal to the car make.
- **Source:** src/app/components/car/searchResult/carCard/index.tsx

### GSRCH-07: Open Filters modal and apply a price filter
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On a results page that returns multiple cars at different price points (use GSRCH-04 against seeded data).
- **Steps:**
  1. On the results list, click the "Filters" button.
  2. In the "All filters" modal, drag the "Daily price" max slider down (e.g. to a low value) to restrict the price range.
  3. Click "Apply filters".
- **Expected:**
  - The modal titled "All filters" opens, showing sections: Sort by, Daily price, Vehicle type, Number of seats, Fuel type, Features, All years, Transmission, and footer buttons "Cancel", "Reset", "Apply filters".
  - After applying, the modal closes.
  - The "Filters" button shows a numeric badge equal to the count of active filters (>= 1).
  - The results header "Found <N> cars available" updates to reflect the filtered count, and a "Reset" button appears next to "Filters".
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (Filters modal, activeFilterCount, apply/reset), src/app/constants/car-filters.ts

### GSRCH-08: Apply Sort by "Price: Low to High"
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On a results page returning 2+ cars at differing prices.
- **Steps:**
  1. Click "Filters".
  2. In the "Sort by" dropdown, select "Price: Low to High".
  3. Click "Apply filters".
- **Expected:**
  - The modal closes and the "Filters" badge shows an increment for the non-default sort.
  - The car cards are ordered by ascending total price (first card's `search-car-price` <= subsequent cards').
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (sortBy, SORT_OPTIONS), src/app/constants/car-filters.ts

### GSRCH-09: Vehicle type filter narrows results
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On a results page with cars of mixed vehicle types.
- **Steps:**
  1. Click "Filters".
  2. Under "Vehicle type", click a type chip (e.g. "SUVs").
  3. Click "Apply filters".
- **Expected:**
  - Selected vehicle type button becomes highlighted (dark background) while open.
  - After apply, the "Filters" badge increments and the results list/count updates to only matching cars (or shows the no-cars-with-filters state if none match).
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (VEHICLE_TYPE_OPTIONS, toggleMultiSelect)

### GSRCH-10: Reset filters restores full results
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** A filter is currently applied (e.g. after GSRCH-07/08/09), badge count >= 1.
- **Steps:**
  1. On the results list, click the "Reset" button next to "Filters" (or open Filters and click "Reset" then "Apply filters").
- **Expected:**
  - The "Filters" badge disappears (active filter count returns to 0).
  - The results header count returns to the unfiltered total and the "Reset" button next to "Filters" is no longer shown.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (resetFilters)

### GSRCH-11: No cars available (empty results, no filters)
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** A valid location with no available cars for the chosen dates (or a far-future obscure date range).
- **Steps:**
  1. Navigate to a `/cars/guest/search?...` URL for a location/date combo that yields zero cars and no active filters.
  2. Wait for results to load.
- **Expected:**
  - The empty-state card shows title "No Cars Available" and message "We couldn't find any cars available for your selected dates and location. Please try different dates or another location."
  - Details box lists the Location ID, Pickup, and Return datetimes.
  - The search bar/BookCard remains available at the top to modify the search.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (cars.length === 0 branch)

### GSRCH-12: No cars match active filters (empty filtered results)
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On a results page that returns cars; then apply a restrictive filter that excludes all of them (e.g. set Daily price max to its minimum or pick an unavailable vehicle type).
- **Steps:**
  1. Click "Filters".
  2. Apply a filter combination known to exclude all cars (e.g. very low max price).
  3. Click "Apply filters".
- **Expected:**
  - The empty-state card shows title "No cars found" with message "Try changing your filters or resetting them to see all cars."
  - A "Reset filters" button is shown; clicking it restores the full result list.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (activeFilterCount > 0 empty branch)

### GSRCH-13: Location closed during selected hours error state
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** A location with restricted operating hours and a pickup/return time outside those hours (executor may need to find such a location, or this may be unreachable if all locations are 24/7).
- **Steps:**
  1. Navigate to a `/cars/guest/search?...` URL whose pickup/return time falls outside the location's operating hours.
  2. Wait for the response.
- **Expected:**
  - An error card shows title "Location Closed During Selected Hours" with the explanatory message about adjusting times.
  - A details box shows Location, Pickup, and Return datetimes.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (ErrorCode.LocationOutOfOperatingHours branch)

### GSRCH-14: Address-based search returns nearby cars
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Home page loaded; Google Places autocomplete available; cars seeded within ~20 miles of a known address.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/
  2. Click the "Enter Address" tab `data-testid="booking-address-tab"`.
  3. Type an address into the address input and select a Google Places suggestion.
  4. Set a future pickup date and return date (via `booking-pickup-date-input`), and ensure times and age are set.
  5. Click the search button `data-testid="booking-search-button"`.
- **Expected:**
  - URL changes to `/cars/guest/search?...` containing `searchMode=address` with `latitude`, `longitude`, `address`, and `timezone` params.
  - Results header "Found <N> cars available" renders; cars within radius are listed.
  - On address search, each card may show "• <X.X> miles away" appended to the location text when `distanceMiles > 0`.
- **Source:** src/app/components/bookCard/index.tsx (handleSearch address branch), src/app/containers/GuestCarPage/SearchResultPage/index.tsx (isAddressSearch, radiusMiles 20), src/app/components/car/searchResult/carCard/index.tsx (distanceMiles)

### GSRCH-15: Search validation — missing pickup location blocks search
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Home page loaded; "Pickup Location" tab active; no location selected.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/
  2. Without selecting a pickup location, click the search button `data-testid="booking-search-button"`.
- **Expected:**
  - A toast error "Pickup location is required" appears (top-center).
  - The URL does NOT change to `/cars/guest/search`.
- **Source:** src/app/components/bookCard/index.tsx (handleSearch validation toasts)

### GSRCH-16: Map markers reflect car list and selecting a marker highlights its card
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On a results page (GSRCH-04) with cars that have valid coordinates; desktop viewport (>= 768px so both panels show).
- **Steps:**
  1. Load a results page with one or more cars.
  2. Wait for the Google Map to render in the right panel.
  3. Click a car marker on the map.
- **Expected:**
  - The map renders markers for cars with valid (non-zero) latitude/longitude.
  - Clicking a marker selects that car: the corresponding card in the left list scrolls into view and is highlighted (light gray background `#f3f4f6`).
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (mapCars filter, handleCarSelect, selectedCarId), src/app/components/map/index.tsx

### GSRCH-17: Mobile list/map toggle
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On a results page with cars; mobile viewport (< 768px).
- **Steps:**
  1. Resize/emulate a mobile viewport and load a results page.
  2. Observe the floating toggle button at the bottom-right.
  3. Tap the toggle button.
- **Expected:**
  - On mobile, only one panel shows at a time. The floating toggle button shows "Map" when in list view and "List" when in map view.
  - Tapping it switches between the car list and the map panel.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (ViewToggleButton, viewMode, LeftPanel/RightPanel media queries)

### GSRCH-18: Favorite (heart) toggle on a car card while logged in
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Logged in as a guest (USER1 or USER2). On a results page with at least one car card. The heart button only renders when authenticated.
- **Steps:**
  1. Log in, then navigate to a results page with cars.
  2. Locate the favorite button `data-testid="search-car-favorite-<id>"` on the first card.
  3. [MUTATING] Click the favorite (heart) button.
  4. [MUTATING] Click it again to unfavorite (cleanup).
- **Expected:**
  - The heart toggles to filled/pink (`#E00070`) when favorited and outline/gray when not.
  - Clicking the heart does NOT navigate to the car detail page (click propagation is stopped).
  - Toggling back restores the original outline state, leaving the account's favorites unchanged.
- **Source:** src/app/components/car/searchResult/carCard/index.tsx (FavoriteButton, handleFavoriteClick stopPropagation), src/app/containers/GuestCarPage/SearchResultPage/index.tsx (onFavoriteClick only when isAuthenticated, useFavorites)

### GSRCH-19: Edit search from results page via the search bar
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On a results page (GSRCH-04); desktop viewport (the BookCard renders inline at the top).
- **Steps:**
  1. On the results page, locate the search bar/BookCard at the top.
  2. Change the pickup location (or dates) to a different valid value.
  3. Trigger the search update (the BookCard syncs the URL on change when `syncUrlOnChange` is set; on mobile a "Minimize" control may be present).
- **Expected:**
  - The URL query string updates to reflect the new location/dates (replace navigation, same `/cars/guest/search` path).
  - The results header count and car list refresh for the new criteria.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (BookCard with syncUrlOnChange, CompactSearchView/Minimize), src/app/components/bookCard/index.tsx (syncUrlOnChange effect)

### GSRCH-20: Airport search shows At airport / Off airport tabs
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** A search whose location/address resolves to an airport (backend returns `isAirportSearch=true`). Executor may need to use an airport address; may be unreachable if no airport data is seeded.
- **Steps:**
  1. Perform an address search at an airport, or load a results URL that resolves to an airport.
  2. Observe the tab row above the car list.
- **Expected:**
  - Two tabs appear: "At airport (<count>)" and "Off airport (<count>)".
  - Default active tab is "At airport"; switching to "Off airport" shows nearby off-airport cars.
  - If no cars deliver to the airport, an "No cars available at this airport" message with a "View Off Airport Cars (<count>)" button is shown.
- **Source:** src/app/containers/GuestCarPage/SearchResultPage/index.tsx (isAirportSearch, activeTab, TabRow, NoCarsMessage)
