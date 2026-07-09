# Backend Architecture

The Wastime backend is an **ASP.NET Core 9.0** REST API. It provides **optional** cloud
capabilities — accounts, multi-device sync, file storage, notifications, reporting — on top of
a security-first, fully automated Azure deployment.

> The mobile app is offline-first and does **not** require this backend to function. The API
> is the additive cloud layer. See [architecture-mobile.md](architecture-mobile.md).

> This document describes the backend at the level of **layers, roles, and patterns** rather
> than concrete project, class, or file names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#backend-api).

---

## Design principles

| Principle | How it shows up |
| --- | --- |
| **Security by default** | Managed Identity, Key Vault with auto-generated secrets, private Blob Storage, OWASP response headers, Firebase token validation. No secrets in code. |
| **Layered / clean dependencies** | A one-directional layer graph isolates the HTTP surface from business logic and from the ORM. |
| **Infrastructure as code** | The entire Azure footprint is reproducible from Bicep / Terraform definitions plus scripted deployment. |
| **Observable** | Structured logging and telemetry with correlation IDs and request timing. |
| **Contract-first clients** | The API publishes an OpenAPI specification; clients are generated from it, never hand-written. |

---

## Solution structure (layers)

The backend is split into focused projects with a strict, one-directional dependency flow
(each arrow = "depends on"):

```text
API layer                    HTTP: controllers · middleware · filters · auth ·
  │                          versioning · Swagger · hosted services
  ▼
Business logic               application services · DTOs · response models ·
  │                          Blob Storage integration
  ▼
Data access                  EF Core DbContext · repositories · Identity persistence
  │
  ├─▶ Persistence infrastructure     lower-level persistence concerns
  └─▶ Entities                       domain & Identity entities
              │
              ▼
        Base domain          shared EF Core primitives

Supporting projects (outside the request path):
  Email service · Migrations runner · Seeding tool · Unit & integration test suites
```

| Layer | Responsibility |
| --- | --- |
| **API layer** | Controllers, the middleware pipeline, authentication, API versioning, Swagger generation, hosted services |
| **Business logic** | Application/domain services, DTOs, response models, Azure Blob Storage integration |
| **Data access** | `DbContext`, repositories, Identity persistence (via the PostgreSQL provider) |
| **Persistence infrastructure** | Lower-level persistence building blocks |
| **Entities** | Domain entities and ASP.NET Core Identity entities |
| **Base domain** | Base types and EF Core primitives shared across layers |
| **Supporting** | Email delivery, a standalone migrations runner, a seeding tool, and test suites |

---

## Request pipeline

The app uses the classic host + startup hosting model. Configuration is layered from base
settings → environment-specific overrides → logging config → user secrets (development) →
a local env file → environment variables. The middleware pipeline runs in this order:

```text
Request
  │
  ▼  Request decompression
  ▼  (Development) developer exception page · Swagger UI
  ▼  HTTPS redirection
  ▼  (Production) OWASP secure headers + security-header post-processing
  ▼  Correlation-id middleware        ← stamps a correlation id on every response
  ▼  Locale / culture middleware      ← per-request culture / i18n
  ▼  (conditional) request-timing     ← latency metrics
  ▼  Routing
  ▼  Rate limiter                     ← configurable rate limiting
  ▼  Response caching
  ▼  CORS
  ▼  Authentication                   ← JWT bearer
  ▼  Firebase token validation        ← verifies the caller's Firebase identity
  ▼  Authorization
  ▼  Response compression
  ▼  Endpoints → controllers
  │
  ▼
Response
```

Global concerns are handled declaratively:

- A global **exception filter** converts unhandled exceptions into consistent error responses.
- Strongly-typed **options** (rate limiting, security headers) and application-info providers
  are bound and wired up at startup.

---

## Authentication & authorization

Identity is anchored in **Firebase**, verified server-side, and mapped onto ASP.NET Core
Identity for roles and authorization:

```text
Client (mobile / web)
   │  Firebase ID token (Google / Apple / Email / …)
   ▼
Authentication (JWT bearer)
   ▼
Firebase validation middleware ──uses──▶ token validator (FirebaseAdmin SDK)
   │  token valid → identity established
   ▼
Authorization  (roles via ASP.NET Core Identity)
   ▼
Controller action
```

| Component | Role |
| --- | --- |
| **FirebaseAdmin** | Verifies Firebase-issued ID tokens without a shared secret |
| **Token validator** | Wraps the validation logic |
| **Validation middleware** | Enforces validation in the request pipeline |
| **JWT bearer** | ASP.NET Core authentication scheme |
| **ASP.NET Core Identity** | User store and role management |

The **same Firebase identity** authenticates a user across the mobile app, the web site, and
the API — a single sign-in surface.

---

## API surface

The API is organized into capability areas rather than a flat list of endpoints:

| Capability area | Purpose |
| --- | --- |
| **Accounts & profile** | User accounts and profile data |
| **Calendars** | Life calendars, milestones, published calendars |
| **Synchronization** | Multi-device data sync (push/pull of change batches) |
| **Notifications** | Push / scheduled notification management |
| **Statistics** | Aggregated statistics |
| **Reports** | Report generation |
| **Favorites** | User favorites |
| **Feature gating** | Feature flags / capability checks |
| **Feedback** | User feedback intake |
| **Role management** | Role administration |
| **Health & diagnostics** | Liveness/readiness checks and key/secret validation utilities |

