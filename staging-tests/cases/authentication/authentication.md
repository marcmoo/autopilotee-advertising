# Authentication Test Cases

Base URL: https://staging.autopilotee.com

Credentials (provided to executor, do not invent others):
- USER1: jidosoju+5@gmail.com / <password: see staging_credentials.local>
- USER2: autopilotee+206503@gmail.com / <password: see staging_credentials.local>
- Role of each account (guest vs host) is unknown; the executor discovers it. Any authenticated account can toggle role via the "Switch to host/guest" button.

How auth UI works (grounded in source):
- All auth flows are MODALS, not separate routes. They are rendered by `AuthModals` in `navBar/navItems.tsx`.
- The nav menu is opened by the hamburger button (FontAwesome `faBars`) at top right. Inside the open menu, "Log in" calls `modalActions.openSignInModal()` and "Sign up" calls `modalActions.openSignUpModal()`.
- Modals can also be opened by URL params: `?signin=true` opens sign-in; `?purpose=VERIFICATION` opens verify-code; `?purpose=RESET_PASSWORD` opens reset-password (see `AuthModals` + `modalStore.initializeFromParams`).
- On successful sign-in a toast "Login successful!" appears and the modal closes after ~500ms. Tokens + user persist to `localStorage` keys `tokens` and `user`.
- On logout (`Sign out` button on `/profile`) tokens/user are cleared and role resets to guest.

Key data-testid selectors (from integration-test commit):
- Sign-in: `signin-modal-container`, `signin-email-input`, `signin-password-input`, `signin-submit-button`, `signin-forgot-password-button`, `signin-signup-link`, `signin-google-button`
- Sign-up: `signup-modal-container`, `signup-firstname-input`, `signup-lastname-input`, `signup-email-input`, `signup-password-input`, `signup-terms-checkbox`, `signup-promotions-checkbox`, `signup-submit-button`, `signup-signin-link`
- Forgot/Reset/Verify modals have NO data-testids; select by visible text/placeholder.

---

### AUTH-01: Sign in with valid credentials
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Logged out (no `tokens` in localStorage). Valid account USER1 exists and is email-confirmed.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true (opens sign-in modal directly).
  2. Wait for `data-testid="signin-modal-container"` to be visible; confirm title "Log in with email".
  3. Fill `data-testid="signin-email-input"` with `jidosoju+5@gmail.com`.
  4. Fill `data-testid="signin-password-input"` with `<password: see staging_credentials.local>`.
  5. [MUTATING] Click `data-testid="signin-submit-button"` ("Continue"). (Creates a real auth session / token on staging.)
- **Expected:**
  - A success toast "Login successful!" appears.
  - The sign-in modal closes within ~1s.
  - `localStorage.tokens` and `localStorage.user` are populated (accessToken/idToken/refreshToken present).
  - Opening the nav menu now shows "Account", "Switch to host/guest", and "Chats" instead of "Log in"/"Sign up".
- **Source:** src/app/components/auth/SignInModal.tsx (handleSubmit, SIGN_IN onCompleted), src/store/authStore/index.ts (setAuth), src/app/components/navBar/navItems.tsx (renderAuthButtons)

### AUTH-02: Sign in with invalid password shows error toast
- **Priority:** P0  **Persona:** Anonymous
- **Preconditions:** Logged out.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Fill `data-testid="signin-email-input"` with `jidosoju+5@gmail.com`.
  3. Fill `data-testid="signin-password-input"` with `WrongPass123!`.
  4. Click `data-testid="signin-submit-button"`.
- **Expected:**
  - An error toast appears with the backend error message (e.g. incorrect username/password). It does not contain "Login successful!".
  - The modal stays open; `signin-modal-container` remains visible.
  - `localStorage.tokens` is not set / user is not authenticated.
- **Source:** src/app/components/auth/SignInModal.tsx (signIn onError else-branch -> toast.error)

### AUTH-03: Sign in with unknown/unregistered email
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Fill `signin-email-input` with `no-such-user-<timestamp>@example.com`.
  3. Fill `signin-password-input` with `Whatever123!`.
  4. Click `signin-submit-button`.
