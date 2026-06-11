# Governance, Security & Ethics

Documents how data is handled, how access is controlled, where AI is used and how it is overseen, and how the system is operated and maintained responsibly.

---

## 1. Scope

**In scope:** This document covers the scheduling automation workflow, its integrations (n8n Cloud, Google Calendar, Google Sheets, Google Drive, Google Apps Script, Trello, Anthropic API), and the operational practices around it.

**Out of scope:** Payment processing, session content and coaching notes, and downstream email delivery. The workflow generates an email draft — the operator sends the actual email manually after the coach approves it.

---

## 2. Data Inventory & Data Handling

### What is collected

| Field | Source | Classification |
|---|---|---|
| Client full name | Coordinator form input | PII |
| Client email address | Coordinator form input | PII |
| Client company name | Coordinator form input | Business information |
| Client timezone | Coordinator form input | Non-sensitive |
| Trello card URL | Coordinator form input | Internal reference |
| Sprint start date, blackout dates | Coordinator form input | Scheduling data |

No payment information, health data, session content, or coaching notes are collected or processed by this system.

### Where data lives

| Store | What it holds | Retention |
|---|---|---|
| n8n execution logs | Full execution input/output including PII | 30 days (n8n Cloud default — review in account settings) |
| Google Sheet (Session Data tab) | Name, email, company, timezone, session dates, calendar event IDs, Drive folder URL, blackout dates | Indefinite until manually archived |
| Google Doc (client Drive folder) | Client name, company name, session schedule | Client-owned; lifecycle managed by coach |
| Trello card comment | Client name, schedule doc link, AI-generated email draft | Board lifecycle managed by coach |
| Google Calendar | Calendar hold events (client name in title) | Deleted on reschedule; otherwise persist until coach removes |

### Who collects it

An operator submits the form on behalf of the client. Clients do not submit data directly — there is no client-facing form or portal.

### Data flow

```
Coordinator Form → n8n workflow
    → Google Calendar (create/delete holds)
    → Google Sheets (read/write session log)
    → Google Drive (copy Doc template)
    → Google Apps Script (fill Doc placeholders)
    → Trello (post approval card + email draft)
```

> **Data Handling Note:** This system applies three governing principles to all data it handles:
> - **Data minimization** — only the fields required for scheduling are collected. No fields exist for coaching notes, payment, or any other purpose.
> - **Purpose limitation** — collected data is used exclusively to produce a session schedule. It is not used for analytics, marketing, or any secondary purpose.
> - **No third-party sharing** — data is shared only with the named vendors in this document, under the terms of each vendor's data processing agreement.

---

## 3. Access Control

Access to all systems follows the principle of least privilege — each account and credential is scoped to the minimum permission needed for its function.

| System | Who has access | Permission scope |
|---|---|---|
| n8n Cloud | Coach account only (no shared logins) | Full workflow admin |
| Google Calendar | n8n credential (OAuth) | Calendar read + write (holds only) |
| Google Sheets | n8n credential (OAuth) | Sheets read + write (session log only) |
| Google Drive | n8n credential (OAuth) | Drive file copy + delete (Doc template + client folder) |
| Google Apps Script | Public web app URL + shared secret | Doc placeholder replacement only; no other Drive access |
| Trello | n8n credential (API key) | Card comments + label updates on specified board only |
| Anthropic API | n8n credential (API key) | Text generation only |

**Operator access:** The coordinator submits the scheduling form but has no direct access to n8n, the Google Sheet, or the Trello API credentials.

**No Gmail or Contacts access:** Google OAuth credentials are intentionally scoped to exclude Gmail and Contacts. The system drafts the email but does not send it.

**Offboarding:** On any team member departure, the following must be completed: rotate the Apps Script shared secret, rotate Google OAuth credentials in n8n, remove the departing member from the Trello board.

---

## 4. AI Use & Human-in-the-Loop (HITL) Gate

