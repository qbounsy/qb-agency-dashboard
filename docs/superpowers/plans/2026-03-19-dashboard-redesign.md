# QB Agency Dashboard — Action-First Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the QB Agency Dashboard from a data-heavy analytics tool into an action-first CRM dashboard with three purpose-driven tabs (Home, Clients, Money), adding task/reminder and activity logging features while preserving all existing commission logic.

**Architecture:** Single-file HTML/CSS/JS app (`index.html`). The entire redesign happens in this one file — replacing existing HTML structure, CSS styles, and JS functions while preserving the core data logic (`norm()`, `calcCommission()`, `fetchAirtable()`, rates table, constants). Two new Airtable tables (Tasks, Activities) will be added alongside the existing Clients table.

**Tech Stack:** Vanilla HTML/CSS/JS, PapaParse 5.4.1 (CDN), Chart.js 4.4.1 (CDN), SheetJS 0.18.5 (CDN), Airtable REST API

**Spec:** `docs/superpowers/specs/2026-03-19-dashboard-redesign-design.md`

---

## Pre-Implementation: Airtable Setup

Before any code changes, the Tasks and Activities tables must exist in Airtable so we have real table/field IDs to hardcode.

- [ ] **Step 1: Create Tasks table in Airtable**

In the Airtable base `appK65HG3vNZebZLu`, create a new table called "Tasks" with these fields:
- `Description` (Single line text)
- `Client` (Link to the existing clients table `tblsUWdPEPGmCtKFO`)
- `Client Name` (Single line text — denormalized for easy display)
- `Due Date` (Date)
- `Completed` (Checkbox)
- `Created From` (Single select: "manual", "alert")
- `Created At` (Date, include time)

Record the table ID and all field IDs.

- [ ] **Step 2: Create Activities table in Airtable**

In the same base, create a new table called "Activities" with these fields:
- `Client` (Link to clients table `tblsUWdPEPGmCtKFO`)
- `Client Name` (Single line text — denormalized for easy display)
- `Type` (Single select: "call", "follow-up", "note", "task-completed")
- `Description` (Long text)
- `Timestamp` (Date, include time)

Record the table ID and all field IDs.

- [ ] **Step 3: Add the new table/field IDs to index.html**

Open `index.html`. At line ~368-370, after the existing Airtable constants, add the new table IDs and field ID mappings. Use the IDs captured from steps 1-2:

```javascript
// Existing (line 368-370):
const AT_BASE='appK65HG3vNZebZLu';
const AT_TABLE='tblsUWdPEPGmCtKFO';
const AT_URL=`https://api.airtable.com/v0/${AT_BASE}/${AT_TABLE}`;

// Add below:
const AT_TASKS_TABLE='tblXXXXXXXXXXXXX';  // Replace with actual ID from step 1
const AT_TASKS_URL=`https://api.airtable.com/v0/${AT_BASE}/${AT_TASKS_TABLE}`;
const AT_ACTIVITIES_TABLE='tblYYYYYYYYYYYYY';  // Replace with actual ID from step 2
const AT_ACTIVITIES_URL=`https://api.airtable.com/v0/${AT_BASE}/${AT_ACTIVITIES_TABLE}`;

// Field ID maps for Tasks table (replace with actual IDs):
const TF={
  description:'fldXXX1',
  client:'fldXXX2',
  clientName:'fldXXX3',
  dueDate:'fldXXX4',
  completed:'fldXXX5',
  createdFrom:'fldXXX6',
  createdAt:'fldXXX7'
};

// Field ID maps for Activities table (replace with actual IDs):
const AF={
  client:'fldYYY1',
  clientName:'fldYYY2',
  type:'fldYYY3',
  description:'fldYYY4',
  timestamp:'fldYYY5'
};
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Airtable table/field IDs for Tasks and Activities"
```

---

### Task 1: Add Tasks & Activities Fetch/Sync Functions

**Files:**
- Modify: `index.html` (JS section, after `fetchAirtable()` at line ~400)

**Context:** The existing `fetchAirtable()` function (line 372-399) fetches client records with pagination. We need equivalent functions for Tasks and Activities, plus global state arrays.

- [ ] **Step 1: Add global state for tasks and activities**

At line 329, where existing globals are declared, add:

```javascript
// Existing line 329:
let clients=[], sortCol=null, sortDir=1, aFilt='upcoming', editIdx=-1, hasUnsaved=false;

