# Landing Site — Marketing Pages, Nav & CTAs

App: autopilotee-landing (Next.js marketing site).

IMPORTANT domain caveat:
- The landing site is a SEPARATE Next.js app from the React car app at https://staging.autopilotee.com. The credentials and staging URL in this task target the React app. The marketing site is typically served from the apex marketing domain (e.g. the production/staging marketing host), not from the React app's host.
- The executor must first DISCOVER the landing site's reachable URL. Try, in order: the marketing root that serves the Next.js `app/` routes (Features `/`, `/demo`, `/pricing`, `/mobile-features`, `/about`, `/privacy`). If none resolve on the available hosts, mark LAND-* as not-applicable for this environment and record the attempted URLs.
- Primary demo CTAs go to Calendly (https://calendly.com/jidosoju/30min) with UTM params appended; "Get Started" goes to /pricing.

---

### LAND-01: Landing home page renders core sections
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Landing site root URL discovered.
- **Steps:**
  1. Navigate to the landing site root "/"
- **Expected:**
  - Home renders without error: video banner, Hero, comparison table, ROI calculator, and a bottom CTA section.
  - Bottom CTA heading "Ready to Own Your Platform?" with a "Book a Demo" button.
  - Page title contains "White-Label ... Software".
  - No literal "Turo" text in visible copy (per content guidelines; Turo allowed only in invisible SEO metadata/JSON-LD).
- **Source:** autopilotee-landing/app/page.tsx; components/CTASection.tsx

### LAND-02: Top navbar links navigate to marketing pages
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On the landing root.
- **Steps:**
  1. Click nav item "Demo"
  2. Go back, click nav item "Pricing"
  3. Go back, click nav item "App showcase"
  4. Go back, click nav item "Features"
- **Expected:**
  - "Demo" -> /demo, "Pricing" -> /pricing, "App showcase" -> /mobile-features, "Features" -> / each load the corresponding page without 404.
  - The active nav item is visually highlighted on its page.
- **Source:** autopilotee-landing/components/Navbar.tsx; lib/siteNav.ts (mainNavItems)

### LAND-03: "Get Started" button routes to pricing
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On the landing root (desktop width so the button is visible).
- **Steps:**
  1. Click the "Get Started" button in the top-right of the navbar
- **Expected:** Navigates to /pricing and the pricing page renders (pricing cards grid).
- **Source:** autopilotee-landing/components/Navbar.tsx (Link href="/pricing"); app/pricing/page.tsx

### LAND-04: "Book a Demo" CTA opens Calendly with UTM params
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** On the landing root.
- **Steps:**
  1. Scroll to the bottom CTA section ("Ready to Own Your Platform?")
  2. Click "Book a Demo"
- **Expected:**
  - Opens https://calendly.com/jidosoju/30min in a new tab (target=_blank, rel=noopener), with UTM query params appended (utm_source=website, utm_medium=cta, utm_campaign=demo_booking, utm_content=cta_section_...).
  - No navigation away from the marketing page in the original tab.
- **Source:** autopilotee-landing/components/CTASection.tsx; components/BookDemoLink.tsx

### LAND-05: Mobile menu opens on small viewport
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** On the landing root with a mobile viewport (< lg breakpoint).
- **Steps:**
  1. Resize/emulate a mobile viewport
  2. Click the hamburger button (aria-label "Open menu")
- **Expected:** The mobile menu (#site-mobile-menu) opens showing the nav items; aria-expanded toggles true. A close action dismisses it.
- **Source:** autopilotee-landing/components/Navbar.tsx; components/MobileMenu.tsx

### LAND-06: Pricing page renders pricing cards
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Landing site reachable.
- **Steps:**
  1. Navigate to /pricing
- **Expected:** Pricing page renders a grid of pricing cards without error.
- **Source:** autopilotee-landing/app/pricing/page.tsx; components/PricingCardsGrid.tsx

### LAND-07: Demo page renders
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Landing site reachable.
- **Steps:**
  1. Navigate to /demo
- **Expected:** The demo page renders (e.g. embedded demo / YouTube demo content) without error.
- **Source:** autopilotee-landing/app/demo/page.tsx; components/YouTubeDemoEmbed.tsx

### LAND-08: Mobile features / app showcase page renders
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Landing site reachable.
- **Steps:**
  1. Navigate to /mobile-features
- **Expected:** The app showcase page renders the mobile features showcase without error.
- **Source:** autopilotee-landing/app/mobile-features/page.tsx; components/MobileFeaturesShowcase.tsx

### LAND-09: About and Privacy marketing pages render
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Landing site reachable.
- **Steps:**
  1. Navigate to /about
  2. Navigate to /privacy
- **Expected:** Both pages render their respective body content (AboutPageBody, PrivacyPageBody) without error and no visible "Turo" copy.
- **Source:** autopilotee-landing/app/about/page.tsx; app/privacy/page.tsx; components/AboutPageBody.tsx; components/PrivacyPageBody.tsx
