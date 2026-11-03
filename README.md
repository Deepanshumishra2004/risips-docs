# Risips API

The Risips API is the backend behind the Risips mutual fund app for Android and iOS. Investors use it to create an
account, complete KYC onboarding, browse funds, and invest: one-time purchases, SIPs, redemptions, switches, STPs and
SWPs. The API handles the fund platform, the KYC services and the payment pages for the app. Every action in the app
maps to one or more requests described here, so testers and integrators can run every journey without the app.

The full list of endpoints, fields and errors is in **[API_REFERENCE.md](API_REFERENCE.md)**.

## Contents

1. [Environments](#1-environments)
2. [Quick start](#2-quick-start)
3. [Authentication](#3-authentication)
4. [Conventions](#4-conventions)
5. [Errors](#5-errors)
6. [Limits](#6-limits)
7. [API flows](#7-api-flows)
   - [7a. Sign up and log in](#7a-sign-up-and-log-in)
   - [7b. Onboarding: PAN with existing KYC](#7b-onboarding-pan-with-existing-kyc)
   - [7c. Onboarding: fresh KYC](#7c-onboarding-fresh-kyc)
   - [7d. One-time investment, redeem and switch](#7d-one-time-investment-redeem-and-switch)
   - [7e. Cart checkout](#7e-cart-checkout)
   - [7f. SIP with AutoPay](#7f-sip-with-autopay)
   - [7g. STP and SWP](#7g-stp-and-swp)
8. [Hosted pages (browser steps)](#8-hosted-pages-browser-steps)
9. [Sandbox test data](#9-sandbox-test-data)

---

## 1. Environments

| Environment | Base URL | Use |
|---|---|---|
| Sandbox | `https://risips-demo-api.azurewebsites.net` | Testing. Fake money and simulated KYC. One-time codes are returned in the response as `dev_otp`, so no SMS or email is needed |
| Production | Not published | Real investors and real money. Not for testing |

Check that you are on the sandbox before you start:

```http
GET https://risips-demo-api.azurewebsites.net/health
```

```json
{ "ok": true, "fp_env": "sandbox", "demo": true, "schemes": 35 }
```

`fp_env` must be `"sandbox"` and `demo` must be `true`. If `fp_env` is `"production"`, stop: that server moves real
money.

On the sandbox, use only the values in [Sandbox test data](#9-sandbox-test-data). Never enter a real person's PAN,
bank account or Aadhaar.

---

## 2. Quick start

**Step 1. Ask for a sign-in code.**

```http
POST /auth/email/start
Content-Type: application/json

{ "email": "qa.tester01@example.com" }
```

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "454804" }
```

**Step 2. Verify the code.** The response contains your session `token`.

```http
POST /auth/email/verify
Content-Type: application/json

{ "email": "qa.tester01@example.com", "code": "454804" }
```

```json
{ "token": "<TOKEN>", "is_new": true, "user": { "id": 345, "next_step": "mobile", "...": "..." } }
```

**Step 3. Send the token on every other request.**

```http
Authorization: Bearer <TOKEN>
```

**Step 4. Check who you are.**

```http
GET /me
Authorization: Bearer <TOKEN>
```

```json
{ "user": { "id": 345, "email": "qa.tester01@example.com", "onboarded": false, "next_step": "mobile", "...": "..." } }
```

**Step 5. Follow `next_step`.** `user.next_step` names the next onboarding step. Call the matching endpoint (see the
[table in 7b](#7b-onboarding-pan-with-existing-kyc)) until `next_step` is `"home"`. The account can then invest.

---

## 3. Authentication

### Email code and token

Risips has no passwords. An investor signs in with a one-time code sent to their email address.

1. `POST /auth/email/start` with the email. A 6-digit code is sent. It is valid for 120 seconds.
2. `POST /auth/email/verify` with the email and the code. The response contains a `token`.
3. Send `Authorization: Bearer <token>` on every request except `/health`, `/auth/email/start` and
   `/auth/email/verify`.

Sign-up and sign-in are the same two calls. A new email creates a new account and returns `"is_new": true`.

A missing, wrong or ended token returns:

```json
{ "error": "Please sign in again." }
```

with HTTP `401`. Sign in again to get a new token.

### Session lifetime

A session ends when any of these happens:

- 30 days pass without a request on that token.
- 180 days pass since sign-in.
- The investor logs out (`POST /auth/logout`), or signs the device out from Settings.
- The PIN is entered wrong 5 times in a row.
- The account deletion is requested (every session ends).

### PIN

After the mobile number is verified, the investor sets a 4-digit PIN (`POST /auth/pin/set`). The app checks it with
`POST /auth/pin/verify` each time it is opened. The PIN does not replace the token; it unlocks the app on a device
that already has one. Five wrong PINs in a row end the session (`401`, `reason: "pin_locked"`). The investor then signs
in again by email. The PIN stays the same.

### Transaction codes (2FA)

Every money action needs a second one-time code, sent to the registered mobile and email. This covers purchases,
redemptions, switches, SIPs, STPs, SWPs, cart checkout, the nominee step and account deletion.

1. `POST /txn/otp { "action": "<action>" }`. The sandbox returns the code as `dev_otp`.
2. Send the request with `"otp": "<code>"` in the body.

A code works for one action only, works once, and lasts 5 minutes. A new code for the same action replaces the old
one. The server checks every other field before the code, so a mistake in the amount does not use up the code. The
action names are listed in [API_REFERENCE.md](API_REFERENCE.md#transaction-otp).

### Postman setup

**1. Create an environment** named `Risips sandbox` with these variables:

| Variable | Initial value | Purpose |
|---|---|---|
| `baseUrl` | `https://risips-demo-api.azurewebsites.net` | The server |
| `email` | `qa.<something>@example.com` | Test email for this run |
| `token` | empty | Filled in after sign-in |
| `otp` | empty | Filled in from `dev_otp` |

Select the environment before you send requests.

**2. Set the token once.** On the collection, open **Authorization**, choose **Bearer Token**, and enter `{{token}}`.
Set each request to **Inherit auth from parent**.

**3. Save the token and code automatically.** Add this to the collection's **Scripts > Post-response** tab (older
versions: **Tests**):

```js
// Save the session token and any sandbox one-time code.
let j = {};
try { j = pm.response.json(); } catch (e) {}
if (j.token)   { pm.environment.set("token", j.token); }
if (j.dev_otp) { pm.environment.set("otp", j.dev_otp); }
```

You can then write `"code": "{{otp}}"` or `"otp": "{{otp}}"` in a request body.

**4. Set the request timeout** to at least 60 seconds (**Settings > General > Request timeout**). Some calls take
several seconds (see [Slow calls](#slow-calls)).

**Optional headers:** `x-device-name: Postman` and `x-device-os: windows`. They appear in the active devices list in
Settings.

---

## 4. Conventions

| Topic | Rule |
|---|---|
| Format | Every request and response body is JSON. Send `Content-Type: application/json` on every request with a body |
| Unknown fields | Extra fields in a request body are ignored |
| Dates | `YYYY-MM-DD`, for example `1992-04-15`. Times are ISO 8601 |
| Amounts | Indian rupees, as numbers, for example `1000` for ₹1,000. Order amounts are whole rupees |
| Ids | Prefixed strings: `mfp_` purchase, `mfr_` redemption, `mfs_` switch, `mfpp_` SIP, `mfsp_` STP, `mfrp_` SWP, `ci_` cart line, `chk_` checkout |
| PAN | Never returned in full. `pan_masked` shows the last 3 characters, for example `*******51Q` |
| Bank account | Never returned in full. `account_masked` shows the last 4 digits, for example `**********1195` |
| Aadhaar | Masked as `XXXX XXXX 1234` |
| Mobile | Stored as 10 digits. `+91`, a leading `0`, spaces and dashes are removed |
| Request id | Every response has an `X-Request-Id` header. Include it, with the time, in every bug report |

---

## 5. Errors

### Error format

Every error has the same shape:

```json
{ "error": "Message for the investor", "reason": "machine_code" }
```

- `error` is always present. It is the text the app shows.
- `reason` is present on some errors (see the table below).
- Some errors add helpful fields, for example `attempts_left`, `retry_after`, `next_step` or `need_mandate`.

A validation error lists every bad field under `fields`. `error` holds the first problem:

```json
{
  "error": "Enter a valid individual PAN (e.g. ABCPX1234K).",
  "reason": "validation",
  "fields": {
    "pan": "Enter a valid individual PAN (e.g. ABCPX1234K).",
    "dob": "Enter a valid date of birth (YYYY-MM-DD)."
  }
}
```

### HTTP status codes

| Status | Meaning |
|---|---|
| `200` | Success |
| `400` | The request is wrong: a bad field, a rule that is not met, or a wrong code. Read `error` |
| `401` | `"Please sign in again."`: the token is missing, wrong or ended. Also used for `pin_locked` |
| `403` | The account cannot invest yet: onboarding is not finished, or a fresh KYC is waiting for KRA approval |
| `404` | The endpoint, fund, order or plan does not exist, or belongs to another user. A malformed id in the URL also returns `404` with `reason: "validation"` |
| `409` | The action does not fit the current state, for example "Your account is already set up." |
| `413` | `"The request is too large."` (an image over the size limit) |
| `422` | A verification or review said no: PAN, bank account, KYC form or order review |
| `429` | Too many requests or too many wrong codes. Wait and try again |
| `500` | `"Something went wrong. Please try again."`. Report it with the `X-Request-Id` |
| `502` | The fund platform or a KYC service returned an error |
| `503` | `"Can’t reach the fund platform right now. Please try again in a minute."`. Try again later |

A body that is not valid JSON returns `400` `"The request body is not valid JSON."`.

### Reason codes

| `reason` | Status | Meaning | What to do |
|---|---|---|---|
| `validation` | 400 (404 for ids in the URL) | A field is missing or invalid | Read `error` and `fields`, fix the input |
| `otp_required` | 400 | A money action was sent without `otp` | Call `POST /txn/otp`, then resend with `otp` |
| `wrong` | 400 | The code is wrong. `attempts_left` shows tries left on this code | Enter the right code |
| `expired` | 400 | The code is too old (sign-in and mobile: 2 minutes; transaction: 5 minutes) | Ask for a new code |
| `missing` | 400 | There is no open code: it was never sent, was already used, or was replaced | Ask for a new code |
| `locked` | 400 | 5 wrong tries on this code | Ask for a new code |
| `rate_limited` | 429 | Too many requests or too many wrong codes. `retry_after` gives the seconds to wait, when present | Wait, then try again |
| `mobile_taken` | 409 | The mobile number is verified on another account | Use a different number |
| `pin_locked` | 401 | 5 wrong PINs in a row. The session has ended | Sign in again by email |
| `wrong_pin` | 400 | The current PIN entered in Settings is wrong | Enter the right PIN |

### Other error fields

| Field | Meaning |
|---|---|
| `attempts_left` | Tries left on this code or PIN |
| `retry_after` | Seconds to wait before trying again |
| `next_step` | The onboarding step that must be done first |
| `need_mandate` | AutoPay must be set up (or set up with a higher limit) first |
| `need`, `limit` | The monthly amount AutoPay must cover, and the current AutoPay limit |
| `checks`, `pv_id` | Details of the PAN check (PAN always masked) |
| `bank_status`, `code` | Details of the bank account check |
| `kyc_status` | The KYC state, for example `with_kra` |
| `cart_id` | The cart line that has a problem |
| `order`, `sip`, `checks` | The object the error refers to |

---

## 6. Limits

The sandbox has the same limits as production. They are the most common cause of an unexpected `429` during testing.

### One-time code rules

These apply to sign-in codes, mobile codes and transaction codes, for each email, mobile or action.

| Rule | Response |
|---|---|
| One new code every 30 seconds | `429` "A code was sent just now. Please try again in N seconds." with `retry_after` |
| At most 20 codes per hour | `429`, `reason: "rate_limited"` |
| 5 wrong tries on one code | The 5th wrong try says "No attempts left: request a new code." (`attempts_left: 0`). Any later try returns `400`, `reason: "locked"` |
| 10 wrong codes in one hour, or 20 in one day, across all codes | `429` "Too many wrong codes. Please try again in N minutes." with `retry_after`. During this block no new code is sent |
| Sign-in and mobile codes last 120 seconds; transaction codes last 300 seconds | `400`, `reason: "expired"` |

Use a separate, throw-away email when you test the wrong-code block. That email stays blocked for about an hour.

### Request limits

| Limit | Applies to |
|---|---|
| 300 requests per minute per network | Every endpoint |
| 20 sign-in code requests per hour per network, and per email | `POST /auth/email/start` |
| 60 code checks per hour per network | `POST /auth/email/verify` |
| 60 code requests per hour per user | `POST /auth/mobile/start`, `POST /txn/otp`, `POST /external/start` |
| 60 mobile code checks per hour per user | `POST /auth/mobile/verify` |
| 10 PIN attempts per 15 minutes per user | PIN set, verify, check and change |
| 30 per hour per user | PAN check, bank check and every fresh-KYC call |
| **30 per 10 minutes per user** | Every order, SIP, STP, SWP and AutoPay call, cart checkout, account deletion, and adding a nominee after onboarding. **Failed calls count too** |

Over a request limit, the API returns:

```json
{ "error": "Too many attempts. Please wait a few minutes and try again.", "reason": "rate_limited" }
```

These responses have no `retry_after`. The `RateLimit` response header shows the seconds left (`t=`). Negative tests
on orders use up the money limit quickly: after about 30 order attempts in 10 minutes, wait 10 minutes.

### Slow calls

| Call | Usual time | Maximum |
|---|---|---|
| `POST /onb/pan` | 1-7 s | 25 s |
| `POST /onb/bank` | about 2 s | 20 s |
| `POST /onb/kyc/esign` | a few seconds | 10 s |
| `POST /invest/lumpsum`, `POST /sip`, `POST /stp`, `POST /swp` | 5-10 s | 45 s |

---

## 7. API flows

Each flow lists the requests in order. All examples use the sandbox base URL. Every request except the first two of
flow 7a needs `Authorization: Bearer <token>`. Responses are trimmed; `"...": "..."` marks more fields.

### 7a. Sign up and log in

Order: email code, mobile number, PIN.

**Step 1. Ask for an email code**

```http
POST /auth/email/start

{ "email": "qa.tester01@example.com" }
```

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "454804" }
```

- `existing` is `true` when the email already has an account.
- The email must end in `.com`, `.in`, `.org`, `.co`, `.net`, `.edu`, `.me` or `.uk`. Capitals and spaces around it are
  ignored.

Next: step 2.

**Step 2. Verify the email code**

```http
POST /auth/email/verify

{ "email": "qa.tester01@example.com", "code": "454804" }
```

```json
{
  "token": "<TOKEN>",
  "is_new": true,
  "user": { "id": 345, "email": "qa.tester01@example.com", "mobile": null, "pin_set": false,
            "onboarded": false, "next_step": "mobile", "...": "..." }
}
```

- Save `token`.
- A wrong code returns `400` `{ "error": "Incorrect code. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }`.
- If the account had asked to be deleted, signing in cancels the deletion and the response adds
  `"deletion_cancelled": true`.

Next: `next_step` is `"mobile"`.

**Step 3. Ask for a mobile code**

```http
POST /auth/mobile/start

{ "mobile": "+91 79262 24018" }
```

```json
{ "ok": true, "expires_in": 120, "code_length": 6, "dev_otp": "623956" }
```

- Spaces, dashes, `+91` and a leading `0` are removed, and the last 10 digits are kept. `+91 79262 24018`,
  `+917926224018`, `07926224018` and `7926224018` are the same number.
- The number must start with 6, 7, 8 or 9, and must not be one digit repeated.
- **One mobile number per account.** A number already verified on another account returns `409`
  `{ "error": "This mobile number is already registered with another Risips account. Please use a different number.", "reason": "mobile_taken" }`,
  in any format. No code is sent.

Next: step 4.

**Step 4. Verify the mobile code**

```http
POST /auth/mobile/verify

{ "mobile": "07926224018", "code": "623956", "bank_consent": true }
```

```json
{ "user": { "mobile": "7926224018", "next_step": "pin", "...": "..." } }
```

- `mobile` must be the same number the code was sent to (any format).
- `bank_consent` is optional.

Next: `next_step` is `"pin"`.

**Step 5. Set the PIN**

```http
POST /auth/pin/set

{ "pin": "1357" }
```

```json
{ "user": { "pin_set": true, "next_step": "pan", "...": "..." } }
```

The PIN is exactly 4 digits.

Next: onboarding, `next_step` is `"pan"` (flow 7b or 7c).

**Returning user.** Sign in with steps 1-2 (`is_new: false`). The app then unlocks with:

```http
POST /auth/pin/verify

{ "pin": "1357" }
```

```json
{ "ok": true }
```

A wrong PIN returns `400` "Incorrect PIN. 4 attempts left." with `attempts_left`. The 5th wrong PIN in a row returns
`401` with `reason: "pin_locked"` and ends the session.

**Log out.** `POST /auth/logout` returns `{ "ok": true }`. The token stops working.

---

### 7b. Onboarding: PAN with existing KYC

Use this flow when the PAN already has a KYC record at the KRA (the common case). On the sandbox, use a PAN like
`XXXPX3751X`. No browser step is needed.

Every onboarding response returns `user.next_step`. Use this table to pick the next call:

| `next_step` | Call |
|---|---|
| `mobile` | `POST /auth/mobile/start`, then `POST /auth/mobile/verify` |
| `pin` | `POST /auth/pin/set` |
| `pan` | `POST /onb/pan` |
| `pan-confirm` | `POST /onb/pan/confirm` (or `POST /onb/pan/reset` to edit the PAN) |
| `occupation` | `POST /onb/profile` with `occupation` |
| `income` | `POST /onb/profile` with `income_slab` |
| `marital` | `POST /onb/profile` with `marital_status` |
| `father` | `POST /onb/profile` with `father_name` |
| `address` | `POST /onb/address` (optional lookup: `GET /onb/pincode/{pin}`) |
| `aadhaar` | Fresh KYC only: `POST /onb/kyc/start`, DigiLocker, `POST /onb/kyc/refresh` |
| `bank` | `POST /onb/bank` (optional lookup: `GET /onb/ifsc/{ifsc}`) |
| `signature-consent` | `POST /onb/declarations` (fresh KYC also `POST /onb/signature`) |
| `kyc-esign` | Fresh KYC only: `POST /onb/kyc/esign`, e-sign, `POST /onb/kyc/refresh` |
| `nominee` | `POST /txn/otp` with `"action": "nominee"`, then `POST /onb/nominee` |
| `esign` | `POST /onb/esign` |
| `home` | Onboarding is complete. The account can invest |

For a PAN with existing KYC the order is: `pan`, `pan-confirm`, `occupation`, `income`, `marital`, `father`,
`address`, `bank`, `signature-consent`, `nominee`, `esign`, `home`.

**Step 1. PAN, name and date of birth**

Checks the PAN, name and date of birth, and the PAN's KYC status at the KRA.

```http
POST /onb/pan

{ "pan": "TGVPS3751Q", "name": "Qa Tester", "dob": "1992-04-15" }
```

```json
{
  "user": { "name": "Qa Tester", "pan_masked": "*******51Q", "kyc_compliant": true,
            "kyc_fresh": false, "kyc_status": "compliant", "next_step": "pan-confirm", "...": "..." },
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

- The 4th character of the PAN must be `P` (an individual). The investor must be 18 or older.
- A failed check returns `422` with `checks` and `pv_id`. See [PAN test data](#pan).

Next: `pan-confirm`.

**Step 2. Confirm the PAN**

```http
POST /onb/pan/confirm
```

No body. Returns `{ "user": { "next_step": "occupation", ... } }`. After this the PAN cannot be changed.

To edit the PAN instead (before confirming), call `POST /onb/pan/reset`. It clears the PAN, name, date of birth and
KYC state, and `next_step` returns to `"pan"`.

**Step 3. Profile**

Send one field per call (as the app does) or all at once.

```http
POST /onb/profile

{ "occupation": "private_sector_service", "income_slab": "above_5lakh_upto_10lakh",
  "marital_status": "single", "father_name": "Ramesh Kumar" }
```

Returns `{ "user": { "next_step": "address", ... } }`. Allowed values are in
[API_REFERENCE.md](API_REFERENCE.md#post-onbprofile).

**Step 4. Address**

Optional lookup:

```http
GET /onb/pincode/560001
```

```json
{ "city": "Bengaluru", "state": "KARNATAKA" }
```

Save the address. City and state are filled from the pincode.

```http
POST /onb/address

{ "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001", "gender": "female" }
```

```json
{ "user": { "address": { "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001",
                         "city": "Bengaluru", "state": "KARNATAKA" },
            "gender": "female", "next_step": "bank", "...": "..." } }
```

Next: `bank`.

**Step 5. Bank account**

Optional lookup: `GET /onb/ifsc/HDFC0000240` returns
`{ "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI" }`.

Use a new account number for every test (see [Bank account test data](#bank-account)).

```http
POST /onb/bank

{ "account_number": "50462188351195", "ifsc": "HDFC0000240" }
```

```json
{
  "user": {
    "bank": { "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI",
              "status": "verified", "verified": true, "account_masked": "**********1195", "...": "..." },
    "next_step": "signature-consent", "...": "..."
  }
}
```

Next: `signature-consent`.

**Step 6. FATCA declaration**

```http
POST /onb/declarations

{ "fatca": true, "running_account": false }
```

Returns `{ "user": { "next_step": "nominee", ... } }`. A PAN with existing KYC needs no signature.

**Step 7. Nominee**

Ask for a transaction code:

```http
POST /txn/otp

{ "action": "nominee" }
```

```json
{ "ok": true, "action": "nominee", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 79••••018", "email": "q••••@example.com" }, "dev_otp": "483157" }
```

Add the nominee:

```http
POST /onb/nominee

{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "483157" }
```

```json
{ "user": { "nominee": { "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "*******34E" },
            "next_step": "esign", "...": "..." } }
```

To opt out instead, send `{ "skip": true, "otp": "<code>" }`. The nominee must be an adult and must not be the
investor.

Next: `esign`.

**Step 8. eSign**

Creates the investor and the investment account at the fund platform.

```http
POST /onb/esign

{ "consent": true }
```

```json
{
  "fp": { "invp": "invp_1c40fba1...", "bank": "bac_edecb0fa...", "mfia": "mfia_16aa8e62...", "...": "..." },
  "user": { "onboarded": true, "next_step": "home",
            "fp": { "investor_profile": "invp_1c40fba1...", "investment_account": "mfia_16aa8e62..." }, "...": "..." }
}
```

- Calling it again is safe and returns the same result.
- Called too early, it names the missing step: `400` `{ "error": "Please complete the bank account step first.", "next_step": "bank" }`.
- After onboarding, every other `POST /onb/*` call returns `409` "Your account is already set up. Contact support to
  change these details."

Next: `home`. The account can invest.

---

### 7c. Onboarding: fresh KYC

Use this flow when the PAN has **no KYC record at the KRA**. On the sandbox, use a PAN like `XXXPX3753X`. The
investor creates a new KYC form: Aadhaar from DigiLocker, a signature, and an e-sign of the form. The form then goes to
the KRA for approval. There is no selfie step.

| # | `next_step` | Call | Browser |
|---|---|---|---|
| 1 | `pan` | `POST /onb/pan` with `"fresh_kyc": true` | No |
| 2 | `pan-confirm` | `POST /onb/pan/confirm` | No |
| 3 | `occupation` to `father` | `POST /onb/profile` | No |
| 4 | `address` | `POST /onb/address` | No |
| 5 | `aadhaar` | `POST /onb/kyc/start`, then open `fetch_url` | **Yes**: DigiLocker |
| 6 | `aadhaar` | `POST /onb/kyc/refresh` | No |
| 7 | `bank` | `POST /onb/bank` | No |
| 8 | `signature-consent` | `POST /onb/signature` and `POST /onb/declarations` | No |
| 9 | `kyc-esign` | `POST /onb/kyc/esign`, then open `esign_url` | **Yes**: e-sign |
| 10 | `kyc-esign` | `POST /onb/kyc/refresh` | No |
| 11 | `nominee` | `POST /txn/otp`, then `POST /onb/nominee` | No |
| 12 | `esign` | `POST /onb/esign` | No |
| 13 | `home` | `GET /onb/kyc/status` until the KRA approves | No |

**Step 1. PAN with `fresh_kyc`**

```http
POST /onb/pan

{ "pan": "HBDPK3753S", "name": "Qa Fresh Tester", "dob": "1990-06-10", "fresh_kyc": true }
```

```json
{
  "user": { "kyc_fresh": true, "kyc_status": "in_progress", "kyc_compliant": false, "next_step": "pan-confirm", "...": "..." },
  "checks": { "readiness": { "status": "failed", "code": "kyc_unavailable" }, "...": "..." },
  "kyc_fresh": true
}
```

- Without `"fresh_kyc": true`, the same PAN returns `422` "No KYC record found for this PAN with the KRA. A fresh KYC
  is needed."
- Only "no KYC record" is accepted for fresh KYC. Other KRA results (on hold, rejected, deactivated) return `422` even
  with the flag. The PAN, name and date of birth checks still apply.

**Steps 2-4. Confirm, profile, address**

As in flow 7b, steps 2-4. After the address, `next_step` is `"aadhaar"`.

**Step 5. Start the KYC form**

The app sends the phone's location, which must be inside India (latitude 6 to 37.5, longitude 68 to 97.5).

```http
POST /onb/kyc/start

{ "latitude": 12.9716, "longitude": 77.5946 }
```

```json
{
  "user": { "next_step": "aadhaar", "...": "..." },
  "fetch_url": "https://s.finprim.com/v2/identity_documents/iddoc_.../forward_to?token=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return"
}
```

- Without a location: `400` "Allow location access to continue."
- Outside India: `400` "Your KYC must be done from within India."
- The KYC form is valid for 7 days (`kyc.expires_at`).

**Step 6. DigiLocker (browser), then refresh**

Open `fetch_url` in a browser. On the sandbox page:

1. Enter any mobile number and any 6-digit PIN, tick the consent box, and tap **Sign In**.
2. Enter any 6-digit OTP.
3. Choose the purpose **Know Your Customer** and tap **Allow**.
4. The page ends on "Returning to Risips…". Close the tab.

Then:

```http
POST /onb/kyc/refresh
```

```json
{
  "user": { "aadhaar_masked": "XXXX XXXX 1234", "next_step": "bank", "...": "..." },
  "kyc": { "id": "kycf_...", "status": "created", "proof_status": "fetched", "fields_needed": ["signature"],
           "signature_provided": false, "...": "..." },
  "error": null
}
```

If `proof_status` is still `"pending"`, call refresh again after a few seconds. If it stays `"pending"` or becomes
`"failed"`, call `POST /onb/kyc/start` again for a new `fetch_url`.

**Step 7. Bank**

As in flow 7b, step 5. Next: `signature-consent`.

**Step 8. Signature and FATCA**

The signature is a PNG or JPEG image sent as a data URL, between 500 bytes and 8 MB. It is also added to the KYC form.

```http
POST /onb/signature

{ "image": "data:image/png;base64,iVBORw0KGgo..." }
```

Returns `{ "user": { "signature_url": "/files/...png?exp=...&sig=...", ... } }`. The `signature_url` link works for
1-2 hours.

```http
POST /onb/declarations

{ "fatca": true }
```

Returns `{ "user": { "next_step": "kyc-esign", ... } }` once DigiLocker, bank and signature are all done.

**Step 9. E-sign the KYC form (browser)**

```http
POST /onb/kyc/esign
```

```json
{ "user": { "next_step": "kyc-esign", "...": "..." },
  "esign_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return" }
```

Open `esign_url` in a browser. On the sandbox page:

1. Tick the confirmation box and tap **esign**.
2. Enter any 6-digit OTP.
3. The page ends on "Returning to Risips…". Close the tab.

If the form is not complete, the call returns `400`, for example "Your KYC form is not complete yet (missing:
signature)." with `next_step`.

**Step 10. Refresh after e-sign**

```http
POST /onb/kyc/refresh
```

`kyc.status` becomes `"awaiting_submission"` or `"submitted"` and `next_step` becomes `"nominee"`. `user.kyc_status`
becomes `"with_kra"` once the form is submitted.

**Steps 11-12. Nominee and eSign**

As in flow 7b, steps 7-8. The account is created (`onboarded: true`).

**Step 13. Wait for the KRA**

The KRA approves a fresh KYC in 1-3 working days. Until then, every order, SIP, STP, SWP, AutoPay and cart call returns
`403`:

```json
{ "error": "Your KYC is with the KRA for approval (usually 1–3 working days). You can invest once it is approved.",
  "kyc_status": "with_kra" }
```

Check the status:

```http
GET /onb/kyc/status
```

```json
{ "user": { "kyc_status": "with_kra", "...": "..." }, "kyc": { "status": "submitted", "...": "..." }, "error": null }
```

| `kyc_status` | Meaning |
|---|---|
| `in_progress` | The form is not submitted yet |
| `with_kra` | Submitted and waiting for the KRA |
| `compliant` | Approved. The account can invest, and a "Your KYC is approved" notification appears |
| `failed` | The form failed or expired. Start again with `POST /onb/kyc/start` |

The server re-checks waiting accounts every 30 minutes. On the sandbox, an account can stay `with_kra`.

---

### 7d. One-time investment, redeem and switch

Needs an onboarded account (`next_step: "home"`). Otherwise every call returns `403` "Please complete your KYC to
invest."

#### One-time investment (lumpsum)

Example fund: ICICI Pru Flexi Cap, `INF109KC1Q80`, minimum ₹500.

**Step 1. Ask for a transaction code**

```http
POST /txn/otp

{ "action": "lumpsum" }
```

Returns `dev_otp`.

**Step 2. Place the order**

```http
POST /invest/lumpsum

{ "isin": "INF109KC1Q80", "amount": 1000, "otp": "196140" }
```

```json
{
  "order": {
    "id": "mfp_69f56e9b7ff44690b7e5fe2716d34cf2", "ref": "#MF000853", "kind": "purchase",
    "isin": "INF109KC1Q80", "amount": 1000, "units": null, "nav": null, "folio": null,
    "state": "submitted", "status": "Payment pending",
    "payment": { "id": 625, "status": "PENDING" },
    "token_url": "https://.../api/pg/payments/netbanking/ondc?txnId=...", "...": "..."
  },
  "token_url": "https://.../api/pg/payments/netbanking/ondc?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return"
}
```

Takes 5-10 seconds.

**Step 3. Pay (browser)**

Open `token_url` in a browser. On the sandbox page:

1. Choose **UPI ID**, type any well-formed UPI ID such as `test@okaxis`, and tap **Pay**. Do not scan the QR code.
2. Wait about 10 seconds for the demo bank page.
3. Tap **Success** (or **Failure** to test a failed payment).
4. The page ends on "Returning to Risips…". Close the tab.

The payment link works once. If it is opened and closed without paying, place a new order.

**Step 4. Check the order**

```http
GET /orders/mfp_69f56e9b7ff44690b7e5fe2716d34cf2?force=1
```

`force=1` re-reads the order from the fund platform. Repeat every 10-20 seconds until it changes.

```json
{ "order": { "state": "successful", "status": "Successful", "units": 50.7871, "nav": 19.69,
             "folio": "<folio number>", "payment": { "status": "SUCCESS", "reference": "pay_..." },
             "token_url": null, "...": "..." } }
```

The holding then appears in `GET /portfolio` under `holdings`. Note the `folio`: redeem, switch, STP and SWP need it.

- An amount ending in **1** (for example 1001) is paid but then fails at the fund house: `state: "failed"`,
  `failure_code: "order_failure_at_gateway"`.
- **Failure** on the demo bank page gives a failed payment (`status: "Payment failed"`).

#### Redeem

Needs a holding. Take `isin`, `folio` and `redeemable` (the most that can be redeemed now) from `GET /portfolio`.

1. `POST /txn/otp { "action": "redeem" }`
2. Redeem an amount, or every unit:

```http
POST /invest/redeem

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 200, "otp": "<code>" }
```

or `{ "isin": "INF109KC1Q80", "folio": "<folio>", "all": true, "otp": "<code>" }`.

```json
{ "order": { "id": "mfr_...", "kind": "redemption", "amount": 200, "state": "submitted", "status": "Submitted to AMC", "...": "..." } }
```

There is no payment step. With `all: true`, `amount` is `null` and `all_units` is `true` until the order is priced.
3. `GET /orders/mfr_...?force=1` until `state` is `"successful"`.

The minimum redemption is ₹100.

#### Switch

Moves money between two funds of the **same fund house**. Example: ICICI Pru Flexi Cap to ICICI Pru Liquid
(`INF109K01VQ1`, switch-in minimum ₹1).

1. `POST /txn/otp { "action": "switch" }`
2. Place the switch:

```http
POST /invest/switch

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>", "amount": 300, "otp": "<code>" }
```

```json
{ "order": { "id": "mfs_...", "kind": "switch", "fund_to": { "short_name": "ICICI Pru Liquid Fund", "...": "..." },
             "state": "submitted", "...": "..." } }
```

3. `GET /orders/mfs_...?force=1`. The order is `successful` once the switch-out is done. `units_in` fills in when the
   units reach the target fund.

---

### 7e. Cart checkout

A cart holds up to 10 lines. Each line is one fund, with `mode` `lumpsum` (one-time) or `sip`. At checkout, all
one-time lines are paid together on one UPI page, and all SIP first instalments on a second page. A cart with a SIP
line needs an approved AutoPay first (flow 7f, steps 1-3).

**Step 1. Add lines**

```http
POST /cart

{ "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000 }
```

Then `{ "isin": "INF109K01BL4", "mode": "sip", "amount": 500 }`.

```json
{
  "items": [
    { "id": "ci_87c58783cf", "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000, "problem": null, "...": "..." },
    { "id": "ci_907ae35612", "isin": "INF109K01BL4", "mode": "sip", "amount": 500, "problem": null, "...": "..." }
  ],
  "sip_total": 500, "lumpsum_total": 1000, "pay_now": 1500, "sip_dates": [1, 2, "...", 28]
}
```

- `pay_now` is the one-time total plus each SIP's first instalment.
- `problem` explains why a line can't be bought now, or is `null`.
- Adding the same fund and mode again replaces the amount.
- `GET /cart` shows the cart. `PATCH /cart/{id}` changes a line's `amount` or `mode`. `DELETE /cart/{id}` removes it.

**Step 2. Ask for a transaction code**

`POST /txn/otp { "action": "cart" }`. One code covers the whole checkout.

**Step 3. Start the checkout**

```http
POST /checkouts

{ "sip_day": 5, "otp": "<code>" }
```

- `sip_day` (1-28) is required when the cart has a SIP line, and must be in `sip_dates`.
- `item_ids` (optional) checks out only those lines.

```json
{ "checkout": {
    "id": "chk_625e066d5e39cb52", "sip_day": 5,
    "items": [ { "cart_id": "ci_87c58783cf", "mode": "lumpsum", "amount": 1000, "state": "queued", "order_id": null, "...": "..." },
               { "cart_id": "ci_907ae35612", "mode": "sip", "amount": 500, "state": "queued", "order_id": null, "...": "..." } ],
    "total": 1500, "placed": 0, "failed": 0, "remaining": 2, "position": 0 } }
```

Nothing is bought yet. Note the checkout `id`.

**Step 4. Place and pay each group**

```http
POST /checkouts/chk_625e066d5e39cb52/next
```

```json
{ "checkout": { "placed": 1, "remaining": 1, "...": "..." },
  "mode": "lumpsum",
  "order_id": "mfp_e4fceae766bc44f7a5579ce4ae8d8054", "order_ids": ["mfp_e4fceae766bc44f7a5579ce4ae8d8054"],
  "order": { "state": "submitted", "status": "Payment pending", "...": "..." },
  "token_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

1. Pay `token_url` in a browser, as in flow 7d, step 3.
2. Call `/next` again for the next group.
3. When nothing is left, `/next` returns `{ "checkout": { ... }, "done": true }`.

A group that fails returns `{ "checkout": { ... }, "failed": { "mode": "...", "isins": [ ... ], "error": "..." } }`.
Its lines stay in the cart. Placed lines leave the cart.

**Step 5. Track or stop**

- `GET /checkouts/{id}` shows a live `status` for each line, for example
  `{ "label": "Payment pending", "tone": "pending", "ref": "#MF000854" }`.
- `POST /checkouts/{id}/stop` marks lines still `queued` as `skipped`. They stay in the cart.

---

### 7f. SIP with AutoPay

A SIP is debited each month from an approved AutoPay (mandate). One approved AutoPay is enough for all SIPs while its
monthly limit covers them. The sandbox supports **monthly** SIPs only.

**Step 1. Check AutoPay**

```http
GET /mandate
```

```json
{ "mandate": null,
  "bank": { "bank_name": "HDFC BANK", "account_masked": "****1195", "ifsc": "HDFC0000240", "branch": "MUMBAI  SANDOZ HOUSE" } }
```

**Step 2. Create AutoPay**

```http
POST /mandate

{ "type": "UPI", "limit": 10000, "need": 1000 }
```

- `type`: `"UPI"` for UPI AutoPay. Any other value, or none, creates an eNACH mandate (`E_MANDATE`).
- `limit`: the most that can be debited in a month. Default ₹1,00,000. A value below ₹1,000 is raised to ₹1,000. UPI is
  capped at ₹1,00,000.
- `need`: optional. The monthly amount it must cover.

```json
{
  "mandate": { "id": 234, "status": "CREATED", "type": "UPI", "limit": 10000, "umrn": null,
               "valid_from": "2026-11-03", "valid_to": "2056-11-02", "...": "..." },
  "auth_url": "https://.../api/pg/payments/netbanking/billdesk?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/mandate/return"
}
```

Set the right limit on the first call. While an AutoPay is still `CREATED`, a new `POST /mandate` of the same type
returns that AutoPay, with its original limit and a new `auth_url`. An approved AutoPay that covers `need` returns
`{ "mandate": { ... }, "auth_url": null }`.

**Step 3. Approve AutoPay (browser)**

Open `auth_url` in a browser. On the sandbox page:

- **UPI AutoPay:** tap **Show QR** (do not scan it), wait for the demo bank page, then tap **Success**.
- **eNACH:** on the BillDesk simulator, choose Status **Success** and tap **Submit**.

The page ends on "Returning to Risips…". Then call `GET /mandate` until `mandate.status` is `"APPROVED"`.

**Step 4. Start the SIP**

1. `POST /txn/otp { "action": "sip" }`
2. Create the SIP:

```http
POST /sip

{ "isin": "INF109KC1Q80", "amount": 1000, "day": 5, "first_now": true, "first_pay": "upi", "otp": "<code>" }
```

| Field | Meaning |
|---|---|
| `amount` | Monthly amount |
| `day` | Debit day, 1-28 |
| `first_now` | Optional. `true` makes the first instalment today |
| `first_pay` | Optional. `"upi"` pays the first instalment now on a UPI page; otherwise AutoPay debits it |
| `installments` | Optional. 1-1200 |

```json
{ "sip": { "id": "mfpp_...", "amount": 1000, "state": "awaiting_payment", "status": "Awaiting first payment",
           "installment_day": 5, "next_debit": "...",
           "installments": [ { "id": "mfp_...", "state": "submitted", "upfront": true } ], "...": "..." },
  "token_url": "https://...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

3. Pay `token_url` in a browser, as in flow 7d, step 3.
4. `GET /sips/mfpp_...` shows `status: "Payment received"`, then `"Active"` once allotted.

Without `first_now`, there is no `token_url` and the SIP starts on the next `day`. If the first payment fails, the SIP
is stopped automatically (`status: "Payment failed"`) and AutoPay is not debited for it.

**Step 5. Manage the SIP**

Each change needs its own transaction code. The SIP must be active.

| Action | Code `action` | Request | Result |
|---|---|---|---|
| Pause 1 or 2 months | `sip_pause` | `POST /sips/{id}/pause { "months": 1, "otp": "<code>" }` | `status: "Paused"`, with `pause.from`, `pause.to`, `pause.resumes_on`. The pause starts at the first instalment at least 2 days away |
| Resume | `sip_resume` | `POST /sips/{id}/resume { "otp": "<code>" }` | `status: "Active"`, `pause: null` |
| Change amount or date | `sip_modify` | `POST /sips/{id}/modify { "amount": 1500, "day": 10, "otp": "<code>" }` | A new SIP with a **new id** (`replaces` = old id). The old SIP becomes `"Modified"` or `"Cancelled"` (`replaced_by` = new id) |
| Stop | `sip_cancel` | `POST /sips/{id}/cancel { "code": "invest_later", "otp": "<code>" }` | `status: "Cancelled"` |

A paused SIP must be resumed before it can be changed.

---

### 7g. STP and SWP

Both need a holding: take `folio` from `GET /portfolio`. Both are monthly only.

#### STP (monthly transfer between two funds)

Same fund house only. Example: ICICI Pru Flexi Cap to ICICI Pru Liquid, ₹1,000 a month for 6 months (the ICICI Pru
minimum is ₹1,000 and 6 transfers).

1. `POST /txn/otp { "action": "stp" }`
2. Create the STP:

```http
POST /stp

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>",
  "amount": 1000, "day": 10, "installments": 6, "otp": "<code>" }
```

```json
{ "stp": { "id": "mfsp_...", "state": "active", "status": "Active", "installment_day": 10,
           "start_date": "...", "number_of_installments": 6, "installments": [], "...": "..." } }
```

The first transfer runs on the next `day`. The source holding must be worth at least one transfer.
3. `GET /stps` and `GET /stps/{id}` show it.
4. To stop it: `POST /txn/otp { "action": "stp_cancel" }`, then `POST /stps/{id}/cancel { "otp": "<code>" }`. Returns
   `status: "Cancelled"`.

#### SWP (monthly withdrawal to the bank)

Example: ICICI Pru Flexi Cap, ₹100 a month for 3 months (the ICICI Pru minimum is ₹1 and 2 withdrawals; Aditya Birla
Sun Life is ₹500 and 6).

1. `POST /txn/otp { "action": "swp" }`
2. Create the SWP:

```http
POST /swp

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 100, "day": 1, "installments": 3, "start_offset": 0, "otp": "<code>" }
```

- Send `"until": "stopped"` instead of `installments` to run until stopped.
- `start_offset` is 0, 1 or 2: start this month, or 1 or 2 months later.

```json
{ "swp": { "id": "mfrp_...", "state": "active", "status": "Active", "withdrawals": 3,
           "first_withdrawal": "...", "next_withdrawal": "...", "until_stopped": false, "...": "..." } }
```

3. `GET /swps` and `GET /swps/{id}` show it.
4. To stop it: `POST /txn/otp { "action": "swp_cancel" }`, then `POST /swps/{id}/cancel { "otp": "<code>" }`. Returns
   `status: "Cancelled"` (it can take a second to show).

---

## 8. Hosted pages (browser steps)

Some steps happen on third-party pages. The API returns a link; open it in a browser (the app opens it in a WebView).
When the page is done, call the API again to read the result.

| Link | Used for | What to do on the sandbox |
|---|---|---|
| `token_url` | UPI payment: lumpsum, cart, SIP first instalment | Choose **UPI ID**, type any well-formed ID such as `test@okaxis`, tap **Pay**, wait about 10 s for the demo bank page, tap **Success** or **Failure**. Do not scan the QR code |
| `auth_url` | AutoPay approval | UPI AutoPay: tap **Show QR** (do not scan), wait for the demo bank page, tap **Success**. eNACH: on the BillDesk simulator choose Status **Success**, tap **Submit** |
| `fetch_url` | DigiLocker Aadhaar (fresh KYC) | Any mobile, any 6-digit PIN, tick consent, **Sign In**, any 6-digit OTP, purpose **Know Your Customer**, **Allow** |
| `esign_url` | KYC form e-sign (fresh KYC) | Tick the box, tap **esign**, any 6-digit OTP |
| `journey_url` | Import external funds | The page shows its own OTP. Enter it, choose the options, **Generate QR code**, **Download QR** |

Each payment link works once. At the end, every page goes to a "Returning to Risips…" page. Close the tab and continue
with the API.

---

## 9. Sandbox test data

Only these values work on the sandbox. In the patterns, `X` is any letter and `N` is any digit. Make up your own
letters and digits so you don't collide with another tester.

### PAN

The 4th letter must be `P`. Avoid `A` and `I` as the 5th letter unless you are testing those errors.

| Pattern | Example | Result |
|---|---|---|
| `XXXPX3751X` | `TGVPS3751Q` | KYC record exists. Normal onboarding (flow 7b) |
| `XXXPX3753X` | `HBDPK3753S` | No KYC record. `422` unless `"fresh_kyc": true` (flow 7c) |
| `XXXPX3752X` | `QWRPX3752K` | `422` "We could not verify your PAN right now. Please try again." |
| `XXXPX3754X` | `QWRPX3754K` | `422` "Your KYC is on hold at the KRA and needs a modification." |
| `XXXPX3755X` | `QWRPX3755K` | `422` "Your KYC was rejected by the KRA. A fresh KYC is needed." |
| `XXXPX3756X` | `QWRPX3756K` | `422` "Your KYC is deactivated. You cannot invest right now." |
| `XXXPX3757X` | `QWRPX3757K` | `422` "Your KYC is a legacy record and needs a modification." |
| `XXXPX3758X` | `QWRPX3758K` | `422` "Your KYC is still being processed by the KRA. Please try again in a few days." |
| `XXXPX3759X` | `QWRPX3759K` | `422` "Your KYC is incomplete at the KRA and needs a modification." |
| Any other digits | `QWRPX1234K` | `422` "We could not verify your PAN right now. Please try again." |
| 5th letter `I` | `QWRPI3751K` | `422` "This PAN is invalid. Please check and re-enter." |
| 5th letter `A` | `QWRPA3751K` | `422` "Your PAN is not linked with Aadhaar. Link it on the income-tax portal and try again." |

### Name and date of birth

| Value | Result |
|---|---|
| Any normal name, e.g. `Qa Tester` | Matches |
| Name `Lord Voldemort` | `422` "The name does not match PAN records. Enter it exactly as on your PAN card." |
| Any date of birth for someone 18 or older, e.g. `1992-04-15` | Matches |
| Date of birth `2000-01-01` | `422` "The date of birth does not match PAN records." |
| Under 18, e.g. `2012-01-01` | `400` "You must be 18 or older to open an account. ..." |

### Mobile number

10 digits, starting with 6, 7, 8 or 9, not one digit repeated. Each number can be verified on one account only. Use a
new number for each account.

### Bank account

The last 4 digits decide the result.

| Last 4 digits | Example | Result |
|---|---|---|
| `11XX` | `...1195` | Verified. **Use this for a passing account** |
| `12XX` | `...1250` | Verification does not complete. `422` "Bank verification is taking longer than usual. ..." |
| `13XX` | `...1350` | `422` "We could not confirm this account belongs to you. ..." |
| `14XX`, `15XX`, `1600` | `...1450` | `422`, refused (low confidence) |
| `31XX` | `...3105` | `422` "Bank verification failed. Please check the account number and IFSC." |

- The account number is 9 to 18 digits. **Use a new, unique number for every test**, for example `50` + 8 random
  digits + `1195`.
- An account number that fails too often is blocked by the provider for everyone. The API then returns `422` "This
  account has had too many verification attempts and can’t be verified right now. Please use another account in your
  name." For example, `123456781195` with IFSC `HDFC0000001` is blocked.
- IFSC: use `HDFC0000240` (HDFC BANK, MUMBAI SANDOZ HOUSE). An unknown IFSC such as `ZZZZ0000001` returns "IFSC not
  found."

### Pincode

| Pincode | Result |
|---|---|
| `110001` | Central Delhi, DELHI |
| `560001` | Bengaluru, KARNATAKA |
| `999999` | `404` "Pincode not found." |
| `12345` | `400` "Enter a 6-digit pincode." |

### Funds

The sandbox has 35 funds from two fund houses: ICICI Prudential and Aditya Birla Sun Life (ABSL).

| ISIN | Fund | Use |
|---|---|---|
| `INF109KC1Q80` | ICICI Pru Flexi Cap | One-time minimum ₹500, SIP minimum ₹100, STP ₹1,000 x 6, SWP ₹1 x 2 |
| `INF109K01VQ1` | ICICI Pru Liquid | Switch-in minimum ₹1. Good switch and STP target |
| `INF109K01BL4` | ICICI Pru Large Cap | Minimum ₹100 |
| `INF209K01AJ8` | ABSL Flexi Cap | Minimum ₹100. A different fund house: a switch to it from ICICI Pru is refused |
| `INF109K01AN2` | ICICI Pru Mid Cap | SIP only. One-time purchase is refused |
| `INF109K01JR4` | ICICI Pru Gilt | Switch-in minimum ₹5,000 |
| `INF109KC19T7` | ICICI Pru Nifty50 Value 20 Index | No STP and no SWP |
| `INF109KC1Y64`, `INF109K01TK8` | ICICI Pru Silver / Gold ETF FOF | Category `Commodity` |

### Order amounts

| Amount | Result |
|---|---|
| Ends in **0**, e.g. `1000` | The order succeeds at the fund house |
| Ends in **1**, e.g. `1001` | The payment succeeds, then the fund house fails the order (`order_failure_at_gateway`). Use it to test the failed-order message |

### Allotment timing

| Environment | Time from payment to `successful` |
|---|---|
| Sandbox | A few seconds to about one minute |
| Production | The next working day. Meanwhile the order stays `submitted` with `payment.status: "SUCCESS"` |

### Codes on hosted pages

The DigiLocker and KYC e-sign pages accept any 6-digit OTP. All other codes come from `dev_otp`.

---

Full endpoint reference: **[API_REFERENCE.md](API_REFERENCE.md)**.
