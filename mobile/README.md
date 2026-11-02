# Risips mobile app

Developer guide for the Risips app in `risips-dev/mobile`. Written for app developers and QA testers.

Other pages in this folder:

| Page | What it covers |
|---|---|
| [SCREENS.md](SCREENS.md) | Every screen: route, what it shows, the backend calls it makes, where it goes next |
| [ONBOARDING.md](ONBOARDING.md) | The sign-up and KYC flow, step by step, for both KYC paths |
| [TESTING.md](TESTING.md) | How a tester checks the app: backends, OTPs, sandbox test data |

## What the app is

Risips is a goal-based mutual fund investing app (in the style of Groww or Coin), built for Arthasuta Finedu Private
Limited, an AMFI-registered mutual fund distributor.

An investor can:

- sign up with email, verify a mobile number, set a PIN, and complete KYC
- browse, search, filter, compare and watchlist funds
- invest one time (lumpsum) or start a SIP, alone or through a cart
- redeem, switch, and set up STPs (transfer plans) and SWPs (withdrawal plans)
- pause, resume, modify and cancel SIPs
- import funds held elsewhere through MF Central
- view orders, reports and statements, notifications, and settings

**The app talks only to the Risips backend.** It never calls FintechPrimitives (FP), Cybrilla, MF Central or any other
provider directly. Pages hosted by those providers (UPI checkout, AutoPay approval, DigiLocker, KYC e-sign, the
MF Central journey) open inside a WebView from a link the backend hands back. The backend decides what each screen may
do: for example, which onboarding step comes next, whether a SIP can be paused, or whether an order is allowed.

## Tech stack

Versions from `mobile/package.json`:

| Piece | Version |
|---|---|
| Expo SDK | `expo ~57.0.24` |
| React Native | `0.86.3` |
| React | `19.2.3` |
| Expo Router (file-based navigation) | `~57.0.22` |
| TypeScript | `~6.0.3` (dev dependency) |
| `react-native-webview` | `13.16.1` (payments, AutoPay, DigiLocker, KYC e-sign, MF Central, reports) |
| `react-native-svg` | `15.15.4` (icons, charts) |
| `react-native-signature-canvas` | `^5.1.1` (signature pad) |
| `react-native-screens` / `react-native-safe-area-context` | `~4.26.0` / `~5.7.0` |

Expo modules in use: `expo-camera` (KYC selfie), `expo-local-authentication` (biometric unlock), `expo-location`
(fresh KYC location check), `expo-secure-store` (session token, theme choice), `expo-updates` (over-the-air updates),
`expo-print` / `expo-sharing` / `expo-file-system` (report PDFs), `expo-linear-gradient`, `expo-haptics`, `expo-font`,
`expo-splash-screen`, `expo-status-bar`, `expo-system-ui`, `expo-constants`, `expo-linking`.

Fonts: Instrument Sans and Sora (`@expo-google-fonts/*`). Icons: `@expo/vector-icons` plus the app's own SVG icons.

`mobile/AGENTS.md` (also loaded by `mobile/CLAUDE.md`) carries the project rules for code changes: always add packages
with `npx expo install`, never hand-edit `ios/` or `android/` (they are generated), and run `npx expo lint` and
`npx tsc --noEmit` before calling a change done.

## Folder layout

```
mobile/
  app.json          base Expo config (name, bundle ids, permissions, plugins, update URL)
  app.config.js     app variants on top of app.json (development / sandbox / production)
  eas.json          EAS build profiles and channels
  scripts/build-apk.js   local signed APK build (npm run build:apk)
  assets/           icons, splash
  src/
    app/            every file is a screen (Expo Router); _layout.tsx files are navigators
      _layout.tsx   root Stack, fonts, theme, session, SANDBOX badge
      index.tsx     splash: routes to welcome / unlock / onboarding / home
      (tabs)/       bottom tab bar: home, explore, holdings
      auth/         email sign-in
      onboarding/   mobile, PIN, PAN, KYC, bank, nominee, eSign ...
      invest/       lumpsum, SIP, payment WebView, results
      checkout/     cart checkout and its paying loop
      sip/ stp/ swp/   plan details, create, cancel, results
      external/     MF Central import
      profile/ settings/ reports/   account areas
      fund/[isin] holding/[isin] order/[id] collection/[key]   detail pages
    components/     shared UI (ui.tsx, OtpScreen, TxnOtp, SelectStep, cart, funds, reports ...)
    lib/            api client, session, hooks, env (sandbox detection), kyc, formatting, options, types
    theme/          light and dark palettes, fonts, type scale, ThemeProvider
```

