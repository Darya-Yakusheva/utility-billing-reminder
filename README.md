# Utility Billing Reminders

An automated data pipeline designed to parse incoming utility bills, ensuring timely payments and significantly reducing the cognitive load of tracking financial deadlines manually.

## Input

*   **Source:** Gmail Trigger.
*   **Trigger Logic:** The system checks the inbox at set intervals or in real-time for emails matching a specific filter, such as `noreply@aguasdoporto.pt subject:" Envio de Documento de Pagamento <...>"`.
*   **Payload:** Downloads the attached binary PDF document for further processing.

## Process

1.  **File Extraction:** Converts the binary data of the PDF attachment into unstructured raw text.
2.  **Parsing:** Utilizes Regular Expressions (Regex) or a JSON model to extract specific target values from the text block.
3.  **Data Points Extracted:** The system accurately isolates the Billing Period (`Período Faturação`), the Amount Due (`Valor a Pagar`), and the Due Date (`Pagar até`). It then converts the start date of the billing period into a formatted, readable month string (e.g., "May").

## Output

*   **Data Storage and Deduplication (Google Sheets):** The pipeline uses an "Update or Insert" operation to log the invoice details. The full Billing Period acts as a unique key; if this exact period already exists in the database, the system halts to prevent duplicate task creation.
*   **Task Management (Todoist):** For strictly unique invoices, a task is created in the designated project.
*   **Task Formatting:** The task name dynamically includes the specific billing month and the exact amount due (e.g., "Pay water bill for May for 32.45 EUR"), while the due date is mapped directly to the task calendar. The task description includes a URL link to the saved PDF invoice.

## Tools

*   **n8n:** Core automation engine.
*   **Gmail API:** For email interception.
*   **Google Sheets API:** For database management and deduplication.
*   **Todoist API:** For deadline tracking and task creation.