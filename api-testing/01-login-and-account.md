# 01. Sign up, sign in and the account

Order: **email code -> mobile number -> PIN**. After that the user goes on to onboarding (file 02 or 03).

Base URL in every example: `https://risips-demo-api.azurewebsites.net` (Postman: `{{baseUrl}}`).
All bodies are JSON with `Content-Type: application/json`. Read [README.md](README.md) first for Postman set-up,
the error format and the limits.

| Step | Call | Needs token? |
|---|---|---|
| 1 | `POST /auth/email/start` | No |
| 2 | `POST /auth/email/verify` | No (gives you the token) |
| 3 | `POST /auth/mobile/start` | Yes |
| 4 | `POST /auth/mobile/verify` | Yes |
| 5 | `POST /auth/pin/set` | Yes |
| 6 | `POST /auth/pin/verify` (app unlock) | Yes |
| 7 | `GET /me` | Yes |
| 8 | `POST /auth/biometric` | Yes |
| 9 | `POST /auth/logout` | Yes |

---

## Step 1. Ask for an email code

**Purpose:** starts sign-up or sign-in. The sandbox returns the code as `dev_otp`.

```
POST {{baseUrl}}/auth/email/start
Content-Type: application/json

{ "email": "qa.m3y2nd@example.com" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/auth/email/start \
  -H "Content-Type: application/json" \
  -d '{"email":"qa.m3y2nd@example.com"}'
```

**Expected (200):**

```json
{ "ok": true, "expires_in": 120, "existing": false, "code_length": 6, "dev_otp": "454804" }
```

**Check:**
- `existing` is `false` for a new email and `true` for an email that already has an account.
- `dev_otp` has 6 digits. `expires_in` is 120 (seconds).

**Negative tests:**

| Send | Expected |
|---|---|
| The same email again within 30 s | `429` `{ "error": "A code was sent just now. Please try again in 30 seconds.", "reason": "rate_limited", "retry_after": 30 }` |
| `{ "email": "bad@foo.xyz" }` (ending not allowed) | `400` `{ "error": "Enter a valid email address.", "reason": "validation", "fields": { "email": "Enter a valid email address." } }` |
| `{ "email": "not-an-email" }` | same 400 |

Allowed endings: `.com .in .org .co .net .edu .me .uk`. Capital letters and spaces around the email are fixed for you
(`QA.Test@Example.com ` = `qa.test@example.com`).

---

## Step 2. Verify the email code (get the token)

**Purpose:** creates the account if it is new, signs in and returns the session **token**.

```
POST {{baseUrl}}/auth/email/verify
Content-Type: application/json

{ "email": "qa.m3y2nd@example.com", "code": "454804" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/auth/email/verify \
  -H "Content-Type: application/json" \
  -d '{"email":"qa.m3y2nd@example.com","code":"454804"}'
```

**Expected (200):**

```json
{
  "token": "<TOKEN>",
  "is_new": true,
  "user": {
    "id": 345, "email": "qa.m3y2nd@example.com", "mobile": null, "name": null,
    "pan_masked": null, "kyc_compliant": false, "kyc_fresh": false, "kyc_status": null,
    "biometric": false, "pin_set": false, "onboarded": false, "onboarded_at": null,
    "member_since": "2026", "next_step": "mobile", "fp": null, "...": "..."
  }
}
```

**Check:**
- `token` is present (32 characters). Save it (the Postman script in the README does this).
- `is_new` is `true` the first time, `false` when signing in again.
- `user.next_step` is `"mobile"` for a new account.
- If the account had asked to be deleted, signing in cancels that and the response also has `"deletion_cancelled": true`
  (see file 07).

**Negative tests:**

| Send | Expected |
|---|---|
| A wrong code, e.g. `"000000"` | `400` `{ "error": "Incorrect code. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }` |
| Wrong codes again | `attempts_left` goes 3, 2, 1, then `"Incorrect code. No attempts left: request a new code."` with `attempts_left: 0` |
| Any code after that (even the right one) | `400` `{ "error": "Too many attempts. Request a new code.", "reason": "locked" }` |
| `"code": "12"` (too short) | `400` `{ "error": "Enter the code we sent you.", "reason": "validation", "fields": { "code": "..." } }` |
| A code more than 2 minutes old | `400` "Code expired. Tap Resend Code." `reason: "expired"` |
| A code that was already used, or no code asked | `400` "Please request a new code." `reason: "missing"` |
| 10 wrong codes within one hour for this email (e.g. 5 wrong on one code, new code after 30 s, 5 wrong again) | `429` `{ "error": "Too many wrong codes. Please try again in 59 minutes.", "reason": "rate_limited", "retry_after": 3507 }` on **both** start and verify. Use a throw-away email for this test |

---

## Step 3. Ask for a mobile code

**Purpose:** links a mobile number to the account. Needs the token.

```
POST {{baseUrl}}/auth/mobile/start
Authorization: Bearer {{token}}
Content-Type: application/json

{ "mobile": "+91 79262 24018" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/auth/mobile/start \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"mobile":"+91 79262 24018"}'
```

**Expected (200):**

```json
{ "ok": true, "expires_in": 120, "code_length": 6, "dev_otp": "623956" }
```

**The `+91` / `0` rules:** spaces, dashes, `+91` and a leading `0` are removed, and the last 10 digits are kept.
So `+91 79262 24018`, `+917926224018`, `07926224018` and `7926224018` are all the **same** number, stored as
`7926224018`.

