# Testing the app

A guide for QA testers and developers checking the Risips app. It covers which backend to test against, where codes
appear, the sandbox test data, and how to get through payments, AutoPay and DigiLocker in the sandbox.

**Never test with real money by accident.** If the small amber **SANDBOX** tag is not at the top of the screen, the
app is talking to a production backend and orders move real money. Stop and check.

## Which app and which backend

| Option | How | Backend | Good for |
|---|---|---|---|
| **Risips Sandbox** app (amber icon, `in.risips.app.sandbox`) | Install the sandbox APK you were given | Hosted sandbox, `https://risips-demo-api.azurewebsites.net` (fake money) | Day-to-day QA |
| Expo Go on an emulator or phone, through Metro | See README "Running locally" | Whatever `EXPO_PUBLIC_API_URL` Metro was started with, or a laptop backend | Developers; testing unreleased changes; testing against a local `backend-ts` |

Notes:

- The app picks its backend at build/bundle time (see README "Choosing the backend"). If it can't reach it, every
  screen says "Can't reach the Risips server at <url>", which shows the address in use.
- The SANDBOX tag, the yellow hints and the API Log follow the **backend** (`GET /health` `fp_env`), not the app: a
  backend on the FP sandbox shows them, a production backend hides them.
- To get the newest screens on an installed app after an over-the-air update: close the app fully, open it (it
  downloads), close it again, open it (it applies). Profile > "App version · Check for updates" shows the running
  update and can fetch one immediately.
- The hosted sandbox (`https://risips-demo-api.azurewebsites.net`) runs the TypeScript backend (`backend-ts`) since
  2026-11-02, so every flow in these docs, fresh KYC and Profile > Nominee included, can be tested there (fresh KYC
  verified there on 2026-11-02).

### Running against a local backend

1. Start the backend on the laptop (`cd backend-ts && npm run dev`). It runs on the FP sandbox with demo mode on unless
   configured for production.
2. Start Metro pointed at it, for example on the Android emulator:
   `EXPO_PUBLIC_API_URL=http://10.0.2.2:8080 REACT_NATIVE_PACKAGER_HOSTNAME=127.0.0.1 npx expo start --lan --port 8081`
   (`10.0.2.2` is the emulator's address for the laptop. Use the port the backend prints; without the variable the app
   uses `http://10.0.2.2:4000`.)
3. Open Expo Go: `adb shell am start -a android.intent.action.VIEW -d exp://127.0.0.1:8081 host.exp.exponent`

Biometric unlock on the emulator: add a fingerprint in the emulator's Settings > Security > Fingerprint, then use the
emulator's Extended controls > Fingerprint to touch the sensor.

## Where OTPs appear

On a sandbox backend in demo mode, **no SMS or email is needed**: the code is shown in a yellow "Sandbox" box on the
screen.

| Code | Where you type it | Where the sandbox shows it |
|---|---|---|
| Email sign-in | `/auth/email-otp` | Yellow box: "Sandbox · your code is NNNNNN" |
| Mobile number | `/onboarding/mobile-otp` | Same yellow box |
| Transaction OTP (every order, SIP / STP / SWP start, change or cancel, nominee, account deletion) | The OTP sheet that slides up | Yellow box in the sheet: "Sandbox · OTP NNNNNN" |
| Old Aadhaar OTP (legacy accounts only) | `/onboarding/aadhaar-otp` | Yellow box |
| DigiLocker and KYC e-sign | Inside Cybrilla's pages | Not shown: the simulator accepts any 6 digits (see below) |
| MF Central import (sandbox stand-in) | Inside the journey page | Shown on the stand-in page itself |

If no yellow box appears: the backend is not in demo mode (or it is production), so codes go by SMS/email or only to
the server log. Codes are 6 digits; an app showing 4 boxes is running old screens (update it as above).

Limits worth knowing: one new code per 30 s, and a limited number of codes and wrong tries per hour per purpose. A 401
on the unlock PIN signs you out.

## Sandbox test data

These patterns come from FintechPrimitives' sandbox and only work on a sandbox backend. Letters marked `X` can be any
letter, `N` any digit.

### PAN (pre-verification and KRA status)

A PAN must be an individual PAN: 5 letters, 4 digits, 1 letter, with **`P` as the 4th letter** (the app rejects
anything else before calling the backend). Use a real-looking name and a DOB that makes the investor 18+.

| PAN pattern | Example | Result in the app |
|---|---|---|
| `XXXPX3751X` | `ABCPX3751K` | **KYC-compliant**: path A (no Aadhaar, selfie or signature) |
| `XXXPX3753X` | `ABCPX3753K` | **No KRA record** (`kyc_unavailable`): path B, fresh KYC, on an app build that has `expo-location`. Older builds get "No KYC record found for this PAN with the KRA. A fresh KYC is needed." |
| `XXXPX3752X`, `3754` to `3759` | `ABCPX3756K` | Other KRA states (constrained, on hold, rejected, deactivated, legacy, under process, incomplete): refused at the PAN step with the matching message |
| Any other digits | `ABCPX1234K` | Not compliant (`unknown`): refused at the PAN step |
| 5th letter `A` (`XXXPANNNNX`) | `ABCPA3751K` | "Your PAN is not linked with Aadhaar..." |
| 5th letter `I` (`XXXPINNNNX`) | `ABCPI3751K` | "This PAN is invalid..." |
| Name `Lord Voldemort` | | "The name does not match PAN records..." |
| DOB `01/01/2000` | | "The date of birth does not match PAN records." |

