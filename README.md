# Paisa — local-first personal finance PWA

Track expenses and income, budgets, bills/SIPs/EMIs, savings goals and net worth.
Installable on Android/iOS from the browser, works fully offline, and data never leaves the device (until you add sync).

**Stack:** React 19 + Vite (plain JavaScript) · Dexie (IndexedDB) · vite-plugin-pwa (Workbox) · Vitest. No chart library: the charts are hand-written SVG.

## Run it

```bash
npm install
npm run dev        # http://localhost:5173  (also on your LAN via --host)
npm test           # 20 unit/integration tests (fake-indexeddb)
npm run build      # → dist/  (static; deploy anywhere)
npm run preview    # serve the production build
```

**Put it on your phone:** deploy `dist/` to any static HTTPS host (Cloudflare Pages: `npx wrangler pages deploy dist`), open the URL in Chrome/Safari, then use **Add to Home screen**. HTTPS is required for the service worker and `crypto.randomUUID` (localhost is exempt).

## Features

| Area | What it does |
|---|---|
| Transactions | Expense / income / transfer, categories, accounts (bank, cash, wallet, credit card), search across all months, filters, day-grouped list |
| Budgets | Monthly limit per category, warning at 80%, alert when over, "₹X/day left" pacing, list of unbudgeted spending with one-tap "Set limit" |
| Bills & recurring | Daily/weekly/monthly/yearly with intervals and end dates. **Auto-record** (SIPs, EMIs, auto-debits) catches up on missed periods. Manual bills get Mark paid / Skip. Shows monthly committed outflow |
| Goals & net worth | Goals with target date and the required monthly saving. Assets (MF, FD, PPF/EPF, gold, property…) and loans. Net worth = account balances + assets − liabilities |
| Data | JSON backup/restore (merge, not overwrite), CSV export (protected against spreadsheet formula injection), persistent-storage request, erase all |

## Architecture

```
src/
  lib/        pure logic: money (integer paise), dates, finance calculations  ← unit-tested
  db/         Dexie schema, repo (save/remove with sync metadata), seed data
  services/   recurring engine, reminders, backup/CSV, sync client
  components/ UI primitives, charts, forms (one bottom sheet per entity)
  screens/    Home, Transactions, Budgets, Bills, Wealth, Settings
  store.jsx   one live Dexie query → React context (UI auto-updates on any write)
```

Key decisions:
- **Money is stored as integer paise**, so there is no floating-point drift. It is formatted with Indian digit grouping (₹1,23,456) and shortened as K/L/Cr.
- **Dates are `YYYY-MM-DD` strings**, so they are timezone-safe and sort correctly. Monthly rules keep an *anchor day*: a bill on the 31st goes Jan 31 → Feb 28 → Mar 31 instead of drifting.
- **Recurring postings are idempotent.** A posted transaction's id is `rec:<ruleId>:<dueDate>`, so two tabs, a crash mid-run, or two synced devices can never double-post.
- **Seed categories and accounts use fixed ids** (`cat-food`, `acc-bank`), so devices converge after sync instead of creating duplicates.

## Sync-ready design ("local first, sync later")

Every row has `id (UUID)`, `createdAt`, `updatedAt`, `deletedAt` (tombstone) and `deviceId`. Every write also goes into an `outbox` table. The client is already in `src/services/sync.js` and tested. A server only needs to implement:

```
POST /sync   Authorization: Bearer <token>
req: { deviceId, cursor, changes: [{ table, row }] }
res: { cursor, changes: [{ table, row }] }   // rows changed by other devices since cursor
```

Rules: last-write-wins on `updatedAt`, deletes are tombstones, and the cursor is issued by the server (never the device clock). A Hono Worker with one D1 table `rows(user_id, tbl, id, data JSON, updated_at, server_seq)` is enough. When it's ready, enter the URL under Settings → Cloud sync.

## Known limits
- Bill notifications fire only when the app is opened. True background reminders need Web Push from a server, which fits in the same Worker. Due bills always show on Home.
- Each device holds its own data until sync exists, so use Backup/Restore to move data between devices.
