# QB Agency Dashboard — Action-First Redesign

## Overview

Redesign the QB Agency Dashboard from a data-heavy analytics tool into an action-first CRM and commission tracking dashboard for a 2-person insurance sales team. The current app suffers from information overload, confusing navigation, and unclear terminology. The redesign reorganizes around three clear purposes: daily workflow (Home), client management (Clients), and financial tracking (Money).

## Goals

- Reduce cognitive load by giving each view a single clear purpose
- Add CRM capabilities: task/reminder system and activity logging
- Make retention alerts more prominent, actionable, and trackable
- Preserve all existing commission calculation logic and Airtable integration
- Keep the single-file HTML/CSS/JS architecture with no build system

## Non-Goals

- Per-user data separation (shared data, 2-person team)
- Push notifications (reminders live inside the dashboard only)
- Replacing Airtable for data entry (team prefers Airtable for adding new records)
- Mobile-first redesign (not requested)

## Architecture

### Tech Stack (unchanged)

- Single-file HTML/CSS/JS (`index.html`)
- CDN dependencies: PapaParse 5.4.1, Chart.js 4.4.1, SheetJS 0.18.5
- Data source: Airtable API (primary), CSV/XLSX upload (fallback), demo data

### Airtable Schema Changes

Two new linked tables in the existing Airtable base:

**Tasks Table**
| Field | Type | Description |
|-------|------|-------------|
| Task ID | Auto-number | Unique identifier |
| Client | Link to Clients | Associated client record |
| Description | Single line text | What needs to be done |
| Due Date | Date | When the task is due |
| Completed | Checkbox | Whether the task is done |
| Created From | Single select | "manual" or "alert" — tracks origin |
| Created At | Date/time | When the task was created |

**Activities Table**
| Field | Type | Description |
|-------|------|-------------|
| Activity ID | Auto-number | Unique identifier |
| Client | Link to Clients | Associated client record |
| Type | Single select | "call", "follow-up", "note", "task-completed" |
| Description | Long text | Details of the interaction |
| Timestamp | Date/time | When the activity occurred |

These tables are separate from the clients table to keep Airtable clean and allow tasks/activities to have their own lifecycle. Table IDs and field IDs will be hardcoded (matching the existing pattern for the clients table). The actual Airtable table/field IDs must be created and captured before implementation.

## Tab Structure

The current 3 tabs (Overview, Year-to-Date, All-Time Totals) are replaced with:

| Tab | Purpose | Answers |
|-----|---------|---------|
| **Home** | Daily workflow | "What do I need to do right now?" |
| **Clients** | Client management & CRM | "What's going on with this client?" |
| **Money** | Commission analytics | "How are we doing financially?" |

## Home Tab

### KPI Bar
Compact horizontal row at the top with four headline numbers:
- Total Active Clients
- Active Policies
- Total Earned Commission
- This Month's Commission

### Tasks Widget
Primary feature, displayed prominently below KPIs:
- Shows all open tasks sorted by due date
- Each task displays: client name, description, due date
- Urgent tasks (overdue or due today) are visually highlighted
- Actions per task: mark complete, view client
- Completing a task automatically logs an activity on the linked client
- "Add Task" button for manually creating tasks

### Priority Alerts
Below tasks, replacing the current 4-category alert toggle system:
- Single sorted list by urgency:
  1. Payment failures (red)
  2. Upcoming drafts within 7 days (amber)
  3. Recently lapsed policies <60 days (gray)
- Each alert shows: summary count + "Create Task" or "View Client" action
- Alerts that have been turned into tasks are visually dimmed/marked

## Clients Tab

### Search Bar
Full-width search at top. Instant filtering by name, phone, or carrier as user types.

### Filter Chips
Single-click toggle chips below search:
- **Status filters**: Active, Pending (matches "Pending First Payment", "Pending Requirement", and "Payment Failed - Lapse Warning"), Lapsed, All
- **Carrier filters**: AIG, RNA, AHL, AMAM, Transamerica
- Multiple chips can be active simultaneously

### Client List
Simplified table with fewer default columns than current:
- Name, Carrier, Status, Monthly Premium, Days Until Draft
- Sortable by clicking column headers
- Click any row to open the client detail panel

### Client Detail Panel
Replaces the current edit modal. Implemented as a right-side slide-in drawer (simplest approach for the single-file constraint):

**Info Section**
- All existing fields: name, phone, carrier, plan type, status, monthly premium, draft date, date of sale, notes, retention notes
- Auto-calculated fields: annual premium, advance commission, total commission, remaining commission
- Edit inline, syncs to Airtable on save

**Activity Log**
- Chronological list of all interactions with this client
- Each entry: type icon, description, timestamp
- Types: call, follow-up, note, task-completed
- "Add Activity" button to log a new interaction

**Tasks**
- Open tasks for this specific client
- Create new task directly from the detail panel
- Complete tasks inline

## Money Tab

### Summary KPIs
Same compact row style as Home:
- YTD Earned Commission
- All-Time Earned Commission
- Total Advance Paid
- Total Remaining

### YTD / All-Time Toggle
Single toggle that switches all sections below between YTD and All-Time views. Consolidates the current separate YTD and All-Time tabs.

### Monthly Breakdown (primary)
Table showing each month:
- Month, Earned Commission, Policy Count
- Current month highlighted
- This is the most important analytics view per team feedback

### By Carrier
Bar chart showing commission totals per carrier (AIG, RNA, AHL, AMAM, Transamerica). Respects the YTD/All-Time toggle.

### Cumulative Trend
Single line chart showing cumulative commission over time. Respects the YTD/All-Time toggle.

## Data Flow

```
Airtable (clients + tasks + activities)
    ↓ fetch on load / sync
normalize → calculate commissions → render tabs
    ↓ on edit/task complete/activity log
PATCH/POST back to Airtable
```

1. On load: fetch clients, tasks, and activities from Airtable (paginated)
2. Normalize client data (existing `norm()` function, unchanged)
3. Calculate commissions (existing `calcCommission()`, unchanged)
4. Render active tab
5. On user actions (edit client, complete task, log activity): update local state + sync to Airtable

## Commission Logic

All existing commission calculation logic is preserved unchanged:
- Rates table keyed by `"Carrier|PlanType"`
- Quick Pay carriers (AMAM, RNA) earn on Date of Sale
- Others earn on First Draft Date
- Annual Premium = Monthly × 12
- Advance Commission = Annual × advance_rate × commission_rate
- Total Commission = Annual × commission_rate
- Remaining = Total − Advance

## UX Simplifications

- **Fewer columns by default** in client list (5 vs current 9+)
- **Single alert list** sorted by urgency instead of 4 toggle categories
- **One analytics view** with a toggle instead of two separate tabs
- **Filter chips** instead of dropdown selects
- **Inline search** instead of column-based sorting as primary navigation
- **Clearer labels** — "Money" instead of "Year to Date", task descriptions instead of raw status codes

## File Upload & Demo Data

- CSV/XLSX upload remains supported as a data source
- Demo data updated to include sample tasks and activities
- Settings UI for Airtable token unchanged

## Testing

- Verify all commission calculations produce identical results to current app
- Test Airtable sync for clients, tasks, and activities (CRUD)
- Test task creation from alerts
- Test activity auto-logging on task completion
- Test filter chips and search across client list
- Test YTD/All-Time toggle on Money tab
- Cross-browser check (Chrome, Safari, Firefox)
