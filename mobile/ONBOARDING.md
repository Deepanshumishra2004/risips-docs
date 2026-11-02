# Onboarding and KYC

How a new investor goes from sign-up to a ready account, and how the app knows which screen to show. Verified against
`mobile/src/app/onboarding/*`, `mobile/src/lib/session.tsx`, `mobile/src/lib/kyc.ts`, and the TypeScript backend
(`backend-ts/src/services/user.service.ts`, `onboarding.service.ts`, `kyc.service.ts`, `orders.service.ts`).

## The backend decides the next screen

The app never keeps its own list of onboarding steps. Every user record from the backend (`GET /me`, and the `user`
returned by almost every `/onb/*` and `/auth/*` call) carries `next_step`. The app maps it to a route:

- `routeForStep(step)`: `home` goes to `/home`; anything else goes to `/onboarding/<step>`.
- `goNext(user)`: clears the navigation stack, then replaces the screen with that route. A finished step is never left
  behind on Back (submitting it again would fail, for example "already confirmed").

So after a screen saves its data, it calls `setUser(r.user)` and `goNext(r.user)`, and the backend's answer picks the
next screen. A user who closes the app and comes back resumes at the right step: the splash (or the unlock screen)
routes to `routeForStep(user.next_step)`.

`nextStep()` in `backend-ts/src/services/user.service.ts` checks these in order and returns the first missing one:

| # | Returned step | Missing data | Who gets it |
|---|---|---|---|
| 0 | `home` | already onboarded (`onboarded_at` set) | everyone after eSign; never goes back into the flow |
| 1 | `mobile` | mobile not verified | everyone |
| 2 | `pin` | PIN not set | everyone |
| 3 | `pan` | PAN not verified | everyone |
| 4 | `pan-confirm` | PAN not confirmed | everyone |
| 5 | `occupation` | occupation | everyone |
| 6 | `income` | income slab | everyone |
| 7 | `marital` | marital status | everyone |
| 8 | `father` | father's or spouse's name | everyone |
| 9 | `address` | address or gender | everyone |
| 10 | `aadhaar` | Aadhaar not verified | only if NOT KYC-compliant |
| 11 | `bank` | bank not verified | everyone |
| 12 | `selfie-intro` | no selfie | only if NOT compliant AND NOT fresh KYC (old accounts; see "Legacy screens") |
| 13 | `signature-consent` | FATCA not declared, or (not compliant and) no signature | everyone (declarations); non-compliant also need a signature |
| 14 | `kyc-esign` | fresh-KYC form not yet e-signed (status not `awaiting_submission` / `submitted`) | only fresh KYC |
| 15 | `nominee` | no nominee and no opt-out (each confirmed with an OTP) | everyone |
| 16 | `esign` | not onboarded | everyone |

"KYC-compliant" (`kyc_compliant` on the user) means FP pre-verification found a valid KYC at the KRA for this PAN
(readiness `verified`). "Fresh KYC" (`kyc_fresh`) means the KRA has no KYC for the PAN and the investor does a new KYC
inside the app.

The screens `mobile-otp` and `pin-done` are not steps of their own: `mobile` pushes `mobile-otp`, and `pin` replaces
itself with `pin-done`.

## Which path a PAN takes

The PAN screen sends `POST /onb/pan { pan, name, dob, fresh_kyc: canDoFreshKyc() }`. FP pre-verification checks four
things together: KRA readiness, PAN (valid and Aadhaar-linked), name, and date of birth.

| Result | What happens |
|---|---|
| All four verified | Path A: KYC-compliant |
| Only readiness fails, with code `kyc_unavailable` (no KRA record), AND the app sent `fresh_kyc: true` | Path B: fresh KYC (`kyc_fresh = true`) |
| Anything else (including `kyc_unavailable` with `fresh_kyc: false`) | `422` with a message; the investor stays on the PAN screen |

