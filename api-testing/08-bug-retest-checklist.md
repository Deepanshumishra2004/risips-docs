# 08. Bug re-test checklist

How to re-test each bug that was reported and fixed (see `backend/CHANGELOG.md` in this docs folder). For each one:
what to send, and what you should get **now**.

Set-up for all of them: README (Postman environment, auto-save script). Use new emails, mobiles and bank numbers.
"Verified 2026-11-03" means the expected result below was seen on the sandbox while writing this guide.

| Id | Short title | Status on the sandbox 2026-11-03 |
|---|---|---|
| RSP-AUTH-001 | Mobile with `+91` / `0` / spaces | Verified |
| RSP-AUTH-002 | New OTP sent while wrong codes are blocked | Verified |
| RSP-AUTH-003 | Wrong code doesn't say tries left | Verified |
| RSP-AUTH-004 | Same mobile on two accounts | Verified |
| RSP-ONB-001 | "Edit PAN" doesn't clear the PAN state | Verified |
| RSP-ONB-002 | DOB error doesn't name the format | Verified |
| RSP-ONB-003 | Full PAN visible in responses | Verified |
| RSP-ONB-004 | Junk pincodes accepted (999999) | Verified |
| RSP-ONB-005 | No clear message when FP blocks a bank account | Verified with the already-blocked number `123456781195` (see below) |
| RSP-ONB-006 | Fresh KYC (no KRA record) can't continue | Verified up to the DigiLocker link; DigiLocker onwards needs a browser |
| RSP-FND-001 | Compare accepts a 3rd fund | Verified |
| RSP-FND-002 | NAV frozen at an old date | Verified |
| RSP-FND-003 | Wrong risk level (debt fund "Very High Risk") | Verified |
| RSP-FND-004 | Gold / silver funds shown as Index | Verified |

---

## RSP-AUTH-001: mobile number formats

**Steps**
1. New account (file 01, Steps 1-2).
2. `POST /auth/mobile/start { "mobile": "+91 79262 24018" }` (use your own new number) -> copy `dev_otp`.
3. `POST /auth/mobile/verify { "mobile": "07926224018", "code": "<dev_otp>" }`.

**Expected now**
- [ ] Step 2: `200`, `dev_otp` present.
- [ ] Step 3: `200`, `user.mobile` is `"7926224018"` (10 digits, no `+91`, no `0`). The code sent for the `+91` form is
      accepted for the `0` form because it is the same number.
- [ ] `"mobile": "12345"` -> `400` "Enter a valid 10-digit mobile number." `reason: "validation"`.

## RSP-AUTH-002: no new code while blocked

Use a **throw-away** email: it will be blocked for about an hour.

**Steps**
1. `POST /auth/email/start { "email": "<throw-away>" }`.
2. `POST /auth/email/verify` with wrong codes 5 times (`000001` ... `000005`). The 6th wrong try -> `reason: "locked"`.
3. Wait 30 s. `POST /auth/email/start` again (new code). 5 wrong codes again (10 wrong in total).
4. Wait 30 s. `POST /auth/email/start` again.

**Expected now**
- [ ] Step 4: `429` `{ "error": "Too many wrong codes. Please try again in 59 minutes.", "reason": "rate_limited", "retry_after": 3507 }` (about 3,500 s).
- [ ] **No `dev_otp`** in that response (no code was sent).
- [ ] `POST /auth/email/verify` with any code also answers the same `429` while blocked.

## RSP-AUTH-003: tries left on a wrong code

**Steps**
1. `POST /auth/email/start`, then `POST /auth/email/verify` with a wrong code, 5 times.
2. For transaction codes: onboarded user, `POST /txn/otp { "action": "lumpsum" }`, then
   `POST /invest/lumpsum { "isin": "INF109KC1Q80", "amount": 1000, "otp": "000000" }`.

**Expected now**
- [ ] Email code, 1st wrong: `400` `{ "error": "Incorrect code. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }`.
- [ ] Then 3, 2, "Incorrect code. 1 attempt left.", then "Incorrect code. No attempts left: request a new code."
      (`attempts_left: 0`), then `reason: "locked"` "Too many attempts. Request a new code."
- [ ] Transaction code: `400` `{ "error": "That OTP is not correct. 4 attempts left.", "reason": "wrong", "attempts_left": 4 }`
      (verified with the nominee code).

## RSP-AUTH-004: one mobile per account

**Steps**
1. Account A: verify mobile `N` (file 01).
2. Account B (new email): `POST /auth/mobile/start` with `N`, then `+91N`, then `0N`.

