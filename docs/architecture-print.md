# Print Architecture

The print constructor on the website turns a life calendar into a file you can print at home or
at a copy shop. It spans three parts: the constructor inside the website, a print slice in the
backend API, and a separate renderer service that builds the files. The preview in the browser
and the file from the server come out of **the same sheet generator at the same version**, so
the file matches the proof the person approved.

> This document describes printing at the level of **roles, patterns and pipeline stages**
> rather than concrete file or module names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#print).

---

## What a person gets

| Aspect | What it means |
| --- | --- |
| **Sources** | Your own calendar, the one you published from the app, after signing in on the site; or an empty grid, with no account at all. A calendar opened by someone else's link can be previewed and tried on, but not printed. |
| **Paper sizes** | A4 and Letter for a home printer, A3 and Tabloid for a copy shop |
| **Look** | Two designs with ready-made styles, palettes or your own colors, cell shape, paper tone, header and footer texts |
| **Legend** | Lists milestones or milestone groups, in the order and position you choose |
| **Pages after the sheet** | Optional pages that list every milestone, in columns, when the legend has no room for all of them |
| **Sheets** | Several sheets per calendar — for the wall, for the desk, as a gift — kept in the browser with their own settings and undo history |
| **The download** | A set of files: a PDF of every page, a 300 dpi picture of each page and a short PDF printing guide |

---

## End-to-end flow

```text
Website (browser)
  calendar from the API  ─or─  an empty grid
        │
        ▼
  sheet snapshot + settings ──▶ shared sheet generator ──▶ HTML preview
        │                         (runs in the browser)      (sandboxed frame)
        │  drafts and undo history in local storage
        ▼
  proof ──▶ "Generate and download"
        │  snapshot · settings · guide items · generator version
        ▼
Backend API — print slice
  sign-in check · limits · render slot
        │
        ▼
Renderer service (Azure Container Apps)
  Node service ── same generator, exact version ──▶ HTML
        │
        ▼
  Gotenberg (headless Chromium) ──▶ PDF · page pictures · guide PDF
        │
        ▼  multipart response: manifest + one part per file
Backend API
  dedicated Blob Storage · generation log
        │
        ▼
Website — download screen ──▶ short-lived HTTPS links ──▶ browser download
```

Everything before the final button runs in the browser. The server is involved only when the
person asks for the files.

---

## The sheet generator

The generator is a **shared TypeScript package** used by both the website and the renderer.

- **Pure code, no framework.** It has no Angular and no DOM: a sheet is a pure function from a
  snapshot and settings to an HTML string. The package build allows only the ES2022 library, so a
  browser API slipping in breaks the build rather than the server render.
- **User text is isolated.** Every piece of text a person typed is escaped and isolated from the
  text around it. The preview shows the result in a sandboxed frame with scripts disabled.
- **Designs are data.** A design is a manifest plus CSS that only sets tokens. The settings panel
  is built from the manifest — its controls, ranges and defaults — so the code never names a
  design, and a new design does not need new code.
- **One version on both sides.** The package follows semver, and the website sends its version
  with every generation. The renderer builds files only with its own exact version and refuses
  any other, and the website then asks the person to reload the page, which loads the matching
  generator.
- **Proven identical in browser and Node.** A fixed set of sheets — every ready-made style of every
  design, an empty grid, list pages, a legend of groups — has SHA-256 digests of its snapshot,
  HTML and response on record. A browser spec and a Node test of the built package both check
  against them, so the two builds cannot drift apart unnoticed.
- **Static fonts.** Chromium embeds a variable font in a PDF as Type 3, which print-shop
  preflight checks reject, so the sheet ships static weights of Outfit and Inter (300, 400, 500).
- **Release.** A version tag runs lint and the full test suite, then publishes the package to a
  private package registry, from which the renderer installs it.

---

## The constructor on the website

| Screen | Role |
| --- | --- |
| **Start** | Your calendars with their sheets, a new sheet, an empty grid, and "Your files" — the history of generated sets with delete |
| **Sheet** | The sheet on a stage with the settings panel; a bottom sheet on a phone |
| **Proof** | The final check and the "Generate and download" button with the remaining allowance |
| **Download** | The files of a set and the printing guide |

- **State in signals.** The section has one state service for the sheet, its settings and its
  history. Preview requests are debounced, a newer request cancels an older one, and repeated
  settings are answered from a cache keyed by a stable hash.
- **Notes with fixes.** The generator reports what will not print as intended — a name cut short,
  a legend that overflows, two milestones in colors too close to tell apart, emoji the fonts do not
  have — and the panel offers a one-step fix or opens the right field.
