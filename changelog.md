# changelog.md — Primrose Scheduler

All significant changes to the app, from first build to present. Most recent first.

---

## [Current] — April 27, 2026

### Fixed
- **PTO Summary report** now pulls from both `ptoData[]` (calendar entries) and `history[]` (day-of log). Previously, PTO entered via the calendar tab was invisible to reports because reports only read from history. Same fix applied to Combined Absence report.
- Deduplication logic: same `staffId|date` key counted only once across both sources.
- PTO detail log now shows a "Calendar" or "Logged" source badge per row.

---

## April 27, 2026

### Added
- **PTO Summary report** — new report type showing PTO days per person, split into Calendar vs Logged columns with totals.
- **Combined Absence Summary report** — all absence types (call-out, NC/NS, absent, PTO) in one report. Per-person breakdown with Unplanned % column. School-wide totals row.
- Report type dropdown now has 5 options: Call-out Summary, PTO Summary, Combined Absence Summary, By Person, By Date.
- Excel export handles all 5 report types (PTO and Combined each generate 2-sheet workbooks: summary + detail).

### Fixed
- **Timeline shift bars** — root cause of missing bars for Allen, Rivera, Yang, Brown, Yancey and others: `shiftDecimal()` was parsing end times like `"6:30"` as 6:30am instead of 6:30pm. Fixed with pair-aware rule: if `end <= start` after parsing, add 12 to end.
- **`fmt12()` and `parseTo24()`** unified and corrected — hours 1–5 are PM, hour 6 is AM (6:00am/6:30am are valid opening shifts).
- **`fmtShift()`** now uses the same pair-aware end correction for display.
- Myrtolli (6:30am), Farmer (6:30am), Ibraheem (6:45am), Santos (6:30am) now show correctly on timeline.
- **Timeline gap** between sticky ruler and first row removed (padding/margin CSS fix).

---

## April 26, 2026 (Session B)

### Added
- **Sticky timeline ruler** — time header stays fixed at top of timeline card while scrolling down through staff rows. Vertical tick lines faintly drawn behind all rows.
- **No Call / No Show (NC/NS)** as a distinct 4th absence type, more severe than regular Call-out. Shows 🚫 icon, bold red text, `pill-ncns` styling.
- NC/NS counter in schedule header bar (separate from call-outs).
- NC/NS tracked separately in call-out reports with its own column.
- **Auto-reset of stale daily statuses** — on app load, any staff member with callout/ncns/absent status from a previous date is automatically reset to Present. `statusDate` field stamps when status was set.
- **Employment Status** fields on staff modal: Active / Probationary / On Leave / Resigned / Terminated / Retired / Seasonal.
- **Start Date and End Date** on staff records.
- **Separation Reason** dropdown (9 options: Voluntary Resignation, Job Abandonment, Performance/Conduct, Policy Violation, Layoff/RIF, Retirement, Mutual Agreement, End of Contract, Other).
- **isActiveEmployee()** function — returns false for terminated/resigned/retired staff after their end date. Filters them from schedule, timeline, break ref, PTO modal, recruiting panel.
- Staff table now shows employment status badge (color-coded) and separation reason.
- Staff table filter by employment status (Active Only default, or specific status).
- **Part-time weekly schedule picker** — when PT checkbox is checked in staff modal, Mon–Fri day checkboxes appear with per-day start/end times.
- `ptSchedule` array stored on staff record. `fmtStaffShift()` displays PT schedule summary instead of shift string.
- **Sortable staff table columns** — click header to sort asc/desc. Name (alpha), Room/Age Group (youngest→oldest), Shift (by start time), Emp Status, Start Date, Type (FT→PT).
- `ROOM_SORT_ORDER` constant for age-group sort.
- `setStaffSort(col)` function wired to column headers.
- **Roster Excel export** (`exportRosterExcel()`) — human-readable staff roster as .xlsx. Separate from the JSON backup/restore.
- Export/Import buttons renamed: "📊 Roster Excel", "⬇ Backup (.json)", "⬆ Restore (.json)" with tooltips explaining each.
- **History tab date range filter** — From/To date pickers added alongside name search and event type filter.
- History Excel export respects all active filters.
- **Partial-day PTO** — PTO modal now has Absence Type dropdown: Full Day(s), Left Early, Arrived Late, Out for Block of Time. Partial entries capture time-out, time-in, and estimated hours.
- `togglePartialFields()` shows/hides relevant time fields. Partial entries stored with `partial:true` flag and `dayType`, `partialStart`, `partialEnd`, `hoursOut` fields.

