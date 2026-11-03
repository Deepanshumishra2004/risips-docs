# 07. Reports, notifications, settings, MF Central import, assistant

Every request needs `Authorization: Bearer {{token}}`. **Everything in this file was run on the sandbox on
2026-11-03** (three test users), unless marked "not run". The MF Central import was run end to end with the
sandbox stand-in.

---

## A. Reports

```
GET {{baseUrl}}/reports/<key>?fy=2026
Authorization: Bearer {{token}}
```

| `key` | Report |
|---|---|
| `capital-gains` | Realised gains, short / long term |
| `holdings` | Holdings by folio |
| `dividends` | Dividend (IDCW) credits |
| `elss` | ELSS (tax saver) investments |
| `orders` | Order history |
| `contract-notes` | Completed orders, each with a contract-note link |
| `contract-note` | One order's contract note: needs `&id=<order id>` |
| `balance` | Risips balance statement (always empty) |
| `gst` | GST invoices (always empty) |

`fy` = the start year of the financial year (`2026` = FY 2026-27). Leave it out for the current year.

**Expected (200)** (orders report, user with one unpaid order):

```json
{
  "key": "orders", "title": "Order history", "period": "FY 2026-27",
  "fy": { "value": 2026, "options": [ { "value": 2026, "label": "FY 2026-27" }, { "value": 2025, "label": "FY 2025-26" }, { "value": 2024, "label": "FY 2024-25" } ] },
  "summary": [ { "label": "Orders placed", "value": "1" }, { "label": "Invested", "value": "₹0.00" }, { "label": "Redeemed", "value": "₹0.00" } ],
  "sections": [ { "title": "Orders", "columns": ["..."], "rows": [ "...1 row..." ] } ],
  "empty": null,
  "notes": ["..."],
  "investor": { "name": "Qa Tester", "pan": "*******51Q" },
  "generated_at": "..."
}
```

**Check:**
- `investor.pan` is masked (only the last 3 characters).
- With no data, `sections[].rows` is empty and `empty` has a friendly sentence, e.g. holdings: "You don't hold any
  mutual fund units yet."; capital gains: "No redemptions or switches in FY 2026-27, so there are no capital gains to
  report."
- Every one of the 9 keys answers 200.

**Printable version:** `GET /reports/orders/html` -> `{ "html": "<!doctype html>..." }` (the app turns it into a PDF).

**Negative tests:**

| Send | Expected |
|---|---|
| `GET /reports/foo` | `404` "Report not found" |
| `GET /reports/orders?fy=abc` | `400` "Choose a financial year." `reason: "validation"` |
| `GET /reports/contract-note` (no `id`) | `404` "Order not found" |
| `GET /reports/contract-note?id=<unpaid order id>` | `400` "A contract note is available once the order is successful." |

---

## B. Notifications

```
GET {{baseUrl}}/notifications
```

**Expected (200):**

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

**Check:**
- Promotions (tag `REWARDS` / `MARKET`, the modal and banner) appear **only** when the `market_insights` setting is on.
  With it off (the default), a new user saw `"notifications": []`, `modal: null`, `banner: null`.
- Orders, SIPs and KYC events add their own notifications (tags `ORDER`, `SIP`, `KYC`, `ACTION NEEDED`, `SECURITY`).
  A fresh-KYC user gets "Your KYC is with the KRA" and later "Your KYC is approved".

| Call | Expected |
|---|---|
| `POST /notifications/read {}` | `{ "ok": true }`: marks all as read; `unread` becomes 0 |
| `POST /notifications/read { "ids": ["promo-festive-2026"] }` | Marks only those (not run) |
| `POST /notifications/seen { "id": "promo-festive-2026" }` | Marks the modal as shown (not run) |

---

## C. Settings

### C1. Notification preferences

```
GET {{baseUrl}}/settings/notifications
```

**Expected (200):**

```json
{ "prefs": { "sip_reminders": true, "portfolio_summary": true, "goals_reviews": true, "market_insights": false, "push": true, "whatsapp": true, "email": false } }
```

```
PUT {{baseUrl}}/settings/notifications
{ "market_insights": true }
```

**Expected (200):** the same `prefs` with `market_insights: true`.

| Send | Expected |
|---|---|
| `{ "foo": true }` | `400` `Unknown preference "foo".` `reason: "validation"` |
| `{ "push": "yes" }` | `400` `"push" must be true or false.` |

### C2. Change PIN

