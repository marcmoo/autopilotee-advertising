# AGENTS.md

Advertising/media support repo for Autopilotee assets, prompts, and demo videos.

## Repository Guidance

- Keep source prompts, message logs, and intentionally shared media assets organized by purpose.
- Do not commit OS metadata such as `.DS_Store`.
- Large video files should be compressed before commit and named descriptively, for example `calendar-blocking_compressed.mp4`.
- If adding generated exports, include the source or prompt that explains how to regenerate them when possible.

## Current Asset Areas

- `promptMessages/` — prompt/message logs and campaign copy drafts.
- `videos/` — compressed demo videos for product/marketing use.
- `emails/` — email assets.
- `staging-tests/` — manual / Playwright-driven QA test suite for the staging web app (`https://staging.autopilotee.com`): master plan, per-area cases, end-to-end workflows, and execution results. Never commit real account passwords here; reference `staging_credentials.local` instead.
