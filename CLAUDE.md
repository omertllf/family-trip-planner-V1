# Family Trip Planner — Project Memory

## Project Identity

- **App:** Family Trip Planner (single-file PWA)
- **Deployed URL:** `https://omertllf.github.io/family-trip-planner-V1/family-trip-planner.html`
- **Source file:** `family-trip-planner.html` (single file — all HTML, CSS, JS inline)
- **Stack:** React 18 (UMD, no build step), Firebase Firestore + Auth (compat SDK 12.14.0), GitHub Pages

---

## Architecture

| Layer | Detail |
|-------|--------|
| **React** | UMD build via CDN, `React.createElement` (no JSX), all in one `<script>` tag |
| **State** | Firestore `onSnapshot` as source of truth when signed in; localStorage fallback |
| **Auth** | Google Auth via `signInWithPopup` (primary); redirect only on `popup-blocked` |
| **Data path** | `users/{uid}/data/{key}` — per-user, not household (note: memory mentions household model but file uses per-user path) |
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
}
```

Entry types: `accommodation`, `flight`, `car`, `activity`
Costs stored in **USD**; display currency converted client-side via live rates (open.er-api.com, 1h cache).

---

## Key Components

| Component | Purpose |
|-----------|---------|
| `App` | Root; auth state, trip selection, layout |
| `TableView` | Main data view; sortable, drag-reorder columns, duplicate detection |
| `DetailCard` | Single-entry detail view (click row → detail) |
| `CalendarView` | Month grid view |
| `GanttView` | Timeline/Gantt view |
| `EntryModal` | Add/edit modal for all 4 entry types |
| `ImportModal` | AI extraction from PDF/image/text/ICS |
| `GmailBridgeModal` | Paste-back Gmail import (no direct MCP from iframe) |
| `BackupModal` | JSON export/import |
| `TripManagerModal` | Create/rename/delete/switch trips |
| `PDFPreviewModal` | iframe print preview |

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

> ⚠️ Memory references a "premium UI redesign" with warm palette (`--canvas`, `--coral`, `--sage`, `--horizon:#1E3A5F`, Plus Jakarta Sans). **This is NOT in the current file.** The file uses the original dark theme. Either that redesign was never merged or was done in a separate branch.

---

## Known Issues / Pending Work

| Item | Status |
|------|--------|
| Debug log panel on login screen (exposes user-agent) | Should be removed — **not present in current file** (may already be removed) |
| Residual dark hex values in Gmail/Backup/Import/PDF modals | Present — modals use hardcoded `#0f172a`, `#334155`, `#334155`, `#7c2d12` etc. |
| Household model vs per-user Firestore path | Memory says household model; file uses `users/{uid}/data/` — discrepancy, verify |
| `signInWithRedirect` fallback | Not implemented in file; only `signInWithPopup` present |

---

## Critical Engineering Rules

1. **No `onSnapshot` + separate `storeGetCloud` fetch** — race condition. `onSnapshot` IS the initial load.
2. **`signInWithPopup` always primary** — `signInWithRedirect` fails on iOS Safari (ITP).
3. **Costs always stored in USD** — conversion is display-only via `fmt$()`.
4. **`claudeExtract()`** calls `claude-sonnet-4-20250514` — check if model string is still current before using.
5. **Validate with `node --check`** after every edit (extract inline script, prepend React/Firebase stubs).

---

## Deployment Workflow

1. Edit file locally
2. Copy into GitHub repo via GitHub Desktop
3. Commit + push
4. Wait ~60s → hard-refresh deployed URL

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

---

## Open Questions (resolve before next session)

- [ ] Was the "premium UI redesign" merged? Current file says no.
- [ ] Was the household model (`households/family-trip-planner-household/data/{key}`) ever deployed? Current file uses per-user path.
- [ ] Was the debug log panel ever added and removed, or never added?
