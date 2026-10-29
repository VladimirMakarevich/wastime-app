# Backend Architecture

The Wastime backend is an **ASP.NET Core 9** REST API. It provides the parts of the product that
cannot live on a phone — publishing life calendars as web pages, the public calendar gallery,
print files of a life calendar, subscriptions, in-app messages, abuse reports, moderation and the
operator's inbox — on top of a security-first, fully automated Azure deployment.

> The mobile app is offline-first and does **not** need this backend for its personal features.
> The API is the additive cloud layer. See [architecture-mobile.md](architecture-mobile.md).

> This document describes the backend at the level of **layers, roles and patterns** rather
> than concrete project, class or file names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#backend-api).

---

## Design principles

| Principle | How it shows up |
| --- | --- |
| **Security by default** | Every endpoint requires authentication unless it is explicitly marked public; Managed Identity, Key Vault, private Blob Storage, OWASP response headers, rate limiting. No secrets in code. |
| **Layered, one-directional dependencies** | The HTTP surface, business logic and the ORM live in separate layers, and nothing lower reaches up. |
| **One use case, one handler** | Business logic is a set of small command and query handlers grouped in vertical slices, without a mediator library. |
| **Infrastructure as code** | The entire Azure footprint, monitoring included, is reproducible from Bicep definitions plus scripted deployment. |
| **Contract-first clients** | The API publishes an OpenAPI specification; every client is generated from it, never hand-written. |
| **Safe releases** | Production only receives a build that already passed on staging, and database migrations are additive. |

---

## Solution structure (layers)

The backend is split into focused projects with a strict, one-directional dependency flow
(each arrow = "depends on"):

```text
API layer                    HTTP: controllers · middleware · exception filter ·
  │                          auth · versioning · Swagger · hosted services
  ▼
Business logic               command & query handlers in vertical slices ·
  │                          area facades · wire DTOs · options · Blob Storage
  ▼
Data access                  EF Core DbContext · entity configurations ·
  │                          Identity persistence
  ├─▶ Data infrastructure    lower-level persistence building blocks
  └─▶ Entities               plain domain & Identity entities, grouped by area
              │
              ▼
        Foundation           typed exceptions · shared models

Supporting projects (outside the request path):
  Email service · Migrations runner · Seeding tool · Unit & integration test suites

Alongside the solution:
  Print renderer       Node service + Gotenberg, called by the API over HTTPS
```

