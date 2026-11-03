# 03. Onboarding with fresh KYC (PAN with no KYC record)

Use this path for a PAN that has **no KYC record at the KRA**. On the sandbox that is a PAN like `XXXPX3753X`
(make up the letters; 4th letter `P`; 5th not `A` or `I`), e.g. `HBDPK3753S`.

Instead of only checking an existing KYC, the user creates a new KYC form: Aadhaar from **DigiLocker**, a
**signature**, and an **e-sign** of the KYC form. The form then goes to the KRA, which approves it in 1-3 working days.
There is **no selfie** step.

**Before you start:** do file 01 until `next_step` is `"pan"`. Every request needs `Authorization: Bearer {{token}}`
and `Content-Type: application/json`.

**What was run on the sandbox (2026-11-03, user 346):** steps 1-5 below by API, plus the signature, the bank step and
every negative test listed. The DigiLocker page, the e-sign page and the steps after them need a browser and were
**not** finished for this guide: they are written from the API reference and the app's sandbox hints. Tell the
developers if they behave differently.

## Order of steps

| # | `next_step` | Call | Browser? |
|---|---|---|---|
| 1 | `pan` | `POST /onb/pan` with `"fresh_kyc": true` | No |
| 2 | `pan-confirm` | `POST /onb/pan/confirm` | No |
| 3 | `occupation` ... `father` | `POST /onb/profile` | No |
| 4 | `address` | `POST /onb/address` | No |
| 5 | `aadhaar` | `POST /onb/kyc/start` -> open `fetch_url` | **Yes** (DigiLocker) |
| 6 | `aadhaar` | `POST /onb/kyc/refresh` after DigiLocker | No |
| 7 | `bank` | `POST /onb/bank` | No |
| 8 | `signature-consent` | `POST /onb/signature` + `POST /onb/declarations` | No |
| 9 | `kyc-esign` | `POST /onb/kyc/esign` -> open `esign_url` | **Yes** (e-sign) |
| 10 | `kyc-esign` | `POST /onb/kyc/refresh` after e-sign | No |
| 11 | `nominee` | `POST /txn/otp` + `POST /onb/nominee` | No |
| 12 | `esign` | `POST /onb/esign` | No |
| 13 | `home` | `GET /onb/kyc/status` until the KRA approves | No |

---

## Step 1. PAN with `"fresh_kyc": true`

### 1a. First, without the flag (what old app builds send)

```
POST {{baseUrl}}/onb/pan
Authorization: Bearer {{token}}

{ "pan": "HBDPK3753S", "name": "Qa Fresh Tester", "dob": "1990-06-10" }
```

**Expected (422):**

```json
{
  "error": "No KYC record found for this PAN with the KRA. A fresh KYC is needed.",
  "checks": {
    "readiness": { "code": "kyc_unavailable", "modification": null, "reason": null, "status": "failed" },
    "pan": { "status": "verified", "value": "*******53S", "...": "..." },
    "...": "..."
  },
  "pv_id": "pv_..."
}
```

This is correct behaviour (not a bug): without the flag the server refuses, for older app versions.

### 1b. With the flag

```
POST {{baseUrl}}/onb/pan
Authorization: Bearer {{token}}
Content-Type: application/json

{ "pan": "HBDPK3753S", "name": "Qa Fresh Tester", "dob": "1990-06-10", "fresh_kyc": true }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/pan \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"pan":"HBDPK3753S","name":"Qa Fresh Tester","dob":"1990-06-10","fresh_kyc":true}'
```

**Expected (200):**

```json
{
  "user": { "kyc_fresh": true, "kyc_status": "in_progress", "kyc_compliant": false, "next_step": "pan-confirm", "...": "..." },
  "checks": { "readiness": { "code": "kyc_unavailable", "status": "failed", "...": "..." }, "...": "..." },
  "kyc_fresh": true
}
```

**Check:** `kyc_fresh: true`, `kyc_status: "in_progress"`, `next_step: "pan-confirm"`.

