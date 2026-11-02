# Risips API reference

Every endpoint of `backend-ts`, verified against the code at commit `428814b` (2026-11-02, branch `dev`).
Routes live in `src/routes/*.ts`, validation in `src/validation/*.ts`, behaviour in `src/services/*.ts`.

Sandbox base URL: `https://risips-demo-api.azurewebsites.net` (demo mode on: OTP responses include `dev_otp`).

## Contents

- [Conventions](#conventions): auth, headers, error envelope, rate limits, the user object
- [System](#system)
- [Auth](#auth)
- [OTP (transaction 2FA)](#otp-transaction-2fa)
- [Onboarding](#onboarding): step order, PAN, profile, address, Aadhaar, fresh KYC, bank, selfie, signature,
  declarations, nominee, eSign
- [Funds, watchlist, portfolio](#funds-watchlist-portfolio)
- [Cart and checkout](#cart-and-checkout)
- [Orders (lumpsum, redeem, switch)](#orders-lumpsum-redeem-switch)
- [SIP and mandates](#sip-and-mandates)
- [Plans (STP, SWP)](#plans-stp-swp)
- [Reports](#reports)
- [Settings](#settings)
- [Notifications](#notifications)
- [External funds (MF Central import)](#external-funds-mf-central-import)
- [Assistant](#assistant)
- [Webhooks](#webhooks)
- [Admin API](#admin-api-adminv1)

---

## Conventions

### Authentication

| Label used below | Meaning |
|---|---|
| **none** | No session needed |
| **user** | `Authorization: Bearer <token>` from `POST /auth/email/verify` (`requireAuth`). Missing, malformed, unknown or expired token: `401 { "error": "Please sign in again." }` |
| **admin key** | `x-admin-key: <ADMIN_API_KEY>` (see [Admin API](#admin-api-adminv1)) |

The app also sends `x-device-name` and `x-device-os` on every request; they are stored on the session and shown
in Settings > Active devices. All bodies are JSON (`Content-Type: application/json`).

### Request ids

Every response carries `X-Request-Id`. Send your own `X-Request-Id` (8-64 characters of `A-Z a-z 0-9 _ -`) to have it
used instead; quote it when reporting a bug.

### Error envelope

`middleware/errorHandler.ts` gives every error one shape:

```json
{ "error": "Message to show the investor", "reason": "machine_code", "...": "extra fields" }
```

- `error` is always present. `reason` and other fields appear only where noted.
- **Validation errors** (`middleware/validate.ts`): `400` with the first problem as `error`, `reason: "validation"`, and
  every problem under `fields` (key = field path, `_` for the whole body):

  ```json
  { "error": "Enter a valid email address.", "reason": "validation", "fields": { "email": "Enter a valid email address." } }
  ```

  A bad **path parameter** (a malformed id in the URL) is a `404` with the same shape, except the pincode and IFSC
  lookups, which answer `400`. Unknown body fields are dropped silently.
- **FP errors**: `502 { error }` with FP's message made readable, or `503 { "error": "Can’t reach the fund platform
  right now. Please try again in a minute." }` when FP did not answer at all. A malformed FP reply gives
  `"The fund platform sent an unexpected reply. Please check Orders before trying again."`.
- Malformed JSON: `400 "The request body is not valid JSON."`. Body too large: `413 "The request is too large."`.
- Unknown route: `404 "Not found"`. Anything unexpected: `500 "Something went wrong. Please try again."`.

### Common reason codes

| `reason` | Status | Meaning |
|---|---|---|
| `validation` | 400 / 404 | Input failed the zod schema; see `fields` |
| `rate_limited` | 429 | Too many requests or too many wrong codes. OTP-service blocks add `retry_after` (seconds); middleware limits put the wait in the `RateLimit` header |
| `otp_required` | 400 | A transaction needs the 2FA code: call `POST /txn/otp`, then resend with `otp` |
| `wrong` | 400 | Wrong OTP; `attempts_left` says how many tries this code has |
| `expired` | 400 | OTP expired |
| `missing` | 400 | No open OTP (never sent, already used, or replaced) |
| `locked` | 400 | 5 wrong tries on this code; request a new one |
| `mobile_taken` | 409 | The mobile is verified on another live account |
| `pin_locked` | 401 | 5 wrong PINs; the session was ended |
| `wrong_pin` | 400 | Wrong current PIN in Settings |

Other extra fields used: `attempts_left`, `retry_after`, `next_step`, `need_mandate`, `limit`, `need`, `checks`,
`pv_id`, `bank_status`, `code`, `kyc_status`, `cart_id`, `order`, `sip`.

### Rate limits

Middleware limits (`middleware/rateLimit.ts`), all in memory, multiplied by `RATE_LIMIT_SCALE`. Over the limit:
`429 { "error": "Too many attempts. Please wait a few minutes and try again.", "reason": "rate_limited" }`.

| Name | Limit | Keyed by | Used on |
|---|---|---|---|
| global | 300 / min | IP | Every route except `/health`, `/fp/webhook/*`, `/admin/v1/*` |
| otpSendIp | 20 / h | IP | `POST /auth/email/start` |
| otpSendEmail | 20 / h | email in body | `POST /auth/email/start` |
| otpSendUser | 60 / h | user | `POST /auth/mobile/start`, `POST /txn/otp`, `POST /swp/otp`, `POST /onb/aadhaar/start`, `POST /external/start` |
| otpVerify | 60 / h | IP | `POST /auth/email/verify`, `POST /mfc-sandbox/api/verify` |
| otpVerifyUser | 60 / h | user | `POST /auth/mobile/verify`, `POST /onb/aadhaar/verify` |
| pin | 10 / 15 min | user | `POST /auth/pin/set`, `POST /auth/pin/verify`, `POST /settings/pin/check`, `POST /settings/pin/change` |
| kycCheck | 30 / h | user | `POST /onb/pan`, `POST /onb/bank`, `POST /onb/kyc/start`, `POST /onb/kyc/refresh`, `POST /onb/kyc/esign`, `GET /onb/kyc/status` |
| money | 30 / 10 min | user | Order, plan, mandate, checkout, deletion and nominee-later writes (marked below) |
| webhook | 600 / min | IP | `POST /fp/webhook/:token` |
| admin | 600 / min | IP | `/admin/v1/*` |

On top of these, the OTP service limits each purpose + target (an email, a user's mobile, a user's 2FA action):

| Rule | Answer |
|---|---|
| One new code per 30 s, 20 per hour | `429 "A code was sent just now. Please try again in 30 seconds."`, `reason: "rate_limited"`, `retry_after` |
| 10 wrong codes per hour, 20 per day, counted across codes | `429 "Too many wrong codes. Please try again in N minutes."`, `reason: "rate_limited"`, `retry_after`. While blocked, **no new code is sent** either |
| 5 wrong tries on one code | `400`, `reason: "locked"` (a new code resets this lock, but not the cross-code cap) |

### The user object

Most auth and onboarding endpoints answer `{ "user": <User> }`, built by `publicUser()` in
`src/services/user.service.ts`:

| Field | Type | Notes |
|---|---|---|
| `id` | number | |
| `email` | string | |
| `mobile` | string \| null | 10 digits, set once verified |
| `name` | string \| null | As entered at the PAN step |
| `first_name` | string \| null | First word of `name` |
| `dob` | string \| null | `YYYY-MM-DD` |
| `gender` | string \| null | `male`, `female`, `transgender` |
| `pan_masked` | string \| null | Last 3 characters visible, e.g. `*******51Z`. **The full PAN is never returned** |
| `aadhaar_masked` | string \| null | `XXXX XXXX 1234` |
| `occupation`, `income_slab`, `marital_status`, `father_name` | string \| null | |
| `address` | object \| null | `{ line1, line2, pincode, city, state }` |
| `bank` | object \| null | `{ ifsc, bank_name, branch, city, pv_id, status, code, verified, account_masked }` (no `account_number`) |
| `nominee` | object \| null | `{ name, dob, relationship, pan }`, `pan` masked like `pan_masked` |
| `nominee_skipped` | boolean | Opted out of nomination |
| `nominee_added_at` | string \| null | Set when added after onboarding |
| `kyc_compliant` | boolean | Pre-verification readiness is `verified` (KRA record is good) |
| `kyc_fresh` | boolean | The investor is doing a fresh KYC (no KRA record) |
| `kyc_status` | string \| null | `compliant`, `in_progress`, `with_kra`, `failed`, or `null` before the PAN step (see below) |
| `selfie_url`, `signature_url` | string \| null | Relative signed links (`/files/...?exp=&sig=`), valid 1-2 h |
| `biometric` | boolean | |
| `pin_set` | boolean | |
| `onboarded` | boolean | FP investment account exists |
| `onboarded_at` | string \| null | ISO time |
| `member_since` | string | Year of sign-up |
| `next_step` | string | The onboarding screen to show next (see [step order](#step-order-and-next_step)) |
| `fp` | object \| null | `{ investor_profile, investment_account }` (FP ids) |

`kyc_status` (`kycStatus()`):

| Value | When |
|---|---|
| `compliant` | Readiness `verified` (a KYC-compliant PAN, or a fresh KYC the KRA has approved) |
| `in_progress` | Fresh KYC, form not yet submitted |
| `with_kra` | Fresh KYC form `submitted`; waiting for the KRA (usually 1-3 working days) |
| `failed` | Fresh KYC form `failed` or `expired`: redo it |
| `null` | Not a fresh KYC and not compliant (e.g. before the PAN step) |

---

## System

### GET /health
Auth none. No rate limit. Returns `{ "ok": true, "fp_env": "sandbox" | "production", "demo": boolean, "schemes": number }`.

### GET|POST /pay/return, /mandate/return, /kyc/return
Auth none. Any method. HTML page "Returning to Risips…". FP's checkout, mandate authorisation, and the fresh-KYC
DigiLocker / e-sign pages redirect here; the app's WebView watches for these URLs and closes itself. Query strings
(e.g. `?order=`, `?sip=`, `?mandate=`, `?step=proof|esign`) are informational.

### GET /files/:name?exp=&sig=
Auth none (the link is the credential). Selfie / signature images from the `selfie_url` / `signature_url` on the user.
Missing `sig`, bad signature, expired, or unknown file: `404` with an empty body.

### GET /debug/trace?since=
Auth user. Demo mode only (else `404`). Returns `{ entries: [...last 150 FP calls with seq > since], log_file }`. The
buffer is shared by every user.

### POST /fp/webhook/:token
See [Webhooks](#webhooks).

---

## Auth

### POST /auth/email/start
Auth none. Rate limits otpSendIp + otpSendEmail + OTP-service rules.

| Field | Type | Rules / message |
|---|---|---|
| `email` | string | Trimmed, lower-cased, max 254, must end in `.com .in .org .co .net .edu .me .uk`. "Enter a valid email address." |

Sends a 6-digit sign-in code, valid 120 s. Sign-up and sign-in are the same call.

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "123456" }
```

`existing` = an account with this email already exists. `dev_otp` only in demo mode.
Errors: 429 OTP rules; 503 "We couldn't send the code. Please try again in a minute." when every MSG91 channel failed.

### POST /auth/email/verify
Auth none. Rate limit otpVerify.

| Field | Type | Rules / message |
|---|---|---|
| `email` | string | As above |
| `code` | string \| number | 4-8 digits. "Enter the code we sent you." |

Creates the account if new, signs in, and cancels a pending account deletion.

```json
{ "token": "<32 chars>", "is_new": true, "user": { "...": "User" }, "deletion_cancelled": true }
```

`deletion_cancelled` only when a deletion request was cancelled.

| Error | When |
|---|---|
| 400 `reason: "wrong"`, `attempts_left` | "Incorrect code. 4 attempts left." (last one: "Incorrect code. No attempts left: request a new code.") |
| 400 `reason: "expired"` | "Code expired. Tap Resend Code." |
| 400 `reason: "missing"` | "Please request a new code." |
| 400 `reason: "locked"` | "Too many attempts. Request a new code." |
| 429 `reason: "rate_limited"`, `retry_after` | "Too many wrong codes. Please try again in N minutes." |
| 409 | "Please request a new code and sign in again." (account erased in the same moment) |

### POST /auth/mobile/start
Auth user. Rate limit otpSendUser + OTP-service rules.

| Field | Type | Rules / message |
|---|---|---|
| `mobile` | string \| number | Non-digits removed, last 10 digits kept, so `+91 98765 01234`, `09876501234` and `9876501234` are the same. Must start 6-9 and not be one digit repeated. "Enter a valid 10-digit mobile number." |

Returns `{ "ok": true, "expires_in": 120, "code_length": 6, "dev_otp": "..." }`. The code is tied to this user and number.

| Error | When |
|---|---|
| 409 `reason: "mobile_taken"` | "This mobile number is already registered with another Risips account. Please use a different number." (checked before any code is sent; erased accounts don't count) |
| 409 | "Your mobile number is already registered. Contact support to change it." (onboarded, different number) |
| 429, 503 | As for email start |

### POST /auth/mobile/verify
Auth user. Rate limit otpVerifyUser.

| Field | Type | Rules |
|---|---|---|
| `mobile` | string \| number | As above (must be the number the code was sent to) |
| `code` | string \| number | 4-8 digits |
| `bank_consent` | boolean | Optional, default `false` |

Returns `{ user }` with `mobile` set. Errors: same OTP errors as email verify; `409 mobile_taken` is checked again
(another account may have verified the number since the code was sent).

### POST /auth/pin/set
Auth user. Rate limit pin. Body `{ "pin": "1234" }` (exactly 4 digits, "PIN must be 4 digits."). Returns `{ user }`.

### POST /auth/pin/verify
Auth user. Rate limit pin. Body `{ "pin": "1234" }`. Returns `{ "ok": true }`.

| Error | When |
|---|---|
| 400 | "No PIN set." |
| 400 `attempts_left` | "Incorrect PIN. 3 attempts left." |
| 401 `reason: "pin_locked"` | 5th wrong PIN in a row: "Too many wrong PIN attempts. Please sign in again with your email." The session is ended |

### POST /auth/biometric
Auth user. Body `{ "enabled": true }` (boolean, "Must be true or false."). Returns `{ user }`.

### POST /auth/logout
Auth user. Ends this session. Returns `{ "ok": true }`.

### GET /me
Auth user. Returns `{ user }`.

---

## OTP (transaction 2FA)

Every transaction (and the nomination / opt-out, and account deletion) needs a one-time code sent to the registered
mobile and email (SEBI 2FA). Pattern:

1. `POST /txn/otp { "action": "<key>" }`
2. Send the transaction with `"otp": "<code>"`.

If `otp` is missing the transaction answers `400 reason: "otp_required"` *after* all other validation, so a bad amount
never uses up a code. The code is checked right before the first FP write and is single-use.

### POST /txn/otp
Auth user. Rate limit otpSendUser + OTP-service rules.

| Field | Type | Rules |
|---|---|---|
| `action` | string | One of the keys below; else "Unknown action." |

| `action` | Authorises | Used by |
|---|---|---|
| `lumpsum` | a one-time investment | `POST /invest/lumpsum` |
| `cart` | your cart checkout | `POST /checkouts` |
| `redeem` | a redemption | `POST /invest/redeem` |
| `switch` | a switch | `POST /invest/switch` |
| `sip` | a new SIP | `POST /sip` |
| `sip_modify` | a change to your SIP | `POST /sips/:id/modify` |
| `sip_cancel` | stopping your SIP | `POST /sips/:id/cancel` |
| `sip_pause` | pausing your SIP | `POST /sips/:id/pause` |
| `sip_resume` | resuming your SIP | `POST /sips/:id/resume` |
| `stp` | a new STP | `POST /stp` |
| `stp_cancel` | stopping your STP | `POST /stps/:id/cancel` |
| `swp` | a new SWP | `POST /swp` |
| `swp_cancel` | stopping your SWP | `POST /swps/:id/cancel` |
| `nominee` | your nomination | `POST /onb/nominee`, `POST /profile/nominee` |
| `delete_account` | deleting your Risips account | `POST /settings/deletion` |

Code: 6 digits, valid 300 s, one per action (a new one replaces the open one for that action).

```json
{
  "ok": true, "action": "lumpsum", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 98••••234", "email": "a••••@example.com" },
  "dev_otp": "123456"
}
```

`sent_to` lists only channels that didn't fail. Errors: `400 "Please verify your mobile number first."`; 429 OTP rules;
503 when every channel failed.

### Errors when a transaction checks its `otp`

| Status, `reason` | `error` |
|---|---|
| 400 `otp_required` | "Confirm this with the OTP sent to your registered mobile and email." |
| 400 `wrong`, `attempts_left` | "That OTP is not correct. 4 attempts left." / "That OTP is not correct. No attempts left: request a new code." |
| 400 `locked` | "Too many wrong attempts. Request a new OTP." |
| 400 `expired` | "This OTP has expired. Request a new one." |
| 400 `missing` | "This OTP has already been used or expired. Request a new one." |
| 429 `rate_limited`, `retry_after` | "Too many wrong codes. Please try again in N minutes." |

---

## Onboarding

All `/onb/*` routes need auth **user**, and send `Cache-Control: no-store`.

**Once onboarded** (`onboarded_at` set), every `POST /onb/*` except `POST /onb/esign` answers
`409 "Your account is already set up. Contact support to change these details."`. GET routes keep working.

### Step order and `next_step`

`nextStep()` in `user.service.ts` returns the first unmet step:

| # | `next_step` | Done when | Endpoint |
|---|---|---|---|
| 1 | `mobile` | mobile verified | `/auth/mobile/start` + `/verify` |
| 2 | `pin` | PIN set | `/auth/pin/set` |
| 3 | `pan` | PAN verified | `POST /onb/pan` |
| 4 | `pan-confirm` | PAN confirmed | `POST /onb/pan/confirm` |
| 5 | `occupation` | occupation set | `POST /onb/profile` |
| 6 | `income` | income slab set | `POST /onb/profile` |
| 7 | `marital` | marital status set | `POST /onb/profile` |
| 8 | `father` | father's / spouse's name set | `POST /onb/profile` |
| 9 | `address` | address **and** gender set | `POST /onb/address` |
| 10 | `aadhaar` | **not KYC-compliant only**: Aadhaar verified | Fresh KYC: `POST /onb/kyc/start` (DigiLocker) then `/onb/kyc/refresh` |
| 11 | `bank` | bank verified | `POST /onb/bank` |
| 12 | `selfie-intro` | **only if not compliant and not fresh KYC** (old sandbox accounts): selfie saved | `POST /onb/selfie` |
| 13 | `signature-consent` | FATCA declared, and (if not compliant) signature saved | `POST /onb/declarations`, `POST /onb/signature` |
| 14 | `kyc-esign` | **fresh KYC only**: KYC form `awaiting_submission` or `submitted` | `POST /onb/kyc/esign`, then `/onb/kyc/refresh` |
| 15 | `nominee` | nomination or opt-out confirmed with OTP | `POST /onb/nominee` |
| 16 | `esign` | FP investor + account created | `POST /onb/esign` |
| 17 | `home` | onboarded | |

So a **KYC-compliant** investor goes `address -> bank -> signature-consent (FATCA only) -> nominee -> esign`, and a
**fresh-KYC** investor goes `address -> aadhaar (DigiLocker) -> bank -> signature-consent (FATCA + signature) ->
kyc-esign -> nominee -> esign`. There is **no selfie** for fresh KYC (the DigiLocker Aadhaar OTP is the liveness
check). Once `onboarded_at` is set, `next_step` is always `home`.

### POST /onb/pan
Rate limit kycCheck. Verifies PAN, name and date of birth with FP pre-verification (polls up to 25 s), plus the PAN's
KYC readiness at the KRA.

| Field | Type | Rules / message |
|---|---|---|
| `pan` | string | Upper-cased. Individual PAN: `^[A-Z]{3}P[A-Z][0-9]{4}[A-Z]$` (4th character `P`). "Enter a valid individual PAN (e.g. ABCPX1234K)." |
| `name` | string | Whitespace collapsed; `^[A-Za-z][A-Za-z .']{2,69}$`. "Enter your full name as on PAN (letters only)." |
| `dob` | string | `YYYY-MM-DD`, a real past date after 1900. "Enter a valid date of birth (YYYY-MM-DD)." |
| `fresh_kyc` | boolean | Optional, default `false`. Send `true` from app builds that have the fresh-KYC screens |

**`fresh_kyc` behaviour:**

- Without it (or `false`), a PAN with **no KRA record** (readiness code `kyc_unavailable`) answers
  `422 "No KYC record found for this PAN with the KRA. A fresh KYC is needed."`.
- With `fresh_kyc: true`, the same PAN answers **200** with `kyc_fresh: true`, provided PAN, name and DOB all
  verified and readiness is the only failure. The investor continues with fresh KYC.
- Every other readiness failure still answers 422, flag or not.

Success:

```json
{
  "user": { "...": "User", "next_step": "pan-confirm", "kyc_status": "in_progress", "kyc_fresh": true },
  "checks": {
    "readiness": { "status": "failed", "code": "kyc_unavailable" },
    "pan": { "status": "verified", "value": "*******53K" },
    "name": { "status": "verified" },
    "date_of_birth": { "status": "verified" }
  },
  "pv_id": "<pre-verification id>",
  "kyc_fresh": true
}
```

`checks` holds FP's result objects (other fields FP sends pass through). `checks.pan.value` is always masked.

A different PAN from the one already stored drops any KYC form started for the old one.

Errors (422 carries `{ checks, pv_id }`, masked the same way):

| Status | `error` | When |
|---|---|---|
| 409 | "Your PAN is already confirmed and can not be edited." | after `/onb/pan/confirm` |
| 400 | "You must be 18 or older to open an account. Investing for a minor is not supported yet." | |
| 422 | "No KYC record found for this PAN with the KRA. A fresh KYC is needed." | readiness `kyc_unavailable` without `fresh_kyc` |
| 422 | "Your KYC was rejected by the KRA. A fresh KYC is needed." | `kyc_rejected` |
| 422 | "Your KYC is incomplete at the KRA and needs a modification." | `kyc_incomplete` |
| 422 | "Your KYC is on hold at the KRA and needs a modification." | `kyc_onhold` |
| 422 | "Your KYC is a legacy record and needs a modification." | `kyc_legacy` |
| 422 | "Your KYC is still being processed by the KRA. Please try again in a few days." | `kyc_underprocess` |
| 422 | "Your KYC is deactivated. You cannot invest right now." | `kyc_deactivated` |
| 422 | "The KRA service is unavailable. Please try again." | `upstream_error` on any check |
| 422 | "Your PAN is not linked with Aadhaar. Link it on the income-tax portal and try again." | pan `aadhaar_not_linked` |
| 422 | "This PAN is invalid. Please check and re-enter." | pan check failed |
| 422 | "The name does not match PAN records. Enter it exactly as on your PAN card." | name check failed |
| 422 | "The date of birth does not match PAN records." | DOB check failed |
| 422 | "We could not verify your PAN right now. Please try again." | pre-verification not completed in time, or another readiness code |

The `READINESS_MSG` table in `onboarding.service.ts` is the source for the readiness messages above.

### POST /onb/pan/confirm
No body. Returns `{ user }` with `next_step: "occupation"`. `400 "Verify your PAN first."` if not verified.

### POST /onb/pan/reset
No body. "No, edit PAN": clears `pan`, `name`, `dob`, verification and confirmation flags, the stored PAN checks,
`kyc_fresh`, any KYC form and the Aadhaar-verified flag. Returns `{ user }` (`next_step: "pan"`).
`409 "Your PAN is already confirmed and can not be edited."` after confirm.

### POST /onb/profile
Any subset of these fields (each optional). Returns `{ user }`.

| Field | Allowed values / message |
|---|---|
| `occupation` | `private_sector_service`, `public_sector_service`, `government_service`, `house_wife`, `student`, `business`, `professional`, `retired`, `agriculture`, `service`, `others`. "Invalid occupation." |
| `income_slab` | `upto_1lakh`, `above_1lakh_upto_5lakh`, `above_5lakh_upto_10lakh`, `above_10lakh_upto_25lakh`, `above_25lakh_upto_1cr`, `above_1cr`. "Invalid income slab." |
| `marital_status` | `single`, `married`, `others`. "Invalid marital status." |
| `gender` | `male`, `female`, `transgender`. "Invalid gender." |
| `father_name` | 3-70 of letters, space, `.`, `'`. "Enter your father's or spouse's full name." |

### GET /onb/pincode/:pin
Returns `{ "city": "...", "state": "..." }` from FP's pincode master.

- Bad format: `400 "Enter a 6-digit pincode."` (reason `validation`).
- Unknown, **or** FP's state doesn't fit the pincode's postal region (first two digits; `utils/pincode.ts`):
  `404 "Pincode not found."`. This rejects FP's junk rows such as `999999` (FP says Nizamabad) and `123456`.

### POST /onb/address

| Field | Type | Rules / message |
|---|---|---|
| `line1` | string | 3-120 chars. "Enter your house / street address." / "Address line is too long." |
| `line2` | string | Optional, max 120 |
| `pincode` | string | 6 digits. "Enter a 6-digit pincode." |
| `gender` | string | `male`, `female`, `transgender`. "Select your gender." |

City and state come from the pincode lookup. Returns `{ user }`. Unknown / junk pincode: `400 "Pincode not found."`.

### POST /onb/aadhaar/start, POST /onb/aadhaar/verify (legacy)
A simulated Aadhaar OTP kept for old sandbox accounts (only KYC-compliant or fresh-KYC investors get past the PAN step,
and neither uses it).

- start: rate limit otpSendUser; body `{ "aadhaar": "2345 6789 0123" }` (12 digits, first 2-9, "Enter your 12-digit
  Aadhaar number."). Returns `{ ok, masked: "XXXX XXXX 0123", dev_otp? }`. For a fresh-KYC investor:
  `409 "Verify your Aadhaar with DigiLocker."` with `next_step: "aadhaar"`.
- verify: rate limit otpVerifyUser; body `{ "code": "123456" }`. Returns `{ user }`. Errors: 400 "Incorrect OTP. Please
  try again." / "Too many attempts. Please request a new OTP." / "OTP expired. Please request a new one." (with
  `reason`); 429 OTP rules.

### Fresh KYC: /onb/kyc/*

For a PAN with no KRA record (`kyc_fresh: true`). Uses Cybrilla POA's KYC Forms API (same host and token as
pre-verification). Sequence:

1. `POST /onb/kyc/start` with the phone's location: creates (or reuses) the KYC form, fills it with the onboarding
   answers, returns a DigiLocker `fetch_url`.
2. The app opens `fetch_url` in a WebView. DigiLocker redirects to `/kyc/return?step=proof`; the app closes it and
   calls `POST /onb/kyc/refresh`. When the Aadhaar proof is `fetched`, `aadhaar_verified` becomes true (and
   `aadhaar_masked` is filled), so `next_step` moves to `bank`.
3. `POST /onb/signature` saves the signature and **forwards it to the KYC form**.
4. `POST /onb/kyc/esign` returns the e-sign `esign_url`. After e-signing, the page redirects to `/kyc/return?step=esign`;
   the app calls `POST /onb/kyc/refresh`. Once the form is `awaiting_submission` or `submitted`, `next_step` moves on
   (to `nominee`). Cybrilla then submits the form to the KRA.
5. The investor finishes onboarding (`nominee`, `esign`) and the FP account is created, but **orders are blocked with
   `403`** until the KRA approves (usually 1-3 working days). The `kyc-approvals` job re-checks every 30 min (at most
   once per 10 min per investor); `GET /onb/kyc/status` also checks.

`kyc` object (`KycFormState`) returned by these routes:

| Field | Notes |
|---|---|
| `id` | KYC form id |
| `status` | `under_review`, `created`, `awaiting_esign`, `awaiting_submission`, `submitted`, `failed`, `expired` |
| `reason` | Cybrilla's reason when failed |
| `proof_status` | DigiLocker Aadhaar fetch: e.g. `pending`, `fetched`, `failed` |
| `fields_needed` | What the form still needs (e.g. `signature`) |
| `signature_provided` | boolean |
| `esign_status`, `expires_at`, `submitted_at` | |

A failed or expired form is redone from DigiLocker on a new form (`aadhaar_verified` goes back to false); the
signature on file is re-sent automatically.

#### POST /onb/kyc/start
Rate limit kycCheck.

| Field | Type | Rules / message |
|---|---|---|
| `latitude` | number | 6 to 37.5. Missing: "Allow location access to continue." Out of range: "Your KYC must be done from within India." |
| `longitude` | number | 68 to 97.5. Same messages |

Returns `{ user, fetch_url, return_url_prefix }` (`return_url_prefix` = `<PUBLIC_URL>/kyc/return`), or
`{ user, fetch_url: null }` when the Aadhaar proof is already fetched.

| Status | `error` |
|---|---|
| 409 | "Your KYC is already on record with the KRA." (not a fresh-KYC investor) |
| 400 | "Please complete your personal details first." (PAN not confirmed, or address / gender / father's name / occupation / income / marital status missing) |
| 503 | "Your KYC form is still being checked. Please try again in a minute." |
| 422 `{ kyc_status, reason }` | "Your KYC form expired before it was finished. Please start again." / "A fresh KYC can’t be submitted for this PAN. Please contact support." (`ineligible_for_fresh_kyc`) / "Your KYC form was not accepted (…). Please check your details and start again." Note: here `reason` is Cybrilla's form reason, not one of our reason codes |
| 502 | "DigiLocker is not available right now. Please try again." |

#### POST /onb/kyc/refresh
Rate limit kycCheck. No body. Re-reads the form (after DigiLocker or e-sign), uploads the saved signature if the form
still needs it. Returns `{ user, kyc, error }` (`error` = failure text or `null`).
Errors: 409 not fresh KYC (as above); `400 "Start your KYC from the Aadhaar step first."`.

#### POST /onb/kyc/esign
Rate limit kycCheck. No body. Returns `{ user, esign_url, return_url_prefix }`, or `{ user, esign_url: null }` when the
form is already `awaiting_submission` / `submitted`. Waits up to 10 s for Cybrilla to move a just-completed form to
`awaiting_esign`.

| Status | `error` |
|---|---|
| 422 `{ kyc_status }` | Form failed / expired (texts as above) |
| 400 `{ next_step }` | "Your KYC form is not complete yet (missing: signature)." |
| 502 | "The e-sign page is not available right now. Please try again." |

#### GET /onb/kyc/status
Rate limit kycCheck. Works after onboarding too (Home banner, KYC status screen). Refreshes a live form, and for a
`submitted` form asks the KRA (throttled to once per 10 min). Returns `{ user, kyc, error }` (`kyc` is `null` if no form).

### GET /onb/ifsc/:ifsc
Returns `{ ifsc, bank_name, branch, city }`. Bad format: `400 "Enter a valid IFSC."` (`^[A-Z]{4}0[A-Z0-9]{6}$`, upper-cased).
Unknown: `404 "IFSC not found."`.

### POST /onb/bank
Rate limit kycCheck. Verifies the account with FP pre-verification (with the investor's PAN, name and DOB; polls up to
20 s).

| Field | Type | Rules / message |
|---|---|---|
| `account_number` | string \| number | Spaces removed; 9-18 digits. "Account number must be 9–18 digits." |
| `ifsc` | string | As above |

Returns `{ user }` with `bank.verified: true`. Errors (`422` carries `{ bank_status, code, pv_id }`):

| Status | `error` | When |
|---|---|---|
| 400 | "IFSC not found." | |
| 422 | "Bank verification is taking longer than usual. Please try again, or use another account." | no result in 20 s |
| 422 | "We could not confirm this account belongs to you. Please use another account in your name." | `code` `low_confidence` or `uncertain` |
| 422 | "This account has had too many verification attempts and can’t be verified right now. Please use another account in your name." | `code` `verification_attempt_limit_exceeded` (FP blocks an account number for everyone after 3 failed checks) |
| 422 | "Bank verification failed. Please check the account number and IFSC." | any other failure |

### POST /onb/selfie, POST /onb/signature
Body up to 12 MB.

| Field | Type | Rules / message |
|---|---|---|
| `image` | string | `data:image/png;base64,…` or `data:image/jpeg;base64,…`. "Invalid image." The decoded bytes must be 500 B to 8 MB ("Image is too small or too large.") and really PNG / JPEG ("Invalid image.") |

Returns `{ user }` (with `selfie_url` / `signature_url`). For a fresh-KYC investor with a live form, the signature is
also uploaded to the KYC form (an FP error there surfaces as 502).

### POST /onb/declarations

| Field | Type | Rules / message |
|---|---|---|
| `fatca` | `true` | Must be `true`. "Please accept the FATCA / CRS declaration." |
| `running_account` | boolean | Optional, default `false` |

Returns `{ user }`.

### POST /onb/nominee
A nomination, or the opt-out, confirmed with a transaction OTP (`action: "nominee"`).

| Field | Type | Rules / message |
|---|---|---|
| `skip` | boolean | Optional. `true` = opt out (other fields ignored) |
| `name` | string | 3-40 letters and spaces. "Enter the nominee’s full name (letters only)." |
| `dob` | string | `YYYY-MM-DD`. "Enter a valid date of birth for the nominee (YYYY-MM-DD)." |
| `relationship` | string | `father`, `mother`, `spouse`, `son`, `daughter`, `brother`, `sister`, `grand_father`, `grand_mother`, `others`. "Select the relationship." |
| `pan` | string | Any PAN format `^[A-Z]{5}[0-9]{4}[A-Z]$`. "Enter the nominee’s PAN (required as identity proof)." |
| `otp` | string | Transaction OTP |

The field checks run in order and report the first failure. Returns `{ user }` (`nominee.pan` masked).

Errors: 400 "Nominee cannot be yourself." / "Minor nominees need guardian details, which this demo does not collect.
Please add an adult nominee or skip." / "Nominee PAN must be different from yours."; OTP errors.

### GET /onb/kyc-form
Returns `{ user, generated_at }` (data for the KYC form preview).

### POST /onb/esign
Body `{ "consent": true }` ("Please agree to eSign the form."). Allowed after onboarding (idempotent).
Creates the investor in FP: investor profile -> phone -> email -> address -> bank account -> nominee (if any) -> MF
investment account (folio defaults: contacts, payout bank, nominee or "show nomination status"). Each FP id is saved as
soon as it exists, so a retry resumes where it stopped.

Returns `{ user, fp }` where `fp` = `{ invp, phone, email, addr, bank, bank_old_id, relp?, mfia, mfia_old_id }`.
Already onboarded: the same shape at once.

Errors: `400 "Please complete the <step> step first."` with `next_step` (e.g. "Please complete the bank account step
first."); `400 "Please verify your mobile number first."`; FP errors 502/503.

---

## Funds, watchlist, portfolio

All need auth **user**.

### Fund summary object
Returned in lists (`summary()` in `catalog.service.ts`):

| Field | Notes |
|---|---|
| `isin`, `name`, `short_name`, `amc`, `amc_short`, `amc_id` | |
| `category` | Broad category: `Equity`, `Debt`, `Hybrid`, `Index`, `Commodity` (gold / silver fund-of-funds), `Solution` |
| `sub_category` | e.g. "Mid Cap Fund" |
| `risk` | The fund house's SEBI riskometer level: `Low Risk`, `Low to Moderate Risk`, `Moderate Risk`, `Moderately High Risk`, `High Risk`, `Very High Risk`. Comes from `scripts/fund-data.ts` (Kuvera's public data); `catalog.json` records the source in `risk_source`, which the API does not return yet |
| `tags` | e.g. `mid`, `large`, `small`, `tax`, `debt`, `index`, `commodity` |
| `nav`, `nav_date` | Latest NAV: the newer of catalog.json and AMFI's daily file (`withLatestNav`) |
| `returns` | `{ 1D, 1M, 6M, 1Y, 3Y, 5Y }` in % (3Y/5Y annualised); `null` where unknown. Recomputed from AMFI when its NAV is newer |
| `lumpsum` | `{ allowed, min, additional_min, multiples, max }` |
| `sip_min` | Monthly SIP minimum, or `null` |
| `stp` | `{ frequency, min, multiples, installments_min, dates, in, out }` or `null` |
| `swp` | `{ frequency, min, multiples, installments_min, dates }` or `null` |
| `switch_in_min` | Minimum switch into this fund, or `null` |
| `top_performer` | Top quarter of its broad category by 3Y return (1Y for younger funds) |

NAVs: `catalog.json` is built on a fixed day; the `amfi-navs` job downloads AMFI's NAV file hourly when the copy is
over 4 h old, and a newer AMFI NAV replaces the stored one (with the 1D change, 1M-5Y returns, and a new chart point).

### GET /funds
Query (all optional, each a single value):

| Param | Meaning |
|---|---|
| `q` | Words that must all appear in name / AMC / category / sub-category (max 100) |
| `collection` | `high`, `commodity`, `mid`, `large`, `small`, `tax`, `debt`, `index`, `foryou`. Unknown: `404 "Unknown collection"` |
| `category` | Comma list, case-insensitive, e.g. `Equity,Hybrid` |
| `risk` | Comma list of risk labels, e.g. `Very High Risk` |
| `top` | Any value: top performers only |
| `sort` | `1M`, `6M`, `1Y`, `3Y` (default), `5Y`: by that return, descending |
| `picks` | Any value: "Our Picks" (with `collection`: its 6 best by 3Y return) |
| `amc_id` | 1-6 digits |
| `switch_in` | Any value: funds that accept switches in |
| `stp_in` | Any value: funds that accept STPs in |

A repeated parameter or over-long value: `400 "Invalid filter."`. Returns
`{ "title": "Mid Cap" | null, "funds": [ { ...summary, "watchlisted": boolean } ] }`.

### GET /funds/collections
Returns `{ collections: [{ key, title }] }` (every collection except `foryou`).

### GET /funds/compare/popular
Returns `{ pairs: [[summary, summary], ...] }`.

### GET /funds/compare?isins=A,B
`isins`: comma-separated, max 60 characters, 1 or 2 ISINs.

| Error | When |
|---|---|
| `400 "You can compare up to 2 funds."` (`reason: validation`) | a 3rd ISIN |
| `400 "Fund not found"` | empty list or a malformed ISIN |
| `404 "Fund not found"` | well-formed ISIN not in the catalogue |

Returns `{ funds: [...], unavailable: ["expense_ratio" | "holdings" | "managers"] }`. Each fund is the summary plus
`exit_load`, `lock_in`, `min_sip`, `history`, `fund_house`, `mandate` (SEBI category rule), `expense_ratio`,
`managers`, `objective`, `portfolio` (top 5 holdings / sectors / ratings), `pros`, `cons`.

### GET /funds/:isin?period=
`period`: `1M`, `6M`, `1Y` (default), `3Y`, `5Y`. Unknown ISIN: `404 "Fund not found"`.

Returns `{ fund, period, period_return, chart }`:

- `fund`: summary plus `lumpsum.min` (the additional-purchase minimum once the investor has a folio at that AMC),
  `lock_in`, `sip` (rules array), `redemption_allowed`, `switch_in_allowed`, `switch_out_allowed`, `stp`, `swp`,
  `watchlisted`, `exit_load`, `tax` (`{ stcg, ltcg }`), `expense_ratio` (`{ regular, direct, date, source }` or null),
  `managers`, `objective`, `portfolio` (all holdings), `peers` (`[{ name, r1, r3, r5 }]`), `source`.
- `chart`: `[[epoch_ms, nav], ...]` (about weekly).
- `period_return`: the catalogue's figure for the period, else computed from the chart.

### POST /watchlist/:isin
Toggles. Returns `{ watchlisted: boolean }`. `404 "Fund not found"`.

### GET /watchlist
Returns `{ funds: [ { ...summary, watchlisted: true } ] }`.

### GET /portfolio
`no-store`. Returns:

| Field | Notes |
|---|---|
| `summary` | `{ current, invested, gain, gain_pct, day_change, day_change_pct, xirr }` (Risips holdings; `xirr` only once the first investment is a year old) |
| `holdings` | Risips holdings: `{ isin, folio, fund, units, nav, nav_date, current, invested, redeemable, gain, gain_pct, avg_nav, xirr, day_change, locked, pending, sip, stp, swp }`. `locked` = a redemption / switch of all units is in progress |
| `in_progress` | Orders not yet processed (order view without `token_url`) |
| `incoming` | Funds on their way in that aren't holdings yet: `{ isin, fund, orders }` |
| `external` | Imported funds: `{ status, hidden, source, as_of, imported_at, error, holdings, summary }` |
| `combined` | Summary over Risips + external holdings (`xirr` null when external funds exist) |

---

## Cart and checkout

Auth **user**. A cart holds up to 10 lines; one checkout pays in at most two groups (all one-time lines on one UPI
checkout, all SIPs on another), because FP can't put lumpsums and SIP instalments on one payment.

Cart view (returned by every cart route):

```json
{
  "items": [ { "id": "ci_0a1b2c3d4e", "isin": "INF109KC1Q80", "mode": "sip", "amount": 1000,
               "added_at": "...", "fund": { "...": "summary" }, "problem": null } ],
  "sip_total": 1000, "lumpsum_total": 0, "pay_now": 1000, "sip_dates": [1, 2, 3, 28]
}
```

`problem` = why the line can't be bought now (scheme rules), else null. `pay_now` includes each SIP's first instalment.
`sip_dates` = SIP days every SIP fund in the cart accepts.

### GET /cart
Returns the cart view.

### POST /cart
Requires an onboarded, KYC-approved investor (403 otherwise, see [orders](#who-can-invest)).

| Field | Type | Rules / message |
|---|---|---|
| `isin` | string | `^IN[A-Z0-9]{10}$` (upper-cased). "Fund not found" |
| `mode` | string | `sip` or `lumpsum`. "Choose SIP or one-time." |
| `amount` | number \| numeric string | > 0, <= 1e9, rounded to the rupee. "Enter an amount." |

The same fund and mode again replaces the amount. Errors: 404 "Fund not found"; 400 scheme-rule message (e.g.
"Minimum investment in X is ₹5,000."); `400 "Your cart can hold up to 10 funds."`.

### PATCH /cart/:id
Body `{ mode?, amount? }`. Path id `ci_` + 10 hex ("This item is no longer in your cart."). Errors: 404 gone; 400 rule;
`409 "This fund is already in your cart with that plan type."`.

### DELETE /cart/:id
Returns the cart view.

### POST /checkouts
Rate limit money. Transaction OTP `action: "cart"` (one code for the whole checkout).

| Field | Type | Rules |
|---|---|---|
| `item_ids` | string[] | Optional, max 10; default = whole cart |
| `sip_day` | number | 1-28 ("Choose a date between 1 and 28."). Required when any SIP is in the selection and must be in `sip_dates` |
| `otp` | string | Transaction OTP |

Checks every line, the SIP date and AutoPay, then records the checkout. Nothing is placed yet. Returns `{ checkout }`:

```json
{ "checkout": { "id": "chk_0123456789abcdef", "sip_day": 5, "created_at": "...",
  "items": [ { "cart_id": "ci_…", "isin": "…", "fund": {}, "mode": "lumpsum", "amount": 5000,
               "state": "queued", "order_id": null, "error": null, "status": null } ],
  "total": 6000, "placed": 0, "failed": 0, "remaining": 2, "position": 0 } }
```

Errors: 403 not allowed to invest; `400 "Your cart is empty."`; `400 "Choose a SIP date that every fund in your cart
accepts."`; 400 a line's problem, with `cart_id`; `409` AutoPay problem with `need_mandate: true` and `need` (the monthly
total the mandate must cover); OTP errors.

### GET /checkouts/:id
Returns `{ checkout }` with a live `status` per line: `{ label, tone, detail?, ref?, sip_id? }`, e.g. "Order
successful", "Payment pending", "SIP active", "Not paid · still in cart". Path id `chk_` + 16 hex ("Checkout not
found").

### POST /checkouts/:id/next
Rate limit money. Places the next unpaid group and returns its one payment link:

```json
{ "checkout": {}, "order_id": "mfp_…", "order_ids": ["mfp_…", "mfp_…"], "mode": "lumpsum",
  "order": { "...": "order view or SIP view" }, "token_url": "https://…", "return_url_prefix": "<PUBLIC_URL>/pay/return" }
```

- Nothing left: `{ checkout, done: true }`.
- The group failed: `{ checkout, failed: { isin, isins, mode, error } }` (lines stay in the cart; the app moves on).
- A double tap while placing: `409 "Your previous order is still being placed. Please wait a moment."`.
- Checkout without OTP record: `409 "This checkout was not confirmed with an OTP. Please check out again."`.
- Lines removed or changed in the cart since the checkout started are not placed (`failed` / `skipped`).
- Placed lines leave the cart.

### POST /checkouts/:id/stop
Marks queued lines `skipped` (they stay in the cart). Returns `{ checkout }` with live statuses.

---

## Orders (lumpsum, redeem, switch)

Auth **user**.

### Who can invest
Every order, plan, mandate and cart write calls `requireInvestor()`:

| Status | `error` | When |
|---|---|---|
| 403 | "Please complete your KYC to invest." | not onboarded |
| 403 `{ kyc_status }` | "Your KYC is with the KRA for approval (usually 1–3 working days). You can invest once it is approved." | fresh KYC not yet approved by the KRA |

### Order view
Returned as `order` by the routes below:

| Field | Notes |
|---|---|
| `id`, `fp_id` | FP id (`mfp_…` purchase, `mfr_…` redemption, `mfs_…` switch) |
| `ref` | `#MF000123` from FP's old_id |
| `kind` | `purchase`, `redemption`, `switch` |
| `isin`, `fund`, `fund_to` | |
| `amount` | null for an all-units redemption / switch until priced |
| `units`, `units_in`, `nav` | |
| `state` | FP state: `under_review`, `pending`, `confirmed`, `submitted`, `successful`, `failed`, `cancelled`, `reversed` |
| `status` | Label: "Processing", "Awaiting payment", "Submitted to AMC", "Successful", "Failed", "Payment pending", "Payment failed", … |
| `failure_code` | FP's code |
| `folio` | Filled at allotment |
| `token_url` | Payment link while the purchase is unpaid, else null |
| `payment` | `{ id, status, reference }` |
| `created_at`, `updated_at`, `succeeded_at`, `traded_on` | |
| `all_units`, `estimated_amount`, `processed_amount`, `processed_units`, `fp_old_id` | |

### POST /invest/lumpsum
Rate limit money. OTP `action: "lumpsum"`.

| Field | Type | Rules / message |
|---|---|---|
| `isin` | string | "Fund not found" |
| `amount` | number \| numeric string | > 0. "Enter an amount." |
| `otp` | string | |

Checks scheme rules first, then the OTP, then: FP create (ONDC) -> wait for review (up to 45 s) -> consent -> UPI
checkout payment -> confirm. The folio already opened at that AMC is reused.

```json
{ "order": { "...": "order view", "state": "submitted", "status": "Payment pending" },
  "token_url": "https://…", "return_url_prefix": "<PUBLIC_URL>/pay/return" }
```

The investor must open `token_url` (single use) and pay. FP then redirects to `/pay/return?order=<id>`.

| Status | `error` |
|---|---|
| 404 | "Fund not found" |
| 400 | "<Fund> accepts SIP only. One-time investments are not open for it right now." |
| 400 | "Minimum investment in <Fund> is ₹5,000." (the additional-purchase minimum once a folio exists) |
| 400 | "Amount for <Fund> must be in multiples of ₹N." |
| 422 `{ order }` | "Order could not be placed. …" (review failed or slow) |
| 503 `{ order }` | "Order could not be placed. … Nothing was debited." (FP failed it right after confirm, e.g. ONDC outage) |

### POST /invest/redeem
Rate limit money. OTP `action: "redeem"`.

| Field | Type | Rules / message |
|---|---|---|
| `isin` | string | |
| `folio` | string | 1-40 of letters, digits, `/`, space, `-`. "Folio is required." |
| `all` | boolean | Optional. `true` = redeem every unit (no amount sent to FP) |
| `amount` | number | Required unless `all`. "Enter an amount." |
| `otp` | string | |

Returns `{ order }`. Errors: 404 fund; `400 "Minimum redemption is ₹100."`; `400 "There are no redeemable units of this
fund in this folio right now."`; `400 "You can redeem up to ₹N from this fund right now."`; `409 "A redemption of all
units in this fund is already in progress. …"`; 422 review failed / slow.

### POST /invest/switch
Rate limit money. OTP `action: "switch"`.

| Field | Type | Rules |
|---|---|---|
| `from_isin`, `to_isin` | string | |
| `folio` | string | |
| `all` | boolean | Optional |
| `amount` | number | Required unless `all` |
| `otp` | string | |

Errors: `400 "You can only switch between funds of the same fund house."`; `400 "<Target> takes switches of at least
₹N."` (target's switch-in minimum, ₹100 if unknown); `400 "You hold ₹N, less than the ₹M minimum switch into
<Target>."`; `400 "You can switch up to ₹N from this fund right now."`; others as redemption. Returns `{ order }`.

### GET /orders
Returns `{ orders: [order view] }` (purchases, redemptions, switches; newest first). Up to 8 open orders are
refreshed from FP first.

### GET /orders/:id?force=1
Path id `mfp_`, `mfr_` or `mfs_` + id ("Order not found", 404). Refreshes open orders (or always with `force`).
Returns `{ order }`. Someone else's order: `404 "Order not found"`.

---

## SIP and mandates

Auth **user**.

### GET /mandate
Returns `{ mandate, bank }`. `mandate` = `{ id, status, type, limit, umrn, valid_from, valid_to, created_at,
bank_account_id, auth_url? }` or null (refreshed from FP). `bank` = `{ bank_name, account_masked, ifsc, branch }`.

### POST /mandate
Rate limit money.

| Field | Type | Rules |
|---|---|---|
| `type` | string | `UPI` = UPI AutoPay; anything else (or nothing) = `E_MANDATE` (eNACH) |
| `limit` | number | Optional. Default 100,000, minimum 1,000; UPI capped at 100,000 |
| `need` | number | Optional: the monthly amount the mandate must cover |

Reuses an approved mandate whose limit covers `need`: `{ mandate, auth_url: null }`. Otherwise creates one and returns
`{ mandate, auth_url, return_url_prefix }` (`<PUBLIC_URL>/mandate/return`). The investor authorises it at `auth_url`.

### SIP view
Returned as `sip` (`planView()` in `sip.service.ts`):

| Field | Notes |
|---|---|
| `id` | `mfpp_…` |
| `isin`, `fund`, `amount` | |
| `state` | FP state, or `awaiting_payment` (first payment not settled) or `failed` (first payment failed) |
| `status` | "Setting up", "Not started", "Activating", "Active", "Awaiting first payment", "Payment received", "Payment failed", "Paused", "Modified", "Cancelled", "Completed", "Failed" |
| `first_payment_failed`, `first_payment_received` | |
| `frequency`, `installment_day`, `start_date`, `next_installment_date` | FP's dates (`next_installment_date` ignores pauses) |
| `next_debit` | The next debit that will actually run (pause-aware); null when stopped |
| `next_after_resume` | Next debit once the pause is lifted |
| `previous_installment_date`, `number_of_installments`, `remaining_installments`, `folio`, `created_at` | |
| `installments` | `[{ id, old_id, state, amount, scheduled_on, traded_on, units, nav, folio, failure_code, upfront }]` |
| `mandate_id`, `paid_count`, `invested`, `units`, `current_value`, `gain_pct` | |
| `pause` | `{ ids, from, to, skipped, resumes_on }` or null |
| `replaced_by`, `replaces`, `cancellation_code` | |

### POST /sip
Rate limit money. OTP `action: "sip"`.

| Field | Type | Rules / message |
|---|---|---|
| `isin` | string | |
| `amount` | number | "Enter an amount." |
| `day` | number | 1-28. "Choose a date between 1 and 28." |
| `frequency` | string | Optional, only `monthly`. "Only monthly SIPs are available." |
| `first_now` | boolean | Optional. `true` = generate the first instalment today |
| `first_pay` | string | Optional, `upi`: pay that first instalment on the UPI checkout (else it is debited from the mandate by NACH) |
| `installments` | number | Optional, 1-1200 ("Choose between 1 and 1,200 instalments."). FP gets `max(scheme minimum or 6, installments or 120)` |
| `otp` | string | |

Order of checks: investor -> scheme rules -> AutoPay -> OTP. Then FP create -> review -> consent + confirm -> active.

```json
{ "sip": { "...": "SIP view", "status": "Awaiting first payment" },
  "token_url": "https://…", "return_url_prefix": "<PUBLIC_URL>/pay/return" }
```

`token_url` only with `first_now: true, first_pay: "upi"`. If that first payment fails, the backend cancels the SIP
(the `sip-first-payments` job does this even if the app is never reopened).

| Status | `error` |
|---|---|
| 400 | "SIP is not available for <Fund> at this frequency." / "Minimum SIP for <Fund> is ₹N." / "SIP amount must be in multiples of ₹N." / "Choose a SIP date between 1 and 28." |
| 409 `{ need_mandate: true }` | "Set up AutoPay first." / "This AutoPay cannot be debited. Please set up AutoPay again." |
| 409 `{ need_mandate: true, limit }` | "This is above your AutoPay limit of ₹N. Set up a new AutoPay with a higher limit." |
| 422 | Review slow / "The fund platform could not register this SIP right now. Nothing was debited; …" / "The fund house did not accept this SIP: …" |

### GET /sips
Returns `{ sips: [SIP view] }`; up to 12 open plans are refreshed first.

### GET /sips/:id
Path id `mfpp_…` ("SIP not found"). Returns `{ sip }`.

### POST /sips/:id/cancel
Rate limit money. Body `{ code?, otp }`, OTP `action: "sip_cancel"`. `code` (max 60) is one of
`amount_not_available`, `investment_returns_not_as_expected`, `amc_support_not_satisfactory`,
`exit_load_not_as_expected`, `switch_to_other_scheme`, `fund_manager_changed`, `investment_goal_complete`,
`mandate_not_ready`, `invest_later`, `customer_support_not_satisfactory`; anything else becomes `invest_later`.
Returns `{ sip }`. `409 "This SIP has already stopped."`.

### POST /sips/:id/pause
Rate limit money. Body `{ months: 1 | 2, otp }` ("A SIP can be paused for up to 2 months."), OTP `action: "sip_pause"`.
Creates an FP skip instruction starting at the first instalment at least 2 days away. Returns `{ sip }`.
Errors: `409 "Only an active SIP can be paused."` / "This SIP is already paused." / "This SIP has no upcoming
instalment to pause."; `400 "SEBI rules allow skipping at most N instalment(s) in a row for this SIP right now."`;
422 FP refused.

### POST /sips/:id/resume
Rate limit money. Body `{ otp }`, OTP `action: "sip_resume"`. Returns `{ sip }`. `409 "This SIP is not paused."`.

### POST /sips/:id/modify
Rate limit money. Body `{ amount, day, otp }`, OTP `action: "sip_modify"`. FP can't edit ONDC plans, so this registers a
new plan on the same folio and mandate (with a skip so it doesn't debit before the old plan would have), then cancels
the old one. Returns `{ sip, old }`.
Errors: `409 "Only an active SIP can be changed."` / "Resume this SIP before changing it."; `400 "Change the amount or
the SIP date first."`; AutoPay 409s; `422 "We couldn't move this SIP to the new date without an extra debit (…)…"`;
`502 { sip }` "Your new SIP is registered, but the old one could not be cancelled. Please cancel it from your SIPs."

---

## Plans (STP, SWP)

Auth **user**. ONDC supports monthly only.

### STP view
`{ id, isin, isin_to, fund, fund_to, amount, state, status, frequency, installment_day, start_date,
next_installment_date, previous_installment_date, number_of_installments, remaining_installments, folio, created_at,
reason, installments }`. `status`: "Setting up", "Not started", "Activating", "Active", "Cancelled", "Completed",
"Failed".

### POST /stp
Rate limit money. OTP `action: "stp"`.

| Field | Type | Rules / message |
|---|---|---|
| `from_isin`, `to_isin` | string | |
| `folio` | string | |
| `amount` | number | Floored to the rupee |
| `day` | number | 1-28 |
| `installments` | number | 1-1200. "Choose how many monthly transfers." |
| `otp` | string | |

Returns `{ stp }`. Errors (400 unless noted): "Choose a different fund to transfer into." / "An STP can only move money
between funds of the same fund house." / "<Fund> does not allow STPs out of it." / "<Fund> does not accept STPs." /
"Minimum transfer is ₹N a month." / "Amount must be in multiples of ₹N." / "An STP needs at least N monthly
transfers." / "Choose a transfer date between 1 and 28." / "You hold ₹N in this fund, less than one transfer."; 409 an
all-units redemption / switch in progress; 422 review failed.

### GET /stps, GET /stps/:id
`{ stps: [...] }` / `{ stp }`. Path id `mfsp_…` ("STP not found").

### POST /stps/:id/cancel
Rate limit money. Body `{ otp }`, OTP `action: "stp_cancel"`. Already stopped: returns `{ stp }` without asking for
the OTP. Never confirmed: `400 "This STP never started, so there is nothing to cancel. No transfers will run on it."`.

### SWP view
Like STP (no `isin_to` / `fund_to`), plus `until_stopped`, `withdrawals`, `first_withdrawal`, `next_withdrawal`.

### POST /swp/otp
Rate limit otpSendUser. Requires an investor. Same as `POST /txn/otp { "action": "swp" }`; kept for older app builds.

### POST /swp
Rate limit money. OTP `action: "swp"`.

| Field | Type | Rules / message |
|---|---|---|
| `isin`, `folio`, `amount`, `day` | | As STP |
| `until` | string | Optional, `stopped` = runs until cancelled (FP gets 1,200 instalments) |
| `installments` | number | 1-1200; required unless `until: "stopped"`. "Choose how many monthly withdrawals." |
| `start_offset` | number | 0-2 (default 0): start 0-2 months later. "The first withdrawal can start at most 2 months later." |
| `otp` | string | |

Returns `{ swp }`. Errors: "<Fund> does not allow SWPs." / "Minimum withdrawal is ₹N a month." / "Amount must be in
multiples of ₹N." / "An SWP needs at least N monthly withdrawals." / "Choose a withdrawal date between 1 and 28." /
"The first withdrawal date is too close to start later. Pick another date or start this month." / "You hold ₹N in this
fund, less than one withdrawal." (400); 409 all-units order in progress; 422 review failed or later start refused.

### GET /swps, GET /swps/:id, POST /swps/:id/cancel
As STP, with path id `mfrp_…` ("SWP not found") and OTP `action: "swp_cancel"`.

---

## Reports

Auth **user**, `no-store`.

### GET /reports/:key?fy=&id=
`key` (`^[a-z-]{2,30}$`, unknown: `404 "Report not found"`):

| Key | Report |
|---|---|
| `capital-gains` | Realised gains (FIFO lots from FP), classified short / long term |
| `holdings` | Holdings by folio |
| `dividends` | IDCW credits |
| `elss` | ELSS investments |
| `orders` | Order history |
| `contract-notes` | Completed orders, each linking to its contract note |
| `contract-note` | One order's contract note; needs `id` (an order or SIP / SWP instalment id). 404 "Order not found"; 400 "No contract note for this order." / "A contract note is available once the order is successful." |
| `balance` | Risips balance statement (empty) |
| `gst` | GST statement (empty) |

`fy` = start year of the financial year (`2026` = FY 2026-27; "Choose a financial year."); default the current FY.

Returns `{ key, title, period, fy?, summary: [{ label, value, tone? }], details?, sections: [{ title, columns, rows:
[{ title, sub, value, value_sub?, tone?, link?, cells }] }], empty, notes, investor: { name, pan (masked) },
generated_at }`.

### GET /reports/:key/html
Same input. Returns `{ html }`: a printable statement (the app makes a PDF of it).

---

## Settings

Auth **user**.

### GET /settings/notifications, PUT /settings/notifications
GET returns `{ prefs }`. PUT body = any of these keys with boolean values (unknown key: `Unknown preference "x".`;
non-boolean: `"x" must be true or false.`); returns `{ prefs }`.

| Key | Default |
|---|---|
| `sip_reminders` | true |
| `portfolio_summary` | true |
| `goals_reviews` | true |
| `market_insights` | false |
| `push` | true |
| `whatsapp` | true |
| `email` | false |

### POST /settings/pin/check
Rate limit pin. Body `{ current }`. Returns `{ ok: true }`. Wrong: `400 "Incorrect PIN. N attempts left."`,
`reason: "wrong_pin"`, `attempts_left`; 5th wrong: `401 pin_locked`.

### POST /settings/pin/change
Rate limit pin. Body `{ current, pin }`. Returns `{ ok: true, user }`. Same PIN errors; `400 "Choose a PIN different
from your current one."`.

### GET /settings/devices
Returns `{ current, others }`; each `{ id, name, os, signed_in_at, last_active_at, current }`.

### POST /settings/devices/:id/logout
Path id 1-12 digits. Returns `{ ok: true }`. `400 "Use Logout to sign out of this device."` (own session);
`404 "That device is already signed out."`.

### POST /settings/devices/logout-others
Returns `{ ok: true, signed_out: <count> }`.

### GET /settings/deletion
Returns `{ checks: { active_plans, pending_orders, ok }, request: { status, requested_at, purge_after } | null,
grace_days: 30 }`.

### POST /settings/deletion
Rate limit money. Body `{ reason?: "not_using" | "other_platform" | "privacy" | "other" | null, otp }` (OTP
`action: "delete_account"`; "Unknown reason."). Requests deletion in 30 days and signs out every session; signing in
again cancels it. Returns `{ ok: true, request }`.
Errors: `409 { checks }` "Stop your active SIPs, STPs and SWPs before deleting your account." / "Wait for your pending
orders to complete before deleting your account."; OTP errors.

### POST /profile/nominee
Rate limit money. Adds a nominee after onboarding (for an investor who opted out). Same body and field checks as
`POST /onb/nominee` (OTP `action: "nominee"`), but `skip: true` is refused. FP copies the nominee only into folios
opened from now on. Returns `{ user }`.
Errors: `409 "Please finish setting up your account first."`; `409 "You already have a nominee. Contact support to change
it."`; `400 "Enter your nominee’s details."`; the same nominee 400s; OTP errors; FP 502.

---

## Notifications

Auth **user**.

### GET /notifications
Built on the fly from orders, SIPs, KYC state, sessions, NSE market holidays and `src/content/*.json`.

```json
{
  "notifications": [ { "id": "order-mfp_…-ok", "tag": "ORDER", "title": "…", "body": "…", "at": "ISO",
                       "route": "/invest/result?id=…", "read": false } ],
  "unread": 3,
  "modal": { "id": "holiday-2026-11-05", "kind": "holiday", "...": "..." },
  "banner": { "id": "…", "title": "…", "body": "…", "route": "…" }
}
```

Tags include `SECURITY`, `MARKET`, `REWARDS`, `ORDER`, `SIP`, `ACTION NEEDED`, `KYC`. Fresh-KYC investors get "Your
KYC is with the KRA" and later "Your KYC is approved". Promos and announcements show only with `market_insights` on;
SIP-due reminders only with `sip_reminders` on. Max 50.

### POST /notifications/read
Body `{ ids?: (string|number)[] }` (max 500). No `ids` = mark everything current read. Returns `{ ok: true }`.

### POST /notifications/seen
Body `{ id }`: marks a modal as shown. Returns `{ ok: true }`.

---

## External funds (MF Central import)

Auth **user** unless noted. Uses MF Central when all `MFC_*` keys are set, else the local stand-in in demo mode,
else unavailable.

### Status object
`{ status: "none" | "otp_sent" | "importing" | "ready" | "failed", provider: "mfcentral" | "sandbox" | null,
available, pan, pan_masked, pan_locked, mobile, email, request: { started_at, qr_at, sent_to } | null, error,
imported_at, as_of, count, sample, hidden, consent_text }`. `pan` is null when the KYC PAN is on file (then only
`pan_masked`).

### GET /external
`no-store`. Polls MF Central if importing; returns the status object.

### POST /external/start
Rate limit otpSendUser.

| Field | Type | Rules / message |
|---|---|---|
| `via` | string | `mobile` (default) or `email` |
| `consent` | `true` | "Please give your consent to fetch your holdings from MF Central." |
| `pan` | string | Optional (max 20); the KYC PAN is used when on file. "Please enter a valid PAN, e.g. ABCDE1234F." |
| `mobile` / `email` | string | Optional; default the account's own |

Returns `{ journey_url, provider, sent_to }`. The app opens `journey_url` in a WebView; MF Central sends an OTP.
Errors: `503 "Importing from MF Central isn’t switched on yet. Please try again later."`; 400 PAN / mobile / email;
400 / 502 MF Central refused.

### POST /external/qr
Body up to 12 MB: `{ qr: "<base64 PNG or data URL>", req_id?, extent? }` ("We didn’t receive the QR code. Tap Download
QR on the MF Central page, then Continue."). Returns the status object (`importing`). `409 "Please start the import
again."`; 400 / 502 QR refused.

### POST /external/cancel, POST /external/visibility, DELETE /external
- cancel: leaves an unfinished journey; returns status.
- visibility: `{ hidden: boolean }` -> `{ ok: true, hidden }`.
- DELETE: removes all imported funds; returns status.

### GET /external/return
Auth none. HTML "Returning to Risips…" page.

### MF Central stand-in (demo mode only)
`GET /mfc-sandbox/journey/:token` (HTML), `POST /mfc-sandbox/api/verify { token, otp }` (rate limit otpVerify; the page
shows the OTP), `POST /mfc-sandbox/api/generate-qr { token, planType?, extentOfData? }`. The statement is released
about 20 s after the QR and contains sample holdings (`sample: true`).

---

## Assistant

Auth **user**.

### GET /assistant
Returns `{ name, intro, groups: [{ title, chips }] }`.

### POST /assistant
Body `{ text?: string (max 500, "Keep the question under 500 characters."), action?: "book_call" }`.
Returns `{ messages: [{ kind: "text", text } | { kind: "sips", items } | { kind: "order", order }], chips, actions:
[{ label, route?, kind }] }`. Intent matching over the investor's own data; never recommends a fund.
`book_call` stores an advisor callback request (one open at a time). Empty text: `400 "Type a question."`.

---

## Webhooks

### POST /fp/webhook/:token
Auth: the secret token in the path (`FP_WEBHOOK_TOKEN`, 24+ chars). Rate limit webhook. Bad / unset token:
`404 { "error": "Not found" }`. Otherwise answers `{ "ok": true }` at once, then re-fetches the order or plan (or the
plan of an instalment) from FP and updates it. FP events carry no signature, so the payload is only a hint. Registered
with `scripts/register-webhooks.ts`.

---

## Admin API (/admin/v1)

Read-only investor and order data for the admin dashboard's backend. Off (`404`) unless `ADMIN_API_KEY` is set.

Headers: `x-admin-key` (required), `x-admin-actor` (the staff user, logged). Own rate limit (600/min per IP), not the
global one. Every call is logged to `logs/admin_api.log`.

| Status | `error` |
|---|---|
| 403 | "Not allowed from this address." (IP not in `ADMIN_API_ALLOWED_IPS`) |
| 401 | "Bad or missing x-admin-key." |
| 405 | "The Admin API is read-only." (any method but GET) |

Never returned: full PAN, bank account numbers, Aadhaar, selfie / signature files, payment links, 2FA records, sessions.

| Endpoint | Query / params | Returns |
|---|---|---|
| `GET /admin/v1/summary` | | `{ investors: { total, kyc_compliant, by_stage }, orders: { <kind>: { <state>: { count, amount } } }, generated_at }` |
| `GET /admin/v1/investors` | `q` (matches name, email, mobile, id; max 100), `stage` (`^[a-z_-]{1,40}$`), `limit` 1-200 (default 50, clamped), `offset` | `{ total, limit, offset, data: [{ id, name, email, mobile, stage, kyc_compliant, pan_masked, signed_up_at, onboarded_at, orders, plans }] }` |
| `GET /admin/v1/investors/:id` | id 1-2147483647 ("No such investor.") | `{ investor, orders, sips, stps, swps }` (`investor` adds dob, gender, profile, address, `aadhaar_masked`, bank (masked), nominee name/relationship, `checks`, `deletion`, `fp`) |
| `GET /admin/v1/investors/:id/portfolio` | | Live portfolio from FP, or `{ holdings: [], note }` |
| `GET /admin/v1/orders` | `kind` (`purchase`, `redemption`, `switch`, `sip`, `stp`, `swp`), `state`, `user_id`, `from`, `to` (`YYYY-MM-DD`, created date UTC), `limit`, `offset` | `{ total, limit, offset, data: [order or plan view + kind, user_id, updated_at] }` |
| `GET /admin/v1/orders/:id` | id `mfp_ mfr_ mfs_ mfpp_ mfsp_ mfrp_` ("No such order.") | One order or plan |

`stage` = the investor's `next_step` while onboarding, `active` once onboarded, or `deletion_requested` / `deleted`.
The Admin API does not expose `kyc_status` / `kyc_fresh` yet.
