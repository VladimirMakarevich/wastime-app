# Mobile App Architecture

The Wastime mobile app is the **product itself**: one Angular + Ionic codebase compiled to
native **iOS** and **Android** binaries with Capacitor, and also runnable in a browser for
development and tests. It is **offline-first** — every personal feature works with no network,
and the backend only comes in for publishing a life calendar, the subscription and in-app
messages.

> This document describes the architecture at the level of **layers, roles and patterns**
> rather than concrete file or module names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#mobile-app).

---

## Design principles

The architecture follows directly from what the product promises.

| Product promise | Architectural consequence |
| --- | --- |
| **Works fully offline** | All reads and writes go to a local **encrypted embedded database**; the network is never on the critical path. |
| **Privacy by default** | No account is needed; analytics and crash reporting stay off until the person agrees on a dedicated consent screen. |
| **Your data, your rules** | Local **export/import**, soft-deletes, and full local ownership of the datastore. |
| **Short, focused visits** | Feature areas are independent and lazily loaded, so the app starts fast and stays small. |

---

## Layered overview

The app is organized into four layers with a strict top-to-bottom dependency direction.

```text
┌───────────────────────────────────────────────────────────────┐
│  Presentation                                                 │
│    routed pages · page-scoped Sandboxes · forms               │
│    shared UI components · charts · rich-text editor           │
└───────────────────────────┬───────────────────────────────────┘
                            │ services / event bus
┌───────────────────────────┴───────────────────────────────────┐
│  Application & cross-cutting                                  │
│    app services · auth · purchases · HTTP interceptors ·      │
│    route guards · event bus · i18n · remote config · consent  │
└───────────────────────────┬───────────────────────────────────┘
                            │ repositories
┌───────────────────────────┴───────────────────────────────────┐
│  Data                                                         │
│    repository layer → object mapping (ORM) → domain entities  │
│    schema migrations · soft-delete · backup · connection mgmt │
│    synchronization (designed, currently dormant)              │
└───────────────────────────┬───────────────────────────────────┘
                            │ database driver
┌───────────────────────────┴───────────────────────────────────┐
│  Persistence & native  (Capacitor bridge)                     │
│    SQLCipher-encrypted SQLite (native) / WebAssembly SQLite   │
│    secure storage · key-value preferences                     │
│    device plugins: notifications, camera, filesystem, …       │
└───────────────────────────────────────────────────────────────┘
```

Dependencies flow **downward only**: presentation depends on the application layer, which
depends on the data layer, which depends on the database driver. Nothing lower reaches up.

---

## Presentation layer

### Feature areas

Each product capability is an independent, lazily loaded feature area.

| Feature area | What it does |
| --- | --- |
| **Home** | Entry points to every feature, with a customizable layout |
| **Time tracking** | Meaningful vs wasted entries, topics with time, deadline and consistency goals, per-topic statistics and a pace projection |
| **Wasted-time guide** | A yes-or-no decision tree that helps decide whether a habit is really wasted time |
| **Timers** | Activity stopwatches with pauses, session splitting and history; a session becomes a time entry |
| **Time entry editing** | Manual creation and editing of time records |
| **Calendars** | Life in weeks, a year in days and life in years, with milestones and milestone groups, Age Stories (a page per year of life) and an optional full-screen life timer |
| **Calendar publishing** | Publishing a life calendar as a web page, with access settings and a history of published versions |
| **Life expectancy** | Country life tables, a custom number, or a short ten-question calculator (with a BMI helper) |
| **Calculators** | Five calculators — 24-hour, Planning, Wasted, Leftover and Basic — with saved results |
| **Converter** | Money → time conversion based on income rates, with a history and the share of expected life it represents |
| **Statistics** | Overview, time and life sections |
| **Journey** | Eight chapters about time, with a reader, bookmarks and source notes |
| **Questions** | Reflective prompts with saved answers |
| **Onboarding** | A welcome screen, three short introductory paths and the consent step |
| **Account** | Profile, settings and reminders, subscription, data backup, data privacy and account deletion, help, feedback, about |
| **In-app messages** | A feed of messages from the Wastime team, read and unread |

A developer sandbox for diagnostics exists in development builds only; a guard keeps it out of
production.

### Page patterns

The presentation layer follows a small set of mandatory patterns, enforced in review and by lint.