- **Expected:**
  - An error toast appears (user not found / auth failed). The modal stays open and no session is created.
- **Source:** src/app/components/auth/SignInModal.tsx (onError handling for REGISTERED_USER_AUTH_FAILED / generic)

### AUTH-04: Sign in with unverified user shows "check your email" resend page
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Logged out. An existing-but-unconfirmed Cognito account is available (executor may use a freshly signed-up account from AUTH-08 that was never confirmed). Skip if no such account.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Enter the unverified account email and its password.
  3. Click `signin-submit-button`.
- **Expected:**
  - The modal switches to the "Please check your Email Inbox" page (heading text "Please check your Email Inbox").
  - A "Resend Verification Code" button is shown; after first auto-send it enters a 60s countdown ("Resend Verification Code (60)").
  - A "Confirmed, Back to Login" button returns to the login form.
- **Source:** src/app/components/auth/SignInModal.tsx (onError UNVERIFIED_USER branch, ResendConfirmationCode flow)

### AUTH-05: Open sign-in modal from nav menu
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/.
  2. Click the hamburger nav toggle button (top right, faBars icon).
  3. In the open menu, click the "Log in" item.
- **Expected:**
  - `data-testid="signin-modal-container"` becomes visible with title "Log in with email".
  - The nav menu closes.
- **Source:** src/app/components/navBar/navItems.tsx (renderAuthButtons "Log in" -> modalActions.openSignInModal)

### AUTH-06: Switch from sign-in modal to sign-up modal and back
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out. Sign-in modal open.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Wait for `signin-modal-container`.
  3. Click `data-testid="signin-signup-link"` ("No Account? Sign Up").
  4. Confirm `data-testid="signup-modal-container"` is visible (title "Let's get started", `signup-firstname-input` present).
  5. Click `data-testid="signup-signin-link"` ("Log in").
- **Expected:**
  - Step 3-4: sign-up modal replaces sign-in modal (only one open at a time per `switchToSignUp`).
  - Step 5: sign-in modal is shown again; `signin-email-input` visible.
- **Source:** src/app/components/auth/SignInModal.tsx (handleSwitchToSignUp), SignUpModal.tsx (switchToSignIn), src/store/authStore/modalStore.ts

### AUTH-07: Sign-up form validation (submit disabled until valid + password rules)
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out. Sign-up modal open.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/ and open Sign up (nav menu -> "Sign up", or AUTH-06 step 3).
  2. Confirm `data-testid="signup-submit-button"` is disabled (no fields filled).
  3. Fill `signup-firstname-input` = "Test", `signup-lastname-input` = "User", `signup-email-input` = "x@y.com".
  4. Type `abc` into `signup-password-input` and observe the requirement checklist.
  5. Change password to `Abcdef12`.
- **Expected:**
  - Step 2: submit button disabled (`isFormValid` false).
  - Step 4: requirement rows show: "At least 8 characters" not met, "At least 1 uppercase letter" not met, "At least 1 lowercase letter" met, "At least 1 number" not met (icons ○ vs ✓).
  - Step 5: all four requirement icons turn ✓ and `signup-submit-button` becomes enabled.
- **Source:** src/app/components/auth/SignUpModal.tsx (validatePassword, isFormValid, PasswordRequirements)

### AUTH-08: Sign up new account triggers verify-code modal
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out. Use a unique, never-registered email the executor controls.
- **Steps:**
  1. Open the Sign up modal.
  2. Fill `signup-firstname-input`, `signup-lastname-input`, `signup-email-input` (unique address), `signup-password-input` = `Abcdef12`.
  3. [MUTATING] Click `data-testid="signup-submit-button"`. (Creates a real Cognito user + sends a verification email.)
- **Expected:**
  - Toast "Please check your email for a confirmation link".
  - Sign-up modal closes and the Verify-code modal opens (heading "Verify your email", subtitle mentions the entered email).
  - Two text inputs (email, Verification Code) and a "Verify Email" button are visible; "Verify Email" is disabled until a code is entered.
- **Source:** src/app/components/auth/SignUpModal.tsx (signUp onCompleted -> openVerifyCodeModal), VerifyCodeModal.tsx

