# Secondary Pages — Help, About, Policies, Account Deletion, Dealer Microsite

App: https://staging.autopilotee.com (AutoPilotee Cars React frontend)

Notes:
- The Help/About/Policy pages are mostly static/content and render for anonymous and authenticated users alike.
- Policy pages fetch a markdown file from /public/policies and render it with ReactMarkdown. Content is post-processed: "Turo" is replaced with "Autopilotee Cars" and "Turo Inc." with "Jidosoju Holding Inc."; phone numbers get a red bold style.
- "Account Deletion" is a CONTENT page (renders /policies/account-deletion.md), NOT an interactive delete-my-account flow. The actual account-deletion action lives in the profile/settings area (out of scope for this file); treat ACCT-* here as informational page checks.
- The Dealership microsite (DealershipLanding, DealershipContact, CarFax viewer) is domain-gated to autopiloteecars.com / jidosojuholding.com and will NOT render on staging.autopilotee.com — see DEAL-00. The /dealership route (DealerPage) on the main app requires a Host account that owns a dealer.

---

### HELP-01: Help & Support page renders with FAQ, contact, articles
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** None (works logged out or in).
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/help
- **Expected:**
  - Header "Help & Support" with subtitle "Find answers to common questions and get in touch with our support team".
  - "Frequently Asked Questions" section with 7 collapsible questions, including "How do I book a car?", "How do I become a host?", "What is your cancellation policy?", "How do I contact customer support?".
  - "Contact Information" section showing phone +1(866)217-2849 (tel link), email support@autopilotee.com (mailto link), and address "2150 N first Street STE 426, San Jose, CA 95131".
  - A "Call Support Now" button linking to tel:+18662172849.
  - "Help Articles" section with 5 articles (Getting Started Guide, Hosting Your Vehicle, Cancellation Policy, Terms of Service, Privacy Policy), each with a "Read More" link.
- **Source:** src/app/containers/HelpPage/index.tsx

### HELP-02: FAQ accordion expand/collapse
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On /help.
- **Steps:**
  1. Click the FAQ question "How do I book a car?"
  2. Observe the answer expands (chevron flips up)
  3. Click the same question again
- **Expected:**
  - Clicking opens exactly one answer at a time; the chevron toggles between down and up.
  - The answer text reveals (e.g. the booking FAQ explains browsing, selecting dates, filtering).
  - Clicking the open question collapses it; opening a different question collapses the previously open one (single-open accordion).
- **Source:** src/app/containers/HelpPage/index.tsx (toggleFAQ, openFAQ state)

### HELP-03: Help article links navigate correctly
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On /help.
- **Steps:**
  1. Click "Read More" under "Cancellation Policy"
- **Expected:** Navigates to /policies/cancellation and the Cancellation Policy page renders. (Other articles link to /, /cars/host, /policies/terms, /policies/privacy.)
- **Source:** src/app/containers/HelpPage/index.tsx (helpArticles links)

### ABOUT-01: About Us page renders
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/about (executor: confirm the exact route; AboutUsPage reuses the HomePage AboutUs section)
- **Expected:** The About Us section content from the homepage renders inside a centered page container without errors.
- **Source:** src/app/containers/AboutUsPage/index.tsx (renders HomePage/aboutUs AboutUs component)

### POL-01: Terms of Service policy page renders
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/policies/terms
- **Expected:**
  - A loading spinner shows briefly, then the page renders.
  - Left/side navigation "Terms and policies" lists: "Terms of service" (active/highlighted), "Cancellation policy", "Privacy policy", "Account deletion".
  - Page title "Terms of Service" and markdown body content render.
  - No occurrence of the literal word "Turo" in visible text (replaced with "Autopilotee Cars").
- **Source:** src/app/containers/TermsOfServicePolicyPage/index.tsx; src/app/containers/PolicyPage/index.tsx; public/policies/terms.md

### POL-02: Cancellation Policy page renders
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/policies/cancellation
- **Expected:** Title "Cancellation Policy"; markdown body renders; "Cancellation policy" highlighted in the side nav; phone numbers (if any) styled red/bold.
- **Source:** src/app/containers/CancellationPolicyPage/index.tsx; public/policies/cancellation.md