| Call | Expected |
|---|---|
| `POST /settings/pin/check { "current": "0000" }` (wrong) | `400` `{ "error": "Incorrect PIN. 4 attempts left.", "attempts_left": 4, "reason": "wrong_pin" }` |
| `POST /settings/pin/check { "current": "1357" }` | `{ "ok": true }` |
| `POST /settings/pin/change { "current": "1357", "pin": "1357" }` | `400` "Choose a PIN different from your current one." |
| `POST /settings/pin/change { "current": "1357", "pin": "2580" }` | `{ "ok": true, "user": { "pin_set": true, ... } }`. The new PIN works in `/auth/pin/verify` |
| 5 wrong `current` PINs in a row | `401` `reason: "pin_locked"`: signed out |

### C3. Active devices

Sign in to the same account twice (two Postman environments, or Postman plus the app) to have two sessions.

| Call | Expected |
|---|---|
| `GET /settings/devices` | `{ "current": { "id": 414, "name": "...", "os": "...", "signed_in_at": "...", "last_active_at": "...", "current": true }, "others": [ { "id": 415, "current": false, "...": "..." } ] }` |
| `POST /settings/devices/414/logout` (your own id) | `400` "Use Logout to sign out of this device." |
| `POST /settings/devices/415/logout` (the other one) | `{ "ok": true }`. The other session's token now gets `401` |
| The same again | `404` "That device is already signed out." |
| `POST /settings/devices/logout-others` | `{ "ok": true, "signed_out": <count> }` |

`name` and `os` come from the `x-device-name` / `x-device-os` headers.

### C4. Add a nominee after onboarding

Only for a user who **skipped** the nominee during onboarding (`nominee_skipped: true`).

1. `POST /txn/otp { "action": "nominee" }`
2.
```
POST {{baseUrl}}/profile/nominee
{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "<code>" }
```

**Expected (200):**

```json
{ "user": { "nominee": { "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "*******34E" },
            "nominee_added_at": "2026-11-03T04:23:26.407Z", "nominee_skipped": false, "...": "..." } }
```

| When | Expected |
|---|---|
| The user already has a nominee | `409` "You already have a nominee. Contact support to change it." |
| The user is not onboarded | `409` "Please finish setting up your account first." |
| `{ "skip": true, ... }` | `400` "Enter your nominee’s details." |
| The same checks as onboarding | e.g. "Nominee PAN must be different from yours." (file 02, Step 7) |

The fund platform adds this nominee only to folios opened from now on.

### C5. Delete the account

1. Check first:
```
GET {{baseUrl}}/settings/deletion
```
**Expected:** `{ "checks": { "active_plans": 0, "pending_orders": 0, "ok": true }, "request": null, "grace_days": 30 }`

2. `POST /txn/otp { "action": "delete_account" }`
3.
```
POST {{baseUrl}}/settings/deletion
{ "reason": "not_using", "otp": "<code>" }
```
`reason`: `not_using`, `other_platform`, `privacy`, `other`, or leave it out.

**Expected (200):**

```json
{ "ok": true, "request": { "status": "requested", "requested_at": "2026-11-03T04:15:15.856Z", "purge_after": "2026-12-03T04:15:15.856Z" } }
```

**Check:** every session is signed out (`GET /me` -> `401`). The account is erased after 30 days.

4. **Cancel the deletion** by signing in again (file 01, Steps 1-2): the verify response has
   `"deletion_cancelled": true`.

| Send | Expected |
|---|---|
| No `otp` | `400` `reason: "otp_required"` |
| `"reason": "bored"` | `400` "Unknown reason." `reason: "validation"` |
| User with an active SIP / STP / SWP | `409 { checks }` "Stop your active SIPs, STPs and SWPs before deleting your account." (not run) |
| User with orders still being processed | `409 { checks }` "Wait for your pending orders to complete before deleting your account." (not run) |

Note (verified): an order that is only waiting for payment did **not** count as a pending order
(`pending_orders: 0`).

---

## D. Import external funds (MF Central, sandbox stand-in)

On the sandbox the real MF Central is replaced by a stand-in page on the sandbox server. It returns **sample**
holdings (real fund names, made-up units). The user does not need to be onboarded, but the PAN step must be done.

### D1. Status

```
GET {{baseUrl}}/external
```

**Expected (200)** (user with a PAN):

```json
{ "status": "none", "provider": "sandbox", "available": true,
  "pan": null, "pan_masked": "TG*****51Q", "pan_locked": true,
  "mobile": "7926224018", "email": "qa.m3y2nd@example.com",
  "request": null, "error": null, "imported_at": null, "as_of": null, "count": 0, "sample": false, "hidden": false,
  "consent_text": "I authorise Risips to fetch my mutual fund holdings from MF Central ..." }
```