### AUTH-09: Sign up with already-registered email switches to sign-in with bilingual notice
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Logged out. Use an email that already exists (e.g. USER1 `jidosoju+5@gmail.com`).
- **Steps:**
  1. Open the Sign up modal.
  2. Fill names, set `signup-email-input` = `jidosoju+5@gmail.com`, password `Abcdef12`.
  3. [MUTATING] Click `signup-submit-button`. (Hits backend signUp; no new account created since email exists.)
- **Expected:**
  - An error toast appears stating the email already exists (bilingual: "Email already exists. Please sign in with Google or use Forgot password to set a new password.").
  - The app switches to the sign-in modal (`signin-modal-container` visible).
- **Source:** src/app/components/auth/SignUpModal.tsx (onError PreSignUpEmailExistsError / UsernameExists branch -> switchToSignIn)

### AUTH-10: Forgot password sends reset link and opens reset modal
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out. Sign-in modal open. Use USER1 email.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Optionally pre-fill `signin-email-input` with `jidosoju+5@gmail.com`.
  3. Click `data-testid="signin-forgot-password-button"` ("Forgot password?").
  4. In the Forgot Password modal (title "Forgot Password"), enter `jidosoju+5@gmail.com` in the Email field.
  5. [MUTATING] Click "Send Reset Link". (Triggers a real password-reset email via Cognito.)
- **Expected:**
  - Step 3: Forgot Password modal opens; current email may be prefilled via `setCurrentEmail`.
  - "Send Reset Link" is disabled when the email field is empty.
  - Step 5: toast "Please check your email for a reset link"; Forgot modal closes and the Reset Password modal opens (title "Reset your password") with the email prefilled.
- **Source:** src/app/components/auth/SignInModal.tsx (forgot-password button), ForgotPwdModal.tsx (onCompleted -> openResetPwdModal)

