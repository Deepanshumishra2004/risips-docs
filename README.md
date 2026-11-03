# risips-docs

Developer and QA documentation for **Risips**, a goal-based mutual fund investing app (client: Arthasuta Finedu
Private Limited). The code lives in `akashatbest/risips` (branch `dev`): the backend in `backend-ts/`, the app in
`mobile/`.

## API testing guide for QA (`api-testing/`)

Start with [api-testing/README.md](api-testing/README.md): how to log in with Postman, which API to call in which
order, what each response and error means, test data, and a checklist to retest the reported bugs. Every request in
it was run on the sandbox.

| Doc | Covers |
|---|---|
| [01-login-and-account.md](api-testing/01-login-and-account.md) | Email and mobile OTP, PIN, sessions |
| [02-onboarding-kyc-compliant.md](api-testing/02-onboarding-kyc-compliant.md) | Onboarding for a PAN that already has KYC |
| [03-onboarding-fresh-kyc.md](api-testing/03-onboarding-fresh-kyc.md) | Onboarding with fresh KYC (no KRA record) |
| [04-funds.md](api-testing/04-funds.md) | Fund list, filters, fund page, compare, watchlist |
| [05-orders-and-payments.md](api-testing/05-orders-and-payments.md) | Lumpsum, redeem, switch, cart, payment |
| [06-sip-stp-swp.md](api-testing/06-sip-stp-swp.md) | AutoPay, SIP, STP, SWP |
| [07-reports-settings-other.md](api-testing/07-reports-settings-other.md) | Reports, notifications, settings, MF Central import, assistant |
| [08-bug-retest-checklist.md](api-testing/08-bug-retest-checklist.md) | Retest steps for every RSP bug |

## Backend (`backend-ts/`)

| Doc | What's in it |
|---|---|
| [backend/README.md](backend/README.md) | What the backend does, how it's built, how to run it, environment variables, jobs, security |
| [backend/API.md](backend/API.md) | Every endpoint: request fields and rules, responses, error codes |
| [backend/FLOWS.md](backend/FLOWS.md) | Step-by-step request sequences (sign-up, onboarding, fresh KYC, orders, SIP, STP, SWP) with sandbox test data |
| [backend/CHANGELOG.md](backend/CHANGELOG.md) | What changed, by date |

## Mobile app (`mobile/`)

| Doc | What's in it |
|---|---|
| [mobile/README.md](mobile/README.md) | Tech stack, folder layout, running locally, app variants, builds and OTA updates |
| [mobile/SCREENS.md](mobile/SCREENS.md) | Every screen: what it shows, which API it calls, where it goes next |
| [mobile/ONBOARDING.md](mobile/ONBOARDING.md) | The onboarding flow, including fresh KYC for a PAN with no KRA record |
| [mobile/TESTING.md](mobile/TESTING.md) | How to test in the app, with sandbox test data |

## Environments

| | Backend | Notes |
|---|---|---|
| Sandbox | `https://risips-demo-api.azurewebsites.net` | FintechPrimitives sandbox; OTPs are returned in responses (`dev_otp`) and shown in the app |
| Production | AWS | Real money. Not for testing |

## Keeping these docs current

Every change to `backend-ts/` or `mobile/` that affects an endpoint, a field, an error message, a validation rule, a
screen or a flow is reflected here in the same piece of work, with a line in the CHANGELOG.
