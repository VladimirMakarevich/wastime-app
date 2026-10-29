# Website Architecture

The Wastime website is an **Angular 21** application built as a **statically prerendered (SSG)
site** with server-side rendering. It carries the landing page, one landing per product feature,
a Markdown-driven blog, a help center with step-by-step guides, a public gallery of published
life calendars, the print constructor, the account area, and the legal and support pages.

> This document describes the site at the level of **roles, patterns and pipeline stages**
> rather than concrete file, route or script names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#website).

---

## Role & goals

The mobile app is where people keep their time diary. The website's job is **discovery and trust**,
so it has to be fast, crawlable and easy to share. It is also where a published life calendar
lives as a web page, and where it becomes a file to print.

| Goal | Architectural consequence |
| --- | --- |
| **SEO / crawlability** | Every route is **prerendered to static HTML** at build time, with canonical URLs, structured data and sitemaps. |
| **Rich sharing** | **Open Graph images** are generated per blog post at build time; every page sets its own OG/Twitter metadata. |
| **Content as code** | The blog and the help center are authored in **Markdown with front-matter**, validated and compiled during the build. |
| **Published calendars** | A life calendar published from the app is viewable by link, and public calendars are browsable in a gallery. |
| **Calendars on paper** | The print constructor builds the sheet in the browser and asks the backend only for the final files. |
| **One identity, one contract** | The site shares **Firebase Authentication** and the **generated API client** with the mobile app. |

---

## Application structure

The site is organized by role and mirrors the mobile app's conventions, so patterns stay
consistent across the two Angular codebases.

| Area | Contents |
| --- | --- |
| **Route-level pages** | Landing, feature landings, calendar gallery and calendar view, print constructor, help center, blog, sign-in, account, legal, press kit, support, contact, not-found |
| **Shared UI components** | Menu, footer, navigation, breadcrumbs, share buttons, the download-app prompt, preloader and similar building blocks |
| **Cross-cutting core** | Services (SEO, auth, locale, sharing, reports), route guards, HTTP interceptors, an internal event bus, and thin adapters over third-party SDKs |
| **Build-time tools** | Pure, unit-tested modules for the content, SEO and prerender pipeline, with the file I/O kept in separate scripts |
| **Sheet generator** | The print sheet generator, kept in the repository as a framework-free package shared with the backend's renderer |
| **App configuration** | The route table plus separate browser and server application configs |

Third-party SDKs (Firebase, the cookie-consent library) are reached only through small adapters
in the core. Feature code never calls them directly, which keeps the code SSR-safe and lets tests
replace the adapters with spies.

---

## Routing & pages

| Route group | Purpose |
| --- | --- |
| **Home / about** | Main landing — what the app is, its principles, features, FAQ and where to download it |
| **Feature landings** | One SEO landing per feature: life calendar, time analysis, life expectancy, the Leftover and Wasted calculators, the money-to-time converter and reflective questions |
| **Calendars** | A filterable gallery of public life calendars, and the page of a single published calendar opened by its access link |
| **Print** | The print constructor — start screen, sheet, proof and download — behind an environment flag |
| **Help** | The help hub, one page per section and one per guide, with previous / next navigation and a table of contents |
| **Blog** | Index, posts, tag pages and pagination |
| **Auth** | Sign-in, sign-up and password recovery |
| **Account** | The user's published calendar, favorites and profile |
| **Legal** | Privacy Policy, Terms of Service, Data Protection and an account-deletion page |
| **Press kit, support, contact** | Brand assets, ways to support the project, and a contact form |
| **Not found** | 404 |

Three guards shape access:

- A **staging-access guard** closes non-production environments to the public.
- A **redirect-authenticated guard** sends signed-in users away from the sign-in and sign-up
  screens.
- An **authenticated guard** protects the account pages that need a signed-in user and sends
  guests to sign-in.

Moved URLs and blog post aliases become **real 301 redirects at the hosting layer**, written into
the host configuration at build time, so search engines see a permanent move rather than a
client-side jump.

