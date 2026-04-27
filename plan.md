# plan.md — Primrose Scheduler Roadmap

## Current State (as of April 27, 2026)

The app is fully functional and deployed. Core features are stable. The focus going forward is refinement, reliability, and GitHub-native development workflow.

### What's Working Well
- Room Schedule with live status cycling (Present / Call-out / NC/NS / Absent / PTO)
- Daily auto-reset of temporary statuses (callout, ncns, absent) at next load
- Historical snapshot view — navigate to any past date, see reconstructed schedule
- Future date view — shows scheduled PTO from the calendar
- PTO Calendar with manual management block (✕), color-coded risk indicators
- Same-room PTO conflict detection (warning only, never blocks)
- Reports: Call-out Summary (with Mon/Fri analysis), PTO Summary, Combined Absence, By Person, By Date
- Excel export for reports and history log (SheetJS)
- Staff roster with employment status, separation reasons, PT weekly schedule
- Sortable staff table (by name, room/age group, shift, status, start date, type)
- Timeline — read-only shift view with sticky time ruler, nap windows
- Break Reference tab with clickable chips opening staff edit modal
- isActiveEmployee() correctly hides terminated staff after their end date
- Google Sheets sync (full JSON payload to/from Apps Script)
- JSON backup/restore + Excel roster export

### Known Issues / Bugs to Watch
- Time ambiguity: The `parseTo24()` + pair-aware end-time fix handles all current roster times correctly. Any new shift entered via the time picker (which outputs HH:MM 24hr) goes through `from24()` and is stored correctly. Test new edge-case shifts through `shiftDecimal()` if timeline bars look wrong.
- SheetJS loads async — if user clicks Export Excel immediately on page load it may say "still loading." This is by design; it resolves within 1–2 seconds normally.
- Google Sheets sync shows "Local only" if the Apps Script URL is unreachable (network/CORS). Data is always safe in localStorage.

## Immediate Priorities

### P1 — GitHub-native workflow (in progress)
- [ ] Establish Claude Code as the commit agent for this repo
- [ ] Confirm GitHub MCP connection is working end-to-end
- [ ] Define commit message convention: `"Feature: ..."`, `"Fix: ..."`, `"Refactor: ..."`
- [ ] Carlos confirms changes in this chat before any commit is made

### P2 — Pending Feature Requests
These came up during the build but weren't implemented yet:

- [ ] **VPK compliance indicator** — 4K1 and 4K2 require 2 teachers minimum for VPK. Currently flagged visually but no formal compliance report.
- [ ] **Staff notes in history** — When a note is added to a staff member's profile, it should optionally be recorded in the history log with a timestamp.
- [ ] **PTO partial-day display on calendar** — Partial PTO entries (left early, arrived late) currently save but the calendar doesn't distinguish them visually from full-day PTO.
- [ ] **PT schedule displayed on timeline** — Part-time staff with a `ptSchedule` show their base `shift` on the timeline rather than their actual working days. The timeline should grey out days they don't work.
- [ ] **Report date defaults** — Reports default to current month. Add a quick-select for "Last 30 days", "Last 90 days", "Year to date".
- [ ] **Attendance trend chart** — Visual chart in reports showing call-out frequency over time by staff member or room.

### P3 — Nice to Have
- [ ] **Print-optimized room schedule** — A clean printable daily roster view (no nav, no sidebars) for posting in classrooms.
- [ ] **Email digest** — Send a daily summary of absences to a management email address (could use Gmail MCP).
- [ ] **Bulk backfill tool** — A UI for entering multiple past events at once, rather than navigating date by date.
- [ ] **Staff photo** — Attach a photo to each staff profile (stored as base64 or a URL).
- [ ] **Substitute tracker** — Track when a float or outside sub covers a room and for which teacher.

## Architecture Considerations for Future Work

### If the app grows significantly
The single-file approach has served well but has limits. If the file exceeds ~4,000 lines or if multiple people need to edit simultaneously, consider:
- Splitting into `index.html`, `app.js`, `styles.css`
- Using a simple bundler (Vite) to keep single-file deployment
- Moving to a proper backend (Supabase or Firebase) instead of Google Sheets cell B1

### Google Sheets B1 cell storage limit
The entire app state is stored as a single JSON string in one cell. Google Sheets cells have a 50,000 character limit per cell. With 42 staff + growing history, monitor payload size. When the history array grows large (6+ months of daily events), consider pruning entries older than 12 months.

### Checking payload size
```javascript
JSON.stringify({staff, ptoData, history, blockedDays}).length
// Should stay under 40,000 characters comfortably
```

## Development Workflow (Claude Code)

1. Carlos describes a change or bug in the Claude chat
2. Claude explains the plan and asks for confirmation before touching any code
3. On approval, Claude reads the current `index.html` from GitHub, makes the targeted edit, runs a Node syntax check, then commits with a descriptive message
4. GitHub Pages auto-deploys within ~60 seconds
5. Carlos refreshes the live URL to verify

### Commit Message Convention
```
Feature: Add [description]
Fix: [Bug description] in [function/area]
Refactor: [What changed and why]
Docs: Update [file]
```

### Pre-commit checklist
- [ ] Node syntax check passes: `node --check <extracted-js>`
- [ ] All 5 report functions present: renderCalloutRpt, renderPersonRpt, renderPtoRpt, renderCombinedRpt, renderDateRpt
- [ ] `parseTo24` and `decimalTo12` present and unchanged
- [ ] `isActiveEmployee` present
- [ ] `autoResetDailyStatuses` wired in both init paths
- [ ] SheetJS loaded async (not as synchronous script tag)
