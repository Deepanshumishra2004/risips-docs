# 06. SIP, AutoPay, STP and SWP

**You need an onboarded user** (file 02). Every request needs `Authorization: Bearer {{token}}` and
`Content-Type: application/json`. Every create / change / stop call needs a transaction code first
(`POST /txn/otp { "action": "..." }`, see file 05), and counts towards the **30 per 10 minutes** money limit.

**What needs what:**

| Thing | Needs a browser? | Needs a folio (a paid, successful order) first? |
|---|---|---|
| AutoPay (mandate) | **Yes**: approve it at `auth_url` | No |
| SIP | Yes, if paying the first instalment now (`token_url`); and the AutoPay must be approved | No |
| SIP pause / resume / change / stop | No | No, but needs an **active** SIP |
| STP | No | **Yes** (the "from" fund must be held, worth at least one transfer) |
| SWP | No | **Yes** (the fund must be held, worth at least one withdrawal) |

**What was run on the sandbox (2026-11-03, user 345):** creating a UPI AutoPay up to its `auth_url`, the SIP / STP /
SWP lists, and every negative test marked "verified". The AutoPay approval page and the UPI payment page need a
browser and were not completed, and the user had no holdings, so creating a SIP, STP or SWP and managing them are
written from the API reference and the project's sandbox notes. Expected results there are marked "expected".

ONDC (the order route the sandbox uses) supports **monthly** plans only.

---

## A. AutoPay (mandate)

One approved AutoPay per user is enough. Later SIPs reuse it as long as its monthly limit covers them.

### A1. See the current AutoPay

```
GET {{baseUrl}}/mandate
Authorization: Bearer {{token}}
```

**Expected (200)** with no AutoPay yet (verified):

```json
{ "mandate": null,
  "bank": { "bank_name": "HDFC BANK", "account_masked": "****1195", "ifsc": "HDFC0000240", "branch": "MUMBAI  SANDOZ HOUSE" } }
```

### A2. Create an AutoPay

```
POST {{baseUrl}}/mandate
Authorization: Bearer {{token}}
Content-Type: application/json

{ "type": "UPI", "limit": 10000, "need": 1000 }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/mandate \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"type":"UPI","limit":10000,"need":1000}'
```

- `type`: `"UPI"` = UPI AutoPay. Anything else (or nothing) = `E_MANDATE` (eNACH, net-banking style).
- `limit`: the most that can be debited a month. Default 100,000. Minimum 1,000 (a lower value is **raised to 1,000
  without an error**, verified). UPI is capped at 100,000.
- `need`: optional, the monthly amount it must cover.

**Expected (200)** (shape verified; on the test run the first call sent `"limit": 500`, so it came back as `1000`):

```json
{
  "mandate": { "id": 234, "status": "CREATED", "type": "UPI", "limit": 10000, "umrn": null,
               "valid_from": "2026-11-03", "valid_to": "2056-11-02", "created_at": "2026-11-03T09:41:07+05:30", "bank_account_id": 275 },
  "auth_url": "https://cybrillarta.s.finprim.com/api/pg/payments/netbanking/billdesk?txnId=...&txnType=1&method=UPI&requestedUpiApps=ALL",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/mandate/return"
}
```