Keep non-screen code out of `src/app/`: Expo Router treats every file there as a route.

## Running locally

This follows `risips-dev/HANDOFF.md` ("How to run") and `mobile/AGENTS.md`. Commands are from a Windows laptop with Git
Bash.

1. **Start a backend.** Either use the hosted sandbox backend (no local server needed; see "Choosing the backend"
   below), or run one on the laptop.
   - The TypeScript backend `backend-ts/` is the current one: `cd backend-ts && npm install && npm run dev`. Its
     default port is **8080** (`PORT` in `backend-ts/src/config/env.ts`); the port it really uses comes from its `.env`.
   - `HANDOFF.md` still describes the older JavaScript backend (`cd backend && npm run dev`, port 4000). `backend/` is
     frozen; new work goes in `backend-ts/`.
   - With `DEMO_MODE` on (the default when not on production FP), OTPs are returned to the app and shown in a yellow
     "Sandbox" hint.
2. **Start the Android emulator.** `HANDOFF.md` uses an AVD named `risips` (Pixel 6, Android 14):
   `"%LOCALAPPDATA%\Android\Sdk\emulator\emulator.exe" -avd risips`
3. **Start Metro** from `mobile/`:
   ```bash
   adb reverse tcp:8081 tcp:8081
   REACT_NATIVE_PACKAGER_HOSTNAME=127.0.0.1 npx expo start --lan --port 8081
   ```
   - Do not use `--localhost`: it binds IPv6 only, and the emulator then fails with "Failed to download remote update".
   - Do not set `CI=1`: it turns off file watching.
4. **Open the app in Expo Go** (installed on the AVD):
   `adb shell am start -a android.intent.action.VIEW -d exp://127.0.0.1:8081 host.exp.exponent`

Other scripts in `package.json`: `npm start` (`expo start`), `npm run android`, `npm run ios`, `npm run web`.

Expo Go only contains Expo's bundled native modules. Everything this app uses today runs in Expo Go (including
`expo-location`, so the fresh-KYC path can be reached there). A library with its own native code that Expo Go does not
ship would need a development build (`npx expo run:android`) instead.

### Choosing the backend (`src/lib/api.ts`)

The backend address is decided once, at start, in this order:

| Order | Source | Typical use |
|---|---|---|
| 1 | `EXPO_PUBLIC_API_URL` environment variable (inlined by Metro / EAS at bundle time) | Point Metro at another backend; set by store builds |
| 2 | `extra.apiUrl` in the app config (`app.config.js` copies `EXPO_PUBLIC_API_URL` into it) | Store builds |
| 3 | Default: `http://10.0.2.2:4000` on Android, `http://localhost:4000` elsewhere | Emulator against a laptop backend on port 4000 |

`10.0.2.2` is the Android emulator's alias for the laptop. Because the default port is 4000 and `backend-ts` defaults
to 8080, a local `backend-ts` needs either `EXPO_PUBLIC_API_URL=http://10.0.2.2:8080` when starting Metro, or
`PORT=4000` on the backend.

Examples:

```bash
# Metro against the hosted sandbox backend
EXPO_PUBLIC_API_URL=https://risips-demo-api.azurewebsites.net REACT_NATIVE_PACKAGER_HOSTNAME=127.0.0.1 npx expo start --lan --port 8081

# Metro against a local backend-ts on 8080
EXPO_PUBLIC_API_URL=http://10.0.2.2:8080 REACT_NATIVE_PACKAGER_HOSTNAME=127.0.0.1 npx expo start --lan --port 8081
```