Only "no KYC record" is let through. Other KRA results (3754 on hold, 3755 rejected, 3756 deactivated, ...) still
answer 422 even with `"fresh_kyc": true`. The name / DOB / PAN checks still apply (see file 02, Step 1).

## Steps 2-4. Confirm, profile, address

Exactly as in file 02 (Steps 2, 3 and 4). After the address, `next_step` is **`"aadhaar"`** (not `"bank"`).

Example used: `POST /onb/profile { "occupation": "professional", "income_slab": "above_10lakh_upto_25lakh",
"marital_status": "married", "father_name": "Suresh Kumar" }` and `POST /onb/address { "line1": "Flat 3, Connaught
Place", "pincode": "110001", "gender": "male" }`.

---

## Step 5. Start the KYC form (get the DigiLocker link)

The app sends the phone's location. It must be inside India (latitude 6 to 37.5, longitude 68 to 97.5).

```
POST {{baseUrl}}/onb/kyc/start
Authorization: Bearer {{token}}
Content-Type: application/json

{ "latitude": 12.9716, "longitude": 77.5946 }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/kyc/start \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"latitude":12.9716,"longitude":77.5946}'
```

**Expected (200):**

```json
{
  "user": { "next_step": "aadhaar", "...": "..." },
  "fetch_url": "https://s.finprim.com/v2/identity_documents/iddoc_8c67.../forward_to?token=...",
  "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return"
}
```

**Check:** `fetch_url` is a link. Copy it for Step 6.

**Negative tests:**

| Send | Expected |
|---|---|
| Before profile / address are done (e.g. right after the PAN) | `400` `{ "error": "Please complete your personal details first." }` |
| `{}` (no location) | `400` "Allow location access to continue." `reason: "validation"` (on `latitude` and `longitude`) |
| `{ "latitude": 51.5, "longitude": -0.12 }` (London) | `400` "Your KYC must be done from within India." `reason: "validation"` |
| From a KYC-compliant user who has not finished onboarding | `409` "Your KYC is already on record with the KRA." (from the API reference) |
| From a user who is already onboarded | `409` "Your account is already set up. Contact support to change these details." |
| `POST /onb/aadhaar/start` (old Aadhaar OTP) from a fresh-KYC user | `409` `{ "error": "Verify your Aadhaar with DigiLocker.", "next_step": "aadhaar" }` |

Right after this call, `POST /onb/kyc/refresh` shows the new form:

```json
{
  "user": { "next_step": "aadhaar", "aadhaar_masked": null, "...": "..." },
  "kyc": {
    "id": "kycf_f59b28dec94047bdb5bd3572700dfdaa", "status": "created", "reason": null,
    "proof_status": "pending", "fields_needed": ["address", "identity_proof", "signature"],
    "signature_provided": false, "esign_status": null, "expires_at": "2026-11-10T09:38:29+0530", "submitted_at": null
  },
  "error": null
}
```

The form expires after 7 days (`expires_at`).

---

## Step 6. DigiLocker in the browser, then refresh  (BROWSER)

1. Open `fetch_url` in a browser (Chrome on your laptop is fine). It shows a page titled **SIMULATION**,
   "Sign In to your account via DigiLocker".
2. Enter **any mobile number** and **any 6-digit PIN**, tick the consent box, tap **Sign In**.
3. Enter **any 6-digit OTP**.
4. For the purpose, choose **Know Your Customer**, then tap **Allow**.
5. The page ends on "Returning to Risips…" (`/kyc/return?step=proof`). Close the tab.

Then:

```
POST {{baseUrl}}/onb/kyc/refresh
Authorization: Bearer {{token}}
```
(no body)

**Expected (200)** once the Aadhaar is fetched:

```json
{
  "user": { "aadhaar_masked": "XXXX XXXX 1234", "next_step": "bank", "...": "..." },
  "kyc": { "status": "created", "proof_status": "fetched", "fields_needed": ["signature"], "...": "..." },
  "error": null
}
```