// Add after:
let tasks=[], activities=[];
```

- [ ] **Step 2: Add fetchTasks() function**

After the existing `fetchAirtable()` function (which ends around line 400), add:

```javascript
async function fetchTasks(){
  let allRecords=[], offset=null;
  do {
    const url=new URL(AT_TASKS_URL);
    url.searchParams.set('pageSize','100');
    if(offset) url.searchParams.set('offset',offset);
    const resp=await fetch(url,{headers:{'Authorization':`Bearer ${AT_TOKEN}`}});
    if(!resp.ok) throw new Error('Airtable Tasks error: '+resp.status);
    const data=await resp.json();
    allRecords=allRecords.concat(data.records);
    offset=data.offset||null;
  } while(offset);
  return allRecords.map(r=>{
    const f=r.fields||{};
    return {
      _airtableId:r.id,
      description:f[TF.description]||'',
      clientId:(f[TF.client]||[])[0]||null,
      clientName:f[TF.clientName]||'',
      dueDate:f[TF.dueDate]?new Date(f[TF.dueDate]):null,
      completed:!!f[TF.completed],
      createdFrom:f[TF.createdFrom]||'manual',
      createdAt:f[TF.createdAt]?new Date(f[TF.createdAt]):null
    };
  });
}
```

- [ ] **Step 3: Add fetchActivities() function**

```javascript
async function fetchActivities(){
  let allRecords=[], offset=null;
  do {
    const url=new URL(AT_ACTIVITIES_URL);
    url.searchParams.set('pageSize','100');
    if(offset) url.searchParams.set('offset',offset);
    const resp=await fetch(url,{headers:{'Authorization':`Bearer ${AT_TOKEN}`}});
    if(!resp.ok) throw new Error('Airtable Activities error: '+resp.status);
    const data=await resp.json();
    allRecords=allRecords.concat(data.records);
    offset=data.offset||null;
  } while(offset);
  return allRecords.map(r=>{
    const f=r.fields||{};
    return {
      _airtableId:r.id,
      clientId:(f[AF.client]||[])[0]||null,
      clientName:f[AF.clientName]||'',
      type:f[AF.type]||'note',
      description:f[AF.description]||'',
      timestamp:f[AF.timestamp]?new Date(f[AF.timestamp]):null
    };
  });
}
```

- [ ] **Step 4: Add createTask() function**

```javascript
async function createTask(clientAirtableId, clientName, description, dueDate, createdFrom='manual'){
  const fields={};
  fields[TF.description]=description;
  fields[TF.clientName]=clientName;
  fields[TF.dueDate]=dueDate?dueDate.toISOString().split('T')[0]:null;
  fields[TF.completed]=false;
  fields[TF.createdFrom]=createdFrom;
  fields[TF.createdAt]=new Date().toISOString();
  if(clientAirtableId) fields[TF.client]=[clientAirtableId];
  const resp=await fetch(AT_TASKS_URL,{
    method:'POST',
    headers:{'Authorization':`Bearer ${AT_TOKEN}`,'Content-Type':'application/json'},
    body:JSON.stringify({fields})
  });
  if(!resp.ok) throw new Error('Create task error: '+resp.status);
  const r=await resp.json();
  const task={
    _airtableId:r.id,
    description,
    clientId:clientAirtableId,
    clientName,
    dueDate,
    completed:false,
    createdFrom,
    createdAt:new Date()
  };
  tasks.push(task);
  return task;
}
```

- [ ] **Step 5: Add completeTask() function**

```javascript
async function completeTask(taskIdx){
  const t=tasks[taskIdx];
  const resp=await fetch(`${AT_TASKS_URL}/${t._airtableId}`,{
    method:'PATCH',
    headers:{'Authorization':`Bearer ${AT_TOKEN}`,'Content-Type':'application/json'},
    body:JSON.stringify({fields:{[TF.completed]:true}})
  });
  if(!resp.ok) throw new Error('Complete task error: '+resp.status);
  t.completed=true;
  // Auto-log activity
  if(t.clientId){
    await logActivity(t.clientId, t.clientName, 'task-completed', 'Completed: '+t.description);
  }
}
```

- [ ] **Step 6: Add logActivity() function**

```javascript
async function logActivity(clientAirtableId, clientName, type, description){
  const fields={};
  fields[AF.clientName]=clientName;
  fields[AF.type]=type;
  fields[AF.description]=description;
  fields[AF.timestamp]=new Date().toISOString();
  if(clientAirtableId) fields[AF.client]=[clientAirtableId];
  const resp=await fetch(AT_ACTIVITIES_URL,{
    method:'POST',
    headers:{'Authorization':`Bearer ${AT_TOKEN}`,'Content-Type':'application/json'},
    body:JSON.stringify({fields})
  });
  if(!resp.ok) throw new Error('Log activity error: '+resp.status);
  const r=await resp.json();
  activities.push({
    _airtableId:r.id,
    clientId:clientAirtableId,
    clientName,
    type,
    description,
    timestamp:new Date()
  });
}
```

- [ ] **Step 7: Update the init/load flow to fetch all three tables**

Find the existing `init()` function (line ~517) and the Airtable auto-load logic (line ~405). The auto-load currently does:

```javascript
// Current (~line 405):
if(AT_TOKEN){fetchAirtable().then(r=>{allRows=r;applyData();}).catch(e=>{...});}
```

Replace with:

```javascript
if(AT_TOKEN){
  Promise.all([fetchAirtable(), fetchTasks(), fetchActivities()])
    .then(([rows, t, a])=>{allRows=rows; tasks=t; activities=a; applyData();})
    .catch(e=>{console.error(e);document.getElementById('uz').classList.remove('hidden');});
}
```

Also update `syncAirtable()` (line ~436) and `saveSettings()` (line ~167) similarly to re-fetch all three tables. The `saveSettings()` function currently only calls `fetchAirtable()` — it must also fetch tasks and activities so the Home tab populates correctly on first token entry:

```javascript
// In saveSettings(), replace:
fetchAirtable().then(r=>{allRows=r;applyData();});
// With:
Promise.all([fetchAirtable(), fetchTasks(), fetchActivities()])
  .then(([rows, t, a])=>{allRows=rows; tasks=t; activities=a; applyData();});
```

- [ ] **Step 8: Update demo data to include sample tasks and activities**

Find `loadDemo()` (line ~491). After it sets `clients=norm(demoData)`, add sample tasks and activities:

```javascript
tasks=[
  {_airtableId:null,description:'Follow up on policy upgrade',clientId:null,clientName:'Demo Client 1',dueDate:new Date(Date.now()+86400000),completed:false,createdFrom:'manual',createdAt:new Date()},
  {_airtableId:null,description:'Review upcoming draft payment',clientId:null,clientName:'Demo Client 2',dueDate:new Date(Date.now()+3*86400000),completed:false,createdFrom:'alert',createdAt:new Date()}
];
activities=[
  {_airtableId:null,clientId:null,clientName:'Demo Client 1',type:'call',description:'Discussed policy renewal options',timestamp:new Date(Date.now()-86400000)},
  {_airtableId:null,clientId:null,clientName:'Demo Client 2',type:'note',description:'Client requested callback next week',timestamp:new Date()}
];
```

- [ ] **Step 9: Verify fetch works, commit**

Open the app in a browser, confirm it loads client data and doesn't error on tasks/activities fetch. Check the browser console for errors.

```bash
git add index.html
git commit -m "feat: add Tasks and Activities Airtable fetch, create, and sync functions"
```

---

### Task 2: Replace Tab Navigation (Overview/YTD/AllTime → Home/Clients/Money)

**Files:**
- Modify: `index.html` (HTML structure lines ~184-279, JS `switchTab()` at line ~502)

**Context:** The current HTML has three tab buttons (Overview, Year to Date, All-Time Totals) and three `tab-page` divs. We replace these with Home, Clients, Money. The CSS for tabs (`.main-tabs`, `.main-tab`, `.tab-page`) is already defined and reusable.

- [ ] **Step 1: Replace tab buttons**

Find the tab navigation (line ~184-189):

```html
<div class="main-tabs">
  <button class="main-tab active" onclick="switchTab('overview',this)">Overview</button>
  <button class="main-tab" onclick="switchTab('ytd',this)">Year to Date</button>
  <button class="main-tab" onclick="switchTab('alltime',this)">All-Time Totals</button>