- **Standalone, OnPush components** with templates and styles in separate files.
- **A page-scoped Sandbox per routed page.** Each root page provides its own facade at component
  level. Business rules, data loading, persistence and calculations live in the Sandbox or the
  services under it; components only bind state and forward user intent. Pages never share a
  Sandbox — shared behavior moves down into core or data services.
- **Signals for state.** Component and Sandbox state is held in Angular signals; observables are
  bridged into signals instead of being subscribed to by hand.
- **Typed form contexts.** Every form is a typed class that owns its form group, defaults, a
  declarative validation schema and its intent methods. Error messages come from a central
  validation service, never from templates.
- **Predictable Android back button.** Handlers use named priority levels (inline overlay,
  overlay guard, page, app-wide fallback). A handler that does not act passes the press on, and
  views subscribe on enter and unsubscribe on leave, so an unsaved-changes prompt can run before
  a sheet closes.
- **Double-tap protection.** Actions that navigate, open overlays or delete are throttled.
  Significant actions give haptic feedback, and destructive ones ask for confirmation.
- **Overlay safety.** Every modal and picker show or dismiss is raced against a watchdog, so a
  stuck animation cannot freeze the app.
- **One header, one content area** per routed page, with a translucent header and full-screen
  content.

### Shared UI

A library of reusable presentational components — charts, color pickers, date and time selectors,
image cropping, searchable selects, avatars, skeleton loaders, swipeable rows, and error and empty
states. The UI is built on the **Ionic component kit**, with **ECharts** for visualization and
**Quill** for rich-text notes. The app currently ships a light theme only, built from design tokens.

---

## Application & cross-cutting layer

This layer holds everything that is neither a screen nor persistence.

| Area | Responsibility |
| --- | --- |
| **App services** | Theming tokens, connectivity, analytics, haptics, logging, localization of dynamic text, backup, remote configuration, statistics math |
| **Authentication** | Google and Apple sign-in through Firebase, tokens in secure storage backed by the platform keychain/keystore; most of the app needs no account at all |
| **Purchases** | A thin wrapper over the RevenueCat SDK for offers, purchase and restore |
| **Consent** | The single gate that decides whether analytics, performance monitoring and crash reporting may run |
| **HTTP interceptors** | A request pipeline that attaches auth tokens, reacts to connectivity and emits performance traces |
| **Internal event bus** | A typed publish/subscribe bus that decouples feature areas |
| **Route guards** | Access control for navigation |
| **Angular building blocks** | Shared pipes, directives, form validators, a throttle decorator and utilities |

Platform capabilities are always reached through these wrappers. Feature code never calls a raw
Capacitor or Firebase plugin, and every service initializes explicitly and degrades gracefully
when a capability is missing.

### Internal event bus

Rather than wiring feature areas directly to one another, cross-cutting events flow through a
typed **event bus**. A timer completing, for example, can publish an event that statistics
consume without the two areas knowing about each other.

---

## Data layer — the offline-first core

All application data lives in a **local embedded database**, accessed through a lightweight
object-mapping layer and a repository abstraction.

```text
Repository layer          One repository per aggregate; typed CRUD & queries
      │  uses
Object mapping (ORM)      Lightweight, decorator-based mapping tailored for SQLite
      │  maps
Domain entities           Plain models extending a common base entity
      │  persisted via
Connection management     Opens/tracks the database connection
      │  runs on
Embedded SQLite driver    SQLCipher-encrypted native SQLite  /  WebAssembly SQLite in the browser
```

### Encryption at rest

On iOS and Android the database is encrypted with **SQLCipher**. The passphrase is random,
generated once on first launch and kept in the platform's secure storage (Keychain on iOS,
encrypted shared preferences on Android). The browser build used for development runs on
WebAssembly SQLite without encryption.

### Object mapping (ORM)

A small, decorator-based mapping layer tailored for SQLite:

- **Entity and column decorators** describe how models map to tables and columns.
- A **row-versioning decorator** stamps each row with a version, the basis for conflict
  detection once sync is enabled.
- **Value transformers** bridge type mismatches between JavaScript and SQLite — booleans stored
  as integers, normalized null/undefined, serialized dates.

### Domain entities & repositories

