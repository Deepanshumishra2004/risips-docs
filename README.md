# risips-docs

Developer and QA documentation for **Risips**, a goal-based mutual fund investing app (client: Arthasuta Finedu
Private Limited). The code lives in `akashatbest/risips` (branch `dev`): the backend in `backend-ts/`, the app in
`mobile/`.

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