### Fixed
- PT schedule modal fields were written with `${i}` JS template syntax inside plain HTML, causing `document.getElementById('ptDay0')` to return null and silently crash `editStaff()`. Fixed by writing all 5 day rows as static HTML with hardcoded IDs (ptDay0–ptDay4).
- Break reference chips now only show active staff (`isActiveEmployee()` check added to `renderBreaks()`).
- Staff Notes sidebar filtered by `isActiveEmployee()` — Harry and Lewis no longer showed after separation date.
- Recruiting panel filtered by `isActiveEmployee()`.
- SheetJS changed from synchronous `<script src>` tag to async dynamic injection, preventing CDN load from blocking app startup.
- `dateChip` restored as visible formatted label above the date picker (was missing, showing browser placeholder).
- `xlsxDownload()` guarded with `typeof XLSX === 'undefined'` check.
- Syntax error: `function goTab(name){` declaration was consumed during Excel export function insertion. Restored.
- `isActiveEmployee()` now checks `endDate >= TODAY` — staff with a future end date still appear until that date passes.

---

## April 26, 2026 (Session A)

### Added
- **PTO modal conflict panel** — live panel in the PTO request modal shows who is already out on each day in the selected range. Blocked days shown in red, same-room colleagues in amber with "SAME ROOM" tag.
- **Manual block days (✕)** — replaced auto-cap system. Managers click any date → "🚫 Block This Day" button. Blocked days stored in `blockedDays[]` array, synced to Google Sheets.
- `toggleBlockDay(ds)` function. `updateBlockedStat()` updates blocked day counter in calendar header.
- **Calendar color thresholds** — 1 out: amber background, 2 out: yellow border, 3+ out: red border, blocked: ✕ overlay.
- `blockedDays[]` added to all save/load/export paths.
- **Remove PTO from Manage Staff** tab — PTO managed via PTO Calendar tab only.
- **PTO list table** below the calendar — shows all PTO requests with Name, Room, Start, End, Days (weekdays), Status badge (Active now / Upcoming / Past), Remove button.
- `renderPtoList()` function. Name search and Upcoming/All/Past filter.
- `confirmRemovePto(id, name, dateLabel)` with confirm prompt.
- **Same-room PTO conflict** — calendar shows purple border when 2+ teachers from same room have PTO on same day. `hasSameRoomConflict(ds)` function.

### Changed
- **PTO submission** — removed hard MAX_PTO cap that blocked submission. Now warns only (confirm prompt lists blocked days and same-room conflicts) but always allows submission.
- Same-room warning in PTO modal is purely informational — conflict panel already shows it visually.
- MAX_PTO constant removed entirely from the codebase.
- Calendar legend updated to explain all 5 states.

---

## April 25, 2026

### Added
- **EPS/PSP distinct color** — `eps2yr` room type with teal color (`#0e7c6b`) to visually distinguish 2-year-old rooms from PS 1/2 (3-year-olds). Legend updated: "EPS/PSP (2yr)" vs "PS 1/2 (3yr)".
- **PTO Request button** fixed — was calling `openPtoModal()` directly instead of `openModal('ptoModal')`, so the overlay never appeared.
- **PTO modal staff list** now shows only active employees.
- **Day detail popup** improved — shows staff name, room, shift hours for everyone on PTO. Same-room colleagues highlighted in amber.
- `hasSameRoomConflict(ds)` function added.

### Fixed
- `getPtoOnDate(ds)` enriched to return `room` field alongside `name` for conflict detection.
- Break times displaying as AM instead of PM (1:00am instead of 1:00pm). Root cause: `fmt12()` was treating all hours < 7 as PM, but break hours stored as "1:00", "2:30" etc. were being treated as 1am/2am. Fixed with context-aware PM rule.

---

## April 24, 2026

### Added
- **Date navigator** in header — ← → arrows with date picker. Arrows skip weekends. "TODAY" pill appears when viewing non-current date.
- **Historical snapshot view** — navigate to past date → room schedule shows reconstructed status from `history[]` + `ptoData[]`. Staff status buttons become read-only. Amber banner shows date and event count.
- **Future date view** — shows `ptoData[]` PTO entries. Banner shows who is scheduled out.
- **`viewDate` state variable** — controls what date the Schedule tab displays. `isLive = viewDate === TODAY`.
- **Log Past Event** button — when viewing a past date, "Mark Absence" changes to "+ Log Past Event (Apr 15)" and opens modal in backfill mode. All staff shown (not just present), entry written to `history[]` for that date without touching live status.
- **History tab** with collapsible day entries, pill badges (call-out/absent/PTO), expand/collapse.
- **Reports tab** with 3 types: Call-out Summary, By Person, By Date.
- **Call-out report** tracks Monday/Friday call-outs with ⚑ flag and Mon/Fri rate %.
- **Open Positions recruiting panel** on schedule sidebar — shows staff flagged as "High Priority" or "Watching" with their shift. Also shows rooms currently understaffed today.
- Recruiting status field in staff modal (Position Filled / High Priority — Recruit Now / Medium — Watching).

