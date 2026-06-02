# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A zero-build, single-file Progressive Web App for family trip planning. The entire application lives in `family-trip-planner.html` — no bundler, no package manager, no build step. React 18.2.0 is loaded from CDN.

## Running the App

Open `family-trip-planner.html` directly in a browser, or serve the directory with any static server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/family-trip-planner.html
```

No install step. No build step. Changes to `family-trip-planner.html` take effect on page reload.

## Architecture

**Everything is in `family-trip-planner.html`** — one `<script>` tag containing the full React app as a single IIFE. React is pulled from CDN via `<script>` tags in `<head>`. There is no module system, no imports, no TypeScript.

### Data Model

All state lives in React and is persisted to `localStorage`:

- `trips_index` — array of all trips, each with shape `{id, name, acc[], flt[], car[], act[]}`
- `active_trip_id` — currently selected trip ID
- Legacy keys (`acc`, `flt`, `car`, `act`) are migrated on first load

Entry shapes:
- **Accommodation** (`acc`): `{id, type, name, platform, checkIn, checkOut, address, contact, url, parking, kitchen, breakfast, totalCost, costPerNight, confirmation, notes}`
- **Flight** (`flt`): `{id, type, airline, flightNo, from, to, depDate, depTime, arrDate, arrTime, cost, bookingRef, url, notes}`
- **Car** (`car`): `{id, type, company, carType, pickupLocation, dropoffLocation, pickupDate, dropoffDate, cost, confirmation, url, notes}`
- **Activity** (`act`): `{id, type, name, category, date, duration, location, cost, url, notes}`

### Key Components (all defined in the single script)

- `App` — root; owns all state, dispatches to `setTrips`/`setActiveId`
- `TableView` — sortable/filterable tabular display
- `CalendarView` — month grid with entry chips
- `GanttView` — horizontal timeline chart
- `EntryModal` — add/edit form for all four entry types; validates required fields and date logic
- `ImportModal` — handles PDF, image, `.ics`, and pasted-text imports; calls `claudeExtract`
- `GmailBridgeModal` — Gmail label scanning workflow
- `BackupModal` — JSON export/import
- `TripManagerModal` — create, rename, delete, switch trips
- `PDFPreviewModal` — print-formatted HTML export
- `FInput`, `FSelect`, `FToggle`, `FTextarea` — thin form field wrappers

### AI Extraction

`claudeExtract(apiKey, content, type)` at the top of the script calls:
- Endpoint: `https://api.anthropic.com/v1/messages`
- Model: `claude-sonnet-4-20250514`
- The user supplies their own Anthropic API key at runtime (stored in `localStorage` as `claude_api_key`)

### PWA

`sw.js` implements a cache-first service worker that caches all four static files (`family-trip-planner.html`, `manifest.json`, `icon-192.svg`, `icon-512.svg`) under the cache name `trip-planner-v1`. Bump the cache name in `sw.js` when deploying breaking changes.

### Validation Rules

Required fields by type: Accommodation → `name, checkIn, checkOut`; Flight → `airline, from, to, depDate`; Car → `company, pickupLocation, pickupDate`; Activity → `name, date`. Date logic: check-out after check-in, arrival not before departure, drop-off not before pick-up.

## Deployment

Drop the four files (`family-trip-planner.html`, `manifest.json`, `sw.js`, `icon-*.svg`) onto any static host (GitHub Pages, Netlify, etc.). No build pipeline needed.