</div>
```

Replace with:

```html
<div class="main-tabs">
  <button class="main-tab active" onclick="switchTab('home',this)">Home</button>
  <button class="main-tab" onclick="switchTab('clients',this)">Clients</button>
  <button class="main-tab" onclick="switchTab('money',this)">Money</button>
</div>
```

- [ ] **Step 2: Replace tab page containers**

Remove all three existing tab-page divs (lines ~192-278, the `page-overview`, `page-ytd`, `page-alltime` divs and ALL their contents). Replace with empty skeleton containers:

```html
<!-- TAB 1: Home -->
<div class="tab-page active" id="page-home">
  <div class="kpi-row" id="kpis-home"></div>
  <div class="section">
    <div class="section-hdr">
      <h3>Tasks</h3>
      <button class="btn btn-sm btn-green" onclick="openAddTask()">+ Add Task</button>
    </div>
    <div class="section-body" id="tasksList"></div>
  </div>
  <div class="section">
    <div class="section-hdr"><h3>Priority Alerts</h3></div>
    <div class="section-body" id="alerts"></div>
  </div>
</div>

<!-- TAB 2: Clients -->
<div class="tab-page" id="page-clients">
  <div style="margin-bottom:16px;">
    <input type="text" id="clientSearch" placeholder="Search by name, phone, or carrier..." oninput="rTable()" style="width:100%;padding:12px 16px;border:1.5px solid var(--g300);border-radius:8px;font-size:14px;">
  </div>
  <div id="filterChips" style="display:flex;gap:6px;flex-wrap:wrap;margin-bottom:16px;"></div>
  <div class="section">
    <div class="section-hdr">
      <h3>Clients <span style="font-size:12px;color:var(--g500);font-weight:400;" id="cc"></span></h3>
      <button class="btn btn-sm btn-green" onclick="openNewClient()">+ Add Client</button>
    </div>
    <div class="tw" style="max-height:600px;overflow-y:auto;"><table><thead id="th"></thead><tbody id="tb"></tbody></table></div>
  </div>
</div>

<!-- TAB 3: Money -->
<div class="tab-page" id="page-money">
  <div class="kpi-row" id="kpis-money"></div>
  <div style="display:flex;gap:4px;margin-bottom:16px;">
    <button class="stab active" id="moneyToggleYTD" onclick="setMoneyView('ytd',this)">YTD</button>
    <button class="stab" id="moneyToggleAll" onclick="setMoneyView('alltime',this)">All-Time</button>
  </div>
  <div class="section">
    <div class="section-hdr"><h3>Monthly Breakdown</h3></div>
    <div class="section-body"><div class="tw"><table class="monthly-tbl" id="monthlyTbl"></table></div></div>
  </div>
  <div class="section">
    <div class="section-hdr"><h3>Commission by Carrier</h3></div>
    <div class="section-body"><div id="carrierCards" class="carrier-cards"></div></div>
  </div>
  <div class="section"><div class="section-hdr"><h3>Commission Trend</h3></div><div class="section-body">
    <div class="chart-grid">
      <div class="chart-box"><canvas id="ch-carrier"></canvas></div>
      <div class="chart-box"><canvas id="ch-cum"></canvas></div>
    </div>
  </div></div>
</div>
```

- [ ] **Step 3: Update switchTab() and renderTab()**

Find `switchTab()` (line ~502) and `renderTab()` (line ~510). Replace both:

```javascript
let moneyView='ytd';

function switchTab(tab,btn){
  document.querySelectorAll('.main-tab').forEach(b=>b.classList.remove('active'));
  btn.classList.add('active');
  document.querySelectorAll('.tab-page').forEach(p=>p.classList.remove('active'));
  document.getElementById('page-'+tab).classList.add('active');
  renderTab(tab);
}

function renderTab(tab){
  destroyCharts();
  if(tab==='home') renderHome();
  else if(tab==='clients') renderClients();
  else if(tab==='money') renderMoney();
}

function setMoneyView(view,btn){
  moneyView=view;
  document.querySelectorAll('#page-money .stab').forEach(b=>b.classList.remove('active'));
  btn.classList.add('active');
  renderMoney();
}
```

- [ ] **Step 4: Update applyData()**

Find `applyData()` (called from init). It currently calls `renderTab('overview')`. Change to:

```javascript
// In applyData(), change:
renderTab('overview');
// To:
renderTab('home');
```

- [ ] **Step 5: Verify tabs switch correctly, commit**

Open the app. Confirm clicking Home/Clients/Money switches tabs (content will be empty or broken — that's expected at this stage). Check no JS errors in console.

```bash
git add index.html
git commit -m "feat: replace Overview/YTD/AllTime tabs with Home/Clients/Money"
```

---

### Task 3: Implement Home Tab (KPIs + Tasks + Alerts)

**Files:**
- Modify: `index.html` (JS section — new `renderHome()` function, update `rAlerts()`)

**Context:** The Home tab needs: a KPI bar (4 cards), a tasks widget, and a simplified priority alerts list. The existing `renderOverview()` (line ~544) has KPI logic we can reuse. The existing `rAlerts()` (line ~698) has alert logic we'll simplify.

- [ ] **Step 1: Write renderHome() function**

Replace the existing `renderOverview()` function (line ~544-576) with `renderHome()`:

```javascript
function renderHome(){
  const act=eligible();
  const ytd=ytdEligible();
  const thisMonth=ytd.filter(c=>{
    const d=c.earnedDate;
    return d&&d.getMonth()===new Date().getMonth();
  });
  const totalEarned=ytd.reduce((s,c)=>s+c.advComm,0);
  const monthEarned=thisMonth.reduce((s,c)=>s+c.advComm,0);

  document.getElementById('kpis-home').innerHTML=`
    <div class="kpi"><div class="lbl">Active Clients</div><div class="val">${act.length}</div></div>
    <div class="kpi good"><div class="lbl">Active Policies</div><div class="val">${clients.filter(c=>c.status==='Active').length}</div></div>
    <div class="kpi"><div class="lbl">Total Earned (YTD)</div><div class="val">${$v(totalEarned)}</div></div>
    <div class="kpi"><div class="lbl">This Month</div><div class="val">${$v(monthEarned)}</div></div>
  `;

  renderTasks();
  renderAlerts();
}
```

- [ ] **Step 2: Write renderTasks() function**

```javascript
function renderTasks(){
  const openTasks=tasks.filter(t=>!t.completed).sort((a,b)=>{
    if(!a.dueDate) return 1;
    if(!b.dueDate) return -1;
    return a.dueDate-b.dueDate;
  });
  const container=document.getElementById('tasksList');
  if(!openTasks.length){
    container.innerHTML='<div class="empty">No open tasks. Create one or let alerts generate them.</div>';
    return;
  }
  const today=new Date(); today.setHours(0,0,0,0);
  container.innerHTML=openTasks.map((t,i)=>{
    const origIdx=tasks.indexOf(t);
    const overdue=t.dueDate&&t.dueDate<today;
    const dueToday=t.dueDate&&t.dueDate.toDateString()===today.toDateString();
    const urgent=overdue||dueToday;
    const dueLbl=t.dueDate?(overdue?'Overdue':dueToday?'Due today':'Due '+t.dueDate.toLocaleDateString()):'No date';
    return `<div class="alert-item">
      <input type="checkbox" onchange="handleCompleteTask(${origIdx})" style="width:18px;height:18px;cursor:pointer;">
      ${urgent?'<span class="badge badge-failed">URGENT</span>':''}
      <div class="alert-text"><strong>${t.clientName||'No client'}</strong> — ${t.description}</div>
      <span style="font-size:12px;color:${urgent?'var(--red)':'var(--g500)'}">${dueLbl}</span>
    </div>`;
  }).join('');
}