**Check:** `kyc.proof_status` is `"fetched"`, `user.aadhaar_masked` is filled, `next_step` moves to `"bank"` (or
further, if the bank / signature are already done).

If `proof_status` is still `"pending"` or `"failed"`: call refresh again after a few seconds. If it stays that way,
call `POST /onb/kyc/start` again to get a new `fetch_url` and redo DigiLocker.

Refresh before `kyc/start` was ever called: `400 "Start your KYC from the Aadhaar step first."`.

---

## Step 7. Bank

Exactly as file 02, Step 5 (use a new `...11XX` account number). After it, `next_step` is `"signature-consent"`.

Note: the sandbox also accepts the bank and the signature **before** DigiLocker is done (verified); `next_step` then
stays `"aadhaar"` until DigiLocker is finished.

---

## Step 8. Signature and FATCA

### 8a. Signature

The image goes in the body as a **data URL**: `data:image/png;base64,...` or `data:image/jpeg;base64,...`. The image
must be a real PNG or JPEG, between **500 bytes and 8 MB**.

This small PNG (a 24 x 12 squiggle, 944 bytes) is accepted by the sandbox. Copy the whole line as the `image` value:

```json
{ "image": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABgAAAAMCAIAAAD3UuoiAAADd0lEQVR4AQFsA5P8AP///////////////////////////////////////////////////////////////////////////////////////////////wD///////////////////////////////////////////////////////////////////////////////////////////////8A////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////AP///////////////////////////////////////////////wAAAP///////////wAAAP///////////////////////////wD///////////////////////////////////////////8AAAD///////////////////8AAAD///////////////////////8A////////////////////////////////////////AAAA////////////////////////////AAAA////////////////////AAAAAP///////////////////////////////////////////////////////////////////////wAAAP///////////////wD///8AAAD///////////////////////////8AAAD///////////////////////////////////////8AAAD///////////8A////////AAAA////////////////////AAAA////////////////////////////////////////////////////////////AP///////////wAAAP///////////wAAAP///////////////////////////////////////////////////wAAAAAAAP///wD///////////////8AAAAAAAAAAAD///////////////////////////////////////////////////////////////8AAAAA////////////////////////////////////////////////////////////////////////////////////////////////seYVFtuA7nsAAAAASUVORK5CYII=" }
```

```
POST {{baseUrl}}/onb/signature
Authorization: Bearer {{token}}
Content-Type: application/json

{ "image": "data:image/png;base64,iVBORw0KGgo...(the whole string above)" }
```

**Expected (200):**

```json
{ "user": { "signature_url": "/files/346_signature_f7090e7c....png?exp=1793685600&sig=1ec5f0...", "next_step": "...", "...": "..." } }
```

**Check:**
- `signature_url` is set. It is a link on the sandbox (add the base URL in front); it works for 1-2 hours, then
  answers 404 (that is by design).
- The signature is also sent to the KYC form: `POST /onb/kyc/refresh` now shows `"signature_provided": true` and
  `signature` is gone from `fields_needed`.

**Negative tests:**

| Send | Expected |
|---|---|
| `{ "image": "hello" }` | `400` "Invalid image." `reason: "validation"` |
| A 1 x 1 pixel PNG (about 70 bytes): `data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==` | `400` "Image is too small or too large." |
| A GIF or text dressed up as PNG | `400` "Invalid image." |
| More than 12 MB of body | `413` "The request is too large." |

### 8b. FATCA

```
POST {{baseUrl}}/onb/declarations
Authorization: Bearer {{token}}

{ "fatca": true }
```

**Expected:** `{ "user": { "next_step": "kyc-esign", "...": "..." } }` (once DigiLocker, bank and signature are done).

---

## Step 9. E-sign the KYC form  (BROWSER)

