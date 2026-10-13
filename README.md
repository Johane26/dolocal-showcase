# DoLocal — Local Services Marketplace (Engineering Showcase)

> **Active development — not production-ready.** This showcase describes engineering work in a private development repository. It does not claim a live deployment, production payment processing, complete financial operations, full test coverage, or commercial launch.

DoLocal is a Botswana-focused local-services marketplace built around service discovery, service requests, provider quotes, orders, fulfillment, and payment lifecycle workflows. Initial market scope uses BWP, English, and the Africa/Gaborone timezone.

The development repository is a modular monolith (not microservices): a FastAPI backend owns validation, authorization, lifecycle transitions, and persistence, with separate Next.js customer/provider and back-office applications consuming it over HTTP. Client-side validation supports usability but is never treated as a security boundary.

- **Developer:** [Johane26 on GitHub](https://github.com/Johane26)
- **Source code:** private development repository; access is not included in this public showcase.
- **Scope of this document:** architecture, stack, implemented behavior with file evidence, quality, setup, and explicit limitations. Development-repository paths below (for example `apps/api/src/api/routers/orders.py`) refer to the private codebase, not to files in this showcase.

## 1. Project overview

DoLocal connects customers looking for local services with providers who publish offerings, respond to service requests, submit quotes, and fulfill orders. Customers can browse published services, submit detailed service requests (including drafts, clarifications, and attachments), compare quotes, place direct or quote-based orders, pay through a checkout lifecycle, and track fulfillment. Providers manage service configuration and publication, respond to targeted requests, manage quotes and revisions, and execute fulfillment through assignment, booking, and completion transitions. Administrators review cancellations and reported content, inspect orders and payments, and follow an audit trail.

The implementation emphasizes vertical slices: domain rules, API contracts, PostgreSQL persistence with Alembic migrations, typed clients, and automated tests. Payment handling is deliberately isolated behind a provider-neutral boundary so marketplace logic does not depend on any specific payment service provider (PSP).

## 2. Architecture

DoLocal is a modular monolith with one authoritative backend API. There are no separate microservices and no shared business-logic package between clients; each frontend calls the API over HTTP with a Supabase bearer token.

```mermaid
flowchart LR
  subgraph Clients
    WEB[apps/web<br/>Next.js customer + provider]
    ADMIN[apps/admin<br/>Next.js back-office]
    MOB[apps/mobile<br/>Expo starter, no API integration]
  end
  subgraph API[apps/api - FastAPI modular monolith]
    ROUTERS[Routers: orders, quotes,<br/>service-requests, payments,<br/>webhooks, admin operations]
    AUTH[auth/dependencies.py<br/>JWKS verification + identity resolution]
    SVC[Services: order, quote_order,<br/>payment_lifecycle, payment_webhooks,<br/>order_cancellation, assignments]
    ORM[SQLAlchemy 2 models<br/>PostgreSQL persistence]
  end
  DB[(PostgreSQL<br/>Alembic migrations)]
  SUPA[Supabase Auth<br/>JWT issuer]
  STORE[Supabase Storage<br/>private request-attachment bucket]
  FAKE[FakePaymentProvider<br/>in-memory PSP double]

  WEB -->|Bearer token + Idempotency-Key| ROUTERS
  ADMIN -->|Bearer token + ADMIN role claim| ROUTERS
  ROUTERS --> AUTH --> SUPA
  ROUTERS --> SVC --> ORM --> DB
  SVC --> STORE
  ROUTERS --> FAKE
```

Main request flow:

1. The browser or admin client signs in through Supabase and sends the access token as a bearer token.
2. The API verifies the JWT against the Supabase JWKS issuer and audience, resolves or creates the local user, and supports a legacy token path only when explicitly configured.
3. Routers enforce ownership and role checks, then delegate to services for locking, validation, transitions, and persistence. Money and order creation require an `Idempotency-Key` header.
4. Payment calls go through the `PaymentProvider` protocol; only `FakePaymentProvider` is implemented. Webhooks arrive at `POST /payments/webhook` with an `X-Payment-Signature` header.
5. Request attachments use a private Supabase Storage bucket with owner and provider-scoped policies configured by migration; the API verifies uploads before accepting submissions.
6. A background asyncio task expires due quotes on API startup.

Responsibilities:

- `apps/api` — authoritative business rules, validation, authorization, lifecycle transitions, persistence.
- `apps/web` — customer and provider workflows (plus transitional `/admin/*` pages pending full cutover).
- `apps/admin` — authoritative back-office workflows: dashboard, providers, services, orders, cancellations, communication reports, reconciliation queue, audit trail.
- `apps/mobile` — Expo client; still the default starter screen with no DoLocal API integration. Post-MVP.


## 3. Technology stack

Only technologies with repository evidence are listed. Versions come from manifests.

| Area | Technology | Version | Purpose and evidence |
|---|---|---|---|
| Backend | Python | `>=3.14` (CI `3.14`) | API language; `apps/api/pyproject.toml`, `.github/workflows/ci.yml` |
| Backend | FastAPI | `>=0.141.1` | Routers, request validation, dependency injection; `apps/api/src/api/main.py`, `apps/api/src/api/routers/` |
| ORM | SQLAlchemy | `>=2.0,<3.0` | Models, row locking, constraints; `apps/api/src/api/models/` |
| Migrations | Alembic | `>=1.15,<2.0` | Versioned schema changes; `apps/api/migrations/versions/` |
| DB driver | psycopg (`binary`) | `>=3.2,<4.0` | PostgreSQL access; `apps/api/pyproject.toml` |
| Auth verification | PyJWT (`crypto`) | `>=2.14.0` | JWKS verification; `apps/api/src/api/auth/dependencies.py` |
| Env | python-dotenv | `>=1.0,<2.0` | Loads `apps/api/.env`; `apps/api/src/api/auth/config.py` |
| Server | uvicorn (`standard`) | `>=0.53.0` | ASGI server; `apps/api/pyproject.toml` |
| Data | PostgreSQL | Unpinned (service) | System of record; enums and partial unique indexes; `apps/api/src/api/models/order.py` |
| Web/Admin | Next.js | `16.3.5` | App Router frontends; `apps/web/package.json`, `apps/admin/package.json` |
| UI | React / React DOM | `19.2.8` (web/admin) | Components; manifests |
| Language | TypeScript | `^5` (web/admin) | Typed clients and pages; `tsc --noEmit` |
| Styling | Tailwind CSS | `^4` | Utility styling; manifests |
| Auth client | `@supabase/ssr`, `@supabase/supabase-js` | `^0.12.7` / `^2.117.x` | Sign-in, session helpers; `apps/web/app/login/page.tsx`, `apps/admin/lib/supabase/` |
| Storage | Supabase Storage | Service (no version) | Private `service-request-clarifications` bucket; `apps/api/src/api/routers/service_requests.py` |
| Validation | Pydantic (via FastAPI) | Via FastAPI dependency | Strict schemas; `apps/api/src/api/routers/quote_orders.py` |
| API quality | Ruff, MyPy strict, pytest | `>=0.16.8` / `>=2.3.1` / `>=9.1.1` | `ruff check`, `mypy`, `pytest`; `apps/api/pyproject.toml` |
| Client quality | ESLint, `eslint-config-next`, `tsc` | `^9` / `16.3.5` | `pnpm lint`, `pnpm typecheck`; manifests |
| Tooling | `uv`, `pnpm` | `uv 0.12.15` (CI), `pnpm 11.22.0` | Locked installs; `.github/workflows/ci.yml` |
| Mobile | Expo, expo-router, React Native | `~57.0.23` / `~57.0.21` / `0.86.3` | Starter shell only; `apps/mobile/package.json` |

Not claimed: Redis/caching, background queues beyond the in-process quote-expiry task, or any PSP SDK.

## 4. Engineering work demonstrated

> “Implemented” means the behavior exists in source. “Verified” is only stated where a test file or recorded check corroborates it. Historical pass counts in roadmap notes are not repeated as current results.

### Server-side authentication, authorization, and ownership

Problem: marketplace actions must not rely on UI gating; every read and write needs an independently enforced identity, ownership, and role decision.

Approach: the API verifies the Supabase JWT (JWKS, issuer, `authenticated` audience), resolves the local user, and threads authentication dependencies through routers. Customer routes scope by `customer_id`; provider routes require an active `ProviderMembership` (plus granular permissions such as `ORDER_CANCEL_PERMISSION`); admin routes require the server-managed `app_metadata.dolocal_role == "ADMIN"` claim. Ownership failures return `404` rather than `403` where appropriate to avoid leaking resource existence.

Evidence: `apps/api/src/api/auth/dependencies.py`, `apps/api/src/api/auth/config.py`, `apps/api/src/api/authorization/customer.py`, `apps/api/src/api/authorization/provider.py`, `apps/api/src/api/routers/admin_operations.py`, `apps/api/src/api/routers/orders.py`, `apps/web/app/login/page.tsx`, `apps/admin/lib/admin-auth.ts`.

### Idempotent creation, replay, and concurrency safeguards

Problem: retries on order, quote-accept, and payment checkout must not create duplicate financial records, while conflicting reuse of a key must be rejected loudly.

Approach: creation endpoints require an `Idempotency-Key` header. The API stores idempotency keys with payload hashes, returns `201` on first creation and `200` with the original response on safe replay, and returns `409` when the same key arrives with a different payload. Database unique constraints back the application check, with an integrity-error replay-lookup fallback for races. Payment idempotency uses a hash-derived advisory lock; webhook ingestion uses a transactional advisory lock on provider plus event identity.

Evidence: `apps/api/src/api/services/order.py`, `apps/api/src/api/services/quote_order.py`, `apps/api/src/api/services/payment_lifecycle.py`, `apps/api/src/api/services/payment_webhooks.py`, `apps/api/src/api/models/order.py`, `apps/api/src/api/models/payment.py`, `apps/api/src/api/routers/quote_orders.py`, `apps/api/src/api/routers/payments.py`, `apps/api/src/api/routers/orders.py`.

### Lifecycle validation for orders, fulfillment, payments, and cancellations

Problem: marketplace state (quotes, orders, fulfillment, payments, bookings) must only move through legal transitions, including cancellation and late-provider-event edge cases.

Approach: models define explicit allowed-transition maps enforced by `transition_to()` methods. Cancellation uses a request/decision flow: non-admin requests for in-progress work stay pending for admin review, while admins can intervene directly; decisions write audit events. Payment cancellation requires a single active checkout with a processing attempt and provider confirmation.

Evidence: `apps/api/src/api/models/order.py`, `apps/api/src/api/models/fulfillment.py`, `apps/api/src/api/models/payment.py`, `apps/api/src/api/services/order_cancellation.py`, `apps/api/src/api/services/payment_lifecycle.py`, `apps/api/src/api/routers/orders.py`, `apps/api/src/api/routers/fulfillment_lifecycle.py`, `apps/api/src/api/routers/bookings.py`, `apps/api/src/api/routers/assignments.py`.

### Database constraints, relationships, and financial snapshots

Problem: application checks alone cannot guarantee money totals, uniqueness, or exactly-one-fulfillment invariants under concurrency.

Approach: non-negative check constraints on money fields, unique idempotency pairs, partial unique indexes preventing duplicate orders per source request or quote, a one-fulfillment-per-order rule, and an immutable financial breakdown snapshot. Foreign keys tie users, providers, memberships, services, requests, quotes, orders, fulfillment, bookings, payments, refunds, payouts, disputes, reviews, notifications, and audit events together.

Evidence: `apps/api/src/api/models/order.py`, `apps/api/src/api/models/payment.py`, `apps/api/src/api/models/order_financial_breakdown.py`, `apps/api/src/api/models/fulfillment.py`, `apps/api/migrations/versions/`.

### Quote acceptance and consistent order creation

Problem: accepting a quote must atomically produce a coherent set of records without double-converting a request or leaving competitors active.

Approach: quote acceptance locks the quote, request, and related rows, validates snapshots and fulfillment capability, creates one order plus one line item plus the required single fulfillment, marks the quote accepted and the request converted, and marks competing active quotes unavailable. Expiry is handled both lazily and by the background worker. Quote creation enforces future expiry, normalized currency, total-equals-components, and pre-order communication-boundary text rules.

Evidence: `apps/api/src/api/services/quote_order.py`, `apps/api/src/api/services/quote_expiry.py`, `apps/api/src/api/routers/quote_orders.py`, `apps/api/src/api/models/quote.py`, `apps/api/src/api/models/quote_revision.py`.

### Signed webhooks, event persistence, and reconciliation flags

Problem: asynchronous provider notifications can be forged, duplicated, delayed, unmatched, or inconsistent with local state.

Approach: `POST /payments/webhook` verifies the raw-body HMAC signature before parsing, enforces a strict payload shape, and processes notifications transactionally. Every authenticated notification is persisted; duplicates replay idempotently; conflicting reuse is rejected; unmatched or lifecycle-conflicting events are recorded rather than silently applied; late or out-of-state captures are flagged for admin reconciliation review.

Evidence: `apps/api/src/api/routers/payment_webhooks.py`, `apps/api/src/api/services/payment_webhooks.py`, `apps/api/src/api/payments/provider.py`, `apps/api/src/api/payments/fake.py`, `apps/api/src/api/models/payment_webhook_event.py`, `apps/api/src/api/routers/admin_operations.py`.

### Provider-neutral payment boundary with a fake provider

Problem: marketplace checkout logic should not hard-code PSP specifics, but real PSP integration is not yet done.

Approach: application services depend only on the `PaymentProvider` protocol (checkout, cancel, verify, webhook verification), with optional refund, payout, and settlement protocols reserved for later. `FakePaymentProvider` is a deterministic in-memory double used for local verification; unconfigured or unknown provider values fail closed, and the fake-completion helper is unavailable outside fake/test mode.

Evidence: `apps/api/src/api/payments/provider.py`, `apps/api/src/api/payments/fake.py`, `apps/api/src/api/routers/payments.py`, `apps/api/src/api/services/payment_lifecycle.py`.

### Admin operations, reported-content review, and auditability

Problem: back-office actions need privileged reads and writes with a traceable record, without duplicating business rules in the client.

Approach: admin routers expose paginated provider, service, offering, and order views, provider activation, fulfillment assignment, cancellation queue and decision, communication-report review, reconciliation queue, and audit-event listing. The API enforces the admin claim on every protected read and write; the admin app adds a sign-in gate, safe return targets, unauthorized handling, and typed API clients. Cancellation decisions and report reviews write audit events.

Evidence: `apps/api/src/api/routers/admin_operations.py`, `apps/api/src/api/routers/orders.py`, `apps/api/src/api/routers/communication_reports.py`, `apps/api/src/api/services/order_cancellation.py`, `apps/api/src/api/services/assignments.py`, `apps/api/src/api/models/audit_event.py`, `apps/admin/app/page.tsx`, `apps/admin/lib/admin-auth.ts`, `apps/admin/lib/api/admin-operations.ts`, `apps/admin/README.md`.

## 5. Features by user role

Only features with source-code support are listed. Screenshots illustrate the UI; they do not prove backend behavior.

### Customer

- Browse published services and view customer-safe offering detail (contact and internal fields excluded; unpublished or ineligible offerings return `404`).
- Create and manage service requests: drafts, submission with attachments, own-request list/detail, cancellation, clarification threads, and targeted versus marketplace requests.
- Compare quotes and accept, reject, or track withdrawal/expiry; acceptance is idempotent and converts the request atomically.
- Create direct orders for eligible offerings (idempotent) and view customer order lists/detail with payment, fulfillment, booking, and cancellation-pending state.
- Start payment checkout, verify payment status, and cancel an active checkout where eligible.
- Submit communication-content reports on service requests; view loading, empty, unavailable, and error states where implemented.

Key files: `apps/api/src/api/routers/service_requests.py`, `apps/api/src/api/routers/quote_orders.py`, `apps/api/src/api/routers/orders.py`, `apps/api/src/api/routers/payments.py`, `apps/api/src/api/routers/customer_provider_services.py`, `apps/api/src/api/routers/services.py`.

### Service provider

- Establish provider context from authenticated identity and active membership.
- Create, configure, and publish service offerings with pricing, service-area, and fulfillment-capability validation.
- Discover actionable and explicitly targeted service requests; participate through clarifications and quotes.
- Create and revise quotes with expiry, pricing, fulfillment-model, and communication-boundary validation; withdraw own active quotes.
- View provider orders, fulfillment, assignments, and bookings; execute allowed fulfillment transitions.
- Request cancellation through the governed workflow (provider reason allow-list; membership and permission checks).

Key files: `apps/api/src/api/routers/provider_context.py`, `apps/api/src/api/routers/provider_services.py`, `apps/api/src/api/routers/provider_orders.py`, `apps/api/src/api/routers/quote_orders.py`, `apps/api/src/api/routers/fulfillment_lifecycle.py`, `apps/api/src/api/routers/bookings.py`, `apps/api/src/api/routers/assignments.py`.

### Administrator

- Sign in through Supabase and pass an admin role gate; every protected API read/write independently enforces the admin claim.
- Dashboard with provider, service, order, cancellation, and audit/reconciliation summaries.
- Manage providers (list, filter, activate), services, and provider offerings.
- Inspect orders with parties, line items, fulfillment, bookings, payments, attempts, webhook events, refunds, and payouts.
- Review cancellation requests and approve or reject with a required reason; decisions apply lifecycle effects and audit writes.
- Review reported communication content and mark reports reviewed/actioned.
- Inspect the reconciliation queue and the audit-event trail.

Key files: `apps/api/src/api/routers/admin_operations.py`, `apps/api/src/api/routers/orders.py`, `apps/api/src/api/routers/communication_reports.py`, `apps/admin/app/page.tsx`, `apps/admin/lib/api/admin-operations.ts`.

## 6. Screenshots

All images below are confirmed files in `Screenshots/` in this showcase repository. They show the UI only; backend behavior is established by the API implementation cited in Sections 4–5, not by the images.

### Getting started

![DoLocal landing page](Screenshots/landing_page.png)

*Landing page (`landing_page.png`) — marketing entry point for the customer app.*

![Marketplace loading state](Screenshots/landing-page-loading-state.png)

*Loading skeleton on the services page (`landing-page-loading-state.png`) — explicit loading state while marketplace data is fetched.*

### Customer workflows

![Customer sign-in](Screenshots/customer-sign-in.png)

*Customer sign-in (`customer-sign-in.png`) — Supabase session entry; API routes re-verify the bearer token server-side (`apps/api/src/api/auth/dependencies.py`).*

![Customer marketplace](Screenshots/customer-marketplace.png)

*Published service marketplace (`customer-marketplace.png`) — renders the customer-safe catalog from `GET /customer/services` (`apps/api/src/api/routers/customer_provider_services.py`).*

![Marketplace filters](Screenshots/customer-services-filters.png)

*Search and filter controls (`customer-services-filters.png`) — client-side discovery over published offerings.*

![Service detail](Screenshots/customer-service-detail.png)

*Service detail view (`customer-service-detail.png`) — customer-safe offering detail; unpublished or ineligible offerings return 404 from the API.*

![No services empty state](Screenshots/no-services-state.png)

*Empty search result (`no-services-state.png`) — explicit empty state with a clear-filters recovery action.*

![Customer service requests](Screenshots/customer-requests.png)

*Service requests list (`customer-requests.png`) — drafts, open requests, clarifications, and attachments (`apps/api/src/api/routers/service_requests.py`).*

![Customer orders](Screenshots/customer-orders.png)

*Customer orders list (`customer-orders.png`) — order status, payment status, and cancellation-pending flags (`apps/api/src/api/routers/orders.py`).*

![Customer order detail](Screenshots/customer-order-detail.png)

*Order detail (`customer-order-detail.png`) — fulfillment state, next-step guidance, and checkout entry for outstanding payment.*

### Payment workflows (fake provider — no real money moves)

![Test checkout](Screenshots/customer-test-checkout.png)

*Test checkout page (`customer-test-checkout.png`) — hosted checkout served by `FakePaymentProvider`; no real PSP is involved.*

![Successful test payment](Screenshots/customer-successful-payment.png)

*Verified payment confirmation (`customer-successful-payment.png`) — checkout completion followed by server-side verification (`apps/api/src/api/routers/payments.py`).*

![Failed simulated payment](Screenshots/customer-failed-payment.png)

*Simulated failure state (`customer-failed-payment.png`) — explicit failure messaging; no real money is charged.*

![Payment retry](Screenshots/customer-payment-retry.png)

*Payment retry (`customer-payment-retry.png`) — outstanding payment can be retried; the order itself is not cancelled by a failed attempt.*

### Provider workflows

![Provider account selection](Screenshots/Provider-acc-selection.png)

*Provider account selection (`Provider-acc-selection.png`) — workspace access is scoped to the provider membership resolved from the signed-in identity (`apps/api/src/api/routers/provider_context.py`).*

![Provider workspace](Screenshots/provider-workspace.png)

*Provider workspace (`provider-workspace.png`) — services, incoming requests, and order records for the selected provider.*

![New service offering](Screenshots/provider-new-offering.png)

*New offering entry (`provider-new-offering.png`) — start of the service configuration flow.*

![Configure offering](Screenshots/provider-configure-offering.png)

*Offering configuration (`provider-configure-offering.png`) — pricing, fulfillment model, service areas, and weekly availability (Africa/Gaborone).*

![Configure offering alternate view](Screenshots/provider-configure-offering-alternate.png)

*Availability configuration detail (`provider-configure-offering-alternate.png`) — recurring time windows with timezone handling.*

![Saved configuration review](Screenshots/config-review.png)

*Saved-configuration review (`config-review.png`) — publication readiness check against direct-order requirements before an offering goes live.*

![Provider quote response](Screenshots/provider-quote-response.png)

*Quote response form (`provider-quote-response.png`) — proposed price, fees, discount, tax, pricing mode, fulfillment model, and payment condition with a live total preview.*

### Administration (separate `apps/admin` application)

![Admin sign-in](Screenshots/admin-login.png)

*Administrator sign-in (`admin-login.png`) — admin access is verified by the server via `app_metadata.dolocal_role == ADMIN`, not by the UI.*

![Admin dashboard](Screenshots/Admin-dashboard.png)

*Platform administration dashboard (`Admin-dashboard.png`) — live totals for providers, services, orders, pending cancellations, and reconciliation events from the admin API (`apps/api/src/api/routers/admin_operations.py`).*

![Reported-content review](Screenshots/admin-reported-content.png)

*Communication-reports review (`admin-reported-content.png`) — reported content is shown only in this admin view; customers and providers never see each other's reports.*

### Platform evidence

![Supabase Auth users](Screenshots/Users-supabase.png)

*Supabase Auth users (`Users-supabase.png`) — the Auth project backing API JWT verification and the web/admin sign-in flows. Shown for platform context only; no account details are endorsed or exposed beyond what the console displays.*

## 7. Testing and code quality

These commands are available per manifests and CI configuration. They are documented as runnable; no current pass is claimed here.

| Area | Command | Working directory | What it runs |
|---|---|---|---|
| API tests | `uv run pytest` | `apps/api` | pytest suite covering orders, quotes, requests, payments, webhooks, provider boundaries |
| Web tests | `pnpm test` | `apps/web` | Node test runner over presentation and auth-state helper tests |
| Admin tests | `pnpm test` | `apps/admin` | Node test runner over admin presentation tests |
| API lint | `uv run ruff check .` | `apps/api` | Ruff |
| API types | `uv run mypy app` | `apps/api` | MyPy strict (path as configured in CI; requires confirmation) |
| Client lint | `pnpm lint` | `apps/web`, `apps/admin`, `apps/mobile` | ESLint / Expo lint |
| Client types | `pnpm typecheck` | `apps/web`, `apps/admin`, `apps/mobile` | `tsc --noEmit` |
| Client build | `pnpm build` | `apps/web`, `apps/admin` | `next build` |

CI (`.github/workflows/ci.yml`, verified): four jobs — API (Ruff + MyPy), Web (install, lint, typecheck), Admin (install, lint, typecheck), Mobile (install, lint, typecheck) — on `main` push/PR. CI does not run tests, builds, migrations, or deployments. Do not infer coverage or passing status from cache files in the development repository.

## 8. Local setup

Concise setup based on manifests, env examples, and code. Configure your own local values; never commit secrets.

Prerequisites: Python `3.14`, `uv 0.12.15`, Node `26`, `pnpm 11.22.0`, PostgreSQL, and a Supabase project for Auth (and Storage for request attachments).

Install (run each command from the repository root; each line is independent):

```bash
# API dependencies
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\api
uv sync --locked

# Web dependencies
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\web
pnpm install --frozen-lockfile

# Admin dependencies
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\admin
pnpm install --frozen-lockfile

# Mobile dependencies
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\mobile
pnpm install --frozen-lockfile
```

Environment variables (names only; values redacted). Only these API variables are read by code (`apps/api/src/api/auth/config.py`, `apps/api/src/api/database/connection.py`, `apps/api/migrations/env.py`, `apps/api/src/api/routers/payments.py`, `apps/api/src/api/main.py`):

- API (`apps/api/.env.example` in development): `DATABASE_URL` (required), `AUTH_ISSUER_URL` (required), `SUPABASE_ANON_KEY`, `CORS_ORIGINS` (defaults to `http://localhost:3000,http://localhost:3001`), `PAYMENT_PROVIDER` (empty/`fake`/`test` selects the fake provider), `API_BASE_URL` (defaults to `http://localhost:8000`). Other placeholders in `.env.example` are not read by the code inspected and are not documented here.
- Web (`apps/web/.env.local.example`): `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`.
- Admin (`apps/admin/.env.local.example`): `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`.

Database: set `DATABASE_URL`, then apply the migrations in `apps/api/migrations/versions/` with Alembic (`apps/api/migrations/env.py` reads `DATABASE_URL` and fails clearly when it is missing). The attachment-storage migration configures the private `service-request-clarifications` bucket only when the target database includes Supabase Storage; on plain local PostgreSQL it is skipped.

Run (ports by convention; no API start command is documented in the development repository, so the API entry below is marked unverified):

```bash
# API (unverified — no start command is documented in the development
# repository; from the code layout `apps/api/src/api/main.py` defines `app`,
# so the expected form is `uvicorn api.main:app`, run from `apps/api/src`)
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\api
uv run uvicorn api.main:app --reload

# Web (http://localhost:3000)
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\web
pnpm dev

# Admin (http://localhost:3001)
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\admin
pnpm dev -- --port 3001

# Mobile (Expo)
cd C:\Users\Pontie\Documents\GitHub\dolocal\apps\mobile
pnpm start
```

The API must allow both `http://localhost:3000` and `http://localhost:3001` through `CORS_ORIGINS`. Admin sign-in requires an account whose `app_metadata.dolocal_role` is `ADMIN`; the API enforces this independently.

Requires confirmation: the API start command above is inferred from the code layout, not from a documented run step; the exact Alembic command used locally; and seed data availability.

## 9. Current limitations and future work

- Payments use `FakePaymentProvider`; no live PSP integration is claimed. Non-fake provider values fail closed.
- Refund, payout, and settlement execution are not complete; related models exist but further money movement is not established.
- Reconciliation is a read-only queue of flagged webhook events; resolution workflows are not fully implemented.
- Browser end-to-end coverage and full financial lifecycle verification remain open; historical test notes are not current-run evidence.
- The mobile app is a starter screen and is not integrated with the DoLocal API.
- Deployment infrastructure and production readiness have not been established (no Dockerfile, compose, or deploy workflow found; CI covers lint/typecheck only).

This project is not production-ready, deployed, fully tested, or commercially launched.

## 10. My engineering contribution

DoLocal is a practical demonstration of full-stack software engineering: FastAPI and Pydantic API design, PostgreSQL data modeling with SQLAlchemy and Alembic integrity constraints, Supabase authentication and storage integration, server-side authorization and ownership enforcement, order/fulfillment/payment state-machine modeling, idempotent creation with concurrency handling, signed-webhook ingestion with reconciliation surfacing, provider-boundary integration design, and back-office operational tooling with auditability.

No team size, employment history, usage metrics, customers, revenue, production impact, or company status is claimed. The showcase intentionally excludes private source code, credentials, personal data, and confidential implementation details.