**Expected now**
- [ ] All three: `409` `{ "error": "This mobile number is already registered with another Risips account. Please use a different number.", "reason": "mobile_taken" }`.
- [ ] No `dev_otp` in these responses (no code sent).
- [ ] Account B can verify a different new number normally.

## RSP-ONB-001: "No, edit PAN"

**Steps**
1. `POST /onb/pan { "pan": "<XXXPX3751X>", "name": "Qa Tester", "dob": "1992-04-15" }` -> `kyc_status: "compliant"`.
2. `POST /onb/pan/reset`.
3. `GET /me`.
4. `POST /onb/pan` again with the same values, then `POST /onb/pan/confirm`, then `POST /onb/pan/reset` again.

**Expected now**
- [ ] Step 2: `pan_masked: null`, `name: null`, `dob: null`, `kyc_status: null`, `kyc_compliant: false`, `next_step: "pan"`.
- [ ] Step 4: the PAN step works again; after confirm, reset answers `409` "Your PAN is already confirmed and can not be edited."
- [ ] For a fresh-KYC PAN (`XXXPX3753X` with `fresh_kyc: true`), reset (before confirm) also clears `kyc_fresh` and
      the KYC form (`kyc_status: null`). (From the changelog; not re-run for this guide.)

## RSP-ONB-002: date-of-birth format in errors

**Steps**
1. `POST /onb/pan { "pan": "<valid>", "name": "Qa Tester", "dob": "15-04-1992" }`.
2. Nominee: `POST /onb/nominee { "name": "Sita Devi", "dob": "20-08-1965", "relationship": "mother", "pan": "ABCPD1234E", "otp": "<code>" }`.

**Expected now**
- [ ] Step 1: `400`, `fields.dob`: "Enter a valid date of birth (YYYY-MM-DD)."
- [ ] Step 2: `400` "Enter a valid date of birth for the nominee (YYYY-MM-DD)." (the code is not used up).

## RSP-ONB-003: the full PAN never leaves the server