`canDoFreshKyc()` (`src/lib/kyc.ts`) returns `true` only when the app can `require('expo-location')`. App builds made
before `expo-location` was added can't, so they send `fresh_kyc: false` and keep getting the 422 for a PAN with no KRA
record. The message they see is the backend's "No KYC record found for this PAN with the KRA. A fresh KYC is needed."
Such users need a newer app build (`expo-location` is native, so an OTA update can't add it).

Other PAN errors the backend returns (from `onboarding.service.ts`): KRA on hold / rejected / legacy / incomplete /
under process / deactivated, "PAN is not linked with Aadhaar", "This PAN is invalid", "The name does not match PAN
records", "The date of birth does not match PAN records", "The KRA service is unavailable", and "You must be 18 or
older".

## Path A: KYC-compliant PAN

```
email sign-in -> mobile -> mobile-otp -> pin -> pin-done -> pan -> pan-confirm -> occupation -> income -> marital
  -> father -> address -> bank -> signature-consent (declarations only) -> nominee -> esign -> done -> home
```

No Aadhaar, selfie or signature is collected: the KRA already holds this investor's KYC.

| Step | Screen | What the investor does | Backend call |
|---|---|---|---|
| Sign-in | `/auth/email`, `/auth/email-otp` | Email, then the emailed code | `POST /auth/email/start`, `POST /auth/email/verify` (returns the session token) |
| Mobile | `/onboarding/mobile`, `/onboarding/mobile-otp` | Mobile number (+91, starts 6-9), code, and the required "I authorise risips to validate my bank details" tick | `POST /auth/mobile/start`, `POST /auth/mobile/verify {mobile, code, bank_consent}` |
| PIN | `/onboarding/pin`, `/onboarding/pin-done` | 4-digit PIN twice; optionally turn on biometric unlock | `POST /auth/pin/set`, `POST /auth/biometric {enabled}` |
| PAN | `/onboarding/pan` | PAN (individual: 4th letter `P`), full name as on PAN, DOB (DD/MM/YYYY, 18+) | `POST /onb/pan` |
| PAN confirm | `/onboarding/pan-confirm` | Checks the PAN card preview and the four results, then "Yes, confirm" | `POST /onb/pan/confirm` |
| Occupation, income, marital | `/onboarding/occupation`, `income`, `marital` | Pick one option each | `POST /onb/profile {occupation | income_slab | marital_status}` |
| Father / spouse | `/onboarding/father` | Name, 3+ characters | `POST /onb/profile {father_name}` |
| Address | `/onboarding/address` | Address lines, pincode (looked up: city, state), gender | `GET /onb/pincode/:pin`, `POST /onb/address` |
| Bank | `/onboarding/bank` | Account number twice, IFSC (looked up: bank and branch) | `GET /onb/ifsc/:ifsc`, `POST /onb/bank` (FP pre-verification; no money moves) |
| Declarations | `/onboarding/signature-consent` | Titled "Declarations". FATCA/CRS tick (required); running-account authorisation (optional) | `POST /onb/declarations {fatca, running_account}`; then `goNext` |
| Nominee | `/onboarding/nominee` | Add a nominee (name, DOB 18+, relationship, PAN) or Skip and confirm the opt-out declaration; both confirmed with a 6-digit OTP | `POST /txn/otp {action: 'nominee'}`, `POST /onb/nominee` |
| eSign | `/onboarding/esign` | Titled "Verify your details": an account opening summary marked "KYC compliant · verified with the KRA"; "Confirm details", tick the consent, "Agree & open account" | `POST /onb/esign {consent: true}` (creates the investor, bank, nominee and MF account at FP) |
| Done | `/onboarding/done` | "ALL SET <name>!" and "Your document verification is in process..." ; "Get Started" | none |

## Path B: fresh KYC (PAN with no KRA record)

```
... -> pan -> pan-confirm -> occupation -> income -> marital -> father -> address
  -> aadhaar (DigiLocker) -> kyc-web (step=proof) -> bank -> signature-consent -> signature
  -> kyc-esign -> kyc-web (step=esign) -> nominee -> esign -> done -> home (KYC banner until the KRA approves)
```

Steps up to address are the same as path A. What differs:

### PAN confirm

The four check results show "KRA KYC status" with a cross, and under them: "Your PAN has no KYC with the KRA yet. We'll
do it here: Aadhaar through DigiLocker, your signature and an e-sign. The KRA then approves it in 1-3 working days."
(`user.kyc_fresh` is true at this point.)

### Aadhaar through DigiLocker (`/onboarding/aadhaar`)

`aadhaar.tsx` renders the DigiLocker version when `user.kyc_fresh` is true. On "Continue with DigiLocker":

1. Loads `expo-location` with `require()` inside the action (not a top-level import, so an older build that receives
   this screen by OTA does not crash at start). If it is missing: "Please update the Risips app to continue your KYC."
2. Asks for location permission. Denied: "Allow location access to continue: your KYC must be done from within India."
   If Android will no longer show the prompt, an alert offers "Open Settings".
3. If location services are off: "Turn on your phone's location, then try again."
4. Uses the last known position (up to 10 minutes old) or a fresh one.
5. `POST /onb/kyc/start {latitude, longitude}`. The backend creates (or reuses) the Cybrilla KYC form, fills in the
   personal details, and returns a DigiLocker `fetch_url` and a `return_url_prefix`.
6. If Aadhaar was already fetched for this form (`fetch_url` is null), it moves straight on with `goNext`.
   Otherwise it opens `/onboarding/kyc-web` with `step=proof`.

### The KYC WebView (`/onboarding/kyc-web`)

`kyc-web.tsx` hosts Cybrilla's pages for both DigiLocker (`step=proof`) and the e-sign (`step=esign`).

- Header: a close (X) button, a lock icon, and "Aadhaar via DigiLocker" or "E-sign your KYC".
- On a sandbox backend, a yellow bar explains the simulator (see [TESTING.md](TESTING.md)).
- When the WebView reaches the return URL (`<backend>/kyc/return...`), it stops loading it and calls
  `POST /onb/kyc/refresh`, which re-reads the form at Cybrilla.
  - If `next_step` has moved past the calling step (`aadhaar` or `kyc-esign`), `goNext`.
  - If not, it goes back to the calling screen with an error: the backend's error text, or "Aadhaar was not fetched from
    DigiLocker. Please try again." / "The e-sign was not completed. Please try again."
- Close or Android Back asks "Leave KYC? You can continue from this step later." Leaving returns to the calling screen.

### Bank, then signature

- Bank is the same screen and check as path A.
- `signature-consent` is titled "Submit your Signature". After the declarations are saved, the app pushes
  `/onboarding/signature` (it does not call `goNext`, because the user is not compliant).
- `signature.tsx`: draw on the pad, "Proceed" sends `POST /onb/signature {image}`. **For a fresh KYC the backend also
  uploads the signature to the KYC form** (`uploadKycSignature`).

**There is no selfie for a fresh KYC.** The DigiLocker Aadhaar OTP serves as the liveness check (backend comment), so
`nextStep()` skips `selfie-intro` when `kyc_fresh` is true.

### E-sign the KYC form (`/onboarding/kyc-esign`)

"Sign your KYC form": "Review and e-sign" calls `POST /onb/kyc/esign`.

- The backend waits briefly for Cybrilla to mark the form ready (`awaiting_esign`) after the signature arrives.
- If the form is incomplete: 400 "Your KYC form is not complete yet (missing: ...)".
- If it is already signed (`awaiting_submission` or `submitted`): no URL, the app moves on with `goNext`.
- Otherwise it opens `/onboarding/kyc-web` with `step=esign` (the e-sign page asks for an Aadhaar OTP). After signing,
  Cybrilla submits the form to the KRA.

### Nominee, eSign, done

- Nominee is the same as path A. In the fresh path it comes after the KYC e-sign.
- `esign.tsx` is titled "Verify details and eSign" and shows the "CKYC & KRA KYC Form" layout: application type New,
  PAN (masked), proof of identity/address "E - Aadhaar" with the masked Aadhaar, the signature image, bank and nominee.
  "Sign Document", tick "consent to eSign this form", "Agree & eSign" calls the same `POST /onb/esign`.
- `done.tsx` shows, when `kyc_status === 'with_kra'`: "Your KYC is with the KRA for approval. It usually takes 1-3
  working days, and we'll let you know as soon as you can invest. No further action needed." Otherwise it shows the
  generic "Your document verification is in process..." line. (`with_kra` means the form status is `submitted`; while
  it is still `awaiting_submission` the status is `in_progress` and the generic line shows.)

### After onboarding: waiting for the KRA

`kyc_status` on the user (`backend-ts/src/services/user.service.ts` `kycStatus`):

| Value | Meaning |
|---|---|
| `compliant` | The KRA shows a valid KYC (always the case on path A) |
| `in_progress` | Fresh KYC being filled in, or signed but not yet submitted |
| `with_kra` | Submitted, waiting for the KRA (usually 1-3 working days) |
| `failed` | The form failed or expired: redo it |
| `null` | Before the PAN step |

While `kyc_status === 'with_kra'`:

- **Home** shows the banner "Your KYC is with the KRA. It usually takes 1-3 working days. You can invest as soon as it's
  approved." Tapping it opens Profile > Account details. Each time Home gains focus it calls `GET /onb/kyc/status` and
  updates the user (the backend throttles the KRA check to once per 10 minutes per investor). The backend also checks
  every 30 minutes on its own (`kyc-approvals` job).
- **Orders are refused** by the backend: 403 "Your KYC is with the KRA for approval (usually 1-3 working days). You can
  invest once it is approved." (`requireInvestor` in `orders.service.ts`).
- **Profile > Account details** shows "KYC status: With the KRA for approval (1-3 working days)", or "Verified with the
  KRA" once compliant. Note: this row only shows when the user has no `aadhaar_masked`. When DigiLocker returned an
  Aadhaar number, the backend stores its last 4 digits and the screen shows the "Aadhaar" row instead of the KYC
  status row.
- **Notifications** carry "Your KYC is with the KRA" and, once approved, "Your KYC is approved" (links to Explore).

Once the KRA approves, readiness turns `verified`, `kyc_compliant` becomes true, `kyc_status` becomes `compliant`, the
banner disappears and orders are accepted.

Known gap (from the project notes, not fixed in the code read here): a KRA rejection is never surfaced. The backend only
acts on `verified`, so a rejected investor would keep seeing "with the KRA".

## PAN privacy: the full PAN is never sent back

The backend never returns the full PAN (`pan_masked` on the user masks all but the last 3 characters; the checks it
returns mask FP's echo of the PAN too). The app handles this as follows:

| Screen | PAN shown | Where it comes from |
|---|---|---|
| PAN confirm | The PAN the investor just typed | Route param `pan`, passed by `pan.tsx` (falls back to `user.pan_masked`) |
| eSign form | Masked | `user.pan_masked` (also "KRA · <masked>" as the KYC number on the fresh form) |
| Profile > Account details | Masked | `user.pan_masked` |

**"No, edit PAN"** on PAN confirm:

1. Collects what was typed: `pan` from the route param, `name` and `dob` from the current user.
2. Calls `POST /onb/pan/reset`, which clears the PAN, name, DOB, the checks, the fresh-KYC flag, any KYC form and the
   Aadhaar flag on the server. (Refused with 409 once the PAN is confirmed.)
3. Replaces the screen with `/onboarding/pan`, passing the typed `pan`, `name` and `dob` as params, so the fields come
   back filled in.

Submitting a different PAN also drops a KYC form started for the previous PAN.

## Legacy screens

These still exist but a new investor does not reach them on the current backend:

| Screen | Why it's unused |
|---|---|
| `aadhaar.tsx` (Aadhaar number form), `aadhaar-otp.tsx`, `aadhaar-confirm.tsx` | The old local Aadhaar OTP stand-in. Only for users who are neither compliant nor fresh KYC, and the PAN step now refuses those. The backend calls it "unused outside old sandbox accounts" and refuses it for fresh-KYC users ("Verify your Aadhaar with DigiLocker.") |
| `selfie-intro.tsx`, `selfie.tsx` | `nextStep()` only asks for a selfie when the user is neither compliant nor fresh KYC (same old accounts) |

## Rules worth testing

- Onboarding answers can't be changed after eSign: the backend answers 409 "Your account is already set up. Contact
  support to change these details." for `/onb/*` writes.
- `POST /onb/esign` is idempotent: a double tap or a retry resumes from what already exists at FP. If a step is missing
  it answers "Please complete the <step> step first." with `next_step`.
- Opening an investment from a fund page before onboarding is finished sends the user to `/onboarding/<next_step>`.
- The PIN is required on every cold start once set (`/unlock`); the app does not re-lock from the background.
- A fresh KYC needs the phone's location once; the KYC must be done from within India.

## Things not verified in this document

- The Cybrilla DigiLocker and e-sign pages themselves (their wording and steps) are third-party pages; the hints in
  the app describe the sandbox simulator only.
- The final KRA approval: sandbox PANs with `3753` never become compliant, so "with the KRA" → "approved" can't be
  seen in the sandbox. (The hosted sandbox runs `backend-ts` since 2026-11-02; path B was verified there that day.)
