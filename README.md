# Utility Billing Reminders

An automated data pipeline designed to parse incoming utility bills, ensuring
timely payments and significantly reducing the cognitive load of tracking
financial deadlines manually.

n8n automation for **Águas do Porto** water bills: catch the invoice email, parse the PDF, log it, and create a Todoist payment task — or email an alert if parsing fails.

## Pipeline

```
Gmail Trigger
  → Extract from File (PDF)
  → Code: extract data (regex)
  → If (Billing Period, Amount, Due Date all found)
       ├─ true  → Google Sheets upsert → Todoist payment task → write Task ID back to Sheet
       └─ false → Gmail alert (manual review; no Sheet row, no Todoist task)
```

## Input

- **Source:** Gmail Trigger (poll ~daily)
- **Filters:** sender `noreply@aguasdoporto.pt`, unread, `has:attachment`
- **Payload:** PDF attachment downloaded and converted to text

## Parsing

Regex extraction from the invoice text:

| Field | Invoice cue |
| --- | --- |
| Billing Period | `Período Faturação` |
| Amount | `Valor a Pagar` |
| Due Date | `Pagar até` |
| Billing Month | derived from period start (e.g. `May`) |
| Meter Reading Window | `Período de Comunicação` (stored for a later version) |

Missing required fields become `"Not found"`. If **Billing Period**, **Amount**, or **Due Date** is missing → fail path (email alert only).

## Output (success path)

- **Google Sheets:** upsert by **Email ID** (dedupe key — one row per email/message)
- **Todoist:** payment task with month + amount in the title, due date from the invoice, billing period in the description
- **Sheet follow-up:** Payment Task ID written back to the same row

Invoice PDF → Google Drive / `Invoice link`, and a meter-reading Todoist task, are planned next — not in the live workflow yet.

## Tools

- **n8n** — orchestration (Docker)
- **Gmail API** — trigger + parse-failure alerts
- **Google Sheets API** — archive + dedupe
- **Todoist API** — payment deadlines

## Workflow export

Sanitized n8n workflow (no credentials or personal resource IDs):

[`workflows/utility-billing.json`](workflows/utility-billing.json)

Import into your own n8n, then attach Gmail / Google Service Account / Todoist credentials and select your Sheet, Todoist project, and alert recipient.