### Changed
- `renderSidebars()` updated to use `effectiveStaff` (built from history + PTO for non-live dates).
- "Mark Absence" button hides for future dates, adapts label for past dates.

---

## April 23, 2026

### Added
- **History tracking** — `history[]` array. `recordEvent(person, type, notes)` called whenever a status changes from present. `removeHistoryToday(staffId)` called when reset to present.
- **Call-out as distinct status** — previously only absent/PTO. Call-out added as explicit type (unplanned, day-of). Status cycle: Present → Call-out → Absent → PTO.
- Call-out displayed in purple throughout (pill-callout, stoggle.callout, sname.callout).
- Call-out counter in schedule header.
- **"Mark Absence" modal** with staff selector, type dropdown, notes field.
- `submitAbsence()` function — updates live status and records to history.

---

## April 22, 2026

### Added
- **PTO Calendar tab** — monthly calendar view with `ptoData[]` entries.
- `getPtoOnDate(ds)` function.
- `openDay(ds)` — day detail modal showing who's on PTO with Remove button.
- `removePto(id)` function.
- **+ Request PTO modal** with staff selector and date range picker.
- `submitPto()` function.
- Staff shown as ⏸ on schedule when on PTO.
- PTO auto-populated from `ptoData[]` when navigating to a date.
- `MAX_PTO` constant (now removed — see April 26).
- Calendar color coding (amber/yellow/red for PTO count).

---

## April 21, 2026

### Added
- **Break Reference Schedule tab** — groups all staff by break time slot, shows name + room. Reference only (not tracked daily).
- Break stats: 1-hr breaks, 30-min breaks, no break counts.
- Break data stored per staff: `breakDur` ("none"|"30"|"60"), `breakStart` (time string).
- `breakLabel(s)` function.
- `bsk(s)` sort key function (break start key for chronological ordering).
- Staff modal fields for break duration and break start time.

---

## April 20, 2026

### Added
- **Google Sheets sync** — `save()` pushes `{staff, ptoData, history}` JSON to Apps Script. `init()` fetches on load.
- `setSyncStatus(state)` — shows "Syncing…" / "Saved" / "Local only" in header.
- **Coverage alerts sidebar** — lists rooms below minimum ratio.
- **Staff Notes sidebar** — shows staff with notes field populated.
- **Float section** on schedule — separate card at bottom for float/unassigned staff.
- Room cards pulse/glow when understaffed (CSS animation).
- VPK badge on 4K1 and 4K2 rooms.
- Nap window badges on room cards.

---

## April 19, 2026

### Added
- **Manage Staff tab** — full roster table with add/edit/delete.
- Staff modal: name, room, shift start/end, break, notes, FT/PT, float checkbox.
- Room filter, type filter in staff table.
- **Export (.json) / Import (.json)** buttons.
- `populateRoomSelects()` — populates all room dropdowns consistently.

---

## March–April 2026 (Initial Build)

### Added
- Single-file HTML app skeleton with DM Sans + JetBrains Mono fonts.
- CSS design system: color tokens, card components, tab switching, modal overlays.
- **Room Schedule tab** — 13 room cards grouped by type (infant/toddler/eps2yr/preschool/prek), each showing staff with shift, status toggle.
- `ROOMS` constant with all 13 rooms, ratios, capacities, nap schedules.
- `DEFAULT_STAFF` array — 42 staff members with room assignments, shifts, break times.
- Status toggle button cycling Present ↔ Absent.
- `localStorage` persistence for staff data.
- Schedule header stats: Present / Absent / PTO counts.
- Room legend (colored dots by age group).
- Initial staffing efficiency analysis in Excel (separate from the app).

---

## Excel Analysis Phase (March 2026)

Before the web app, Claude produced:
- `Primrose_Efficiency_Analysis_Complete.xlsx` — 13-room analysis with staffing costs, OT risk, efficiency scores.
- `Nap_Break_Schedule.xlsx` — nap-aligned break schedule for all rooms.
- `Primrose_Efficiency_Analysis_Revised.xlsx` — updated with Carlos's staffing data.

Key findings used in app design:
- Room ratios: 1:4 (infant), 1:6 (toddler), 1:11 (EPS/PSP), 1:15 (PS), 1:20 (4K).
- Peak coverage hours: 8am–5pm. Early/late shifts needed for 6:30am open and 6:30pm close.
- Float staff essential for ratio compliance during breaks and call-outs.
- VPK rooms (4K1, 4K2) require 2 teachers minimum at all times.
