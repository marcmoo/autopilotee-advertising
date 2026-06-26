# AGENTS.md

Marketing strategy, automation prompts, task definitions, and reusable channel copy for **carsharingwhitelabel.com**.

## Marketing Asset Rules

- This repo is the source of truth for **carsharingwhitelabel.com** marketing assets and automation prompts. Do not duplicate campaign assets into product repos unless a specific implementation needs a copy.
- `emails/` contains cold outreach templates and lead lists. `emails/p2p_car_rental_companies.csv` is the initial 506-row seed list; the email column is mostly empty, so enrichment is the first automation task.
- `marketing/planning/saas-marketing-gtm.md` is the bilingual SaaS marketing and monetization strategy for positioning `autopilotee-cars` + `autopiloteeFlutter` as the **carsharingwhitelabel.com** white-label rental SaaS.
- `marketing/agent-brief/agent-operating-brief.md` is the machine-readable operating brief for Hermes (Gemma 12B) and OpenClaw.
- `marketing/agent-brief/tasks/*.yaml` defines scheduled automation tasks for enrichment, scraping, classification, generation, distribution, and reply tracking.
- `marketing/agent-brief/templates/*.md` contains reusable channel copy for cold email, Reddit, Facebook groups, X/Twitter, LinkedIn, DMs, and blog/SEO outlines.

## Marketing Automation Operating Rules

- **Hermes (Gemma 12B)** is the content/classification brain: classify leads, score fit, personalize copy, and draft replies.
- **OpenClaw** is the browser/computer automation layer: enrich contacts, scrape leads, post/comment/DM, and send approved cold email.
- Start from `emails/p2p_car_rental_companies.csv`; enrich missing emails and contacts before expanding to new lead sources.
- All automation must funnel qualified prospects to `carsharingwhitelabel.com` and the Calendly CTA `https://calendly.com/jidosoju/30min`.
- Keep the core messaging consistent: "Every feature Turo has, we have more — your own white-label car-sharing web + mobile app."
- Do not publish, DM, comment, or send cold emails without a human approval gate. Agent-generated content is a draft until approved.
- Cold email must include real sender identity, physical mailing address, and one-click unsubscribe. Honor opt-outs permanently.
- Respect each platform's ToS. Use rate limits, account warm-up, and value-first participation; never blast identical copy.