- **Drafts stay in the browser.** Up to 30 sheets live in local storage, keyed by a short hash of
  the account rather than its id. They hold only the person's choices and typed text: no birth
  date and no milestones. A change is written after a short pause and immediately when the tab is
  hidden or closed.
- **Sign-in only when needed.** An empty grid is generated without an account. A guest who asks
  for a calendar's files goes to sign-in and comes back to the same proof with the same sheet.
- **Printing the web page itself** prints a hint pointing to the file: the browser shrinks the
  page into the printer's margins, and the sheet would no longer pass the ruler check.
- **The printing guide** is built by one pure function from the settings, the design and the
  sheet's measurements: paper size, mode, paper tone, list pages and the grid width to check with
  a ruler — in centimeters, and in inches to a sixteenth on Letter and Tabloid. For a copy shop
  it adds a line to hand over or copy as is. The same items appear on the download screen and in
  the guide PDF.
- **Analytics** sends three events — generation started, ready, file downloaded — only after
  consent, with no sheet content or settings and no set id in the page address.

---

## The print slice of the API

| Operation | Purpose |
| --- | --- |
| **Allowance** | How many files are left, and when the next one frees up |
| **Generate** | Checks access and limits, takes a render slot, calls the renderer, stores the set |
| **Read a set** | The set and its files, for the download screen after a reload |
| **File link** | A short-lived, read-only HTTPS link to one file |
| **History** | The signed-in account's sets, paged |
| **Delete** | Removes a set's files; the record keeps only the date and format, so the limit is unchanged |

- **Limits live in configuration**, so an environment changes them with a restart rather than a
  release. Today: 10 calendar files in any rolling 30 days per account, 3 empty grids a day per
  visitor and per address, and 5 generations an hour per person.
- **A load ceiling** allows 4 renders at once per API instance. A request waits up to 20 seconds
  for a slot, then gets a "busy" answer with a time to retry.
- **Stable refusals.** Every refusal carries an error target — sign-in, one of the limits with its
  retry time, busy, outdated generator, invalid sheet with the generator's codes, gone — which
  the website turns into a clear message and a way out.
- **Short retention.** Sets live in a dedicated storage account without soft delete: a calendar's
  set for 15 days, an empty grid's for one day. Deleting the account deletes its sets.
- **Privacy.** Client addresses are stored only as an HMAC under a per-environment secret, and
  visitor identities are cleared after two days. The sheet itself never goes into logs.

---

## The renderer service

A small **Node** service with one endpoint, deployed next to **Gotenberg**, a document-conversion
server built on headless Chromium.

- **One job.** It accepts a render request from the API only, with a key of at least 32
  characters from Key Vault. It checks the generator version, the empty-grid flag and the product
  (files only; posters are not printed yet), builds the HTML with the generator and asks Gotenberg
  for the files.
- **The set.** A PDF of every page; a 300 dpi picture of each page, taken by showing one page at a
  time — PNG on white paper, JPEG when a paper tone is printed into the file; and the guide PDF,
  typeset with the sheet's own fonts and colors.
- **Gotenberg is pinned and locked down.** The image is pinned by tag and digest, because a newer
  Chromium can move the layout by a fraction of a millimeter and the file has to match the proof.
  Scripts are off, Chromium may open only the files of the request, private addresses are refused,
  unused routes are not served, and a conversion ends after 60 seconds.
- **Logs** are one JSON line per request — status, time, format, pages, file count, codes — with no
  snapshot and no guide text.
- **Speed.** A one-page set takes about 2 seconds; a Tabloid sheet with three list pages, about
  5 seconds and 8 MB.

---

## Infrastructure & delivery

```text
Backend API (App Service)
     │  HTTPS, render key
     ▼
Azure Container Apps — one app, two containers, scales to zero, at most two replicas
  ├─ renderer service   0.25 vCPU · 0.5 GiB
  └─ Gotenberg          0.75 vCPU · 1.5 GiB
     ▲
     │  images pulled with a managed identity
Azure Container Registry
```

| Concern | Approach |
| --- | --- |
| **Ingress** | Only from the API's outbound addresses, and only with the key |
| **Storage** | A separate storage account for print files |
| **Cost** | Consumption plan with scale to zero; a monthly budget with alerts |
| **IaC** | The same Bicep templates as the rest of the backend, applied after a `what-if` preview |
| **CI** | Tests run against a real Gotenberg on every pull request; a push to the main branch builds both images, tags them with the commit and deploys staging as one revision; production runs the images already proven on staging, without a rebuild |
| **Release order** | A new generator version is published and pinned in the renderer before the website starts sending it |

---

## Price

Files of your own calendar are free until December 1, 2026; the website states the conditions
after that date. The empty grid stays free.