async function handleCompleteTask(idx){
  try{
    await completeTask(idx);
    renderHome();
    showToast('Task completed');
  }catch(e){console.error(e);showToast('Error completing task');}
}
```

- [ ] **Step 3: Rewrite renderAlerts() (simplified priority list)**

Replace the existing `rAlerts()` function (line ~698-758) and `fAlert()` (line ~759):

```javascript
function renderAlerts(){
  const today=new Date(); today.setHours(0,0,0,0);
  const alerts=[];

  // 1. Payment failures (red)
  const failed=clients.filter(c=>c.status==='Payment Failed - Lapse Warning');
  if(failed.length){
    alerts.push({color:'urgent',text:`${failed.length} payment failure${failed.length>1?'s':''} need attention`,
      clients:failed,action:'task'});
  }

  // 2. Upcoming drafts within 7 days (amber)
  const upcoming=clients.filter(c=>c.status==='Active'&&c.daysUntilDraft!==null&&c.daysUntilDraft>=0&&c.daysUntilDraft<=7);
  if(upcoming.length){
    alerts.push({color:'warn',text:`${upcoming.length} draft${upcoming.length>1?'s':''} coming up this week`,
      clients:upcoming,action:'view'});
  }

  // 3. Recently lapsed <60 days (gray)
  const lapsed=clients.filter(c=>{
    if(c.status!=='Lapsed') return false;
    const dd=c.draftDate||c.dateOfSale;
    if(!dd) return true;
    return daysSince(dd)<60;
  });
  if(lapsed.length){
    alerts.push({color:'info',text:`${lapsed.length} recently lapsed polic${lapsed.length>1?'ies':'y'}`,
      clients:lapsed,action:'view'});
  }

  const container=document.getElementById('alerts');
  if(!alerts.length){
    container.innerHTML='<div class="empty">No alerts — all clear!</div>';
    return;
  }

  container.innerHTML=alerts.map((a,i)=>{
    const hasTasked=a.clients.every(c=>tasks.some(t=>!t.completed&&t.clientName===c.name));
    return `<div class="alert-item"${hasTasked?' style="opacity:.5"':''}>
      <div class="alert-dot ${a.color}"></div>
      <div class="alert-text">${a.text}</div>
      ${a.action==='task'?`<button class="btn btn-sm btn-outline" onclick="createTasksFromAlert(${i})" ${hasTasked?'disabled':''}>Create Tasks</button>`
        :`<button class="btn btn-sm btn-outline" onclick="filterByStatus('${a.clients[0].status}')">View Clients</button>`}
    </div>`;
  }).join('');

  // Store alerts for task creation reference
  window._currentAlerts=alerts;
}