**Check:** `pan` is `null` (the full PAN is never returned). Note this `pan_masked` shows the first 2 and last 3
characters (`TG*****51Q`), unlike the user object (`*******51Q`).

### D2. Start

```
POST {{baseUrl}}/external/start
{ "via": "mobile", "consent": true }
```

**Expected (200):**

```json
{ "journey_url": "https://risips-demo-api.azurewebsites.net/mfc-sandbox/journey/0974498...", "provider": "sandbox", "sent_to": "+91 ******4018" }
```

Without `"consent": true`: `400` "Please give your consent to fetch your holdings from MF Central."

### D3. The journey page  (BROWSER, but on the sandbox server)

1. Open `journey_url` in a browser. The page **shows the OTP** ("OTP is 295710").
2. Type it in the 6 boxes, tap the button. (A wrong OTP shows "Incorrect OTP. Please try again.")
3. Pick plan type and "Summary of holdings", tap **Generate QR code**.
4. Tap **Download QR** (saves `cas-request-qr.png`).

In the app, the page hands the QR to the app automatically. In Postman you send it yourself:

### D4. Send the QR

Get the QR as text: on the journey page, right-click the QR image -> **Copy image address**. That copies a long text
starting with `data:image/png;base64,`. (Do not upload it to an online converter.) Send it:

```
POST {{baseUrl}}/external/qr
{ "qr": "data:image/png;base64,iVBORw0KGgo...", "extent": "SUMMARY" }
```

**Expected (200):** the status object with `"status": "importing"` and `request.qr_at` set.

Empty `qr`: `400` "We didn’t receive the QR code. Tap Download QR on the MF Central page, then Continue."

Tip: steps D3-D4 can also be done with the stand-in's own API: `POST /mfc-sandbox/api/verify { "token": "<last part of journey_url>", "otp": "<otp shown on page>" }`
-> `{ "ok": true }`, then `POST /mfc-sandbox/api/generate-qr { "token": "...", "planType": "BOTH", "extentOfData": "SUMMARY" }`
-> `{ "qr": "<base64>" }`, then `POST /external/qr { "qr": "<that base64>" }`. No token header is needed for the two
`/mfc-sandbox/api/*` calls.

### D5. Wait about 20 seconds, then check

`GET /external` -> `"status": "ready"`, `"count": 5`, `"sample": true`, `"as_of": "2026-11-02"`.

`GET /portfolio` -> `external.holdings` has the 5 sample funds, e.g.:

```json
{ "isin": "INF879O01027", "folio": "15938274", "external": true,
  "fund": { "name": "Parag Parikh Flexi Cap Fund", "plan": "Direct Plan", "...": "..." },
  "units": 412.553, "nav": 89.7397, "nav_date": "2026-11-02", "current": 37022.38, "invested": 28261.36, "gain": 8761.02, "...": "..." }
```

and `combined` adds them to the Risips holdings (`combined.xirr` is `null` when external funds exist).

### D6. Hide, remove, cancel

| Call | Expected |
|---|---|
| `POST /external/visibility { "hidden": true }` | `{ "ok": true, "hidden": true }` |
| `POST /external/visibility { "hidden": false }` | `{ "ok": true, "hidden": false }` |
| `DELETE /external` | Status with `"status": "none"`, `"count": 0` |
| `POST /external/cancel` | Leaves an unfinished journey; returns the status |

---

## E. Assistant

```
GET {{baseUrl}}/assistant
```

**Expected (200):**

```json
{ "name": "Qa", "intro": "I can pull up your holdings, SIPs, orders and KYC status, and explain how mutual funds work.",
  "groups": [ { "title": "YOUR ACCOUNT", "chips": ["How are my SIPs doing?", "When is my next SIP debit?", "Track my last order", "What’s my KYC status?"] },
              { "title": "STATEMENTS", "chips": ["..."] }, { "title": "LEARN", "chips": ["..."] } ] }
```

```
POST {{baseUrl}}/assistant
{ "text": "What is my SIP status?" }
```

**Expected (200):**

```json
{ "messages": [ { "kind": "text", "text": "You don’t have a running SIP yet. Pick a fund in Explore and choose “Start SIP” to set one up." } ],
  "chips": ["SIP vs lump sum", "Explain risk levels"],
  "actions": [ { "label": "Explore funds", "route": "/(tabs)/explore", "kind": "primary" } ] }
```

**Check:** answers use the user's own data and never recommend a specific fund.

| Send | Expected |
|---|---|
| `{ "text": "" }` or `{}` | `400` "Type a question." |
| More than 500 characters | `400` "Keep the question under 500 characters." (not run) |
| `{ "action": "book_call" }` | Stores a callback request (one open at a time) (not run) |
