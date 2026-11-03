# Risips API testing guide (for QA)

Start here. This guide is for testing the Risips backend by hand with **Postman** against the **sandbox**. You do not
need to be a developer. Every flow file gives you the requests in order, what to send, what you should get back, and
what each error means.

Everything in these files was run against the sandbox on 2026-11-02 / 2026-11-03. The responses shown are real
responses from those runs, trimmed to the parts that matter (`"...": "..."` means "more fields here"). Ids, codes and
tokens will be different for you.

## Contents

| File | What it covers |
|---|---|
| [README.md](README.md) (this file) | Set-up, how sign-in works, errors, waits and limits, test data |
| [01-login-and-account.md](01-login-and-account.md) | Sign up / sign in by email, mobile number, PIN, `/me`, logout |
| [02-onboarding-kyc-compliant.md](02-onboarding-kyc-compliant.md) | Full onboarding for a PAN that already has KYC (the common case) |
| [03-onboarding-fresh-kyc.md](03-onboarding-fresh-kyc.md) | Onboarding for a PAN with no KYC record (DigiLocker, signature, KYC e-sign) |
| [04-funds.md](04-funds.md) | Fund list, filters, collections, fund page, compare, watchlist |
| [05-orders-and-payments.md](05-orders-and-payments.md) | One-time investment and UPI payment, redeem, switch, orders, cart and checkout |
| [06-sip-stp-swp.md](06-sip-stp-swp.md) | AutoPay, SIP (start, pause, resume, change, stop), STP, SWP |
| [07-reports-settings-other.md](07-reports-settings-other.md) | Reports, notifications, settings, account deletion, import from MF Central, assistant |
| [08-bug-retest-checklist.md](08-bug-retest-checklist.md) | How to re-test the reported bugs RSP-AUTH-001..004, RSP-ONB-001..006, RSP-FND-001..004 |

Suggested order for a new tester: README -> 01 -> 02 -> 04 -> 05 -> 06 -> 07. Do 03 when you test fresh KYC, and 08
when you re-test bugs.

---

## 1. What the API is

The Risips app (Android / iOS) talks to one backend server. The backend talks to the fund platform
(FintechPrimitives, "FP") and to the KYC services. When you test the API you are doing exactly what the app does,
without the screens.

- **Sandbox base URL:** `https://risips-demo-api.azurewebsites.net`
- The sandbox uses **fake money** and **fake KYC**. Only use the test data in [section 8](#8-test-data-cheat-sheet).
  Never type a real person's PAN, bank account or Aadhaar.
- Every request and response body is JSON. Send the header `Content-Type: application/json` on every request with a
  body.

**First check: is the sandbox up?**

```
GET https://risips-demo-api.azurewebsites.net/health
```

```json
{ "ok": true, "fp_env": "sandbox", "demo": true, "schemes": 35, "build": "6e50501acd3f..." }
```

Check: `fp_env` is `"sandbox"` and `demo` is `true`. If `fp_env` says `"production"`, **stop**: that server moves
real money.

---

## 2. How sign-in works

There is no password. You sign in with a one-time code sent to the email address.

1. `POST /auth/email/start` with the email. The server "sends" a 6-digit code.
2. `POST /auth/email/verify` with the email and the code. The answer contains a **`token`**.
3. Send that token on every other request in a header:

   ```
   Authorization: Bearer <TOKEN>
   ```

The token stays valid until you log out, an account is deleted, or the PIN is locked. If it stops working you get
`401 { "error": "Please sign in again." }`: just sign in again.

Sign-up and sign-in are the **same two calls**. A new email creates a new account (`"is_new": true`).

### Where do the codes come from?

On the sandbox no SMS or email is sent. **The code comes back in the response**, in a field called `dev_otp`:

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "454804" }
```

This is true for every code: the email code, the mobile code, and the transaction codes (`POST /txn/otp`). The
DigiLocker and e-sign pages (fresh KYC) are different: they accept any 6 digits (see file 03).

### Transaction codes (2FA)

Every money action (buy, sell, SIP, STP, SWP, cart checkout), the nominee step and account deletion need an extra
code:

1. `POST /txn/otp { "action": "<name>" }` -> the response has `dev_otp`.
2. Send the real request with `"otp": "<that code>"` in the body.

A code is for one action name only (for example a `lumpsum` code does not work for `redeem`), works **once**, and lasts
5 minutes. If you forget the `otp`, the request answers `400 reason: "otp_required"` (and no code is used up). The
action names are listed in file 05.

---

## 3. Postman set-up

### 3.1 Create an environment

In Postman: **Environments -> + (new)**, name it `Risips sandbox`, and add these variables:

| Variable | Initial value | What it is for |
|---|---|---|
| `baseUrl` | `https://risips-demo-api.azurewebsites.net` | The server |
| `email` | `qa.<something>@example.com` | Your test email for this run |
| `token` | (empty) | Filled in automatically after sign-in |
| `otp` | (empty) | Filled in automatically from `dev_otp` |

