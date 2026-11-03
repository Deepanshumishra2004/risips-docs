# 05. Orders and payments: one-time investment, redeem, switch, cart

**You need an onboarded user** (`onboarded: true`, `next_step: "home"`), made with file 02. A user who is not
onboarded gets `403 "Please complete your KYC to invest."` on every call here; a fresh-KYC user waiting for the KRA
gets `403 "Your KYC is with the KRA for approval ..."` (file 03).

Every request needs `Authorization: Bearer {{token}}` and `Content-Type: application/json`.

**What was run on the sandbox (2026-11-03, user 345):** the OTP calls, a ₹1,000 lumpsum up to the payment link, the
order list and detail, a cart checkout up to its payment link, and every negative test marked "verified". The
**UPI payment page needs a browser** and was not paid for this guide, so the user had no holdings: redeem, switch and
the "after payment" results are written from the API reference and the project's sandbox notes. Mark anything that
behaves differently in your bug report.

## Sandbox rules for orders

- **Amount ending in 0** (e.g. 1000): the order succeeds at the fund house.
- **Amount ending in 1** (e.g. 1001): the payment succeeds but the fund house fails the order
  (`state: "failed"`, `failure_code: "order_failure_at_gateway"`). Use it to test the failed-order / refund message.
- After you pay, allotment on the sandbox takes from a few seconds to about a minute. (On production it is the next
  working day.)
- Each payment link (`token_url`) works **once**. If you open it and close it without paying, place a new order.
- **Limit:** order-type calls (including failed ones and negative tests) are limited to **30 per 10 minutes per user**.
  After that every one answers `429 "Too many attempts. Please wait a few minutes and try again."`. Wait 10 minutes.

## Transaction codes (2FA)

Every money call needs a code first: `POST /txn/otp { "action": "<name>" }`, then send `"otp"` with the call.

| `action` | Use it for |
|---|---|
| `lumpsum` | `POST /invest/lumpsum` |
| `redeem` | `POST /invest/redeem` |
| `switch` | `POST /invest/switch` |
| `cart` | `POST /checkouts` (one code for the whole cart) |
| `sip`, `sip_modify`, `sip_cancel`, `sip_pause`, `sip_resume` | SIP calls (file 06) |
| `stp`, `stp_cancel`, `swp`, `swp_cancel` | STP / SWP calls (file 06) |
| `nominee` | `POST /onb/nominee`, `POST /profile/nominee` |
| `delete_account` | `POST /settings/deletion` (file 07) |

```
POST {{baseUrl}}/txn/otp
Authorization: Bearer {{token}}

{ "action": "lumpsum" }
```

**Expected (200):**

```json
{ "ok": true, "action": "lumpsum", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 79••••018", "email": "q••••@example.com" }, "dev_otp": "196140" }
```

Rules: 6 digits, 5 minutes, one use, one action. A new code for the same action replaces the old one. The server checks
your amount and other fields **before** the code, so a mistake in the amount does not use up the code.

| Error when sending the code with the order | Meaning |
|---|---|
| `400 reason: "otp_required"` "Confirm this with the OTP sent to your registered mobile and email." | No `otp` sent |
| `400 reason: "wrong"` "That OTP is not correct. 4 attempts left." | Wrong code (`attempts_left`) |
| `400 reason: "locked"` "Too many wrong attempts. Request a new OTP." | 5 wrong tries |
| `400 reason: "expired"` "This OTP has expired. Request a new one." | Older than 5 minutes |
| `400 reason: "missing"` "This OTP has already been used or expired. Request a new one." | Used already (verified: sending the same code for a second order) |
| `429 reason: "rate_limited"` "Too many wrong codes. ..." | 10 wrong codes in an hour for that action |

---

## A. One-time investment (lumpsum)

Example fund: ICICI Pru Flexi Cap `INF109KC1Q80`, minimum ₹500.

### A1. Get a code

`POST /txn/otp { "action": "lumpsum" }` -> copy `dev_otp`.

### A2. Place the order

```
POST {{baseUrl}}/invest/lumpsum
Authorization: Bearer {{token}}
Content-Type: application/json

{ "isin": "INF109KC1Q80", "amount": 1000, "otp": "196140" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/invest/lumpsum \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"isin":"INF109KC1Q80","amount":1000,"otp":"196140"}'
```

Takes 5-10 seconds.

**Expected (200)** (verified, trimmed):

