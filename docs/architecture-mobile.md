# Mobile App Architecture

The Wastime mobile app is the **primary product**: a single Angular + Ionic codebase compiled
to native **iOS** and **Android** binaries with Capacitor, and also runnable on the web. It is
**offline-first** — every feature works with no network, and cloud sync is an optional layer
on top.

> This document describes the architecture at the level of **layers, roles, and patterns**
> rather than concrete file or module names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#mobile-app).

---

## Design principles

The architecture is a direct expression of the product's promises:

| Product promise | Architectural consequence |
| --- | --- |
| **Works fully offline** | All reads/writes hit a local **encrypted embedded database**; the network is never on the critical path. |
| **Privacy by default** | Cloud sync is opt-in and **consent-gated**; data leaves the device only after explicit user consent. |
| **Your data, your rules** | Local **export/import**, soft-deletes, and full local ownership of the datastore. |
| **A tool, not a life center** | Feature areas are independent and lazily loaded; the app stays small and fast. |

---

## Layered overview

The app is organized into four layers with a strict top-to-bottom dependency direction.

```text
┌───────────────────────────────────────────────────────────────┐
│  Presentation                                                  │
│    feature screens · shared UI components                      │
│    mobile UI kit · router · charts · rich-text editor          │
└───────────────────────────┬───────────────────────────────────┘
                            │ services / event bus
┌───────────────────────────┴───────────────────────────────────┐
│  Application & cross-cutting                                   │
│    app services · auth · HTTP interceptors · route guards      │
│    internal event bus · i18n · remote config · theming · …     │
└───────────────────────────┬───────────────────────────────────┘
                            │ repositories
┌───────────────────────────┴───────────────────────────────────┐
│  Data                                                          │
│    repository layer → object mapping (ORM) → domain entities   │
│    schema migrations · soft-delete · connection management     │
│    synchronization (optional, consent-gated — in progress)     │
└───────────────────────────┬───────────────────────────────────┘
                            │ database driver
┌───────────────────────────┴───────────────────────────────────┐
│  Persistence & native  (Capacitor bridge)                      │
│    encrypted embedded SQLite (native) / WebAssembly SQLite (web)│
│    secure storage · key-value preferences                      │
│    device plugins: notifications, camera, filesystem, …        │
└───────────────────────────────────────────────────────────────┘
```

Dependencies flow **downward only**: presentation depends on the application layer, which
depends on the data layer, which depends on the database driver. Nothing lower reaches up.

---

## Presentation layer

### Feature areas

Each product capability is an independent, lazily loaded feature area. They map one-to-one to
the user-facing features:

| Feature area | What it does |
| --- | --- |
| **Time tracking** | Meaningful vs Wasted spaces, topics and groups, quotas and forecasts |
| **Timers** | A single active timer with pauses; sessions convert into time entries |
| **Time entry editing** | Manual creation and editing of time records |
| **Life calendars** | Life in Weeks / Life in Days, with milestones and groups |
| **Life-expectancy setup** | Country-based or manual life-expectancy configuration |
| **Calculators** | The five calculators (Live, Wasted, Basic, 24H, Reverse) |
| **Converter** | Money → Time conversion and accounts |
| **Statistics** | Overviews, trends, and insights |
| **Journey** | Roadmap / time-journey stories |
| **Reflective questions** | Guided questions |
| **Onboarding** | First-run setup flow |
| **Shell** | Dashboard, profile/account, and sign-in |

### Shared UI

A library of reusable presentational components — charts, color pickers, date/time selectors,
image cropping, searchable selects, avatars, skeleton loaders, swipeable rows, and error/empty
states. The UI is built on a **mobile component kit** with a **charting library** for
visualization and a **rich-text editor** for notes.

---

## Application & cross-cutting layer

This layer holds everything that is neither a screen nor persistence. It is organized by
responsibility:

| Area | Responsibility |
| --- | --- |
| **App services** | Cross-app concerns: theming / dark mode, connectivity, analytics, haptics, logging, localization, backup, remote configuration |
| **Authentication** | Federated sign-in (Google / Apple), secure token storage backed by the platform keychain/keystore, and a guest mode |
| **HTTP interceptors** | A request pipeline that attaches auth tokens, reacts to connectivity, and emits performance traces |
| **Internal event bus** | A typed publish/subscribe bus that decouples feature areas |
| **Route guards** | Access control for navigation |
| **Global state** | Small, app-wide runtime state (e.g. device configuration) |
| **Angular building blocks** | Shared pipes, directives, form validators, and utilities |