---

## Rendering strategy: SSR + prerendering

The site is built as a **prerendered Angular app**:

- **`@angular/ssr` + `@angular/platform-server`** provide server rendering during the build.
- The build prerenders every route from a **generated route list** — the static routes plus
  every blog route derived from the content manifest, so a new post gets its page automatically.
- The build **fails** if a route from the list was not produced, so a broken page cannot ship
  silently.

```text
Generated route list   (static routes + blog routes from the content manifest)
        │
        ▼
Production build   → prerenders each route to static HTML
        │
        ▼
Static HTML + a hydrating Angular app
```

Pages whose content comes from the API — the calendar gallery, a published calendar, the account
area, the print constructor — are prerendered as a **shell**. Static text, SEO metadata and a loading state are in the
HTML; the data is fetched in the browser after hydration. Heavy libraries that touch the DOM are
deferred or split into their own lazy chunk.

### SSR-safety rules

- Dependencies come in through constructor injection; the functional `inject()` is blocked by
  lint everywhere except functional guards and interceptors.
- Browser-only APIs sit behind a platform check, the document comes through Angular's DI token,
  and browser side effects run after render, never in constructors.
- Nothing in the server render path depends on `window`, storage or timers.

---

## SEO

- Every routed page sets its title, description, canonical URL, OG/Twitter tags and JSON-LD
  through **one central SEO service**. Image URLs are absolute and each page has a single `h1`.
- **Indexing is controlled by one environment flag.** With indexing off, pages render
  `noindex, nofollow` and no canonical. The base HTML template is `noindex` and carries no
  canonical, so a fallback path can never pose as the home page.
- A published calendar gets its own title, description and share URL, while its canonical stays
  on the calendar page itself.
- Guides and help sections carry structured data of their own, built through a shared JSON-LD
  service.
- After the production build, a target-aware step emits **sitemaps** (index, pages, blog and
  help), an **RSS feed** and an **`llms.txt`** for the production host, along with `robots.txt`
  and the redirect rules. The staging build gets a `robots.txt` that relies on `noindex` headers
  instead.

---

## Content & blog pipeline

The blog is **content-as-code**. Authors write Markdown with front-matter; a Node pipeline
compiles it into JSON assets and prerender routes.

```text
Markdown posts   (front-matter + per-post assets)
        │
        │  validation      (fails the build on bad content)
        ▼
   Content build
     • gray-matter → parse front-matter
     • marked → Markdown to HTML
     • internal wiki-link rewriting + SEO href normalization
     • word count, reading time, pagination
        │
        ├──▶ posts manifest        (metadata for all published posts)
        └──▶ per-post JSON          (full post + rendered HTML)
        │
        ├──▶ per-post Open Graph images   (canvas)
        ├──▶ redirect rules               (moved URLs and post aliases → host 301s)
        └──▶ generated prerender route list   (paginated blog + tag indexes)
```

At runtime the blog reads the generated manifest and per-post JSON. There is **no CMS or database**
in the request path, which keeps the site static and cheap to host.

The pipeline's logic lives in pure modules that are unit-tested with Node's built-in test runner;
the scripts around them only read and write files. A separate, manually run step uses `sharp` to
cut the landing pages' phone showcases out of the mobile app's store screenshots.

---

## Help center

The help center is a second Markdown collection built by the same pipeline: about 70 step-by-step
guides in nine sections — life calendar, Time Diary, calculators, converter, life expectancy,
questions, statistics, Journey and account.

```text
Guide Markdown                          Mobile app repository
(front-matter: order, app version,      Playwright capture suite: runs the app's web build
 related guides)                        in its iPhone look, seeds the data each guide needs
        │                                       │  screenshots
        ▼                                       ▼
  validation + content build  ◀──────  screenshot import (webp in several widths)
        │
        ├──▶ hub, section and guide JSON
        ├──▶ prerender routes and the help sitemap
        └──▶ structured data per guide
```

