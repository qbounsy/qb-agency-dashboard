# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QB Agency Dashboard is a single-file insurance agency commission tracking tool. The entire application lives in `index.html` — HTML, CSS, and JavaScript are all inline. There is no build step, bundler, or package manager.

## How to Run

Open `index.html` directly in a browser, or serve it with any static file server:
```
open index.html
# or
python3 -m http.server 8000
```

## Architecture

**Single-file app** (~1000 lines) with three CDN dependencies loaded via `<script>` tags:
- **PapaParse** — CSV parsing for file import
- **Chart.js** — bar/line/doughnut charts for commission analytics
- **SheetJS (xlsx)** — `.xlsx` file reading (imports the "Client List" sheet)

**Data flow:** File upload (CSV/XLSX) or Airtable API fetch → `norm()` normalizes rows into client objects → `calcCommission()` computes derived fields → `applyData()` renders all views.

**Data sources (two modes):**
1. **File import** — user uploads `.csv` or `.xlsx`; parsed client-side
2. **Airtable sync** — fetches from a hardcoded Airtable base/table using a Personal Access Token stored in `localStorage` (`AT_TOKEN`). Edits are pushed back via PATCH/POST. Field IDs (e.g., `fldFOieTf6s33Zh5H`) are hardcoded.

**Commission logic** (`rates` object, line ~332): Commission rates are keyed by `"Carrier|PlanType"` with two factors: `a` (advance rate) and `c` (commission rate). Advance commission = annual premium × a × c. Total commission = annual premium × c. "Quick pay" carriers (AMAM, RNA) earn on approval date; others earn on first draft date.

**Three main tabs:**
- **Overview** — KPIs, action alerts (draft reminders, payment failures, lapsed policies), filterable/sortable client table, carrier/status charts
- **Year to Date** — monthly earned commission table, YTD carrier cards, trend charts
- **All-Time Totals** — lifetime aggregates, cumulative charts, plan type breakdown

**Key global state:** `clients` array (all client objects), `allRows` (raw Airtable backup), `hasUnsaved` flag, `editIdx` (modal edit target).

## Key Functions

- `norm(data)` — normalizes raw imported/fetched rows into uniform client objects, maps carrier/plan name variants
- `calcCommission(c)` — computes annual premium, advance/total/remaining commission, earned date, days-until-draft
- `applyData()` — master render entry point: populates filters, renders active tab
- `renderTab(tab)` — renders KPIs + charts + table for the given tab (overview/ytd/alltime)
- `rAlerts()` — builds action alerts with urgency tiers (draft proximity, payment failures, lost policies)
- `rTable()` — renders the client table with filtering and sorting
- `openEdit(idx)` / `saveEdit()` — modal edit with live commission preview and Airtable sync
- `fetchAirtable()` / `syncAirtable()` — Airtable integration with pagination

## Conventions

- CSS uses custom properties defined in `:root` (navy, blue, green, red, etc.)
- Currency formatting via `$v()` helper
- Carrier name normalization maps in `cMap`; plan type normalization in `pMap`
- Status values: Active, Pending First Payment, Pending Requirement, Payment Failed - Lapse Warning, Lapsed, Cancelled, Declined
- `EXCL` array lists statuses excluded from active commission calculations
