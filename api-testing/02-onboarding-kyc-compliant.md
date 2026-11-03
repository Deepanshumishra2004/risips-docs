# 02. Onboarding with a KYC-compliant PAN

This is the normal path: the PAN already has a good KYC record at the KRA. Use a PAN like `XXXPX3751X` (make up the
`X` letters, keep `P` as the 4th letter and avoid `A` / `I` as the 5th), e.g. `TGVPS3751Q`. The examples in this
file already belong to the guide's test user: make up your own letters each time.

**Before you start:** do file 01 until `next_step` is `"pan"` (email, mobile and PIN done). Every request here needs
`Authorization: Bearer {{token}}` and `Content-Type: application/json`.

**Whole run verified on the sandbox on 2026-11-03** (user 345): PAN -> profile -> address -> bank -> declarations ->
nominee -> eSign -> `onboarded: true`. No browser is needed for this path.

## Which API comes next? (`next_step`)

Every onboarding response returns `user.next_step`. Look it up here:

| `next_step` | What to call | Notes |
|---|---|---|
| `mobile` | `POST /auth/mobile/start` + `/auth/mobile/verify` | File 01 |
| `pin` | `POST /auth/pin/set` | File 01 |
| `pan` | `POST /onb/pan` | Step 1 |
| `pan-confirm` | `POST /onb/pan/confirm` (or `POST /onb/pan/reset` to edit) | Step 2 |
| `occupation` | `POST /onb/profile` with `occupation` | Step 3 |
| `income` | `POST /onb/profile` with `income_slab` | Step 3 |
| `marital` | `POST /onb/profile` with `marital_status` | Step 3 |
| `father` | `POST /onb/profile` with `father_name` | Step 3 |
| `address` | `POST /onb/address` (optional lookup `GET /onb/pincode/:pin` first) | Step 4 |
| `aadhaar` | Fresh KYC only: `POST /onb/kyc/start` (DigiLocker) | File 03 |
| `bank` | `POST /onb/bank` (optional lookup `GET /onb/ifsc/:ifsc` first) | Step 5 |
| `selfie-intro` | Only old sandbox accounts: `POST /onb/selfie` | Not used by new accounts |
| `signature-consent` | `POST /onb/declarations` (fresh KYC also `POST /onb/signature`) | Step 6 |
| `kyc-esign` | Fresh KYC only: `POST /onb/kyc/esign` | File 03 |
| `nominee` | `POST /txn/otp {action:"nominee"}` then `POST /onb/nominee` | Step 7 |
| `esign` | `POST /onb/esign` | Step 8 |
| `home` | Done. The user can invest | |

For a KYC-compliant PAN you will see: `pan -> pan-confirm -> occupation -> income -> marital -> father -> address ->
bank -> signature-consent -> nominee -> esign -> home`. No Aadhaar, selfie or signature.

---

## Step 1. PAN, name and date of birth

**Purpose:** checks the PAN, name and date of birth with the fund platform, and checks the PAN's KYC status.

```
POST {{baseUrl}}/onb/pan
Authorization: Bearer {{token}}
Content-Type: application/json

{ "pan": "TGVPS3751Q", "name": "Qa Tester", "dob": "1992-04-15" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/pan \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"pan":"TGVPS3751Q","name":"Qa Tester","dob":"1992-04-15"}'
```

Takes 1-7 seconds.

**Expected (200):**

```json
{
  "user": {
    "name": "Qa Tester", "first_name": "Qa", "dob": "1992-04-15",
    "pan_masked": "*******51Q",
    "kyc_compliant": true, "kyc_fresh": false, "kyc_status": "compliant",
    "next_step": "pan-confirm", "...": "..."
  },
  "checks": {
    "readiness":     { "code": null, "modification": null, "reason": null, "status": "verified" },
    "pan":           { "code": null, "reason": null, "status": "verified", "value": "*******51Q" },
    "name":          { "code": null, "reason": null, "status": "verified", "value": "Qa Tester" },
    "date_of_birth": { "code": null, "reason": null, "status": "verified", "value": "1992-04-15" }
  },
  "pv_id": "pv_3ad4a3e6eb8945e6b0a7b6d3950805b9",
  "kyc_fresh": false
}
```

**Check:**
- `kyc_status` is `"compliant"`, `kyc_compliant` is `true`, `next_step` is `"pan-confirm"`.
- The full PAN appears **nowhere** in the response: `pan_masked` and `checks.pan.value` show only the last 3 characters.
- All four `checks` are `"verified"`.

**Negative tests** (each one is a real result from the sandbox):