Select the environment (top right) before sending requests.

### 3.2 Headers for every request

- `Content-Type: application/json`
- `Authorization: Bearer {{token}}` (on everything except `/health`, `/auth/email/start` and `/auth/email/verify`)

Tip: put the `Authorization` header on a Postman **Collection** (Collection -> Authorization -> Type "Bearer Token",
Token `{{token}}`), and set each request's Authorization to "Inherit auth from parent". Then you never paste the token.

Optional: `x-device-name: Postman` and `x-device-os: windows`. They show up in Settings > Active devices.

### 3.3 Save the token and the code automatically

Put this in the **Scripts -> Post-response** tab (older Postman: "Tests" tab) of the Collection, so it runs after
every request:

```js
// Save the sign-in token and any sandbox OTP into the environment.
let j = {};
try { j = pm.response.json(); } catch (e) {}
if (j.token)   { pm.environment.set("token", j.token); }
if (j.dev_otp) { pm.environment.set("otp", j.dev_otp); }
```

Now you can write `"code": "{{otp}}"` or `"otp": "{{otp}}"` in a body, and it uses the last code the server sent.

### 3.4 Use new test data each run

- A new email each time you want a fresh account (e.g. `qa.akshita.0311a@example.com`).
- A new mobile number per account (a number can be on only one account).
- A new bank account number for each bank test (see [section 8](#8-test-data-cheat-sheet)).
- Type these numbers yourself and keep a note of them. Postman's `{{$randomInt}}` gives 0-1000, so it can make a
  number of the wrong length.

### 3.5 Request ids for bug reports

Every response has an `X-Request-Id` header. Put it in your bug report, with the time.

---

## 4. Error format

Every error looks the same:

```json
{ "error": "Message for the investor", "reason": "machine_code", "...": "sometimes extra fields" }
```

- `error` is always there. It is the text the app shows.
- `reason` is there only for some errors (table below).
- A **validation error** (bad input) is `400` with `reason: "validation"`, and lists **every** bad field under
  `fields`:

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

  `error` is the first problem; `fields` has all of them. Unknown extra fields in a body are ignored (no error).
- A badly formed id in the URL (e.g. `GET /orders/abc`) is a `404` with `reason: "validation"`. The pincode and IFSC
  lookups answer `400` instead.

### Table of `reason` codes

| `reason` | HTTP | What it means for you |
|---|---|---|
| `validation` | 400 (404 for ids in the URL) | Your input is wrong. Read `error` and `fields` |
| `otp_required` | 400 | You sent a money action without `otp`. Call `POST /txn/otp` first |
| `wrong` | 400 | Wrong code. `attempts_left` says how many tries this code has left |
| `expired` | 400 | The code is too old (email/mobile 2 min, transaction 5 min). Ask for a new one |
| `missing` | 400 | There is no open code: never sent, already used, or replaced by a newer one |
| `locked` | 400 | 5 wrong tries on this code. Ask for a new code |
| `rate_limited` | 429 | Too many requests or too many wrong codes. Wait. `retry_after` = seconds to wait (when present) |
| `mobile_taken` | 409 | This mobile number is already verified on another account |
| `pin_locked` | 401 | 5 wrong PINs in a row. The session is ended: sign in again by email |
| `wrong_pin` | 400 | Wrong current PIN in Settings (PIN check / change) |

Some errors carry other helpful fields instead: `next_step` (which onboarding step is missing), `attempts_left`,
`retry_after`, `need_mandate` (set up AutoPay first), `need` / `limit` (AutoPay amounts), `checks` and `pv_id` (PAN
check details), `bank_status` and `code` (bank check details), `kyc_status`, `cart_id`.

### Other HTTP statuses you may see

| HTTP | `error` | Meaning |
|---|---|---|
| 401 | "Please sign in again." | Token missing, wrong or ended. Sign in again |
| 403 | "Please complete your KYC to invest." | This user is not onboarded yet |
| 403 | "Your KYC is with the KRA for approval ..." | Fresh-KYC user waiting for KRA approval (file 03) |
| 404 | "Not found" | Wrong URL |
| 409 | various | The action does not fit the current state (e.g. "Your account is already set up ...") |
| 413 | "The request is too large." | Body too big (images over the limit) |
| 422 | various | The fund platform or KYC check said no (PAN, bank, order review) |
| 502 | various | The fund platform answered with an error |
| 503 | "Can’t reach the fund platform right now. Please try again in a minute." | The fund platform did not answer. Retry later; report it if it lasts |
| 500 | "Something went wrong. Please try again." | A server bug. Always report it with the `X-Request-Id` |

Malformed JSON (e.g. a missing comma) gives `400 "The request body is not valid JSON."`.

---

## 5. Waits and limits you will hit

The sandbox has the same limits as production. They are the most common reason for a "strange" error while testing.

### Codes (email, mobile, transaction)

| Rule | What you see |
|---|---|
| One new code per **30 seconds** for the same email / mobile / action | `429 "A code was sent just now. Please try again in N seconds."`, `retry_after` |
| At most 20 codes per hour for the same email / mobile / action | `429`, `rate_limited` |
| **5 wrong tries** on one code | 5th wrong: "Incorrect code. No attempts left: request a new code." (`attempts_left: 0`). 6th: `400 reason: "locked"` |
| **10 wrong codes in an hour** (20 in a day), counted across all codes for that email or action | `429 "Too many wrong codes. Please try again in N minutes."`. During this block **no new code is sent** either |
| Email code lasts 120 s, transaction code 300 s | `400 reason: "expired"` |

Tip: when you test the "10 wrong codes" block, use a throw-away email. That email is blocked for about an hour.

### Request limits

| Limit | Applies to | What you see |
|---|---|---|
| 300 requests per minute per network | everything | `429 "Too many attempts. Please wait a few minutes and try again."` |
| 20 sign-in code requests per hour per network | `POST /auth/email/start` | same 429. Everyone in one office shares this |
| 60 code checks per hour per network | `POST /auth/email/verify` | same 429 |
| 10 PIN tries per 15 min per user | PIN set / verify / check / change | same 429 |
| 30 per hour per user | PAN check, bank check, fresh-KYC calls | same 429 |
| **30 per 10 min per user** | every order, SIP / STP / SWP call, AutoPay, checkout, account deletion, adding a nominee later | same 429. **Negative tests count too**: after about 30 order attempts in 10 minutes, wait 10 minutes |

These 429s have no `retry_after`. Postman shows a `RateLimit` header with the time left (`t=` seconds).

### PIN

5 wrong PINs in a row (`POST /auth/pin/verify` or the Settings PIN check) end the session: `401 reason: "pin_locked"`.
Sign in again by email; the PIN stays the same.

### Slow calls (normal, not bugs)

| Call | Usual time |
|---|---|
| `POST /onb/pan` | 1-7 s (can wait up to 25 s) |
| `POST /onb/bank` | about 2 s (can wait up to 20 s) |
| `POST /invest/lumpsum`, `POST /sip`, plans | 5-10 s (can wait up to 45 s) |

Set Postman's request timeout to at least 60 s (Settings -> General -> Request timeout, 0 = never).

---

## 6. The `user` object and `next_step`

Most sign-in and onboarding calls answer `{ "user": { ... } }`. The fields testers look at most:

| Field | Meaning |
|---|---|
| `id`, `email`, `mobile` | The account |
| `next_step` | **Which onboarding step comes next.** See the table in file 02 |
| `pan_masked` | The PAN with only the last 3 characters shown, e.g. `*******51Q`. The full PAN is never returned |
| `kyc_compliant` | `true` when the PAN already has a good KYC record |
| `kyc_fresh` | `true` when the user is doing a fresh KYC |
| `kyc_status` | `compliant`, `in_progress`, `with_kra`, `failed`, or `null` before the PAN step |
| `bank` | Bank details, `account_masked` only (never the full number) |
| `onboarded` | `true` once the account is fully set up and can invest |

---

## 7. Hosted pages that need a browser

Some steps happen on third-party web pages, not in the API. In Postman you get a link; you open it in a browser (or the
app does it in a WebView). After you finish there, you call the API again to check the result.

| Link in the response | Used for | What to do on the sandbox | File |
|---|---|---|---|
| `token_url` | UPI payment (lumpsum, cart, SIP first payment) | Choose UPI ID, type any well-formed id like `test@okaxis`, tap **Pay**, wait about 10 s for the demo bank page, tap **Success** (or **Failure**). Do not scan the QR | 05, 06 |
| `auth_url` | AutoPay (mandate) approval | BillDesk simulator: Status **Success**, **Submit**. UPI AutoPay: **Show QR** (don't scan), wait for the demo bank page, **Success** | 06 |
| `fetch_url` | DigiLocker Aadhaar (fresh KYC) | Any mobile, any 6-digit PIN, any 6-digit OTP, purpose **Know Your Customer**, **Allow** | 03 |
| `esign_url` | KYC form e-sign (fresh KYC) | Tick the box, tap **esign**, any 6-digit OTP | 03 |
| `journey_url` | MF Central import (sandbox stand-in) | The page shows its own OTP; type it, Generate QR, Download QR | 07 |

Each payment link works **once**. If you open it and leave, that order can't be paid; place a new order.

At the end, these pages go to a "Returning to Risips…" page on the sandbox (`/pay/return`, `/mandate/return`,
`/kyc/return`). That is expected: close the tab and go back to Postman.

---

## 8. Test data cheat-sheet

Only these values work on the sandbox. `X` = any letter, `N` = any digit.

### PAN (4th letter must be `P`; avoid `A` and `I` as the 5th letter unless you test those errors)

| Pattern | Example | Result |
|---|---|---|
| `XXXPX3751X` | `TGVPS3751Q`, `LHZPX3751Z` | KYC-compliant: normal onboarding (file 02) |
| `XXXPX3753X` | `HBDPK3753S` | No KYC record: 422 unless you send `"fresh_kyc": true` (file 03) |
| `XXXPX3754X` | `QWRPX3754K` | 422 "Your KYC is on hold at the KRA and needs a modification." |
| `XXXPX3755X` | `QWRPX3755K` | 422 "Your KYC was rejected by the KRA. A fresh KYC is needed." |
| `XXXPX3756X` | `QWRPX3756K` | 422 "Your KYC is deactivated. You cannot invest right now." |
| `XXXPX3752X`, `3757`, `3758`, `3759` | | 422 with the matching KRA message (not re-run for this guide) |
| any other digits | `QWRPX1234K` | 422 "We could not verify your PAN right now. Please try again." |
| 5th letter `I` | `QWRPI3751K` | 422 "This PAN is invalid. Please check and re-enter." |
| 5th letter `A` | `QWRPA3751K` | 422 "Your PAN is not linked with Aadhaar. ..." |
| Name `Lord Voldemort` | | 422 "The name does not match PAN records. ..." |
| DOB `2000-01-01` | | 422 "The date of birth does not match PAN records." |

Use any normal name (e.g. `Qa Tester`) and a date of birth that makes the person 18 or older, written `YYYY-MM-DD`.
Make up the letters yourself, so you don't collide with another tester's PAN.

### Mobile number

10 digits, starting with 6, 7, 8 or 9, not all the same digit. Each number can be verified on **one** account only.
`+91 98765 01234`, `09876501234` and `9876501234` count as the same number.

### Bank account (the last 4 digits decide the result)

| Last 4 digits | Result |
|---|---|
| `11XX`, e.g. `...1195` | Verified |
| `12XX` | Did **not** verify on 2026-11-03: stayed pending, "Bank verification is taking longer than usual. ..." |
| `13XX`, e.g. `...1350` | 422 "We could not confirm this account belongs to you. ..." (`code: "uncertain"`) |
| `14XX`, `15XX`, `1600` | Expected to be refused (low confidence); not re-run |
| `31XX`, e.g. `...3105` | 422 "Bank verification failed. Please check the account number and IFSC." |

- Account number: 9 to 18 digits. **Make the front digits random for every test**, e.g. `50` + 8 random digits +
  `1195`. FP blocks an account number for everyone after 3 failed checks on it.
- IFSC: `HDFC0000240` (HDFC BANK, MUMBAI SANDOZ HOUSE). Must be a real IFSC; `ZZZZ0000001` -> 404 "IFSC not found."

### Pincodes

| Pincode | Result |
|---|---|
| `110001` | Central Delhi, DELHI |
| `560001` | Bengaluru, KARNATAKA |
| `999999`, `123456` | 404 "Pincode not found." (junk rows in FP's list are refused) |
| `12345` | 400 "Enter a 6-digit pincode." |

### Funds (sandbox has 35 funds, only ICICI Prudential and Aditya Birla Sun Life)

| ISIN | Fund | Notes |
|---|---|---|
| `INF109KC1Q80` | ICICI Pru Flexi Cap | One-time minimum ₹500, SIP minimum ₹100, STP minimum ₹1,000 x 6, SWP ₹1 x 2 |
| `INF109K01VQ1` | ICICI Pru Liquid | Switch-in minimum ₹1. Good switch / STP target |
| `INF109K01BL4` | ICICI Pru Large Cap | Minimum ₹100 |
| `INF209K01AJ8` | ABSL Flexi Cap | Minimum ₹100. Other fund house (switch to it is refused from ICICI) |
| `INF109K01AN2` | ICICI Pru Mid Cap | **SIP only**: one-time purchase refused |
| `INF109K01JR4` | ICICI Pru Gilt | Switch-in minimum ₹5,000 |
| `INF109KC19T7` | ICICI Pru Nifty50 Value 20 Index | No STP and no SWP |
| `INF109KC1Y64`, `INF109K01TK8` | ICICI Pru Silver / Gold ETF FOF | Category `Commodity` |

### Order amounts (sandbox only)

- An amount ending in **0** (e.g. 1000) succeeds at the fund house.
- An amount ending in **1** (e.g. 1001) fails at the fund house after the payment (to test the failed-order screen).

### OTP in hosted pages

DigiLocker and the KYC e-sign simulator accept any 6 digits.
