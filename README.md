# DoLocal — Local Services Marketplace

DoLocal is an actively developed local-services marketplace designed for Botswana. It connects customers looking for services with providers who can publish offerings, respond to service requests, and manage orders.

The project focuses on building reliable marketplace workflows with clear business rules, API-level authorization, and a modular architecture.

## Project Highlights

- **Provider service management:** Configure services and manage publication workflows.
- **Marketplace discovery:** Browse published services.
- **Customer requests and quotes:** Submit service requests, review provider quotes, and accept or reject offers.
- **Direct orders:** Create orders directly for eligible services.
- **Order management:** Support customer and provider order views with lifecycle controls.
- **Backend business rules:** Centralize validation, authorization, persistence, and lifecycle transitions in the API.
- **Administration:** Develop internal tools for marketplace operations and oversight.

## Technology Stack

| Area | Technologies |
|---|---|
| Backend API | Python, FastAPI |
| Data persistence | PostgreSQL, SQLAlchemy, Alembic |
| Customer and provider web application | Next.js, TypeScript |
| Administration | Separate Next.js application |
| Mobile client | Separate application; marketplace workflows remain post-MVP |
| Tooling | `uv`, `pnpm` |

## Architecture

DoLocal is structured as a modular monolith with a shared backend API.

- `apps/api` — business logic, validation, authorization, lifecycle transitions, and persistence.
- `apps/web` — customer and provider workflows.
- `apps/admin` — internal administration and operational interfaces.
- `apps/mobile` — a client of the shared API, with marketplace workflows still planned.

The API is the authoritative boundary for business rules. Client-side validation supports usability but is not treated as a security boundary.

## Engineering Focus

Development emphasizes end-to-end vertical slices across domain rules, API contracts, database migrations, user interfaces, and automated tests.

Key areas of engineering work include:

- Modeling service requests, quotes, orders, and their lifecycle transitions.
- Enforcing provider and customer authorization boundaries.
- Maintaining database changes through migrations.
- Separating payment-provider integration from core marketplace workflows.
- Building operational interfaces for administration and oversight.

## Current Status

**Active development — not production-ready.**

Core marketplace workflows have been implemented to varying degrees. Payment and financial operations remain in progress.

The current development scope does **not** establish production payment processing, completed refund or payout execution, fully verified financial reconciliation, or production deployment.

Historical test results documented in the development repository are not a guarantee that every test passes against the current working tree. Full browser end-to-end verification and financial lifecycle verification remain outstanding.

## Project Links

- **Developer portfolio:** [Johane26 on GitHub](https://github.com/Johane26)
- **Source code:** Private development repository; access is not included in this public showcase.

## About

DoLocal is a software engineering project focused on practical marketplace design, API-driven architecture, access control, data modeling, and incremental delivery.

The showcase describes the project at a high level. It intentionally excludes private source code, credentials, personal data, and confidential implementation details.