| Send | Expected |
|---|---|
| `"pan": "ABCDE1234F"` (4th letter not `P`), `"dob": "15-04-1992"` | `400` `reason: "validation"`, `fields.pan`: "Enter a valid individual PAN (e.g. ABCPX1234K).", `fields.dob`: "Enter a valid date of birth (YYYY-MM-DD)." |
| `"dob": "2012-01-01"` (under 18) | `400` "You must be 18 or older to open an account. Investing for a minor is not supported yet." |
| `"name": "Lord Voldemort"` | `422` "The name does not match PAN records. Enter it exactly as on your PAN card." (`checks.name.code: "mismatch"`) |
| `"dob": "2000-01-01"` | `422` "The date of birth does not match PAN records." (`checks.date_of_birth.code: "mismatch"`) |
| PAN with 5th letter `I`, e.g. `QWRPI3751K` | `422` "This PAN is invalid. Please check and re-enter." (`checks.pan.code: "invalid"`) |
| PAN with 5th letter `A`, e.g. `QWRPA3751K` | `422` "Your PAN is not linked with Aadhaar. Link it on the income-tax portal and try again." |
| `QWRPX3753K` (no KRA record) without `fresh_kyc` | `422` "No KYC record found for this PAN with the KRA. A fresh KYC is needed." (see file 03) |
| `QWRPX3754K` | `422` "Your KYC is on hold at the KRA and needs a modification." |
| `QWRPX3755K` | `422` "Your KYC was rejected by the KRA. A fresh KYC is needed." |
| `QWRPX3756K` | `422` "Your KYC is deactivated. You cannot invest right now." |
| `QWRPX1234K` (other digits) | `422` "We could not verify your PAN right now. Please try again." (`checks.readiness.code: "unknown"`) |
| Any `POST /onb/pan` after Step 2 (confirm) | `409` "Your PAN is already confirmed and can not be edited." |

Every 422 also carries `checks` and `pv_id` (with the PAN masked). A failed PAN check does not move `next_step`.

Note: the PAN check is limited to 30 per hour per user. Spread the negative tests over two test users if needed.

---

## Step 2. Confirm the PAN (or edit it)

### 2a. "Yes, confirm"

```
POST {{baseUrl}}/onb/pan/confirm
Authorization: Bearer {{token}}
```
(no body)

**Expected (200):** `{ "user": { "next_step": "occupation", "...": "..." } }`

After this the PAN can't be changed: `POST /onb/pan` and `POST /onb/pan/reset` both answer
`409 "Your PAN is already confirmed and can not be edited."`.

Without a verified PAN first: `400 "Verify your PAN first."`.

### 2b. "No, edit PAN" (do this **before** confirm)

```
POST {{baseUrl}}/onb/pan/reset
Authorization: Bearer {{token}}
```

**Expected (200):**

```json
{ "user": { "pan_masked": null, "name": null, "dob": null, "kyc_status": null, "kyc_compliant": false, "next_step": "pan", "...": "..." } }
```

**Check:** PAN, name, DOB and KYC state are all cleared and `next_step` is back to `"pan"`. Send Step 1 again.

---

## Step 3. Profile questions

You can send one answer per call (like the app screens) or all at once.

```
POST {{baseUrl}}/onb/profile
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "occupation": "private_sector_service",
  "income_slab": "above_5lakh_upto_10lakh",
  "marital_status": "single",
  "father_name": "Ramesh Kumar"
}
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/profile \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"occupation":"private_sector_service","income_slab":"above_5lakh_upto_10lakh","marital_status":"single","father_name":"Ramesh Kumar"}'
```

**Expected (200):** `{ "user": { "occupation": "private_sector_service", "...": "...", "next_step": "address" } }`

Sending only `{ "occupation": "private_sector_service" }` gives `next_step: "income"`, and so on.

**Allowed values:**

| Field | Values |
|---|---|
| `occupation` | `private_sector_service`, `public_sector_service`, `government_service`, `house_wife`, `student`, `business`, `professional`, `retired`, `agriculture`, `service`, `others` |
| `income_slab` | `upto_1lakh`, `above_1lakh_upto_5lakh`, `above_5lakh_upto_10lakh`, `above_10lakh_upto_25lakh`, `above_25lakh_upto_1cr`, `above_1cr` |
| `marital_status` | `single`, `married`, `others` |
| `father_name` | 3-70 letters, spaces, `.` or `'` |

**Negative tests:**

| Send | Expected |
|---|---|
| `{ "occupation": "pilot" }` | `400` "Invalid occupation." `reason: "validation"` |
| `{ "income_slab": "lots" }` | `400` "Invalid income slab." (from the API reference, not re-run) |
| `{ "marital_status": "divorced" }` | `400` "Invalid marital status." (not re-run) |
| `{ "father_name": "R2" }` | `400` "Enter your father's or spouse's full name." (not re-run) |