async function createTasksFromAlert(alertIdx){
  const alert=window._currentAlerts[alertIdx];
  if(!alert) return;
  try{
    for(const c of alert.clients){
      const existing=tasks.find(t=>!t.completed&&t.clientName===c.name);
      if(!existing){
        await createTask(c._airtableId, c.name, alert.text.replace(/\d+ /,'')+'— '+c.name, new Date(), 'alert');
      }
    }
    renderHome();
    showToast('Tasks created from alert');
  }catch(e){console.error(e);showToast('Error creating tasks');}
}
```

- [ ] **Step 4: Add openAddTask() for manual task creation**

```javascript
function openAddTask(){
  const name=prompt('Client name (or leave blank):');
  const desc=prompt('Task description:');
  if(!desc) return;
  const dueDateStr=prompt('Due date (YYYY-MM-DD, or leave blank):');
  const dueDate=dueDateStr?new Date(dueDateStr+'T12:00:00'):null;
  const client=name?clients.find(c=>c.name.toLowerCase().includes(name.toLowerCase())):null;
  createTask(client?client._airtableId:null, client?client.name:(name||''), desc, dueDate, 'manual')
    .then(()=>{renderHome();showToast('Task created');})
    .catch(e=>{console.error(e);showToast('Error creating task');});
}
```

- [ ] **Step 5: Verify Home tab renders KPIs, tasks, and alerts correctly, commit**

Open the app, load demo data. Confirm:
- 4 KPI cards show at top
- Tasks widget shows with sample tasks, checkbox completes them
- Alerts show grouped by urgency with action buttons

```bash
git add index.html
git commit -m "feat: implement Home tab with KPIs, tasks widget, and priority alerts"
```

---

### Task 4: Implement Clients Tab (Search + Filters + Table + Detail Panel)

**Files:**
- Modify: `index.html` (CSS for drawer + filter chips, JS for `renderClients()`, rewrite `rTable()`, new drawer functions)

**Context:** The Clients tab replaces the client list from Overview and the edit modal. It adds: full-width search, filter chips, simplified 5-column table, and a slide-in detail panel (replacing the modal).

- [ ] **Step 1: Add CSS for filter chips and detail drawer**

In the `<style>` section (before `</style>`, line ~134), add:

```css
/* Filter chips */
.chip{display:inline-block;padding:5px 14px;border-radius:20px;font-size:12px;font-weight:600;cursor:pointer;border:1.5px solid var(--g300);background:#fff;color:var(--g700);transition:all .15s;}
.chip:hover{border-color:var(--blue);color:var(--blue);}
.chip.active{background:var(--navy);color:#fff;border-color:var(--navy);}
.chip-sep{color:var(--g300);margin:0 4px;font-size:16px;line-height:1;}

/* Client detail drawer */
.drawer-overlay{position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,.4);z-index:1000;opacity:0;pointer-events:none;transition:opacity .2s;}
.drawer-overlay.open{opacity:1;pointer-events:all;}
.drawer{position:fixed;top:0;right:0;width:480px;max-width:90vw;height:100%;background:#fff;z-index:1001;box-shadow:-4px 0 20px rgba(0,0,0,.15);transform:translateX(100%);transition:transform .25s ease;overflow-y:auto;}
.drawer-overlay.open .drawer{transform:translateX(0);}
.drawer-header{padding:20px 24px;border-bottom:1px solid var(--g200);display:flex;justify-content:space-between;align-items:center;position:sticky;top:0;background:#fff;z-index:1;}
.drawer-header h2{font-size:18px;font-weight:700;color:var(--navy);}
.drawer-section{padding:16px 24px;border-bottom:1px solid var(--g100);}
.drawer-section h4{font-size:13px;font-weight:600;color:var(--g500);text-transform:uppercase;letter-spacing:.3px;margin-bottom:10px;}
.activity-entry{display:flex;gap:10px;padding:8px 0;border-bottom:1px solid var(--g50);}
.activity-entry:last-child{border-bottom:none;}
.activity-icon{width:28px;height:28px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:12px;flex-shrink:0;}
.activity-icon.call{background:#DEF7EC;color:#03543F;}
.activity-icon.follow-up{background:#FEF3C7;color:#92400E;}
.activity-icon.note{background:#DBEAFE;color:#1E40AF;}
.activity-icon.task-completed{background:#E0E7FF;color:#3730A3;}
```

- [ ] **Step 2: Add drawer HTML**

After the existing edit modal HTML (line ~283-323), add the drawer overlay:

```html
<!-- Client Detail Drawer -->
<div class="drawer-overlay" id="drawerOverlay" onclick="if(event.target===this)closeDrawer()">
  <div class="drawer" id="clientDrawer">
    <div class="drawer-header">
      <h2 id="drawerTitle">Client Details</h2>
      <button class="modal-close" onclick="closeDrawer()">&times;</button>
    </div>
    <div id="drawerContent"></div>
  </div>
</div>
```

- [ ] **Step 3: Write renderClients() and update rTable()**

Replace the existing `rTable()` function (line ~762) with a new version, and add `renderClients()`:

```javascript
let activeFilters={statuses:[],carriers:[],search:''};

function renderClients(){
  // Render filter chips (multiple can be active simultaneously)
  const statuses=['Active','Pending','Lapsed'];
  const carriers=['AIG','RNA','AHL','AMAM','Transamerica'];
  const chipContainer=document.getElementById('filterChips');
  const noStatusFilter=activeFilters.statuses.length===0;
  chipContainer.innerHTML=
    `<span class="chip${noStatusFilter?' active':''}" onclick="clearFilter('statuses')">All</span>`
    +statuses.map(s=>`<span class="chip${activeFilters.statuses.includes(s)?' active':''}" onclick="toggleFilter('statuses','${s}')">${s}</span>`).join('')
    +'<span class="chip-sep">|</span>'
    +carriers.map(c=>`<span class="chip${activeFilters.carriers.includes(c)?' active':''}" onclick="toggleFilter('carriers','${c}')">${c}</span>`).join('');
  rTable();
}

function toggleFilter(key,val){
  const arr=activeFilters[key];
  const idx=arr.indexOf(val);
  if(idx>-1) arr.splice(idx,1); else arr.push(val);
  renderClients();
}

function clearFilter(key){
  activeFilters[key]=[];
  renderClients();
}

function filterByStatus(status){
  // Called from alerts "View Clients" button
  const mapped=status.startsWith('Pending')||status.startsWith('Payment')?'Pending':status;
  activeFilters.statuses=[mapped];
  switchTab('clients',document.querySelectorAll('.main-tab')[1]);
}

function rTable(){
  const search=(document.getElementById('clientSearch')||{}).value||'';
  activeFilters.search=search.toLowerCase();

  let list=[...clients];

  // Apply filters (multiple chips can be active)
  if(activeFilters.statuses.length){
    list=list.filter(c=>{
      return activeFilters.statuses.some(s=>{
        if(s==='Pending') return c.status.startsWith('Pending')||c.status==='Payment Failed - Lapse Warning';
        return c.status===s;
      });
    });
  }
  if(activeFilters.carriers.length) list=list.filter(c=>activeFilters.carriers.includes(c.carrier));
  if(activeFilters.search){
    list=list.filter(c=>
      (c.name||'').toLowerCase().includes(activeFilters.search)||
      (c.phone||'').toLowerCase().includes(activeFilters.search)||
      (c.carrier||'').toLowerCase().includes(activeFilters.search)
    );
  }

  // Sort
  if(sortCol){
    list.sort((a,b)=>{
      let va=a[sortCol],vb=b[sortCol];
      if(typeof va==='string') return va.localeCompare(vb)*sortDir;
      return ((va||0)-(vb||0))*sortDir;
    });
  }

  document.getElementById('cc').textContent=list.length+' client'+(list.length!==1?'s':'');

  // Simplified 5-column header
  document.getElementById('th').innerHTML=`<tr>
    <th style="cursor:pointer" onclick="doSort('name')">Name</th>
    <th style="cursor:pointer" onclick="doSort('carrier')">Carrier</th>
    <th style="cursor:pointer" onclick="doSort('status')">Status</th>
    <th style="cursor:pointer" onclick="doSort('monthlyPremium')">Premium</th>
    <th style="cursor:pointer" onclick="doSort('daysUntilDraft')">Draft In</th>
  </tr>`;

  document.getElementById('tb').innerHTML=list.map((c,i)=>{
    const idx=clients.indexOf(c);
    const dd=c.daysUntilDraft;
    const ddCls=dd!==null&&dd<=3?'draft-urgent':dd!==null&&dd<=7?'draft-warning':'';
    return `<tr class="clickable-row" onclick="openDrawer(${idx})">
      <td><strong>${c.name||'—'}</strong></td>
      <td>${c.carrier||'—'}</td>
      <td>${sBadge(c.status)}</td>
      <td>${$v(c.monthlyPremium)}</td>
      <td class="${ddCls}">${dd!==null?dd+' days':'—'}</td>
    </tr>`;
  }).join('');
}
```

- [ ] **Step 4: Write drawer open/close and content rendering**

```javascript
let drawerIdx=-1;

function openDrawer(idx){
  drawerIdx=idx;
  const c=clients[idx];
  const clientActivities=activities.filter(a=>a.clientName===c.name).sort((a,b)=>(b.timestamp||0)-(a.timestamp||0));
  const clientTasks=tasks.filter(t=>!t.completed&&t.clientName===c.name);

  const actIcons={call:'📞',['follow-up']:'🔄',note:'📝',['task-completed']:'✅'};

  document.getElementById('drawerTitle').textContent=c.name||'Client Details';
  document.getElementById('drawerContent').innerHTML=`
    <div class="drawer-section">
      <h4>Info</h4>
      <div class="form-grid">
        <div class="form-group"><label>Phone</label><input type="text" id="dw-phone" value="${c.phone||''}"></div>
        <div class="form-group"><label>Status</label>
          <select id="dw-status">
            ${['Active','Pending First Payment','Pending Requirement','Payment Failed - Lapse Warning','Lapsed','Cancelled','Declined'].map(s=>`<option${c.status===s?' selected':''}>${s}</option>`).join('')}
          </select>
        </div>
        <div class="form-group"><label>Carrier</label>
          <select id="dw-carrier">
            ${['AIG','RNA','AHL','AMAM','Transamerica'].map(cr=>`<option${c.carrier===cr?' selected':''}>${cr}</option>`).join('')}
          </select>
        </div>
        <div class="form-group"><label>Plan Type</label>
          <select id="dw-plan">
            ${['Level','Graded','GI','Standard','Modified','ROP'].map(p=>`<option${c.planType===p?' selected':''}>${p}</option>`).join('')}
          </select>
        </div>
        <div class="form-group"><label>Monthly Premium</label><input type="number" step="0.01" id="dw-premium" value="${c.monthlyPremium||0}"></div>
        <div class="form-group"><label>Draft Date</label><input type="date" id="dw-draft" value="${dtToInput(c.draftDate)}"></div>
        <div class="form-group"><label>Date of Sale</label><input type="date" id="dw-dos" value="${dtToInput(c.dateOfSale)}"></div>
        <div class="form-group"><label>Annual Premium</label><div class="calc-val">${$v(c.annualPremium||0)}</div></div>
        <div class="form-group"><label>Advance Commission</label><div class="calc-val">${$v(c.advComm||0)}</div></div>
        <div class="form-group"><label>Total Commission</label><div class="calc-val">${$v(c.totalComm||0)}</div></div>
        <div class="form-group"><label>Remaining</label><div class="calc-val">${$v(c.remainComm||0)}</div></div>
        <div class="form-group full"><label>Notes</label><textarea id="dw-notes">${c.notes||''}</textarea></div>
        <div class="form-group full"><label>Retention Notes</label><textarea id="dw-retnotes">${c.retNotes||''}</textarea></div>
      </div>
      <div style="margin-top:12px;display:flex;gap:8px;justify-content:flex-end;">
        <button class="btn btn-sm btn-green" onclick="saveDrawer()">Save Changes</button>
      </div>
    </div>

    <div class="drawer-section">
      <h4>Tasks <button class="btn btn-sm btn-outline" style="float:right;font-size:11px;" onclick="addClientTask(${idx})">+ Task</button></h4>
      ${clientTasks.length?clientTasks.map(t=>{
        const ti=tasks.indexOf(t);
        return `<div class="alert-item" style="padding:6px 0;">
          <input type="checkbox" onchange="handleCompleteTask(${ti}).then(()=>openDrawer(${idx}))" style="width:16px;height:16px;cursor:pointer;">
          <div class="alert-text" style="font-size:13px;">${t.description}</div>
          <span style="font-size:11px;color:var(--g500);">${t.dueDate?t.dueDate.toLocaleDateString():'—'}</span>
        </div>`;
      }).join(''):'<div style="color:var(--g500);font-size:13px;">No open tasks</div>'}
    </div>

    <div class="drawer-section">
      <h4>Activity Log <button class="btn btn-sm btn-outline" style="float:right;font-size:11px;" onclick="addClientActivity(${idx})">+ Log Activity</button></h4>
      ${clientActivities.length?clientActivities.map(a=>`
        <div class="activity-entry">
          <div class="activity-icon ${a.type}">${actIcons[a.type]||'📝'}</div>
          <div style="flex:1;">
            <div style="font-size:13px;">${a.description}</div>
            <div style="font-size:11px;color:var(--g500);">${a.type} • ${a.timestamp?a.timestamp.toLocaleString():'—'}</div>
          </div>
        </div>
      `).join(''):'<div style="color:var(--g500);font-size:13px;">No activity logged yet</div>'}
    </div>
  `;

  document.getElementById('drawerOverlay').classList.add('open');
}

function closeDrawer(){
  document.getElementById('drawerOverlay').classList.remove('open');
  drawerIdx=-1;
}

function addClientTask(idx){
  const c=clients[idx];
  const desc=prompt('Task description:');
  if(!desc) return;
  const dueDateStr=prompt('Due date (YYYY-MM-DD, or leave blank):');
  const dueDate=dueDateStr?new Date(dueDateStr+'T12:00:00'):null;
  createTask(c._airtableId, c.name, desc, dueDate, 'manual')
    .then(()=>{openDrawer(idx);showToast('Task created');})
    .catch(e=>{console.error(e);showToast('Error creating task');});
}

function addClientActivity(idx){
  const c=clients[idx];
  const type=prompt('Activity type (call, follow-up, note):');
  if(!type||!['call','follow-up','note'].includes(type)){showToast('Invalid type');return;}
  const desc=prompt('Description:');
  if(!desc) return;
  logActivity(c._airtableId, c.name, type, desc)
    .then(()=>{openDrawer(idx);showToast('Activity logged');})
    .catch(e=>{console.error(e);showToast('Error logging activity');});
}
```

- [ ] **Step 5: Write saveDrawer() — sync edits back to Airtable**

This reuses logic from the existing `saveEdit()` function (line ~847). Adapt it to read from drawer fields:

```javascript
async function saveDrawer(){
  const c=clients[drawerIdx];
  c.phone=document.getElementById('dw-phone').value;
  c.status=document.getElementById('dw-status').value;
  c.carrier=document.getElementById('dw-carrier').value;
  c.planType=document.getElementById('dw-plan').value;
  c.monthlyPremium=parseFloat(document.getElementById('dw-premium').value)||0;
  c.draftDate=inputToDt(document.getElementById('dw-draft').value);
  c.dateOfSale=inputToDt(document.getElementById('dw-dos').value);
  c.notes=document.getElementById('dw-notes').value;
  c.retNotes=document.getElementById('dw-retnotes').value;
  calcCommission(c);

  if(c._airtableId&&AT_TOKEN){
    try{
      const fields={
        'Phone Number':c.phone,
        'Policy Status':c.status,
        'Carrier':c.carrier,
        'Plan Type':c.planType,
        'Monthly Premium':c.monthlyPremium,
        'First Draft Date':c.draftDate?c.draftDate.toISOString().split('T')[0]:'',
        'Date of Sale':c.dateOfSale?c.dateOfSale.toISOString().split('T')[0]:'',
        'Notes':c.notes,
        'Retention Notes':c.retNotes
      };
      await fetch(`${AT_URL}/${c._airtableId}`,{
        method:'PATCH',
        headers:{'Authorization':`Bearer ${AT_TOKEN}`,'Content-Type':'application/json'},
        body:JSON.stringify({fields})
      });
    }catch(e){console.error(e);showToast('Error saving to Airtable');}
  }

  openDrawer(drawerIdx); // Refresh drawer with new calc values
  renderTab(document.querySelector('.main-tab.active').textContent.toLowerCase());
  showToast('Changes saved');
}
```

- [ ] **Step 6: Verify Clients tab works end-to-end, commit**

Test: search filters clients, chips toggle, click opens drawer, edit saves, activity/task creation works.

```bash
git add index.html
git commit -m "feat: implement Clients tab with search, filter chips, and detail drawer"
```

---

### Task 5: Implement Money Tab (KPIs + Monthly + Carrier + Charts)

**Files:**
- Modify: `index.html` (JS — new `renderMoney()`, reuse logic from existing `renderYTD()` and `renderAllTime()`)

**Context:** The Money tab consolidates YTD and All-Time views with a toggle. Existing functions `renderYTD()` (line ~578) and `renderAllTime()` (line ~621) have the chart/table logic we need. We combine them into one `renderMoney()` function.

- [ ] **Step 1: Write renderMoney() function**

Remove the existing `renderYTD()` (line ~578-619) and `renderAllTime()` (line ~621-658) functions. Replace with:

```javascript
function renderMoney(){
  const act=eligible();
  const yr=curYear();
  const ytd=ytdEligible();
  const ytdEarned=ytd.reduce((s,c)=>s+c.advComm,0);
  const allEarned=act.reduce((s,c)=>s+c.advComm,0);
  const allTotal=act.reduce((s,c)=>s+c.totalComm,0);
  const allRemain=act.reduce((s,c)=>s+c.remainComm,0);

  // KPIs
  document.getElementById('kpis-money').innerHTML=`
    <div class="kpi good"><div class="lbl">YTD Earned</div><div class="val">${$v(ytdEarned)}</div></div>
    <div class="kpi"><div class="lbl">All-Time Earned</div><div class="val">${$v(allEarned)}</div></div>
    <div class="kpi"><div class="lbl">Advance Paid</div><div class="val">${$v(allEarned)}</div></div>
    <div class="kpi"><div class="lbl">Remaining</div><div class="val">${$v(allRemain)}</div></div>
  `;

  const isYTD=moneyView==='ytd';
  const pool=isYTD?ytd:act;

  // Monthly breakdown table
  const monthly=Array.from({length:12},()=>({earned:0,count:0}));
  pool.forEach(c=>{
    if(c.earnedDate){
      const m=c.earnedDate.getMonth();
      const matchYear=isYTD?c.earnedDate.getFullYear()===yr:true;
      if(matchYear){monthly[m].earned+=c.advComm;monthly[m].count++;}
    }
  });

  const now=new Date();
  let totalEarned=0,totalCount=0;
  const tbl=document.getElementById('monthlyTbl');
  tbl.innerHTML=`<thead><tr><th style="text-align:left">Month</th><th>Earned</th><th>Policies</th></tr></thead><tbody>`+
    monthly.map((m,i)=>{
      totalEarned+=m.earned;totalCount+=m.count;
      const isCur=isYTD&&i===now.getMonth();
      const isFuture=isYTD&&i>now.getMonth();
      return `<tr class="${isCur?'':isFuture?'future':''}">
        <td>${MONTHS[i]}${isCur?' ★':''}</td><td>${$v(m.earned)}</td><td>${m.count}</td></tr>`;
    }).join('')+
    `<tr class="total-row"><td>Total</td><td>${$v(totalEarned)}</td><td>${totalCount}</td></tr></tbody>`;

  // Carrier cards
  const byCarrier={};
  pool.forEach(c=>{if(!byCarrier[c.carrier])byCarrier[c.carrier]={earned:0,total:0,count:0};
    byCarrier[c.carrier].earned+=c.advComm;byCarrier[c.carrier].total+=c.totalComm;byCarrier[c.carrier].count++;});
  document.getElementById('carrierCards').innerHTML=Object.entries(byCarrier).map(([name,d])=>`
    <div class="carrier-card">
      <div class="cc-name">${name}</div>
      <div class="cc-row"><span>Earned</span><span class="cc-val">${$v(d.earned)}</span></div>
      <div class="cc-row"><span>Total</span><span class="cc-val">${$v(d.total)}</span></div>
      <div class="cc-row"><span>Policies</span><span class="cc-val">${d.count}</span></div>
    </div>
  `).join('');

  // Charts
  const carriers=Object.keys(byCarrier);
  charts['ch-carrier']=new Chart(document.getElementById('ch-carrier'),{
    type:'bar',
    data:{labels:carriers,datasets:[{label:'Commission',data:carriers.map(c=>byCarrier[c].earned),
      backgroundColor:'rgba(46,80,144,.7)'}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
      scales:{y:{beginAtZero:true,ticks:{callback:v=>$v(v)}}}}
  });

  const cumData=[];let cum=0;
  monthly.forEach((m,i)=>{cum+=m.earned;cumData.push(cum);});
  charts['ch-cum']=new Chart(document.getElementById('ch-cum'),{
    type:'line',
    data:{labels:MONTHS,datasets:[{label:'Cumulative',data:cumData,
      borderColor:'var(--blue)',backgroundColor:'rgba(46,80,144,.1)',fill:true,tension:.3}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
      scales:{y:{beginAtZero:true,ticks:{callback:v=>$v(v)}}}}
  });
}
```

- [ ] **Step 2: Verify Money tab renders correctly with YTD/All-Time toggle, commit**

Open app, load data, click Money tab. Toggle between YTD and All-Time. Verify monthly table, carrier cards, and charts all update.

```bash
git add index.html
git commit -m "feat: implement Money tab with YTD/All-Time toggle, charts, and carrier cards"
```

---

### Task 6: Clean Up — Remove Dead Code & Update Topbar

**Files:**
- Modify: `index.html` (remove old functions, update header, remove old HTML)

**Context:** After Tasks 2-5, the old `renderOverview()`, `renderYTD()`, `renderAllTime()`, `buildCarrierCards()`, `buildPlanCards()`, `fAlert()`, and old filter HTML are no longer used. The topbar subtitle also needs updating.

- [ ] **Step 1: Update topbar subtitle**

Find line ~138:
```html
<div class="sub">Commission Tracking &bull; Payment Monitoring &bull; Carrier Payout Timing</div>
```
Replace with:
```html
<div class="sub">CRM &bull; Client Retention &bull; Commission Tracking</div>
```

- [ ] **Step 2: Remove dead functions**

Remove these functions if they still exist and are no longer called:
- `renderOverview()` (replaced by `renderHome()`)
- `renderYTD()` (replaced by `renderMoney()`)
- `renderAllTime()` (replaced by `renderMoney()`)
- `buildCarrierCards()` (logic inlined in `renderMoney()`)
- `buildPlanCards()` (plan type breakdown removed — YAGNI)
- `fAlert()` (old alert tab toggle — replaced by single list)
- Old `rTable()` (replaced by new version in Task 4)
- `openEdit()` / `saveEdit()` (replaced by drawer functions for existing clients)

**Keep the edit modal HTML** (lines ~283-323) and `openNewClient()` / `saveNewClient()` / `closeModal()` / `updateCalcPreview()` — these are still used for creating new clients via the "+ Add Client" button. The modal is only for *new* client creation now; existing client editing uses the drawer.

Keep: `norm()`, `calcCommission()`, `fetchAirtable()`, `exportCSV()`, `loadDemo()`, `switchTab()`, `eligible()`, `ytdEligible()`, `doSort()`, `sBadge()`, `dtToInput()`, `inputToDt()`, `daysSince()`, `showToast()`, `openNewClient()`, `saveNewClient()`, `closeModal()`, all settings/token functions.

- [ ] **Step 3: Remove old CSS that's no longer needed**

Review and remove CSS rules for elements that no longer exist:
- `.alert-phone` (old alert style — now using generic `.alert-item`)
- Old filter dropdowns CSS (`.filters`, `.fg` — replaced by chips)

Keep all reusable CSS: `.kpi`, `.section`, `.btn`, `.badge`, `.chart-grid`, `.monthly-tbl`, `.carrier-cards`, `.modal-overlay` (still used for new client), `.toast`, etc.

- [ ] **Step 4: Verify nothing is broken after cleanup, commit**

Test all three tabs, data loading, task/activity operations, client editing. Check console for errors.

```bash
git add index.html
git commit -m "refactor: remove dead code, update topbar, clean up unused CSS"
```

---

### Task 7: Final Verification & Polish

**Files:**
- Modify: `index.html` (minor fixes)
- Modify: `CLAUDE.md` (update to reflect new architecture)

- [ ] **Step 1: End-to-end testing checklist**

Open app in browser and verify each item:
- [ ] Home tab: KPIs show correct numbers
- [ ] Home tab: Tasks display, can be completed, auto-log activity
- [ ] Home tab: Alerts show in priority order, "Create Tasks" works
- [ ] Clients tab: Search filters by name, phone, carrier
- [ ] Clients tab: Filter chips toggle correctly
- [ ] Clients tab: Table shows 5 columns, sorted by click
- [ ] Clients tab: Drawer opens, shows info + activities + tasks
- [ ] Clients tab: Drawer save syncs to Airtable
- [ ] Clients tab: Add activity and add task from drawer work
- [ ] Money tab: KPIs show correct amounts
- [ ] Money tab: Monthly table shows correct data
- [ ] Money tab: YTD/All-Time toggle works
- [ ] Money tab: Charts render
- [ ] File upload (CSV/XLSX) still works
- [ ] Demo data works
- [ ] Export CSV works
- [ ] Settings (Airtable token) works
- [ ] Commission calculations match old app values

- [ ] **Step 2: Update CLAUDE.md**

Update `CLAUDE.md` to reflect the new 3-tab structure (Home/Clients/Money), new functions (`renderHome`, `renderClients`, `renderMoney`, `createTask`, `completeTask`, `logActivity`), new global state (`tasks`, `activities`, `activeFilters`, `moneyView`), and new CSS classes (`.chip`, `.drawer`).

- [ ] **Step 3: Commit**

```bash
git add index.html CLAUDE.md
git commit -m "docs: update CLAUDE.md for redesigned dashboard architecture"
```