```json
{
  "order": {
    "id": "mfp_69f56e9b7ff44690b7e5fe2716d34cf2", "ref": "#MF000853", "kind": "purchase",
    "isin": "INF109KC1Q80", "fund": { "short_name": "ICICI Pru Flexi Cap fund", "...": "..." }, "fund_to": null,
    "amount": 1000, "units": null, "nav": null,
    "state": "submitted", "status": "Payment pending", "failure_code": null, "folio": null,
    "token_url": "https://thestupidinvestor.s.finprim.com/api/pg/payments/netbanking/ondc?txnId=...",
    "payment": { "id": 625, "status": "PENDING" },
    "created_at": "2026-11-03 04:10:33", "succeeded_at": null, "fp_old_id": 853, "...": "..."
  },
  "token_url": "https://thestupidinvestor.s.finprim.com/api/pg/payments/netbanking/ondc?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return"
}
```

**Check:** `state: "submitted"`, `status: "Payment pending"`, `payment.status: "PENDING"`, a `token_url` is present.

### A3. Pay in the browser  (BROWSER)

1. Open `token_url` in a browser.
2. Choose **UPI ID**. Type any well-formed UPI id, e.g. `test@okaxis`. Tap **Pay**. **Do not scan the QR code.**
3. Wait about 10 seconds. A **demo bank page** appears with **Success** and **Failure**.
4. Tap **Success** (or **Failure** to test a failed payment).
5. The page ends on "Returning to Risips…" (`/pay/return?order=mfp_...`). Close the tab.

### A4. Check the order

```
GET {{baseUrl}}/orders/mfp_69f56e9b7ff44690b7e5fe2716d34cf2?force=1
Authorization: Bearer {{token}}
```

`force=1` asks the server to re-read the order from the fund platform now.

**Before paying** (verified):

```json
{ "order": { "state": "submitted", "status": "Payment pending", "payment": { "id": 625, "status": "PENDING", "reference": null }, "folio": null, "...": "..." } }
```

**After paying Success** (expected; numbers are illustrative; repeat every 10-20 s until it changes):

```json
{ "order": { "state": "successful", "status": "Successful", "units": 50.7871, "nav": 19.69,
             "folio": "<folio number>", "payment": { "status": "SUCCESS", "reference": "pay_..." }, "token_url": null } }
```

**Check:** `folio` is filled, `units` and `nav` are filled, `token_url` is now `null`. The holding appears in
`GET /portfolio` under `holdings` (with `folio`, `units`, `current`). Write down the `folio`: you need it for redeem,
switch, STP and SWP.

**Amount ending in 1** (e.g. 1001): after paying Success, the order ends `state: "failed"` with
`failure_code: "order_failure_at_gateway"` (payment `SUCCESS`). The app should show a refund message.

**Payment Failure** on the demo bank page: the order shows a failed payment (`status` like "Payment failed").

### Negative tests (all verified)

| Send | Expected |
|---|---|
| `"amount": 100` (below ₹500) | `400` "Minimum investment in ICICI Pru Flexi Cap fund is ₹500." |
| `"isin": "INF109K01AN2"` (SIP-only fund) | `400` "ICICI Pru Mid Cap Fund accepts SIP only. One-time investments are not open for it right now." |
| `"amount": 0` | `400` "Enter an amount." `reason: "validation"` |
| `"isin": "INF109KZZZZZ"` | `404` "Fund not found" |
| No `otp` | `400` `reason: "otp_required"` |
| The same `otp` again for a second order | `400` "This OTP has already been used or expired. Request a new one." `reason: "missing"` |
| A user who is not onboarded | `403` "Please complete your KYC to invest." |