---

## Step 4. Address

### 4a. Pincode lookup (optional; the app uses it to fill city and state)

```
GET {{baseUrl}}/onb/pincode/560001
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "city": "Bengaluru", "state": "KARNATAKA" }`  (and `110001` -> `{ "city": "Central Delhi", "state": "DELHI" }`)

| Pincode | Expected |
|---|---|
| `999999` | `404` `{ "error": "Pincode not found." }` |
| `123456` | `404` `{ "error": "Pincode not found." }` |
| `12345` | `400` `{ "error": "Enter a 6-digit pincode.", "reason": "validation", "fields": { "pin": "..." } }` |

### 4b. Save the address

```
POST {{baseUrl}}/onb/address
Authorization: Bearer {{token}}
Content-Type: application/json

{ "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001", "gender": "female" }
```

**Expected (200):**

```json
{
  "user": {
    "address": { "line1": "12, 4th Cross, Indiranagar", "line2": "", "pincode": "560001", "city": "Bengaluru", "state": "KARNATAKA" },
    "gender": "female",
    "next_step": "bank", "...": "..."
  }
}
```

**Check:** city and state are filled from the pincode (you don't send them). `gender` is saved here.

**Negative tests:**

| Send | Expected |
|---|---|
| `"pincode": "999999"` | `400` `{ "error": "Pincode not found." }` |
| `{ "line1": "12", "pincode": "560001" }` (short line, no gender) | `400` "Enter your house / street address.", `fields` has both `line1` and `gender` ("Select your gender.") |
| `"gender": "other"` | `400` "Select your gender." |

`gender` values: `male`, `female`, `transgender`. `line1` 3-120 characters, `line2` optional (max 120).

---

## Step 5. Bank account

### 5a. IFSC lookup (optional)

```
GET {{baseUrl}}/onb/ifsc/HDFC0000240
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI" }`

| IFSC | Expected |
|---|---|
| `ZZZZ0000001` | `404` `{ "error": "IFSC not found." }` |
| `HDFC123` | `400` `{ "error": "Enter a valid IFSC.", "reason": "validation", ... }` |

### 5b. Verify the account

Make up a **new** account number each time: 2 random front digits, 8 random digits, and the last 4 decide the result
(`1195` = verified). Example: `50` + `46218835` + `1195` = `50462188351195`.

```
POST {{baseUrl}}/onb/bank
Authorization: Bearer {{token}}
Content-Type: application/json

{ "account_number": "50462188351195", "ifsc": "HDFC0000240" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/bank \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"account_number":"50462188351195","ifsc":"HDFC0000240"}'
```

Takes about 2 seconds (up to 20).

**Expected (200):**

```json
{
  "user": {
    "bank": {
      "ifsc": "HDFC0000240", "bank_name": "HDFC BANK", "branch": "MUMBAI  SANDOZ HOUSE", "city": "MUMBAI",
      "pv_id": "pv_511182e029cb4629a79214993362b38d", "status": "verified", "code": null,
      "verified": true, "account_masked": "**********1195"
    },
    "next_step": "signature-consent", "...": "..."
  }
}
```

**Check:** `bank.verified` is `true`, only the last 4 digits are visible (`account_masked`), there is no
`account_number` field, `next_step` is `"signature-consent"`.

**Negative tests:**

| Send | Expected |
|---|---|
| `"account_number": "12345"` | `400` "Account number must be 9–18 digits." `reason: "validation"` |
| A new number ending `31XX`, e.g. `50` + 8 random digits + `3105` | `422` `{ "error": "Bank verification failed. Please check the account number and IFSC.", "bank_status": "failed", "code": "bank_verification_failed", "pv_id": "pv_..." }` |
| A new number ending `13XX`, e.g. `...1350` | `422` `{ "error": "We could not confirm this account belongs to you. Please use another account in your name.", "bank_status": "failed", "code": "uncertain", "pv_id": "..." }` |
| A new number ending `12XX`, e.g. `...1250` | On 2026-11-03 this did **not** verify: after 20 s, `422` "Bank verification is taking longer than usual. ..." with `bank_status: "pending"`. Use `11XX` for a passing account |
| `14XX` / `15XX` / `1600` | Expected `422` refused (low confidence); not re-run for this guide |
| The same failing number 3 times | after that FP blocks it: `422` "This account has had too many verification attempts and can’t be verified right now. Please use another account in your name." |
| `"ifsc": "ZZZZ0000001"` | `400` "IFSC not found." |
| Sandbox slow | `422` "Bank verification is taking longer than usual. Please try again, or use another account." Try again with another new number |

---

## Step 6. FATCA declaration

```
POST {{baseUrl}}/onb/declarations
Authorization: Bearer {{token}}
Content-Type: application/json

{ "fatca": true, "running_account": false }
```

**Expected (200):** `{ "user": { "next_step": "nominee", "...": "..." } }`

**Negative test:** `{ "fatca": false }` -> `400` "Please accept the FATCA / CRS declaration." `reason: "validation"`.

A KYC-compliant user does **not** need a signature.

---

## Step 7. Nominee (or opt out), with a transaction code

### 7a. Ask for the code

```
POST {{baseUrl}}/txn/otp
Authorization: Bearer {{token}}
Content-Type: application/json

{ "action": "nominee" }
```

**Expected (200):**

```json
{
  "ok": true, "action": "nominee", "expires_in": 300, "resend_in": 30,
  "sent_to": { "mobile": "+91 79••••018", "email": "q••••@example.com" },
  "dev_otp": "483157"
}
```

`{ "action": "foo" }` -> `400` "Unknown action." `reason: "validation"`.

### 7b. Add the nominee

```
POST {{baseUrl}}/onb/nominee
Authorization: Bearer {{token}}
Content-Type: application/json

{ "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "ABCPD1234E", "otp": "483157" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/nominee \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"name":"Sita Devi","dob":"1965-08-20","relationship":"mother","pan":"ABCPD1234E","otp":"483157"}'
```

**Expected (200):**

```json
{ "user": { "nominee": { "name": "Sita Devi", "dob": "1965-08-20", "relationship": "mother", "pan": "*******34E" },
            "next_step": "esign", "...": "..." } }
```

**Check:** the nominee PAN comes back masked. `next_step` is `"esign"`.

**Or opt out** (no nominee): `{ "skip": true, "otp": "<code>" }` -> `nominee_skipped: true`, `next_step: "esign"`.

`relationship` values: `father`, `mother`, `spouse`, `son`, `daughter`, `brother`, `sister`, `grand_father`,
`grand_mother`, `others`. The nominee PAN can be any PAN format (`ABCDE1234F`), it does not need `P`.

**Negative tests** (a failed check does not use up the code, except a wrong code):

| Send | Expected |
|---|---|
| No `otp` | `400` `{ "error": "Confirm this with the OTP sent to your registered mobile and email.", "reason": "otp_required" }` |
| `"dob": "20-08-1965"` | `400` "Enter a valid date of birth for the nominee (YYYY-MM-DD)." |
| Your own PAN as the nominee PAN | `400` "Nominee PAN must be different from yours." |
| A child, e.g. `"dob": "2020-08-20"` | `400` "Minor nominees need guardian details, which this demo does not collect. Please add an adult nominee or skip." |
| Your own name as the nominee name (same as the PAN name) | `400` "Nominee cannot be yourself." |
| `"otp": "000000"` | `400` `{ "error": "That OTP is not correct. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }` |
| The same code a second time (after success) | `400` "This OTP has already been used or expired. Request a new one." `reason: "missing"` |

---

## Step 8. eSign (creates the investor at the fund platform)

```
POST {{baseUrl}}/onb/esign
Authorization: Bearer {{token}}
Content-Type: application/json

{ "consent": true }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/onb/esign \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" -d '{"consent":true}'
```

**Expected (200):**

```json
{
  "fp": {
    "invp": "invp_1c40fba1...", "phone": "phone_cb3e...", "email": "email_bceb...", "addr": "addr_fa77...",
    "bank": "bac_edecb0fa...", "bank_old_id": 275, "relp": "relp_bac8...",
    "mfia": "mfia_16aa8e6268a04a0dbf06965a49422a92", "mfia_old_id": 229
  },
  "user": {
    "onboarded": true, "onboarded_at": "...", "next_step": "home",
    "fp": { "investor_profile": "invp_1c40fba1...", "investment_account": "mfia_16aa8e62..." }, "...": "..."
  }
}
```

**Check:**
- `user.onboarded` is `true`, `next_step` is `"home"`.
- `fp.mfia` (the investment account) is present. `relp` is there only when a nominee was added.
- Calling it again is safe: the same answer comes back at once.

**Negative tests:**

| Send | Expected |
|---|---|
| `{ "consent": false }` | `400` "Please agree to eSign the form." `reason: "validation"` |
| eSign before the other steps, e.g. right after the PAN | `400` `{ "error": "Please complete the PAN step first.", "next_step": "pan-confirm" }` (the step name changes with what is missing, e.g. "Please complete the bank account step first.") |
| Any other `POST /onb/*` after onboarding, e.g. `POST /onb/profile` | `409` "Your account is already set up. Contact support to change these details." |

The user can now invest: go to file 04 (funds) and file 05 (orders).
