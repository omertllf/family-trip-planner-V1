# Family Trip Planner — App Documentation

**Last updated:** June 2026  
**Deployed at:** https://omertllf.github.io/family-trip-planner-V1/family-trip-planner.html  
**Source file:** `family-trip-planner.html` (single-file React + Firebase PWA)

---

## 1. Functionality Overview

### Entry Types
The app tracks four booking categories, each with a dedicated form and table view:

| Type | Key Fields | Cost Field |
|------|-----------|------------|
| **Accommodation** | Name, Platform, Check-In/Out, Address, Confirmation | `totalCost` + auto `costPerNight` |
| **Flight** | Airline, Flight#, From/To airports, Dep/Arr datetime, Booking Ref | `cost` |
| **Car Rental** | Company, Car Type, Pickup/Dropoff location+date, Confirmation | `cost` |
| **Activity** | Name, Category, Date, Duration, Location | `cost` |

### Views
- **Table View** — sortable, column-draggable (All tab), duplicate detection (⚠️ badge on same date/name), search. **Click any row to open the detail card.**
- **Detail Card** — full read-only view of a single entry with Edit/Delete actions and a Back button.
- **Calendar View** — monthly grid; entries appear on their start-through-end date range; click to edit.
- **Gantt / Timeline View** — horizontal bar chart sorted by start date; click bars to edit.

### Cost Summary
Header always shows running totals: Stays / Flights / Cars / Activities / Grand Total.

### Gap Detection
Automatically warns if there is a night gap between consecutive accommodation check-out and next check-in.

### Trips
Multi-trip support. Each trip is an independent data container (acc/flt/car/act arrays). Trip switcher in the header. Full rename/delete/create via ⚙️ Trip Manager.

### Dark / Light Mode
☀️/🌙 toggle in the toolbar. Preference persisted to localStorage/Firestore. Default: dark. Full CSS variable system — switching is instantaneous with no flicker.

---

## 2. Data & Import

### Manual Entry
`+ Add` button → type picker (if on All tab) → form modal with field validation.

### AI Import (file/image/text)
`⬆️ Import` → choose method:
- **Photo / Screenshot** — JPG/PNG uploaded, sent to Claude API (`claude-sonnet-4-20250514`) as base64 image.
- **PDF Confirmation** — PDF uploaded, sent as base64 document.
- **.ics Calendar File** — parsed client-side; user selects type per event before import.
- **Paste Text / Email** — raw text sent to Claude API for extraction.

Claude returns JSON matching the entry schema. Result is pre-filled in the edit modal for review before saving.

### Gmail Import (Copy-Paste Bridge)
`📧 Gmail` → select a label → **Send Request** sends a structured extraction prompt to Claude in this chat (or opens `claude.ai/new` in a new tab). Claude scans the label and returns a `gmail_import` JSON block. User copies the full response and pastes it back into the **Paste Response** step.

Parser handles three response formats:
1. Fenced code block: ` ```gmail_import { "entries": [...] } ``` `
2. Raw JSON object: `{ "entries": [...] }`
3. Bare array: `[...]`

After parsing, a preview screen lets the user select which bookings to import.

**Known Gmail label IDs (hardcoded in `KNOWN_LABELS`):**

| Label Name | Label ID |
|-----------|---------|
| 2026 trip to USA | `Label_3699931737615885849` |
| Italy summer vacation | `Label_5603701076709231285` |
| Romania | `Label_6679335078370083063` |
| Greece Olympus | `Label_8098878853189728542` |
| Paris | `Label_8820736268900932572` |
| Inbox | `INBOX` |

### Backup / Restore
`💾 Backup` → download all trips as `.json` (v2 format). Restore accepts v2 format and legacy v1 single-trip format.

---

## 3. Architecture

### Stack
- **React 18.2** — UMD build via CDN, no build step. All JSX written as `h(type, props, ...children)` calls.
- **Firebase Firestore** (compat SDK 12.14) — cloud persistence at `users/{uid}/data/{key}`.
- **Firebase Auth** (compat SDK 12.14) — Google OAuth via `signInWithPopup`. No redirect flow (unreliable on iOS Safari / ITP).
- **Anthropic API** — `api.anthropic.com/v1/messages`, model `claude-sonnet-4-20250514`, called client-side for all AI extraction.
- **PWA** — `manifest.json` + `sw.js` service worker. `apple-touch-icon` + theme-color meta.
- **GitHub Pages** — static hosting. Deploy: edit → GitHub Desktop → commit → push → ~60s propagation.

### Single-File Structure (in order)
```
<head>
  CDN scripts: React, ReactDOM, Firebase (app/firestore/auth)
  Firebase init + global db/auth/googleProvider

<body>
  <div id="root">
  <script> IIFE containing:
    Constants & config (COLORS, PLATFORMS, KNOWN_LABELS, REQUIRED, FIELD_LABELS)
    Utility functions (safeNum, fmt$, nights, fmtD, cityOf, typeIcon, typeLabel, matchesSearch, validate)
    Constructor functions (newAcc, newFlt, newCar, newAct, newTrip)
    Storage (storeGet, storeSet, storeGetCloud)
    ICS parser (parseICS, icsToEntry)
    File helpers (toB64, toText)
    AI extraction (claudeExtract)
    Print HTML builder (buildPrintHTML)
    UI primitives (Overlay, MH, R2, FInput, FSelect, FToggle, FTextarea)
    GmailBridgeModal
    EntryModal
    ImportModal + ICSRow
    BackupModal
    TripManagerModal
    PDFPreviewModal
    DetailCard
    TableView
    CalendarView
    GanttView
    App (root component)
    ReactDOM.createRoot(...).render(h(App))
  Service worker registration
```

