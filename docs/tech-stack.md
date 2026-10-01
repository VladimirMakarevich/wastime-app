# Technology Stack

This document is the full, versioned breakdown of the technologies behind **Wastime**, with the
reasoning behind the key choices. The three architecture documents build on it:

- [Mobile app architecture](architecture-mobile.md)
- [Backend architecture](architecture-backend.md)
- [Website architecture](architecture-web.md)

Wastime is delivered as **three cooperating applications** that share a single, generated API
contract, with an operator portal in development:

| Application | Role | Core technology |
| --- | --- | --- |
| **Mobile app** | The product itself (iOS &amp; Android), fully offline | Ionic + Angular + Capacitor |
| **Backend API** | Calendar publishing, the public gallery, subscriptions, in-app messages, moderation | ASP.NET Core 9 |
| **Website** | Landing, feature pages, blog, published calendars, account area | Angular 21 with SSR / prerendering |
| **Admin portal** *(in development)* | Moderation and in-app messages for the operator | Angular 22 + Angular Material |

> Versions below reflect the projects' current manifests. Minor and patch versions move over
> time; the major versions and the architectural choices are the stable part of this document.

---

## At a glance

| Area | Technology | Version |
| --- | --- | --- |
| **Language (front)** | TypeScript | 5.9 |
| **Language (back)** | C# / .NET | .NET 9 |
| **Mobile framework** | Ionic + Angular | Ionic 8 · Angular 21 |
| **Native runtime** | Capacitor | 8 |
| **Web framework** | Angular + SSR | 21 |
| **Backend framework** | ASP.NET Core | 9 |
| **ORM** | Entity Framework Core | 9 |
| **Database** | PostgreSQL (Azure Flexible Server) | Npgsql provider 9 |
| **On-device store** | SQLite with SQLCipher (Capacitor) / sql.js (browser) | plugin 8 · sql.js 1.14 |
| **Auth** | Firebase Authentication, ID tokens validated by the API | Firebase JS SDK 12 · FirebaseAdmin 3.2 |
| **Subscriptions** | RevenueCat | Capacitor SDK 12 |
| **Cloud** | Microsoft Azure | — |
| **API contract** | OpenAPI / Swagger → generated Angular client | OpenAPI 3 |
| **CI/CD** | GitHub Actions + Docker | — |

---

## Mobile app

The mobile app is a **hybrid (web-tech) application** compiled to native iOS and Android
binaries. Every personal feature works offline; the backend is used only for publishing,
the subscription and in-app messages.

### Framework & runtime

| Technology | Version | Purpose |
| --- | --- | --- |
| **Angular** | 21 | Application framework — standalone components, signals, DI, router |
| **Ionic Framework** (`@ionic/angular`, `@ionic/core`) | 8.7 | Mobile UI component library and navigation |
| **Capacitor** (`@capacitor/core`) | 8.1 | Native runtime bridging web code to iOS/Android APIs |
| **TypeScript** | 5.9 | Language |
| **RxJS** | 7.8 | Async streams, bridged into signals |
| **Zone.js** | 0.16 | Angular change detection |

### On-device storage (offline-first)

| Technology | Version | Purpose |
| --- | --- | --- |
| **`@capacitor-community/sqlite`** | 8 | SQLite on iOS/Android, encrypted with **SQLCipher** |
| **`sql.js`** (WASM) | 1.14 | SQLite compiled to WebAssembly for the browser build |
| **`@ionic/storage-angular`** | 4 | Key/value storage abstraction |
| **`@capacitor/preferences`** | 8 | Small native key/value settings |
| **`capacitor-secure-storage-plugin`** | 0.13 | Keychain/keystore storage for the database passphrase and tokens |

