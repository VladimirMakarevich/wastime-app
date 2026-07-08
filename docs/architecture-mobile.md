# Mobile App Architecture

The Wastime mobile app is the **primary product**: a single Angular + Ionic codebase compiled
to native **iOS** and **Android** binaries with Capacitor, and also runnable on the web. It is
**offline-first** — every feature works with no network, and cloud sync is an optional layer
on top.

For versions and package details, see [tech-stack.md](tech-stack.md#mobile-app).

---

## Design principles

The architecture is a direct expression of the product's promises:

| Product promise | Architectural consequence |
| --- | --- |
| **Works fully offline** | All reads/writes hit a local **encrypted SQLite** database; the network is never on the critical path. |
| **Privacy by default** | Cloud sync is opt-in and **consent-gated**; data leaves the device only after explicit user consent. |
| **Your data, your rules** | Local **export/import (JSON)**, soft-deletes, and full local ownership of the datastore. |
| **A tool, not a life center** | Feature modules are independent and lazily loaded; the app stays small and fast. |

---

## Layered overview

```text
┌───────────────────────────────────────────────────────────────┐
│  Presentation                                                  │
│    pages/ (feature screens)   components/ (shared UI)          │
│    Ionic components · Angular router · Quill · ECharts         │
└───────────────────────────┬───────────────────────────────────┘
                            │ services / message bus
┌───────────────────────────┴───────────────────────────────────┐
│  Application & cross-cutting  (@core/)                          │
│    services · auth · interceptors · guards · message-bus       │
│    states · i18n · remote-config · dark-mode · statistics      │
└───────────────────────────┬───────────────────────────────────┘
                            │ repositories (storages)
┌───────────────────────────┴───────────────────────────────────┐
│  Data  (data/)                                                 │
│    storages/ (repositories) → custom ORM → entities            │
│    migrations · soft-delete · sqlite.service                   │
│    sync/ (optional, consent-gated — in progress)               │
└───────────────────────────┬───────────────────────────────────┘
                            │ SQLite driver
┌───────────────────────────┴───────────────────────────────────┐
│  Persistence & native  (Capacitor)                             │
│    @capacitor-community/sqlite (native, encrypted)             │
│    sql.js / WASM (web)   ·   secure storage · preferences      │
│    device plugins: notifications, camera, filesystem, …        │
└───────────────────────────────────────────────────────────────┘
```

Dependencies flow **downward only**: presentation depends on `@core`, `@core` depends on the
data layer, the data layer depends on the SQLite driver. Nothing lower reaches up.

---

## Presentation layer

### Feature modules (`src/app/pages/`)

Each product capability is an independent, lazily loaded feature area:

| Module | Product feature |
| --- | --- |
| `time-trackers/` | Time Tracking (Meaningful vs Wasted), topics & groups |
| `timers-v2/` | Timers with pauses; converts sessions into time entries |
| `edit-time/` | Manual time-entry creation and editing |
| `calendars/` | Life in Weeks / Life in Days, milestones & groups |
| `life-expectancy/` | Life-expectancy setup (country-based or manual) |
| `calculators/` | The 5 calculators (Live, Wasted, Basic, 24H, Reverse) |
| `converters/` | Money → Time converter and accounts |
| `statistics/` | Stats & insights, trend slides |
| `journey/` | Roadmap / time-journey stories |
| `questions/` | Reflective questions |
| `onboarding/` | First-run setup flow |
| `home/`, `main/`, `account/`, `auth/` | Shell, dashboard, profile, sign-in |

### Shared UI (`src/app/components/`)

Reusable presentational building blocks: `charts/`, `color-picker/`, `date-select/`,
`time-select/`, `profile-image-cropper/`, `searchable-select/`, `user-avatar/`,
`skeleton-page/`, `loading/`, `swipe-item/`, `init-error-panel/`, and more. UI is built on
**Ionic** components with **ECharts** for visualization and **Quill** for rich text.

---

## Application & cross-cutting layer (`@core/`)

`@core` holds everything that is not a screen and not persistence.

| Area | Responsibility |
| --- | --- |
| `services/` | Cross-app services: `dark-mode`, `network`, `statistics`, `life-expectancy`, `haptics`, `log`, `locale-info`, `screen`, `data-backup`, `remote-config`, `i18n`, `infrastructure` |
| `auth/` | Firebase authentication (`auth.service`), `secure-token-storage.service` (keychain/keystore-backed tokens), guest mode |
| `interceptors/` | HTTP pipeline: `auth.interceptor` (attach tokens), `network.interceptor` (connectivity), `performance-http.interceptor` (Firebase Performance traces) |
| `message-bus/` | Decoupled in-app event bus (`decorators`, `factories`, `messages`, `services`) so modules communicate without direct references |
| `guards/` | Route guards |
| `states/` | Global runtime state (e.g. `device-config.global`) |
| `pipes/`, `directives/`, `form-validation/`, `forms/` | Angular building blocks |
| `errors/`, `enums/`, `extensions/`, `utils/` | Shared primitives |

### Internal message bus

Rather than wiring feature modules directly to one another, cross-cutting events flow through
a typed **message bus** (`@core/message-bus`). This keeps features decoupled — for example, a
timer completing can publish an event that statistics and sync consume independently.

---

## Data layer (`data/`) — the offline-first core

This is the heart of the app. All application data lives in a **local SQLite database**,
accessed through a small custom ORM and a repository ("storage") layer.

```text
storages/*.storage.ts        Repository per aggregate (topics, time-items, milestones, …)
      │  uses
data/orm/                    Custom decorator-based ORM (TypeORM-inspired)
      │  maps
data/enteties/*.entity.ts    Entity definitions (entity.base.ts + per-feature entities)
      │  persisted via
sqlite.service.ts            SQLite connection management
tracked-sqlite-connection.ts Connection wrapper with change tracking
      │  runs on
@capacitor-community/sqlite  Native encrypted SQLite  /  sql.js (WASM) on web
```

### Custom ORM (`data/orm/`)

A lightweight, decorator-based mapping layer (inspired by TypeORM) tailored for SQLite:

- **Decorators** (`decorators/`) — `entity/`, `columns/`, plus `Versioned` (optimistic
  concurrency version field), `GeneratedMetadataArgs`, `EmbeddedMetadataArgs`, and
  `ValueTransformer`.
- **Transformers** (`transformers/`) — bridge type mismatches between JS and SQLite, e.g.
  `boolean-number.transformer` (SQLite has no boolean type) and `nullish.transformer`.
- **Utilities** — `MixedList`, `ObjectUtils`.

The `Versioned` decorator is deliberate: per-row versioning underpins **optimistic concurrency
and conflict detection** for the sync engine.

### Entities (`data/enteties/`)

Domain models mapped to tables, all extending `entity.base.ts`. They mirror the product
features — e.g. `topic.entity`, `time-item.entity`, `timer-state.entity`,
`life-milestone.entity`, `life-expectancy.entity`, `converter-account.entity`, and the
calculator entities (`live-`, `wasted-`, `basic-`, `twenty-four-`, `reverse-calculation`).

### Repositories (`data/storages/`)

Each entity has a `*.storage.ts` repository extending `storage.base.ts`, exposing typed CRUD
and query operations to the application layer. Cross-cutting data concerns live alongside:

- `soft-delete.service.ts` — rows are **soft-deleted** (tombstoned), which is required for
  correct sync and for undo.
- `raw-database.storage.ts` — low-level access for export/import and diagnostics.
- Feature repositories such as `timer-v2-repository.ts` and `calendar-config-repository.ts`.

### Migrations (`data/migrations/`)

A versioned schema migration system (`migration.service.ts` + `upgrades/`) evolves the local
database as the app updates, so existing users' data is upgraded safely on device.

### Initialization (`initialize.app.service.ts`)

Bootstraps the datastore on launch: opens the (encrypted) SQLite connection, runs pending
migrations, and prepares repositories. Startup has guard timeouts and an
`init-error-panel/` fallback so a corrupt or slow init is recoverable rather than fatal.

---

## Offline-first & synchronization

### Local-first writes

Every user action writes to local SQLite **immediately** and the UI updates from local state.
There is no "saving to server" spinner on the critical path — the network is never required.

### Sync engine (optional, consent-gated — *in progress*)

The `data/sync/` module implements an **outbox-based synchronization engine** for optional
multi-device sync. It is architected but **not yet enabled** (`sync.types.ts` is gated behind
a `TODO: enable when BE ready`), so the current shipping app is local-only. The design:

```text
Local write ──▶ Outbox (sync-outbox.repository)
                     │  batched push
                     ▼
             sync.worker.ts  (Web Worker, off the UI thread)
                     │  HTTP (typed API client)
                     ▼
              Backend API  ◀──▶  PostgreSQL
                     │  pull changes (since cursor)
                     ▼
        Conflict detection (Versioned rows + expectedVersion)
                     │
        ┌────────────┴─────────────┐
   auto-merge                sync-conflict.repository
                             → sync-conflict-viewer (user resolves)
```

Key building blocks already scaffolded in `data/sync/`:

| Component | Role |
| --- | --- |
| `sync-outbox.repository` | Queue of local changes pending push (outbox pattern) |
| `sync-state.repository` | Sync cursor / last-synced sequence per entity |
| `idempotency-cache.repository` | Idempotency keys so retried pushes don't duplicate |
| `sync-conflict.repository` + `sync-conflict-viewer.service` | Persist and surface conflicts for user resolution |
| `sync-outbox-producer.service` | Turns local writes into outbox items |
| `sync-consent.service` | **Privacy gate** — sync only runs after explicit consent |
| `sync-activity-log.service`, `sync-metrics.service`, `sync-status.service` | Observability of sync |
| `sync-error-handler.service` | Retriable vs terminal error handling |
| `sync.worker.ts` | Runs push/pull off the main thread |

The design goals are explicit in the code: **idempotent, versioned, conflict-aware, and
consent-first** — matching the product's privacy stance.

### Network awareness

`@core/services/network.service` and `network.interceptor` make the app connectivity-aware
(via `@capacitor/network`), so sync (when enabled) only activates on suitable connections and
the UI degrades gracefully offline.

---

## Native configuration

Native behavior is declared in `capacitor.config.ts` (in the app repository). Highlights:

| Setting | Value / effect |
| --- | --- |
| `appId` / `appName` | `com.wastime.waste.app` / `Wastime` |
| **SQLite encryption** | Enabled on both iOS and Android; dedicated iOS keychain prefix |
| **CapacitorHttp** | Enabled — native HTTP layer |
| **Keyboard** | Native resize; style switched dynamically by `DarkModeService` |
| **Firebase Auth** | Providers `google.com`, `apple.com`; native auth enabled |
| **Crashlytics** | Enabled, including during development |
| **Status bar** | Overlays the web view (edge-to-edge), with safe-area handling |

Native capabilities used across features (notifications, camera, file picker, filesystem,
share, haptics, keep-awake, in-app-review) are listed in
[tech-stack.md](tech-stack.md#native-device-capabilities-capacitor-plugins).

---

## Example data flow: logging a time entry

```text
1. User taps "log" in  pages/time-trackers  (or stops a timer in  timers-v2)
2. Feature service calls  time-item.storage  (repository)
3. Repository maps the entity via the custom ORM and writes to local SQLite
      → soft-delete/versioning fields maintained automatically
4. sqlite.service commits; tracked-connection records the change
5. UI updates from local state (instant); statistics recompute
6. (When sync is enabled) sync-outbox-producer enqueues the change;
   sync.worker pushes it after consent + connectivity checks
```

Notice that steps 1–5 are entirely local and synchronous with the user — the cloud (step 6)
is additive and never blocks the experience.

---

## Build, versioning & release

- **Web build → native sync:** Angular builds to `www/`, then `npx cap sync` copies it into
  the native Android/iOS projects.
- **API client generation:** `npm run generate:api-client` regenerates the typed Angular
  client into `projects/waste-api-client` from the backend's `openapi/swagger.json`.
- **Automated versioning:** a pre-commit hook bumps the patch version; release scripts sync
  that version into Android `build.gradle` and iOS `project.pbxproj`
  (`versionCode = major*10000 + minor*100 + patch`).
- **Store builds:** Android `./gradlew bundleRelease`; iOS `xcodebuild … archive`.

See [tech-stack.md](tech-stack.md#native-tooling) for the full toolchain.
