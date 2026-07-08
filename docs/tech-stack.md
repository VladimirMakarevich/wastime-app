# Technology Stack

This document is the full, versioned breakdown of the technologies behind **Wastime**, with
the reasoning behind the key choices. It is the reference that the three architecture
documents build on:

- [Mobile app architecture](architecture-mobile.md)
- [Backend architecture](architecture-backend.md)
- [Web site architecture](architecture-web.md)

Wastime is delivered as **three cooperating applications** that share a single, generated API
contract:

| Application | Role | Core technology |
| --- | --- | --- |
| **Mobile app** | The primary product (iOS &amp; Android) | Ionic + Angular + Capacitor |
| **Backend API** | Optional cloud sync, accounts, storage | ASP.NET Core 9.0 |
| **Web site** | Marketing landing + blog, web access | Angular 21 with SSR |

> Versions below reflect the current manifests (`package.json`, `*.csproj`). Minor/patch
> versions move over time; the major versions and the architectural choices are the stable
> part of this document.

---

## At a glance

| Area | Technology | Version |
| --- | --- | --- |
| **Language (front)** | TypeScript | 5.9 |
| **Language (back)** | C# / .NET | .NET 9.0 |
| **Mobile framework** | Ionic + Angular | Ionic 8 · Angular 21 |
| **Native runtime** | Capacitor | 8 |
| **Web framework** | Angular + SSR | 21 |
| **Backend framework** | ASP.NET Core | 9.0 |
| **ORM** | Entity Framework Core | 9.0 |
| **Database** | PostgreSQL (Azure Flexible Server) | 9.x provider (Npgsql) |
| **On-device store** | SQLite (Capacitor) / sql.js (web) | 8 |
| **Auth** | Firebase Authentication + JWT | — |
| **Cloud** | Microsoft Azure | — |
| **Platform services** | Firebase, RevenueCat | — |
| **API contract** | OpenAPI / Swagger → generated Angular client | 3.x |
| **CI/CD** | GitHub Actions + Docker | — |

---

## Mobile app

The mobile app is a **hybrid (web-tech) application** compiled to native iOS and Android
binaries. It is offline-first: the full feature set works with no network connection, and
cloud sync is optional.

### Framework & runtime

| Technology | Version | Purpose |
| --- | --- | --- |
| **Angular** | 21 | Application framework (standalone components, DI, router) |
| **Ionic Framework** (`@ionic/angular`, `@ionic/core`) | 8 | Mobile UI component library and navigation |
| **Capacitor** (`@capacitor/core`) | 8 | Native runtime bridging web code to iOS/Android APIs |
| **TypeScript** | 5.9 | Language |
| **RxJS** | 7.8 | Reactive state and async streams |
| **Zone.js** | 0.16 | Angular change detection |

### On-device storage (offline-first)

| Technology | Version | Purpose |
| --- | --- | --- |
| **`@capacitor-community/sqlite`** | 8 | Encrypted SQLite database on iOS/Android |
| **`sql.js`** (WASM) | — | SQLite compiled to WebAssembly, used when running on the web |
| **`@ionic/storage-angular`** | 4 | Key/value storage abstraction |
| **`@capacitor/preferences`** | 8 | Small native key/value settings |
| **`capacitor-secure-storage-plugin`** | 0.13 | Secure (keychain/keystore) storage for sensitive values |