```
POST {{baseUrl}}/onb/kyc/esign
Authorization: Bearer {{token}}
```
(no body; can take up to 10 s)

**Expected (200):**

```json
{ "user": { "next_step": "kyc-esign", "...": "..." }, "esign_url": "https://...", "return_url_prefix": "https://risips-demo-api.azurewebsites.net/kyc/return" }
```

If the form is already e-signed: `{ "user": {...}, "esign_url": null }`.

In the browser:

1. Open `esign_url`.
2. Tick the confirmation box, tap **esign**.
3. Enter **any 6-digit OTP**.
4. The page ends on "Returning to Risips…" (`/kyc/return?step=esign`). Close the tab.

**Negative tests:**

| When | Expected |
|---|---|
| Before DigiLocker is done | `400` `{ "error": "Your KYC form is not complete yet (missing: address, identity proof).", "next_step": "aadhaar" }` (verified) |
| Before the signature is sent | `400` "Your KYC form is not complete yet (missing: signature)." with `next_step` |
| Form expired or failed | `422` with `kyc_status`, e.g. "Your KYC form expired before it was finished. Please start again." |

## Step 10. Refresh after e-sign

```
POST {{baseUrl}}/onb/kyc/refresh
```

**Expected:** `kyc.status` is `"awaiting_submission"` or `"submitted"`, and `next_step` is `"nominee"`.
`user.kyc_status` becomes `"with_kra"` once the form is `submitted`.

## Steps 11-12. Nominee and eSign

Exactly as file 02, Steps 7 and 8 (`POST /txn/otp { "action": "nominee" }`, `POST /onb/nominee`,
`POST /onb/esign { "consent": true }`). The account is created (`onboarded: true`).

Before every earlier step is done, `POST /onb/esign` names the missing step, e.g.
`400 { "error": "Please complete the Aadhaar step first.", "next_step": "aadhaar" }` (verified).

---

## Step 13. Waiting for the KRA

Until the KRA approves the KYC (usually 1-3 working days), **every order, SIP, STP, SWP, AutoPay and cart call is
refused**:

```json
{ "error": "Your KYC is with the KRA for approval (usually 1–3 working days). You can invest once it is approved.",
  "kyc_status": "with_kra" }
```
(HTTP `403`)

Before onboarding is finished, the same calls answer `403 { "error": "Please complete your KYC to invest." }`
(verified, e.g. `POST /invest/lumpsum` and `POST /cart`).

Check the KYC status:

```
GET {{baseUrl}}/onb/kyc/status
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "user": { "kyc_status": "with_kra", "...": "..." }, "kyc": { "status": "submitted", "...": "..." }, "error": null }`

| `kyc_status` | Meaning |
|---|---|
| `in_progress` | Form not submitted yet |
| `with_kra` | Submitted, waiting for the KRA |
| `compliant` | Approved: the user can invest. A "Your KYC is approved" notification appears |
| `failed` | Form failed or expired: redo it from DigiLocker (`POST /onb/kyc/start`) |

The server re-checks waiting users every 30 minutes (at most once per 10 minutes per user). It is **not known** whether
the sandbox KRA ever approves a sandbox fresh KYC, so a user may stay `with_kra`. That is not a bug in itself; note
how long it stayed.

## Quick checklist

- [ ] PAN without `fresh_kyc` -> 422 "No KYC record found ..."
- [ ] PAN with `fresh_kyc: true` -> 200, `kyc_status: "in_progress"`
- [ ] `kyc/start` outside India / without location -> 400
- [ ] DigiLocker in browser -> refresh shows `proof_status: "fetched"`, `next_step: "bank"`
- [ ] Signature (sample above) -> `signature_provided: true`
- [ ] `kyc/esign` -> e-sign in browser -> refresh shows `awaiting_submission` / `submitted`
- [ ] Nominee + eSign -> `onboarded: true`, `kyc_status: "with_kra"`
- [ ] Any order -> 403 "Your KYC is with the KRA ..."