API **versioning** keeps the contract stable as it evolves.

---

## Data layer

| Concern | Technology |
| --- | --- |
| **ORM** | Entity Framework Core 9 |
| **Database** | PostgreSQL — Azure Database for PostgreSQL, Flexible Server |
| **Provider** | Npgsql.EntityFrameworkCore.PostgreSQL |
| **Identity** | ASP.NET Core Identity with EF Core stores |
| **Migrations** | A dedicated runner project applies schema changes |
| **Seeding** | A separate tool seeds reference data |

Business services operate on repositories rather than touching EF Core directly from
controllers, keeping the HTTP layer thin.

---

## Synchronization (server side)

The backend implements the **server half of the offline-first sync** described in the
[mobile architecture](architecture-mobile.md#offline-first--synchronization):

| Piece | Role |
| --- | --- |
| **Sync endpoints** | Push/pull of change batches |
| **Sync services & contracts** | Sync business logic and transfer objects |
| **Background maintenance** | A hosted service maintains sync state/data |
| **Entity versioning** | Server-side version checks enable conflict detection, mirroring the client's per-row versions |

This matches the client's outbox/versioned design — the client engine is scaffolded and
consent-gated, and turns on once the server contract is finalized.

---

## Background services

Long-running work runs as **hosted services**, off the request path:

| Background worker | Purpose |
| --- | --- |
| **Sync maintenance** | Periodic housekeeping of sync data/state |
| **Risk backfill** | Backfills / derives risk-related data |

---

## File storage

Binary assets (e.g. images) are stored in **Azure Blob Storage** (via `Azure.Storage.Blobs`).
Storage is **private and encrypted** — access is brokered by the API, never exposed publicly.

---

## Cloud infrastructure (Azure)

The entire footprint is **infrastructure-as-code** and deployed with a single scripted command.
Both **Bicep** and **Terraform** definitions are maintained.

```text
                       ┌──────────────────────┐
   Clients  ───────▶   │  ASP.NET Core API    │   (Docker / Linux)
                       │  App Service /        │
                       │  Container            │
                       └───┬───────┬───────┬───┘
                           │       │       │
             ┌─────────────┘       │       └──────────────┐
             ▼                     ▼                      ▼
   ┌───────────────────┐  ┌────────────────┐   ┌────────────────────┐
   │ PostgreSQL        │  │ Blob Storage   │   │ Application Insights│
   │ Flexible Server   │  │ (private)      │   │ (telemetry/logs)    │
   └───────────────────┘  └────────────────┘   └────────────────────┘
             ▲                     ▲
             └───────── secrets ───┴──────────┐
                                              ▼
                                   ┌────────────────────┐
                                   │ Azure Key Vault    │
                                   │ + Managed Identity │
                                   └────────────────────┘
```

| Service | Role |
| --- | --- |
| **App Service / Container** | Hosts the API (Docker, Linux target) |
| **Azure Database for PostgreSQL (Flexible Server)** | Primary datastore |
| **Azure Blob Storage** | Private encrypted file storage |
| **Azure Key Vault** | Secrets + auto-generated credentials, accessed via Managed Identity |
| **Application Insights** | Telemetry, tracing, metrics, log sink |

### Infrastructure automation

- **IaC:** Bicep and Terraform definitions describe the full footprint.
- **Scripted operations:** a set of PowerShell scripts cover one-command infrastructure
  deployment, service-principal creation, secret provisioning (app config, Firebase,
  Application Insights, database connection string), database password rotation, and teardown.
- Passwords are **generated automatically** (24-char, cryptographically strong) and stored
  **only in Key Vault** ("zero-knowledge" — they never appear in code or config).

---

## Observability

| Layer | Mechanism |
| --- | --- |
| **Logging** | Serilog (structured), configured from files + environment overrides |
| **Telemetry** | Application Insights |
| **Log shipping** | A Serilog sink forwards logs to Azure |
| **Correlation** | Middleware stamps a correlation id per request |
| **Timing** | Middleware records request latency |

---

## API contract & client generation

The API is the **single source of truth** for its clients:

```text
ASP.NET Core controllers
   │  Swashbuckle  (emits the specification at build time)
   ▼
OpenAPI specification
   │  distributed to the app & web projects
   ▼
OpenAPI generator (typescript-angular)
   ▼
generated Angular API client  →  consumed by mobile app & web site
```

Because clients are generated, any breaking API change surfaces as a **compile error** in the
front-end, not a runtime surprise.

---

## CI/CD

- **GitHub Actions** builds, tests, and deploys the API.
- **Docker** containerizes the API for consistent local and cloud runs.
- Unit and integration test suites run in the pipeline.

---

## Security summary

- 🔐 **No secrets in code** — Key Vault + Managed Identity.
- 🔑 **Auto-generated credentials** — 24-char, cryptographically strong, Key-Vault-only.
- 🛡️ **OWASP security headers** — via `OwaspHeaders.Core` + post-processing middleware.
- 👤 **Firebase-verified identities** — server-side token validation on every authenticated request.
- 🔒 **Private encrypted Blob Storage** — no public object access.
- 🚦 **Rate limiting + CORS** — configurable, enforced in the pipeline.
- 🧾 **Correlation + structured logs** — traceable, auditable requests.