SQLite is configured with **encryption on both platforms** and a dedicated iOS keychain
prefix (see [`capacitor.config.ts`](architecture-mobile.md#native-configuration)).

### Native device capabilities (Capacitor plugins)

| Plugin | Capability |
| --- | --- |
| `@capacitor/local-notifications` | Reminders and scheduled notifications |
| `@capacitor/camera`, `@capawesome/capacitor-file-picker` | Photo capture and file selection |
| `@capacitor/filesystem` | Import/export of JSON data |
| `@capacitor/share`, `@capacitor/clipboard` | Sharing and copy |
| `@capacitor/haptics` | Tactile feedback |
| `@capacitor/network` | Connectivity awareness for sync |
| `@capacitor/keyboard`, `capacitor-plugin-safe-area`, `@capacitor/status-bar`, `@capacitor/splash-screen` | Native chrome / layout |
| `@capacitor/screen-orientation`, `@capacitor-community/keep-awake` | Screen behavior |
| `@capacitor-community/in-app-review` | Native store review prompts |

### Platform services

| Service | Package | Purpose |
| --- | --- | --- |
| **Firebase Authentication** | `@capacitor-firebase/authentication`, `firebase` | Native Google / Apple sign-in, guest mode |
| **Firebase Analytics** | `@capacitor-firebase/analytics` | Product analytics |
| **Firebase Crashlytics** | `@capacitor-firebase/crashlytics` | Crash reporting |
| **Firebase Performance** | `@capacitor-firebase/performance` | Performance monitoring |
| **Firebase Remote Config** | `@capacitor-firebase/remote-config` | Feature flags / remote tuning |
| **RevenueCat** | `@revenuecat/purchases-capacitor` | In-app subscriptions and entitlements |

### UI, visualization & input

| Technology | Version | Purpose |
| --- | --- | --- |
| **Apache ECharts** (`echarts`, `ngx-echarts`) | 6 | Statistics charts and life-calendar visualizations |
| **Quill** (`ngx-quill`, `quill-mention`) | 2 | Rich-text notes |
| **Swiper** | 12 | Carousels / onboarding slides |
| **Maskito** (`@maskito/*`) | 5 | Input masking |
| **`ngx-image-cropper`** | 9 | Avatar / image cropping |
| **`ngx-color`** | 10 | Color pickers for topics and milestones |
| **`ionicons`**, **`animate.css`** | 8 / 4 | Icons and CSS animations |
| **`zod`** | 4 | Runtime schema validation |
| **`lodash-es`** | 4 | Utilities |

### Native tooling

| Tool | Purpose |
| --- | --- |
| `@capacitor/cli`, `@capacitor/android`, `@capacitor/ios` | Native project sync and builds |
| `@capacitor/assets` | Icon and splash-screen generation |
| Android Studio / Gradle, Xcode | Platform builds and store submission |
| Custom version scripts | Sync `package.json` version into Android `build.gradle` and iOS `project.pbxproj` |

---

## Backend API

The backend is an **ASP.NET Core 9.0** REST API following a layered (Clean-ish) architecture.
It exists to support **optional** cloud sync, accounts, and file storage — the app does not
require it to function.

### Runtime & framework

| Technology | Version | Purpose |
| --- | --- | --- |
| **.NET** | 9.0 | Runtime and SDK |
| **ASP.NET Core** | 9.0 | Web API framework |
| **C#** | 13 (implicit usings enabled) | Language |

### Project layers

The solution (`Ti.Waste.sln`) is split into focused projects with a one-directional
dependency flow:

```text
Ti.Waste.Api          → HTTP layer: controllers, auth, versioning, Swagger
   └─ Ti.Waste.Core   → Business logic / services (+ Azure Blob Storage)
        └─ Ti.Waste.Data           → EF Core DbContext, repositories
             ├─ Ti.Waste.Data.Infrastructure  → persistence infrastructure
             └─ Ti.Waste.Entities             → domain & Identity entities
                  └─ Ti.Waste                  → base domain / EF Core primitives

Supporting projects:
  Ti.Waste.Email       → email composition/sending
  Ti.Waste.Migrations  → EF Core migrations runner
  Ti.Waste.SeedingApp  → data seeding
  *.UnitTests / *.IntegrationTests → test suites
```

### Data access

| Technology | Version | Purpose |
| --- | --- | --- |
| **Entity Framework Core** | 9.0 | ORM |
| **Npgsql.EntityFrameworkCore.PostgreSQL** | 9.0 | PostgreSQL provider |
| **ASP.NET Core Identity** (EF Core stores) | 9.0 | User accounts and identity model |
| **PostgreSQL** (Azure Flexible Server) | — | Relational database |

### Security & authentication

| Technology | Version | Purpose |
| --- | --- | --- |
| **FirebaseAdmin** | 3.2 | Verifies Firebase-issued identity tokens server-side |
| **Microsoft.AspNetCore.Authentication.JwtBearer** | 9.0 | JWT bearer authentication |
| **OwaspHeaders.Core** | 9.7 | Secure HTTP response headers (OWASP best practices) |
| **Azure Key Vault** | — | Secret storage; auto-generated 24-char credentials, Managed Identity |

### API contract & documentation

| Technology | Version | Purpose |
| --- | --- | --- |
| **Swashbuckle.AspNetCore** | 8.1 | Generates the OpenAPI/Swagger spec (`swagger.json`) |
| **ASP.NET Core API Versioning** | 5.1 | URL/media-type API versioning |

The generated `swagger.json` is the **single source of truth** for clients — see
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
| **Azure App Service / Container** | Hosts the API (Docker, Linux) |
| **Azure Database for PostgreSQL — Flexible Server** | Primary datastore |
| **Azure Blob Storage** (`Azure.Storage.Blobs` 12.20) | Private, encrypted file storage |
| **Azure Key Vault** | Secrets and auto-generated passwords |
| **Application Insights** | Monitoring and telemetry |

### Backend tooling

| Tool | Purpose |
| --- | --- |
| **Docker** (`docker-compose.yml`) | Containerization, local + deploy |
| **PowerShell IaC scripts** (`src/infra`) | Automated Azure infrastructure deployment |
| **GitHub Actions** | CI/CD |

---

## Web site

The web site is an **Angular 21** application rendered with **SSR / prerendering** for SEO,
serving a marketing landing page and a markdown-driven blog, plus web access to the product.

### Framework & rendering

| Technology | Version | Purpose |
| --- | --- | --- |
| **Angular** | 21 | Application framework |
| **`@angular/ssr`**, **`@angular/platform-server`** | 21 | Server-side rendering + static prerendering |
| **`@angular/localize`** | 21 | i18n support |
| **TypeScript** | 5.9 | Language |

### UI

| Technology | Version | Purpose |
| --- | --- | --- |
| **Bootstrap** | 5.3 | Layout and components |
| **`@ng-bootstrap/ng-bootstrap`** | 20 | Angular Bootstrap widgets |
| **`bootstrap-icons`** | 1.13 | Icon set |
| **AOS** | 2.3 | Animate-on-scroll effects |
| **Swiper** | 12 | Carousels |
| **`@angular-slider/ngx-slider`**, **`ngx-mask`** | 21 / 20 | Interactive inputs |
| **`vanilla-cookieconsent`** | 3.1 | Cookie consent / privacy |

### Content, blog & SEO

| Technology | Version | Purpose |
| --- | --- | --- |
| **`gray-matter`** | 4 | Parses front-matter from markdown blog posts |
| **`marked`** | 17 | Markdown → HTML rendering |
| **`canvas`** | 3 | Generates Open Graph share images at build time |
| Custom Node scripts | — | Build blog content, prerender routes, generate SEO artifacts (sitemap, redirects, OG images) |

### Hosting & deployment

| Target | Config | Notes |
| --- | --- | --- |
| **Firebase Hosting** | `firebase.json`, `.firebaserc` | Primary hosting + Firebase Auth |
| **Azure Static Web Apps** | `staticwebapp.config.json` | Alternative deploy target |

---

## Shared / cross-cutting

### Shared API contract

Both the mobile app and the web site talk to the backend through a **typed Angular client
generated from the backend's OpenAPI spec** — not hand-written HTTP calls:

```text
ASP.NET Core (Swashbuckle)
        │  produces
        ▼
   openapi/swagger.json
        │  openapi-generator-cli (typescript-angular)
        ▼
 projects/waste-api-client   ← imported by mobile app & web site
```

- Generator: **`@openapitools/openapi-generator-cli`** (`typescript-angular`)
- Command: `npm run generate:api-client`
- Benefit: request/response types, enums, and services stay in lock-step with the backend;
  breaking changes surface at compile time.

### Firebase (shared identity & product platform)

Firebase provides authentication across mobile and web (Google, Apple, Email/Password, plus a
**guest mode**), while the backend uses **FirebaseAdmin** to verify those identities. On
mobile, Firebase additionally supplies Analytics, Crashlytics, Performance, and Remote Config.

### Quality & developer experience

| Concern | Tooling |
| --- | --- |
| **Linting** | ESLint 9 + `@angular-eslint` (strict rules: explicit access modifiers &amp; return types), Stylelint (SCSS) |
| **Formatting** | Prettier |
| **Testing (front)** | Karma + Jasmine (unit), Angular e2e |
| **Testing (back)** | Unit + integration test projects |
| **Git hooks** | Husky (web), pre-commit version bump (mobile) |
| **Versioning** | Automated semver bump on commit; synced into native Android/iOS project files |
| **CI/CD** | GitHub Actions across all three repositories |

---

## Notable decisions & rationale

- **Offline-first with on-device SQLite.** The product promise is privacy and "works without
  the internet." An encrypted SQLite database (native) / sql.js (web) makes the full feature
  set available offline; the backend is an *optional* sync layer, not a hard dependency.
- **One codebase, three targets via Capacitor.** Angular + Ionic + Capacitor lets the same
  TypeScript codebase ship to iOS, Android, and the web, keeping feature parity cheap.
- **Layered .NET solution.** Splitting `Api / Core / Data / Entities` enforces a clean
  dependency direction, isolates EF Core from business logic, and keeps the domain testable.
- **OpenAPI-generated client.** Generating the client from the server's spec removes a whole
  class of front/back drift bugs and makes API changes compile-time visible.
- **SSR/prerender for the marketing site.** A separate Angular SSR app (rather than the SPA)
  gives fast, crawlable, SEO-friendly pages with generated sitemaps and OG images.
- **Security by default on the backend.** Managed Identity, Key Vault with auto-generated
  secrets, private Blob Storage, and OWASP security headers keep secrets out of code.
