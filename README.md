<div align="center">

<img src="docs/assets/cover.png" alt="Wastime — Life Calendar &amp; Time Diary. The app icon next to a phone showing one year of life as a grid of days." width="100%" />

### Life Calendar &amp; Time Diary

*See your time. Live on purpose.*

[![Get it on Google Play](https://img.shields.io/badge/Google%20Play-get%20the%20app-34a853?style=for-the-badge&logo=googleplay&logoColor=white)](https://play.google.com/store/apps/details?id=com.wastime.waste.app)
[![App Store](https://img.shields.io/badge/App%20Store-coming%20soon-555?style=for-the-badge&logo=apple&logoColor=white)](#download)
[![Website](https://img.shields.io/badge/wastime.app-website-e8446a?style=for-the-badge)](https://wastime.app/)

`Offline-first` · `No account needed` · `No ads` · `Free core`

[What it is](#what-is-wastime) · [Screenshots](#screenshots) · [Features](#whats-inside) ·
[Principles](#principles) · [Architecture](#how-its-built) · [Tech stack](#technology-stack)

</div>

---

## What is Wastime?

Most days feel full, yet by evening it is hard to say what they were spent on. **Wastime** is an
offline life calendar and time diary for anyone who wants to know where their hours really go.
You write down an activity, decide whether it was meaningful or wasted, and see what your
ordinary weeks add up to over a year. Then you can step back and look at your whole life laid
out in weeks.

The app turns that vague feeling into a record and leaves the conclusions to you. Everything
personal stays on your phone and works offline, with no account and no ads.

> **About this repository.** This is the public overview of the Wastime project. It describes
> the product, its architecture and the technology behind it. The application source code lives
> in separate private repositories.

---

## Screenshots

<div align="center">

<table>
  <tr>
    <td><img src="docs/assets/phones/screenshot-1.webp" width="200" alt="Time Diary entries split into Wasted and Meaningful, with a total and a ten-year projection of the current pace" /></td>
    <td><img src="docs/assets/phones/screenshot-2.webp" width="200" alt="A new Time Diary entry — duration, topic, date and an optional note" /></td>
    <td><img src="docs/assets/phones/screenshot-3.webp" width="200" alt="Life calendar — one year of life as a grid of days with colored milestones" /></td>
    <td><img src="docs/assets/phones/screenshot-4.webp" width="200" alt="Topic overview — averages, totals and a projection over the next ten years" /></td>
  </tr>
  <tr>
    <td><img src="docs/assets/phones/screenshot-5.webp" width="200" alt="Life expectancy calculator built on short questions about habits" /></td>
    <td><img src="docs/assets/phones/screenshot-6.webp" width="200" alt="Money to time converter — the hours of life behind a purchase" /></td>
    <td><img src="docs/assets/phones/screenshot-7.webp" width="200" alt="Yes-or-no guide that helps tell whether a habit is really wasted time" /></td>
    <td><img src="docs/assets/phones/screenshot-8.webp" width="200" alt="Statistics — the current year in weeks and days, lived and remaining" /></td>
  </tr>
</table>

</div>

---

## What's inside

### 📅 Your life in weeks

The life calendar draws your life as a grid of weeks, with the ones already lived filled in and
the rest still open. You can zoom into a single year as days or step out to whole years.
Milestones mark the events that shaped you, and **Age Stories** gives each year of your life a
page of its own. When you want to share it, the calendar can become a web page that is public,
open only to people with the link, or visible to you alone.

### ⏳ Time Diary

The **Time Diary** is the everyday level, a diary of your hours. You write down where they go,
by hand or with a stopwatch that keeps its pauses and can split a long session into parts, and
sort each entry into Wasted or Meaningful yourself, since only you can say what was wasted.
Every entry belongs to a topic, and a topic can carry a time goal, a deadline or a consistency
target. Statistics show totals, daily averages and your best days, weeks and months, with a
chart of how the two kinds of time balance out and a projection of your pace over a year, or
ten.

### 💸 What a purchase costs in hours

Tell the converter what you earn, before or after tax, and any price turns into the hours of work
behind it. A history adds up what your spending came to in working time and what share of your
expected life it took.

### 🧮 Time calculators

| Calculator | What it answers |
| --- | --- |
| **24-hour** | How much of the day is still free once everything already in it is counted |
| **Planning** | How much daily time a goal asks of you, working backwards from a deadline |
| **Wasted** | What a habit that stopped giving anything back adds up to |
| **Leftover** | Roughly how much time you have left with someone you love, from how often you meet and both of your life expectancies |
| **Basic** | Totals for any recurring activity |

### ❤️ A life expectancy to build on

The calendars and several calculators rest on your expected lifespan. It can come from life
tables for the countries you choose, from a number of your own, or from ten short questions about
sleep, stress, exercise and similar habits. The result is a statistical estimate for reflection
that cannot predict how long you will live or replace medical advice, and the app says so before
the first question.

### 🧭 Room to reflect

**Journey** is eight chapters about time and the thinking behind the app, and it reopens where you
left off. **Questions** offers prompts about life, death, meaning and time, and keeps your
answers. A short yes-or-no guide helps when you are unsure whether a habit really belongs on the
wasted side.

### 🔒 Yours to keep

A local backup exports your data to a file you can restore later. Publishing a life calendar is
the one thing that needs an account and a connection.

---

## Principles

| Principle | What it means in practice |
| --- | --- |
| **Your data stays on your device** | Diary entries, calendars and calculations live in an encrypted database on the phone and work without a network or an account. A calendar leaves the device only when you publish it, and analytics runs only if you agree to it. |
| **Built for short visits** | You open the app to write something down or look at the picture, and then you put it away. There is nothing to scroll endlessly, no ads, and the only notifications are reminders you schedule yourself. |
| **The conclusions are yours** | Wastime counts and draws. What a wasted hour means to you, and what to do about it, stays your decision. |

### Pricing

Publishing your life calendar is free three times in any 30 days. **Wastime Pro** removes that
limit. Everything else in the app is free and stays on your device.

---

## Download

| | |
| --- | --- |
| 🤖 **Google Play (Android)** | [play.google.com/store/apps/details?id=com.wastime.waste.app](https://play.google.com/store/apps/details?id=com.wastime.waste.app) |
| 🍏 **App Store (iOS)** | Coming soon |
| 🌐 **Website** | [wastime.app](https://wastime.app/) — feature pages, blog and published life calendars |

The app is in English.

---

## How it's built

Wastime is three cooperating applications that share one API contract (OpenAPI). The mobile app
is the product itself and runs fully on the device. The backend takes care of the few things
that really need a server, such as publishing a calendar and checking the subscription.

```text
┌──────────────────────────┐            ┌──────────────────────────┐
│  Mobile app              │            │  Website                 │
│  Ionic · Angular ·       │            │  Angular SSR / prerender │
│  Capacitor (iOS/Android) │            │  landing · blog ·        │
│  encrypted SQLite        │            │  calendar gallery        │
└────────────┬─────────────┘            └────────────┬─────────────┘
             │   HTTPS · generated OpenAPI client     │
             │   Firebase ID token                    │
             ▼                                        ▼
        ┌──────────────────────────────────────────────────┐
        │  Backend API — ASP.NET Core 9                    │
        │  publishing · gallery · subscriptions ·          │
        │  in-app messages · moderation                    │
        └──────────────────────────┬───────────────────────┘
                                   │
        ┌──────────────────────────┴───────────────────────┐
        │  Azure — PostgreSQL · Blob Storage · Key Vault · │
        │          Application Insights                    │
        │  Firebase — Auth · Analytics · Crashlytics · …   │
        │  RevenueCat — subscriptions                      │
        └──────────────────────────────────────────────────┘
```

| Document | Contents |
| --- | --- |
| 📱 [Mobile app architecture](docs/architecture-mobile.md) | Layers and page patterns, the offline-first data core, phased startup, content pipeline, subscriptions, native configuration |
| 🔧 [Backend architecture](docs/architecture-backend.md) | Layered .NET design, Firebase-based auth, calendar publishing, RevenueCat integration, Azure infrastructure, release flow |
| 🌐 [Website architecture](docs/architecture-web.md) | Angular SSR and prerendering, the Markdown blog pipeline, SEO artifacts, hosting |
| 🧱 [Technology stack](docs/tech-stack.md) | The full stack with versions and the reasoning behind the key choices |

---

## Technology stack

| Area | Technologies |
| --- | --- |
| **Mobile** | Ionic 8, Angular 21, Capacitor 8, TypeScript 5.9, encrypted SQLite, ECharts 6 |
| **Backend** | ASP.NET Core 9 (C#), Entity Framework Core 9, PostgreSQL |
| **Website** | Angular 21 with SSR / prerendering, Bootstrap 5, Markdown-driven blog |
| **Cloud** | Microsoft Azure — App Service, PostgreSQL Flexible Server, Blob Storage, Key Vault, Application Insights |
| **Platform services** | Firebase (Auth, Analytics, Crashlytics, Performance, Remote Config), RevenueCat |
| **Delivery** | GitHub Actions, Docker, Azure Static Web Apps behind Cloudflare |

See [docs/tech-stack.md](docs/tech-stack.md) for the full breakdown.

---

## Status

Wastime is live on Google Play and in active development. This repository contains documentation
only and is updated as the product changes.

## Contact

Questions, ideas and press requests go through the [contact page](https://wastime.app/contact).
Logos and product images are in the [press kit](https://wastime.app/marketing).

---

## License

The documentation in this repository is licensed under
**[Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](LICENSE)**.
You may read and share it with attribution, but not use it commercially or distribute modified
versions.

The **Wastime** name, logos, screenshots and the application source code are **not** covered by
this license and remain the property of their respective owners. See [LICENSE](LICENSE) for
details.
