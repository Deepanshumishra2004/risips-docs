# Risips backend (TypeScript)

Developer guide for `backend-ts/`, the API behind the Risips mobile app. Verified against the code on branch `dev`
at commit `428814b` (2026-11-02).

Other documents in this folder:

| File | What it covers |
|---|---|
| [API.md](API.md) | Every endpoint: auth, rate limit, request fields and rules, responses, errors |
| [FLOWS.md](FLOWS.md) | End-to-end request sequences with example JSON, plus sandbox test data |
| [CHANGELOG.md](CHANGELOG.md) | Recent changes (2026-10-30 to 2026-11-02) |

---

## 1. What the backend does

Risips is a goal-based mutual fund app built for Arthasuta Finedu Private Limited (an AMFI-registered distributor).
The backend is the only component that talks to the fund platform. It:

- **Signs investors in**: email OTP (sign-up and sign-in are the same step), mobile OTP, a 4-digit app PIN, bearer
  sessions.
- **Onboards investors (KYC)** through FintechPrimitives (FP, by Cybrilla):
  - PAN, name and date of birth are checked with FP **pre-verification**, which also returns the PAN's KYC
    readiness at the KRA.
  - Bank accounts are checked with FP pre-verification too.
  - A PAN with no KRA record can do a **fresh KYC** through Cybrilla POA's KYC Forms API: DigiLocker Aadhaar,
    signature, e-sign of the KYC form.
  - On the final eSign step the investor profile, contacts, bank, nominee and MF investment account are created in FP.
- **Places orders** on FP's ONDC gateway: lumpsum purchases (paid on FP's UPI checkout), redemptions, switches,
  SIPs (with UPI AutoPay / eNACH mandates), STPs (switch plans) and SWPs (redemption plans), plus a multi-fund cart.
- **Shows funds**: a curated catalogue (`data/catalog.json`, 35 ICICI Pru and ABSL schemes) with today's NAV laid
  over it from **AMFI**'s daily NAV file.