### POL-03: Privacy Policy page renders
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/policies/privacy
- **Expected:** Title "Privacy Policy"; markdown body renders; "Privacy policy" highlighted in the side nav.
- **Source:** src/app/containers/PrivacyPolicyPage/index.tsx; public/policies/privacy.md

### POL-04: Policy side-navigation cross-links work
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On any /policies/* page.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/policies/terms
  2. Click "Privacy policy" in the "Terms and policies" side nav
  3. Click "Account deletion" in the side nav
- **Expected:**
  - Step 2 navigates to /policies/privacy and that page's title renders.
  - Step 3 navigates to /account-deletion and the Account Deletion content page renders.
  - The active item highlight follows the current route.
- **Source:** src/app/containers/PolicyPage/PolicyNavigation.tsx

### ACCT-01: Account Deletion content page renders
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** None. (This is a content page, not the delete action.)
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/account-deletion
- **Expected:**
  - Title "Account Deletion"; markdown from /policies/account-deletion.md renders.
  - The "Terms and policies" side nav shows "Account deletion" highlighted.
  - No real account is affected by viewing this page (no mutation).
- **Source:** src/app/containers/AccountDeletionPage/index.tsx (renders PolicyPage with /policies/account-deletion.md)

### ACCT-02: Account deletion error fallback when markdown missing
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** None. Negative check; only verifiable if the markdown fetch fails.
- **Steps:**
  1. Navigate to a policy route whose markdown is unavailable (executor: only run if a 404 on the .md can be induced)
- **Expected:** The page shows red error text "Unable to load <Title>. Please try again later." while the side nav still renders.
- **Source:** src/app/containers/PolicyPage/index.tsx (error state)

### DEAL-00: Dealership microsite is domain-gated (negative on staging)
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** None.
- **Steps:**
  1. On https://staging.autopilotee.com, attempt to reach a dealership-only route such as /cars_on_sale or /contact (the dealership Contact page)
- **Expected:**
  - These dealership-only routes are NOT active on the autopilotee.com domain. The app only enables dealership routes when hostname includes autopiloteecars.com or jidosojuholding.com.
  - Expect the main-app behavior instead (e.g. catch-all spinner/redirect), confirming the dealership microsite does not render on staging.autopilotee.com.
- **Source:** src/App.tsx (isDealershipSite hostname check; renderDealershipRoutes)

### DEAL-01: Dealer page (/dealership) loads for a Host with a dealer
- **Priority:** P2  **Persona:** Host
- **Preconditions:** Logged in as the account that is a Host and owns a dealer (GET_MY_DEALER returns data). Discover which of USER1/USER2 qualifies; if neither, mark as not-applicable.
- **Steps:**
  1. Log in as the host account
  2. Navigate to https://staging.autopilotee.com/dealership
- **Expected:**
  - The dealer page renders without crashing: a left column with car preview / dealer info / navigation list, plus dealer cars-on-sale and agreements content.
  - If the account has no dealer, the page renders an empty/no-dealer state rather than erroring.
- **Source:** src/app/containers/DealerPage/index.tsx (GET_MY_DEALER, GET_SPECIFIC_DEALER_AGREEMENTS)

### DEAL-02: Dealership Contact form validation (microsite, requires dealer domain)
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Only reachable on a dealership domain (autopiloteecars.com / jidosojuholding.com), NOT on staging.autopilotee.com. Run only if a dealership staging domain is available; otherwise mark not-applicable per DEAL-00.
- **Steps:**
  1. Open the dealership "/contact" page
  2. Click "Send Message" with empty required fields
  3. Fill First Name, Last Name, Email, and Message, leave Phone blank
  4. [MUTATING] Click "Send Message"
- **Expected:**
  - Step 2: a red toast "Please fill out all fields" appears; no email sent (button disabled until firstName, lastName, email, message all present).
  - Step 4: button shows "Sending..." then a green success banner "Your message has been sent successfully. We'll get back to you soon!" appears for ~5s and the form clears.
  - GraphQL operation: sendContactFormEmail.
- **Source:** src/app/containers/DealershipContact/index.tsx
