# Backend changelog

Changes to `backend-ts` from 2026-10-30 to 2026-11-02, from the git history of `C:\risips-dev` (branch `dev`, plus
`origin/dev` for the sandbox switch). Commit ids in brackets.

## 2026-11-02

### Fund data: daily NAV, real riskometer, Commodity category [`428814b`]

- **RSP-FND-002, daily NAV.** `catalog.json`'s NAVs were frozen at its build date (25 Sep). The fund list, fund page,
  compare, cart and portfolio now show AMFI's latest NAV: `withLatestNav()` in `catalog.service.ts` replaces `nav`,
  `nav_date`, the 1D change and the 1M/6M/1Y/3Y/5Y returns when AMFI's NAV is newer, and adds a chart point. The new
  `amfi-navs` job refreshes AMFI's download hourly once it is more than 4 hours old. AMFI history now also gives 1M and
  6M returns.
- **RSP-FND-003, riskometer.** `risk` was our own estimate from the category (a pure-debt retirement plan showed
  "Very High Risk"). `scripts/fund-data.ts` now takes each fund's SEBI riskometer level from Kuvera's public data and
  records `risk_source` in `catalog.json`. All 35 funds have it; 6 changed. (`risk_source` is not in API responses.)
- **RSP-FND-004, Commodity.** Gold and silver fund-of-funds are category `Commodity`, not `Index`. Categories are now
  Equity, Debt, Hybrid, Index, Commodity, Solution; "Top Performer" ranks within each.
- `catalog.json` refreshed (TER, managers, portfolios, risk).

### Sandbox runs backend-ts [`1633397`, on `origin/dev`]

- The Azure sandbox (`https://risips-demo-api.azurewebsites.net`) runs `backend-ts` (deployed from `dev` at `2fe013a`)
  instead of the JavaScript backend, so fresh KYC can be tested there. This unblocked QA's **RSP-ONB-006** (fresh
  KYC): the earlier failures were the old backend plus a Postman body without `"fresh_kyc": true`. Verified with a
  `…3753…` sandbox PAN: without the flag `422 kyc_unavailable` (by design, for old app builds), with it `200`,
  `next_step: "pan-confirm"`, `kyc_status: "in_progress"`.

## 2026-10-30

### One mobile number per account [`b7f3429`]

- A mobile number that another live account has verified is refused at `POST /auth/mobile/start` (before a code is
  sent) and again at `POST /auth/mobile/verify`: `409`, `reason: "mobile_taken"`. Numbers are compared as stored
  (10 digits), so `+91` / `0` prefixes can't get around it. Erased accounts don't count; existing duplicates were not
  changed.

### Tester bug fixes [`30f0404`]

| Id | Change |
|---|---|
| RSP-AUTH-001 | Documented that `+91`, `0`-prefixed and spaced mobiles are stored as the last 10 digits |
| RSP-AUTH-002 | No new OTP is sent while too many wrong codes block checking: `429 reason: "rate_limited"` with `retry_after`. The per-code 5-try lock and the cross-code cap (10 wrong / hour, 20 / day) are documented |
| RSP-AUTH-003 | A wrong sign-in or transaction code says how many tries are left: `attempts_left`, "Incorrect code. 3 attempts left." / "That OTP is not correct. 3 attempts left." |
| RSP-AUTH-004 | Same mobile verified on two accounts: fixed by "One mobile number per account" above [`b7f3429`]. Verified on the sandbox 2026-11-02: `9876501234`, `+919876501234` and `09876501234` from a new account all get `409 mobile_taken`, no OTP. Users 180 and 246 still share that number from before the fix |
| RSP-ONB-001 | `POST /onb/pan/reset` clears the PAN, name, DOB, KRA check results and any fresh-KYC form; the app refills the typed values |
| RSP-ONB-002 | Date-of-birth errors name the API format: "Enter a valid date of birth (YYYY-MM-DD)." (nominee too) |
| RSP-ONB-003 | The full PAN never leaves the server: users carry `pan_masked` only, the nominee's PAN is masked, PAN-check `checks.pan.value` is masked, and the MF Central status returns `pan: null` when the KYC PAN is on file |
| RSP-ONB-004 | A pincode is accepted only when FP's state fits the pincode's postal region (`utils/pincode.ts`); FP's junk rows (999999 as Nizamabad, 123456 as Delhi) answer "Pincode not found." |
| RSP-ONB-005 | Own message for FP's `verification_attempt_limit_exceeded` (FP blocks an account number for everyone after 3 failed checks): "This account has had too many verification attempts and can’t be verified right now. Please use another account in your name." |
| RSP-ONB-006 | Fixed by "Fresh KYC for PANs with no KRA record" below [`29bae4a`]. Through the API, send `"fresh_kyc": true` on `POST /onb/pan` (the app does this itself); testable on the sandbox since 2026-11-02 |
| RSP-FND-001 | `GET /funds/compare` refuses a third fund: `400 "You can compare up to 2 funds."` |

### Fresh KYC for PANs with no KRA record [`29bae4a`]

- `POST /onb/pan` takes `fresh_kyc: true`. A PAN whose readiness is `kyc_unavailable` then continues with a fresh KYC
  instead of stopping (`422` without the flag, as before). Other readiness failures still stop at the PAN step.
- New `kyc.service.ts` on Cybrilla POA's KYC Forms API: create and fill the form, DigiLocker Aadhaar link, signature
  upload, e-sign link, KRA status check.
- New routes: `POST /onb/kyc/start` (with the phone's latitude / longitude, which must be inside India),
  `POST /onb/kyc/refresh`, `POST /onb/kyc/esign`, `GET /onb/kyc/status`, and the `/kyc/return` page.
- Step order for fresh KYC: `aadhaar` (DigiLocker) -> `bank` -> `signature-consent` -> `kyc-esign` -> `nominee` ->
  `esign`. No selfie (the DigiLocker OTP is the liveness check).
- The user gets `kyc_fresh` and `kyc_status` (`compliant`, `in_progress`, `with_kra`, `failed`).
- Orders, plans, mandates and cart writes answer `403` with `kyc_status` until the KRA approves. The new
  `kyc-approvals` job re-checks every 30 minutes; KYC notifications ("with the KRA", "approved").
- Open review points noted at the time (not fixed in these commits): a KRA rejection is not surfaced yet (the investor
  stays `with_kra`); each waiting investor costs a pre-verification call about every 30 minutes with no end date; the
  Admin API does not expose `kyc_status`.