### Internal event bus

Rather than wiring feature areas directly to one another, cross-cutting events flow through a
typed **event bus**. This keeps features decoupled — for example, a timer completing can
publish an event that statistics and synchronization consume independently.

---

## Data layer — the offline-first core

This is the heart of the app. All application data lives in a **local embedded database**,
accessed through a lightweight object-mapping layer and a repository abstraction.

```text
Repository layer          One repository per aggregate; typed CRUD & queries
      │  uses
Object mapping (ORM)      Lightweight, decorator-based mapping tailored for SQLite
      │  maps
Domain entities           Plain models extending a common base entity
      │  persisted via
Connection management     Opens/tracks the database connection
      │  runs on
Embedded SQLite driver    Encrypted native SQLite  /  WebAssembly SQLite on the web
```

### Object mapping (ORM)

A small, decorator-based mapping layer (in the spirit of established ORMs) tailored for SQLite:

- **Entity & column decorators** describe how models map to tables and columns.
- A **row-versioning decorator** stamps each row with a version — the basis for optimistic
  concurrency and conflict detection during sync.
- **Value transformers** bridge type mismatches between JavaScript and SQLite — for example,
  representing booleans as integers (SQLite has no boolean type) and normalizing null/undefined.

### Domain entities

Domain models are mapped to tables and share a common base entity that carries cross-cutting
fields (identity, timestamps, soft-delete and version markers). There is one entity per domain
concept across the feature set — time records, topics, timers, milestones, calculator results,
converter accounts, and so on.

### Repositories

Each aggregate is accessed through a repository built on a shared base, exposing typed CRUD and
query operations to the application layer. Cross-cutting data concerns live alongside:

- **Soft-delete** — rows are tombstoned rather than physically removed, which is required for
  correct synchronization and for undo.
- **Raw datastore access** — a low-level path used for export/import and diagnostics.

### Schema migrations

A versioned migration system evolves the local database as the app updates, so existing users'
data is upgraded safely on device.

---

## Application startup & initialization

Because the datastore must be ready before most features can render, startup is a deliberate
**multi-phase sequence** with a live progress screen and a stall watchdog — not a single
blocking call.

```text
Bootstrap ─▶ Phase 1: Critical (blocking, progress UI) ─▶ app interactive
                    │                                          │
              progress + watchdog                             ▼
                                              Phase 2: Secondary (after "stable")
                                                               │
                                                               ▼
                                                Phase 3: Background (on idle)
```

### Bootstrap

Before Angular starts, the app initializes the Firebase application, registers the web
components it needs (carousel elements, and — on the web — the WebAssembly SQLite element), and
configures the dependency-injection container: the HTTP client and its **interceptor chain**
(connectivity → auth → crash reporting → performance), the router (with route preloading
disabled to protect first paint), Firebase authentication (with native persistence), a global
error handler, and the internal event bus. A lightweight app-initializer starts performance
tracking.

### Phase 1 — Critical initialization (blocking)

The first phase brings the datastore up; it is the only part that gates the UI. It runs behind
a dedicated **progress screen** and advances through well-defined, individually reported steps:

```text
starting → key-value store ready → database opened
        → schema migrations (init → running → complete) → ready
```

- **Key-value store & query tracking** — bring up lightweight settings storage first.
- **Database** — open the encrypted embedded database (on the web, set up the WebAssembly store
  beforehand).
- **Schema migrations** — apply any pending migrations while streaming progress to the screen;
  a minimum display time avoids a distracting flash when migrations are fast.
- **Launch-type detection** — the app classifies the launch as **first install**, **update**,
  or **normal** (by comparing the stored version against the current one), which drives
  onboarding vs. update flows.

### Progress reporting & stall detection

A single **initialization-state service** holds the current step, percentage, and message as
reactive signals, which the progress screen renders live. A **watchdog** runs alongside: on an
interval it checks the time elapsed since the last progress tick and enforces a **per-step
timeout** (falling back to a default). If a step makes no progress within its budget, the app
raises a structured *stall* error and shows an **error / recovery panel** instead of hanging —
so a slow or corrupt start is recoverable rather than fatal.

