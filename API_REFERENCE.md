# Risips API reference

Every endpoint of the Risips API that testers and app integrators use, grouped by module. For setup, authentication,
limits, end-to-end flows and sandbox test data, see **[README.md](README.md)**.

- **Sandbox base URL:** `https://risips-demo-api.azurewebsites.net`
- All request and response bodies are JSON. Send `Content-Type: application/json` on every request with a body.
- **Auth** in each endpoint is either **None** or **Bearer** (`Authorization: Bearer <token>`).
- Responses are trimmed. `"...": "..."` marks more fields.
- An Admin API and webhook endpoints also exist. They serve internal systems only and are not described here.

## Contents

- [Shared objects](#shared-objects)
  - [Error](#error) · [User](#user) · [KYC form](#kyc-form) · [Fund summary](#fund-summary) · [Order](#order) ·
    [SIP](#sip) · [STP](#stp) · [SWP](#swp) · [Cart](#cart) · [Checkout](#checkout) · [Mandate](#mandate)
- [System](#system)
- [Authentication](#authentication)
- [Transaction OTP](#transaction-otp)
- [Onboarding](#onboarding)
- [Fresh KYC](#fresh-kyc)
- [Funds](#funds)
- [Watchlist](#watchlist)
- [Cart](#cart-1)
- [Orders](#orders)
- [Portfolio](#portfolio)
- [AutoPay (mandate)](#autopay-mandate)
- [SIP](#sip-1)
- [STP](#stp-1)
- [SWP](#swp-1)
- [Reports](#reports)
- [Notifications](#notifications)
- [Settings](#settings)
- [External funds import](#external-funds-import)
- [Assistant](#assistant)

---

## Shared objects

### Error

Every error response has this shape.

| Field | Type | Description |
|---|---|---|
| `error` | string | Message for the investor. Always present |
| `reason` | string | Machine code. Present on some errors (see [README, Reason codes](README.md#reason-codes)) |
| `fields` | object | Validation errors only. One message per bad field, keyed by field name |
| other | various | Some errors add `attempts_left`, `retry_after`, `next_step`, `need_mandate`, `need`, `limit`, `checks`, `pv_id`, `bank_status`, `code`, `kyc_status`, `cart_id`, `order` or `sip` |

```json
{ "error": "Enter a valid email address.", "reason": "validation", "fields": { "email": "Enter a valid email address." } }
```

**Errors common to every endpoint**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | A field is missing or invalid (message per field) |
| 400 | | "The request body is not valid JSON." |
| 401 | | "Please sign in again." (Bearer endpoints: token missing, wrong or ended) |
| 404 | | "Not found" (unknown endpoint) |
| 413 | | "The request is too large." |
| 429 | `rate_limited` | "Too many attempts. Please wait a few minutes and try again." |
| 500 | | "Something went wrong. Please try again." |
| 502 | | A fund platform error, in readable form |
| 503 | | "Can’t reach the fund platform right now. Please try again in a minute." |

### User

Returned as `user` by authentication and onboarding endpoints and by `GET /me`.

| Field | Type | Description |
|---|---|---|
| `id` | number | Account id |
| `email` | string | Sign-in email |
| `mobile` | string \| null | 10 digits, once verified |
| `name` | string \| null | Name as entered at the PAN step |
| `first_name` | string \| null | First word of `name` |
| `dob` | string \| null | Date of birth, `YYYY-MM-DD` |
| `gender` | string \| null | `male`, `female` or `transgender` |
| `pan_masked` | string \| null | PAN with the last 3 characters visible, e.g. `*******51Q`. The full PAN is never returned |
| `aadhaar_masked` | string \| null | e.g. `XXXX XXXX 1234` |
| `occupation` | string \| null | See [POST /onb/profile](#post-onbprofile) |
| `income_slab` | string \| null | See [POST /onb/profile](#post-onbprofile) |
| `marital_status` | string \| null | See [POST /onb/profile](#post-onbprofile) |
| `father_name` | string \| null | Father's or spouse's name |
| `pep` | string \| null | Politically exposed person (PMLA): `none`, `exposed` or `related`; null until answered |
| `address` | object \| null | `{ line1, line2, pincode, city, state }` |
| `bank` | object \| null | `{ ifsc, bank_name, branch, city, pv_id, status, code, verified, account_masked }`. The full account number is never returned |
| `nominee` | object \| null | `{ name, dob, relationship, pan }`. `pan` is masked |
| `nominee_skipped` | boolean | The investor opted out of nomination |
| `nominee_added_at` | string \| null | Set when a nominee was added after onboarding |
| `kyc_compliant` | boolean | The PAN has a good KYC record at the KRA |
| `kyc_fresh` | boolean | The investor is doing a fresh KYC |
| `kyc_status` | string \| null | See the table below |
| `kra_hold` | boolean | The monthly KRA re-check found the PAN not compliant: new purchases are paused, redemptions go on |
| `selfie_url`, `signature_url` | string \| null | Relative signed image links, valid 1-2 hours |
| `biometric` | boolean | Biometric unlock is on |
| `pin_set` | boolean | A PIN is set |
| `onboarded` | boolean | Onboarding is complete and the account can invest (subject to KYC approval) |
| `onboarded_at` | string \| null | ISO time onboarding completed |
| `member_since` | string | Year of sign-up |
| `next_step` | string | The next onboarding step (see the table below) |
| `fp` | object \| null | `{ investor_profile, investment_account }`: ids at the fund platform |

**`kyc_status` values**

| Value | Meaning |
|---|---|
| `compliant` | The PAN has a good KYC record, or the fresh KYC was approved |
| `in_progress` | Fresh KYC, form not yet submitted |
| `with_kra` | Fresh KYC submitted, waiting for the KRA (usually 1-3 working days) |
| `failed` | Fresh KYC form failed or expired. Start it again |
| `null` | Before the PAN step |

**`next_step` values**

| Value | Done when | Endpoint |
|---|---|---|
| `mobile` | Mobile verified | [POST /auth/mobile/start](#post-authmobilestart), [POST /auth/mobile/verify](#post-authmobileverify) |
| `pin` | PIN set | [POST /auth/pin/set](#post-authpinset) |
| `pan` | PAN verified | [POST /onb/pan](#post-onbpan) |
| `pan-confirm` | PAN confirmed | [POST /onb/pan/confirm](#post-onbpanconfirm) |
| `occupation`, `income`, `marital`, `father` | Profile answer saved | [POST /onb/profile](#post-onbprofile) |
| `pep` | Politically exposed person answered (PMLA) | [POST /onb/profile](#post-onbprofile) |
| `address` | Address and gender saved | [POST /onb/address](#post-onbaddress) |
| `aadhaar` | Fresh KYC only: Aadhaar fetched from DigiLocker | [POST /onb/kyc/start](#post-onbkycstart), [POST /onb/kyc/refresh](#post-onbkycrefresh) |
| `bank` | Bank account verified | [POST /onb/bank](#post-onbbank) |
| `selfie-intro` | Older accounts only: selfie saved (PAN without KYC, not on the fresh-KYC path) | POST /onb/selfie |
| `signature-consent` | FATCA declared (fresh KYC: and signature saved) | [POST /onb/declarations](#post-onbdeclarations), [POST /onb/signature](#post-onbsignature) |
| `kyc-esign` | Fresh KYC only: KYC form e-signed | [POST /onb/kyc/esign](#post-onbkycesign), [POST /onb/kyc/refresh](#post-onbkycrefresh) |
| `nominee` | Nominee added or opted out | [POST /onb/nominee](#post-onbnominee) |
| `esign` | Investor created at the fund platform | [POST /onb/esign](#post-onbesign) |
| `home` | Onboarding complete | |

### KYC form

Returned as `kyc` by the fresh-KYC endpoints.

| Field | Type | Description |
|---|---|---|
| `id` | string | KYC form id, `kycf_...` |
| `status` | string | `under_review`, `created`, `awaiting_esign`, `awaiting_submission`, `submitted`, `failed`, `expired` |
| `reason` | string \| null | Reason when the form failed |
| `proof_status` | string | DigiLocker Aadhaar: `pending`, `fetched` or `failed` |
| `fields_needed` | string[] | What the form still needs, e.g. `["signature"]` |
| `signature_provided` | boolean | The signature is on the form |
| `esign_status` | string \| null | E-sign state |
| `expires_at` | string | The form expires 7 days after it is created |
| `submitted_at` | string \| null | Time the form went to the KRA |

### Fund summary

Returned in fund lists.

| Field | Type | Description |
|---|---|---|
| `isin` | string | Fund ISIN |
| `name`, `short_name` | string | Full and short name |
| `amc`, `amc_short`, `amc_id` | string, string, number | Fund house. One `amc` / `amc_short` per `amc_id` (e.g. 15 = "ICICI Pru") |
| `category` | string | `Equity`, `Debt`, `Hybrid`, `Index`, `Commodity` or `Solution` |
| `sub_category` | string | e.g. `Mid Cap Fund` |
| `risk` | string | Riskometer: `Low Risk`, `Low to Moderate Risk`, `Moderate Risk`, `Moderately High Risk`, `High Risk`, `Very High Risk` |
| `tags` | string[] | e.g. `mid`, `large`, `small`, `tax`, `debt`, `index`, `commodity` |
| `nav`, `nav_date` | number, string | Latest NAV and its date (the last working day) |
| `returns` | object | `{ 1D, 1M, 6M, 1Y, 3Y, 5Y }` in %. 3Y and 5Y are annualised. `null` when the fund is too young |
| `lumpsum` | object | `{ allowed, min, additional_min, multiples, max }`. `allowed: false` for SIP-only funds |
| `sip_min` | number \| null | Monthly SIP minimum |
| `stp` | object \| null | `{ in, out, frequency, min, multiples, installments_min, dates }` |
| `swp` | object \| null | `{ frequency, min, multiples, installments_min, dates }` |
| `switch_in_min` | number \| null | Minimum switch into this fund |
| `top_performer` | boolean | Top quarter of its category by 3-year return |
| `purchase_note` | string \| null | "Not taking new investments right now. You can still redeem or switch out." when the fund takes neither lumpsum nor SIP (the fund house has stopped new money, e.g. the gold FoFs); null otherwise |
| `watchlisted` | boolean | On this user's watchlist |

### Order

Returned as `order` for purchases, redemptions and switches.

| Field | Type | Description |
|---|---|---|
| `id` | string | `mfp_...` purchase, `mfr_...` redemption, `mfs_...` switch |
| `ref` | string | Display reference, e.g. `#MF000853` |
| `kind` | string | `purchase`, `redemption` or `switch` |
| `isin`, `fund` | string, object | The fund (for a switch: the source fund) |
| `fund_to` | object \| null | Switch target fund |
| `amount` | number \| null | Rupees. `null` for an all-units redemption or switch until priced |
| `units`, `units_in`, `nav` | number \| null | Filled at allotment. `units_in` is the units received by a switch |
| `state` | string | `under_review`, `pending`, `confirmed`, `submitted`, `successful`, `failed`, `cancelled`, `reversed` |
| `status` | string | Label: "Processing", "Awaiting payment", "Payment pending", "Submitted to AMC", "Successful", "Failed", "Payment failed" |
| `failure_code` | string \| null | Why the order failed, e.g. `order_failure_at_gateway` |
| `folio` | string \| null | Filled at allotment |
| `token_url` | string \| null | Payment link while a purchase is unpaid |
| `payment` | object | `{ id, status, reference, method }`. `status`: `PENDING`, `SUCCESS`, `FAILED`. `method`: `UPI` or `NETBANKING` (orders placed from 13 Nov; older ones have none) |
| `created_at`, `updated_at`, `succeeded_at`, `traded_on` | string \| null | Times (ISO 8601 with their timezone, e.g. `2026-11-09T07:24:00Z`) and trade date |
| `all_units` | boolean | An all-units redemption or switch |
| `estimated_amount`, `processed_amount`, `processed_units` | number \| null | Amounts as processed |

### SIP

Returned as `sip`.

| Field | Type | Description |
|---|---|---|
| `id` | string | `mfpp_...` |
| `isin`, `fund`, `amount` | | Fund and monthly amount |
| `state` | string | Plan state, or `awaiting_payment` (first payment not settled) or `failed` (first payment failed) |
| `status` | string | "Setting up", "Not started", "Activating", "Active", "Awaiting first payment", "Payment received", "Payment failed", "Paused", "Modified", "Cancelled", "Completed", "Failed" |
| `first_payment_received`, `first_payment_failed` | boolean | `first_payment_received` is `true` only while the first instalment is paid but not yet allotted ("Payment received"); it goes back to `false` once units are allotted. `paid_count` counts paid instalments |
| `frequency`, `installment_day`, `start_date` | | Schedule |
| `next_installment_date` | string \| null | FP's scheduled date. It ignores pauses and a modified SIP's skipped first months, so it can be earlier than `next_debit` |
| `next_debit` | string \| null | The next debit that will actually run, adjusted for pauses. `null` when stopped |
| `next_after_resume` | string \| null | The next debit if the SIP is resumed now. While paused this can be the first skipped date |
| `previous_installment_date`, `number_of_installments`, `remaining_installments` | | Schedule |
| `folio`, `created_at`, `mandate_id` | | |
| `installments` | array | `[{ id, old_id, state, amount, scheduled_on, traded_on, units, nav, folio, failure_code, upfront }]` |
| `paid_count`, `invested`, `units`, `current_value`, `gain_pct` | number | Totals so far. `units` / `current_value` are the SIP's units still held: units redeemed or switched out of its folio come off (0 → `null` after a full redemption) |
| `pause` | object \| null | `{ ids, from, to, skipped, resumes_on }` |
| `replaces`, `replaced_by` | string \| null | Links between an old SIP and the SIP that replaced it after a change |
| `cancellation_code` | string \| null | Reason given when stopped |
| `skip_ignored` | array | Instalments FP created for a date the investor paused (it should not): `[{ id, old_id, scheduled_on, state, amount, skip_id }]`. Normally empty. The investor gets an `ACTION NEEDED` notification and the backend logs it |

### STP

Returned as `stp`.

| Field | Description |
|---|---|
| `id` | `mfsp_...` |
| `isin`, `fund`, `isin_to`, `fund_to` | Source and target funds |
| `amount`, `frequency`, `installment_day` | Monthly amount and day |
| `state`, `status` | `status`: "Setting up", "Not started", "Activating", "Active", "Cancelled", "Completed", "Failed" |
| `start_date`, `next_installment_date`, `previous_installment_date` | Schedule |
| `number_of_installments`, `remaining_installments` | Counts |
| `folio`, `created_at`, `reason`, `installments` | |

### SWP

Returned as `swp`. The same fields as [STP](#stp) without `isin_to` and `fund_to`, plus:

| Field | Description |
|---|---|
| `until_stopped` | `true` when it runs until stopped |
| `withdrawals` | Number of withdrawals |
| `first_withdrawal`, `next_withdrawal` | The real first / next withdrawal, after `start_offset` skipped months. `start_date` is FP's plan start, which can be earlier. A cancelled, completed or failed plan has `next_withdrawal` and `next_installment_date` `null`, and `first_withdrawal` `null` if it never came due |

### Cart

Returned by every cart endpoint.

| Field | Type | Description |
|---|---|---|
| `items` | array | `[{ id, isin, mode, amount, added_at, fund, problem }]`. `id` is `ci_...`. `problem` explains why a line can't be bought now, or is `null` |
| `sip_total` | number | Sum of monthly SIP amounts |
| `lumpsum_total` | number | Sum of one-time amounts |
| `pay_now` | number | One-time total plus each SIP's first instalment |
| `sip_dates` | number[] | SIP days every SIP fund in the cart accepts |

### Checkout

Returned as `checkout`.

| Field | Type | Description |
|---|---|---|
| `id` | string | `chk_...` |
| `sip_day` | number \| null | SIP day for SIP lines |
| `created_at` | string | |
| `items` | array | `[{ cart_id, isin, fund, mode, amount, state, order_id, error, status }]`. `state`: `queued`, `placed`, `failed`, `skipped`. `status` (in `GET /checkouts/{id}`): `{ label, tone, detail?, ref?, sip_id? }` |
| `total` | number | Amount to pay now |
| `placed`, `failed`, `remaining`, `position` | number | Progress counters |

### Mandate

Returned as `mandate` (AutoPay).

| Field | Type | Description |
|---|---|---|
| `id` | number | Mandate id |
| `status` | string | `CREATED` (waiting for approval), `APPROVED`, or a failed or cancelled state |
| `type` | string | `UPI` or `E_MANDATE` |
| `limit` | number | Most that can be debited in a month |
| `umrn` | string \| null | Bank mandate reference, once registered |
| `valid_from`, `valid_to` | string | Validity dates |
| `created_at` | string | |
| `bank_account_id` | number | The bank account debited |

---

## System

### GET /health

Shows which environment the server is.

**Auth:** None

**Example request**

```http
GET /health
```

**Example response** `200`

```json
{ "ok": true, "fp_env": "sandbox", "demo": true, "schemes": 35 }
```

| Field | Description |
|---|---|
| `fp_env` | `sandbox` or `production` |
| `demo` | `true` on the sandbox: one-time codes are returned as `dev_otp` |
| `schemes` | Number of funds |

### GET /pay/return, /mandate/return, /kyc/return, /external/return

The "Returning to Risips…" page that hosted payment, AutoPay, KYC and import pages end on. The app closes its WebView
when it reaches one of these addresses. Query strings such as `?order=`, `?mandate=` or `?step=` are for information.

**Auth:** None. **Response:** an HTML page.

### GET /files/{name}

Downloads a signature or selfie image from a `signature_url` or `selfie_url`.

**Auth:** None. The link itself carries an expiring signature (`exp`, `sig`).

**Errors**

| Status | When |
|---|---|
| 404 | Missing or wrong signature, expired link, or unknown file (empty body) |

---

## Authentication

### POST /auth/email/start

Sends a 6-digit sign-in code to the email. Sign-up and sign-in use the same call.

**Auth:** None

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | Yes | Max 254 characters. Must end in `.com`, `.in`, `.org`, `.co`, `.net`, `.edu`, `.me` or `.uk`. Trimmed and lower-cased |

**Example request**

```http
POST /auth/email/start

{ "email": "qa.tester01@example.com" }
```

**Example response** `200`

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "454804" }
```

`existing` is `true` when the email already has an account. `dev_otp` is returned on the sandbox only. The code is
valid for 120 seconds.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter a valid email address." |
| 429 | `rate_limited` | "A code was sent just now. Please try again in N seconds." (`retry_after`) |
| 429 | `rate_limited` | "Too many wrong codes. Please try again in N minutes." (`retry_after`). No code is sent |
| 503 | | "We couldn't send the code. Please try again in a minute." |

### POST /auth/email/verify

Checks the sign-in code. Creates the account if it is new, and returns a session token. Signing in also cancels a
pending account deletion.

**Auth:** None

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | Yes | As for `/auth/email/start` |
| `code` | string or number | Yes | 4-8 digits |

**Example request**

```http
POST /auth/email/verify

{ "email": "qa.tester01@example.com", "code": "454804" }
```

**Example response** `200`

```json
{
  "token": "<TOKEN>",
  "is_new": true,
  "user": { "id": 345, "email": "qa.tester01@example.com", "mobile": null, "pin_set": false,
            "onboarded": false, "next_step": "mobile", "...": "..." }
}
```

`token` is 32 characters. The response adds `"deletion_cancelled": true` when a deletion request was cancelled.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter the code we sent you." |
| 400 | `wrong` | "Incorrect code. 4 attempts left." (`attempts_left`). Last try: "Incorrect code. No attempts left: request a new code." |
| 400 | `locked` | "Too many attempts. Request a new code." |
| 400 | `expired` | "Code expired. Tap Resend Code." |
| 400 | `missing` | "Please request a new code." |
| 409 | | "Please request a new code and sign in again." |
| 429 | `rate_limited` | "Too many wrong codes. Please try again in N minutes." (`retry_after`) |

### POST /auth/mobile/start

Sends a 6-digit code to the mobile number to link it to the account.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `mobile` | string or number | Yes | Spaces, dashes, `+91` and a leading `0` are removed and the last 10 digits are kept. Must start with 6-9 and not be one digit repeated |

**Example request**

```http
POST /auth/mobile/start

{ "mobile": "+91 79262 24018" }
```

**Example response** `200`

```json
{ "ok": true, "expires_in": 120, "code_length": 6, "dev_otp": "623956" }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter a valid 10-digit mobile number." |
| 409 | `mobile_taken` | "This mobile number is already registered with another Risips account. Please use a different number." No code is sent |
| 409 | | "Your mobile number is already registered. Contact support to change it." (onboarded account, different number) |
| 429 | `rate_limited` | "A code was sent just now. Please try again in N seconds." |
| 503 | | "We couldn't send the code. Please try again in a minute." |

### POST /auth/mobile/verify

Checks the mobile code and saves the number.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `mobile` | string or number | Yes | The number the code was sent to (any format) |
| `code` | string or number | Yes | 4-8 digits |
| `bank_consent` | boolean | No | Default `false` |

**Example request**

```http
POST /auth/mobile/verify

{ "mobile": "07926224018", "code": "623956", "bank_consent": true }
```

**Example response** `200`

```json
{ "user": { "mobile": "7926224018", "next_step": "pin", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `wrong` | "Incorrect code. 4 attempts left." (`attempts_left`) |
| 400 | `locked` | "Too many attempts. Request a new code." |
| 400 | `expired` | "Code expired. Tap Resend Code." |
| 400 | `missing` | "Please request a new code." (no code, or a different number) |
| 409 | `mobile_taken` | "This mobile number is already registered with another Risips account. Please use a different number." |
| 429 | `rate_limited` | "Too many wrong codes. Please try again in N minutes." |

### POST /auth/pin/set

Sets the 4-digit app PIN.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `pin` | string | Yes | Exactly 4 digits |

**Example request**

```http
POST /auth/pin/set

{ "pin": "1357" }
```

**Example response** `200`

```json
{ "user": { "pin_set": true, "next_step": "pan", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "PIN must be 4 digits." |
| 429 | `rate_limited` | Over 10 PIN attempts in 15 minutes |

### POST /auth/pin/verify

Checks the PIN when the app is opened.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `pin` | string | Yes | 4 digits |

**Example request**

```http
POST /auth/pin/verify

{ "pin": "1357" }
```

**Example response** `200`

```json
{ "ok": true }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "No PIN set." |
| 400 | | "Incorrect PIN. 4 attempts left." (`attempts_left`) |
| 401 | `pin_locked` | "Too many wrong PIN attempts. Please sign in again with your email." The session ends |
| 429 | `rate_limited` | Over 10 PIN attempts in 15 minutes |

### POST /auth/biometric

Turns biometric unlock on or off.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `enabled` | boolean | Yes | `true` or `false` |

**Example request**

```http
POST /auth/biometric

{ "enabled": true }
```

**Example response** `200`

```json
{ "user": { "biometric": true, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Must be true or false." |

### POST /auth/logout

Ends this session. The token stops working.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{ "ok": true }
```

### GET /me

Returns the signed-in user.

**Auth:** Bearer

**Example request**

```http
GET /me
```

**Example response** `200`

```json
{ "user": { "id": 345, "email": "qa.tester01@example.com", "mobile": "7926224018", "pan_masked": "*******51Q",
            "kyc_status": "compliant", "onboarded": true, "next_step": "home", "...": "..." } }
```

---

## Transaction OTP

Money actions, the nominee step and account deletion need a transaction code. Ask for it here, then send it as `otp`
with the action. A code is for one action, works once, and lasts 300 seconds. A new code for the same action replaces
the old one.

### POST /txn/otp

Sends a 6-digit transaction code to the registered mobile and email.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `action` | string | Yes | One of the actions below |

| `action` | Used by |
|---|---|
| `lumpsum` | [POST /invest/lumpsum](#post-investlumpsum) |
| `redeem` | [POST /invest/redeem](#post-investredeem) |
| `switch` | [POST /invest/switch](#post-investswitch) |
| `cart` | [POST /checkouts](#post-checkouts) |
| `sip` | [POST /sip](#post-sip) |
| `sip_modify` | [POST /sips/{id}/modify](#post-sipsidmodify) |
| `sip_cancel` | [POST /sips/{id}/cancel](#post-sipsidcancel) |
| `sip_pause` | [POST /sips/{id}/pause](#post-sipsidpause) |
| `sip_resume` | [POST /sips/{id}/resume](#post-sipsidresume) |
| `stp` | [POST /stp](#post-stp) |
| `stp_cancel` | [POST /stps/{id}/cancel](#post-stpsidcancel) |
| `swp` | [POST /swp](#post-swp) |
| `swp_cancel` | [POST /swps/{id}/cancel](#post-swpsidcancel) |
| `nominee` | [POST /onb/nominee](#post-onbnominee), [POST /profile/nominee](#post-profilenominee) |
| `delete_account` | [POST /settings/deletion](#post-settingsdeletion) |

**Example request**

```http
POST /txn/otp

{ "action": "lumpsum" }
```

**Example response** `200`

```json
{
  "ok": true, "action": "lumpsum", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 79••••018", "email": "q••••@example.com" },
  "dev_otp": "196140"
}
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Unknown action." |
| 400 | | "Please verify your mobile number first." |
| 429 | `rate_limited` | "A code was sent just now. Please try again in N seconds." / "Too many wrong codes. Please try again in N minutes." |
| 503 | | The code could not be sent |

### POST /swp/otp

The same as `POST /txn/otp` with `"action": "swp"`. Kept for older app versions.

**Auth:** Bearer. Requires an account that can invest.

### Errors when an action checks its `otp`

Every endpoint that takes `otp` can return these. Other fields are checked first, so a wrong amount does not use up
the code.

| Status | `reason` | `error` |
|---|---|---|
| 400 | `otp_required` | "Confirm this with the OTP sent to your registered mobile and email." |
| 400 | `wrong` | "That OTP is not correct. 4 attempts left." (`attempts_left`). Last try: "That OTP is not correct. No attempts left: request a new code." |
| 400 | `locked` | "Too many wrong attempts. Request a new OTP." |
| 400 | `expired` | "This OTP has expired. Request a new one." |
| 400 | `missing` | "This OTP has already been used or expired. Request a new one." |
| 429 | `rate_limited` | "Too many wrong codes. Please try again in N minutes." (`retry_after`) |

---

## Onboarding

All onboarding endpoints need **Bearer** auth. Once the account is onboarded, every `POST /onb/*` call except
`POST /onb/esign` returns `409` "Your account is already set up. Contact support to change these details." The `GET`
endpoints keep working.

### POST /onb/pan

Checks the PAN, name and date of birth, and the PAN's KYC status at the KRA. Takes 1-7 seconds (up to 25).

**Auth:** Bearer. Limit: 30 per hour per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `pan` | string | Yes | Individual PAN: 5 letters (4th is `P`), 4 digits, 1 letter. Upper-cased |
| `name` | string | Yes | 3-70 characters: letters, spaces, `.` and `'`, starting with a letter |
| `dob` | string | Yes | `YYYY-MM-DD`, a real past date after 1900. The investor must be 18 or older |
| `fresh_kyc` | boolean | No | Default `false`. `true` lets a PAN with no KRA record continue with fresh KYC |

**Example request**

```http
POST /onb/pan

{ "pan": "TGVPS3751Q", "name": "Qa Tester", "dob": "1992-04-15" }
```

**Example response** `200`

```json
{
  "user": { "name": "Qa Tester", "pan_masked": "*******51Q", "kyc_compliant": true, "kyc_fresh": false,
            "kyc_status": "compliant", "next_step": "pan-confirm", "...": "..." },
  "checks": {
    "readiness":     { "status": "verified", "code": null },
    "pan":           { "status": "verified", "value": "*******51Q" },
    "name":          { "status": "verified", "value": "Qa Tester" },
    "date_of_birth": { "status": "verified", "value": "1992-04-15" }
  },
  "pv_id": "pv_3ad4a3e6eb8945e6b0a7b6d3950805b9",
  "kyc_fresh": false
}
```

With `"fresh_kyc": true` and a PAN with no KRA record, the response is `200` with `kyc_fresh: true`,
`kyc_status: "in_progress"` and `checks.readiness.code: "kyc_unavailable"`. Every `422` response carries `checks` and
`pv_id`, with the PAN masked.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter a valid individual PAN (e.g. ABCPX1234K)." / "Enter your full name as on PAN (letters only)." / "Enter a valid date of birth (YYYY-MM-DD)." |
| 400 | | "You must be 18 or older to open an account. Investing for a minor is not supported yet." |
| 409 | | "Your PAN is already confirmed and can not be edited." |
| 422 | | "No KYC record found for this PAN with the KRA. A fresh KYC is needed." (without `fresh_kyc`) |
| 422 | | "Your KYC was rejected by the KRA. A fresh KYC is needed." |
| 422 | | "Your KYC is incomplete at the KRA and needs a modification." |
| 422 | | "Your KYC is on hold at the KRA and needs a modification." |
| 422 | | "Your KYC is a legacy record and needs a modification." |
| 422 | | "Your KYC is still being processed by the KRA. Please try again in a few days." |
| 422 | | "Your KYC is deactivated. You cannot invest right now." |
| 422 | | "The KRA service is unavailable. Please try again." |
| 422 | | "Your PAN is not linked with Aadhaar. Link it on the income-tax portal and try again." |
| 422 | | "This PAN is invalid. Please check and re-enter." |
| 422 | | "The name does not match PAN records. Enter it exactly as on your PAN card." |
| 422 | | "The date of birth does not match PAN records." |
| 422 | | "We could not verify your PAN right now. Please try again." |

### POST /onb/pan/confirm

Confirms the PAN. After this the PAN cannot be changed.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{ "user": { "next_step": "occupation", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Verify your PAN first." |

### POST /onb/pan/reset

"Edit PAN": clears the PAN, name, date of birth, KYC state and any fresh-KYC form. Only before the PAN is confirmed.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{ "user": { "pan_masked": null, "name": null, "dob": null, "kyc_status": null, "kyc_compliant": false,
            "next_step": "pan", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 409 | | "Your PAN is already confirmed and can not be edited." |

### POST /onb/profile

Saves profile answers. Send any subset of fields.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `occupation` | string | No | `private_sector_service`, `public_sector_service`, `government_service`, `house_wife`, `student`, `business`, `professional`, `retired`, `agriculture`, `service`, `others` |
| `income_slab` | string | No | `upto_1lakh`, `above_1lakh_upto_5lakh`, `above_5lakh_upto_10lakh`, `above_10lakh_upto_25lakh`, `above_25lakh_upto_1cr`, `above_1cr` |
| `marital_status` | string | No | `single`, `married`, `others` |
| `father_name` | string | No | 3-70 characters: letters, spaces, `.` and `'` |
| `gender` | string | No | `male`, `female`, `transgender` |
| `pep` | string | No | Politically exposed person (PMLA): `none`, `exposed` (the investor is one) or `related` (related to one). Error: "Please tell us whether you are a politically exposed person." |

**Example request**

```http
POST /onb/profile

{ "occupation": "private_sector_service", "income_slab": "above_5lakh_upto_10lakh",
  "marital_status": "single", "father_name": "Ramesh Kumar" }
```

**Example response** `200`

```json
{ "user": { "occupation": "private_sector_service", "next_step": "address", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Invalid occupation." / "Invalid income slab." / "Invalid marital status." / "Invalid gender." / "Enter your father's or spouse's full name." |

### GET /onb/pincode/{pin}

Looks up the city and state for a pincode.

**Auth:** Bearer

| Parameter | In | Rules |
|---|---|---|
| `pin` | path | 6 digits |

**Example request**

```http
GET /onb/pincode/560001
```

**Example response** `200`

```json
{ "city": "Bengaluru", "state": "KARNATAKA" }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter a 6-digit pincode." |
| 404 | | "Pincode not found." |

### POST /onb/address

Saves the address and gender. City and state are filled from the pincode.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `line1` | string | Yes | 3-120 characters |
| `line2` | string | No | Max 120 characters |
| `pincode` | string | Yes | 6 digits, a known pincode |
| `gender` | string | Yes | `male`, `female`, `transgender` |

**Example request**

```http
POST /onb/address

{ "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001", "gender": "female" }
```

**Example response** `200`

```json
{ "user": { "address": { "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001",
                         "city": "Bengaluru", "state": "KARNATAKA" },
            "gender": "female", "next_step": "bank", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter your house / street address." / "Address line is too long." / "Enter a 6-digit pincode." / "Select your gender." |
| 400 | | "Pincode not found." |

### GET /onb/ifsc/{ifsc}

Looks up a bank branch.

**Auth:** Bearer

| Parameter | In | Rules |
|---|---|---|
| `ifsc` | path | 4 letters, `0`, 6 letters or digits. Upper-cased |

**Example request**

```http
GET /onb/ifsc/HDFC0000240
```

**Example response** `200`

```json
{ "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI" }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter a valid IFSC." |
| 404 | | "IFSC not found." |

### POST /onb/bank

Verifies that the bank account belongs to the investor. Takes about 2 seconds (up to 20).

**Auth:** Bearer. Limit: 30 per hour per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `account_number` | string or number | Yes | 9-18 digits. Spaces are removed |
| `ifsc` | string | Yes | A valid, known IFSC |

**Example request**

```http
POST /onb/bank

{ "account_number": "50462188351195", "ifsc": "HDFC0000240" }
```

**Example response** `200`

```json
{
  "user": {
    "bank": { "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI",
              "pv_id": "pv_511182e029cb4629a79214993362b38d", "status": "verified", "code": null,
              "verified": true, "account_masked": "**********1195" },
    "next_step": "signature-consent", "...": "..."
  }
}
```

Every `422` response carries `bank_status`, `code` and `pv_id`.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Account number must be 9–18 digits." / "Enter a valid IFSC." |
| 400 | | "Please complete the PAN step first." (`next_step: "pan"`): the bank check needs the verified PAN |
| 400 | | "IFSC not found." |
| 422 | | "Bank verification is taking longer than usual. Please try again, or use another account." |
| 422 | | "We could not confirm this account belongs to you. Please use another account in your name." (`code`: `uncertain` or `low_confidence`) |
| 422 | | "This account has had too many verification attempts and can’t be verified right now. Please use another account in your name." (`code`: `verification_attempt_limit_exceeded`) |
| 422 | | "Bank verification failed. Please check the account number and IFSC." |

### POST /onb/signature

Saves the investor's signature. Required for fresh KYC; the signature is also added to the KYC form.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `image` | string | Yes | Data URL: `data:image/png;base64,...` or `data:image/jpeg;base64,...`. A real PNG or JPEG, 500 bytes to 8 MB. Request body up to 12 MB |

**Example request**

```http
POST /onb/signature

{ "image": "data:image/png;base64,iVBORw0KGgo..." }
```

**Example response** `200`

```json
{ "user": { "signature_url": "/files/346_signature_f7090e7c.png?exp=1793685600&sig=1ec5f0...", "next_step": "signature-consent", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Invalid image." |
| 400 | | "Image is too small or too large." |
| 413 | | "The request is too large." |
| 502 | | The image could not be added to the KYC form |

### POST /onb/declarations

Records the FATCA / CRS declaration.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `fatca` | boolean | Yes | Must be `true` |
| `running_account` | boolean | No | Default `false` |

**Example request**

```http
POST /onb/declarations

{ "fatca": true, "running_account": false }
```

**Example response** `200`

```json
{ "user": { "next_step": "nominee", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Please accept the FATCA / CRS declaration." |

### POST /onb/nominee

Adds a nominee, or opts out. Needs a transaction code (`"action": "nominee"`).

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `skip` | boolean | No | `true` opts out. Other nominee fields are then ignored |
| `name` | string | Yes, unless `skip` | 3-40 letters and spaces. Not the investor |
| `dob` | string | Yes, unless `skip` | `YYYY-MM-DD`. The nominee must be an adult |
| `relationship` | string | Yes, unless `skip` | `father`, `mother`, `spouse`, `son`, `daughter`, `brother`, `sister`, `grand_father`, `grand_mother`, `others` |
| `pan` | string | Yes, unless `skip` | Any PAN format (5 letters, 4 digits, 1 letter). Not the investor's PAN |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /onb/nominee

{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "483157" }
```

**Example response** `200`

```json
{ "user": { "nominee": { "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "*******34E" },
            "nominee_skipped": false, "next_step": "esign", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter the nominee’s full name (letters only)." / "Enter a valid date of birth for the nominee (YYYY-MM-DD)." / "Select the relationship." / "Enter the nominee’s PAN (required as identity proof)." |
| 400 | | "Nominee cannot be yourself." |
| 400 | | "Minor nominees need guardian details, which this demo does not collect. Please add an adult nominee or skip." |
| 400 | | "Nominee PAN must be different from yours." |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /onb/kyc-form

Returns the data for the KYC form preview.

**Auth:** Bearer

**Example response** `200`

```json
{ "user": { "...": "..." }, "generated_at": "2026-11-03T04:20:00.000Z" }
```

### POST /onb/esign

The last onboarding step. Creates the investor and the investment account at the fund platform. Safe to call again:
an onboarded account gets the same result at once.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `consent` | boolean | Yes | Must be `true` |

**Example request**

```http
POST /onb/esign

{ "consent": true }
```

**Example response** `200`

```json
{
  "fp": { "invp": "invp_1c40fba1...", "phone": "phone_cb3e...", "email": "email_bceb...", "addr": "addr_fa77...",
          "bank": "bac_edecb0fa...", "bank_old_id": 275, "relp": "relp_bac8...",
          "mfia": "mfia_16aa8e6268a04a0dbf06965a49422a92", "mfia_old_id": 229 },
  "user": { "onboarded": true, "next_step": "home",
            "fp": { "investor_profile": "invp_1c40fba1...", "investment_account": "mfia_16aa8e62..." }, "...": "..." }
}
```

`relp` is present only when a nominee was added.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Please agree to eSign the form." |
| 400 | | "Please complete the <step> step first." with `next_step`, e.g. "Please complete the bank account step first." |
| 400 | | "Please verify your mobile number first." |
| 502 / 503 | | Fund platform error. Retry: the call resumes where it stopped |

---

## Fresh KYC

For a PAN with no KRA record (`kyc_fresh: true`). These endpoints build a new KYC form: Aadhaar from DigiLocker, the
signature, and an e-sign. The form then goes to the KRA. All need **Bearer** auth and are limited to 30 per hour per
user. See [README, flow 7c](README.md#7c-onboarding-fresh-kyc) for the full sequence.

### POST /onb/kyc/start

Creates (or reuses) the KYC form and returns the DigiLocker link.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `latitude` | number | Yes | Phone location, 6 to 37.5 (inside India) |
| `longitude` | number | Yes | Phone location, 68 to 97.5 (inside India) |

**Example request**

```http
POST /onb/kyc/start

{ "latitude": 12.9716, "longitude": 77.5946 }
```

**Example response** `200`

```json
{
  "user": { "next_step": "aadhaar", "...": "..." },
  "fetch_url": "https://s.finprim.com/v2/identity_documents/iddoc_.../forward_to?token=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return"
}
```

`fetch_url` is `null` when the Aadhaar is already fetched.

The form is filled with the onboarding answers, PEP included (`none` → `no_exposure`, `exposed` → `pep`,
`related` → `related_pep`). If the fund platform fails while filling them, `fetch_url` is still returned and the next
[POST /onb/kyc/refresh](#post-onbkycrefresh) fills them with the location sent here.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Allow location access to continue." / "Your KYC must be done from within India." |
| 400 | | "Please complete your personal details first." |
| 409 | | "Your KYC is already on record with the KRA." (not a fresh-KYC account) |
| 409 | | "Your account is already set up. Contact support to change these details." |
| 422 | | "Your KYC form expired before it was finished. Please start again." / "A fresh KYC can’t be submitted for this PAN. Please contact support." / "Your KYC form was not accepted (...). Please check your details and start again." (with `kyc_status`) |
| 422 | | "Your KYC form could not be filled: <reason>. Please check your details." (the fund platform refused an onboarding answer, with `kyc_status`) |
| 502 | | "DigiLocker is not available right now. Please try again." (the fund platform failed or did not answer) |
| 503 | | "Your KYC form is still being checked. Please try again in a minute." |

### POST /onb/kyc/refresh

Re-reads the KYC form after DigiLocker or e-sign. Fills any onboarding answer the form still lists in `fields_needed`
(for example `pep_details`) and adds the saved signature if the form still needs it.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{
  "user": { "aadhaar_masked": "XXXX XXXX 1234", "next_step": "bank", "...": "..." },
  "kyc": { "id": "kycf_f59b28dec94047bdb5bd3572700dfdaa", "status": "created", "reason": null,
           "proof_status": "fetched", "fields_needed": ["signature"], "signature_provided": false,
           "esign_status": null, "expires_at": "2026-11-10T09:38:29+0530", "submitted_at": null },
  "error": null
}
```

`error` holds the failure text when the form failed, otherwise `null`.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Start your KYC from the Aadhaar step first." |
| 409 | | "Your KYC is already on record with the KRA." |

### POST /onb/kyc/esign

Returns the e-sign link for the completed KYC form. Can take up to 10 seconds.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{ "user": { "next_step": "kyc-esign", "...": "..." },
  "esign_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return" }
```

`esign_url` is `null` when the form is already e-signed.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Your KYC form is not complete yet (missing: ...)." with `next_step`, e.g. "(missing: signature)" |
| 422 | | The form failed or expired (with `kyc_status`) |
| 502 | | "The e-sign page is not available right now. Please try again." |

### GET /onb/kyc/status

Returns the KYC state. Works after onboarding too. For a submitted form, it checks with the KRA (at most once per 10
minutes).

**Auth:** Bearer

**Example response** `200`

```json
{ "user": { "kyc_status": "with_kra", "...": "..." },
  "kyc": { "status": "submitted", "...": "..." },
  "error": null }
```

`kyc` is `null` when there is no KYC form.

---

## Funds

Any signed-in user can use these endpoints. Onboarding does not need to be finished.

### GET /funds

Lists funds, with optional filters.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `q` | query | No | Search words. All must match the name, fund house or category. Max 100 characters |
| `collection` | query | No | `high`, `commodity`, `mid`, `large`, `small`, `tax`, `debt`, `index`, `foryou` |
| `category` | query | No | Comma list, any case, e.g. `equity,hybrid` |
| `risk` | query | No | Comma list of risk labels, e.g. `Low to Moderate Risk` |
| `top` | query | No | Any value: top performers only |
| `sort` | query | No | `1M`, `6M`, `1Y`, `3Y` (default) or `5Y`. Highest return first |
| `picks` | query | No | Any value: "Our picks" (with `collection`) |
| `amc_id` | query | No | Fund house id, e.g. `15` ICICI Prudential, `3` Aditya Birla Sun Life |
| `switch_in` | query | No | Any value: funds that accept switches in |
| `stp_in` | query | No | Any value: funds that accept STPs in |

Each parameter may appear once.

**Example request**

```http
GET /funds?category=Commodity
```

**Example response** `200`

```json
{
  "title": null,
  "funds": [
    {
      "isin": "INF109KC1Y64", "name": "ICICI Prudential Silver ETF FOF - Regular Plan - Growth",
      "short_name": "ICICI Pru Silver ETF FOF", "amc": "ICICI Prudential Mutual Fund", "amc_short": "ICICI Pru", "amc_id": 15,
      "category": "Commodity", "sub_category": "FoF Domestic", "risk": "Very High Risk", "tags": ["commodity"],
      "nav": 26.7905, "nav_date": "2026-11-02",
      "returns": { "1D": -1.3, "1M": -18.01, "6M": -25.34, "1Y": 19.36, "3Y": 33.76, "5Y": null },
      "lumpsum": { "allowed": true, "min": 100, "additional_min": 100, "multiples": 1, "max": 999999999 },
      "sip_min": 100,
      "stp": { "in": true, "out": true, "frequency": "monthly", "min": 1000, "multiples": 1, "installments_min": 6, "dates": [1, 2, "...", 28] },
      "swp": { "frequency": "monthly", "min": 1, "multiples": 1, "installments_min": 2, "dates": [1, 2, "...", 28] },
      "switch_in_min": 100, "top_performer": true, "watchlisted": false
    }
  ]
}
```

`title` is the collection name when `collection` is used, e.g. `"Mid Cap"`. The Commodity collection also includes
ICICI Pru Multi Asset Allocation Fund (category Hybrid, tagged `commodity`), so it has one more fund than the Commodity
category filter. The Index Funds collection has every Index-category fund, sector index funds included (Nifty Pharma
Index), so it matches `category=Index`. An unknown category returns an empty list.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Invalid filter." (a repeated parameter, a value that is too long, or a `sort` that is not `1M`, `6M`, `1Y`, `3Y` or `5Y`) |
| 404 | | "Unknown collection" |

### GET /funds/collections

Lists the collection tiles.

**Auth:** Bearer

**Example response** `200`

```json
{ "collections": [
  { "key": "high", "title": "High Return" }, { "key": "commodity", "title": "Commodity" },
  { "key": "mid", "title": "Mid Cap" }, { "key": "large", "title": "Large Cap" },
  { "key": "small", "title": "Small Cap" }, { "key": "tax", "title": "Tax Saver (ELSS)" },
  { "key": "debt", "title": "Debt & Liquid" }, { "key": "index", "title": "Index Funds" }
] }
```

`foryou` works in `GET /funds?collection=foryou` but is not listed here.

### GET /funds/{isin}

Returns one fund's full page and its NAV chart.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `isin` | path | Yes | Fund ISIN |
| `period` | query | No | Chart range: `1M`, `3M`, `6M`, `1Y` (default), `3Y`, `5Y`, or `all` (the whole chart once, see below). Any other value uses `1Y` |

**Example request**

```http
GET /funds/INF109KC1Q80?period=3Y
```

**Example response** `200`

```json
{
  "fund": {
    "isin": "INF109KC1Q80", "nav": 19.69, "nav_date": "2026-11-02", "risk": "Very High Risk", "category": "Equity",
    "lumpsum": { "allowed": true, "min": 500, "additional_min": 500, "min_for_you": 500, "multiples": 1, "max": 999999999 },
    "sip": [ { "frequency": "monthly", "min": 100, "multiples": 1, "installments_min": 6, "dates": [1, "...", 28] } ],
    "lock_in": "None", "exit_load": "1% if redeemed or switched out within 1 month of allotment; nil after (since 6 Apr 2026)", "tax": { "stcg": "20%", "ltcg": "12.5%*" },
    "expense_ratio": { "regular": 1.73, "direct": 0.88, "date": "2026-10-29", "source": "AMFI TER disclosure" },
    "managers": ["Rajat Chandak", "Sharmila D'Silva"],
    "objective": "To generate long-term capital appreciation ...",
    "portfolio": { "as_of": "2026-09-30", "aum_cr": 25141, "count": 77, "mix": { "equity": 97.16, "debt": 0.52, "funds": 0 },
                   "holdings": [ { "name": "TVS Motor Company Ltd.", "industry": "Automobiles", "kind": "equity", "pct": 9.51 } ] },
    "peers": [ { "isin": "INF209K01AJ8", "name": "ABSL Flexi Cap Fund", "r1": -0.91, "r3": 14.03, "r5": 12.53 } ],
    "redemption_allowed": true, "switch_in_allowed": true, "switch_out_allowed": true,
    "watchlisted": false, "...": "..."
  },
  "period": "3Y",
  "period_return": 14.03,
  "chart": [[1790294400000, 20.15], "...", [1793577600000, 19.69]]
}
```

- `fund` has every [Fund summary](#fund-summary) field plus the page fields above.
- `lumpsum.min` is the first-purchase minimum, the same figure as the fund list and compare. `lumpsum.min_for_you` is
  what this user may invest: the additional-purchase minimum (`additional_min`) once they have a folio at that fund
  house, otherwise `min`.
- `peers` lists up to 3 other funds of the same sub-category, each with its `isin`. The fund itself is never in it.
- `exit_load` and `lock_in` come from the fund house's own documents (SID / KIM / monthly factsheet), checked 2026-11-11; a fund without them says "See the scheme document", except an ELSS (tax saver) fund, which always shows "3 years (ELSS)". `lock_in` is "None" when there is none.
- `chart` is `[time in milliseconds, NAV]` pairs, oldest first. The last point is the latest NAV. 1M, 3M, 6M and 1Y have a
  point for every trading day (about 22, 63, 125 and 250); 3Y and 5Y have about one a week (about 156 and 260). The
  history comes from AMFI's NAVs (via mfapi.in) and is refreshed every 6 hours. If it can't be reached, the chart falls
  back to the weekly points saved in the catalogue.
- `period=all` returns 5 years in one chart: every trading day of the last year and one point a week before it (about
  500 points). `period_return` is `null`; each period's figure is in `fund.returns`. The app asks for `all` once and
  cuts 1M to 5Y itself, so changing the period makes no call.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "Fund not found" |

### GET /funds/compare

Compares one or two funds.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `isins` | query | Yes | 1 or 2 different ISINs, comma-separated |

**Example request**

```http
GET /funds/compare?isins=INF109KC1Q80,INF209K01AJ8
```

**Example response** `200`

```json
{ "funds": [ { "isin": "INF109KC1Q80", "exit_load": "1% if redeemed or switched out within 1 month of allotment; nil after (since 6 Apr 2026)", "lock_in": "None", "min_sip": 100, "...": "..." },
             { "isin": "INF209K01AJ8", "...": "..." } ],
  "unavailable": [] }
```

Each fund has the [Fund summary](#fund-summary) fields plus `exit_load` and `lock_in` (as on the fund page: from the fund house's documents, or "See the scheme document"), `min_sip`, `history`,
`fund_house`, `mandate`, `expense_ratio`, `managers`, `objective`, `portfolio`, `pros` and `cons`. `unavailable` lists
data missing for these funds (`expense_ratio`, `holdings`, `managers`).

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "You can compare up to 2 funds." |
| 400 | `validation` | "Pick two different funds to compare." (the same ISIN twice) |
| 400 | `validation` | "Fund not found" (empty list or a malformed ISIN) |
| 404 | | "Fund not found" (a well-formed ISIN that is not listed) |

### GET /funds/compare/popular

Returns ready-made comparison pairs.

**Auth:** Bearer

**Example response** `200`

```json
{ "pairs": [ [ { "isin": "INF109KC1Q80", "...": "..." }, { "isin": "INF209K01AJ8", "...": "..." } ] ] }
```

---

## Watchlist

### POST /watchlist/{isin}

Adds the fund to the watchlist, or removes it if it is already there.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `isin` | path | Yes | Fund ISIN |

**Example request**

```http
POST /watchlist/INF109KC1Q80
```

**Example response** `200`

```json
{ "watchlisted": true }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "Fund not found" |

### GET /watchlist

Lists watchlisted funds.

**Auth:** Bearer

**Example response** `200`

```json
{ "funds": [ { "isin": "INF109KC1Q80", "watchlisted": true, "...": "..." } ] }
```

---

## Cart

A cart holds up to 10 lines. At checkout, all one-time lines are paid on one UPI page and all SIP first instalments on
a second. Every cart endpoint returns the [Cart](#cart) object.

### GET /cart

Returns the cart.

**Auth:** Bearer

**Example response** `200`

```json
{ "items": [], "sip_total": 0, "lumpsum_total": 0, "pay_now": 0, "sip_dates": [1, 2, "...", 28] }
```

### POST /cart

Adds a line. The same fund and mode again replaces the amount.

**Auth:** Bearer. Requires an account that can invest.

| Field | Type | Required | Rules |
|---|---|---|---|
| `isin` | string | Yes | Fund ISIN |
| `mode` | string | Yes | `lumpsum` or `sip` |
| `amount` | number | Yes | More than 0, rounded to the rupee. Must meet the fund's minimum |

**Example request**

```http
POST /cart

{ "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000 }
```

**Example response** `200`

```json
{
  "items": [ { "id": "ci_87c58783cf", "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000,
               "added_at": "...", "fund": { "...": "..." }, "problem": null } ],
  "sip_total": 0, "lumpsum_total": 1000, "pay_now": 1000, "sip_dates": [1, 2, "...", 28]
}
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Choose SIP or one-time." / "Enter an amount." |
| 400 | | The fund rule, e.g. "Minimum investment in ICICI Pru Flexi Cap fund is ₹500." |
| 400 | | "Your cart can hold up to 10 items." (10 lines: one fund as both one-time and SIP is 2 items) |
| 400 | | "Your first <fund house> investment opens your folio with <fund house>. Buy one <fund house> fund now and add the others once it is allotted (usually by the next working day), so all your <fund house> funds are in one folio." (a second line at a fund house where the investor has no folio yet; also shown as that line's `problem`) |
| 400 | | "Your first investment with <fund house> (<fund>) is still being processed. …" (an order there is waiting for its folio) |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 404 | | "Fund not found" |

### PATCH /cart/{id}

Changes a line's amount or mode.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `id` | string (path) | Yes | `ci_` + 10 characters |
| `mode` | string | No | `lumpsum` or `sip` |
| `amount` | number | No | As for `POST /cart` |

**Example request**

```http
PATCH /cart/ci_87c58783cf

{ "amount": 2000 }
```

**Example response** `200`: the cart.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | The fund rule for the new amount or mode |
| 404 | | "This item is no longer in your cart." |
| 409 | | "This fund is already in your cart with that plan type." |

### DELETE /cart/{id}

Removes a line.

**Auth:** Bearer

**Example request**

```http
DELETE /cart/ci_907ae35612
```

**Example response** `200`: the cart.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "This item is no longer in your cart." |

### POST /checkouts

Starts a checkout. Checks every line, the SIP date and AutoPay, and records the checkout. Nothing is bought yet. Needs
a transaction code (`"action": "cart"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `item_ids` | string[] | No | Up to 10 cart line ids. Default: the whole cart |
| `sip_day` | number | When the selection has a SIP line | 1-28, and in the cart's `sip_dates` |
| `pay_method` | string | No | `upi` (default) or `netbanking`: how every payment of this checkout is made. Error: "Choose UPI or net banking." |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /checkouts

{ "otp": "330197" }
```

**Example response** `200`

```json
{ "checkout": {
    "id": "chk_625e066d5e39cb52", "sip_day": null, "created_at": "2026-11-03 04:21:12",
    "items": [ { "cart_id": "ci_87c58783cf", "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000,
                 "state": "queued", "order_id": null, "error": null, "status": null, "fund": { "...": "..." } } ],
    "total": 1000, "placed": 0, "failed": 0, "remaining": 1, "position": 0 } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Choose a date between 1 and 28." |
| 400 | | "Your cart is empty." |
| 400 | | "Choose a SIP date that every fund in your cart accepts." |
| 400 | | A line's problem, with `cart_id` |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 409 | | "Set up AutoPay first." with `need_mandate: true` and `need` |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /checkouts/{id}/next

Places the next unpaid group and returns its payment link.

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Parameter | In | Required | Rules |
|---|---|---|---|
| `id` | path | Yes | `chk_` + 16 characters |

**Example request**

```http
POST /checkouts/chk_625e066d5e39cb52/next
```

**Example response** `200`

```json
{ "checkout": { "items": [ { "state": "placed", "order_id": "mfp_e4fceae766bc44f7a5579ce4ae8d8054", "...": "..." } ],
                "placed": 1, "remaining": 0, "...": "..." },
  "mode": "lumpsum",
  "order_id": "mfp_e4fceae766bc44f7a5579ce4ae8d8054",
  "order_ids": ["mfp_e4fceae766bc44f7a5579ce4ae8d8054"],
  "order": { "state": "submitted", "status": "Payment pending", "...": "..." },
  "token_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

- When nothing is left: `{ "checkout": { ... }, "done": true }`.
- When a group fails: `{ "checkout": { ... }, "failed": { "mode": "...", "isins": [ ... ], "error": "..." } }`. Its
  lines stay in the cart.
- For a SIP group, `order` is a [SIP](#sip) and `token_url` pays the first instalments.
- Placed lines leave the cart. Lines removed or changed in the cart after the checkout started are not placed.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "Checkout not found" |
| 409 | | "Your previous order is still being placed. Please wait a moment." |
| 409 | | "This checkout was not confirmed with an OTP. Please check out again." |

### GET /checkouts/{id}

Returns the checkout with a live status for each line.

**Auth:** Bearer

**Example response** `200`

```json
{ "checkout": { "id": "chk_625e066d5e39cb52",
                "items": [ { "cart_id": "ci_87c58783cf", "state": "placed",
                             "status": { "label": "Payment pending", "tone": "pending", "ref": "#MF000854" }, "...": "..." } ],
                "...": "..." } }
```

Status labels include "Payment pending", "Order successful", "SIP active" and "Not paid · still in cart".

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "Checkout not found" |

### POST /checkouts/{id}/stop

Stops the checkout. Lines still `queued` become `skipped` and stay in the cart.

**Auth:** Bearer. No body.

**Example response** `200`: `{ "checkout": { ... } }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "Checkout not found" |

---

## Orders

### Who can invest

Every order, SIP, STP, SWP, AutoPay and cart write checks that the account can invest. Redemptions and SWPs skip the
KRA-hold check (it is the investor's own money):

| Status | `error` | When |
|---|---|---|
| 403 | "Please complete your KYC to invest." | Onboarding is not finished |
| 403 | "Your KYC is with the KRA for approval (usually 1–3 working days). You can invest once it is approved." (with `kyc_status`) | A fresh KYC is waiting for the KRA |
| 403 | "Your KYC is on hold at the KRA. New investments are paused until it is fixed; you can still redeem. Please contact support to update your KYC." (the first sentence follows the KRA code; with `kyc_status: "on_hold"`) | The monthly KRA re-check found the PAN not compliant (`user.kra_hold`) |

### POST /invest/lumpsum

Places a one-time purchase and returns its payment link. Takes 5-10 seconds (up to 45). Needs a transaction code
(`"action": "lumpsum"`). The folio already open at that fund house is reused.

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `isin` | string | Yes | Fund ISIN. The fund must allow one-time purchases |
| `amount` | number | Yes | More than 0. At least the fund's minimum, and in its multiples |
| `pay_method` | string | No | `upi` (default) or `netbanking`: the checkout `token_url` opens for that method. Error: "Choose UPI or net banking." |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /invest/lumpsum

{ "isin": "INF109KC1Q80", "amount": 1000, "otp": "196140" }
```

**Example response** `200`

```json
{
  "order": {
    "id": "mfp_69f56e9b7ff44690b7e5fe2716d34cf2", "ref": "#MF000853", "kind": "purchase",
    "isin": "INF109KC1Q80", "fund": { "short_name": "ICICI Pru Flexi Cap fund", "...": "..." }, "fund_to": null,
    "amount": 1000, "units": null, "nav": null, "state": "submitted", "status": "Payment pending",
    "failure_code": null, "folio": null, "payment": { "id": 625, "status": "PENDING" },
    "token_url": "https://.../api/pg/payments/netbanking/ondc?txnId=...",
    "created_at": "2026-11-03 04:10:33", "succeeded_at": null, "...": "..."
  },
  "token_url": "https://.../api/pg/payments/netbanking/ondc?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return"
}
```

Open `token_url` to pay. It works once.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter an amount." |
| 400 | | "Minimum investment in <fund> is ₹N." |
| 400 | | "Amount for <fund> must be in multiples of ₹N." |
| 400 | | "<fund> accepts SIP only. One-time investments are not open for it right now." |
| 400 | | "<fund> isn't taking new investments right now. You can still redeem or switch out units you hold." (no lumpsum and no SIP: `purchase_note` is set; POST /sips gives the same) |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 404 | | "Fund not found" |
| 409 | | "Your first investment with <fund house> (<fund>) is still being processed. Once the fund house allots it, usually by the next working day, you can invest in more <fund house> funds and they will all be in the same folio." (`folio_pending: true`, `pending_order`). One folio per fund house: a folio exists only after the first order is allotted |
| 422 | | "Order could not be placed. ..." (with `order`) |
| 503 | | "Order could not be placed. ... Nothing was debited." (with `order`). Try again in a few minutes |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /invest/redeem

Sells units of a holding. No payment step. Needs a transaction code (`"action": "redeem"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `isin` | string | Yes | Fund ISIN |
| `folio` | string | Yes | 1-40 characters: letters, digits, `/`, space, `-` |
| `amount` | number | Unless `all` | At least ₹100, at most the holding's `redeemable` value |
| `all` | boolean | No | `true` redeems every unit |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /invest/redeem

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 200, "otp": "<code>" }
```

**Example response** `200`

```json
{ "order": { "id": "mfr_...", "kind": "redemption", "amount": 200, "state": "submitted", "status": "Submitted to AMC", "...": "..." } }
```

With `all: true`, `amount` is `null` and `all_units` is `true` until the order is priced.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Folio is required." / "Enter an amount." |
| 400 | | "Minimum redemption is ₹100." |
| 400 | | "There are no redeemable units of this fund in this folio right now." |
| 400 | | "ELSS (tax saver) units are locked for 3 years from purchase, so they can't be redeemed yet. Your first units unlock on DD-MM-YYYY." |
| 400 | | "You can redeem up to ₹N from this fund right now." |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 404 | | "Fund not found" |
| 409 | | "A redemption of all units in this fund is already in progress. ..." |
| 422 | | The order review failed |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /invest/switch

Moves money from one fund to another fund of the same fund house. Needs a transaction code (`"action": "switch"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `from_isin` | string | Yes | Fund held in `folio` |
| `to_isin` | string | Yes | A fund of the same fund house |
| `folio` | string | Yes | As for redeem |
| `amount` | number | Unless `all` | At least the target's switch-in minimum, at most the redeemable value |
| `all` | boolean | No | `true` switches every unit |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /invest/switch

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>", "amount": 300, "otp": "<code>" }
```

**Example response** `200`

```json
{ "order": { "id": "mfs_...", "kind": "switch", "fund_to": { "short_name": "ICICI Pru Liquid Fund", "...": "..." },
             "state": "submitted", "...": "..." } }
```

The order is `successful` once the switch-out is done. `units_in` fills in when the units reach the target fund.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "You can only switch between funds of the same fund house." |
| 400 | | "<target fund> takes switches of at least ₹N." |
| 400 | | "You hold ₹N, less than the ₹M minimum switch into <target fund>." |
| 400 | | "An exit load of up to P% applies to some of your units in this fund and comes out of what reaches <target fund>. Switch at least ₹N so that its ₹M minimum is met." (units still inside the source fund's exit-load period: N = M ÷ (1 − P/100), rounded up) |
| 400 | | "Your ₹N would be about ₹K after an exit load of up to P%, less than the ₹M minimum switch into <target fund>." (`all: true`, same rule) |
| 400 | | "There are no redeemable units of this fund in this folio right now." |
| 400 | | "ELSS (tax saver) units are locked for 3 years from purchase, so they can't be switched yet. Your first units unlock on DD-MM-YYYY." |
| 400 | | "You can switch up to ₹N from this fund right now." |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 404 | | "Fund not found" |
| 409 | | An all-units redemption or switch is already in progress |
| 422 | | The order review failed |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /orders

Lists purchases, redemptions and switches, newest first. SIP instalments are not listed here: they are in each SIP's `installments` (`GET /sips/{id}`), and the orders report lists them too.

**Auth:** Bearer

**Example response** `200`

```json
{ "orders": [ { "id": "mfp_69f56e9b7ff44690b7e5fe2716d34cf2", "kind": "purchase", "amount": 1000,
                "state": "submitted", "status": "Payment pending", "token_url": "https://...", "...": "..." } ] }
```

### GET /orders/{id}

Returns one order.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `id` | path | Yes | `mfp_`, `mfr_` or `mfs_` id |
| `force` | query | No | `1` re-reads the order from the fund platform now |

**Example request**

```http
GET /orders/mfp_69f56e9b7ff44690b7e5fe2716d34cf2?force=1
```

**Example response** `200`

```json
{ "order": { "id": "mfp_69f56e9b7ff44690b7e5fe2716d34cf2", "state": "successful", "status": "Successful",
             "units": 50.7871, "nav": 19.69, "folio": "<folio number>",
             "payment": { "id": 625, "status": "SUCCESS", "reference": "pay_..." }, "token_url": null, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | `validation` | "Order not found" (malformed id) |
| 404 | | "Order not found" (unknown id, or another user's order) |

---

## Portfolio

### GET /portfolio

Returns the investor's holdings and totals.

**Auth:** Bearer

**Example response** `200`

```json
{
  "summary": { "current": 1003.2, "invested": 1000, "gain": 3.2, "gain_pct": 0.32,
               "day_change": 0, "day_change_pct": 0, "xirr": null },
  "holdings": [ { "isin": "INF109KC1Q80", "folio": "<folio>", "fund": { "...": "..." }, "units": 50.7871,
                  "nav": 19.69, "nav_date": "2026-11-02", "current": 1000, "invested": 1000, "redeemable": 1000,
                  "gain": 0, "gain_pct": 0, "...": "..." } ],
  "in_progress": [],
  "incoming": [],
  "external": { "status": "none", "hidden": false, "holdings": [], "...": "..." },
  "combined": { "current": 1003.2, "...": "..." }
}
```

| Field | Description |
|---|---|
| `summary` | Totals over Risips holdings. `xirr` appears once the first investment is a year old |
| `holdings` | One line per fund and folio: `isin, folio, fund, units, nav, nav_date, current, invested, redeemable, gain, gain_pct, avg_nav, xirr, day_change, locked, pending, sip, stp, swp`. `locked` is `true` while an all-units redemption or switch runs. Rupee and % figures are rounded to 2 decimals; `redeemable` is `0` when less than ₹1 can be sold (e.g. locked ELSS units). `units` are as the registrar holds them (3 decimals), so they can differ from an order's 4-decimal units. `exit_load`: `{ text, applies, since, message? }` or `null`; `applies` is `true` when units in this folio were bought inside the fund's exit-load period (since `since`), and `message` is the warning the app shows before a redemption or switch |
| `in_progress` | Paid orders still being processed. Orders waiting for payment are not listed here |
| `incoming` | Funds on their way in that are not holdings yet: `{ isin, fund, orders }` |
| `external` | Imported funds (see [External funds import](#external-funds-import)): `{ status, hidden, source, as_of, imported_at, error, holdings, summary }` |
| `combined` | Totals over Risips and external holdings, whether or not they are hidden (`external.hidden` only tells the app to show `summary` instead). `xirr` is `null` when external funds exist |

---

## AutoPay (mandate)

### GET /mandate

Returns the current AutoPay and the bank account it uses.

**Auth:** Bearer

**Example response** `200`

```json
{ "mandate": null,
  "bank": { "bank_name": "HDFC BANK", "account_masked": "****1195", "ifsc": "HDFC0000240", "branch": "MUMBAI  SANDOZ HOUSE" } }
```

`mandate` is a [Mandate](#mandate) when one exists. An investor can have several mandates on their bank account; this
shows the one that matters, out of all of them:
- the **approved** mandate with the highest limit, so a newer one that was rejected or never authorised doesn't hide it;
- except while the investor is authorising a new mandate with a higher limit (the first 30 minutes after `POST /mandate`):
  then that one, so the app can wait for its outcome;
- with none approved, the newest (being set up, rejected, …).

### POST /mandate

Creates an AutoPay and returns its approval link.

**Auth:** Bearer. Requires an account that can invest. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `type` | string | No | `UPI` for UPI AutoPay. Any other value, or none, creates an eNACH mandate (`E_MANDATE`) |
| `limit` | number | No | Monthly limit. Default ₹1,00,000. A lower value than ₹1,000 is raised to ₹1,000. UPI is capped at ₹1,00,000 |
| `need` | number | No | The monthly amount the AutoPay must cover |

**Example request**

```http
POST /mandate

{ "type": "UPI", "limit": 10000, "need": 1000 }
```

**Example response** `200`

```json
{
  "mandate": { "id": 234, "status": "CREATED", "type": "UPI", "limit": 10000, "umrn": null,
               "valid_from": "2026-11-03", "valid_to": "2056-11-02", "created_at": "2026-11-03T09:41:07+05:30",
               "bank_account_id": 275 },
  "auth_url": "https://.../api/pg/payments/netbanking/billdesk?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/mandate/return"
}
```

- An approved AutoPay that covers `need` is reused: `{ "mandate": { ... }, "auth_url": null }`. Every mandate on the
  investor's bank account counts, not just the newest: a later one that was rejected or never authorised doesn't stop it.
- The last AutoPay this app started, if still `CREATED` with the same `type`, is reused with its original limit, and a new
  `auth_url` is returned.
- Open `auth_url` to approve it, then poll `GET /mandate` until `status` is `APPROVED`.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 403 | | [Cannot invest yet](#who-can-invest) |

---

## SIP

### POST /sip

Starts a monthly SIP. Needs an approved AutoPay that covers the amount, and a transaction code (`"action": "sip"`).
Takes 5-10 seconds (up to 45).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `isin` | string | Yes | Fund ISIN |
| `amount` | number | Yes | Monthly amount. At least the fund's SIP minimum, and in its multiples |
| `day` | number | Yes | Debit day, 1-28 |
| `frequency` | string | No | Only `monthly` |
| `first_now` | boolean | No | `true` makes the first instalment today |
| `first_pay` | string | No | `upi` or `netbanking` pays the first instalment now on the checkout page (`token_url`). Otherwise AutoPay debits it |
| `installments` | number | No | 1-1200. Default 120 (10 years), or the fund's minimum number of instalments if that is higher |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /sip

{ "isin": "INF109KC1Q80", "amount": 1000, "day": 5, "first_now": true, "first_pay": "upi", "otp": "<code>" }
```

**Example response** `200`

```json
{ "sip": { "id": "mfpp_...", "amount": 1000, "state": "awaiting_payment", "status": "Awaiting first payment",
           "installment_day": 5, "next_debit": "...",
           "installments": [ { "id": "mfp_...", "state": "submitted", "upfront": true } ], "...": "..." },
  "token_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

`token_url` is present only with `first_now: true` and `first_pay` `"upi"` or `"netbanking"`. If that payment fails, the SIP is stopped
automatically (`status: "Payment failed"`).

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Enter an amount." / "Choose a date between 1 and 28." / "Only monthly SIPs are available." / "Choose between 1 and 1,200 instalments." |
| 400 | | "Minimum SIP for <fund> is ₹N." / "SIP amount must be in multiples of ₹N." / "SIP is not available for <fund> at this frequency." / "Choose a SIP date between 1 and 28." |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 409 | | "Your first investment with <fund house> (<fund>) is still being processed. Once the fund house allots it, usually by the next working day, you can invest in more <fund house> funds and they will all be in the same folio." (`folio_pending: true`, `pending_order`). One folio per fund house: a folio exists only after the first order is allotted |
| 409 | | "Set up AutoPay first." (`need_mandate: true`) |
| 409 | | "This is above your AutoPay limit of ₹N. Set up a new AutoPay with a higher limit." (`need_mandate: true`, `limit`) |
| 409 | | "This AutoPay cannot be debited. Please set up AutoPay again." (`need_mandate: true`) |
| 422 | | "The fund platform could not register this SIP right now. Nothing was debited; ..." / "The fund house did not accept this SIP: ..." |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /sips

Lists the investor's SIPs.

**Auth:** Bearer

**Example response** `200`

```json
{ "sips": [ { "id": "mfpp_...", "status": "Active", "amount": 1000, "next_debit": "2026-12-05", "...": "..." } ] }
```

### GET /sips/{id}

Returns one [SIP](#sip).

**Auth:** Bearer

**Example response** `200`: `{ "sip": { ... } }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "SIP not found" |

### GET /sips/{id}/pause-plan

What a pause would do, before the investor confirms it. Nothing is sent to the fund platform.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `months` | query | Yes | `1` or `2` |

**Example response** `200`

```json
{ "pause_plan": { "months": 2, "next_debit": "2026-11-14", "still_debited": "2026-11-14",
  "skipped": ["2026-12-14", "2027-01-14"], "resumes_on": "2027-02-14", "from_date": "2026-12-12", "to_date": "2027-01-14",
  "message": "Your ₹500.00 instalment on 14 Nov 2026 is less than 2 days away, so it will still be debited. The pause skips 14 Dec 2026 and 14 Jan 2027, and your SIP resumes on 14 Feb 2027." } }
```

- `still_debited` is the next instalment when it is less than 2 days away (it is debited; the pause starts the month
  after), otherwise `null`. Show `message` on the pause screen.
- `from_date` / `to_date` are what the fund platform receives: `from_date` is 2 days before the first skipped
  instalment (the fund platform refuses a skip whose `from_date` is the instalment date itself).
- Errors: 409 "Only an active SIP can be paused." / "This SIP is already paused." / "This SIP has no upcoming
  instalment to pause.", 400 "A SIP can be paused for up to 2 months.", 404 "SIP not found".

### POST /sips/{id}/pause

Skips the next 1 or 2 monthly instalments. The pause starts at the first instalment at least 2 days away (see
[GET /sips/{id}/pause-plan](#get-sipsidpause-plan), which the response also returns as `pause_plan`). Needs a
transaction code (`"action": "sip_pause"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `months` | number | Yes | `1` or `2` |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /sips/mfpp_.../pause

{ "months": 1, "otp": "<code>" }
```

**Example response** `200`

```json
{ "sip": { "status": "Paused", "pause": { "from": "2026-12-05", "to": "2026-12-05", "resumes_on": "2027-01-05", "...": "..." }, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "A SIP can be paused for up to 2 months." |
| 400 | | "SEBI rules allow skipping at most N instalment(s) in a row for this SIP right now." |
| 404 | | "SIP not found" |
| 409 | | "Only an active SIP can be paused." / "This SIP is already paused." / "This SIP has no upcoming instalment to pause." |
| 422 | | The fund platform refused the pause |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /sips/{id}/resume

Ends a pause. Needs a transaction code (`"action": "sip_resume"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `otp` | string | Yes | Transaction code |

**Example response** `200`

```json
{ "sip": { "status": "Active", "pause": null, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "SIP not found" |
| 409 | | "This SIP is not paused." |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /sips/{id}/modify

Changes the amount or the debit day. A new SIP is created on the same folio and AutoPay, with a new id, and the old SIP
is stopped. The new SIP starts in the month the old one would have debited next, so no month is charged twice. Needs a transaction code
(`"action": "sip_modify"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `amount` | number | Yes | New monthly amount |
| `day` | number | Yes | New debit day, 1-28 |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /sips/mfpp_.../modify

{ "amount": 1500, "day": 10, "otp": "<code>" }
```

**Example response** `200`

```json
{ "sip": { "id": "mfpp_new...", "amount": 1500, "installment_day": 10, "replaces": "mfpp_old...", "...": "..." },
  "old": { "id": "mfpp_old...", "status": "Modified", "replaced_by": "mfpp_new...", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Change the amount or the SIP date first." |
| 404 | | "SIP not found" |
| 409 | | "Only an active SIP can be changed." / "Resume this SIP before changing it." |
| 409 | | AutoPay limit too low (`need_mandate: true`) |
| 422 | | "We couldn't move this SIP to the new date without an extra debit (...)..." |
| 502 | | "Your new SIP is registered, but the old one could not be cancelled. Please cancel it from your SIPs." (with `sip`) |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### POST /sips/{id}/cancel

Stops a SIP. Needs a transaction code (`"action": "sip_cancel"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `code` | string | No | Reason: `amount_not_available`, `investment_returns_not_as_expected`, `amc_support_not_satisfactory`, `exit_load_not_as_expected`, `switch_to_other_scheme`, `fund_manager_changed`, `investment_goal_complete`, `mandate_not_ready`, `invest_later`, `customer_support_not_satisfactory`. Any other value is saved as `invest_later`. `cancellation_code` in the response is this code (FP itself stores `custom_reason` whatever it is sent) |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /sips/mfpp_.../cancel

{ "code": "invest_later", "otp": "<code>" }
```

**Example response** `200`

```json
{ "sip": { "status": "Cancelled", "cancellation_code": "invest_later", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "SIP not found" |
| 409 | | "This SIP has already stopped." |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

---

## STP

A Systematic Transfer Plan moves a fixed amount each month from one fund to another fund of the same fund house.
Monthly only.

### POST /stp

Creates an STP. The first transfer runs on the next `day`. Needs a transaction code (`"action": "stp"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `from_isin` | string | Yes | Fund held in `folio`, which must allow STPs out |
| `to_isin` | string | Yes | A different fund of the same fund house, which must accept STPs |
| `folio` | string | Yes | Folio of the source holding |
| `amount` | number | Yes | Monthly amount, floored to the rupee. At least the fund's STP minimum. The holding must be worth at least one transfer |
| `day` | number | Yes | 1-28 |
| `installments` | number | Yes | 1-1200, and at least the fund's minimum number of transfers |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /stp

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>",
  "amount": 1000, "day": 10, "installments": 6, "otp": "<code>" }
```

**Example response** `200`

```json
{ "stp": { "id": "mfsp_...", "state": "active", "status": "Active", "installment_day": 10,
           "start_date": "2026-11-10", "number_of_installments": 6, "installments": [], "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Choose how many monthly transfers." |
| 400 | | "Choose a different fund to transfer into." |
| 400 | | "An STP can only move money between funds of the same fund house." |
| 400 | | "<fund> does not allow STPs out of it." / "<fund> does not accept STPs." |
| 400 | | "Minimum transfer is ₹N a month." / "Amount must be in multiples of ₹N." |
| 400 | | "An STP needs at least N monthly transfers." |
| 400 | | "Choose a transfer date between 1 and 28." |
| 400 | | "You hold ₹N in this fund, less than one transfer." |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 409 | | An all-units redemption or switch is in progress |
| 422 | | The plan review failed |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /stps

Lists the investor's STPs.

**Auth:** Bearer

**Example response** `200`: `{ "stps": [ { "id": "mfsp_...", "status": "Active", "...": "..." } ] }`

### GET /stps/{id}

Returns one [STP](#stp).

**Auth:** Bearer

**Example response** `200`: `{ "stp": { ... } }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "STP not found" |

### POST /stps/{id}/cancel

Stops an STP. Needs a transaction code (`"action": "stp_cancel"`). An STP that has already stopped is returned without
asking for a code.

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `otp` | string | Yes | Transaction code |

**Example response** `200`

```json
{ "stp": { "status": "Cancelled", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "This STP never started, so there is nothing to cancel. No transfers will run on it." |
| 404 | | "STP not found" |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

---

## SWP

A Systematic Withdrawal Plan redeems a fixed amount each month to the investor's bank account. Monthly only.

### POST /swp

Creates an SWP. Needs a transaction code (`"action": "swp"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `isin` | string | Yes | Fund held in `folio`, which must allow SWPs |
| `folio` | string | Yes | Folio of the holding |
| `amount` | number | Yes | Monthly amount. At least the fund's SWP minimum. The holding must be worth at least one withdrawal |
| `day` | number | Yes | 1-28 |
| `installments` | number | Unless `until` | 1-1200, and at least the fund's minimum number of withdrawals |
| `until` | string | No | `stopped`: runs until stopped |
| `start_offset` | number | No | `0` (default), `1` or `2`: start this month, or 1 or 2 months later |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /swp

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 100, "day": 1, "installments": 3, "start_offset": 0, "otp": "<code>" }
```

**Example response** `200`

```json
{ "swp": { "id": "mfrp_...", "state": "active", "status": "Active", "withdrawals": 3,
           "first_withdrawal": "2026-12-01", "next_withdrawal": "2026-12-01", "until_stopped": false, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Choose how many monthly withdrawals." / "The first withdrawal can start at most 2 months later." |
| 400 | | "<fund> does not allow SWPs." |
| 400 | | "Minimum withdrawal is ₹N a month." / "Amount must be in multiples of ₹N." |
| 400 | | "An SWP needs at least N monthly withdrawals." |
| 400 | | "Choose a withdrawal date between 1 and 28." |
| 400 | | "The first withdrawal date is too close to start later. Pick another date or start this month." |
| 400 | | "You hold ₹N in this fund, less than one withdrawal." |
| 403 | | [Cannot invest yet](#who-can-invest) |
| 409 | | An all-units redemption or switch is in progress |
| 422 | | The plan review failed, or the later start was refused |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /swps

Lists the investor's SWPs.

**Auth:** Bearer

**Example response** `200`: `{ "swps": [ { "id": "mfrp_...", "status": "Active", "...": "..." } ] }`

### GET /swps/{id}

Returns one [SWP](#swp).

**Auth:** Bearer

**Example response** `200`: `{ "swp": { ... } }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 404 | | "SWP not found" |

### POST /swps/{id}/cancel

Stops an SWP. Needs a transaction code (`"action": "swp_cancel"`). The `Cancelled` status can take a second to show.

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `otp` | string | Yes | Transaction code |

**Example response** `200`

```json
{ "swp": { "status": "Cancelled", "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | The SWP never started, so there is nothing to cancel |
| 404 | | "SWP not found" |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

---

## Reports

### GET /reports/{key}

Returns a statement.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `key` | path | Yes | See the table below |
| `fy` | query | No | Start year of the financial year, e.g. `2026` for FY 2026-27. Default: the current year |
| `id` | query | For `contract-note` | An order id, or a SIP or SWP instalment id |

| `key` | Report |
|---|---|
| `capital-gains` | Realised gains, short and long term. One row per FIFO lot with its order ref; the sale value is what the investor received for that redemption / switch |
| `holdings` | Holdings by folio |
| `dividends` | Dividend (IDCW) credits |
| `elss` | ELSS (tax saver) investments |
| `orders` | Order history |
| `contract-notes` | Completed orders, each linking to its contract note |
| `contract-note` | One order's contract note (units exactly as allotted, the same figure as the order) |
| `balance` | Risips balance statement (always empty) |
| `gst` | GST invoices (always empty) |

**Example request**

```http
GET /reports/orders?fy=2026
```

**Example response** `200`

```json
{
  "key": "orders", "title": "Order history", "period": "FY 2026-27",
  "fy": { "value": 2026, "options": [ { "value": 2026, "label": "FY 2026-27" }, { "value": 2025, "label": "FY 2025-26" } ] },
  "summary": [ { "label": "Orders placed", "value": "1" }, { "label": "Invested", "value": "₹0.00" } ],
  "sections": [ { "title": "Orders", "columns": ["..."], "rows": [ { "title": "...", "sub": "...", "value": "...", "cells": ["..."] } ] } ],
  "empty": null,
  "notes": ["..."],
  "investor": { "name": "Qa Tester", "pan": "*******51Q" },
  "generated_at": "..."
}
```

With no data, `rows` is empty and `empty` holds a sentence such as "You don't hold any mutual fund units yet."

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Choose a financial year." |
| 400 | | "A contract note is available once the order is successful." / "No contract note for this order." |
| 404 | | "Report not found" |
| 404 | | "Order not found" (`contract-note` without a valid `id`) |

### GET /reports/{key}/html

Returns the same report as a printable HTML page. The app turns it into a PDF.

**Auth:** Bearer. Same parameters and errors as `GET /reports/{key}`.

**Example response** `200`

```json
{ "html": "<!doctype html>..." }
```

---

## Notifications

### GET /notifications

Returns the notification list, plus an optional pop-up (`modal`) and `banner`. Up to 50 items. Every `at` is ISO 8601 in UTC (`…Z`).

**Auth:** Bearer

**Example response** `200`

```json
{
  "notifications": [
    { "id": "promo-festive-2026", "tag": "REWARDS", "title": "Festive rewards on SIPs are live",
      "body": "Start or step up a SIP this season to unlock exclusive rewards.", "at": "2026-09-20T03:30:00Z",
      "route": "/collection/foryou?mode=sip", "read": true }
  ],
  "unread": 0,
  "modal": { "id": "promo-festive-2026", "kind": "promo", "title": "Invest this festive season, unlock rewards", "...": "..." },
  "banner": { "id": "festive-2026", "title": "Festive rewards on SIPs", "...": "..." }
}
```

- Tags: `ORDER`, `SIP`, `KYC`, `ACTION NEEDED`, `SECURITY`, `MARKET`, `REWARDS`.
- Market insights appear only when the `market_insights` setting is on (off by default): the market-holiday notice
  (`MARKET`, "Market holiday: …"), promotions (`REWARDS`), announcements, and the `modal` (holiday or promo) and `banner`.
  With it off, `modal` and `banner` are `null`. SIP-due reminders appear only when `sip_reminders` is on.
- A fresh-KYC investor gets "Your KYC is with the KRA" and later "Your KYC is approved".

### POST /notifications/read

Marks notifications as read.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `ids` | array of string or number | No | Up to 500. Without `ids`, every current notification is marked read |

**Example request**

```http
POST /notifications/read

{ "ids": ["promo-festive-2026"] }
```

**Example response** `200`: `{ "ok": true }`

### POST /notifications/seen

Marks a pop-up (`modal`) as shown.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `id` | string | Yes | The modal's `id` |

**Example response** `200`: `{ "ok": true }`

---

## Settings

### GET /settings/notifications

Returns notification preferences.

**Auth:** Bearer

**Example response** `200`

```json
{ "prefs": { "sip_reminders": true, "portfolio_summary": true, "goals_reviews": true, "market_insights": false,
             "push": true, "whatsapp": true, "email": false } }
```

The values shown are the defaults.

### PUT /settings/notifications

Changes notification preferences. Send any of the keys above with `true` or `false`.

**Auth:** Bearer

**Example request**

```http
PUT /settings/notifications

{ "market_insights": true }
```

**Example response** `200`: `{ "prefs": { ..., "market_insights": true } }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | `Unknown preference "foo".` |
| 400 | `validation` | `"push" must be true or false.` |

### POST /settings/pin/check

Checks the current PIN before a change.

**Auth:** Bearer. Limit: 10 PIN attempts per 15 minutes.

| Field | Type | Required | Rules |
|---|---|---|---|
| `current` | string | Yes | 4 digits |

**Example response** `200`: `{ "ok": true }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `wrong_pin` | "Incorrect PIN. 4 attempts left." (`attempts_left`) |
| 401 | `pin_locked` | 5 wrong PINs in a row. The session ends |

### POST /settings/pin/change

Changes the PIN.

**Auth:** Bearer. Limit: 10 PIN attempts per 15 minutes.

| Field | Type | Required | Rules |
|---|---|---|---|
| `current` | string | Yes | The current PIN |
| `pin` | string | Yes | New 4-digit PIN, different from the current one |

**Example request**

```http
POST /settings/pin/change

{ "current": "1357", "pin": "2580" }
```

**Example response** `200`

```json
{ "ok": true, "user": { "pin_set": true, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Choose a PIN different from your current one." |
| 400 | `wrong_pin` | "Incorrect PIN. N attempts left." (`attempts_left`) |
| 401 | `pin_locked` | 5 wrong PINs in a row. The session ends |

### GET /settings/devices

Lists signed-in devices. `name` and `os` come from the `x-device-name` and `x-device-os` headers.

**Auth:** Bearer

**Example response** `200`

```json
{
  "current": { "id": 414, "name": "Postman", "os": "windows", "signed_in_at": "2026-11-12T07:08:00Z", "last_active_at": "2026-11-12T07:15:00Z", "current": true },
  "others": [ { "id": 415, "name": "Pixel 8", "os": "android", "signed_in_at": "...", "last_active_at": "...", "current": false } ]
}
```

### POST /settings/devices/{id}/logout

Signs out another device.

**Auth:** Bearer

| Parameter | In | Required | Rules |
|---|---|---|---|
| `id` | path | Yes | Device id, 1-12 digits |

**Example response** `200`: `{ "ok": true }`

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Use Logout to sign out of this device." |
| 404 | | "That device is already signed out." |

### POST /settings/devices/logout-others

Signs out every other device.

**Auth:** Bearer. No body.

**Example response** `200`

```json
{ "ok": true, "signed_out": 1 }
```

### POST /profile/nominee

Adds a nominee after onboarding, for an investor who opted out during onboarding. The nominee is added to folios opened
from now on. Needs a transaction code (`"action": "nominee"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

Fields: as for [POST /onb/nominee](#post-onbnominee), except that `skip` is not allowed.

**Example request**

```http
POST /profile/nominee

{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "<code>" }
```

**Example response** `200`

```json
{ "user": { "nominee": { "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "*******34E" },
            "nominee_added_at": "2026-11-03T04:23:26.407Z", "nominee_skipped": false, "...": "..." } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Enter your nominee’s details." (`skip` sent) |
| 400 | | The same nominee checks as onboarding, e.g. "Nominee PAN must be different from yours." |
| 409 | | "Please finish setting up your account first." |
| 409 | | "You already have a nominee. Contact support to change it." |
| 502 | | Fund platform error |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

### GET /settings/deletion

Shows whether the account can be deleted, and any open deletion request.

**Auth:** Bearer

**Example response** `200`

```json
{ "checks": { "active_plans": 0, "pending_orders": 0, "ok": true }, "request": null, "grace_days": 30 }
```

`pending_orders` counts orders being processed. Orders only waiting for payment are not counted.

### POST /settings/deletion

Requests account deletion. Every session is signed out, and the account is erased after 30 days. Signing in again
within 30 days cancels the request. Needs a transaction code (`"action": "delete_account"`).

**Auth:** Bearer. Limit: 30 money requests per 10 minutes per user.

| Field | Type | Required | Rules |
|---|---|---|---|
| `reason` | string \| null | No | `not_using`, `other_platform`, `privacy`, `other` |
| `otp` | string | Yes | Transaction code |

**Example request**

```http
POST /settings/deletion

{ "reason": "not_using", "otp": "<code>" }
```

**Example response** `200`

```json
{ "ok": true, "request": { "status": "requested", "requested_at": "2026-11-03T04:15:15.856Z",
                           "purge_after": "2026-12-03T04:15:15.856Z" } }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Unknown reason." |
| 409 | | "Stop your active SIPs, STPs and SWPs before deleting your account." (with `checks`) |
| 409 | | "Wait for your pending orders to complete before deleting your account." (with `checks`) |
| 400 / 429 | | [OTP errors](#errors-when-an-action-checks-its-otp) |

---

## External funds import

Imports mutual fund holdings the investor holds outside Risips, through MF Central. The PAN step must be done;
onboarding does not need to be finished. On the sandbox, MF Central is replaced by a stand-in that returns **sample**
holdings (real fund names, made-up units, `sample: true`).

### Status object

Returned by most endpoints in this module.

| Field | Description |
|---|---|
| `status` | `none`, `otp_sent`, `importing`, `ready` or `failed` |
| `provider` | `mfcentral`, `sandbox` or `null` |
| `available` | Import can be used |
| `pan`, `pan_masked`, `pan_locked` | `pan` is `null` when the KYC PAN is on file. `pan_masked` shows the first 2 and last 3 characters, e.g. `TG*****51Q` |
| `mobile`, `email` | Contacts used for the MF Central code |
| `request` | `{ started_at, qr_at, sent_to }` or `null` |
| `error` | Failure text or `null` |
| `imported_at`, `as_of` | Import time and statement date |
| `count` | Number of imported funds |
| `sample` | `true` for sandbox sample data |
| `hidden` | Imported funds are hidden from the portfolio |
| `consent_text` | The consent text shown to the investor |

### GET /external

Returns the status. While importing, it checks for the statement.

**Auth:** Bearer

**Example response** `200`

```json
{ "status": "none", "provider": "sandbox", "available": true,
  "pan": null, "pan_masked": "TG*****51Q", "pan_locked": true,
  "mobile": "7926224018", "email": "qa.tester01@example.com",
  "request": null, "error": null, "imported_at": null, "as_of": null, "count": 0, "sample": false, "hidden": false,
  "consent_text": "I authorise Risips to fetch my mutual fund holdings from MF Central ..." }
```

### POST /external/start

Starts an import and returns the MF Central page link. MF Central sends the investor a code.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `via` | string | No | `mobile` (default) or `email` |
| `consent` | boolean | Yes | Must be `true` |
| `pan` | string | No | Max 20 characters. The KYC PAN is used when on file |
| `mobile`, `email` | string | No | Default: the account's own |

**Example request**

```http
POST /external/start

{ "via": "mobile", "consent": true }
```

**Example response** `200`

```json
{ "journey_url": "https://risips-demo-api.azurewebsites.net/mfc-sandbox/journey/0974498...", "provider": "sandbox",
  "sent_to": "+91 ******4018" }
```

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "Please give your consent to fetch your holdings from MF Central." |
| 400 | | "Please enter a valid PAN, e.g. ABCDE1234F." / an invalid mobile or email |
| 400 / 502 | | MF Central refused the request |
| 503 | | "Importing from MF Central isn’t switched on yet. Please try again later." |

On the `journey_url` page, enter the OTP (the sandbox page shows it), choose the plan type and the summary, tap
**Generate QR code**, then **Download QR**. In the app, the page hands the QR to the app. In Postman, copy the QR image
address (a `data:image/png;base64,...` text) and send it with `POST /external/qr`.

### POST /external/qr

Sends the QR code from the MF Central page. The statement is ready about 20 seconds later on the sandbox.

**Auth:** Bearer. Request body up to 12 MB.

| Field | Type | Required | Rules |
|---|---|---|---|
| `qr` | string | Yes | Base64 PNG or data URL |
| `req_id` | string | No | Request id from the MF Central page |
| `extent` | string | No | e.g. `SUMMARY` |

**Example request**

```http
POST /external/qr

{ "qr": "data:image/png;base64,iVBORw0KGgo...", "extent": "SUMMARY" }
```

**Example response** `200`: the status object with `"status": "importing"` and `request.qr_at` set. When the import
finishes, `GET /external` shows `"status": "ready"` and the funds appear in `GET /portfolio` under `external.holdings`.

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | `validation` | "We didn’t receive the QR code. Tap Download QR on the MF Central page, then Continue." |
| 400 / 502 | | The QR was refused |
| 409 | | "Please start the import again." |

### POST /external/visibility

Hides or shows imported funds in the portfolio.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `hidden` | boolean | Yes | |

**Example response** `200`: `{ "ok": true, "hidden": true }`

### POST /external/cancel

Leaves an unfinished import. Returns the status object.

**Auth:** Bearer. No body.

### DELETE /external

Removes every imported fund. Returns the status object with `"status": "none"` and `"count": 0`.

**Auth:** Bearer

### Sandbox stand-in helpers

On the sandbox only, the stand-in page can also be driven by API. These calls need no token.

| Endpoint | Body | Response |
|---|---|---|
| `GET /mfc-sandbox/journey/{token}` | | The stand-in HTML page (the `journey_url`) |
| `POST /mfc-sandbox/api/verify` | `{ "token": "<last part of journey_url>", "otp": "<code shown on the page>" }` | `{ "ok": true }` |
| `POST /mfc-sandbox/api/generate-qr` | `{ "token": "...", "planType": "BOTH", "extentOfData": "SUMMARY" }` | `{ "qr": "<base64>" }`. Send it to `POST /external/qr` |

---

## Assistant

An in-app helper that answers questions using the investor's own data. It never recommends a specific fund.

### GET /assistant

Returns the greeting and suggested questions.

**Auth:** Bearer

**Example response** `200`

```json
{ "name": "Qa",
  "intro": "I can pull up your holdings, SIPs, orders and KYC status, and explain how mutual funds work.",
  "groups": [ { "title": "YOUR ACCOUNT", "chips": ["How are my SIPs doing?", "When is my next SIP debit?", "Track my last order", "What’s my KYC status?"] },
              { "title": "STATEMENTS", "chips": ["..."] }, { "title": "LEARN", "chips": ["..."] } ] }
```

### POST /assistant

Asks a question, or requests an advisor callback.

**Auth:** Bearer

| Field | Type | Required | Rules |
|---|---|---|---|
| `text` | string | Unless `action` | Max 500 characters |
| `action` | string | No | `book_call`: requests an advisor callback (one open request at a time) |

**Example request**

```http
POST /assistant

{ "text": "What is my SIP status?" }
```

**Example response** `200`

```json
{ "messages": [ { "kind": "text", "text": "You don’t have a running SIP yet. Pick a fund in Explore and choose “Start SIP” to set one up." } ],
  "chips": ["SIP vs lump sum", "Explain risk levels"],
  "actions": [ { "label": "Explore funds", "route": "/(tabs)/explore", "kind": "primary" } ] }
```

`messages[].kind` is `text`, `sips` (with `items`) or `order` (with `order`).

**Errors**

| Status | `reason` | `error` |
|---|---|---|
| 400 | | "Type a question." |
| 400 | `validation` | "Keep the question under 500 characters." |