Restart Metro (with `--clear` if needed) after changing the variable: it is baked into the bundle.

If the server can't be reached, every call fails with `ApiError` status `0` and the message
"Can't reach the Risips server at <url>. Is the backend running?", so the message itself tells you which address the app
is using.

## App variants and builds

One codebase, three variants. `app.config.js` reads `APP_VARIANT` (set by the `eas.json` build profile) and layers it on
`app.json`.

| Variant | `APP_VARIANT` | App name | Bundle id / package | Deep-link scheme | Update channel | Backend |
|---|---|---|---|---|---|---|
| Development | unset | Risips | `in.risips.app` (from app.json) | `risips` | none in Expo Go (code comes from Metro); app.json names `production` | Default address (emulator) or `EXPO_PUBLIC_API_URL` |
| Sandbox | `sandbox` | Risips Sandbox | `in.risips.app.sandbox` | `risips-sandbox` | `sandbox` | `https://risips-demo-api.azurewebsites.net` (set in eas.json) |
| Production | `production` | Risips | `in.risips.app` | `risips` | `production` | Whatever `EXPO_PUBLIC_API_URL` is at build time (not set in eas.json) |

Rules enforced by `app.config.js`:

- An unknown `APP_VARIANT` throws.
- **Sandbox and production refuse to build unless `EXPO_PUBLIC_API_URL` is an `https://` address.** This stops a store
  build from silently falling back to the emulator address. `eas.json` sets the URL for the sandbox profiles only, so
  the production profile gets it from outside eas.json (for EAS, from the EAS project's environment variables; check
  there before building).
- Store builds (sandbox and production) turn off Android cleartext (http) traffic.
- The sandbox variant gets an amber Android icon background so testers can tell it from the real app, and it can be
  installed next to it.
- Signed OTA updates are opt-in (`EXPO_UPDATES_SIGNING=1`). Leave it off: a build that embeds the certificate rejects
  every unsigned update.

`eas.json` build profiles:

| Profile | Variant | Output | Channel |
|---|---|---|---|
| `production` | production | store build (auto-increment) | `production` |
| `apk` | production | Android APK | `production` |
| `sandbox` | sandbox | store build (auto-increment) | `sandbox` |
| `sandbox-apk` | sandbox | Android APK | `sandbox` |

There is no `development` profile in `eas.json`.

Local APK without EAS: `npm run build:apk -- sandbox` (or `-- production`) runs `scripts/build-apk.js`. It takes the
variant's env from `eas.json`, needs the Android SDK and JDK 17, and the upload keystore password from the root `.env`
(ask the project owner; never commit it). The APK lands in `mobile/dist/`.

iOS: only the production bundle id has an App Store Connect app. A sandbox iOS build would need its own app record.

### Over-the-air (OTA) updates vs a new build

An installed app is a native **shell** (name, icon, permissions, native modules, update channel) plus the **JavaScript
screens**, which `eas update` can replace without a new install.

- `runtimeVersion` uses the `appVersion` policy, so an update reaches only builds with the same app version (`1.0.0`
  in app.json today).
- Publish from `mobile/` with the variant and backend the installed apps expect, for example:
  ```bash
  APP_VARIANT=sandbox EXPO_PUBLIC_API_URL=https://risips-demo-api.azurewebsites.net \
    npx eas-cli@latest update --channel sandbox --environment production --platform android --message "..."
  ```
  `package.json` has `update:ios` / `update:android` for the production channel. **`APP_VARIANT`,
  `EXPO_PUBLIC_API_URL` and `--channel` must match the target apps**, or the update ships the wrong backend address.
- An OTA from the current branch ships every app commit since the last update on that channel. Check the update list
  first.
- On a phone, an update is downloaded on one launch and applied on the next: close the app fully, reopen, close, reopen.
  Profile > "App version · Check for updates" shows which code is running (built-in or an update id, channel, runtime
  version) and can fetch and apply an update on the spot.

**A new native build is required (an OTA is not enough) when a change touches the shell**, for example:

- adding a package with native code (such as `expo-location`, which fresh KYC needed)
- adding or changing a config plugin, permission, `infoPlist` entry, icon, scheme or bundle id in `app.json` /
  `app.config.js`
- changing the app version (which also changes the runtime version)

Screens must not crash on builds that lack a newer native module. `onboarding/aadhaar.tsx` loads `expo-location` with
`require()` inside the action, not as a top-level import, and `src/lib/kyc.ts` `canDoFreshKyc()` checks for it the same
way. Follow that pattern when an OTA uses a module older builds may not have: Expo Router loads every screen file at
start, so a top-level import of a missing native module would crash the app on launch.

## Sandbox-only UI

Some UI exists only for test environments. **It follows the backend, not the build.** `src/lib/env.ts`:

- `VARIANT`: the build variant from `extra.variant` (`development`, `sandbox` or `production`).
- `useSandbox()`: calls `GET /health` once (cached for the app session) and returns `true` unless the backend answers
  `fp_env: "production"`. Until `/health` answers, or if it fails, it falls back to the build: `true` for every
  variant except `production`. A failed call is retried on the next screen that uses the hook.

So a development build pointed at a production backend shows no sandbox UI, and a production build pointed at the
sandbox backend does show it.

| Component | Where | What it does |
|---|---|---|
| `SandboxBadge` (`components/SandboxBadge.tsx`) | Root layout, over every screen | Small amber "SANDBOX" tag at the top |
| `DemoHint` (`components/ui.tsx`) | OTP screens, transaction OTP sheet, PAN, bank | Yellow box "Sandbox · ..." with a test tip or the OTP |
| `SandboxOnly` (`components/ui.tsx`) | Home tools, Profile, fund page | Renders its children only on a sandbox backend |
| Inline `useSandbox()` checks | `invest/pay.tsx`, `onboarding/kyc-web.tsx`, `profile/account.tsx` | Hints on the payment / AutoPay / DigiLocker / e-sign pages; the "FINTECHPRIMITIVES (SANDBOX)" label |

Things gated this way: the **API Log** tool on Home and "FintechPrimitives API log" in Profile (the `/apilog` screen,
which reads `GET /debug/trace`; the backend answers 404 there unless it runs in demo mode), the "Sandbox build" footer
in Profile, and "Sandbox only offers Regular plans" on the fund page.

The OTP hint needs two things: the backend must return `dev_otp` (it only does in demo mode) and the app must see a
sandbox backend.

## Theming (light and dark)

`src/theme/index.tsx`:

- Two palettes, `palettes.dark` and `palettes.light`, plus fonts, a type scale, `radius`, and helpers `ink(a)` (the
  foreground colour at an alpha) and `fade(a)` (the page background at an alpha).
- `ThemeProvider` loads the saved choice from SecureStore key `risips.theme`: `system` (default), `light` or `dark`.
  `system` follows the phone. The choice is also pushed to native UI with `Appearance.setColorScheme`.
- Users change it in **Settings > Appearance**.
- `useTheme()` returns `{ scheme, dark, colors, type, ink, fade, pref, setPref }`.

Rules for screen code: take colours from `useTheme().colors` (never hard-code white text or a dark background), and
build `StyleSheet`s inside a `useMemo` keyed on the theme when they use colours (see `onboarding/esign.tsx`
`makeS(theme)`). A few surfaces are deliberately fixed in both themes (for example the signature pad and the home promo
card).

## Session and auth (`src/lib/session.tsx`)

`SessionProvider` wraps the app and exposes `useSession()`:

| Field | Meaning |
|---|---|
| `user` | The public user record from `GET /me` (type `User`), or `null` |
| `ready` | The saved token has been loaded and `/me` tried |
| `unlocked`, `setUnlocked` | Set after sign-in, PIN/biometric unlock and onboarding "Get Started". Nothing reads it today (see note) |
| `setUser` | Replace the user (screens call it with the `user` returned by most backend calls) |
| `refresh()` | Re-read `/me`. A 401 clears the token and the user |
| `signIn(token, user)` | Saves the token, sets the user, marks unlocked |
| `signOut()` | `POST /auth/logout` (errors ignored), clears the token and user, goes to `/` |