**Negative tests:**

| Send | Expected |
|---|---|
| `"12345"` | `400` `{ "error": "Enter a valid 10-digit mobile number.", "reason": "validation", "fields": { "mobile": "..." } }` |
| `"9999999999"` (one digit repeated) | same 400 |
| A number starting with 0-5 after cleaning, e.g. `"5123456789"` | same 400 |
| A number already verified on another account (in any format: `+91...`, `0...`) | `409` `{ "error": "This mobile number is already registered with another Risips account. Please use a different number.", "reason": "mobile_taken" }`. **No code is sent** (no `dev_otp`) |
| Same number again within 30 s | `429` "A code was sent just now. ..." |
| After onboarding, a different number | `409` "Your mobile number is already registered. Contact support to change it." |

How to test `mobile_taken`: finish step 4 on account A, sign up a second email (account B), and send account A's
number from account B, also as `+91<number>` and `0<number>`.

---

## Step 4. Verify the mobile code

```
POST {{baseUrl}}/auth/mobile/verify
Authorization: Bearer {{token}}
Content-Type: application/json

{ "mobile": "07926224018", "code": "623956", "bank_consent": true }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/auth/mobile/verify \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
  -d '{"mobile":"07926224018","code":"623956","bank_consent":true}'
```

**Expected (200):** `{ "user": { "mobile": "7926224018", "next_step": "pin", "...": "..." } }`

**Check:**
- `user.mobile` is the 10-digit number (no `+91`, no `0`).
- `next_step` is `"pin"`.
- `bank_consent` is optional (`true` / `false`).

**Negative tests:**

| Send | Expected |
|---|---|
| Wrong code | `400` "Incorrect code. 4 attempts left." `reason: "wrong"`, `attempts_left: 4` |
| A different number from the one the code was sent to | `400` `{ "error": "Please request a new code.", "reason": "missing" }` |
| A number someone else verified in the meantime | `409` `reason: "mobile_taken"` |

---

## Step 5. Set the PIN

```
POST {{baseUrl}}/auth/pin/set
Authorization: Bearer {{token}}

{ "pin": "1357" }
```

```bash
curl -X POST https://risips-demo-api.azurewebsites.net/auth/pin/set \
  -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" -d '{"pin":"1357"}'
```

**Expected (200):** `{ "user": { "pin_set": true, "next_step": "pan", "...": "..." } }`

**Negative tests:**

| Send | Expected |
|---|---|
| `"12a4"` or `"123"` or `"12345"` | `400` `{ "error": "PIN must be 4 digits.", "reason": "validation", "fields": { "pin": "PIN must be 4 digits." } }` |

---

## Step 6. Check the PIN (app unlock)

**Purpose:** what the app does when it is opened again.

```
POST {{baseUrl}}/auth/pin/verify
Authorization: Bearer {{token}}

{ "pin": "1357" }
```

**Expected (200):** `{ "ok": true }`

**Negative tests:**

| Send | Expected |
|---|---|
| Before a PIN is set | `400` `{ "error": "No PIN set." }` |
| Wrong PIN | `400` `{ "error": "Incorrect PIN. 4 attempts left.", "attempts_left": 4 }` (no `reason` field here) |
| Wrong PIN again | 3, 2, then `"Incorrect PIN. 1 attempt left."` |
| 5th wrong PIN in a row | `401` `{ "error": "Too many wrong PIN attempts. Please sign in again with your email.", "reason": "pin_locked" }` |
| Any call with the same token after that | `401` "Please sign in again." |

After `pin_locked`: sign in again with steps 1-2 (wait 30 s after the last email code). The PIN stays the same and
the right PIN works again.

---

## Step 7. Who am I? (`GET /me`)

```
GET {{baseUrl}}/me
Authorization: Bearer {{token}}
```

```bash
curl https://risips-demo-api.azurewebsites.net/me -H "Authorization: Bearer <TOKEN>"
```

**Expected (200):** `{ "user": { ... } }` (the same user object as above).

**Check:** `next_step` tells you what to do next. Use `GET /me` whenever you are not sure where an account is.

**Negative test:** no `Authorization` header, or a made-up token -> `401 { "error": "Please sign in again." }`.

---

## Step 8. Biometric on / off (optional)

```
POST {{baseUrl}}/auth/biometric
Authorization: Bearer {{token}}

{ "enabled": true }
```

**Expected (200):** `{ "user": { "biometric": true, "...": "..." } }`. `"enabled": "yes"` -> 400 "Must be true or false."

---

## Step 9. Log out

```
POST {{baseUrl}}/auth/logout
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "ok": true }`

**Check:** `GET /me` with the same token now answers `401 "Please sign in again."`.

Other devices can be signed out from Settings (file 07).

---

## Quick end-to-end check (new account)

1. `POST /auth/email/start` -> copy `dev_otp`.
2. `POST /auth/email/verify` -> `is_new: true`, `next_step: "mobile"`, save `token`.
3. `POST /auth/mobile/start` with a new number -> copy `dev_otp`.
4. `POST /auth/mobile/verify` -> `next_step: "pin"`.
5. `POST /auth/pin/set` -> `next_step: "pan"`.
6. Continue with file 02 (KYC-compliant PAN) or file 03 (fresh KYC).