### What AI does

The Anthropic Claude API generates a client-facing email draft. Inputs to the API: coordinator name, client name, and a brief description of the coaching program context. No session dates, health data, financial data, or sensitive PII beyond the client's name are included in the API call.

### API provider data policy

Anthropic does not use API-submitted data to train its models by default. For enterprise data processing requirements, a Data Processing Agreement is available from Anthropic.

### The HITL Gate

**The workflow stops before sending any communication to the client.**

After generating the schedule and drafting the email, the workflow posts both to a Trello card labeled "Email Draft" and sets the card status to "Waiting for Coach Approval." No email is sent, no calendar hold is confirmed, and no client is notified until the coach manually reviews the output, makes any edits, and gives explicit approval.

The operator sends the email only after receiving that approval.

This gate is a deliberate architectural choice, not a limitation of the system. It ensures that:
- A human reviews every output before it reaches a client
- The coach retains full control over client communication
- Any inaccuracies in the draft are caught before they cause confusion

---

## 5. Action Log

### Automatic (system-generated)

| Log | Contents | Retention |
|---|---|---|
| n8n execution log | Full input/output for every run, timestamp, execution mode, status | 30 days |
| Error handler Trello card | Workflow name, failing node, error message, execution ID, timestamp, link to execution | Until manually archived from the board |

Every workflow execution — successful or failed — is logged in n8n automatically. No additional logging infrastructure is required.

### Manual (coordinator responsibility)

The following events are not logged automatically and should be noted as a comment on the Trello card:

- When the coach gives approval (date and any changes requested)
- When the coordinator sends the email to the client (date)
- When the client confirms the schedule (date)

This creates a lightweight audit trail using the Trello card that already exists in the workflow, without requiring a separate system.

---

## 6. Error Handling & Escalation Path

### Automated error detection

A dedicated error handler workflow is registered as the error handler for the scheduling workflow. When any execution fails, it automatically creates a card in the System Errors lane on the project board, including: which path failed (new client vs. reschedule), the failing node name, the error message, the execution ID, the execution timestamp, and a direct link to the n8n execution for debugging.

### Escalation path

```
Workflow fails
    ↓
Error card appears in System Errors lane (automatic)
    ↓
Developer reviews the error card
    ↓
Is it a data entry issue? (wrong email format, bad URL, mismatched client name)
    → YES: Correct the form data and resubmit. No developer needed.
    → NO: Escalate to developer with the Trello error card link.
          Developer diagnoses and fixes; coordinator resubmits when resolved.
    ↓
Does the error affect a client with an active or imminent session?
    → YES: Notify the coach immediately.
           The coordinator handles client communication manually while the fix is in progress.
    → NO: Normal resolution timeline applies.
```

### Per-node resilience

Not all node failures are treated equally:
- **Delete calendar holds** (reschedule path): set to `continueOnFail` — a stale or already-deleted event ID should not abort the entire reschedule
- **Sheet append nodes**: use `retryOnFail` — the Sheets API has occasional rate limits that resolve on retry

---

## 7. Change Management

All changes to the workflow go through the following process:

1. **Staging first** — changes are tested using separate test credentials and pinned mock data before being deployed to production
2. **Export before changing** — the workflow JSON is exported and saved locally before any change is made to the production workflow
3. **No changes during active scheduling windows** — if a client is mid-program and scheduling is in progress, production changes are deferred until a safe window
4. **Written change requests** — all change requests from the coach or coordinator are submitted in writing (email or Trello). Verbal requests are not actioned until confirmed in writing so there is a record
5. **Version log** — a version history is maintained in the project documentation noting what changed, when, and why

---

## 8. Credential Management

| Credential | Where stored | Type |
|---|---|---|
| Google OAuth (Calendar, Sheets, Drive) | n8n encrypted credential store | OAuth 2.0 |
| Anthropic API key | n8n encrypted credential store | API key |
| Trello API key + token | n8n encrypted credential store | API key |
| Apps Script shared secret | n8n encrypted credential store + Apps Script source | Shared secret |