Token handling (`src/lib/api.ts`): the session token is stored in SecureStore under `risips_token` and sent as
`Authorization: Bearer <token>`. Every request also sends `x-device-name` and `x-device-os` (shown under
Settings > Active devices).

Start-up routing (`src/app/index.tsx`, the splash):

1. Wait for the session to be `ready`.
2. No user: go to `/welcome`.
3. User with a PIN set (`pin_set`): go to `/unlock`.
4. Otherwise: go to `routeForStep(user.next_step)`.
5. A saved token with no user means `/me` failed without a 401 (server unreachable). The splash then shows
   "Can't reach Risips right now" and a Try again button instead of sending the investor to sign-up.

Unlock (`src/app/unlock.tsx`):

- If the user turned on biometrics (`user.biometric`), the biometric prompt opens at once. Success unlocks.
- Otherwise the user types the 4-digit PIN; it is checked with `POST /auth/pin/verify` when the 4th digit is entered.
- A 401 from that call (too many wrong tries, or an expired session) signs the user out after 1.5 s; they sign in again
  by email.
- After unlock the app goes to `routeForStep(user.next_step)`, so a half-onboarded user resumes onboarding.

Onboarding navigation helpers:

- `routeForStep(step)`: `'home'` maps to `/home`, anything else to `/onboarding/<step>`.
- `goNext(user)`: dismisses the whole stack (`router.dismissAll()` when possible), then `router.replace`s to the next
  step. This keeps finished steps off the back stack: going back to a finished step and submitting it again would fail
  ("already confirmed").

Note: the app does not re-lock when it goes to the background. The PIN/biometric prompt appears only on a cold start
(through the splash). `unlocked` is set in several places but no screen checks it.

## Calling the backend

### `api` and `ApiError` (`src/lib/api.ts`)

```ts
api.get<T>(path)   api.post<T>(path, body)   api.patch<T>(path, body)   api.put<T>(path, body)   api.del<T>(path)
```

- JSON in and out; `post` / `patch` / `put` default the body to `{}`.
- A network failure throws `ApiError(0, "Can't reach the Risips server at <url>. Is the backend running?")`.
- A non-2xx response throws `ApiError(status, data.error || "Request failed (<status>)", data)`. The full JSON body is in
  `err.data` (for example `checks` on a failed PAN check, `next_step`, or a `reason` on an OTP error).
- `fileUrl(path)` turns a backend file path (selfie, signature: signed, expiring links) into a full URL.

### `useApi` and `useAction` (`src/lib/hooks.ts`)

`useApi<T>(path, deps?)` loads a GET resource:

- Returns `{ data, error, loading, reload, setData }`.
- **Reloads every time the screen gains focus** (`useFocusEffect`), so going back to a screen refreshes it.
- A `null` path does nothing (use it for "not yet").
- When the path changes, old data is cleared; replies from older requests are ignored.

`useAction(fn)` wraps a button action:

- Returns `{ run, busy, error, setError }`.
- `run(...)` ignores taps while a run is in flight (a ref guard, so a fast double tap can't place two orders).
- A thrown error's `message` lands in `error`, and `run` resolves `undefined`. Screens show it with `<ErrorText>`.

### Transaction OTP (`src/components/TxnOtp.tsx`)

Every money or instruction change needs a 6-digit code (SEBI two-factor rule). `useTxnOtp()` opens a sheet, sends the
code with `POST /txn/otp { action }`, and calls the screen's `submit(otp)` with it. Actions: `lumpsum`, `cart`,
`redeem`, `switch`, `sip`, `sip_modify`, `sip_cancel`, `sip_pause`, `sip_resume`, `stp`, `stp_cancel`, `swp`,
`swp_cancel`, `nominee`, `delete_account`. A code error (`otp_required`, `wrong`, `expired`, `missing`, `locked`) keeps
the sheet open; any other error closes it and shows on the screen.
