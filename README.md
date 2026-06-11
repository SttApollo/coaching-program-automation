
A fully automated session scheduling system built for a neuroscience-backed leadership coaching program. A single form submission by the operations team produces three things:

1. **A filled schedule document** — a branded Google Doc with all session dates and times populated, copied from a master template and placed directly in the client's Drive folder
2. **Calendar holds** — tentative blocks created on the coach's Google Calendar for every session across all sprints
3. **A Trello approval card** — the schedule doc link and a drafted client email posted to the project board, ready for the coach to review and approve

The workflow reads a live calendar, calculates all session slots against busy blocks, timezone constraints and blackout dates, and handles cascading reschedules — all triggered by a single form submission with zero manual steps in between.

---

## The Problem

The operations team was manually scheduling 4–7 coaching sessions per client sprint, across multiple systems: Google Calendar, a shared spreadsheet, a branded Google Doc schedule, and Trello. Each new client took 30–45 minutes to set up. Rescheduling was even harder — it meant deleting calendar holds one by one, updating the spreadsheet, regenerating the doc, and reposting to Trello.

The system needed to handle:
- A rolling 9-sprint program where sprints 1, 5, and 9 have a different session structure
- Lisa's fixed availability window (11am–3pm MT, weekdays only)
- Client timezones across 8 zones (ET through Sydney)
- Client blackout dates that persist and merge across reschedules
- A minimum 21-day gap from first session to Embed within each sprint
- Cascading reschedules — moving sprint 3 forward means all subsequent sprints recalculate

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


  ```mermaid
  flowchart TD
      subgraph NEW["New Client Path"]
          direction TB
          A([Form Trigger]) --> B[Respond & Redirect]
          B --> C[Check Existing Sessions\nGoogle Sheets]
          C --> D[Read Calendar Events\nGoogle Calendar]
          D --> E[Schedule Builder\nJavaScript]
          E --> F[Create Calendar Holds\nGoogle Calendar]
          F --> G[Log Sessions to Sheet\nGoogle Sheets]
          G --> H[Copy Doc Template\nGoogle Drive]
          H --> I[Build Replacement Map\nJavaScript]
          I --> J[Fill Doc Placeholders\nApps Script]
          J --> K[Draft Approval Email\nJavaScript]
          K --> L[Post to Trello\nTrello]
          L --> M[Update Approval Label\nTrello]
      end
  
      subgraph RESCHED["Reschedule Path"]
          direction TB
          N([Form Trigger]) --> O[Respond & Redirect]
          O --> P[Get Client Sessions\nGoogle Sheets]
          P --> Q[Filter From Sprint N\nJavaScript]
          Q --> R[Delete Calendar Holds\nGoogle Calendar]
          R --> S[Sort Rows Descending\nJavaScript]
          S --> T[Delete Sheet Rows\nGoogle Sheets]
          T --> U[Delete Old Doc\nGoogle Drive]
          U --> V[Read Calendar Events\nGoogle Calendar]
          V --> W[Schedule Builder\nJavaScript]
          W --> X[Create Calendar Holds\nGoogle Calendar]
          X --> Y[Log Sessions to Sheet\nGoogle Sheets]
          Y --> Z[Copy Doc Template\nGoogle Drive]
          Z --> AA[Build Replacement Map\nJavaScript]
          AA --> AB[Fill Doc Placeholders\nApps Script]
          AB --> AC[Draft Approval Email\nJavaScript]
          AC --> AD[Post to Trello\nTrello]
          AD --> AE[Update Approval Label\nTrello]
      end
  
      subgraph ERR["Error Handler Workflow"]
          direction LR
          AF([Error Trigger]) --> AG[Detect Path & Format\nJavaScript]
          AG --> AH[Create Error Card\nTrello]
      end   

      NEW -.->|on failure| ERR
      RESCHED -.->|on failure| ERR 
  ```

---

## Build Approach

### Start with the data model, not the nodes

Before touching n8n, the first question was: what does the reschedule path need to read back? That determined what the new client path had to write. The Sheet column structure was designed around reschedule requirements — particularly Column J (Calendar Event ID, needed for deletion) and Column L (Client Drive Folder URL, so the reschedule path doesn't need to ask for it again).

### Map the full user journey before building

The workflow handles two distinct users: the Journey Master (who fills in the form) and Lisa (who approves the output in Trello). Every output — the schedule doc, the email draft, the Trello comment — was spec'd against what those users actually needed to see before a node was built.

### Prove the hard part first

The scheduling algorithm was the highest-risk piece. It was built and tested independently before connecting it to Calendar, Sheets, or Trello. Only once slot-finding was reliable with real timezone data did the integrations get layered in.

### Two forms, not a Switch node

Initial architecture used a single form with a Switch node to route new client vs reschedule. This was scrapped. Two completely separate `formTrigger` nodes with custom URL paths is simpler, more reliable, and easier to debug — each path is fully independent with no shared state at the trigger level.

---

## Technical Notes

  Timezone math without `luxon`, the 21-day sprint spacing constraint, descending row deletion, calendar window sizing, and the Apps Script workaround for the Docs API — see [docs/technical-notes.md](docs/technical-notes.md).

---

## Security & Safety

### Access code on both form triggers

Both the new client form and the reschedule form require an access code validated in the first Code node before any data is read or written. The check runs before calendar reads, Sheet writes, or any external API call.

### Input validation before execution

The new client Code node validates all inputs before touching any external system:
- Email format (regex)
- Google Drive URL contains `/folders/`
- Trello URL contains `trello.com/c/`
- All blackout dates match `YYYY-MM-DD` format

If any check fails, execution stops with a descriptive error before any holds are created or rows are written.

### Duplicate submission guard

Before scheduling, the workflow queries the Sheet for existing sessions matching the client email. If rows are found, execution stops with an error directing the Journey Master to use the reschedule form instead. This prevents double-scheduling the same client.

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

**Sheet as source of truth for reschedule** — The reschedule form only asks for client email, new start date, and the existing Doc URL. Everything else (client name, timezone, phase, drive folder) is read back from the Sheet. This minimises the chance of the Journey Master entering mismatched data.

**Blackout dates stored and merged** — Client blackout dates are written to the Sheet on every run and merged with new entries on each reschedule. The scheduler always works from the full accumulated set.