Other errors you may see: `422 { order }` "Order could not be placed. ..." (the fund platform's review failed),
`503 { order }` "Order could not be placed. ... Nothing was debited." (fund platform outage; try later).

---

## B. Orders list and detail

```
GET {{baseUrl}}/orders
Authorization: Bearer {{token}}
```

**Expected (200)** (verified): `{ "orders": [ { "id": "mfp_69f5...", "kind": "purchase", "amount": 1000, "state": "submitted", "status": "Payment pending", "token_url": "https://...", "...": "..." } ] }`

Newest first. Purchases (`mfp_`), redemptions (`mfr_`) and switches (`mfs_`) are all here.

| Send | Expected |
|---|---|
| `GET /orders/mfp_00000000000000000000000000000000` (unknown, or another user's order) | `404` `{ "error": "Order not found" }` |
| `GET /orders/abc` (bad id) | `404` `{ "error": "Order not found", "reason": "validation", "fields": { "id": "Order not found" } }` |

Order `status` labels you will see: "Processing", "Awaiting payment", "Payment pending", "Submitted to AMC",
"Successful", "Failed", "Payment failed".

---

## C. Redeem (sell)  -  needs a holding (a paid, successful order first)

Take `isin` and `folio` from `GET /portfolio` -> `holdings[]`. The holding also has `redeemable` (the most you can
redeem now).

1. `POST /txn/otp { "action": "redeem" }`
2. Redeem an amount:

```
POST {{baseUrl}}/invest/redeem
Authorization: Bearer {{token}}
Content-Type: application/json

{ "isin": "INF109KC1Q80", "folio": "<folio>", "amount": 200, "otp": "<code>" }
```

or everything: `{ "isin": "INF109KC1Q80", "folio": "<folio>", "all": true, "otp": "<code>" }`

**Expected (200):** `{ "order": { "id": "mfr_...", "kind": "redemption", "amount": 200, "state": "submitted", "status": "Submitted to AMC", "...": "..." } }`
(no payment step). With `all: true`, `amount` is `null` and `all_units` is `true` until it is priced.

Then `GET /orders/mfr_...?force=1` until `state: "successful"` with `units` and `nav` (seconds to a minute on the
sandbox).

**Negative tests:**

| Send | Expected |
|---|---|
| `"amount": 50` | `400` "Minimum redemption is ₹100." (verified) |
| A folio with no units of this fund | `400` "There are no redeemable units of this fund in this folio right now." (verified) |
| More than `redeemable` | `400` "You can redeem up to ₹N from this fund right now." |
| A second "all units" while the first is still running | `409` "A redemption of all units in this fund is already in progress. ..." |
| No `folio` | `400` "Folio is required." |

---

## D. Switch  -  needs a holding

Moves money from one fund to another fund **of the same fund house**. Example: ICICI Pru Flexi Cap -> ICICI Pru Liquid
(`INF109K01VQ1`, switch-in minimum ₹1).

1. `POST /txn/otp { "action": "switch" }`
2.
```
POST {{baseUrl}}/invest/switch
Authorization: Bearer {{token}}
Content-Type: application/json

{ "from_isin": "INF109KC1Q80", "to_isin": "INF109K01VQ1", "folio": "<folio>", "amount": 300, "otp": "<code>" }
```

**Expected (200):** `{ "order": { "id": "mfs_...", "kind": "switch", "fund_to": { "short_name": "ICICI Pru Liquid Fund", "...": "..." }, "state": "submitted", "...": "..." } }`

After the switch-out is done, `state` is `successful`; `units_in` fills in when the units arrive in the target fund.

**Negative tests:**

| Send | Expected |
|---|---|
| To a fund of another fund house (`"to_isin": "INF209K01AJ8"`, ABSL) | `400` "You can only switch between funds of the same fund house." (verified) |
| Below the target's switch-in minimum (`"to_isin": "INF109K01JR4"` Gilt, `"amount": 600`) | `400` "ICICI Prudential Gilt Fund - Regular Plan - Growth takes switches of at least ₹5,000." (verified) |
| A folio with no units | `400` "There are no redeemable units of this fund in this folio right now." (verified) |
| More than you hold | `400` "You can switch up to ₹N from this fund right now." |

---

## E. Cart and checkout

A cart holds up to 10 lines. Each line is one fund with `mode` `lumpsum` (one-time) or `sip`. At checkout, all
one-time lines are paid together on **one** UPI page, and all SIP first instalments on a **second** one.

### E1. Add to the cart

```
POST {{baseUrl}}/cart
Authorization: Bearer {{token}}
Content-Type: application/json

{ "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000 }
```

then `{ "isin": "INF109K01BL4", "mode": "sip", "amount": 500 }`.

**Expected (200)** (verified, trimmed):

```json
{
  "items": [
    { "id": "ci_87c58783cf", "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000, "fund": { "...": "..." }, "problem": null },
    { "id": "ci_907ae35612", "isin": "INF109K01BL4", "mode": "sip", "amount": 500, "fund": { "...": "..." }, "problem": null }
  ],
  "sip_total": 500, "lumpsum_total": 1000, "pay_now": 1500, "sip_dates": [1, 2, "...", 28]
}
```

**Check:** `pay_now` = one-time total + each SIP's first instalment. `problem` is `null` for lines that can be bought.
Adding the same fund and mode again replaces the amount.

### E2. View, change, remove

| Call | Expected |
|---|---|
| `GET /cart` | The cart view above (empty cart: `"items": []`, `pay_now: 0`) |
| `PATCH /cart/ci_87c58783cf { "amount": 2000 }` | Cart view with the new amount (verified) |
| `PATCH /cart/ci_87c58783cf { "mode": "sip" }` | Changes the line to SIP (verified). If the same fund is already in the cart as SIP: `409` "This fund is already in your cart with that plan type." |
| `DELETE /cart/ci_907ae35612` | Cart view without that line (verified) |
| `PATCH /cart/ci_0000000000 {...}` | `404` "This item is no longer in your cart." (verified) |

**Negative tests (POST /cart):** `"amount": 100` on Flexi Cap -> `400` "Minimum investment in ICICI Pru Flexi Cap
fund is ₹500."; `"mode": "xyz"` -> `400` "Choose SIP or one-time."; not onboarded -> `403` "Please complete your KYC to
invest."; an 11th fund -> `400` "Your cart can hold up to 10 funds." (all verified except the last).

### E3. Check out

1. `POST /txn/otp { "action": "cart" }`
2.
```
POST {{baseUrl}}/checkouts
Authorization: Bearer {{token}}
Content-Type: application/json

{ "otp": "<code>" }
```
Add `"sip_day": 5` (1-28) when the cart has a SIP line. Optional `"item_ids": ["ci_..."]` checks out only some lines.

**Expected (200)** (verified, one one-time line):

```json
{ "checkout": {
    "id": "chk_625e066d5e39cb52", "sip_day": null, "created_at": "2026-11-03 04:21:12",
    "items": [ { "cart_id": "ci_87c58783cf", "isin": "INF209K01AJ8", "mode": "lumpsum", "amount": 1000,
                 "state": "queued", "order_id": null, "error": null, "status": null, "fund": { "...": "..." } } ],
    "total": 1000, "placed": 0, "failed": 0, "remaining": 1, "position": 0 } }
```

Nothing is bought yet. Note the checkout `id`.

| Send | Expected |
|---|---|
| Cart with a SIP line, no `sip_day` | `400` "Choose a SIP date that every fund in your cart accepts." (verified) |
| Cart with a SIP line, no approved AutoPay | `409` `{ "error": "Set up AutoPay first.", "need_mandate": true, "need": 500 }` (verified). Set up AutoPay first (file 06) |
| Empty cart | `400` "Your cart is empty." |

### E4. Place and pay each group

```
POST {{baseUrl}}/checkouts/chk_625e066d5e39cb52/next
Authorization: Bearer {{token}}
```

**Expected (200)** (verified):

```json
{ "checkout": { "items": [ { "state": "placed", "order_id": "mfp_e4fceae766bc44f7a5579ce4ae8d8054", "...": "..." } ], "placed": 1, "remaining": 0, "...": "..." },
  "order_id": "mfp_e4fceae766bc44f7a5579ce4ae8d8054", "order_ids": ["mfp_e4fceae766bc44f7a5579ce4ae8d8054"],
  "mode": "lumpsum", "order": { "state": "submitted", "status": "Payment pending", "...": "..." },
  "token_url": "https://thestupidinvestor.s.finprim.com/api/pg/payments/netbanking/ondc?txnId=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/pay/return" }
```

1. Pay `token_url` in the browser exactly as in A3.
2. Call `/next` again for the next group (the SIP group, if any).
3. When nothing is left: `{ "checkout": {...}, "done": true }` (verified).

A group that fails comes back as `{ "checkout": {...}, "failed": { "mode": "...", "isins": [...], "error": "..." } }`;
its lines stay in the cart. Placed lines leave the cart (verified: `GET /cart` was empty afterwards).

### E5. Checkout status and stop

| Call | Expected |
|---|---|
| `GET /checkouts/chk_625e066d5e39cb52` | Each line has a live `status`, e.g. `{ "label": "Payment pending", "tone": "pending", "ref": "#MF000854" }` (verified). Later "Order successful", "SIP active", "Not paid · still in cart" |
| `POST /checkouts/chk_625e066d5e39cb52/stop` | Lines still `queued` become `skipped` and stay in the cart. Returns `{ checkout }` (verified) |
| `GET /checkouts/chk_0000000000000000` | `404` "Checkout not found" (verified) |
| Double tap on `/next` | `409` "Your previous order is still being placed. Please wait a moment." |

---

## F. Portfolio

```
GET {{baseUrl}}/portfolio
Authorization: Bearer {{token}}
```

**Expected (200)** for a user with no paid orders yet (verified):

```json
{ "summary": { "current": 0, "invested": 0, "gain": 0, "gain_pct": 0, "day_change": 0, "day_change_pct": 0, "xirr": null },
  "holdings": [], "in_progress": [], "incoming": [],
  "external": { "status": "none", "hidden": false, "holdings": [], "...": "..." },
  "combined": { "current": 0, "...": "..." } }
```

After a successful purchase, `holdings` has one line per fund + folio:
`{ isin, folio, fund, units, nav, nav_date, current, invested, redeemable, gain, gain_pct, ... }`.

Note (verified): an order that is waiting for payment (`Payment pending`) does **not** appear in `in_progress`; orders
show there once paid and being processed.