- **Imports funds held elsewhere** from **MF Central** (CAS via MF Central's hosted journey), with a local stand-in
  in demo mode.
- **Delivers OTPs** through **MSG91** (SMS and email), once its templates are configured. Until then codes go to the
  server log, and in demo mode they are also returned in API responses as `dev_otp`.
- Serves reports, notifications, settings, a rule-based assistant, FP webhooks, and a read-only **Admin API** for the
  admin dashboard (run by another team on another server).

## 2. Stack

| | |
|---|---|
| Runtime | Node 24 (any Node >= 22.18). `.ts` files run directly with `--experimental-strip-types`; there is no build step |
| HTTP | Express 5, helmet, express-rate-limit |
| Validation | zod 4 |
| Database | PostgreSQL via Drizzle ORM (`pg` pool in production, embedded PGlite on a laptop / the sandbox) |
| Tests | `node:test` |

Only "erasable" TypeScript is allowed (`erasableSyntaxOnly`): no `enum`, `namespace` or constructor parameter
properties; type-only imports say `import type`; relative imports end in `.ts`.

## 3. Architecture

A request flows through these layers:

```
routes/x.routes.ts        path -> middleware chain -> controller
  requireAuth             bearer session (middleware/auth.ts)
  limits.*                rate limits (middleware/rateLimit.ts)
  validate(schema)        zod schema for body / query / params (validation/*.ts)
controllers/x.controller.ts   reads the validated input, calls a service, sends JSON
services/x.service.ts         business logic, no Express (takes a user / ctx and plain data)
  db/repositories/*           every SQL query (Drizzle)
  integrations/*              FP, AMFI, MF Central, MSG91, NSE holidays
```

Errors are thrown as `ApiError(status, message, extra)` and turned into `{ error, ...extra }` by
`middleware/errorHandler.ts` (see [API.md](API.md#error-envelope)).

### Folder layout

```
src/
  server.ts            starts HTTP + background jobs; graceful shutdown on SIGTERM/SIGINT (30 s cap)
  app.ts               Express app: request id, helmet, global rate limit, JSON body limit, routes, 404, errors
  config/env.ts        loads .env, validates it with zod, exports one typed `config` and `paths`
  routes/              one file per module; index.ts mounts them all
  validation/          zod schemas (messages are what the app shows)
  controllers/         thin: validated input -> service -> res.json
  services/            business logic (onboarding, kyc, orders, sip, stp, swp, cart, catalog, funds, reports, ...)
  integrations/        fp.client.ts (+ fp.schemas.ts models), amfi.ts, mfcentral.client.ts, mfcentral.sandbox.ts,
                       msg91.ts, nse.ts
  db/                  connection.ts (pg / PGlite + Drizzle), schema.ts, migrations.ts, sqliteImport.ts, repositories/
  middleware/          auth, adminAuth, validate, rateLimit, security (helmet, htmlPage, noStore), bodyParser,
                       errorHandler, requestContext
  jobs/scheduler.ts    background timers
  types/               domain, catalogue, external-holding types; Express request extension
  utils/               errors, logger, crypto, format (dates/money), mask, redact (PII in logs), pincode, request
  content/             promos.json, announcements.json, market-holidays.json (edited without a code change)
data/                  catalog.json, catalog-rules.production.json, amfi_navs.json (cache, gitignored)
scripts/               catalogue build, fund data, production rules, webhook registration, SQLite copy, deploy
test/                  node:test suites; smoke-onboard.cjs drives a running server end to end
```

## 4. Running locally

```bash
cd backend-ts
npm install
cp .env.example .env     # then fill in the keys you need (never commit .env)
npm run dev              # watch mode; sandbox unless FP_ENV=production
```

The server listens on `PORT` (default 8080) on `0.0.0.0`. Check it with `GET /health`.

- **Sandbox vs production.** `FP_ENV` unset (or `sandbox`) uses FP's sandbox tenant. `FP_ENV=production` talks to the
  live tenant and moves real money. Production refuses to start without `APP_SECRET` and `DATABASE_URL`, or with
  `DEMO_MODE` on, and it needs every `FP_*` / `FP_PV_*` production key.
- **Demo mode.** On whenever `FP_ENV` is not production **unless `DEMO_MODE=0`** (`config.demoMode = !production &&
  DEMO_MODE !== '0'`). In demo mode:
  - every OTP response also carries the code as `dev_otp` (sign-in, mobile, transaction 2FA, legacy Aadhaar);
  - MSG91 is never used;
  - the MF Central stand-in pages (`/mfc-sandbox/*`) are mounted;
  - `GET /debug/trace` returns the FP call log (shared by every user, so demo only);
  - the FP call log is not redacted.
- **Config file.** `config/env.ts` reads `RISIPS_ENV` if set, else `backend-ts/.env`, else `../.env`. Variables
  already in the environment win over the file. A key set to nothing (`NAME=`) counts as not set.
- **State folder.** Uploads, logs, the embedded database and `app.secret` live in `RISIPS_STATE_DIR` (default: the
  `backend-ts` folder; with `FP_ENV=production`, `backend-ts/state-production`).

### npm scripts

| Script | What it does |
|---|---|
| `npm start` | Runs `src/server.ts` |
| `npm run dev` | Same, with `--watch` |
| `npm run typecheck` | `tsc --noEmit` (strict) |
| `npm run lint` | ESLint (type-aware: no un-awaited promises, no await on a non-promise) |
| `npm test` | `node --test` over `test/*.test.ts`, one file at a time (security, rate limits, validation, migrations, Admin API, FP schemas, units) |
| `npm run check` | typecheck + lint + test |
| `npm run catalog` | Rebuilds `data/catalog.json` from FP (sandbox) + AMFI history (`scripts/build-catalog.ts`), then runs `fund-data` |
| `npm run fund-data` | Adds expense ratio (AMFI TER), managers/objective and riskometer (Kuvera public data) and portfolio holdings (fund house monthly files) to `catalog.json` (`scripts/fund-data.ts`) |
| `npm run migrate-sqlite -- <path>` | Copies an old SQLite database into the Postgres in `DATABASE_URL` (one transaction; refuses if Postgres already has users) |

Other scripts, run with `node --experimental-strip-types scripts/<name>.ts`:

| Script | What it does |
|---|---|
| `prod-rules.ts` | With `FP_ENV=production`: writes `data/catalog-rules.production.json` (each scheme's trading rules on the live tenant). Read-only against FP |
| `stp-rules.ts` | Patches STP, SWP and switch-in rules into `catalog.json` without a rebuild |
| `fund-facts.ts` | Adds NAV-history `history` (since date, CAGR) to `catalog.json` |
| `register-webhooks.ts` | Dry run by default; `--apply` registers FP order/plan webhooks at `<PUBLIC_URL>/fp/webhook/<FP_WEBHOOK_TOKEN>` (needs an https `PUBLIC_URL`) |
| `deploy.sh` | Zips and deploys to an Azure App Service named by `AZ_RG` / `AZ_APP` (no default target) |

## 5. Environment variables

Names and meaning only. Copy `.env.example` and fill in values; never commit `.env`.

### Core

| Variable | Meaning |
|---|---|
| `FP_ENV` | `sandbox` (default) or `production` (live FP tenant, real money) |
| `DEMO_MODE` | Demo helpers (see above). On by default outside production; `0` turns it off. Production refuses any other value |
| `PORT` | HTTP port (default 8080) |
| `PUBLIC_URL` | Base URL the phone uses to reach this server; used for payment / mandate / KYC return URLs and the MF Central stand-in. Default `http://10.0.2.2:<PORT>` (Android emulator) |
| `RISIPS_STATE_DIR` | Writable folder for uploads, logs, embedded database, `app.secret` |
| `RISIPS_ENV` | Path of an env file to load instead of `.env` |
| `LOG_LEVEL` | `debug`, `info` (default), `warn`, `error`. Logs are JSON lines in production |

### Database

| Variable | Meaning |
|---|---|
| `DATABASE_URL` | `postgres://` URL. Required in production. Empty: embedded Postgres (PGlite) in `<state>/data/pg` |

### Security

| Variable | Meaning |
|---|---|
| `APP_SECRET` | 32+ characters. Hashes OTP codes and signs selfie/signature links. Required in production; without it the sandbox generates one into `<state>/app.secret` |
| `TRUST_PROXY` | Number of reverse-proxy hops in front (default 1), or `false`. `req.ip` (rate limits, FP `user_ip`) is only the client's address when this is right |
| `SESSION_IDLE_DAYS` | Session ends after this many days unused (default 30) |
| `SESSION_MAX_DAYS` | Session ends after this many days in total (default 180) |
| `LOG_OTPS` | `0` stops OTPs being printed to the server log when no MSG91 channel delivered them. Default: printed (production included) |
| `RATE_LIMIT_SCALE` | Multiplies every middleware rate limit (default 1; tests raise it) |
| `FP_WEBHOOK_TOKEN` | 24+ character secret path segment for `POST /fp/webhook/:token`. Webhooks answer 404 until it is set |

### Admin API

| Variable | Meaning |
|---|---|
| `ADMIN_API_KEY` | 24+ characters. The Admin API (`/admin/v1`) answers 404 unless set; callers send it as `x-admin-key` |
| `ADMIN_API_ALLOWED_IPS` | Comma-separated caller IPs (optional) |

### FintechPrimitives

| Variable | Meaning |
|---|---|
| `FP_SANDBOX_CLIENT_ID`, `FP_SANDBOX_CLIENT_SECRET` | Sandbox main-API credentials (required outside production) |
| `FP_PV_SANDBOX_CLIENT_ID`, `FP_PV_SANDBOX_CLIENT_SECRET` | Sandbox pre-verification / KYC Forms credentials (required outside production) |
| `FP_SANDBOX_BASE_URL`, `FP_SANDBOX_TENANT`, `FP_PV_SANDBOX_BASE_URL`, `FP_PV_SANDBOX_AUTH_TENANT` | Optional sandbox overrides (defaults are built in) |
| `FP_BASE_URL`, `FP_TENANT`, `FP_CLIENT_ID`, `FP_CLIENT_SECRET` | Production main API (required with `FP_ENV=production`) |
| `FP_PV_BASE_URL`, `FP_PV_AUTH_URL`, `FP_PV_AUTH_TENANT`, `FP_PV_CLIENT_ID`, `FP_PV_CLIENT_SECRET` | Production pre-verification (required with `FP_ENV=production`) |

### MF Central (all of the first eight, or none: then demo mode uses the local stand-in)

| Variable | Meaning |
|---|---|
| `MFC_BASE_URL` | MF Central API base (default: their UAT host) |
| `MFC_CLIENT_ID`, `MFC_CLIENT_SECRET`, `MFC_USERNAME`, `MFC_PASSWORD` | OAuth credentials |
| `MFC_AES_KEY` | Payload encryption key |
| `MFC_PRIVATE_JWK`, `MFC_PUBLIC_JWK` | Our signing key and MF Central's verification key, as one-line JSON |
| `MFC_AES_IV` | Optional IV override |
| `MFC_EP_SUMMARY`, `MFC_EP_DETAIL`, `MFC_EP_CONSENT`, `MFC_EP_VALIDATE_QR`, `MFC_EP_DOCUMENT` | Optional endpoint-name overrides (v1 names by default) |

### MSG91 (OTP delivery; each channel turns on once its keys are set, never in demo mode)

| Variable | Meaning |
|---|---|
| `MSG91_AUTHKEY` | API key |
| `MSG91_SMS_TEMPLATE_ID` | Flow template linked to the DLT-approved OTP text (variables `otp`, `purpose`, `minutes`) |
| `MSG91_EMAIL_TEMPLATE_ID`, `MSG91_EMAIL_DOMAIN`, `MSG91_EMAIL_FROM` | Email channel (all three needed) |

## 6. Background jobs

`src/jobs/scheduler.ts` runs timers inside the API process. A run is skipped while the previous one is still going;
a failure is logged and the timer keeps going. Run only one instance, or these run once per instance.

| Job | Every | What it does |
|---|---|---|
| `sip-first-payments` | 10 min | Refreshes recent SIPs waiting on their up-front first payment; cancels a SIP whose first payment failed before FP debits the mandate |
| `account-purge` | 1 h | Erases accounts whose 30-day deletion window has passed |
| `mfc-imports` | 1 min | Polls MF Central for imports waiting on the statement (gives up after 1 h) |
| `kyc-approvals` | 30 min | For fresh-KYC investors whose form is `submitted` and not yet approved: re-runs pre-verification; when readiness is `verified`, sets `kyc_approved_at` and orders open. Throttled to once per 10 min per investor |
| `amfi-navs` | 1 h | Refreshes AMFI's daily NAV download when it is older than 4 h (the fund list then shows today's NAV and returns) |
| `session-cleanup` | 6 h | Deletes expired sessions and OTPs that expired over an hour ago |

At start-up the AMFI NAV index is also loaded once.

## 7. Database and migrations

PostgreSQL. Tables (`src/db/schema.ts`):

| Table | Holds |
|---|---|
| `users` | `id`, `email` (unique), `mobile`, `pin_hash`, `data` (JSONB: everything else about the investor), `created_at` |
| `sessions` | `token_hash` (SHA-256 of the bearer token), `user_id`, `device`, `os`, `ip`, `created_at`, `last_seen` |
| `otps` | `(purpose, target)` key, `code_hash` (HMAC), `expires_at` (epoch ms), `attempts` |
| `orders` | One row per FP order or plan: `id` (FP id), `user_id`, `kind` (`purchase`, `redemption`, `switch`, `sip`, `stp`, `swp`), `isin`, `isin_to`, `amount`, `state`, `data` (JSONB), timestamps, `seq` |
| `checkouts` | Cart checkouts (`chk_…`) |
| `watchlist` | `(user_id, isin)` |
| `ext_imports` | MF Central import snapshot per user |
| `mfc_sandbox` | MF Central stand-in sessions (demo) |
| `meta` | `schema_version` |

- **Migrations** (`src/db/migrations.ts`) run at start-up, each in one transaction, with a row lock so two processes
  starting together don't both apply one. Migration 1 is the JavaScript backend's schema; migration 2 hashes session
  tokens and OTP codes and adds an `orders(kind, state)` index.
- **Schema changes:** add a migration **and** change `schema.ts`. `test/schema.test.ts` fails if they disagree.
  drizzle-kit is not used.
- **Timestamps** come out as `'YYYY-MM-DD HH:MM:SS'` in UTC (no zone suffix).
- **Concurrency:** read-modify-write of `users.data` (cart, MF Central state, deletion, notifications) runs under a row
  lock (`usersRepo.update`).
- **Embedded Postgres:** without `DATABASE_URL`, PGlite in `<state>/data/pg`; a lock file stops a second server
  opening the same folder. An old `data/risips.db` (SQLite) is copied in on first start.
- **One process:** rate-limit counters, OTP cross-code counters, the cart double-tap guard and the jobs are in memory.
  Before running more than one instance, move them to a shared store / worker.

## 8. Environments

| Environment | Where | Notes |
|---|---|---|
| Laptop | `npm run dev` | Sandbox FP, demo mode, embedded Postgres |
| Sandbox | `https://risips-demo-api.azurewebsites.net` | Runs `backend-ts` since 2026-11-02, built from `dev`. FP sandbox tenant, demo mode on (responses carry `dev_otp`), embedded Postgres. The "Risips Sandbox" app talks to it. QA tests here with Postman |
| Production | production (AWS) | `FP_ENV=production`, Postgres database, demo mode off. Real money: production orders and PAN checks on real people need the product owner's OK |

`GET /health` answers `{ "ok": true, "fp_env": "sandbox", "demo": true, "schemes": 35 }`; check `fp_env` before
testing.

Branches: work goes to `dev`; `prod` is what production runs, moved only by the product owner.

## 9. Security notes

- **Sessions.** `POST /auth/email/verify` returns a random 24-byte token (32 base64url characters). Send it as
  `Authorization: Bearer <token>`. Only its SHA-256 is stored. A session ends after `SESSION_IDLE_DAYS` (30) unused or
  `SESSION_MAX_DAYS` (180) in total; then `401 "Please sign in again."`.
- **PIN.** scrypt, salted per user, constant-time compare. 5 wrong PINs in a row end the session
  (`401 reason: "pin_locked"`).
- **OTPs** (`services/otp.service.ts`, `db/repositories/otps.repository.ts`):
  - stored only as HMACs; sign-in codes are 6 digits, valid 120 s; transaction codes 6 digits, valid 300 s;
  - each code locks after 5 wrong tries (`400 reason: "locked"`); a wrong code returns `attempts_left`;
  - per purpose + target: at most one new code per 30 s and 20 per hour; at most 10 wrong codes per hour and 20 per
    day across codes. Past that, checking a code **and** asking for a new one both answer
    `429 { reason: "rate_limited", retry_after }`, so no unusable code is sent;
  - these counters are in memory: a restart clears them.
- **Middleware rate limits** (per IP / user / email, `429 reason: "rate_limited"`, wait in the `RateLimit` header):
  see the table in [API.md](API.md#rate-limits).
- **PAN masking.** The full PAN is never in a response: users carry `pan_masked` (`*******51Z`), the nominee's PAN is
  masked, PAN-check results mask `checks.pan.value`, and the MF Central status hides the KYC PAN. Bank accounts are
  returned as `account_masked`; Aadhaar as `XXXX XXXX 1234`.
- **Logs.** PAN, bank, phone, email, address and names are masked in the FP call log outside demo mode. Every request
  gets an id (`X-Request-Id`, or the caller's own if it is 8-64 safe characters), logged with the user id.
- **Errors.** FP's raw replies and stack traces stay in the log; the app gets a message.
- **Headers.** helmet (strict CSP for JSON; a page CSP for the few HTML pages), no `X-Powered-By`, `Cache-Control:
  no-store` on personal data. JSON bodies are capped at 100 KB (12 MB on `/onb/selfie`, `/onb/signature`,
  `/external/qr`).
- **Files.** Selfies and signatures are served only through signed links that expire within 2 hours; uploads must
  really be PNG or JPEG (500 bytes to 8 MB).
- **Webhooks.** The secret sits in the URL (constant-time compare). The event only triggers a re-fetch from FP.
- **Admin API.** Key compared in constant time, optional IP allowlist, GET only, audit log
  (`logs/admin_api.log`, with the `x-admin-actor` header), own rate limit.

## 10. Things that differ from older notes

- `backend-ts/README.md` says OTPs are printed to the log "only when `LOG_OTPS` allows (off in production)". The code
  defaults `LOG_OTPS` to on everywhere, production included (`config.logOtps = LOG_OTPS !== '0'`; the comment in
  `env.ts` says this is deliberate until the MSG91 templates are set). Set `LOG_OTPS=0` to stop it.
- `risk_source` (where a fund's risk level comes from) is stored in `catalog.json` but is **not** included in any API
  response yet: `summary()` in `catalog.service.ts` does not copy it.