| Layer | Responsibility |
| --- | --- |
| **API layer** | The composition root — controllers, the middleware pipeline, authentication and authorization, API versioning, Swagger generation, hosted services. All HTTP-specific code lives only here. |
| **Business logic** | One **command** (write) or **query** (read) handler per use case, each behind its own interface, grouped in vertical slices. Area facades bundle the handlers of one area for controllers. Knows nothing about HTTP. |
| **Data access** | The only layer with a database context; every other layer reaches the database through its interface. |
| **Entities** | Plain classes without service logic, grouped by area. |
| **Foundation** | Typed exceptions and shared models, with no business logic. |
| **Supporting** | Email delivery, a standalone migrations runner, an idempotent seeding tool and the test suites. |
| **Print renderer** | A separate Node service in the same repository that turns a print sheet into files; the API is its only caller. See [Print architecture](architecture-print.md#the-renderer-service). |

All projects target the same framework version, and moving it is a coordinated step across
every project, the ORM provider, Docker images and CI. REST is the only transport; reliability
comes from per-request idempotency guards.

---

## Request pipeline

The app uses the classic host + startup model. Configuration is layered from base settings →
environment-specific overrides → logging config → user secrets (development) → a local env file
→ environment variables. The middleware pipeline runs in this order:

```text
Request
  │
  ▼  Request decompression
  ▼  (Development) developer exception page · Swagger UI
  ▼  HTTPS redirection
  ▼  (Production) OWASP secure headers + security-header post-processing
  ▼  Correlation-id middleware        ← stamps a correlation id on every response
  ▼  Locale middleware                ← negotiates the request's locale once
  ▼  (conditional) request timing     ← latency metrics
  ▼  Routing
  ▼  Rate limiter                     ← sliding-window, per-IP policies
  ▼  Response caching
  ▼  CORS
  ▼  Authentication                   ← JWT bearer, Firebase as the authority
  ▼  Firebase validation middleware   ← verifies the token, adds identity & role claims
  ▼  Authorization                    ← role / permission policies
  ▼  Response compression
  ▼  Endpoints → controllers
  │
  ▼
Response
```

Errors are handled declaratively. A global **exception filter** turns typed exceptions into one
consistent error shape, and a refusal carries a stable **error target** that clients branch on
instead of parsing message text. Validation errors use the standard problem-details format, with
field names exactly as the client sent them, so a form can show the error on the right field.

---

## Authentication & authorization

Identity is anchored in **Firebase**. There is no first-party token: the API validates the
Firebase ID token directly and maps the caller onto ASP.NET Core Identity for roles.

```text
Client (mobile / web)
   │  Firebase ID token (Google / Apple / email)
   ▼
JWT bearer authentication   (Firebase as the token authority)
   ▼
Firebase validation middleware ──uses──▶ FirebaseAdmin SDK
   │  token verified → user id + role and permission claims from the database
   ▼
Authorization  (role and permission policies)
   ▼
Controller action
```

| Mechanism | Role |
| --- | --- |
| **JWT bearer + FirebaseAdmin** | Two-step verification of Firebase-issued ID tokens, without a shared secret |
| **Role and permission claims** | Added only by the validation middleware; claims with the same names coming from the token are ignored |
| **Fallback policy** | Every endpoint requires an authenticated user unless it is explicitly marked public |
| **Policies** | Role policies (Administrator, Moderator, Owner) and permission policies (block or manage users, analytics, system management) |
| **Operator namespace** | All operator endpoints live under one admin route prefix, restricted to administrators; a test walks every route under it and fails on any that is not. The admin portal is their only client |

The **same Firebase identity** works across the mobile app, the website and the API.

---

## API surface

The API is organized into capability areas, versioned under `/api/v{version}`:

| Capability area | Purpose |
| --- | --- |
| **Calendars** | Publish a life calendar (quota-checked, idempotent per device), read the active and past publications, delete; anonymous reads by access link, a paged public gallery ranked by locale, and related calendars |
| **Publish allowance** | Remaining free publications, the entitlement state and when the allowance resets |
| **Print** | The print allowance, generating a set of print files, reading a set, short-lived file links, the account's history of sets and deleting a set |
| **Favorites** | Saving public calendars to favorites |
| **Reports** | Anonymous abuse reports against calendars (with an optional attachment), and client error reports from the mobile app |
| **Notifications** | A pull-based feed of in-app messages, localized, with caching headers; operators draft, edit, publish, withdraw and delete messages |
| **Moderation** | The operator's queue, calendar cards with their reports, decisions on calendars, opening attachments, blocking authors |
| **Inbox** | A read-only view for the operator of feedback, account deletions and error reports from the app |
| **Accounts & profile** | User accounts, profile data and account deletion |
| **Subscriptions** | The signed RevenueCat webhook (not part of the public OpenAPI document) |
| **Statistics** | Aggregated statistics |
| **Features & feedback** | Feature voting and user feedback intake |
| **Role management** | Role administration |
| **Health & diagnostics** | Liveness checks and key validation |

JSON uses camelCase properties and enums as strings. Length limits are declared on the request
DTOs, so they show up in the OpenAPI document and clients check them before sending.

---

## Print files

The API owns the server half of the website's print constructor: it decides whether a set may be
generated, renders it through the print renderer and keeps it for a short time.

```text
Website ──▶ Print API ──▶ limits (account / visitor / address) ──▶ render slot
                │                                                    │
                │                                     Print renderer (Container App)
                │                                     Node service + Gotenberg
                ▼                                                    │
        dedicated Blob Storage  ◀──── PDF · page pictures · guide ───┘
                │
                ▼
     short-lived read-only HTTPS links ──▶ browser download
```

- **Limits from configuration**: 10 calendar files in any rolling 30 days per account, 3 empty
  grids a day per visitor and per address, 5 generations an hour per person, and at most 4 renders
  at once per instance with a short wait for a free slot.
- **Retention**: a calendar's set is kept for 15 days, an empty grid's for one day, in a storage
  account used only for print. Deleting a set removes its files but keeps a nameless record for the
  limit; deleting the account deletes its sets.
- **Privacy**: client addresses are stored only as an HMAC, visitor identities are cleared by a
  hosted service after two days, and neither the sheet nor its text reaches the logs.
- **Version lock**: the renderer accepts only its own version of the sheet generator, so a website
  running an older generator gets a refusal that asks for a page reload.

The full design is in [Print architecture](architecture-print.md).

---

## Calendar publishing

A published calendar is a **snapshot** of the person's life calendar, taken at the moment of
publishing and served to the website.

- **Access levels.** *Public* calendars are listed in the gallery and open by link; *link*
  calendars are unlisted and open only by link; *private* calendars get no link at all.
- **Links.** A share link combines a URL-safe random token with a slug made from the calendar's
  name.
- **Gallery.** The public gallery is paged and ranked by how well a calendar's author locale and
  allowed languages match the visitor.
- **Idempotency.** A publish request is guarded per device, so a retried request never creates a
  second publication, and a conflict object comes back when two devices disagree.
- **Showcase.** A seeding job publishes a small set of hand-written public calendars under a
  service account, so the gallery is never empty. It runs only through a dedicated workflow.

---

## Subscriptions

**Wastime Pro** removes the free publishing limit; it is the only paid capability.

```text
RevenueCat ──signed webhook──▶ API ──re-read customer──▶ RevenueCat REST API
                                │
                                ▼
                  entitlement expiry stored on the user
                                │
                                ▼
                 publish command checks quota & entitlement
```

- **The webhook is only a signal.** Its HMAC signature is verified over the raw body within a
  short time window, it has its own rate limit, and it is excluded from the OpenAPI document. The
  handler ignores the event type: for every customer id in the payload it re-reads the current
  state from RevenueCat's REST API and stores the entitlement expiry. This makes processing
  idempotent and independent of the order in which events arrive. If RevenueCat is unreachable,
  the API answers with an error and RevenueCat retries.
- **The quota lives in the publish command.** Free users get three publications in any rolling
  30 days, counted from an append-only publications log written in the same transaction as the
  publication. Before refusing, the command re-checks RevenueCat once, so a purchase made a moment
  ago is honored even if the webhook has not arrived yet.
- **Clients never decide.** The mobile app only displays the allowance returned by the API; a
  refusal comes back with a stable error target the app turns into a clear message.
- Deleting an account also deletes the matching RevenueCat customer.

---

## Moderation & risk scoring

Public calendars are user content, so the API protects visitors from abuse.

- Anyone viewing a calendar can **report** it under one of several categories (scam, phishing,
  hate, doxxing, spam and others), optionally attaching a file that is stored in private Blob
  Storage.
- Each report adds a **category-weighted score** that decays over time, and a calendar moves
  through risk states from *safe* to *blocked*. Calendars in the two highest states, and calendars
  of blocked users, are hidden from the gallery and from link access.
- A **background service** periodically recalculates every calendar's score and state from its
  report history, with an optional distributed lock.
- One visibility rule applies to every public read — the gallery, a link, related calendars — so a
  hidden calendar cannot leak through a side door.
- The **admin portal** gives the operator the moderation queue: calendar cards with their reports
  and attachments, a decision on each calendar, and blocking an author without touching the risk
  of their calendars. Attachments open only through HTTPS links the API issues.
- Deleting an account anonymizes the reports, feedback and error reports it left behind.

---

## In-app messages

Messages from the Wastime team reach the app through a **pull-based feed** — there is no push
notification channel. The feed supports cursors, acknowledged ids and a server hint for the next
poll; message details use ETags so unchanged content costs a `304`. Message text is stored per
locale with an English fallback.

Messages have a status. The operator writes a draft in the admin portal, publishes it, and can
withdraw or delete it later; the app's feed serves only published messages.

---

## Localization

A locale middleware negotiates the request's locale once, from the standard language headers and
a few client hints, defaulting to `en-US`. Today it drives two things — the language of in-app
messages and the locale ranking of the public gallery. API and error messages are in English.

---

## Data layer

| Concern | Technology |
| --- | --- |
| **ORM** | Entity Framework Core 9 |
| **Database** | PostgreSQL — Azure Database for PostgreSQL, Flexible Server |
| **Provider** | Npgsql.EntityFrameworkCore.PostgreSQL |
| **Schema conventions** | snake_case tables and columns, one mapping configuration per entity, GUID keys, UTC timestamps |
| **Identity** | ASP.NET Core Identity with EF Core stores |
| **Migrations** | Applied by a dedicated runner, before the new code is deployed; additive by rule |
| **Seeding** | An idempotent tool; developer seeders refuse to run against a non-local database |

Handlers work through the data layer's interface rather than touching EF Core from controllers,
which keeps the HTTP layer thin.

---

## Synchronization (designed, currently dormant)

The server half of the offline-first sync described in the
[mobile architecture](architecture-mobile.md#offline-first--synchronization) is designed as a
client-authoritative engine — per-entity state with last-writer-wins merging, a per-user change
sequence and idempotency keys. Like the client engine it is **switched off**: its endpoints,
services and maintenance job are disabled until multi-device sync is ready to ship.

---

## File storage

Binary assets are stored in **Azure Blob Storage**. Storage is **private and encrypted** — access
is brokered by the API and never exposed publicly. Report attachments and print files live in
separate storage accounts: print files are short-lived by design, so their account has no soft
delete, and files are handed out only through short-lived read-only HTTPS links.

---

## Cloud infrastructure (Azure)

The entire footprint is **infrastructure-as-code** in Bicep and deployed with a scripted command.

```text
                       ┌───────────────────────┐       ┌─────────────────────┐
   Clients  ───────▶   │  ASP.NET Core API     │ ────▶ │  Print renderer     │
                       │  App Service          │       │  Container Apps     │
                       │  (Docker / Linux)     │       │  Node + Gotenberg   │
                       └───┬───────┬───────┬───┘       └──────────▲──────────┘
                           │       │       │                      │ images
                           │       │       │           ┌──────────┴──────────┐
                           │       │       │           │  Container Registry │
                           │       │       │           └─────────────────────┘
             ┌─────────────┘       │       └──────────────┐
             ▼                     ▼                      ▼
   ┌───────────────────┐  ┌────────────────┐   ┌─────────────────────┐
   │ PostgreSQL        │  │ Blob Storage   │   │ Application Insights│
   │ Flexible Server   │  │ (private;      │   │ + Log Analytics     │
   │                   │  │  print apart)  │   │                     │
   └───────────────────┘  └────────────────┘   └─────────────────────┘
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
| **App Service** | Hosts the API (Docker, Linux) |
| **Azure Database for PostgreSQL (Flexible Server)** | Primary datastore |
| **Azure Blob Storage** | Private encrypted file storage; a separate account for print files |
| **Azure Container Apps** | The print renderer — consumption plan, scales to zero, reachable only from the API |
| **Azure Container Registry** | The renderer's images, pulled with a managed identity |
| **Azure Key Vault** | Secrets and auto-generated credentials, accessed via Managed Identity and authorized by RBAC role assignments |
| **Application Insights + Log Analytics** | Telemetry, tracing, metrics, log sink |
| **Monitoring** | An availability test, an action group and metric alerts, defined in the same IaC |
| **Security logs** | App Service HTTP, audit and platform logs, PostgreSQL logs and Key Vault audit events go to Log Analytics; password-based publishing credentials (deployment console and FTP) are refused |
| **Budget** | A monthly budget with alerts on the print renderer's resources |

### Infrastructure automation

- **IaC.** Bicep definitions describe the full footprint, monitoring included. Infrastructure
  changes are applied deliberately, ahead of the code that needs them, always starting with a
  `what-if` preview.
- **Scripted operations.** PowerShell scripts cover infrastructure deployment, service-principal
  creation, secret provisioning (app config, Firebase, Application Insights, database connection
  string), database password rotation and teardown.
- Passwords are **generated automatically** (24 characters, cryptographically strong) and stored
  **only in Key Vault**, so they never appear in code or config.
- **Key Vault access is granted by RBAC role assignments** rather than access policies, so applying
  the templates no longer empties the vault's permissions for a moment.

---

## Observability

| Layer | Mechanism |
| --- | --- |
| **Logging** | Serilog (structured), configured from files and environment overrides; the product's own code logs at Information level, frameworks at Warning; no personal data, user content, file names or free-text reasons in logs |
| **Telemetry** | Application Insights |
| **Log shipping** | A Serilog sink forwards logs to Azure |
| **Correlation** | Middleware stamps a correlation id on every response |
| **Timing** | Middleware records request latency |

---

## API contract & client generation

The API is the **single source of truth** for its clients:

```text
ASP.NET Core controllers
   │  Swashbuckle  (exports the specifications on build)
   ▼
OpenAPI documents   (committed with every contract change)
   ├─ mobile  → the mobile app
   ├─ web     → the website
   └─ admin   → the admin portal
   │
   ▼
OpenAPI generator (typescript-angular)
   ▼
a generated Angular API client in each project
```

Each client gets a document with only the routes it uses, so an operator route never appears in
the app's client and a contract change touches only the clients it concerns.

Because clients are generated, a breaking API change surfaces as a **compile error** in the
front-end rather than a runtime surprise. Removing a field or route happens only after every
client has stopped using it.

---

## CI/CD & release flow

```text
Pull request ──▶ build · test · publish  (+ renderer tests against a real Gotenberg)
      │
      ▼  merge to main
Staging deploy (automatic) ──▶ migrations ──▶ App Service ──▶ health check ──▶ smoke test
                           └─▶ renderer images by commit ──▶ Container App revision
      │
      ▼  read-only release preflight · manual workflow run with explicit confirmation, same commit
Production deploy ──▶ migrations ──▶ App Service ──▶ health check ──▶ smoke test
                  └─▶ renderer images imported by digest from staging, no rebuild
```

- **GitHub Actions** builds, tests and publishes on every pull request and push. Secrets come
  from Key Vault, and the pipeline checks the target database host before running migrations.
- **Staging first.** Every push to the main branch deploys staging. Production deploys only from
  a manually started run, and only a commit that is already live on staging and passed its smoke
  test. Before promoting, a read-only preflight script lists what the release would ship and what
  has to happen first — non-additive migrations, removed or tightened contract, production
  settings, infrastructure templates and renderer changes — and stops on anything unsafe.
- **Additive migrations.** Migrations run before the new code and are never rolled back, so they
  must work with the previous version. Removing or renaming a column takes two releases.
- **A green health check is not the end.** A release counts as done only after a production smoke
  test confirms the new behavior and telemetry shows no unexplained server errors.
- **Rollback** means running the same workflow on the previous production commit.
- **Docker** containerizes the API for consistent local and cloud runs; a separate workflow lints
  the repository's Markdown documentation.

---

## Testing

| Suite | Tooling | Scope |
| --- | --- | --- |
| **Unit** | xUnit, Moq, MockQueryable, EF Core InMemory | Over 700 tests — handlers, quota and entitlement logic, print limits, risk scoring, validators |
| **Integration** | xUnit, `WebApplicationFactory`, EF Core InMemory, FluentAssertions | Around 170 tests — the HTTP pipeline end to end, print endpoints with a stubbed renderer, and a check that every operator route is restricted to administrators |
| **Print renderer** | Node's built-in test runner, a real Gotenberg in CI | About 50 tests — request checks, page pictures, the guide PDF, embedded fonts |

Paths that depend on PostgreSQL-specific behavior (concurrency tokens, jsonb, raw SQL) are not
trusted to the in-memory provider and need a real PostgreSQL instance.

---

## Security summary

- 🔐 **No secrets in code** — Key Vault + Managed Identity.
- 🔑 **Auto-generated credentials** — 24 characters, cryptographically strong, stored only in Key
  Vault.
- 🛡️ **OWASP security headers** — via `OwaspHeaders.Core` plus post-processing middleware; HSTS in
  production.
- 👤 **Firebase-verified identities** — server-side token validation on every authenticated request.
- 🚪 **Authenticated by default** — public endpoints are opted out explicitly; operator routes are
  administrator-only and checked by a test.
- ✍️ **Signed webhooks** — HMAC verification over the raw body with a timestamp window.
- 🔒 **Private encrypted Blob Storage** — no public object access; files leave only through
  short-lived read-only links.
- 🖨️ **An isolated renderer** — reachable only from the API with a key; Chromium runs without
  scripts and may open only the files of the request.
- 📜 **Audit logs** — App Service, PostgreSQL and Key Vault audit events are kept in Log Analytics,
  publishing by password is refused, and Key Vault access goes through RBAC.
- 🚦 **Rate limiting + CORS** — sliding-window per-IP policies, enforced in the pipeline.
- 🧾 **Correlation + structured logs** — traceable requests, without personal data.
