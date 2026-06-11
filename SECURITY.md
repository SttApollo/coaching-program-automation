# Security

## Credential management

All credentials (Google OAuth, Anthropic API key, Trello API key, Apps Script shared secret) are stored in n8n's encrypted credential store (AES-256 at rest). No credentials appear in the workflow JSON files, `.env` files committed to version control, or any other plaintext location.

The `.gitignore` in this repository excludes `.env` files. The sanitized workflow JSON in this repository has all real credential references, node IDs, account identifiers, and client-specific URLs removed or replaced with `YOUR_*` placeholders. It is safe to inspect but cannot be imported and run without supplying real credentials.

## Least-privilege design

Each integration uses the minimum OAuth scope required:

- **Google Calendar** — read events + create/delete holds only; no access to Gmail, Contacts, or Drive beyond what is explicitly granted
- **Google Sheets** — read + write to the session log sheet only
- **Google Drive** — copy file + delete file; scoped to the target folder
- **Trello** — card comment creation + label updates on the specified board; no board admin access
- **Apps Script web app** — protected by a shared secret passed in the request body; the endpoint validates the secret before executing any operation

No single credential has access to more than one system.

## What the sanitized workflow JSON contains

The `workflow-sanitized.json` and `error-handler-sanitized.json` files in this repository are redacted exports of the production workflow. They contain:

- Node structure and connection graph
- JavaScript logic in Code nodes
- Expression patterns and configuration

They do not contain:

- Real n8n credential IDs
- Google account identifiers or OAuth tokens
- Trello board, list, or label IDs
- Apps Script deployment URLs or shared secrets
- Client names, emails, or Drive folder URLs
- Any data from real executions