There is one entity per domain concept — time records, topics, timers, milestones, age stories,
calculator results, converter rates, journey bookmarks, in-app messages and so on. They share a
base entity that carries identity, timestamps and soft-delete and version markers.

Each aggregate is accessed through a repository built on a shared base. Alongside them:

- **Soft-delete** — rows are tombstoned rather than physically removed, which keeps undo and
  future sync correct.
- **Backup** — a guided export writes selected data to a JSON file, and a guided import restores
  it; both are covered by tests that run on a real SQLite engine.

### Schema migrations

The schema is defined by **versioned migrations** that run on the device during startup, so an
app update upgrades the existing database in place, with progress shown on screen.

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

Before Angular starts, the app initializes Firebase, registers the web components it needs
(carousel elements and, in the browser, the WebAssembly SQLite element) and configures dependency
injection — the HTTP client with its **interceptor chain** (connectivity → auth → crash reporting
→ performance), the router (with preloading disabled to protect first paint), Firebase
authentication with native persistence, a global error handler and the event bus.

### Phase 1 — Critical initialization (blocking)

The first phase brings the datastore up; it is the only part that gates the UI. It runs behind a
dedicated **progress screen** and advances through individually reported steps:

```text
starting → key-value store ready → database opened
        → schema migrations (init → running → complete) → ready
```

- **Key-value store** — lightweight settings storage comes up first.
- **Database** — open the encrypted database (in the browser, set up the WebAssembly store first).
- **Schema migrations** — apply pending migrations while streaming progress to the screen; a
  minimum display time avoids a distracting flash when they are fast.
- **Launch-type detection** — the launch is classified as **first install**, **update** or
  **normal** by comparing the stored version with the current one, which decides between
  onboarding and the regular flow.

### Progress reporting & stall detection

An **initialization-state service** holds the current step, percentage and message as signals,
and the progress screen renders them live. A **watchdog** runs alongside with a **per-step
timeout** and an overall stall timeout. If a step makes no progress within its budget, the app
raises a structured *stall* error and shows an **error / recovery panel** instead of hanging, so
a slow or corrupt start is recoverable rather than fatal.

### Phase 2 — Secondary initialization (deferred)

Once Angular reports the application is **stable**, a second phase loads device settings,
resolves locale information and initializes the remaining app services, without blocking the
now-interactive UI.

### Phase 3 — Background initialization (idle)

Non-critical work is scheduled during **idle time**, with a timeout fallback. These are
best-effort tasks — analytics initialization with retries, for example — kept off the startup
critical path entirely.

### Why phased?

Only the datastore and its migrations gate the UI; everything else waits until the app is stable
or idle. Together with the explicit step model, the progress screen and the watchdog, this keeps
**time-to-interactive** short and makes a normally opaque process **observable and resilient**.

---

## Calendar publishing & the subscription

Publishing a life calendar is the one feature that needs an account and a connection, and the
only paid capability in the app.

```text
Life calendar (local) ──▶ publish settings ──▶ Backend API ──▶ web page on the Wastime website
                              │                    │
                       access level          quota check
                  public · link · private   (free allowance or Wastime Pro)
```

- **Access levels.** *Public* calendars appear in the website's gallery and open for anyone with
  the link; *link* calendars are unlisted and open only for someone who has the link; *private*
  calendars open only for their owner after signing in on the site.
- **History.** Each publication is recorded on the device, so earlier versions can be reviewed,
  and the details screen copies or shares the link.
- **Quota on the server.** The free plan allows three publications in any rolling 30 days, and
  **Wastime Pro** removes the limit. The backend is the source of truth for both the quota and the
  entitlement; the app only displays the remaining allowance and maps a refusal to a clear
  message. Nothing on the client can unlock a paid capability by itself.
