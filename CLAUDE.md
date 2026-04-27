# CLAUDE.md — Primrose Scheduler Project Context

## What This Project Is

A staff scheduling and attendance tracking web app for **Primrose School of Glen Kernan**, a Primrose franchise preschool in Jacksonville, FL. The owner is Carlos, who manages day-to-day operations including staff scheduling, attendance, PTO, and HR reporting.

The app is a **single-file HTML application** (`index.html`) deployed via GitHub Pages at:
`https://unquote0325.github.io/primrose-scheduler/`

Data is persisted in **two places simultaneously**:
1. Browser `localStorage` (keys: `pgk_staff`, `pgk_pto`, `pgk_history`, `pgk_blocked`)
2. Google Sheets via Apps Script (cell B1 of "Primrose Scheduler Data" sheet)

## Repository

- **GitHub:** `https://github.com/unquote0325/primrose-scheduler`
- **Main file:** `index.html` (single-file app, ~2,900 lines)
- **Deployment:** GitHub Pages (auto-deploys on push to main)

## Google Sheets Backend

Apps Script URL:
```
https://script.google.com/macros/s/AKfycbzZ9mTf_GNR8GDvUYQgDZTfnsuJVIdPbSC88n9EGy3EcgOde_ujnbWjbjins-BdxMbx/exec
```

The Apps Script stores the entire app state as a JSON blob in cell B1:
```javascript
function doGet() { /* returns B1 as JSON */ }
function doPost(e) { /* writes e.postData.contents to B1 */ }
```

Payload shape: `{ staff, ptoData, history, blockedDays }`

## School Configuration

**Operating hours:** 6:30am – 6:30pm, Mon–Fri

**13 rooms** with ratios, capacities, nap schedules:

| ID | Name | Type | Ratio | Cap | Min Staff | Nap |
|----|------|------|-------|-----|-----------|-----|
| infant1–3 | Infant 1–3 | infant | 1:4 | 10/10/16 | 3/3/4 | None |
| tod1–4 | Toddler 1–4 | toddler | 1:6 | 13/16/17/18 | 3 | 12:00–2:00 (tod4: 12:15–2:15) |
| eps, psp | EPS, PSP | eps2yr (2yr) | 1:11 | 25 | 3 | 12:30–2:30 |
| ps1, ps2 | PS 1–2 | preschool (3yr) | 1:15 | 24 | 2 | 1:00–3:00 |
| 4k1, 4k2 | 4K1–2 | prek | 1:20 | 20 | 2 | 1:00–3:00, VPK=true |
| float | Float | float | — | — | 0 | — |

**Room sort order** (youngest to oldest, float last):
`infant1 → infant2 → infant3 → tod1 → tod2 → tod3 → tod4 → eps → psp → ps1 → ps2 → 4k1 → 4k2 → float`

## Critical Time System

All times are stored as **ambiguous short strings** (e.g., `"6:30"`, `"9:30"`, `"1:00"`) in the database. The app resolves them using `parseTo24()`:

```javascript
function parseTo24(t) {
  // h >= 13 → already 24hr (from time picker)
  // h === 12 → noon
  // h >= 6  → AM  (6:00am, 7:00am … 11:00am)
  // h 0–5   → PM  (no school activity 1am–5am, so must be afternoon)
  // returns decimal 24-hr hours (e.g. 18.5 for 6:30pm)
}
```

**Shift end-time fix:** `shiftDecimal()` and `fmtShift()` apply a pair-aware rule — if `end <= start` after parsing, add 12 to end. This correctly resolves `"9:30-6:30"` as 9:30am–6:30pm.

**Never use raw hour comparisons for time logic.** Always go through `parseTo24()`.

## Data Structures

### staff[] — Staff Roster
```javascript
{
  id: number,
  name: string,              // e.g. "Torres, Anita"
  room: string,              // room id e.g. "infant1", "eps", "float"
  shift: string,             // "6:30-3:00" (stored as short ambiguous string)
  breakDur: "none"|"30"|"60",
  breakStart: string|null,   // e.g. "1:00" (PM resolved by parseTo24)
  status: "present"|"callout"|"ncns"|"absent"|"pto",
  statusDate: string|null,   // YYYY-MM-DD — date status was set (for auto-reset)
  partTime: boolean,
  float: boolean,            // Float POSITION type (not employment type)
  ptSchedule: [{day:"Mon", start:"8:00", end:"14:00"}, ...] | undefined,
  empStatus: "active"|"probation"|"leave"|"resigned"|"terminated"|"retired"|"seasonal",
  startDate: string|null,    // YYYY-MM-DD
  endDate: string|null,      // YYYY-MM-DD — separation date
  termReason: string|null,   // e.g. "voluntary_resignation"
  notes: string|undefined,
  recruit: "high"|"med"|undefined,
}
```

### ptoData[] — PTO Calendar Entries
```javascript
{
  id: number,
  staffId: number,
  start: string,  // YYYY-MM-DD
  end: string,    // YYYY-MM-DD
  notes: string|undefined,
  // Partial day fields (optional):
  partial: boolean,
  dayType: "partial"|"partial_late"|"partial_out",
  partialStart: string,  // HH:MM
  partialEnd: string,
  hoursOut: number|null,
}
```

