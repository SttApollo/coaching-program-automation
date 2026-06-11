# coaching-program-automation
n8n workflow that auto-schedules multi-sprint coaching sessions across timezones, generates branded schedule documents, and routes everything through a Trello approval flow — with calendar conflict detection and cascading  reschedules.

A fully automated session scheduling system built for a neuroscience-backed leadership coaching program. A single form submission by the operations team produces three things:

1. **A filled schedule document** — a branded Google Doc with all session dates and times populated, copied from a master template and placed directly in the client's Drive folder
2. **Calendar holds** — tentative blocks created on the coach's Google Calendar for every session across all sprints
3. **A Trello approval card** — the schedule doc link and a drafted client email posted to the project board, ready for the coach to review and approve

The workflow reads a live calendar, calculates all session slots against timezone constraints and blackout dates, and handles cascading reschedules — all triggered by a single form submission with zero manual steps in between.

---

## The Problem

The operations team was manually scheduling 4–7 coaching sessions per client sprint across multiple systems: Google Calendar, a shared spreadsheet, a branded Google Doc schedule, and Trello. Each new client took hours to set up. Rescheduling was even harder — it meant deleting calendar holds one by one, updating the spreadsheet, regenerating the doc, and reposting to Trello.

The system needed to handle:
- A rolling 9-sprint program where sprints 1, 5, and 9 have a different session structure
- The coach's fixed availability window (11am–3pm MT, weekdays only)
- Client time zones across 8 zones (ET through Sydney)
- Client blackout dates that persist and merge across reschedules
- A minimum 21-day gap from the first session to the last session within each sprint
- Cascading reschedules — moving one sprint forward means all subsequent sprints recalculate

---

## Architecture

Two completely separate trigger paths in a single n8n workflow:

```
NEW CLIENT PATH
Form (new client) → Check existing sessions → Get calendar events
    → Schedule builder (JS) → Create calendar holds → Log to Sheet
    → Copy Doc template → Fill placeholders (Apps Script)
    → Draft email → Post to Trello → Update Trello label

RESCHEDULE PATH
Form (reschedule) → Get existing Sheet rows → Filter from sprint N
    → Delete old calendar holds → Delete old Sheet rows → Delete old Doc
    → Get calendar events → Schedule builder (JS) → Create calendar holds
    → Log to Sheet → Copy Doc template → Fill placeholders (Apps Script)
    → Draft email → Post to Trello → Update Trello label
```

**Integrations:** n8n Cloud · Google Calendar · Google Sheets · Google Drive · Google Apps Script · Trello

---

## Build Approach

### Start with the data model, not the nodes

Before touching n8n, the first question was: what does the reschedule path need to read back? That determined what the new client path had to write. The Sheet column structure was designed around reschedule requirements, Calendar Event ID needed for deletion, and Client Drive Folder URL, so the reschedule path doesn't need to ask for it again.


### Prove the hard part first

The scheduling algorithm was the highest-risk piece. It was built and tested independently before being connected to other systems. Only once slot-finding was reliable with real timezone data did the integrations get layered in.

### Two forms, not a Switch node

Initial architecture used a single form with a Switch node to route new clients vs. rescheduling. This was scrapped. Two completely separate `formTrigger` nodes with custom URL paths are simpler, more reliable, and easier to debug — each path is fully independent with no shared state at the trigger level.

---

## Key Technical Challenges

### Timezone overlap without a timezone library

n8n Cloud's Code node sandbox blocks `luxon`. All timezone math is implemented with native `Date` + `Intl.DateTimeFormat`. The slot finder converts candidate times to both the coach's timezone (MT) and the client's timezone and checks each window independently — a slot is only valid if it fits inside 11 am–3 pm MT **and** 9 am–5 pm in the client's local time.

### Sprint spacing rules with a 21-day constraint

Each session must be at least 7 days after the previous one, but there's an additional constraint: the Embed session must be at least 21 days after the first session of the sprint (Activate or Crystallize+Activate). The algorithm tracks `firstSessionActual` per sprint and enforces the 21-day floor before searching for the Embed slot.

### Reschedule cascade without row index corruption