- **Purchases** go through the **RevenueCat** SDK (monthly and yearly plans). RevenueCat informs
  the backend through a signed webhook — see
  [backend architecture](architecture-backend.md#subscriptions).
- Calendars that are already published keep working whether or not a subscription is active.

---

## Notifications & in-app messages

Two separate mechanisms, neither of them used for marketing:

| Mechanism | How it works |
| --- | --- |
| **Reminders** | Daily or weekly local notifications that the person schedules in Settings, delivered by the OS with no server involved |
| **In-app messages** | A feed pulled from the backend, cached in the local database, with read/unread state; it appears inside the app rather than as a push notification |

---

## Journey content pipeline

The eight Journey chapters are authored as **Markdown** in the repository and compiled into JSON
artifacts at build time — the same content-as-code approach the website uses for its blog.

```text
Chapter Markdown + chapter order + source registry
        │
        ▼
Pure, unit-tested parser   → strict validation (fails the build)
        │
        ▼
Committed JSON artifacts   → loaded by the in-app reader
```

The parser rejects a broken heading hierarchy or page numbering, text outside a page,
unsupported Markdown, duplicate chapter keys, and footnotes that do not match the source
registry. A check command rebuilds the artifacts in memory and fails if the committed ones are
stale.

---

## Privacy & consent

- **No account is required** for anything except publishing a calendar.
- Analytics, performance monitoring and crash reporting are **off at the native level** until the
  person answers on the consent screen at the end of onboarding. The answer can be changed any
  time on the Data & Privacy screen.
- A purpose is collected only when **three switches agree** — the person's answer, a remote kill
  switch and the build configuration. None of them can turn collection on alone, and no stored
  answer means no collection. Events from before the decision are buffered, then released or
  dropped.
- Personal content (notes, entries, personal data) never goes into analytics, logs or crash
  reports — only product and technical events.
- Account deletion is available in the app, including revoking Sign in with Apple.

---

## Offline-first & synchronization

### Local-first writes

Every user action writes to the local database **immediately**, and the UI updates from local
state. There is no "saving to server" step on the critical path.

### Sync engine (designed, currently dormant)

An **outbox-based synchronization engine** for optional multi-device sync is architected on both
the client and the server, but it is **switched off** in the shipping app, which keeps personal
data local-only. The design:

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

| Concern | Approach |
| --- | --- |
| **Outbox** | Local changes are queued for push |
| **Sync cursor** | A per-entity "last synced" position drives incremental pulls |
| **Idempotency** | Idempotency keys ensure retried pushes never duplicate |
| **Conflicts** | Version mismatches are detected, stored and surfaced for the person to resolve |
| **Consent gate** | Sync would run only after explicit consent — the privacy boundary |
| **Off-thread** | Push and pull run in a background worker, never blocking the UI |

---

## Native configuration

| Concern | Effect |
| --- | --- |
| **Database encryption** | SQLCipher on iOS and Android, with a keychain/keystore-held passphrase |
| **Native HTTP** | Requests use the platform's native HTTP layer |
| **Keyboard** | Native resize behavior |
| **Federated auth** | Google and Apple providers with native sign-in |
| **Crash reporting** | Available in all builds, gated by consent |
| **Status bar & edges** | Edge-to-edge layout with safe-area handling on both platforms |

The full set of native capabilities is listed in
[tech-stack.md](tech-stack.md#native-device-capabilities-capacitor-plugins).

---

## Example data flow: logging a time entry

```text
1. The person logs time on the tracking screen (or stops a timer)
2. The page's Sandbox calls the corresponding repository
3. The repository maps the entity via the ORM and writes to the encrypted local database
      → soft-delete and version fields are maintained automatically
4. The write is committed
5. Signals update the UI from local state; statistics and projections recompute
```

Every step is local. Nothing in this flow waits for, or talks to, a server.

---

## Testing

| Tier | What it covers |
| --- | --- |
| **Unit** (Karma + Jasmine) | Around 240 spec files — statistics and projection math, calculators, converters, life expectancy, ORM transformers, validators, Sandboxes and services, on a fake database connection |
| **Integration** | Backup round-trips and the migration schema, run on a real SQLite engine |
| **End-to-end** (Playwright) | The backup export and re-import through a real file download and file picker |

New or changed isolated business logic ships with specs as part of the definition of done.

---

## Build, versioning & release

- **Web build → native sync.** The Angular build output is synced into the native iOS and Android
  projects with the Capacitor CLI.
- **API client generation.** A typed API client is generated from the backend's OpenAPI spec and
  never edited by hand, so the app and backend stay in lock-step.
- **Automated versioning.** A pre-commit step bumps the version and syncs it into the native
  Android and iOS project files.
- **Store builds** are produced with the standard Android and iOS toolchains and released through
  Google Play and App Store Connect.

See [tech-stack.md](tech-stack.md#native-tooling--quality) for the toolchain details.
