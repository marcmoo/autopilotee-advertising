# Workflow: Authentication Journey

Chains the atomic AUTH cases into one realistic session covering sign-in, persistence, role switching, and sign-out. All auth flows are MODALS (not routes), rendered by `AuthModals` in `navBar/navItems.tsx`.

- **Env:** https://staging.autopilotee.com (Chrome via Playwright MCP)
- **Credentials:** see `staging_credentials.local` (USER1, USER2)
- **Key selectors:** `signin-modal-container`, `signin-email-input`, `signin-password-input`, `signin-submit-button`, `signin-signup-link`, `signin-forgot-password-button`, `signin-google-button`; `signup-modal-container`, `signup-firstname-input`, `signup-lastname-input`, `signup-email-input`, `signup-password-input`, `signup-terms-checkbox`, `signup-submit-button`.
- **localStorage keys:** `tokens`, `user`, `role`.

## Setup
1. Open https://staging.autopilotee.com and clear `localStorage` (`tokens`, `user`, `role`) to guarantee a logged-out start.

## Flow
1. **AUTH-05** — Open the hamburger nav; confirm "Log in" / "Sign up" appear.
2. **AUTH-06** — In the sign-in modal, click `signin-signup-link` to switch to sign-up, then `signup-signin-link` back. Confirm both modals toggle.
3. **AUTH-02** — Attempt sign-in with a wrong password; confirm error toast, modal stays open, no `tokens`.
4. **AUTH-01** — [MUTATING-session] Sign in with USER1 valid credentials (see `staging_credentials.local`). Confirm "Login successful!" toast, modal closes, `localStorage.tokens`+`user` populated.
5. **AUTH-12** — Reload the page; confirm session persists (still authenticated, nav shows "Account"/"Chats").
6. **AUTH-15** — Open hamburger, click "Switch to host"; confirm host-only nav links appear and `localStorage.role` updates. Toggle back to guest. Record whether USER1 is the host account.
7. **AUTH-13** — [MUTATING-session] Go to `/profile`, click "Sign out"; confirm tokens/user cleared, role reset to guest, nav shows "Log in"/"Sign up".

## Role-discovery branch
- If USER1 has no host links after step 6, repeat steps 4-6 with USER2 to identify the host account. Persist the discovered mapping for downstream host workflows.

## Optional / careful (run only with approval)
- **AUTH-08** — [MUTATING] Sign up a brand-new unique email -> verify-code modal appears (do NOT submit a fake code).
- **AUTH-10** — [MUTATING] Forgot-password sends a real reset email.
- **AUTH-18** — Initiate Google OAuth redirect only; do not complete Google login.

## Teardown
- Ensure the session is signed out (AUTH-13) so later anonymous cases start clean, OR keep USER1 signed in if proceeding directly into the guest-booking workflow.