Deleting Sheet rows while iterating over them shifts all subsequent row numbers. The fix: sort the rows to delete in **descending** order by row number before the delete loop runs, so deletions at higher indices don't affect the indices of rows still to be deleted.

### Google Docs API — no OAuth in HTTP Request nodes

n8n's HTTP Request node doesn't support Google OAuth credentials directly. The workaround: a standalone Google Apps Script deployed as a public web app handles all Doc placeholder replacement. n8n POSTs a `batchUpdate`-style payload + a shared secret; the Apps Script validates the secret and calls the Docs API server-side.

### `alwaysOutputData` on the session check node

The duplicate submission check reads the Sheet for existing rows before scheduling. For a new client, this returns 0 rows — which in n8n stops execution at that node. Setting `alwaysOutputData: true` on the Sheets node ensures the flow continues even when no rows are found. The downstream Code node already handles an empty result correctly.

---

## Error Handling & Resilience

### Per-node error strategy

Not all failures are equal. The `Delete calendar holds` node (reschedule path) is set to `continueOnFail` — if an event ID is stale or already deleted, the reschedule shouldn't abort. All Sheet append nodes use `retryOnFail` since Sheets API has occasional rate limits.

### Dedicated error workflow

A separate `Error Handler` workflow is registered as the error handler for the main scheduling workflow. When any execution fails, it:

1. Detects which path failed (new client vs reschedule) from the failing node name
2. Formats a structured error card
3. Creates a card in the **System Errors** lane on the project management board

```
Error Trigger → Format error card (Code) → Create Trello card
```

The path detection logic:
```javascript
const isReschedule = nodeName.includes('Reschedule')
  || nodeName === 'Form1'
  || nodeName === 'redirect to form1';
```

Error card content includes: which path failed, failing node name, error message, execution ID, execution mode (real submission vs test run), timestamp, and a direct link to the execution in n8n.

---

## Security & Safety

### Access code on both form triggers

Both the new client form and the reschedule form require an access code validated in the first Code node before any data is read or written. The check runs before the calendar reads, sheet writes, or any external API call.

### Input validation before execution

The new client Code node validates all inputs before touching any external system:
- Email format (regex)
- Google Drive URL contains `/folders/`
- Trello URL contains `trello.com/c/`
- All blackout dates match `YYYY-MM-DD` format

If any check fails, execution stops with a descriptive error before any holds are created or rows are written.

### Duplicate submission guard

Before scheduling, the workflow queries the Sheet for existing sessions matching the client email. If rows are found for the same client email and phase number, execution stops with an error directing the Journey Master to use the reschedule form. This prevents double-scheduling while allowing returning clients to be onboarded for a new phase.

### Descending row deletion

Sheet rows are deleted in descending order by row number. This is a data integrity measure — deleting in ascending order would shift remaining row indices mid-loop, causing the wrong rows to be deleted.

---

## Tech Stack

| Tool | Role |
|---|---|
| n8n Cloud | Workflow engine |
| Google Calendar API | Read busy blocks, create/delete holds |
| Google Sheets API | Session log (read + write) |
| Google Drive API | Copy Doc template |
| Google Apps Script | Doc placeholder replacement (Docs API workaround) |
| Trello API | Post schedule + email draft for approval |
| JavaScript (ES2020) | Scheduling algorithm, timezone math, replacement map |

---

## Key Design Decisions

**No luxon in Code nodes** — n8n Cloud's sandbox blocks it. All timezone handling uses native `Date` + `Intl.DateTimeFormat`. More verbose but fully portable.

**Apps Script for Docs API** — Google OAuth credentials can't be used in n8n's HTTP Request node. A standalone Apps Script deployed as a public web app handles Doc manipulation server-side. n8n authenticates with a shared secret.

**Two form triggers, not a Switch** — Simpler to reason about, easier to test independently, no shared state at the trigger level.

**Sheet as source of truth for reschedule** — The reschedule form only asks for client email, new start date, and the existing Doc URL. Everything else (client name, timezone, phase, drive folder) is read back from the Sheet. This minimises the chance of the operator entering mismatched data.

**Blackout dates stored and merged** — Client blackout dates are written to the Sheet on every run and merged with new entries on each reschedule. The scheduler always works from the full accumulated set.

