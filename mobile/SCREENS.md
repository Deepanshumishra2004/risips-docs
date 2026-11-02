# Screens

Every screen under `mobile/src/app/`, grouped by area. Routes come from the file path (Expo Router): `(tabs)` is a
route group and does not appear in the URL, `[isin]` is a route parameter, `index.tsx` is the folder's own route.

How to read the tables:

- **Calls**: backend endpoints the screen calls itself (`api.*` or `useApi`). `useApi` GETs reload whenever the screen
  regains focus. Shared components add calls of their own; these are listed once under "Shared components".
- **Next**: where the screen navigates. "replace" means the screen is swapped out, so Back does not return to it.
- "OTP" means the action goes through the transaction OTP sheet (`useTxnOtp`): `POST /txn/otp {action}` first, then the
  order call carries `otp`.

For the onboarding screens in depth (both KYC paths), see [ONBOARDING.md](ONBOARDING.md).

## Shared components that call the backend

| Component | Call | Used by |
|---|---|---|
| `components/TxnOtp.tsx` (`useTxnOtp`) | `POST /txn/otp {action}` | Every order, plan change, nominee, account deletion |
| `components/CartIcon.tsx` | `GET /cart` (item count badge) | Home header, invest screens |
| `components/cart.tsx` `addToCart` | `POST /cart {isin, mode, amount}` | Lumpsum and SIP screens ("add to cart") |
| `components/HomeModals.tsx` | `POST /notifications/seen {id}` | Home (one-time holiday / promo modal) |
| `components/reports.tsx` | `GET /reports/<key>/html` | Report viewer and PDF download |
| `components/SelectStep.tsx` | `POST /onb/profile {<field>}` | Occupation, income, marital status steps |
| `lib/session.tsx` | `GET /me`, `POST /auth/logout` | App start, sign-out |
| `lib/env.ts` | `GET /health` (raw fetch) | Sandbox detection |

