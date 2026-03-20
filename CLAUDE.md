# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QB Agency Dashboard is a single-file insurance agency CRM and commission tracking tool. The entire application lives in `index.html` — HTML, CSS, and JavaScript are all inline. There is no build step, bundler, or package manager.

## How to Run

Open `index.html` directly in a browser, or serve it with any static file server:
```
open index.html
# or
python3 -m http.server 8000
```

## Architecture

**Single-file app** (~1350 lines) with three CDN dependencies loaded via `<script>` tags:
- **PapaParse** — CSV parsing for file import
- **Chart.js** — bar/line/doughnut charts for commission analytics
- **SheetJS (xlsx)** — `.xlsx` file reading (imports the "Client List" sheet)

**Data flow:** File upload (CSV/XLSX) or Airtable API fetch → `norm()` normalizes rows into client objects → `calcCommission()` computes derived fields → `init()` renders active tab. Tasks and activities are fetched from separate Airtable tables alongside client data.

**Data sources (two modes):**
1. **File import** — user uploads `.csv` or `.xlsx`; parsed client-side
2. **Airtable sync** — fetches from a hardcoded Airtable base/table using a Personal Access Token stored in `localStorage` (`AT_TOKEN`). Edits are pushed back via PATCH/POST. Field IDs (e.g., `fldFOieTf6s33Zh5H`) are hardcoded.

**Airtable tables:**
- **Clients** — `tblsUWdPEPGmCtKFO` (main client/policy data)
- **Tasks** — `tblSZHvD7FfzxrqZi` (CRM tasks with field ID map `TF`)
- **Activities** — `tbl2xuC4F1Kqd9Xwn` (activity log with field ID map `AF`)

**Commission logic** (`rates` object): Commission rates are keyed by `"Carrier|PlanType"` with two factors: `a` (advance rate) and `c` (commission rate). Advance commission = annual premium × a × c. Total commission = annual premium × c. "Quick pay" carriers (AMAM, RNA) earn on approval date; others earn on first draft date.

**Three main tabs:**
- **Home** — KPIs (active clients, pending, pipeline value, tasks due), task list with add/complete actions, priority alerts (draft reminders, payment failures, lapsed policies) with "create tasks from alert" capability
- **Clients** — search bar, filter chips (status + carrier), sortable client table, slide-in detail drawer for viewing/editing client info and activity log
- **Money** — commission KPIs, YTD/All-Time toggle, monthly breakdown table, carrier cards, commission trend charts (bar + cumulative line)

**Key global state:** `clients` array (all client objects), `tasks` array, `activities` array, `activeFilters` object (`{statuses, carriers, search}`), `moneyView` ('ytd'|'alltime'), `drawerIdx` (drawer edit target), `editIdx` (modal edit target), `hasUnsaved` flag, `dataSource` ('airtable'|'prebaked'|'file').

## Key Functions

- `norm(data)` — normalizes raw imported/fetched rows into uniform client objects, maps carrier/plan name variants
- `calcCommission(c)` — computes annual premium, advance/total/remaining commission, earned date, days-until-draft
- `init()` — master render entry point: shows dashboard, sets data source badge, calls `renderTab()`
- `renderTab(tab)` — dispatches to the correct tab renderer (home/clients/money)
- `renderHome()` — renders Home tab KPIs, calls `renderTasks()` and `renderAlerts()`
- `renderTasks()` — renders task list with complete/add actions
- `renderAlerts()` — builds priority alerts with urgency tiers (draft proximity, payment failures, lost policies)
- `renderClients()` — builds filter chips and calls `rTable()`
- `renderMoney()` — renders Money tab KPIs, monthly table, carrier cards, and charts based on `moneyView`
- `rTable()` — renders the client table with search/filter/sort
- `openDrawer(idx)` / `saveDrawer()` / `closeDrawer()` — slide-in client detail drawer with inline editing, activity log, and Airtable sync
- `openNewClient()` / `saveNewClient()` — modal for adding new clients
- `saveEdit()` — saves edits from modal (existing client via drawer uses `saveDrawer()`)
- `createTask(clientId, name, desc, due, from)` — creates a task in Airtable and local state
- `completeTask(idx)` — marks task complete in Airtable, logs activity
- `logActivity(clientId, name, type, desc)` — creates activity entry in Airtable
- `handleCompleteTask(idx)` — UI handler for task completion with re-render
- `createTasksFromAlert(alertIdx)` — generates tasks from a priority alert
- `openAddTask()` — opens prompt to create a manual task
- `toggleFilter(key, val)` — toggles a filter chip and re-renders client table
- `filterByStatus(status)` — sets status filter and switches to Clients tab
- `setMoneyView(view, btn)` — switches between YTD and All-Time in Money tab
- `fetchAirtable()` / `syncAirtable()` — Airtable client data integration with pagination
- `fetchTasks()` / `fetchActivities()` — Airtable tasks/activities fetch with pagination

## Conventions

- CSS uses custom properties defined in `:root` (navy, blue, green, red, etc.)
- Currency formatting via `$v()` helper
- Carrier name normalization maps in `cMap`; plan type normalization in `pMap`
- Status values: Active, Pending First Payment, Pending Requirement, Payment Failed - Lapse Warning, Lapsed, Cancelled, Declined
- `EXCL` array lists statuses excluded from active commission calculations
- CSS classes: `.chip` (filter chips), `.drawer-overlay` / `.drawer` / `.drawer-section` (client detail drawer), `.activity-entry` / `.activity-icon` (activity log entries)
- Edit modal is used for new client creation; existing clients use the slide-in drawer