So keep the 5th letter something other than `A` or `I` (for example `X`) unless you are testing those errors.

To redo the PAN step, tap **"No, edit PAN"** on the confirm screen: the fields come back filled with what you typed.
Once you tap "Yes, confirm", the PAN can't be changed.

### Bank account

The IFSC must be real (it is looked up), for example `HDFC0000001`. The **last 4 digits** of the account number decide
the result:

| Last 4 digits | Result |
|---|---|
| `11XX` | Verified (very high confidence). The in-app hint suggests `1191` to `1199`, for example `50100123451195` |
| `12XX` | High confidence at FP, but on 2026-11-03 it stayed pending ("taking longer than usual"). Use `11XX` |
| `13XX` / `14XX` / `15XX` | Uncertain / low / very low confidence: may be refused ("We could not confirm this account belongs to you") |
| `1600` | Zero confidence |
| `31XX` | Fails: "Bank verification failed. Please check the account number and IFSC." |

**Use a unique account number each time** (change the digits before the last 4). FP blocks an account number for
everyone after 3 failed checks on it, and the app then says "This account has had too many verification attempts and
can't be verified right now. Please use another account in your name." A number another tester already used may be
blocked.

### Funds and amounts

- The sandbox has only ICICI Prudential and Aditya Birla Sun Life schemes, and only Regular plans.
- Each fund has its own minimums; the app shows them and the backend enforces them.
- On the FP sandbox, the order amount can decide the outcome at the fund house: an amount ending in 0 succeeds, one
  ending in 1 fails (useful to test the failed-order screens).

## Getting through the hosted pages in the sandbox

The app shows a yellow hint bar at the top of each of these pages when the backend is a sandbox.

### UPI payment (lumpsum, cart, SIP first instalment)

1. On the checkout page, **type any well-formed UPI ID** (for example `demo@okaxis`) and tap **Pay**.
2. Wait about 10 seconds. A **demo bank page** appears.
3. Tap **Success** (or **Failure** to test a failed payment). The app returns to the result screen by itself.

Do not scan the QR code. Each checkout link works once: if you leave it ("Leave payment?"), that order can't be paid
again; place a new one. On the sandbox an order is usually allotted within a minute; on production it takes until the
next working day.

### AutoPay (mandate) for SIPs

- BillDesk simulator: choose Status **"Success"** and tap **Submit**.
- UPI AutoPay: tap **Show QR** (don't scan it), wait for the demo bank page, then tap **Success**.

The app comes back to the SIP, cart or Bank & AutoPay screen and continues.

### DigiLocker (fresh KYC, path B)

On the "Aadhaar via DigiLocker" page the sandbox simulator accepts anything:

1. Enter **any mobile number**, **any 6-digit PIN** and **any 6-digit OTP**.
2. For the purpose, choose **Know Your Customer**.
3. Tap **Allow**.

The app then checks the form and moves to the bank step. If it returns to the Aadhaar screen with "Aadhaar was not
fetched from DigiLocker", try again.

The app asks for **location permission** before DigiLocker. On the emulator, set a location in Extended controls >
Location if it can't get one.

### KYC form e-sign (fresh KYC)

On the "E-sign your KYC" page: tick the confirmation, tap **esign**, and enter **any 6-digit OTP**.

## What to check, by area

| Area | Check |
|---|---|
| Sign-in | Email code; resend after 30 s; wrong code message; sign out and back in |
| Unlock | PIN on cold start; biometric prompt if enabled; too many wrong PINs signs you out |
| Onboarding path A | Compliant PAN goes straight from address to bank, then "Declarations" (no signature), nominee, "Verify your details" |
| Onboarding path B | `3753` PAN: PAN confirm note about DigiLocker; location prompt; DigiLocker; bank; signature; KYC e-sign; nominee; "Verify details and eSign"; done screen says the KYC is with the KRA |
| KYC waiting | Home banner, Profile KYC status, orders refused with the "with the KRA" message |
| PAN privacy | Confirm screen shows the typed PAN; Profile and eSign show it masked; "No, edit PAN" refills the form |
| Bank | IFSC lookup shows bank and branch; mismatched account numbers; `31XX` failure; unique numbers |
| Invest | Lumpsum OTP, UPI checkout, result screen (success and failure); SIP with AutoPay setup and first payment |
| Cart | One-time and SIP lines; checkout pays at most twice (one-time group, SIP group); stopping keeps unpaid lines |
| Sell | Redeem by amount and "All units"; switch within the same fund house; result screens |
| Plans | SIP pause (max 2 months), resume, modify, cancel; STP and SWP create and cancel |
| Import | MF Central sandbox journey, importing screen, hide external investments |
| Settings | Light / dark / system appearance on every screen you test; change PIN; active devices; notification toggles |

When you report a bug, include: the app (Risips Sandbox or Expo Go), the running update shown in Profile > App
version, the backend (SANDBOX tag or not), the screen, and the time. On a sandbox backend, Home > API Log shows every
FintechPrimitives call the backend made, which helps developers.

## Things this guide could not verify

- The exact wording and steps inside FP's checkout, the BillDesk / UPI AutoPay simulators, and Cybrilla's DigiLocker
  and e-sign pages: they are third-party pages. The steps above follow the app's own sandbox hints and the project
  notes.
- Which backend version the hosted sandbox runs today (see "Which app and which backend").
- Bank results for `12XX` to `1600`: the backend accepts only a `verified` result, and the exact status FP returns for
  each confidence band was not checked here.
