# Family Trip Planner — Project Memory

## Project Identity

- **App:** Family Trip Planner (single-file PWA)
- **Deployed URL (primary):** `https://familytripplanner.vercel.app/family-trip-planner.html`
- **Deployed URL (GitHub Pages):** `https://omertllf.github.io/family-trip-planner-V1/family-trip-planner.html`
- **Source file:** `family-trip-planner.html` (single file — all HTML, CSS, JS inline)
- **Stack:** React 18 (UMD, no build step), Firebase Firestore + Auth (compat SDK 12.14.0), GitHub Pages + Vercel

---

## Architecture

| Layer | Detail |
|-------|--------|
| **React** | UMD build via CDN, `React.createElement` (no JSX), all in one `<script>` tag |
| **State** | Firestore `onSnapshot` as source of truth when signed in; localStorage fallback |
| **Auth** | Google Auth via `signInWithPopup` (primary) + Email/Password via `AuthScreen` |
| **Data path** | `households/{householdId}/data/trips_index` — shared per household |
| **Household resolve** | On login: query `households WHERE allowedEmails array-contains email`, pick oldest. Create new only if none found. |
| **Persistence** | `storeSet()` writes both localStorage and Firestore simultaneously |
| **PWA** | `sw.js` + `manifest.json` + `icon-192.svg` (separate files, not inline) |

---

## Data Model

```
trips_index: Trip[]

Trip {
  id: "trip_" + timestamp
  name: string
  acc: Accommodation[]
  flt: Flight[]
  car: CarRental[]
  act: Activity[]
  expenses: Expense[]
}

Expense {
  id: number (timestamp)
  date: string (YYYY-MM-DD)
  category: "Fuel"|"Groceries"|"Restaurants"|"Shopping"|"Parking"|"Tickets"|"Other"  ← mandatory
  description: string  ← optional
  addedBy: string      ← optional free text (who spent it)
  amount: number (stored in the currency field's currency)
  currency: "USD"|"EUR"|"ILS"
  notes: string
}
// expToUSD() converts expense amounts to USD for totals/display
```

Entry types: `accommodation`, `flight`, `car`, `activity`
Costs stored in **USD**; display currency converted client-side via live rates (open.er-api.com, 1h cache).

---

## Key Components

| Component | Purpose |
|-----------|---------|
| `App` | Root; auth state, trip selection, layout |
| `AuthScreen` | Login/register screen (Google + email/password) |
| `TableView` | Main data view; sortable, drag-reorder columns, duplicate detection |
| `DetailCard` | Single-entry detail view (click row → detail) |
| `CalendarView` | Month grid view |
| `GanttView` | Timeline/Gantt view |
| `EntryModal` | Add/edit modal for all 4 entry types |
| `ExpenseModal` | Add/edit in-trip expense entries |
| `ExpenseSummaryView` | Expenses tab: category/person breakdown + SVG pie chart + full table |
| `PieChart` | SVG pie chart helper used in ExpenseSummaryView |
| `ImportModal` | AI extraction from PDF/image/text/ICS |
| `GmailBridgeModal` | Paste-back Gmail import (no direct MCP from iframe) |
| `BackupModal` | JSON export/import |
| `TripManagerModal` | Create/rename/delete/switch trips |
| `PDFPreviewModal` | iframe print preview |
| `HouseholdModal` | View members, copy/share invite link |

---

## Current UI Theme (Dark Mode Default)

CSS variables via `:root` (dark) and `body.light-mode` (light):

```css
--bg: #0f172a          /* dark slate */
--surface: #1e293b
--surface2: #334155
--border: #1e293b
--text: #e2e8f0
--primary: #6366f1     /* indigo */
--danger: #f87171
--success: #10b981
--input-bg: #0f172a
```

---

## Known Issues / Pending Work

| Item | Status |
|------|--------|
| Residual dark hex values in Gmail/Backup/Import/PDF modals | Present — modals use hardcoded `#0f172a`, `#334155`, `#7c2d12` etc. instead of CSS vars |
| `signInWithRedirect` fallback | Not implemented; only `signInWithPopup` present |

---

## Auth

- **Google Sign-in**: `signInWithPopup` (primary, always)
- **Email/Password**: `createUserWithEmailAndPassword` + `signInWithEmailAndPassword` via `AuthScreen` component
- Both options shown on the login screen; email/password supports register + login toggle

---

## Household Sync

- Data path: `households/{householdId}/data/trips_index`
- On every login, app queries: `households WHERE allowedEmails array-contains <email>` and picks the **oldest** household (lowest `createdAt`). Only creates a new household if none found.
- This ensures all devices for the same user converge on the same household automatically.
- Invite link `?join=householdId` still works and takes priority over the email query.
- Falls back to localStorage-cached `household_id` only if Firestore is unreachable (offline).

---

## In-Trip Expenses

- Stored as `expenses: []` on each Trip object (added to `newTrip()`)
- `expToUSD(exp)` converts amounts from entry's currency to USD for totals
- `EXPENSE_CATS`: Fuel, Groceries, Restaurants, Shopping, Parking, Tickets, Other
- `EXPENSE_COLORS` / `EXPENSE_CAT_ICONS`: maps categories to colors/emojis
- Header "💰 Total" box includes in-trip expenses (`tot + expTotal`)
- Header shows "💳 In-Trip Expenses" box; clicking it navigates to expenses tab
- "💳 Expenses" tab in the nav bar renders `ExpenseSummaryView`
- "＋ Add" button opens type picker (which includes "💸 Expense") or adds expense directly when on expenses tab
- `Expense.category` is **mandatory**; `description` and `addedBy` are optional
- `ExpenseSummaryView` has **By Category / By Person** toggle; pie chart and bar chart update accordingly

---

## Critical Engineering Rules

1. **No `onSnapshot` + separate `storeGetCloud` fetch** — race condition. `onSnapshot` IS the initial load.
2. **`signInWithPopup` always primary** — `signInWithRedirect` fails on iOS Safari (ITP).
3. **Costs always stored in USD** — conversion is display-only via `fmt$()`.
4. **`claudeExtract()`** calls `claude-sonnet-4-20250514` — verify model string is current before using.
5. **Validate with `node --check`** after every edit (extract inline script, check with node).

---

## Deployment Workflow

1. Edit `family-trip-planner.html` (and optionally `CLAUDE.md`, `sw.js`)
2. Commit + push to `main` → Vercel auto-deploys immediately; GitHub Pages deploys ~60s later
3. Firebase authorized domains: both `familytripplanner.vercel.app` and `omertllf.github.io` must be listed

---

## Gmail Integration

- **Architecture:** Paste-back bridge (Claude generates prompt → user runs in separate Claude chat → pastes response back)
- **Why not direct MCP:** Artifact iframes cannot authenticate MCP servers
- **Parse order:** fenced ` ```gmail_import ``` ` block → raw `{"entries":[...]}` JSON → bare array
- **Known labels on file:**

| Label ID | Name |
|----------|------|
| `Label_3699931737615885849` | 2026 trip to USA |
| `Label_5603701076709231285` | Italy summer vacation |
| `Label_6679335078370083063` | Romania |
| `Label_8098878853189728542` | Greece Olympus |
| `Label_8820736268900932572` | Paris |

---

## Firebase Config (public, in-file)

```
Project: family-trip-planner-f1b14
Auth domain: family-trip-planner-f1b14.firebaseapp.com
```

Firestore Security Rules must be managed manually in Firebase Console.
