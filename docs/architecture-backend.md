# Backend Architecture

The Wastime backend is an **ASP.NET Core 9.0** REST API (internally *Ti.Waste API*). It
provides **optional** cloud capabilities — accounts, multi-device sync, file storage,
notifications, reporting — on top of a security-first, fully automated Azure deployment.

> The mobile app is offline-first and does **not** require this backend to function. The API
> is the additive cloud layer. See [architecture-mobile.md](architecture-mobile.md).

For versions and package details, see [tech-stack.md](tech-stack.md#backend-api).

---

## Design principles

| Principle | How it shows up |
| --- | --- |
| **Security by default** | Managed Identity, Key Vault with auto-generated 24-char secrets, private Blob Storage, OWASP response headers, Firebase token validation. No secrets in code. |
| **Layered / clean dependencies** | A one-directional project graph isolates the HTTP layer from business logic and EF Core. |
| **Infrastructure as code** | The entire Azure footprint is reproducible from Bicep / Terraform + PowerShell scripts. |
| **Observable** | Structured logging (Serilog) and telemetry (Application Insights) with correlation IDs and request timing. |
| **Contract-first clients** | The API publishes an OpenAPI spec; clients are generated from it, never hand-written. |

---

## Solution structure (layers)

The solution (`Ti.Waste.sln`) is split into focused projects with a strict, one-directional
dependency flow (each arrow = "depends on"):

```text
Ti.Waste.Api                     HTTP layer
  controllers · middleware · filters · auth · versioning · Swagger
        │
        ▼
Ti.Waste.Core                    Business logic
  Services/ (Users, Calendar, Sync, Notifications, Reports, Features,
             Feedback, Authorization, Errors) · BlobStorage · DTOs
        │
        ▼
Ti.Waste.Data                    Data access
  EF Core DbContext · repositories · ASP.NET Core Identity stores
        │
        ├─▶ Ti.Waste.Data.Infrastructure   persistence infrastructure
        └─▶ Ti.Waste.Entities              domain & Identity entities
                    │
                    ▼
              Ti.Waste                     base domain / EF Core primitives

Supporting projects (outside the request path):
  Ti.Waste.Email        email composition / sending
  Ti.Waste.Migrations   EF Core migrations runner (has its own appsettings)
  Ti.Waste.SeedingApp   database seeding
  *.UnitTests / *.IntegrationTests
```

| Project | Responsibility |
| --- | --- |
| **Ti.Waste.Api** | Controllers, middleware pipeline, authentication, API versioning, Swagger generation, hosted services |
| **Ti.Waste.Core** | Application/business services, DTOs, response models, Azure Blob Storage integration |
| **Ti.Waste.Data** | `DbContext`, repositories, Identity persistence (Npgsql) |
| **Ti.Waste.Data.Infrastructure** | Lower-level persistence infrastructure |
| **Ti.Waste.Entities** | Domain entities and ASP.NET Core Identity entities |
| **Ti.Waste** | Base domain types and EF Core primitives shared by the layers |
| **Ti.Waste.Email** | Email templates and delivery |
| **Ti.Waste.Migrations** | Applies EF Core migrations as a standalone runner |
| **Ti.Waste.SeedingApp** | Seeds reference/initial data |

---

## Request pipeline

The app uses the classic `Program` + `Startup` hosting model. Configuration is layered from
`appsettings.json` → environment-specific JSON → `serilog.json` → User Secrets (dev) →
`.env` → environment variables. The middleware pipeline (from `Startup.Configure`) is:

```text
Request
  │
  ▼  UseRequestDecompression
  ▼  (Development) DeveloperExceptionPage · Swagger · Swagger UI
  ▼  UseHttpsRedirection
  ▼  (Production) OWASP SecureHeaders + SecurityHeadersPostProcessingMiddleware
  ▼  ResponseHeaderCorrelationMiddleware   ← correlation id on every response
  ▼  LocaleMiddleware                       ← per-request culture / i18n
  ▼  (conditional) RequestTimingMiddleware  ← latency metrics
  ▼  UseRouting
  ▼  UseRateLimiter                         ← configurable rate limiting
  ▼  UseResponseCaching
  ▼  UseCors
  ▼  UseAuthentication                      ← JWT bearer
  ▼  FirebaseValidationMiddleware           ← verifies Firebase identity token
  ▼  UseAuthorization
  ▼  UseResponseCompression
  ▼  UseEndpoints → MapControllers
  │
  ▼
Response
```

Global concerns are handled declaratively:

- **`ExceptionFilter`** (in `Filters/`) — converts unhandled exceptions into consistent error
  responses.
- **`Applications/`** — strongly-typed options (`RateLimitingOptions`, `SecurityHeadersOptions`)
  and application info providers, wired up at startup.

---

## Authentication & authorization

Identity is anchored in **Firebase**, verified server-side, and mapped onto ASP.NET Core
Identity for roles/authorization:

```text
Client (mobile / web)
   │  Firebase ID token (Google / Apple / Email / …)
   ▼
UseAuthentication (JwtBearer)
   ▼
FirebaseValidationMiddleware ──uses──▶ FirebaseTokenValidator  (FirebaseAdmin SDK)
   │  token valid → identity established
   ▼
UseAuthorization  (roles via ASP.NET Core Identity)
   ▼
Controller action
```

| Component | Role |
| --- | --- |
| **FirebaseAdmin** | Verifies Firebase-issued ID tokens without a shared secret |
| **`FirebaseTokenValidator`** (`Authorization/`) | Wraps validation logic |
| **`FirebaseValidationMiddleware`** | Enforces validation in the pipeline |
| **JwtBearer** | ASP.NET Core authentication scheme |
| **ASP.NET Core Identity** | User store and role management (`RoleManagementController`) |

This means the **same Firebase identity** authenticates a user across the mobile app, the web
site, and the API — a single sign-in surface.

---

## API surface (controllers)

| Controller | Purpose |
| --- | --- |
| `UserController` | Accounts, profile |
| `CalendarController` | Life calendars, milestones, published calendars |
| `SyncController`, `EntitySyncController` | Multi-device data synchronization endpoints |
| `NotificationsController` | Push/scheduled notification management |
| `StatsController` | Aggregated statistics |
| `ReportsController` | Reports generation |
| `FavoritesController` | User favorites |
| `FeaturesController` | Feature flags / capability gating |
| `FeedbackController` | User feedback intake |
| `RoleManagementController` | Role administration |
| `HealthController` | Liveness/readiness health checks |
| `KeyCheckController` | Key/secret validation utility |

API **versioning** (`Microsoft.AspNetCore.Mvc.Versioning`) keeps the contract stable as it
evolves.

---

## Data layer

| Concern | Technology |
| --- | --- |
| **ORM** | Entity Framework Core 9 |
| **Database** | PostgreSQL — Azure Database for PostgreSQL, Flexible Server |
| **Provider** | Npgsql.EntityFrameworkCore.PostgreSQL |
| **Identity** | ASP.NET Core Identity with EF Core stores |
| **Migrations** | `Ti.Waste.Migrations` — a dedicated runner project applies schema changes |
| **Seeding** | `Ti.Waste.SeedingApp` — seeds reference data |

Business services in `Ti.Waste.Core/Services/` (Users, Calendar, Sync, Notifications, Reports,
Features, Feedback, Authorization) operate on repositories rather than touching EF Core
directly from controllers, keeping the HTTP layer thin.

---

## Synchronization (server side)

The backend implements the **server half of the offline-first sync** described in the
[mobile architecture](architecture-mobile.md#offline-first--synchronization):

| Piece | Role |
| --- | --- |
| `SyncController` / `EntitySyncController` | Push/pull endpoints for change batches |
| `Core/Services/Sync` + `Core/Dtos/Sync` | Sync business logic and transfer contracts |
| `EntitySyncMaintenanceHostedService` | Background maintenance of sync state/data |
| Entity versioning | Server-side version checks enable conflict detection (mirrors the client's `Versioned` rows) |

This matches the client's outbox/versioned design — the client engine is scaffolded and
consent-gated, and turns on once the server contract is finalized.

---

## Background services

Long-running work runs as **hosted services**, off the request path:

| Hosted service | Purpose |
| --- | --- |
| `EntitySyncMaintenanceHostedService` | Periodic sync-data maintenance/housekeeping |
| `RiskBackfillHostedService` | Backfills/derives risk-related data |

---

## File storage

Binary assets (e.g. images) are stored in **Azure Blob Storage** via
`Ti.Waste.Core/BlobStorage` (`Azure.Storage.Blobs`). Storage is **private and encrypted** —
access is brokered by the API, never exposed publicly.

---

## Cloud infrastructure (Azure)

The entire footprint is **infrastructure-as-code** and deployed with a single command
(`deploy-infrastructure.ps1`). Both **Bicep** and **Terraform** definitions are maintained
under `src/infra/`.

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

### Infrastructure automation (`src/infra/`)

- **IaC:** `bicep/` and `terraform/` definitions.
- **PowerShell scripts:** `deploy-infrastructure.ps1`, `create-service-principal.ps1`,
  `add-app-config-secrets.ps1`, `add-firebase-secret-azcli.ps1`,
  `create-appinsights-secrets.ps1`, `create-db-connection-string.ps1`,
  `rotate-db-password.ps1`, `cleanup.ps1`.
- Passwords are **generated automatically** (24-char, cryptographically strong) and stored
  **only in Key Vault** ("zero-knowledge" — they never appear in code or config).

---

## Observability

| Layer | Mechanism |
| --- | --- |
| **Logging** | Serilog (structured), configured from `serilog.json` + env overrides |
| **Telemetry** | Application Insights (`Microsoft.ApplicationInsights.AspNetCore`) |
| **Log shipping** | `Serilog.Sinks.ApplicationInsights` forwards logs to Azure |
| **Correlation** | `ResponseHeaderCorrelationMiddleware` stamps a correlation id per request |
| **Timing** | `RequestTimingMiddleware` records latency |

---

## API contract & client generation

The API is the **single source of truth** for its clients:

```text
ASP.NET Core controllers
   │  Swashbuckle.AspNetCore  (build target emits the spec in Debug)
   ▼
OpenApiSpec/swagger.json
   │  distributed to the app & web repos as openapi/swagger.json
   ▼
openapi-generator-cli (typescript-angular)
   ▼
generated Angular API client  →  consumed by mobile app & web site
```

Because clients are generated, any breaking API change surfaces as a **compile error** in the
front-end, not a runtime surprise.

---

## CI/CD

- **GitHub Actions** (`.github/`) builds, tests, and deploys the API.
- **Docker** (`docker-compose.yml`) containerizes the API for consistent local and cloud runs.
- Integration and unit test projects (`Ti.Waste.Api.IntegrationTests`,
  `TI.Waste.Core.UnitTests`) run in the pipeline.

---

## Security summary

- 🔐 **No secrets in code** — Key Vault + Managed Identity.
- 🔑 **Auto-generated credentials** — 24-char, cryptographically strong, Key-Vault-only.
- 🛡️ **OWASP security headers** — via `OwaspHeaders.Core` + post-processing middleware.
- 👤 **Firebase-verified identities** — server-side token validation on every authenticated request.
- 🔒 **Private encrypted Blob Storage** — no public object access.
- 🚦 **Rate limiting + CORS** — configurable, enforced in the pipeline.
- 🧾 **Correlation + structured logs** — traceable, auditable requests.