**Important (verified):** while an AutoPay is still `CREATED` (not approved yet), calling `POST /mandate` again with
the same `type` **reuses that AutoPay with its old limit** and only gives a new `auth_url`. Example: the first call sent
`"limit": 500` (stored as 1,000); the second call sent `"limit": 10000` and still got `"limit": 1000`. So choose the
right limit on the first call. (Reported as a possible bug in file 08's notes.)

### A3. Approve it in the browser  (BROWSER)

Open `auth_url`.
- **UPI AutoPay:** tap **Show QR** (don't scan it), wait for the demo bank page, tap **Success**.
- **eNACH (BillDesk simulator):** choose Status **Success**, tap **Submit**.

The page ends on "Returning to Risips…" (`/mandate/return?mandate=234`).

### A4. Check it is approved

`GET /mandate` until `mandate.status` is `"APPROVED"` (expected; may take a little while). A later
`POST /mandate` with `need` below the limit then answers `{ "mandate": {...}, "auth_url": null }` (nothing to approve).

---

## B. SIP

### B1. Start a SIP

1. AutoPay approved (section A).
2. `POST /txn/otp { "action": "sip" }`
3.
```
POST {{baseUrl}}/sip
Authorization: Bearer {{token}}
Content-Type: application/json

{ "isin": "INF109KC1Q80", "amount": 1000, "day": 5, "first_now": true, "first_pay": "upi", "otp": "<code>" }
```

| Field | Meaning |
|---|---|
| `amount` | Monthly amount (Flexi Cap minimum ₹100) |
| `day` | Debit day, 1-28 |
| `first_now` | Optional. `true` = make the first instalment today |
| `first_pay` | Optional, `"upi"` = pay that first instalment now on the UPI page (else it is debited from the AutoPay) |
| `installments` | Optional, 1-1200 |
| `frequency` | Optional, only `"monthly"` |

**Expected (200)** (expected):

```json
{ "sip": { "id": "mfpp_...", "amount": 1000, "state": "awaiting_payment", "status": "Awaiting first payment",
           "installment_day": 5, "next_debit": "...", "installments": [ { "id": "mfp_...", "state": "submitted", "upfront": true } ], "...": "..." },
  "token_url": "https://...", "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

4. **(BROWSER)** Pay `token_url` exactly like a lumpsum (file 05, A3).
5. `GET /sips/mfpp_...` -> `status: "Payment received"`, then `"Active"` once allotted.

Without `first_now`, there is no `token_url`: the SIP starts on the next `day`.

If the first payment **fails** (demo bank page: Failure), the server stops the SIP by itself: `status:
"Payment failed"` (the AutoPay is never debited for it).

**Negative tests:**

| Send | Expected |
|---|---|
| No approved AutoPay | `409` `{ "error": "Set up AutoPay first.", "need_mandate": true }` (verified) |
| `"amount": 50` | `400` "Minimum SIP for ICICI Pru Flexi Cap fund is ₹100." (verified) |
| `"day": 30` | `400` "Choose a date between 1 and 28." `reason: "validation"` (verified) |
| `"frequency": "weekly"` | `400` "Only monthly SIPs are available." `reason: "validation"` (verified) |
| Amount above the AutoPay limit | `409` `{ "error": "This is above your AutoPay limit of ₹N. Set up a new AutoPay with a higher limit.", "need_mandate": true, "limit": N }` |
| AutoPay approved with FP's simulate tool | `409` "This AutoPay cannot be debited. Please set up AutoPay again." |
| No `otp` | `400` `reason: "otp_required"` (the order of checks is: investor, fund rules, AutoPay, then the code) |

### B2. List and detail

| Call | Expected |
|---|---|
| `GET /sips` | `{ "sips": [ ... ] }` (verified: `{ "sips": [] }` with none) |
| `GET /sips/mfpp_...` | `{ "sip": {...} }` |
| `GET /sips/mfpp_123` | `404` "SIP not found" (verified) |

SIP `status` values: "Setting up", "Not started", "Activating", "Active", "Awaiting first payment", "Payment received",
"Payment failed", "Paused", "Modified", "Cancelled", "Completed", "Failed". Useful fields: `next_debit` (the next
debit that will really happen, aware of pauses), `installments[]`, `paid_count`, `pause`.

### B3. Pause (skip 1 or 2 months)  -  needs an active SIP

`POST /txn/otp { "action": "sip_pause" }`, then

```
POST {{baseUrl}}/sips/mfpp_.../pause
{ "months": 1, "otp": "<code>" }
```

**Expected:** `{ "sip": { "status": "Paused", "pause": { "from": "...", "to": "...", "resumes_on": "..." }, "...": "..." } }`.
The pause starts at the first instalment at least 2 days away.

| Send | Expected |
|---|---|
| `"months": 3` | `400` "A SIP can be paused for up to 2 months." `reason: "validation"` (verified) |
| An unknown SIP id | `404` "SIP not found" (verified) |
| SIP not active | `409` "Only an active SIP can be paused." |
| Already paused | `409` "This SIP is already paused." |
| No upcoming instalment | `409` "This SIP has no upcoming instalment to pause." |

### B4. Resume

`POST /txn/otp { "action": "sip_resume" }`, then `POST /sips/mfpp_.../resume { "otp": "<code>" }`.
**Expected:** `{ "sip": { "status": "Active", "pause": null, ... } }`. Not paused: `409` "This SIP is not paused."

### B5. Change amount or date

`POST /txn/otp { "action": "sip_modify" }`, then

```
POST {{baseUrl}}/sips/mfpp_.../modify
{ "amount": 1500, "day": 10, "otp": "<code>" }
```

**Expected:** `{ "sip": { new SIP, "replaces": "mfpp_old..." }, "old": { old SIP, "status": "Modified" or "Cancelled", "replaced_by": "mfpp_new..." } }`.
The fund platform can't edit these SIPs, so the server creates a new SIP on the same folio and AutoPay and stops the
old one. The new SIP gets a **new id**.

| Send | Expected |
|---|---|
| Same amount and day | `400` "Change the amount or the SIP date first." |
| SIP not active | `409` "Only an active SIP can be changed." |
| SIP paused | `409` "Resume this SIP before changing it." |
| Above the AutoPay limit | `409` with `need_mandate: true` |

### B6. Stop (cancel)

`POST /txn/otp { "action": "sip_cancel" }`, then

```
POST {{baseUrl}}/sips/mfpp_.../cancel
{ "code": "invest_later", "otp": "<code>" }
```

`code` (optional reason): `amount_not_available`, `investment_returns_not_as_expected`, `amc_support_not_satisfactory`,
`exit_load_not_as_expected`, `switch_to_other_scheme`, `fund_manager_changed`, `investment_goal_complete`,
`mandate_not_ready`, `invest_later`, `customer_support_not_satisfactory`. Anything else becomes `invest_later`.

**Expected:** `{ "sip": { "status": "Cancelled", ... } }`. Again: `409` "This SIP has already stopped."

---

## C. STP (monthly transfer between two funds)  -  needs a holding

Same fund house only. Example: ICICI Pru Flexi Cap -> ICICI Pru Liquid, ₹1,000 x 6 (ICICI STP minimum ₹1,000 and 6
transfers). Take `folio` from `GET /portfolio`.

1. `POST /txn/otp { "action": "stp" }`
2.
```
POST {{baseUrl}}/stp
Authorization: Bearer {{token}}
Content-Type: application/json

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>",
  "amount": 1000, "day": 10, "installments": 6, "otp": "<code>" }
```

**Expected (200)** (expected):
`{ "stp": { "id": "mfsp_...", "state": "active", "status": "Active", "installment_day": 10, "start_date": "...", "number_of_installments": 6, "installments": [] } }`

The first transfer runs on the next `day`. `GET /stps` and `GET /stps/mfsp_...` show it.

**Negative tests** (all verified, user without holdings):

| Send | Expected |
|---|---|
| `to_isin` = `from_isin` | `400` "Choose a different fund to transfer into." |
| To another fund house (`INF209K01AJ8`) | `400` "An STP can only move money between funds of the same fund house." |
| `"amount": 500` | `400` "Minimum transfer is ₹1,000 a month." |
| `"installments": 3` | `400` "An STP needs at least 6 monthly transfers." |
| From Nifty50 Value 20 (`INF109KC19T7`) | `400` "ICICI Pru Nifty50 Value 20 Index Fund does not allow STPs out of it." |
| Correct request, but no holding in that folio | `400` "You hold ₹0 in this fund, less than one transfer." |
| `GET /stps/mfsp_0000` | `404` "STP not found" |

### Stop an STP

`POST /txn/otp { "action": "stp_cancel" }`, then `POST /stps/mfsp_.../cancel { "otp": "<code>" }`.
**Expected:** `{ "stp": { "status": "Cancelled", ... } }`. Already stopped: returns the STP without asking for a code.
Never confirmed: `400` "This STP never started, so there is nothing to cancel. No transfers will run on it."
Unknown id: `404` "STP not found" (verified).

---

## D. SWP (monthly withdrawal to the bank)  -  needs a holding

Example: ICICI Pru Flexi Cap, ₹100 x 3 (ICICI SWP minimum ₹1 and 2 withdrawals; ABSL ₹500 and 6).

1. `POST /txn/otp { "action": "swp" }` (older app builds use `POST /swp/otp`, which does the same)
2.
```
POST {{baseUrl}}/swp
Authorization: Bearer {{token}}
Content-Type: application/json

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 100, "day": 1, "installments": 3, "start_offset": 0, "otp": "<code>" }
```

- `"until": "stopped"` instead of `installments` = runs until you stop it.
- `start_offset`: 0, 1 or 2 = start this month, 1 or 2 months later.

**Expected (200)** (expected):
`{ "swp": { "id": "mfrp_...", "state": "active", "status": "Active", "withdrawals": 3, "first_withdrawal": "...", "next_withdrawal": "...", "until_stopped": false } }`

**Negative tests** (all verified, user without holdings):

| Send | Expected |
|---|---|
| `"installments": 1` | `400` "An SWP needs at least 2 monthly withdrawals." |
| `"start_offset": 3` | `400` "The first withdrawal can start at most 2 months later." `reason: "validation"` |
| Neither `installments` nor `until` | `400` "Choose how many monthly withdrawals." `reason: "validation"` |
| Nifty50 Value 20 (`INF109KC19T7`) | `400` "ICICI Pru Nifty50 Value 20 Index Fund does not allow SWPs." |
| Correct request, no holding | `400` "You hold ₹0 in this fund, less than one withdrawal." |

### Stop an SWP

`POST /txn/otp { "action": "swp_cancel" }`, then `POST /swps/mfrp_.../cancel { "otp": "<code>" }`.
**Expected:** `{ "swp": { "status": "Cancelled", ... } }` (it can take a second to show). `GET /swps`, `GET /swps/:id`
as for STP (`404 "SWP not found"`).

---

## E. Suggested test order once you have a paid holding

1. File 05 A: buy ₹1,000 of Flexi Cap, pay, wait for `successful`, note the `folio`.
2. A: create a UPI AutoPay with limit 10,000, approve it in the browser, wait for `APPROVED`.
3. B1: SIP ₹1,000 on day 5 with `first_now` + `first_pay: "upi"`, pay in the browser.
4. B3 pause 1 month -> B4 resume -> B5 change to ₹1,500 day 10 -> B6 stop the new SIP.
5. C: STP Flexi Cap -> Liquid ₹1,000 x 6 -> stop it.
6. D: SWP Flexi Cap ₹100 x 3 -> stop it.