All credentials are stored in n8n's encrypted credential store (AES-256 at rest). No credentials are stored in workflow JSON files, environment variable files committed to version control, or any other plaintext location.

**Recommended rotation schedule:**
- Annually as a standard practice
- Immediately upon any team member departure
- Immediately if a credential is suspected to be compromised

**Offboarding checklist:**
- Rotate Google OAuth credentials (revoke and regenerate in Google Cloud Console)
- Rotate the Apps Script shared secret (update in both Apps Script source and n8n credential)
- Rotate Anthropic API key
- Rotate Trello API key and token
- Remove departing member from Trello board
- Transfer n8n account ownership to coach (if developer is leaving)

---

## 9. Data Retention & Deletion

| Store | Retention policy | Who manages |
|---|---|---|
| n8n execution logs | 30-day default | Configure in n8n Cloud account settings |
| Google Sheet (session log) | Until manually archived | Coach/coordinator |
| Google Docs (schedule docs) | Indefinite — lives in client's Drive folder | Coach |
| Trello cards | Board lifecycle | Coach |
| Google Calendar holds | Deleted on reschedule; otherwise persist | Workflow (on reschedule) or coach (manual) |

**Recommended practice:** Archive (or delete) Google Sheet rows for completed clients annually. A client whose program is complete no longer needs active rows in the session log. Archiving reduces the risk of the log growing unbounded and keeps the Sheet performant.

A written data retention policy agreed with the coach is recommended before handling a high volume of clients.

---

## 10. Ethics & Responsible Automation

This section documents the ethical design choices made in this system.

### Human oversight by design

No automated action reaches an end client. The system produces drafts and calendar holds — it does not send emails, confirm bookings, or initiate any client-facing action. A human (the coach) reviews every AI output and every generated schedule before the coordinator takes the next step.

This design aligns with **NIST AI RMF 1.0 — Govern 1.1**: "Policies, processes, procedures, and practices across the organization related to the mapping, measuring, and managing of AI risks are in place, transparent, and implemented effectively" — and specifically with the framework's emphasis on human oversight as a primary control for AI-generated outputs.

### Data minimization

The system collects only the fields required to produce a session schedule. There are no fields for coaching notes, payment information, health status, session feedback, or any other category. This is not an oversight — it is a deliberate constraint.

### Purpose limitation

Data collected by this system is used exclusively to produce a session schedule and associated materials (Doc, calendar holds, Trello card). It is not shared with third parties beyond the named vendors in this document, not aggregated for analytics, and not used for any marketing or secondary purpose.

### Automation boundaries

The workflow ends at the Trello approval card. Sending the email is the operator's action, not the system's. This boundary is intentional: it is where responsibility transfers from the automation to a person. Removing that boundary — automating the email send — would eliminate the human review step.

These design choices collectively reflect **ISO/IEC 42001:2023** principles for responsible AI system design: human oversight, transparency, data minimization, and defined operational boundaries.

---

## 11. Measurement

### Operational metrics

| Metric | Target | How to measure |
|---|---|---|
| Time to schedule per client | < 5 minutes from form submission to Trello card appearing | n8n execution log (start → end timestamp) |
| Error rate | 0 unhandled failures per month | System Errors Trello lane |
| Approval turnaround time | Baseline to be established after go-live | Trello card (created timestamp vs. approval comment timestamp) |

### Review cadence

- **First 3 months post-launch:** Monthly check of n8n execution logs and System Errors Trello lane
- **Ongoing:** Quarterly review; adjust if error rate or scheduling volume changes significantly

### Success signal

The operator no longer needs to manually cross-reference four systems (Calendar, Sheet, Docs, Trello) to schedule a client. The primary measure of success is that the full scheduling process — from form submission to Trello approval card — completes without coordinator intervention.