### history[] — Daily Event Log
```javascript
[{
  date: string,  // YYYY-MM-DD
  entries: [{
    staffId: number,
    name: string,
    room: string,       // display name e.g. "Infant 1"
    shift: string,      // stored shift string
    type: "callout"|"ncns"|"absent"|"pto",
    notes: string,
    time: string,       // "backfilled" or "HH:MM AM/PM"
  }]
}]
```

### blockedDays[] — Management-Blocked PTO Days
```javascript
["2026-05-26", "2026-12-24", ...]  // YYYY-MM-DD strings
```

## Status System

Staff `status` field cycles: `present → callout → ncns → absent → pto → present`

**Auto-reset:** On app load, `autoResetDailyStatuses()` resets any `callout`, `ncns`, or `absent` status back to `present` if `statusDate !== TODAY`. PTO is exempt (managed via ptoData).

**Absence types:**
- `callout` — unplanned, called management day-of
- `ncns` — No Call/No Show (most serious, did not notify)
- `absent` — notified ahead of time
- `pto` — scheduled PTO

**isActiveEmployee(s):** Returns false for terminated/resigned/retired staff whose `endDate` has passed. These staff are hidden from the schedule, timeline, break reference, PTO modal, and recruiting panel, but their history is preserved.

## Key Functions Reference

| Function | Purpose |
|----------|---------|
| `parseTo24(t)` | Canonical time resolver — all math uses this |
| `decimalTo12(dec)` | Display converter — decimal → "6:30pm" |
| `fmtShift(shift)` | Pair-aware shift display — "9:30-6:30" → "9:30am – 6:30pm" |
| `fmtStaffShift(s)` | Shift display for a staff object (handles PT schedule) |
| `shiftDecimal(str)` | {start, end} decimals for timeline math |
| `isActiveEmployee(s)` | Whether staff should appear in live views |
| `save()` | Persists to localStorage + Google Sheets |
| `renderAll()` | Re-renders all visible UI components |
| `autoResetDailyStatuses()` | Clears stale daily absences on app load |
| `getPtoOnDate(ds)` | PTO calendar entries for a given date |
| `hasSameRoomConflict(ds)` | True if 2+ same-room teachers on PTO that day |
| `toggleBlockDay(ds)` | Adds/removes a date from blockedDays |

## App Tabs

1. **Room Schedule** — Live daily view. Date navigator (← → TODAY). Past dates show historical snapshots from `history[]` + `ptoData[]`. Future dates show `ptoData[]` only.
2. **Break Schedule** — Reference only (not tracked daily). Chips are clickable → opens staff edit modal.
3. **Timeline** — Read-only shift bars grouped by room. Sticky time ruler. Shows nap windows.
4. **PTO Calendar** — Monthly calendar + full PTO list below. Manual block days with ✕. Yellow border = 2 out, red border = 3+ out, purple = same-room conflict. Blocked days (management manual) shown with ✕.
5. **Manage Staff** — Sortable roster table. Columns: Name ↑↓, Room/Age Group ↑↓, Shift ↑↓, Emp Status ↑↓, Start Date ↑↓, Type ↑↓. Employment status with separation reasons. PT weekly schedule picker.
6. **Reports** — 5 types: Call-out Summary, PTO Summary, Combined Absence Summary, By Person, By Date. All pull from BOTH `history[]` (logged events) and `ptoData[]` (calendar PTO). Excel export via SheetJS.
7. **History** — Collapsible day entries. Date range filter + name search + event type filter. Excel export.

## External Libraries

- **SheetJS (xlsx 0.18.5)** — loaded async from cdnjs to avoid blocking app startup
- **Google Fonts** — DM Sans + JetBrains Mono

## Important Design Decisions

- **No hard PTO cap** — system warns but never blocks submission. Management manually blocks days via the ✕ system.
- **Same-room PTO conflict** — shown as warning in the PTO request modal and as purple border on calendar. Never blocks.
- **Break management removed from daily workflow** — staff feedback was that tracking breaks day-to-day added stress. Break times are in staff profiles for reference only.
- **Float is a position type, not employment type** — float teachers are still FT or PT. The float checkbox means "no fixed room."
- **JSON for backup, Excel for human use** — Backup/Restore uses `.json` (round-trips all data). Roster Excel export uses SheetJS for human-readable spreadsheet.
- **Two PTO sources** — ptoData (calendar) and history (day-of log). Reports must merge both and deduplicate by `staffId|date` key.

## Development Notes

- **Always run a Node syntax check** after any JS edit: extract the `<script>` block and run `node --check`
- **SheetJS is loaded async** — always guard with `if(typeof XLSX === 'undefined')` before use
- **str_replace pitfalls** — when inserting large blocks adjacent to function declarations, the declaration line can be consumed. Always verify all function declarations are present after large edits.
- The app is a **single HTML file** — CSS, JS, and HTML all in one. Keep it that way.
- `from24()` converts HH:MM (from `<input type="time">`) back to the short stored format
- `toTime24()` converts stored short format → HH:MM for populating time pickers
