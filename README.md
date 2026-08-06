# Utility Billing Reminders

An automated data pipeline designed to parse incoming utility bills, ensuring
timely payments and significantly reducing the cognitive load of tracking
financial deadlines manually.

n8n automation for **Águas do Porto** water bills: catch the invoice email, skip duplicates, parse the PDF, save the invoice to Google Drive, log it, and create Todoist tasks for payment and meter reading — or email an alert with a Drive link if required fields fail to parse.

## Pipeline

```
Gmail Trigger
  → Lookup email ID in sheet
  → Email is new?
       ├─ false → stop (already processed; no Drive/Sheet/Todoist)
       └─ true  → Extract invoice PDF text
                    → Parse invoice fields
                    → Upload invoice to Drive
                    → Required fields found?
                         ├─ true  → Upsert invoice row (incl. Invoice link)
                         │            ├─ Create payment task → Write payment task ID to sheet
                         │            └─ Create meter reading task → Write meter reading task ID to sheet
                         └─ false → Send parse failure alert (Drive file link)
```

## Input

- **Source:** Gmail Trigger (poll ~daily)
- **Filters:** sender `noreply@aguasdoporto.pt`, unread, `has:attachment`
- **Payload:** PDF attachment downloaded and converted to text

## Deduplication

Before PDF extract/parse, the workflow looks up the Gmail message **Email ID** in Google Sheets.

- If the ID already exists → stop (no second file, row, or Todoist tasks)
- If not → continue with extract → parse → upload → success/fail paths

## Parsing

Regex extraction from the invoice text:

| Field | Invoice cue |
| --- | --- |
| Invoice Date | `Emissão` (fallback: Gmail message `date`, `YYYY-MM-DD`) |
| Billing Period | `Período Faturação` |
| Amount | `Valor a Pagar` |
| Due Date | `Pagar até` |
| Billing Month | derived from period start (e.g. `May`) |
| Meter Reading Window | `Período de Comunicação` |

Missing required fields become `"Not found"`. If **Billing Period**, **Amount**, or **Due Date** is missing → fail path (alert only; no Sheet/Todoist). A missing **Meter Reading Window** does **not** fail the run: the meter-reading task is still created (without a due date) and asks for a manual check. **Invoice Date** always resolves (PDF or Gmail date) so the Drive filename stays usable.

## Output

### Always after a new email (before success/fail split)

- **Google Drive:** PDF uploaded as `water {{Invoice Date}}.pdf`
- File id used for links on both paths

### Success path

- **Google Sheets:** upsert by **Email ID** (includes **Invoice Date** and **Invoice link**)
- **Todoist — payment:** month + amount in the title, due date from the invoice, billing period in the description; **Payment Task ID** written back to the sheet
- **Todoist — meter reading:** always created after a successful upsert
  - If the window was parsed → description includes the window + portal link; due date = **end** of the window
  - If the window is `"Not found"` → no due date; description asks to check manually (Gmail deep link + portal link); **Meter Reading Task ID** still written to the sheet

### Fail path

- **Email alert** with a link to the uploaded Drive file (not only the Gmail message)
- No Sheet row, no Todoist tasks

## Tools

- **n8n** — orchestration (Docker)
- **Gmail API** — trigger + parse-failure alerts (OAuth2)
- **Google Drive API** — invoice PDF archive (OAuth2)
- **Google Sheets API** — archive + early dedupe + upsert (service account)
- **Todoist API** — payment and meter-reading deadlines

## Workflow export

Sanitized n8n workflow (no credentials or personal resource IDs):

[`workflows/utility-billing.json`](workflows/utility-billing.json)

Import into your own n8n, then attach Gmail / Google Drive OAuth2 / Google Service Account / Todoist credentials and select your Drive folder, Sheet, Todoist project, alert recipient, and meter-reading portal URL.
