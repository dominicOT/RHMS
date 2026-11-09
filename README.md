# RetailHub

**A full-stack retail platform for a physical store with an online storefront: point of sale, inventory, stock audits, online orders and reporting, all on one database.**

Built for a working multi-category store in Sierra Leone (shoes, bags, clothes, accessories, food and essentials).

> 🔒 The source code is private under a client agreement. This repository describes the project: what it does, how it is built, and the engineering decisions behind it. A walkthrough is available on request.

![Next.js](https://img.shields.io/badge/Next.js_16-000?logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React_19-20232a?logo=react&logoColor=61dafb)
![TypeScript](https://img.shields.io/badge/TypeScript-3178c6?logo=typescript&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169e1?logo=postgresql&logoColor=white)
![Drizzle](https://img.shields.io/badge/Drizzle_ORM-c5f74f?logo=drizzle&logoColor=black)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS_4-06b6d4?logo=tailwindcss&logoColor=white)
![GSAP](https://img.shields.io/badge/GSAP-88ce02?logo=greensock&logoColor=black)

<!-- Add screenshots here, e.g. ![POS](docs/pos.png) ![Dashboard](docs/dashboard.png) -->

---

## Overview

RetailHub is made of two Next.js applications that share one PostgreSQL database:

| App | Who uses it | What it does |
| --- | --- | --- |
| **Management Hub** | Store staff | Point of sale, catalog, inventory, stock takes, online order fulfilment, reports, user management and an audit trail |
| **Storefront** | Customers | Browsing, a swipeable product feed, cart and checkout (pay on delivery), with live stock levels from the store |

```
              ┌──────────────────────┐        ┌──────────────────────┐
  Staff  ───▶ │   Management Hub     │        │     Storefront       │ ◀─── Customers
 (till,       │  POS · Inventory ·   │        │  Shop · Feed · Cart  │
  tablet,     │  Orders · Reports    │        │  · Checkout          │
  desktop)    └──────────┬───────────┘        └──────────┬───────────┘
                         │      PostgreSQL (one schema)  │
                         └──────────────┬────────────────┘
                                        ▼
                     products · variants · stock ledger · sales ·
                     online orders · stock takes · audit log · …
```

**Scale:** about 23k lines of TypeScript, 19 tables, 30+ API endpoints and 8 versioned migrations.

---

## Features

### 🧾 Point of sale
- **Works offline.** If the server can't be reached, the till keeps going: the sale is queued in IndexedDB, a receipt prints with a temporary `OFF-…` number, and the queue syncs (oldest first) once the connection is back. Idempotency keys mean a resent sale can never be recorded twice.
- **Barcode scanning** with USB or Bluetooth scanners in keyboard mode. Scanner keystrokes are caught anywhere on the page and kept away from focused buttons, so a scan can never press "Complete sale". Camera scanning (ZXing) is there as a fallback.
- **Customer-facing display.** A second window shows the customer the live cart and a thank-you screen. It is synced over `BroadcastChannel` without going through the server, and only data the customer may see (names, quantities, prices) is sent.
- **Stock conflicts** at checkout are handled cleanly: if another till or an online order took the last unit, the cashier gets a dialog explaining what is short instead of a failed sale.
- PDF receipts, reprints and PIN-pad sign-in for quick cashier changes.

### 📦 Inventory and catalog
- Products with **variant matrices** (for example colour × size), with stock and reorder levels per variant.
- **Single stock ledger:** every change (sale, intake, adjustment, online order) goes through one function that updates the quantity and writes the ledger row in the same transaction.
- **Stock takes** run as a workflow: *in progress → pending approval → approved/cancelled*. Counted and expected quantities are reconciled, and each variance is tagged with a reason (damage, loss or count error).
- **Bulk CSV import** with a downloadable template, plus **label printing** with QR codes.
- Product photos come from the device camera (HEIC is converted in the browser), are compressed, and are stored and served from the database.

### 🛒 Online orders
- Storefront orders arrive in the Hub in real time.
- Confirming an order is an atomic step: stock rows are locked (`SELECT … FOR UPDATE`), shortages are checked, the order becomes a sale and the items leave the shelf, or the whole confirmation is refused with an itemised shortage message. Cancelling a confirmed order reverses all of it.

### 📊 Dashboard and reports
- **Live dashboard** pushed over Server-Sent Events: revenue, sales, low-stock alerts, peak trading hours and an activity feed, with no polling.
- Reports for **revenue and profit**, **stock movement**, **variance** and **shift totals**, with in-store and online sales compared side by side. Reports can be exported to CSV.

### 🔐 Access control and accountability
- **Three roles** (Super Admin, Inventory Keeper and Cashier), each with a permission matrix. The API enforces the permissions; route guards in the UI only control what each role sees.
- Sessions are signed JWTs (`jose`) in httpOnly cookies, and **the user is re-read on every API request**, so revoking access or changing a role applies immediately.
- **Append-only audit log** written inside the same transaction as the change it records, so a change and its audit entry are always saved together or not at all.
- A **read-only catalog API** (`/api/v1/products`) for partner apps. It uses rotating API keys and never exposes cost prices or reorder levels.

### 🛍️ Storefront
- Shop, product pages, cart and pay-on-delivery checkout, all with **live stock levels** from the store.
- A **swipeable product feed** with reactions and comments that works without customer accounts (each browser gets an anonymous id). The order is a weighted random shuffle (Efraimidis–Spirakis) that favours the categories a visitor chose and liked without hiding the rest.
- Checkout is idempotent and checks stock on the server before an order is accepted.

### ✨ Design and UX
- Custom design system built on shadcn/ui and Tailwind CSS 4, with light and dark themes.
- Motion with GSAP: page transitions, FLIP animations, staggered reveals and count-up numbers. All of it respects `prefers-reduced-motion`.
- A command palette (⌘K), responsive layouts from till screen to phone, and self-hosted fonts.

---

## Engineering highlights

- **Money is stored as integers.** Every amount is an integer number of cents (Leones), so rounding errors can't build up.
- **Correct under concurrency.** Several tills plus the storefront can compete for the same last unit. Row locks inside transactions keep stock from going negative and prevent selling the same unit twice.
- **Retries are safe.** Idempotency keys on POS checkout and storefront orders mean flaky connections and double clicks can't create duplicates.
- **Offline first where it matters.** The till keeps working through power or network cuts, and synced sales arrive with their original times.
- **Typed end to end.** TypeScript in strict mode, Zod validation at every API boundary, React Query for server state, Zustand for local state, and versioned SQL migrations with Drizzle.
- **Written for real conditions.** The design assumes unreliable connectivity, cheap barcode scanners, shared till terminals and staff with varied technical experience.

---

## Tech stack

| Layer | Tools |
| --- | --- |
| Framework | Next.js 16 (App Router, Route Handlers), React 19 |
| Language | TypeScript (strict) |
| Data | PostgreSQL, Drizzle ORM and drizzle-kit migrations |
| State and data fetching | TanStack Query, TanStack Table, Zustand |
| Forms and validation | React Hook Form, Zod |
| UI | Tailwind CSS 4, shadcn/ui (Base UI), Lucide icons, Recharts |
| Motion | GSAP |
| Auth | jose (JWT), bcrypt |
| Devices and output | ZXing (barcodes), jsPDF (receipts), IndexedDB, BroadcastChannel, Server-Sent Events |
| Hosting | Render (web service + managed PostgreSQL) |

---

## Status

Built for and deployed to the client. Source code is available for review under NDA on request.

---

## Credits

- **Dominic** ([dominicOT](https://github.com/dominicOT)): architecture, the Management Hub (POS, inventory, orders, reports, auth and access control), the shared database schema, and storefront features including the product feed.
- **Bella** ([Bellaadamasesay] (https://github.com/Bellaadamasesay/)): built the storefront, from the first commit through the shop, product pages, cart, checkout and orders.
