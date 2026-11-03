# Risips API flows

End-to-end request sequences with example JSON, for QA (Postman against the sandbox) and app developers. Field rules
and every error are in [API.md](API.md). Verified against `backend-ts` at commit `428814b` (2026-11-02).

Examples use the sandbox, `https://risips-demo-api.azurewebsites.net`, where demo mode is on: every OTP response carries
the code as `dev_otp`, so no SMS or email is needed. Responses are trimmed to the fields that matter (`"...": ...`
marks more). Ids, codes and tokens are illustrative.

## Contents

1. [Before you start](#1-before-you-start)
2. [Sign-up / sign-in](#2-sign-up--sign-in)
3. [Onboarding: KYC-compliant PAN](#3-onboarding-kyc-compliant-pan)
4. [Onboarding: fresh KYC (no KRA record)](#4-onboarding-fresh-kyc-no-kra-record)
5. [Lumpsum purchase with payment](#5-lumpsum-purchase-with-payment)
6. [SIP with first payment](#6-sip-with-first-payment)
7. [Redemption](#7-redemption)
8. [Switch](#8-switch)
9. [STP](#9-stp)
10. [SWP](#10-swp)
11. [Cart checkout](#11-cart-checkout)
12. [Sandbox test data](#12-sandbox-test-data)

---

## 1. Before you start

- `GET /health` and check `"fp_env": "sandbox"` and `"demo": true`.
- Every request after sign-in needs `Authorization: Bearer <token>`. Optionally send `x-device-name` / `x-device-os`.
- Errors always look like `{ "error": "...", "reason": "...", ... }`. A `400` with `reason: "validation"` lists every
  bad field under `fields`.
- Rate limits are real on the sandbox too. Sign-in codes: 20 per email and 20 per IP per hour, one new code per 30 s.
  Ten wrong codes in an hour block that sign-in (or that transaction type) with `429 rate_limited` and `retry_after`;
  during the block no new code is sent either. A restart of the sandbox clears these counters.
- Every transaction needs a 2FA code: `POST /txn/otp { "action": "<action>" }` first, then send `"otp"` with the
  transaction. A transaction sent without `otp` answers `400 reason: "otp_required"`.

## 2. Sign-up / sign-in

Email OTP -> mobile OTP -> PIN. Sign-up and sign-in are the same calls.

**2.1 Ask for an email code**

```http
POST /auth/email/start
{ "email": "qa.tester01@example.com" }
```
```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "482913" }
```

**2.2 Verify it: this creates the account (if new) and returns the session token**

```http
POST /auth/email/verify
{ "email": "qa.tester01@example.com", "code": "482913" }
```
```json
{
  "token": "q3V9...32 chars...",
  "is_new": true,
  "user": { "id": 57, "email": "qa.tester01@example.com", "mobile": null, "pan_masked": null,
            "kyc_status": null, "pin_set": false, "onboarded": false, "next_step": "mobile", "...": "..." }
}
```

A wrong code:

```json
{ "error": "Incorrect code. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }
```

**2.3 Mobile**: `+91`, a leading `0` and spaces are accepted; the number is stored as 10 digits.

```http
POST /auth/mobile/start
Authorization: Bearer <token>
{ "mobile": "+91 98765 01234" }
```
```json
{ "ok": true, "expires_in": 120, "code_length": 6, "dev_otp": "730155" }
```

```http
POST /auth/mobile/verify
{ "mobile": "9876501234", "code": "730155", "bank_consent": true }
```
```json
{ "user": { "mobile": "9876501234", "next_step": "pin", "...": "..." } }
```

A number already verified on another live account:

```json
{ "error": "This mobile number is already registered with another Risips account. Please use a different number.",
  "reason": "mobile_taken" }
```

**2.4 PIN**

```http
POST /auth/pin/set
{ "pin": "1357" }
```
```json
{ "user": { "pin_set": true, "next_step": "pan", "...": "..." } }
```

Next time the app opens, it unlocks with `POST /auth/pin/verify { "pin": "1357" }` -> `{ "ok": true }`. Five wrong PINs
in a row end the session (`401 reason: "pin_locked"`); sign in again with the email code.

`GET /me` returns `{ user }` at any time; `user.next_step` says which onboarding screen comes next.

## 3. Onboarding: KYC-compliant PAN

Use a sandbox PAN ending `3751` plus a letter, e.g. `LHZPX3751Z` (see [test data](#12-sandbox-test-data)).
Steps: `pan -> pan-confirm -> occupation -> income -> marital -> father -> address -> bank -> signature-consent ->
nominee -> esign -> home`. No Aadhaar, selfie or signature for a compliant PAN.

**3.1 PAN**

```http
POST /onb/pan
{ "pan": "LHZPX3751Z", "name": "Test Investor", "dob": "1992-04-15" }
```
```json
{
  "user": { "name": "Test Investor", "pan_masked": "*******51Z", "kyc_compliant": true,
            "kyc_status": "compliant", "kyc_fresh": false, "next_step": "pan-confirm", "...": "..." },
  "checks": {
    "readiness": { "status": "verified" },
    "pan": { "status": "verified", "value": "*******51Z" },
    "name": { "status": "verified" },
    "date_of_birth": { "status": "verified" }
  },
  "pv_id": "...",
  "kyc_fresh": false
}
```

The date must be `YYYY-MM-DD`. "No, edit PAN" is `POST /onb/pan/reset` (clears PAN, name, DOB and KYC state).

**3.2 Confirm, then the profile questions (one call per screen, or all at once)**

```http
POST /onb/pan/confirm
POST /onb/profile
{ "occupation": "private_sector_service", "income_slab": "above_5lakh_upto_10lakh",
  "marital_status": "single", "father_name": "Ramesh Kumar" }
```
```json
{ "user": { "next_step": "address", "...": "..." } }
```

**3.3 Address** (pincode looked up first is optional; the address call checks it again)

```http
GET /onb/pincode/560001
```
```json
{ "city": "Bangalore", "state": "Karnataka" }
```
```http
POST /onb/address
{ "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001", "gender": "female" }
```
```json
{ "user": { "address": { "line1": "...", "pincode": "560001", "city": "...", "state": "..." }, "next_step": "bank" } }
```

A junk pincode such as `999999` is refused: `404` (lookup) / `400` (address) `"Pincode not found."`.

**3.4 Bank** (FP pre-verification; can take up to 20 s)

```http
GET /onb/ifsc/HDFC0000001
POST /onb/bank
{ "account_number": "000123451100", "ifsc": "HDFC0000001" }
```
```json
{ "user": { "bank": { "ifsc": "HDFC0000001", "bank_name": "HDFC Bank", "verified": true,
            "account_masked": "********1100", "...": "..." }, "next_step": "signature-consent" } }
```

A failed check:

```json
{ "error": "Bank verification failed. Please check the account number and IFSC.",
  "bank_status": "failed", "code": "...", "pv_id": "..." }
```

FP blocks an account number for everyone after 3 failed checks on it (`code: "verification_attempt_limit_exceeded"`,
"This account has had too many verification attempts …"). Use a fresh number for each failing test.

**3.5 FATCA declaration**

```http
POST /onb/declarations
{ "fatca": true, "running_account": false }
```
-> `next_step: "nominee"`.

**3.6 Nominee (or opt-out), confirmed with a 2FA code**

```http
POST /txn/otp
{ "action": "nominee" }
```
```json
{ "ok": true, "action": "nominee", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 98••••234", "email": "q••••@example.com" }, "dev_otp": "615204" }
```
```http
POST /onb/nominee
{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "615204" }
```
or opt out: `{ "skip": true, "otp": "615204" }`. -> `next_step: "esign"`, `nominee.pan` masked.

**3.7 eSign: creates the investor and account in FP**

```http
POST /onb/esign
{ "consent": true }
```
```json
{ "user": { "onboarded": true, "next_step": "home", "fp": { "investor_profile": "invp_…", "investment_account": "mfia_…" } },
  "fp": { "invp": "invp_…", "phone": "…", "email": "…", "addr": "…", "bank": "bac_…", "bank_old_id": 123,
          "mfia": "mfia_…", "mfia_old_id": 456 } }
```

Calling it before every step is done answers `400 "Please complete the <step> step first."` with `next_step`. It is
safe to retry. After this, every other `POST /onb/*` answers `409`.

## 4. Onboarding: fresh KYC (no KRA record)

Use a sandbox PAN ending `3753` (readiness `kyc_unavailable`), e.g. `ABCPR3753K`. The app sends `fresh_kyc: true`.
Steps: `pan -> pan-confirm -> occupation … address -> aadhaar (DigiLocker) -> bank -> signature-consent (FATCA +
signature) -> kyc-esign -> nominee -> esign`. No selfie.

**4.1 PAN without the flag is refused (older app builds)**

```http
POST /onb/pan
{ "pan": "ABCPR3753K", "name": "Test Investor", "dob": "1992-04-15" }
```
```json
{ "error": "No KYC record found for this PAN with the KRA. A fresh KYC is needed.",
  "checks": { "readiness": { "status": "failed", "code": "kyc_unavailable" }, "pan": { "value": "*******53K", "...": "..." } },
  "pv_id": "..." }
```

**4.2 With the flag it goes through**

```http
POST /onb/pan
{ "pan": "ABCPR3753K", "name": "Test Investor", "dob": "1992-04-15", "fresh_kyc": true }
```
```json
{ "user": { "kyc_fresh": true, "kyc_status": "in_progress", "kyc_compliant": false, "next_step": "pan-confirm" },
  "checks": { "readiness": { "status": "failed", "code": "kyc_unavailable" }, "...": "..." },
  "kyc_fresh": true }
```

Only `kyc_unavailable` is let through. Other readiness results (on hold, rejected, legacy, …) still answer `422`.

**4.3 Confirm, profile, address** as in [3.2](#3-onboarding-kyc-compliant-pan)-3.3. Then `next_step: "aadhaar"`.

**4.4 Start the KYC form and get the DigiLocker link** (location must be inside India)

```http
POST /onb/kyc/start
{ "latitude": 12.9716, "longitude": 77.5946 }
```
```json
{ "user": { "next_step": "aadhaar", "...": "..." },
  "fetch_url": "https://…digilocker…",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return" }
```

Outside India: `400 "Your KYC must be done from within India."`. Missing profile answers: `400 "Please complete your
personal details first."`.

**4.5 DigiLocker.** Open `fetch_url` in a browser, complete DigiLocker. It ends on `/kyc/return?step=proof`. Then:

```http
POST /onb/kyc/refresh
```
```json
{ "user": { "aadhaar_masked": "XXXX XXXX 1234", "next_step": "bank" },
  "kyc": { "id": "kycf_…", "status": "created", "proof_status": "fetched", "fields_needed": ["signature"],
           "signature_provided": false, "...": "..." },
  "error": null }
```

**4.6 Bank** as in 3.4.

**4.7 Signature and FATCA.** The signature is also sent to the KYC form.

```http
POST /onb/signature
{ "image": "data:image/png;base64,iVBORw0KGgo..." }
POST /onb/declarations
{ "fatca": true }
```
-> `next_step: "kyc-esign"`.

**4.8 E-sign the KYC form**

```http
POST /onb/kyc/esign
```
```json
{ "user": { "next_step": "kyc-esign" }, "esign_url": "https://…", "return_url_prefix": "…/kyc/return" }
```

Open `esign_url`, sign, land on `/kyc/return?step=esign`, then `POST /onb/kyc/refresh`: once `kyc.status` is
`awaiting_submission` or `submitted`, `next_step` is `nominee`.

**4.9 Nominee and eSign** as in 3.6-3.7. The FP account is created.

**4.10 Waiting for the KRA.** `user.kyc_status` is `with_kra` once the form is `submitted`. Any order, plan, mandate
or cart write answers:

```json
{ "error": "Your KYC is with the KRA for approval (usually 1–3 working days). You can invest once it is approved.",
  "kyc_status": "with_kra" }
```
(status `403`). `GET /onb/kyc/status` and the `kyc-approvals` job (every 30 min) re-check; when the KRA shows the PAN
as compliant, `kyc_status` becomes `compliant` and a "Your KYC is approved" notification appears. Whether the sandbox
KRA ever approves a sandbox fresh KYC was not confirmed.

## 5. Lumpsum purchase with payment

Example fund: ICICI Pru Flexi Cap, `INF109KC1Q80` (sandbox minimum ₹500). In the sandbox, an amount ending in **0**
succeeds and one ending in **1** fails at the fund house.

```http
POST /txn/otp
{ "action": "lumpsum" }
POST /invest/lumpsum
{ "isin": "INF109KC1Q80", "amount": 1000, "otp": "204611" }
```
```json
{
  "order": { "id": "mfp_…", "ref": "#MF000812", "kind": "purchase", "amount": 1000, "state": "submitted",
             "status": "Payment pending", "token_url": "https://…", "payment": { "id": 9876, "status": "PENDING" } },
  "token_url": "https://…",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return"
}
```

The call takes several seconds (it waits for FP's review). Then pay:

1. Open `token_url` (single use: reopening shows "Invalid Token", and a burned link means a new order).
2. Choose **UPI ID**, type any well-formed id, e.g. `zz9qx7notreal@nohandle123`, tap Pay. Do not scan the QR.
3. After about 10 s a demo bank page shows **Success / Failure**. Tap Success.
4. The page redirects to `/pay/return?order=mfp_…`.

Then poll:

```http
GET /orders/mfp_…
```
```json
{ "order": { "state": "successful", "status": "Successful", "units": 41.2345, "nav": 24.25,
             "folio": "1234567/89", "payment": { "status": "SUCCESS", "reference": "pay_…" } } }
```

Allotment in the sandbox takes seconds; on production it is the next working day (the order sits in `submitted`
meanwhile, with `payment.status: "SUCCESS"`).

## 6. SIP with first payment

**6.1 AutoPay mandate** (once per investor; later SIPs reuse it while its limit covers them)

```http
POST /mandate
{ "type": "UPI", "limit": 10000, "need": 1000 }
```
```json
{ "mandate": { "id": 321, "status": "CREATED", "type": "UPI", "limit": 10000 },
  "auth_url": "https://…", "return_url_prefix": "…/mandate/return" }
```

Open `auth_url` and authorise it, then `GET /mandate` until `mandate.status` is `APPROVED`. Without an approved mandate,
`POST /sip` answers `409 "Set up AutoPay first."` with `need_mandate: true`.

**6.2 Create the SIP and pay the first instalment now on UPI**

```http
POST /txn/otp
{ "action": "sip" }
POST /sip
{ "isin": "INF109KC1Q80", "amount": 1000, "day": 5, "first_now": true, "first_pay": "upi", "otp": "550912" }
```
```json
{
  "sip": { "id": "mfpp_…", "amount": 1000, "state": "awaiting_payment", "status": "Awaiting first payment",
           "installment_day": 5, "next_debit": "…", "installments": [ { "id": "mfp_…", "state": "submitted", "upfront": true } ] },
  "token_url": "https://…",
  "return_url_prefix": "…/pay/return"
}
```

Pay on `token_url` exactly as for a lumpsum. `GET /sips/mfpp_…` then shows `status: "Payment received"` and, once
allotted, `"Active"`. If the first payment fails, the backend cancels the SIP (`status: "Payment failed"`), so the
mandate is never debited for it. Later instalments are debited from the mandate on the SIP day.

Without `first_now`, the SIP simply starts on the next `day` (no `token_url`).

**6.3 Manage it** (each with its own 2FA action): `POST /sips/:id/pause { months: 1|2, otp }` (`sip_pause`),
`/resume { otp }` (`sip_resume`), `/modify { amount, day, otp }` (`sip_modify`), `/cancel { code?, otp }`
(`sip_cancel`).

## 7. Redemption

Needs a holding: take `folio` and `isin` from `GET /portfolio` (`holdings[]`).

```http
POST /txn/otp
{ "action": "redeem" }
POST /invest/redeem
{ "isin": "INF109KC1Q80", "folio": "1234567/89", "amount": 200, "otp": "118230" }
```
or everything: `{ "isin": "…", "folio": "…", "all": true, "otp": "…" }`.

```json
{ "order": { "id": "mfr_…", "kind": "redemption", "amount": 200, "state": "submitted", "status": "Submitted to AMC" } }
```

Minimum ₹100; more than the redeemable value is refused with "You can redeem up to ₹N from this fund right now.".
`GET /orders/mfr_…` shows `successful` with units and NAV (seconds in the sandbox).

## 8. Switch

Same fund house only. Example: Flexi Cap -> ICICI Pru Liquid (`INF109K01VQ1`, switch-in minimum ₹1).

```http
POST /txn/otp
{ "action": "switch" }
POST /invest/switch
{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "1234567/89", "amount": 300, "otp": "903114" }
```
```json
{ "order": { "id": "mfs_…", "kind": "switch", "fund_to": { "short_name": "ICICI Pru Liquid Fund" }, "state": "submitted" } }
```

A switch shows `successful` after its out-leg; `units_in` fills when the units arrive in the target fund.

## 9. STP

Monthly transfer between two funds of one fund house. Example: Flexi Cap -> Liquid, ₹1,000 x 6 (ICICI STP minimum
₹1,000, 6 instalments). The source holding must be worth at least one transfer.

```http
POST /txn/otp
{ "action": "stp" }
POST /stp
{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "1234567/89",
  "amount": 1000, "day": 10, "installments": 6, "otp": "771020" }
```
```json
{ "stp": { "id": "mfsp_…", "state": "active", "status": "Active", "installment_day": 10, "start_date": "…",
           "number_of_installments": 6, "installments": [] } }
```

The first transfer runs on the next `day`. Cancel: `POST /stps/mfsp_…/cancel { "otp": "…" }` (`stp_cancel`).

## 10. SWP

Monthly withdrawal to the bank. Example: Flexi Cap, ₹100 x 3 (ICICI SWP minimum ₹1 and 2 instalments; ABSL ₹500 and 6).

```http
POST /txn/otp
{ "action": "swp" }
POST /swp
{ "isin": "INF109KC1Q80", "folio": "1234567/89", "amount": 100, "day": 1, "installments": 3, "start_offset": 0, "otp": "662801" }
```
```json
{ "swp": { "id": "mfrp_…", "state": "active", "status": "Active", "withdrawals": 3, "first_withdrawal": "…",
           "next_withdrawal": "…", "until_stopped": false } }
```

`"until": "stopped"` (instead of `installments`) runs until cancelled; `start_offset: 1|2` starts 1-2 months later.
Cancel: `POST /swps/mfrp_…/cancel { "otp": "…" }` (`swp_cancel`).

## 11. Cart checkout

One OTP for the whole checkout; at most two payments (one-time lines together, SIP first instalments together).

```http
POST /cart
{ "isin": "INF109KC1Q80", "mode": "lumpsum", "amount": 1000 }
POST /cart
{ "isin": "INF209K01AJ8", "mode": "sip", "amount": 500 }
GET /cart
```
```json
{ "items": [ { "id": "ci_1a2b3c4d5e", "mode": "lumpsum", "amount": 1000, "problem": null },
             { "id": "ci_6f7a8b9c0d", "mode": "sip", "amount": 500, "problem": null } ],
  "sip_total": 500, "lumpsum_total": 1000, "pay_now": 1500, "sip_dates": [1, 2, 3, "…", 28] }
```

With a SIP in the cart, an approved mandate covering the SIP total is needed first (see 6.1).

```http
POST /txn/otp
{ "action": "cart" }
POST /checkouts
{ "sip_day": 5, "otp": "330197" }
```
```json
{ "checkout": { "id": "chk_0123456789abcdef", "items": [ { "mode": "lumpsum", "state": "queued" }, { "mode": "sip", "state": "queued" } ],
                "total": 1500, "placed": 0, "remaining": 2, "position": 0 } }
```

Then place and pay each group in turn:

```http
POST /checkouts/chk_0123456789abcdef/next
```
```json
{ "checkout": { "...": "..." }, "mode": "lumpsum", "order_id": "mfp_…", "order_ids": ["mfp_…"],
  "order": { "...": "order view" }, "token_url": "https://…", "return_url_prefix": "…/pay/return" }
```

Pay on `token_url`, then call `/next` again for the SIP group (its `token_url` pays the first instalment). When nothing
is left: `{ "checkout": {…}, "done": true }`. A group that fails comes back as `{ "checkout": {…}, "failed": { "mode",
"isins", "error" } }` and its lines stay in the cart. `GET /checkouts/:id` shows a live status per line;
`POST /checkouts/:id/stop` leaves the rest unpaid (they stay in the cart).

## 12. Sandbox test data

From the project's FP sandbox notes. Use only these test values on the sandbox; never real people's PANs.

### PAN: KRA readiness (pre-verification)

The KYC status is set by the PAN's digits `3751`-`3759` (the 4th character must be `P`, an individual):

| PAN pattern | Readiness | What the API does |
|---|---|---|
| `…3751…`, e.g. `LHZPX3751Z` | verified (KYC-compliant) | 200, `kyc_status: "compliant"`. **Verified on backend-ts** |
| `…3753…`, e.g. `ABCPR3753K` | `kyc_unavailable` (no KRA record) | 422 without `fresh_kyc`; 200 fresh KYC with `fresh_kyc: true`. **Verified on backend-ts 2026-11-02** |
| `…3752…` | constrained | 422 (generic "We could not verify your PAN right now." unless FP's code matches a known one) |
| `…3754…` | on hold | 422 "…on hold at the KRA…" |
| `…3755…` | rejected | 422 "…rejected by the KRA…" |
| `…3756…` | deactivated | 422 "…deactivated…" |
| `…3757…` | legacy | 422 "…legacy record…" |
| `…3758…` | under process | 422 "…still being processed…" |
| `…3759…` | incomplete | 422 "…incomplete at the KRA…" |
| any other | not compliant | 422 |

The 3752 and 3754-3759 mapping is taken from the project notes in the order listed there; only 3751 and 3753 were
re-checked against `backend-ts`. Treat the others as expected behaviour to confirm.

### PAN: identity checks (pre-verification)

| Input | Result |
|---|---|
| PAN of the form `XXXPI` + 4 digits + letter (5th character `I`) | PAN invalid: "This PAN is invalid. Please check and re-enter." |
| PAN of the form `XXXPA` + 4 digits + letter (5th character `A`) | "Your PAN is not linked with Aadhaar. …" |
| Name `Lord Voldemort` | "The name does not match PAN records. …" |
| DOB `2000-01-01` | "The date of birth does not match PAN records." |

So for the readiness tests above, avoid `A` and `I` as the 5th character.

### Bank accounts (pre-verification)

The last four digits of the account number set the result:

| Last 4 digits | FP result | Expected API behaviour |
|---|---|---|
| `11XX` | very high confidence | verified |
| `12XX` | high confidence | not verified on 2026-11-03: stays pending, 422 "taking longer than usual". Use `11XX` |
| `13XX` | uncertain | 422 "We could not confirm this account belongs to you. …" |
| `14XX`, `15XX`, `1600` | low / very low / zero confidence | 422 (refused) |
| `31XX` | failed | 422 "Bank verification failed. …" |

Only the patterns, not every code path, are from FP's docs; the exact `code` FP returns for 14XX-1600 was not checked.
**FP blocks an account number for everyone after 3 failed checks** (`verification_attempt_limit_exceeded`), so vary the
leading digits for each failing test. The sandbox has also been seen leaving bank checks at `accepted` for a while,
which the API reports as "Bank verification is taking longer than usual. …".

### Orders

- Only ICICI Pru and ABSL schemes exist in the sandbox (the 35 funds in `GET /funds`).
- Amount ending in **0**: the order succeeds. Ending in **1**: it fails at the fund house
  (`order_failure_at_gateway`, after the payment succeeds; the app shows refund messaging).
- UPI checkout: type a well-formed UPI ID (e.g. `zz9qx7notreal@nohandle123`), wait ~10 s for the demo bank page, tap
  Success. Never scan the QR. Links are single use.
- Never use FP's "simulate" APIs on this tenant: simulated payments don't advance ONDC orders and simulated mandates
  can't be debited.