### AUTH-11: Reset password modal validation (passwords-match + requirements)
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Reset Password modal open (via AUTH-10, or navigate to https://staging.autopilotee.com/?purpose=RESET_PASSWORD&email=jidosoju%2B5%40gmail.com).
- **Steps:**
  1. Confirm "Reset your password" modal is visible with Email, Verification Code, New Password, Confirm Password fields.
  2. Enter New Password = `Abcdef12` and observe the requirement checklist turn green.
  3. Enter Confirm Password = `Different1` (mismatch).
  4. Observe the "Reset Password" button and validation message.
  5. Correct Confirm Password to `Abcdef12` and enter any value in Verification Code.
- **Expected:**
  - Step 2: all four password requirement icons turn ✓.
  - Step 3: "Passwords do not match" validation message shown; "Reset Password" button disabled.
  - Step 5: with email + code + matching valid password, the "Reset Password" button becomes enabled. (Do NOT submit without a real code, or it will toast a backend error.)
- **Source:** src/app/components/auth/ResetPwdModal.tsx (validatePassword, passwordsMatch, isFormValid)

### AUTH-12: Session persists across page reload
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Successfully signed in (AUTH-01 complete); `localStorage.tokens` present.
- **Steps:**
  1. While logged in, reload https://staging.autopilotee.com/ (full page refresh).
  2. Wait for app to initialize (`authActions.initAuth` runs on load).
  3. Open the nav menu.
- **Expected:**
  - The user remains authenticated after reload (no re-login prompt).
  - Nav menu shows "Account", "Switch to host/guest", "Chats" (authenticated state), not "Log in"/"Sign up".
  - `localStorage.tokens` and `localStorage.user` still present.
- **Source:** src/store/authStore/index.ts (initAuth / restoreTokens reads localStorage and refetches GetCurrentUser)

### AUTH-13: Sign out clears session and resets to guest
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Signed in. On a desktop session.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/profile.
  2. [MUTATING] Click the "Sign out" button. (Clears server/device session token, role reset.)
- **Expected:**
  - Toast "Logged out successfully".
  - Redirected to home (`/`).
  - `localStorage.tokens` and `localStorage.user` removed; role reset to guest (`localStorage.role` removed).
  - Nav menu now shows "Log in" / "Sign up".
- **Source:** src/app/containers/UserPage/ProfilePage/index.tsx (handleLogout), src/store/authStore/index.ts (logout), roleStore (resetRole)

### AUTH-14: Protected content prompts sign-in when logged out
- **Priority:** P1  **Persona:** Anonymous
- **Preconditions:** Logged out (clear localStorage `tokens`/`user`).
- **Steps:**
  1. Navigate directly to https://staging.autopilotee.com/guest/mybookings.
  2. Wait for the page to settle (networkidle).
- **Expected:**
  - The page does not display another user's bookings.
  - An authentication prompt is reachable: either a sign-in modal/login form appears (email + password inputs), or the page shows an empty/unauthenticated state with a way to log in. (Note: routes are not hard-guarded with a redirect in source; the page renders but data requires auth. Confirm no booking data leaks.)
- **Source:** e2e/auth-flows.spec.ts (navigates to /guest/mybookings expecting a login form), src/App.tsx (route is public component MyBookingsPage), src/app/components/navBar/navItems.tsx (My Bookings only rendered when authenticated)

### AUTH-15: Role switch toggles between host and guest (authenticated only)
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Signed in with an account that has host capability (executor discovers which of USER1/USER2). For pure UI toggle, any authenticated account works.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/ while logged in.
  2. Note the "Switch to host" / "Switch to guest" button label in the nav button row.
  3. Click the role toggle button.
  4. Open the nav menu and inspect the listed nav items.
- **Expected:**
  - Clicking the toggle switches role and navigates to `/` (home).
  - The button label flips (host <-> guest).
  - In HOST mode the menu shows host-only links: Locations, Insurance, Product Types, Tripfee, Coupons, Damage Claims, Refund Invoices, Manage ShortID, and "Trips" (instead of "My Bookings").
  - In GUEST mode the menu shows "My Bookings", "Cancelled Bookings", "Damage Claims".
  - `localStorage.role` updates to the selected role.
  - The role toggle button is NOT shown when logged out.
- **Source:** src/app/components/navBar/navItems.tsx (toggleRole, renderNavLinks host branch, ToggleButton render condition), src/store/roleStore/index.ts (setRole)

### AUTH-16: Logged-out user has no role toggle and role stays guest
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Logged out.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/.
  2. Open the nav menu and inspect the nav button row.
- **Expected:**
  - No "Switch to host/guest" button is present.
  - Menu shows only public items plus "Log in" / "Sign up"; no host links, no "My Bookings", no "Chats".
- **Source:** src/app/components/navBar/navItems.tsx (ToggleButton gated on isAuthenticated; renderBookingOrTrips returns null when not authenticated), src/store/roleStore/index.ts (initRole forces GUEST when not authenticated)

### AUTH-17: Verify-code modal requires a code before submit
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Verify-code modal open (after AUTH-08, or navigate to https://staging.autopilotee.com/?purpose=VERIFICATION&email=test%40example.com).
- **Steps:**
  1. Confirm "Verify your email" modal is visible.
  2. Leave the Verification Code field empty; observe the "Verify Email" button.
  3. Type any value into the Verification Code field.
- **Expected:**
  - Step 2: "Verify Email" button is disabled while the code field is empty/whitespace.
  - Step 3: button becomes enabled once a non-empty code is entered.
  - (Do NOT submit a fake code against a real account; it will toast a backend error.)
- **Source:** src/app/components/auth/VerifyCodeModal.tsx (isDisabled={!codeState.trim()}, handleVerifyCode)

### AUTH-18: Google OAuth sign-in button redirects to Cognito hosted UI
- **Priority:** P2  **Persona:** Anonymous
- **Preconditions:** Logged out. Sign-in modal open.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/?signin=true.
  2. Click `data-testid="signin-google-button"` (Google icon).
- **Expected:**
  - The browser begins an OIDC redirect to the Cognito/Google hosted login (`auth.signinRedirect` with `identity_provider=Google`). URL leaves the app toward the Cognito domain / Google consent.
  - Do NOT complete a real Google login. Verify only that the redirect is initiated, then navigate back.
- **Source:** src/app/components/auth/SignInModal.tsx (handleCognitoGoogleSignIn -> auth.signinRedirect)