### Data Model

```
localStorage / Firestore key: "trips_index"
Value: Trip[]

Trip {
  id: string          // "trip_" + timestamp
  name: string
  acc: Accommodation[]
  flt: Flight[]
  car: Car[]
  act: Activity[]
}
```

Entry IDs are `Date.now()` integers (or `Date.now() + Math.random()` for batch imports).

### State Management
All state lives in the `App` component. No external state library.

Key state variables:

| Variable | Type | Purpose |
|----------|------|---------|
| `trips` / `activeTripId` | array / string | Full trip list and active trip |
| `tab` | string | `"all" \| "accommodations" \| "flights" \| "cars" \| "activities"` |
| `view` | string | `"table" \| "calendar" \| "gantt"` |
| `modal` | object / null | `{ isNew, item, det[] }` — entry being edited |
| `detailEntry` | object / null | Entry open in detail card view |
| `darkMode` | boolean | UI theme; persisted |
| `user` / `authLoading` / `loaded` | mixed | Auth and data loading gates |
| `imp` | `true \| "backup" \| false` | Import / backup modal state |

### Persistence Strategy
1. **Write:** every mutation calls `storeSet(key, value)` — writes to both `localStorage` AND Firestore `users/{uid}/data/{key}`.
2. **Read on load:** `storeGetCloud` first (Firestore), falls back to `localStorage`.
3. **Real-time sync:** `onSnapshot` on `trips_index` keeps the UI in sync across devices.

**Critical:** `onSnapshot`'s first callback serves as the initial data load. Do not run a separate `storeGetCloud` fetch in the same effect — it causes a race condition that overwrites freshly-saved data.

### Theme System
CSS custom properties on `:root` (dark) and `body.light-mode` (light). All component colors reference variables.

| Variable | Dark value | Light value |
|----------|-----------|-------------|
| `--bg` | `#0f172a` | `#f1f5f9` |
| `--surface` | `#1e293b` | `#ffffff` |
| `--surface2` | `#334155` | `#e2e8f0` |
| `--border` | `#1e293b` | `#e2e8f0` |
| `--text` | `#e2e8f0` | `#0f172a` |
| `--text2` | `#94a3b8` | `#334155` |
| `--text3` | `#64748b` | `#64748b` |
| `--text4` | `#475569` | `#94a3b8` |
| `--primary` | `#6366f1` | `#4f46e5` |
| `--input-bg` | `#0f172a` | `#f8fafc` |
| `--overlay-bg` | `rgba(0,0,0,.75)` | `rgba(0,0,0,.4)` |

Entry type accent colors (`COLORS` map) are fixed hex and intentionally not theme-variable — they are brand colors that work on both themes.

---

## 4. Known Issues & Constraints

| Issue | Status |
|-------|--------|
| `signInWithPopup` may be blocked by some browsers/extensions | Fallback to `signInWithRedirect` on `auth/popup-blocked` error — not yet implemented |
| Service worker caches aggressively — hard refresh needed after deploy | Known GitHub Pages behavior |
| Some modals (Gmail, Backup, Import, PDF Preview) may have residual non-CSS-var colors | Styling pass pending |
| Gmail copy-paste bridge requires user to manually copy Claude's response | By design; direct API removed (see decision log) |

---

## 5. Decision Log

### Gmail Direct API — Removed
A direct browser-based Gmail API integration was built and tested (using Google Identity Services + `gmail.readonly` OAuth scope). It was rolled back because:

- Requires a separate Google Cloud OAuth Client ID to be hardcoded in the source.
- The `gmail.readonly` scope is **sensitive** — production use beyond 100 test users requires Google OAuth verification, brand review (2–3 days), and a potential annual security assessment (~$500/yr).
- For Testing mode (personal use), users see an "unverified app" interstitial on first consent, which is confusing for family members.
- The copy-paste bridge achieves the same result with zero infrastructure overhead and works today.

The Gmail MCP approach (Claude reads Gmail natively within the chat session) remains the cleanest path for direct access.

---

## 6. Firestore Security Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/data/{key} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Paste into Firebase Console → Firestore → Rules.

---

## 7. Deployment Checklist

1. Edit `family-trip-planner.html`
2. Syntax check:
   ```
   node -e "require('acorn').parse(require('fs').readFileSync('family-trip-planner.html','utf8').match(/\(function\(\)\{[\s\S]*\}\)\(\);/)[0],{ecmaVersion:2020})" && echo OK
   ```
3. Copy to GitHub repo via GitHub Desktop
4. Commit + Push
5. Wait ~60 seconds
6. Hard-refresh (Ctrl+Shift+R) to bypass service worker cache