The database passphrase is random, generated on first launch and held in secure storage (see
[encryption at rest](architecture-mobile.md#encryption-at-rest)).

### Native device capabilities (Capacitor plugins)

| Plugin | Capability |
| --- | --- |
| `@capacitor/local-notifications` | Daily and weekly reminders the person schedules |
| `@capacitor/camera`, `@capawesome/capacitor-file-picker` | Photo capture and file selection |
| `@capacitor/filesystem` | Backup export and import |
| `@capacitor/share`, `@capacitor/clipboard` | Sharing and copying, e.g. a published calendar's link |
| `@capacitor/haptics` | Tactile feedback |
| `@capacitor/network` | Connectivity awareness |
| `@capacitor/app`, `@capacitor/device`, `@capacitor/browser`, `@capacitor/app-launcher` | App lifecycle, device info, in-app browser and external links |
| `@capacitor/keyboard`, `@capacitor/status-bar`, `capacitor-plugin-safe-area`, `@capawesome/capacitor-android-edge-to-edge-support` | Native chrome, safe areas and edge-to-edge layout |
| `@capacitor/screen-orientation`, `@capacitor-community/keep-awake` | Screen behavior |
| `@capacitor/toast` | Native toasts |
| `@capacitor-community/in-app-review` | Native store review prompt |

### Platform services

| Service | Package | Purpose |
| --- | --- | --- |
| **Firebase Authentication** | `@capacitor-firebase/authentication`, `firebase` | Native Google and Apple sign-in |
| **Firebase Analytics** | `@capacitor-firebase/analytics` | Product analytics, after consent |
| **Firebase Crashlytics** | `@capacitor-firebase/crashlytics` | Crash reporting, after consent |
| **Firebase Performance** | `@capacitor-firebase/performance` | Performance monitoring, after consent |
| **Firebase Remote Config** | `@capacitor-firebase/remote-config` | Feature flags, remote tuning and kill switches |
| **RevenueCat** | `@revenuecat/purchases-capacitor` | Wastime Pro subscriptions — offers, purchase, restore |

### UI, visualization & input

| Technology | Version | Purpose |
| --- | --- | --- |
| **Apache ECharts** (`echarts`, `ngx-echarts`) | 6 | Statistics charts and life-calendar visualizations |
| **Quill** (`ngx-quill`, `quill-mention`) | 2 | Rich-text notes and Age Stories |
| **Swiper** | 12 | Carousels and onboarding slides |
| **Maskito** (`@maskito/*`) | 5 | Input masking |
| **`ngx-image-cropper`** | 9 | Avatar and image cropping |
| **`ngx-color`** | 10 | Color pickers for topics and milestones |
| **`ionicons`**, **`animate.css`** | 8 / 4 | Icons and CSS animations |
| **`lodash-es`** | 4 | Utilities |

### Native tooling & quality

| Tool | Purpose |
| --- | --- |
| `@capacitor/cli`, `@capacitor/android`, `@capacitor/ios` | Native project sync and builds |
| `@capacitor/assets` | Icon and splash-screen generation |
| Android Studio / Gradle, Xcode | Platform builds and store submission |
| `@openapitools/openapi-generator-cli` | Generates the typed API client |
| Karma + Jasmine | Unit and integration specs |
| Playwright | End-to-end tests of the backup round-trip |
| ESLint 9 + `@angular-eslint` | Linting with strict rules (explicit access modifiers and return types) |
| Custom version scripts | Bump the version on commit and sync it into the native Android and iOS projects |

---

## Backend API

The backend is an **ASP.NET Core 9** REST API with a layered architecture and command/query
handlers in vertical slices.

### Runtime & framework

| Technology | Version | Purpose |
| --- | --- | --- |
| **.NET** | 9 | Runtime and SDK |
| **ASP.NET Core** | 9 | Web API framework |
| **C#** | 13 (implicit usings enabled) | Language |

### Project layers

```text
API layer            → HTTP: controllers, middleware, auth, versioning, Swagger, hosted services
  └─ Business logic  → command & query handlers, area facades, DTOs, Blob Storage
       └─ Data access             → EF Core DbContext, entity configurations
            ├─ Data infrastructure
            └─ Entities            → domain & Identity entities
                 └─ Foundation     → typed exceptions, shared models

Supporting projects (outside the request path):
  Email service       → email composition and sending
  Migrations runner   → applies EF Core migrations
  Seeding tool        → reference data and showcase calendars
  Unit / integration test suites
```

### Data access

| Technology | Version | Purpose |
| --- | --- | --- |
| **Entity Framework Core** | 9 | ORM |
| **Npgsql.EntityFrameworkCore.PostgreSQL** | 9 | PostgreSQL provider |
| **ASP.NET Core Identity** (EF Core stores) | 9 | User accounts and roles |
| **PostgreSQL** (Azure Flexible Server) | — | Relational database |

### Security & authentication

| Technology | Version | Purpose |
| --- | --- | --- |
| **FirebaseAdmin** | 3.2 | Verifies Firebase-issued ID tokens server-side |
| **Microsoft.AspNetCore.Authentication.JwtBearer** | 9 | Bearer authentication with Firebase as the authority |
| **OwaspHeaders.Core** | 9.7 | Secure HTTP response headers (OWASP best practices) |
| **ASP.NET Core rate limiting** | 9 | Sliding-window per-IP policies |
| **Azure Key Vault** | — | Secret storage, auto-generated credentials, Managed Identity |

### Integrations

| Technology | Purpose |
| --- | --- |
| **RevenueCat REST API** | Source of truth for entitlements, called through a typed `HttpClient` after each signed webhook |
| **Azure Blob Storage** (`Azure.Storage.Blobs` 12.20) | Private storage for report attachments and other files |

### API contract & documentation

| Technology | Version | Purpose |
| --- | --- | --- |
| **Swashbuckle.AspNetCore** | 8.1 | Generates the OpenAPI specification |
| **ASP.NET Core API Versioning** | 5.1 | URL-based API versioning |

The generated OpenAPI specification is the **single source of truth** for clients — see
[Shared API contract](#shared-api-contract).

### Observability

| Technology | Version | Purpose |
| --- | --- | --- |
| **Serilog** (`Serilog.AspNetCore`) | 4.3 / 9.0 | Structured logging |
| **Serilog.Sinks.ApplicationInsights** | 4.0 | Ships logs to Azure |
| **Microsoft.ApplicationInsights.AspNetCore** | 2.22 | Telemetry, traces, metrics |

### Cloud infrastructure (Microsoft Azure)

| Service | Purpose |
| --- | --- |
| **Azure App Service** | Hosts the API (Docker, Linux) |
| **Azure Database for PostgreSQL — Flexible Server** | Primary datastore |
| **Azure Blob Storage** | Private, encrypted file storage |
| **Azure Key Vault** | Secrets and auto-generated passwords |
| **Application Insights + Log Analytics** | Monitoring, telemetry, availability test and alerts |

### Backend tooling & tests

| Tool | Purpose |
| --- | --- |
| **Bicep** | Infrastructure as code for the whole Azure footprint, monitoring included |
| **PowerShell scripts** | Infrastructure deployment, secret provisioning, password rotation |
| **Docker** / Docker Compose | Containerization, local and deployed |
| **GitHub Actions** | Build, test, staging and production deploys, showcase seeding |
| **xUnit, Moq, MockQueryable, FluentAssertions** | Unit and integration tests |
| **`WebApplicationFactory`** + EF Core InMemory | In-process integration tests of the HTTP pipeline |

---

## Website

The website is an **Angular 21** application **prerendered** at build time, with a Markdown-driven
blog and pages that load API data in the browser.

### Framework & rendering

| Technology | Version | Purpose |
| --- | --- | --- |
| **Angular** | 21 | Application framework |
| **`@angular/ssr`**, **`@angular/platform-server`** | 21 | Server rendering and static prerendering |
| **TypeScript** | 5.9 | Language |

### UI

| Technology | Version | Purpose |
| --- | --- | --- |
| **Bootstrap** | 5.3 | Layout and components |
| **`@ng-bootstrap/ng-bootstrap`** | 20 | Angular Bootstrap widgets |
| **`bootstrap-icons`** | 1.13 | Icon set |
| **Quill** | 2 | Rendering a year's Age Stories on a published calendar |
| **AOS** | 2.3 | Animate-on-scroll effects |
| **Swiper** | 12 | Carousels |
| **`@angular-slider/ngx-slider`**, **`ngx-mask`** | 21 / 20 | Interactive inputs |
| **`vanilla-cookieconsent`** | 3.1 | Cookie consent |

### Content, blog & SEO

| Technology | Version | Purpose |
| --- | --- | --- |
| **`gray-matter`** | 4 | Parses front-matter from Markdown posts |
| **`marked`** | 17 | Markdown → HTML rendering |
| **`canvas`** | 3 | Generates Open Graph images for blog posts at build time |
| **`sharp`** | 0.35 | Cuts landing showcase images out of the app's store screenshots |
| Custom Node scripts + pure tool modules | — | Validate and build content, prerender routes, generate sitemaps, RSS, redirects and OG images |

### Hosting & deployment

| Target | Notes |
| --- | --- |
| **Azure Static Web Apps** behind Cloudflare | Production at `wastime.app`, deployed from a release branch |
| **Firebase Hosting** | Closed staging, `noindex` everywhere |

### Website quality

| Tool | Purpose |
| --- | --- |
| ESLint, Stylelint, Prettier | Linting and formatting |
| Karma + Jasmine | Unit tests in random order, with a coverage ratchet |
| Node's built-in test runner | Tests for the build-time pipeline modules |
| Husky | Pre-commit lint and format |

---

## Admin portal *(in development)*

An operator-facing web app for moderating published calendars and managing in-app messages.

| Technology | Version | Purpose |
| --- | --- | --- |
| **Angular** + **Angular Material** | 22 | Application framework and UI; no SSR, everything is behind sign-in |
| **TypeScript** | 6.0 | Language |
| **Firebase JS SDK** | — | Operator sign-in, with roles checked by the API |
| **Generated API client** | — | The same OpenAPI-driven client approach as the other apps |
| **Vitest** | 5 | Unit tests |

---

## Shared / cross-cutting

### Shared API contract

Every client talks to the backend through a **typed Angular client generated from the backend's
OpenAPI spec** — never hand-written HTTP calls:

```text
ASP.NET Core (Swashbuckle)
        │  produces
        ▼
   OpenAPI specification
        │  OpenAPI generator (typescript-angular)
        ▼
 generated Angular API client   ← imported by the mobile app, website and admin portal
```

- Generator: **`@openapitools/openapi-generator-cli`** (`typescript-angular`).
- Invoked as a single command that regenerates the client from the specification.
- Request and response types, enums and services stay in lock-step with the backend, and
  breaking changes surface at compile time.

### Firebase (shared identity & product platform)

Firebase provides authentication across mobile and web (Google, Apple, email/password), and the
backend validates those identities with **FirebaseAdmin**. On mobile, Firebase also supplies
Analytics, Crashlytics, Performance and Remote Config, all behind the consent gate.

### Quality & developer experience

| Concern | Tooling |
| --- | --- |
| **Linting** | ESLint + `@angular-eslint` (strict rules), Stylelint (SCSS) |
| **Formatting** | Prettier |
| **Testing (front)** | Karma + Jasmine, Playwright (mobile e2e), Vitest (admin portal) |
| **Testing (back)** | xUnit unit and integration suites |
| **Documentation** | Markdown context linting in CI across the repositories |
| **Versioning** | Automated semver bump on commit; synced into the native Android/iOS projects |
| **CI/CD** | GitHub Actions in every repository |

---

## Notable decisions & rationale

- **Offline-first with encrypted on-device SQLite.** The product promise is privacy and "works
  without the internet." A SQLCipher-encrypted database on the phone makes every personal feature
  available offline; the backend only adds what genuinely needs a server.
- **One codebase, two stores via Capacitor.** Angular + Ionic + Capacitor ship the same
  TypeScript code to iOS and Android, which keeps feature parity cheap for a small team.
- **Page-scoped Sandboxes and signals.** Keeping logic out of components and state in signals
  makes screens easy to test and keeps change detection predictable with OnPush.
- **Layered .NET solution with command/query handlers.** One handler per use case keeps the
  business logic small, testable and independent of HTTP, without a mediator library.
- **The server decides about money.** The publication quota and the entitlement live in the
  backend; the RevenueCat webhook is only a signal to re-read the customer, which makes it
  idempotent and order-independent.
- **OpenAPI-generated clients.** Generating every client from the server's spec removes a whole
  class of front/back drift bugs and makes API changes visible at compile time.
- **Prerendered website with API shells.** Static HTML keeps the site fast and crawlable, while
  pages backed by live data prerender as a shell and fetch in the browser.
- **Security by default on the backend.** Authenticated-by-default endpoints, Managed Identity,
  Key Vault with auto-generated secrets, private Blob Storage and OWASP headers keep secrets out
  of code and mistakes out of production.