- Every step of a guide has a screenshot of the iPhone screen, framed by a phone the site draws
  itself. Screenshots are captured by a script, not by hand, so a guide can be recaptured when the
  app changes.
- A section links to the feature landings it belongs to, and each landing links back.
- The mobile app's help sheets link to the full guides on the site.
- Help and blog pages switch through route resolvers, so the new page appears in one step
  instead of an empty frame first.

---

## Print constructor

The print constructor turns your own published calendar, or an empty grid, into a set of files
for printing: a PDF of every page, a 300 dpi picture of each page and a printing guide.

- The sheet is built **in the browser** by a framework-free generator that the backend's renderer
  runs at the same version, so the downloaded file matches the proof on screen.
- Sheets, settings and undo history stay in the browser's storage; the server sees the sheet only
  when the person asks for the files.
- The final files come from the backend: it checks sign-in and limits, renders through a
  Node + Gotenberg service and returns short-lived download links. "Your files" on the start
  screen keeps the account's sets until they expire.

The full design — the generator, the constructor, the API slice, the renderer and its
infrastructure — is in [Print architecture](architecture-print.md).

---

## Authentication

The site uses **Firebase Authentication** — the same identity system as the mobile app and the
backend, so one account works everywhere. Sign-in supports Google, Apple and email with password.
Apple sign-in on the web uses the popup flow; a redirect flow for iOS Safari is planned. Most of
the site needs no account: signing in is for the account area, favorites, a private calendar and
the files of your own calendar in the print constructor.

---

## Build & deployment

```text
                 ┌───────────────────────────────┐
                 │  Content + Angular source     │
                 └───────────────┬───────────────┘
                                 │ validate → build blog content → OG images →
                                 │ redirects → prerender routes → production build →
                                 │ SEO artifacts (per host)
                ┌────────────────┴─────────────────┐
                ▼                                  ▼
      Azure Static Web Apps                 Firebase Hosting
      production · behind Cloudflare        staging · noindex
```

| Target | Notes |
| --- | --- |
| **Azure Static Web Apps** (production) | Serves `wastime.app` behind Cloudflare. Deploys automatically from a release branch that only fast-forwards from the main branch. Indexing on. |
| **Firebase Hosting** (staging) | A closed pre-production copy, gated and sent with `noindex` headers on every path. |

New sections reach production behind environment flags: the help center and the print
constructor were built and checked on staging, then switched on for `wastime.app` in one release
in October 2026.

---

## Shared with the rest of the system

- **API client** — like the mobile app, the site consumes a **generated Angular client**, built
  from the backend's OpenAPI document for the website, so web ↔ API stays type-safe. See
  [backend architecture](architecture-backend.md#api-contract--client-generation).
- **Firebase identity** — one auth system across app, web and API.
- **Locale negotiation** — the UI is English; a locale service still negotiates the visitor's
  locale and passes it to the API in request headers.
- **Privacy** — analytics cookies wait for consent through a cookie-consent banner, and page
  addresses reach analytics without their query string, campaign tags aside.
- **Themes** — the site has a light and a dark theme with a switcher in the navigation.
- **Conventions** — a shared cross-cutting core (event bus, guards, interceptors, services) keeps
  the two Angular codebases consistent.

---

## Quality & tooling

| Concern | Tooling |
| --- | --- |
| **Linting** | ESLint (`@angular-eslint`), Stylelint for SCSS |
| **Formatting** | Prettier |
| **Unit tests** | Karma + Jasmine in random order, with a **coverage ratchet** — global thresholds sit just below the coverage reached and only move up |
| **Build-time tests** | Node's built-in test runner for the pipeline modules and the built sheet generator |
| **Docs** | Markdown context linting in CI |
| **Git hooks** | Pre-commit lint and format |
| **CI/CD** | GitHub Actions — tests on every pull request, production deploy from the release branch |