### Phase 2 — Secondary initialization (deferred)

Once Angular reports the application is **stable**, a second phase runs **without blocking** the
now-interactive UI: it loads device settings, resolves locale information, and initializes the
remaining app services.

### Phase 3 — Background initialization (idle)

Non-critical work is deferred further still and scheduled during **idle time** (via the
browser's idle-callback, with a timeout fallback). These are best-effort tasks — for example
analytics initialization with retries — kept off the startup critical path entirely.

### Why phased?

Splitting startup this way keeps **time-to-interactive** short: only the datastore and its
migrations gate the UI; everything else is pushed to after-stable or idle. The explicit step
model, the progress screen, and the watchdog together make a normally opaque process
**observable and resilient**.

---

## Offline-first & synchronization

### Local-first writes

Every user action writes to the local database **immediately**, and the UI updates from local
state. There is no "saving to server" step on the critical path — the network is never
required.

### Sync engine (optional, consent-gated — *in progress*)

An **outbox-based synchronization engine** provides optional multi-device sync. It is
architected but **not yet enabled** in the shipping app, which is currently local-only. The
design:

```text
Local write ──▶ Outbox (queue of pending changes)
                     │  batched push
                     ▼
             Background worker  (off the UI thread)
                     │  HTTPS (typed API client)
                     ▼
              Backend API  ◀──▶  Cloud database
                     │  pull changes (since a cursor)
                     ▼
        Conflict detection (row versions + expected version)
                     │
        ┌────────────┴─────────────┐
   auto-merge                conflict store
                             → user-facing resolution
```

The building blocks are designed to be **idempotent, versioned, conflict-aware, and
consent-first**:

| Concern | Approach |
| --- | --- |
| **Outbox** | Local changes are queued for push (outbox pattern) |
| **Sync cursor** | Per-entity "last synced" position drives incremental pulls |
| **Idempotency** | Idempotency keys ensure retried pushes never duplicate |
| **Conflicts** | Version mismatches are detected, stored, and surfaced for user resolution |
| **Consent gate** | Sync only runs after explicit user consent — the privacy boundary |
| **Observability** | Activity log, metrics, and status reporting for sync |
| **Off-thread** | Push/pull run in a background worker, never blocking the UI |

This mirrors the product's privacy stance: nothing leaves the device implicitly.

### Network awareness

The app is connectivity-aware, so sync (when enabled) only activates on suitable connections
and the UI degrades gracefully offline.

---

## Native configuration

Native behavior is declared through the Capacitor bridge. Highlights:

| Concern | Effect |
| --- | --- |
| **Database encryption** | Enabled on both iOS and Android, with platform keychain-backed keys |
| **Native HTTP** | Requests use the platform's native HTTP layer |
| **Keyboard** | Native resize behavior; style switches with the app's light/dark theme |
| **Federated auth** | Google and Apple providers, with native sign-in |
| **Crash reporting** | Enabled, including during development |
| **Status bar** | Edge-to-edge layout with safe-area handling |

The full set of native capabilities (notifications, camera, file picking, filesystem, share,
haptics, keep-awake, in-app review) is listed in
[tech-stack.md](tech-stack.md#native-device-capabilities-capacitor-plugins).

---

## Example data flow: logging a time entry

```text
1. User logs time in the tracking feature (or stops a timer)
2. A feature service calls the corresponding repository
3. The repository maps the entity via the ORM and writes to the local database
      → soft-delete and version fields are maintained automatically
4. The write is committed and the change is tracked
5. The UI updates from local state (instant); statistics recompute
6. (When sync is enabled) the change is enqueued in the outbox; a background
   worker pushes it after consent and connectivity checks
```

Steps 1–5 are entirely local and synchronous with the user — the cloud (step 6) is additive
and never blocks the experience.

---

## Build, versioning & release

- **Web build → native sync:** the Angular build output is synced into the native iOS/Android
  projects via the Capacitor CLI.
- **API client generation:** a typed API client is generated from the backend's OpenAPI spec,
  so the app and backend stay in lock-step.
- **Automated versioning:** a pre-commit step bumps the version and syncs it into the native
  Android and iOS project files.
- **Store builds:** produced with the standard Android and iOS platform toolchains.

See [tech-stack.md](tech-stack.md#native-tooling) for the toolchain details.