**Steps** (search each response for your full PAN, e.g. with Postman's search in the response body)
1. `POST /onb/pan` (200 and 422 answers).
2. `GET /me`, `GET /onb/kyc-form`.
3. After a nominee: the `nominee` in `GET /me`.
4. `GET /reports/holdings`.
5. `GET /external`.

**Expected now**
- [ ] `pan_masked` is like `*******51Q`; `checks.pan.value` is masked the same way, also in 422 answers.
- [ ] `nominee.pan` is masked (`*******34E`).
- [ ] Reports: `investor.pan` masked.
- [ ] `GET /external`: `pan: null`, `pan_masked` like `TG*****51Q`.
- [ ] Your full PAN appears in **none** of these responses.

## RSP-ONB-004: junk pincodes

**Steps:** `GET /onb/pincode/999999`, `GET /onb/pincode/123456`, `GET /onb/pincode/560001`, then
`POST /onb/address { "line1": "12, 4th Cross, Indiranagar", "pincode": "999999", "gender": "female" }`.

**Expected now**
- [ ] `999999` and `123456`: `404` `{ "error": "Pincode not found." }`.
- [ ] `560001`: `200` `{ "city": "Bengaluru", "state": "KARNATAKA" }`.
- [ ] Address with `999999`: `400` `{ "error": "Pincode not found." }`.

## RSP-ONB-005: message when FP blocks a bank account

FP is supposed to block an account number after 3 failed checks, and then the API should say:
"This account has had too many verification attempts and can’t be verified right now. Please use another account in
your name."

**Steps**
1. User at the bank step (or any user before onboarding is finished).
2. `POST /onb/bank { "account_number": "123456781195", "ifsc": "HDFC0000001" }`. This number is already blocked at
   FP for everyone (it failed too often in earlier testing), so the first call shows the message.

**Expected now**
- [ ] `422`, `code: "verification_attempt_limit_exceeded"`, "This account has had too many verification attempts and
      can’t be verified right now. Please use another account in your name." **Verified on the sandbox 2026-11-03.**
- [ ] Then a unique number ending `11XX` (e.g. `70` + 8 random digits + `1195`, IFSC `HDFC0000240`) verifies (`200`).

**Note:** making a new number fail 3-4 times (ending `31XX`) did **not** make FP block it on 2026-11-03: all calls
answered "Bank verification failed …". FP's limit is on FP's side and isn't documented, so use the number above to
see the message. (Also seen: a failed check does not remove a bank account that was already verified.)

## RSP-ONB-006: fresh KYC

**Steps:** follow file 03.

**Expected now**
- [ ] `POST /onb/pan` with a `XXXPX3753X` PAN and **no** flag: `422` "No KYC record found for this PAN with the KRA. A fresh KYC is needed." (by design).
- [ ] With `"fresh_kyc": true`: `200`, `kyc_fresh: true`, `kyc_status: "in_progress"`, `next_step: "pan-confirm"`.
- [ ] After confirm, profile and address: `next_step: "aadhaar"`.
- [ ] `POST /onb/kyc/start { "latitude": 12.9716, "longitude": 77.5946 }`: `200` with a `fetch_url`.
- [ ] DigiLocker in the browser, then `POST /onb/kyc/refresh`: `proof_status: "fetched"`, `next_step` moves on (needs a browser).
- [ ] Signature, declarations, KYC e-sign, nominee, eSign: `onboarded: true`, `kyc_status: "with_kra"` (needs a browser).
- [ ] Any order while waiting: `403` "Your KYC is with the KRA for approval ...".

Common mistake: forgetting `"fresh_kyc": true` in the Postman body. That gives the 422 above, which is **not** the bug.

## RSP-FND-001: compare at most 2 funds

**Steps:** `GET /funds/compare?isins=INF109KC1Q80,INF209K01AJ8,INF109K01VQ1`, then with only the first two.

**Expected now**
- [ ] 3 funds: `400` `{ "error": "You can compare up to 2 funds.", "reason": "validation", "fields": { "isins": "..." } }`.
- [ ] 2 funds: `200` with 2 funds in `funds`.

## RSP-FND-002: daily NAV

**Steps:** `GET /funds`, `GET /funds/INF109KC1Q80`, `GET /funds/compare?isins=INF109KC1Q80,INF209K01AJ8`, `GET /cart`
(with a line in it).

**Expected now**
- [ ] Every fund's `nav_date` is the **last working day** (on 2026-11-03 all 35 funds showed `2026-11-02`), not
      September 2026.
- [ ] The fund page's last `chart` point equals `nav`.
- [ ] `returns` has `1D`, `1M`, `6M`, `1Y`, `3Y`, `5Y` (numbers, or `null` for young funds).
- [ ] Re-check the next working day: `nav_date` should move forward (the server refreshes from AMFI every hour once the
      data is 4 hours old).

## RSP-FND-003: real riskometer

**Steps:** `GET /funds?category=Debt`, `GET /funds/INF109KC1UC4` (ICICI Pru Retirement Fund Pure Debt Plan).

**Expected now**
- [ ] Debt funds are **not** "Very High Risk". Seen 2026-11-03: Money Market, Liquid and Gilt "Low to Moderate Risk";
      the corporate bond funds and ABSL Liquid "Moderate Risk".
- [ ] ICICI Pru Retirement Fund Pure Debt Plan: "Moderately High Risk" (it used to show "Very High Risk").
- [ ] Every fund has one of the 6 SEBI labels: `Low Risk`, `Low to Moderate Risk`, `Moderate Risk`,
      `Moderately High Risk`, `High Risk`, `Very High Risk`.

## RSP-FND-004: Commodity category

**Steps:** `GET /funds?category=Commodity`, `GET /funds?category=Index`.

**Expected now**
- [ ] Commodity: 4 funds: ICICI Pru Silver ETF FOF (`INF109KC1Y64`), ABSL Silver ETF FOF (`INF209KB13F9`), ABSL Gold
      Fund (`INF209K01PF4`), ICICI Pru Gold ETF FOF (`INF109K01TK8`).
- [ ] None of those 4 is in the Index list (Index has 5 funds: Nifty Pharma, Nifty Next 50, two Nifty 50, Nifty50
      Value 20).
- [ ] Note: `GET /funds?collection=commodity` has 5 (it adds ICICI Pru Multi Asset Allocation Fund, a Hybrid fund
      tagged commodity). Report it only if the product owner says it should not be there.

---

## Other things noticed while writing this guide (worth a ticket)

| What | Seen |
|---|---|
| AutoPay limit ignored on a repeat call | While an AutoPay is still `CREATED`, a new `POST /mandate` with a higher `limit` reuses the old AutoPay with its **old** limit (file 06, A2) |
| AutoPay `limit` below 1,000 | Silently raised to 1,000, no error |
| Unknown fund `period` | `GET /funds/:isin?period=2Y` answers 200 with 1Y instead of a 400 |
| Mask formats differ | User / reports: `*******51Q`. MF Central status: `TG*****51Q`. Bank: user `**********1195`, `GET /mandate` `****1195` |
| File links with a malformed `sig` | `GET /files/...&sig=bad` answers `400` with a technical message ("Invalid string: must match pattern ...") instead of the documented empty `404` |
| Bank `12XX` accounts | Stayed "pending" (422 "taking longer than usual") instead of verifying |
