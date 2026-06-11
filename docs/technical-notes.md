# Technical Notes

Implementation details behind the less obvious parts of the scheduling workflow.

---

## Timezone overlap without a timezone library

n8n Cloud's Code node sandbox blocks `luxon`. All timezone math is implemented with native `Date` + `Intl.DateTimeFormat`. The slot finder converts candidate times to both the coach's timezone (MT) and the client's timezone and checks each window independently — a slot is only valid if it fits inside 11am–3pm MT **and** 9am–5pm in the client's local time.

```javascript
function fromTZComponents(dateStr, hour, minute, tz) {
  // Reconstruct a UTC instant from local components without luxon
  // by iterating offsets until the local representation matches
  const [year, month, day] = dateStr.split('-').map(Number);
  const approxUtc = new Date(Date.UTC(year, month - 1, day, hour, minute));
  for (let offset = -840; offset <= 840; offset++) {
    const candidate = new Date(approxUtc.getTime() + offset * 60000);
    const local = toTZ(candidate, tz);
    if (local.year === year && local.month === month &&
        local.day === day && local.hour === hour && local.minute === minute) {
      return candidate;
    }
  }
  return approxUtc;
}
```

This handles DST transitions correctly because it validates against `Intl` output rather than assuming a fixed offset.

---

## Sprint spacing rules with a 21-day constraint

Each session must be at least 7 days after the previous one, but there's an additional constraint: the Embed session must be at least 21 days after the first session of the sprint (Activate or Crystallize+Activate). The algorithm tracks `anchorSessionStart` per sprint and enforces the 21-day floor before searching for the Embed slot.

---

## Reschedule cascade without row index corruption

Deleting Sheet rows while iterating them shifts all subsequent row numbers. The fix: sort the rows to delete in **descending** order by row number before the delete loop runs, so deletions at higher indices don't affect the indices of rows still to be deleted.

---

## Calendar lookup window

The Google Calendar read uses `timeMax = now + 12 months`. This was initially set to 5 months, which would silently miss conflicts for clients in later sprints of a 9-sprint program (which runs ~8.5 months). The window was extended after calculating the worst-case schedule length.

---

## Google Docs API — no OAuth in HTTP Request nodes

n8n's HTTP Request node doesn't support Google OAuth credentials directly. The workaround: a standalone Google Apps Script deployed as a public web app handles all Doc placeholder replacement. n8n POSTs a `batchUpdate`-style payload + a shared secret; the Apps Script validates the secret and calls the Docs API server-side.

---

## `alwaysOutputData` on the session check node

The duplicate submission check reads the Sheet for existing rows before scheduling. For a new client this returns 0 rows — which in n8n stops execution at that node. Setting `alwaysOutputData: true` on the Sheets node ensures the flow continues even when no rows are found. The downstream Code node already handles an empty result correctly.