## Start, sign-in and unlock

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/` | `index.tsx` | Brand splash (about 1.2 s) | `GET /me` through the session | No user: `/welcome`. PIN set: `/unlock`. Otherwise `routeForStep(next_step)`. All replace. If a token is saved but the server can't be reached: "Can't reach Risips right now" + Try again |
| `/welcome` | `welcome.tsx` | Logo, "Continue using Google / Apple / Email", legal text | none | Email: `/auth/email`. Google and Apple only show "coming soon" with a "Use email" button |
| `/auth/email` | `auth/email.tsx` | Email field | `POST /auth/email/start {email}` | `/auth/email-otp` with `email`, `dev` (sandbox OTP), `len` (code length, default 4) |
| `/auth/email-otp` | `auth/email-otp.tsx` | OTP boxes (`OtpScreen`), resend, a consent checkbox (not required here) | `POST /auth/email/start` (resend), `POST /auth/email/verify {email, code}` | Signs in, then replace to `routeForStep(next_step)`. If the account had a pending deletion, shows "Your account deletion request has been cancelled" |
| `/unlock` | `unlock.tsx` | "Welcome back", PIN field, "Use biometric unlock", "Not you? Sign out" | `POST /auth/pin/verify {pin}` | Replace to `routeForStep(next_step)`. Biometric prompt opens by itself when enabled. A 401 signs out after 1.5 s |

Rules:

- The email must look like `x@y.zz`. Codes are 6 digits on current backends (`code_length`); the app falls back to 4.
- OTP code lifetime shown on screen: 2 minutes for email/mobile, 5 minutes for Aadhaar (the backend decides the real
  expiry).

## Onboarding (`/onboarding/*`)

The backend's `user.next_step` decides which of these opens. Full flow in [ONBOARDING.md](ONBOARDING.md).

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/onboarding/mobile` | `mobile.tsx` | Mobile number (+91, must start 6-9, 10 digits), "logged in as <email>. Edit" (Edit signs out) | `POST /auth/mobile/start {mobile}` | `/onboarding/mobile-otp` |
| `/onboarding/mobile-otp` | `mobile-otp.tsx` | OTP + required checkbox "I authorise risips to validate my bank details" | `POST /auth/mobile/start` (resend), `POST /auth/mobile/verify {mobile, code, bank_consent}` | `goNext` (about 350 ms later). On error, "Wrong number? Change" goes back |
| `/onboarding/pin` | `pin.tsx` | Set and re-enter a 4-digit PIN; "Retry! PIN did not match." | `POST /auth/pin/set {pin}` | Replace to `/onboarding/pin-done` |
| `/onboarding/pin-done` | `pin-done.tsx` | "Pin Set Successfully!", biometric toggle (needs a sensor and an enrolled fingerprint/face) | `POST /auth/biometric {enabled}` | `goNext` |
| `/onboarding/pan` | `pan.tsx` | PAN, full name as on PAN, date of birth; sandbox hint | `POST /onb/pan {pan, name, dob, fresh_kyc}` | Push `/onboarding/pan-confirm` with `checks` and the typed `pan` |
| `/onboarding/pan-confirm` | `pan-confirm.tsx` | PAN card preview (typed PAN, name, DOB), the four check results, fresh-KYC note | `POST /onb/pan/confirm`, `POST /onb/pan/reset` | "Yes, confirm": `goNext`. "No, edit PAN": reset, then replace to `/onboarding/pan` with the typed values |
| `/onboarding/occupation` | `occupation.tsx` | Dropdown (`SelectStep`) | `POST /onb/profile {occupation}` | `goNext` |
| `/onboarding/income` | `income.tsx` | Dropdown | `POST /onb/profile {income_slab}` | `goNext` |
| `/onboarding/marital` | `marital.tsx` | Dropdown (Single / Married) | `POST /onb/profile {marital_status}` | `goNext` |
| `/onboarding/father` | `father.tsx` | Father's or spouse's name (3+ characters) | `POST /onb/profile {father_name}` | `goNext` |
| `/onboarding/address` | `address.tsx` | Address lines, pincode (looked up live), gender | `GET /onb/pincode/:pin`, `POST /onb/address {line1, line2, pincode, gender}` | `goNext` |
| `/onboarding/aadhaar` | `aadhaar.tsx` | Fresh KYC: "Verify your Aadhaar" + Continue with DigiLocker. Otherwise: the old Aadhaar-number form | Fresh: `POST /onb/kyc/start {latitude, longitude}`. Old: `POST /onb/aadhaar/start {aadhaar}` | Fresh: push `/onboarding/kyc-web` (step `proof`), or `goNext` if already fetched. Old: `/onboarding/aadhaar-otp` |
| `/onboarding/aadhaar-otp` | `aadhaar-otp.tsx` | 6-digit Aadhaar OTP (old path only) | `POST /onb/aadhaar/start` (resend), `POST /onb/aadhaar/verify {code}` | Replace to `/onboarding/aadhaar-confirm` |
| `/onboarding/aadhaar-confirm` | `aadhaar-confirm.tsx` | Masked Aadhaar card (old path only) | none | "Proceed for KYC": `goNext`. "No, edit Aadhaar": `/onboarding/aadhaar` |
| `/onboarding/kyc-web` | `kyc-web.tsx` | WebView for DigiLocker (`step=proof`) or KYC e-sign (`step=esign`), sandbox hint bar | `POST /onb/kyc/refresh` when the return URL loads | `goNext` if the step moved on, else back to the calling screen with an `error` |
| `/onboarding/bank` | `bank.tsx` | Account number twice, IFSC (looked up live), sandbox hint | `GET /onb/ifsc/:ifsc`, `POST /onb/bank {account_number, ifsc}` | `goNext` |
| `/onboarding/selfie-intro` | `selfie-intro.tsx` | Selfie instructions (red / green ring) | none | Push `/onboarding/selfie` |
| `/onboarding/selfie` | `selfie.tsx` | Front camera in a ring, Capture / Retake / Confirm | `POST /onb/selfie {image}` (base64 JPEG) | `goNext` |
| `/onboarding/signature-consent` | `signature-consent.tsx` | "Declarations" (KYC-compliant) or "Submit your Signature" (others): running-account (optional) and FATCA (required) checkboxes | `POST /onb/declarations {fatca, running_account}` | Compliant: `goNext`. Others: push `/onboarding/signature` |
| `/onboarding/signature` | `signature.tsx` | Signature pad, Reset / Proceed | `POST /onb/signature {image}` (PNG data URL) | `goNext` |
| `/onboarding/kyc-esign` | `kyc-esign.tsx` | "Sign your KYC form" | `POST /onb/kyc/esign` | Push `/onboarding/kyc-web` (step `esign`), or `goNext` if already signed |
| `/onboarding/nominee` | `nominee.tsx` | Nominee name, DOB (18+), relationship, PAN; Skip opens an opt-out declaration | OTP (`nominee`), then `POST /onb/nominee` with the nominee or `{skip: true}` | `goNext` |
| `/onboarding/esign` | `esign.tsx` | Account opening form summary; compliant: "Verify your details" + "Confirm details"; fresh: "CKYC & KRA KYC Form" + "Sign Document" | `POST /onb/esign {consent: true}` | Replace to `/onboarding/done` |
| `/onboarding/done` | `done.tsx` | "ALL SET <first name>!" and a status line | none | "Get Started": replace to `/home` |

## Tabs (`(tabs)/`)

A custom floating pill tab bar (`(tabs)/_layout.tsx`) with three tabs. The routes are `/home`, `/explore` and
`/holdings`.

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/home` | `(tabs)/home.tsx` | Wealth, invested, 1D return, total return, XIRR; One Time / Start SIP or Start Investing; KYC-with-KRA banner; promo banner; picks; Tools (Compare, Calculate, Import, Cart, All Orders, AutoPay & Bank, API Log on sandbox); header icons (assistant, cart, bell with unread count, profile); one-time holiday/promo modal | `GET /portfolio`, `GET /funds?picks=1`, `GET /sips`, `GET /notifications`; `GET /onb/kyc/status` on focus while `kyc_status === 'with_kra'` | `/explore`, `/collection/foryou?mode=sip`, `/holdings`, `/assistant`, `/cart`, `/notifications`, `/profile`, `/profile/account` (KYC banner), `/compare`, `/calculator`, `/external/import`, `/orders`, `/profile/autopay`, `/apilog`, banner route |
| `/explore` | `(tabs)/explore.tsx` | Explore / Watchlist tabs; Our Picks; Collections; All Mutual Funds with search, 1Y/3Y/5Y sort and filters | `GET /funds?picks=1`, `GET /funds/collections`, `GET /funds?sort=&q=&<filters>` (search debounced 200 ms), `GET /watchlist` (Watchlist tab), `POST /watchlist/:isin` (toggle) | `/collection/[key]`, `/fund/[isin]`, `/filters?scope=explore|watch` |
| `/holdings` | `(tabs)/holdings.tsx` | Portfolio / SIPs tabs. Summary card (opens to "Hide external investments"), holdings incl. imported funds, orders in progress; SIP, STP and SWP lists with this month's SIP total | `GET /portfolio`, `GET /sips`, `GET /stps`, `GET /swps`, `POST /external/visibility {hidden}` | `/holding/[isin]` (with folio), `/external/[key]`, `/order/[id]` (a fund still on its way in), `/sip/[id]`, `/stp/[id]`, `/swp/[id]`, `/orders`, `/explore`, `/collection/foryou?mode=sip` |

Rules:

- Home hides "Start a SIP" style offers until `/sips` answers, and shows no ₹0 flash before `/portfolio` loads.
- Imported (MF Central) funds count towards Wealth unless hidden on the Holdings tab.
- `/holdings?tab=SIPs` opens the SIPs tab (the assistant links there).

## Funds

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/fund/[isin]` | `fund/[isin].tsx` | Fund page: return and chart with period picker, One-time / Start SIP, facts, expense ratio, holdings analysis, managers, collapsible sections; watchlist heart | `GET /funds/:isin?period=`, `POST /watchlist/:isin` | `/invest/lumpsum`, `/invest/sip`. If the user is not onboarded, the invest buttons open `/onboarding/<next_step>` instead |
| `/collection/[key]` | `collection/[key].tsx` | Collection title, search, its Our Picks (top 6), every fund in it with filters and 1Y/3Y/5Y | `GET /funds?collection=<key>&picks=1`, and the full collection list | `/fund/[isin]`, or `/invest/sip?isin=` when opened with `mode=sip` (from Home "Start SIP") |
| `/filters` | `filters.tsx` | Filter groups (category, risk ...) with counts, Cancel / Apply. `?scope=` is `explore`, `watch`, `collection:<key>` or `compare` | GETs the list being filtered to count the options | Replace to `/explore` or back |
| `/compare` | `compare.tsx` | Two "Add fund" slots, popular pairs, fund picker, side-by-side accordion. `/compare?isins=a,b` opens pre-filled | `GET /funds?sort=&q=` (picker), `GET /funds/compare?isins=`, `GET /funds/compare/popular` | Fund picker in place; back closes the picker first |
| `/calculator` | `calculator.tsx` | SIP / Lumpsum return calculator with sliders (computed on the phone) | none | `/explore`, `/collection/foryou?mode=sip` |

## Invest (lumpsum, SIP, payment)

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/invest/lumpsum` | `invest/lumpsum.tsx` | Amount entry, confirm sheet, add-to-cart | `GET /funds/:isin?period=1M`; OTP (`lumpsum`), then `POST /invest/lumpsum {isin, amount, otp}` | Replace to `/invest/pay` with the checkout `token_url`. A fund with no lumpsum (`lumpsum.allowed === false`) shows only "Start a SIP instead" (`/invest/sip`) |
| `/invest/sip` | `invest/sip.tsx` | Amount, SIP date (1-28; FP does not support 29-31), AutoPay type, confirm sheet, add-to-cart | `GET /funds/:isin?period=1M`, `GET /mandate`, `POST /mandate {type, need, limit}`, OTP (`sip`), then `POST /sip {isin, amount, day, frequency:'monthly', first_now:true, first_pay:'upi', otp}` | No approved AutoPay: `/invest/pay?kind=mandate`, which returns here with `mandate_done=1`. Then `/invest/pay` for the first instalment's UPI checkout, then `/invest/sip-result` |
| `/invest/pay` | `invest/pay.tsx` | WebView for FP pages: the ONDC UPI checkout (`kind=payment`) or the AutoPay approval page (`kind=mandate`); sandbox hint bar; close asks "Leave payment?" | none (it watches for the return URL) | Payment: `next` route if given, else `/invest/result?id=<order>`. Mandate: `next` (default `/profile/autopay`) with `mandate_done=1` |
| `/invest/result` | `invest/result.tsx` | Lumpsum result: green / amber / red by state, polls until final; cut-off time note; retry and contract note | `GET /orders/:id`; retry: OTP, `POST /invest/lumpsum` | `/invest/pay` (retry), `/invest/lumpsum` (change amount), `/reports/contract-note`, `/holdings`, `/home` |
| `/invest/sip-result` | `invest/sip-result.tsx` | SIP started (amber while the first UPI payment confirms; polls about 3 min), AutoPay failed, or first payment failed | `GET /sips/:id`, `GET /funds/:isin`, `GET /mandate`, `POST /mandate` (retry AutoPay) | `/sip/[id]`, `/invest/sip` (retry), `/home` |

Rules:

- A checkout link is single-use. The app always `replace`s into `/invest/pay`, so Back can never reopen a used link.
- UPI app buttons on the checkout (`upi://`, `tez://`, `phonepe://`, `intent://...`) are handed to the phone's UPI app.
- UPI AutoPay is capped at ₹1 lakh; a bigger SIP needs a different AutoPay type.
- If the first SIP payment fails, the backend cancels the plan, and the app shows the SIP as failed.

## Cart and checkout

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/cart` | `cart.tsx` | Cart lines (SIP or one-time), ⋮ menu (edit, move to watchlist, view fund, remove), empty state | `GET /cart`, `GET /watchlist`, `PATCH /cart/:id`, `DELETE /cart/:id`, `POST /watchlist/:isin` | `/checkout`, `/fund/[isin]`, `/explore` |
| `/checkout` | `checkout/index.tsx` | Confirm sheet: SIP date, pay using (UPI; Netbanking shown disabled), amount breakdown, Pay | `GET /cart`, `GET /mandate`, `POST /mandate`, OTP (`cart`, one code for the whole checkout), `POST /checkouts` | Needs AutoPay for SIPs (409 `need_mandate`): `/invest/pay?kind=mandate`, back here with `mandate_done=1`. Then replace to `/checkout/[id]` |
| `/checkout/[id]` | `checkout/[id].tsx` | Paying loop and result. One-time funds pay on one UPI checkout, SIPs on another (at most two payments). Polls every 5 s while a paid line confirms | `GET /checkouts/:id`, `POST /checkouts/:id/next`, `POST /checkouts/:id/stop` | `/invest/pay` (with a title like "Paying 1 of 2"), `/invest/result`, `/sip/[id]`, `/cart`, `/holdings`, `/home` |

Rules: the next payment only opens on a tap (a link burns as soon as it loads). Android Back mid-checkout asks
"Stop here?"; unpaid lines stay in the cart.

## Holdings, orders and selling

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/holding/[isin]` | `holding/[isin].tsx` | One holding (fund + folio): value, returns, SIP / STP / SWP cards, its orders; Manage menu; Redeem / Buy More | `GET /portfolio`, `GET /orders` | `/fund/[isin]`, `/order/[id]`, `/sip/[id]` or `/invest/sip`, `/stp/[id]` or `/stp/new`, `/swp/[id]` or `/swp/new`, `/switch`, `/redeem`, `/invest/lumpsum` |
| `/orders` | `orders.tsx` | All lumpsum, redemption and switch orders placed from the app (SIP instalments live on each SIP) | `GET /orders` | `/order/[id]` |
| `/order/[id]` | `order/[id].tsx` | Switch or redemption result; polls until settled (a switch also waits for its in-leg units); cut-off note | `GET /orders/:id` | `/orders`, `/reports/contract-note`, `/holdings`, `/home`, retry to `/switch` or `/redeem` |
| `/redeem` | `redeem.tsx` | Redeem by amount or "All units", folio, exit load, confirm sheet | `GET /portfolio`, `GET /funds/:isin?period=1M`, OTP (`redeem`), `POST /invest/redeem` (`{isin, folio, amount}` or `{isin, folio, all: true}`) | Replace to `/order/[id]` |
| `/switch` | `switch.tsx` | From fund, same-fund-house target picker, amount or all, confirm sheet | `GET /portfolio`, `GET /funds?amc_id=&switch_in=1`, OTP (`switch`), `POST /invest/switch {from_isin, to_isin, folio, amount | all}` | Replace to `/order/[id]` |
| `/external/[key]` | `external/[key].tsx` | A fund held outside Risips (read-only), distributor if known | `GET /portfolio` | `/fund/[isin]`, `/invest/lumpsum` (buy it through Risips), `/holdings` |

Rules: when every unit is already leaving (a full redemption or switch still being processed), the holding blocks new
sell-side orders with "Order in progress". ONDC redeems by amount only (no units).

## SIP management (`/sip/*`)

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/sip/[id]` | `sip/[id].tsx` | SIP details: state, next debit, AutoPay used, instalments; Modify / Pause / Cancel (active) or Resume / Cancel (paused) when the backend allows | `GET /sips/:id`, `GET /mandate`, OTP (`sip_pause` / `sip_resume`), `POST /sips/:id/pause {months}`, `POST /sips/:id/resume` | `/sip/modify`, `/sip/cancel`, `/sip/result?kind=paused|resumed`, `/invest/sip`, the replacing plan |
| `/sip/modify` | `sip/modify.tsx` | New amount and date | `GET /sips/:id`, `GET /funds/:isin`, `GET /mandate`, `POST /mandate`, OTP (`sip_modify`), `POST /sips/:id/modify` | `/invest/pay?kind=mandate` if the amount is above the AutoPay limit, then `/sip/result?kind=modified` |
| `/sip/cancel` | `sip/cancel.tsx` | Reason list; "change the amount" offers Modify instead | `GET /sips/:id`, OTP (`sip_cancel`), `POST /sips/:id/cancel {code, otp}` | `/sip/result?kind=cancelled`, `/sip/modify`, back to `/sip/[id]` (pause instead) |
| `/sip/result` | `sip/result.tsx` | Paused / resumed / cancelled / modified result | `GET /sips/:id`, `GET /mandate` (resumed) | `/sip/[id]`, `/holdings?tab=SIPs`, `/invest/sip` |

Rules: a pause is at most 2 months (`MAX_PAUSE_MONTHS`) and starts with the next instalment at least 2 days away.
FP can't edit an ONDC plan, so Modify registers a new plan and cancels the old one. Cancelling keeps the units.

## STP and SWP

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/stp/new` | `stp/new.tsx` | From fund, same-fund-house target picker (funds that accept STP), monthly amount, day, instalments, confirm sheet | `GET /portfolio`, `GET /funds?amc_id=&stp_in=1`, OTP (`stp`), `POST /stp {from_isin, to_isin, folio, amount, day, installments, otp}` | Replace to `/stp/result` |
| `/stp/result` | `stp/result.tsx` | STP started (waits until the plan is active) | `GET /stps/:id` | `/stp/[id]`, `/home` |
| `/stp/[id]` | `stp/[id].tsx` | STP details and Cancel (no modify on ONDC) | `GET /stps/:id`, OTP (`stp_cancel`), `POST /stps/:id/cancel` | stays |
| `/swp/new` | `swp/new.tsx` | Monthly withdrawal amount, day (1-28), start month, until options; Quarterly shown disabled | `GET /portfolio` | Push `/swp/review` |
| `/swp/review` | `swp/review.tsx` | Review, "Confirm with OTP" | `GET /portfolio`, OTP (`swp`), `POST /swp` | Replace to `/swp/result` |
| `/swp/result` | `swp/result.tsx` | SWP set up (waits until active) | `GET /swps/:id` | `/holding/[isin]`, `/home` |
| `/swp/[id]` | `swp/[id].tsx` | SWP details and Cancel | `GET /swps/:id`, OTP (`swp_cancel`), `POST /swps/:id/cancel` | stays |

Rules: ONDC supports monthly plans only. A later SWP start month becomes a skip instruction, which FP needs a few days
ahead.

## Import external funds (MF Central)

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/external/import` | `external/import.tsx` | Mobile or email (as registered in the folios), PAN, consent; Remove imported funds | `GET /external`, `POST /external/start {via, consent, mobile|email, pan}`, `DELETE /external` | Push `/external/journey`; replace to `/external/importing` if an import is running |
| `/external/journey` | `external/journey.tsx` | MF Central's hosted pages in a WebView (OTP, choose, QR) | `POST /external/qr` (the QR the page posts), `POST /external/cancel` (on leave) | Replace to `/external/importing` |
| `/external/importing` | `external/importing.tsx` | "Importing all external funds", turns into "N funds imported" | `GET /external` (polls) | Done; `/external/import` |

On a sandbox backend the journey is a local stand-in page (`/mfc-sandbox/...`) that shows its own OTP; the statement
arrives about 20 s after the QR step and holds sample holdings.

## Profile

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/profile` | `profile/index.tsx` | Menu: Account Details, Bank & AutoPay, Nominee, Orders, Reports, SIPs & Holdings, Return Calculator, Investor Charters, fingerprint toggle, Settings, FP API log (sandbox), App version / Check for updates, Sign out | `POST /auth/biometric {enabled}` | The listed screens |
| `/profile/account` | `profile/account.tsx` | Name, DOB, masked PAN, gender, selfie, masked mobile and email; KYC details (Aadhaar or KYC status, address, occupation, income, marital status, father's name, nominee); FP investor profile and account ids | none (reads the session user) | none |
| `/profile/autopay` | `profile/autopay.tsx` | Bank account, active AutoPay details, set up AutoPay | `GET /mandate`, `POST /mandate {type, limit: 100000}` | `/invest/pay?kind=mandate`, back here with `mandate_done=1` |
| `/profile/nominee` | `profile/nominee.tsx` | Current nominee, or a form to add one (same fields as onboarding) | OTP (`nominee`), `POST /profile/nominee` | stays |
| `/charters` | `charters.tsx` | Investor charters and grievance reports; only the SEBI charter has a link, the rest say "will be published before launch" | none | external links |

Rules: a nominee added after onboarding goes only on folios opened from then on; the screen says so.

## Settings

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/settings` | `settings/index.tsx` | Appearance (System / Light / Dark), Manage notifications, Change PIN, Delete account, Active devices, Logout | none (Logout: `POST /auth/logout` via the session) | the listed screens |
| `/settings/notifications` | `settings/notifications.tsx` | Toggles: SIP reminders, portfolio summary, goals and review calls, market insights; push, WhatsApp, email. Each saves at once | `GET /settings/notifications`, `PUT /settings/notifications` | none |
| `/settings/pin` | `settings/pin.tsx` | Current PIN, new PIN, confirm; "Forgot your PIN?" explains sign out and sign in by email | `POST /settings/pin/check {current}`, `POST /settings/pin/change {current, pin}` | Success sheet. An error that ends the session signs out |
| `/settings/devices` | `settings/devices.tsx` | This device and other signed-in devices (name, OS); log one or all others out | `GET /settings/devices`, `POST /settings/devices/:id/logout`, `POST /settings/devices/logout-others` | none |
| `/settings/delete` | `settings/delete.tsx` | Checks (no active plans, no pending orders), reason, Delete account | `GET /settings/deletion`, OTP (`delete_account`), `POST /settings/deletion {otp, reason}` | Replace to `/settings/deleted` |
| `/settings/deleted` | `settings/deleted.tsx` | "Delete account requested" (already signed out) | none | Replace to `/` |

Rules: the Delete button stays disabled until the backend's checks pass. Signing in again within the grace period
(30 days per the email sign-in code) cancels the request.

## Reports

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/reports` | `reports/index.tsx` | Groups: Profit & Loss (dividends), Tax (ELSS, capital gains, GST invoice), Holdings, Transactions (order history, balance statement, contract notes) | none | `/reports/[key]` |
| `/reports/[key]` | `reports/[key].tsx` | One report (FY picker where relevant), View report, Download (PDF) | `GET /reports/<key>?...`, `GET /reports/<key>/html` (download) | `/reports/view`, linked reports (for example one contract note) |
| `/reports/view` | `reports/view.tsx` | The report's printable HTML in a WebView, Download | `GET /reports/<key>/html` | none |

Several of these depend on data FP does not provide (for example dividends, ELSS, GST); the backend shapes what is
shown, and some may be empty.

## Other screens

| Route | File | What it shows | Calls | Next |
|---|---|---|---|---|
| `/notifications` | `notifications.tsx` | Notification list with tags (ORDER, SIP, KYC ...), "Today / 1d ago" dates | `GET /notifications`, `POST /notifications/read` | Each item's `route` |
| `/assistant` | `assistant.tsx` | Chat-style help: welcome, suggestion chips, answers with actions and an investment-advice guardrail. `/assistant?q=<topic>` asks at once | `GET /assistant`, `POST /assistant {text}` or `{action}` | Routes in the replies; onboarding next step; `/home` |
| `/apilog` | `apilog.tsx` | Sandbox only: every FP call the backend made, newest first, live or paused, tap to expand | `GET /debug/trace?since=` (polled) | none |

`/apilog` is only linked from `SandboxOnly` UI, and the backend answers 404 on it unless it runs in demo mode.
