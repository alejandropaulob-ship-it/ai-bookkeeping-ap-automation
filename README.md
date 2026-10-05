# AI Bookkeeping & AP Automation

An AI-powered Accounts Payable (AP) automation workflow built with n8n, OpenRouter, Supabase, Gmail, and deterministic financial validation.

This project automates invoice intake, AI-powered document extraction, financial validation, duplicate detection, database storage, notifications, and exception handling.

## Project Overview

The system receives invoices through email, processes supported document formats, extracts structured invoice information using AI, validates the extracted financial data using deterministic workflow logic, checks for duplicates, stores the invoice record, and notifies the sender of the result.

The workflow is designed around an important principle:

> AI performs document understanding, while deterministic workflow logic performs financial validation.

This separation helps reduce the risk of accepting incorrect or inconsistent invoice data solely because an AI model produced a plausible result.

## Workflow Architecture

The high-level workflow is:

Invoice Email → Attachment Processing → File Type Routing → Document Conversion → AI Extraction → Parse & Normalize → Financial Validation → Extraction Quality Check → Status Routing → Duplicate Detection → Database Storage → Notification

The workflow supports:

- PDF
- JPG
- JPEG
- PNG
- HEIC
- HEIF

HEIC and HEIF files are routed through a dedicated conversion process before image extraction.

## Core Capabilities

### 1. Invoice Intake

Invoices are received through a Gmail-triggered workflow with attachments enabled.

The workflow identifies supported invoice files and separates invoice attachments before routing them according to their file type.

This allows different document formats to follow the appropriate processing path.

### 2. AI Invoice Extraction

The workflow uses an AI extraction layer to read invoice documents and return structured information.

The extraction process can identify:

- Vendor
- Invoice number
- Invoice date
- Due date
- Customer name
- Currency
- Payment method
- Payment reference
- Total amount
- Amount due
- Payment received
- Tax
- Line items
- Financial breakdown
- Document adjustments

The AI extraction instructions are designed to treat the document as the source of truth.

The extraction layer is explicitly instructed to:

- Never guess or invent information
- Preserve visible printed financial values
- Return null when information cannot be reliably read
- Distinguish invoice information from document or reference numbers
- Preserve separate financial rows
- Avoid performing accounting calculations unless explicitly instructed

This keeps the AI focused on document understanding rather than silently modifying the source document.

## 3. Deterministic Financial Validation

AI extraction is not treated as the final authority.

After extraction, the workflow independently validates the financial information using deterministic JavaScript logic.

The validation layer checks information including:

- Vendor
- Invoice number
- Invoice date
- Line items
- Amount breakdown
- Subtotal
- Subtotal after discount
- Tax
- Discount
- Additional charges
- Total
- Expected total
- Payment received
- Amount due
- Remaining balance

When sufficient information is available, the workflow independently calculates expected financial values and compares them against the extracted information.

This provides a second validation layer after AI extraction.

## 4. Amount Due Handling

The workflow specifically distinguishes between the full invoice total and the remaining amount due.

When an invoice contains a clearly printed final balance after payment, the printed value is preserved as the extracted amount due.

The validation layer can then compare:

- Printed amount due
- Calculated expected amount due
- Payment received
- Invoice total
- Discounts
- Additional charges

The workflow does not automatically replace the printed amount simply because the numbers do not reconcile mathematically.

This is important because real-world invoices may contain:

- Payments already received
- Discounts
- Additional charges
- Fees or surcharges
- Other adjustments
- Printed balances that require human review

When the printed financial information does not reconcile with the expected calculation, the invoice can be routed to `REVIEW_REQUIRED`.

## 5. Tax Validation

Tax is extracted directly from the invoice.

The workflow is designed to preserve the printed tax amount rather than automatically replacing it with a calculated value.

The extracted tax is also included in the financial breakdown so it can be validated downstream.

When a tax rate is available, the validation layer can independently calculate the expected tax and compare it against the extracted tax amount.

If the tax amount cannot be reliably read, the extraction layer can return null rather than guessing.

## 6. Extraction Quality & Bounded Retry

The workflow includes an extraction-quality check for cases where the AI may incorrectly merge multiple financial lines.

One specific condition monitored by the workflow is:

`Multiple financial lines`

When this condition is detected, the workflow can perform a second extraction attempt.

The retry process:

1. Increases the retry count.
2. Stores retry metadata.
3. Provides additional instructions to the AI.
4. Requests that visible financial rows remain separate.
5. Requests that document numbers and reference numbers are not confused with invoice numbers.
6. Sends the result back through the extraction and parsing process.

The retry behavior is intentionally bounded so that an invoice cannot enter an uncontrolled extraction loop.

## 7. Invoice Status Routing

After extraction and validation, the workflow determines the appropriate processing status.

### VALID

Invoices that pass the current automated validation checks can continue through the normal processing path.

The invoice can then proceed to duplicate checking, database storage, and confirmation notification.

### REVIEW_REQUIRED

Invoices containing validation issues, ambiguous information, or extraction-quality problems can be routed for human review.

The review path prevents questionable invoice data from being treated as automatically accepted.

The workflow can provide information such as:

- Vendor
- Invoice number
- Invoice date
- Customer
- Printed amount due
- Calculated expected amount
- Extraction issues
- Source file

No final automated processing decision is made for invoices routed to review.

## 8. Duplicate Invoice Detection

Before a normal invoice is saved as a new record, the workflow creates an invoice fingerprint using normalized invoice information.

The fingerprint includes values such as:

- Vendor
- Invoice number
- Invoice date
- Currency
- Total amount

These values are normalized and combined into the invoice fingerprint.

The fingerprint is then checked against existing invoice records in Supabase.

If a matching record already exists:

- A duplicate notification is sent.
- The existing record is preserved.
- A new invoice record is not created.

This provides a deterministic duplicate-detection layer before database insertion.

## 9. HEIC and HEIF Processing

HEIC and HEIF invoice images are handled through a dedicated conversion path.

The workflow:

1. Identifies HEIC or HEIF files.
2. Sends the original image to the HEIF conversion service.
3. Converts the image to JPEG.
4. Normalizes the resulting file metadata.
5. Processes the converted image through the invoice extraction workflow.

The workflow also maintains a document-level fingerprint for HEIC/HEIF documents.

This allows the system to detect previously processed documents even when they follow the specialized HEIC/HEIF processing path.

Non-duplicate HEIC/HEIF documents currently continue through the database path with a `manual_review` status.

## 10. Database Storage

Invoice records are stored in Supabase.

The normal invoice processing path currently stores information including:

- Invoice fingerprint
- Vendor
- Invoice number
- Invoice date
- Customer name
- Currency
- Total amount
- Tax
- Amount breakdown
- Source file
- Sender email
- Processing timestamp
- Extraction status

The database acts as the persistent record used by the workflow for invoice storage and duplicate checking.

## 11. Automated Notifications

The workflow provides automated email notifications based on the processing result.

### Valid Invoice

When an invoice passes the current automated validation checks, the workflow sends a confirmation email to the original sender.

The confirmation can include:

- Vendor
- Invoice number
- Invoice date
- Invoice total
- Payment received
- Final total due
- Processing status

### Review Required

When an invoice requires human review, the workflow sends a review-required notification.

The notification can include:

- Vendor
- Invoice number
- Invoice date
- Customer
- Printed amount due
- Calculated expected amount
- Extraction issues
- Source file

### Duplicate Invoice

When a duplicate invoice is detected, the workflow sends a duplicate notification instead of creating another invoice record.

The notification can include:

- Vendor
- Invoice number
- Invoice date
- Customer
- Total
- Existing record ID
- Original file

### Duplicate HEIC/HEIF Document

HEIC/HEIF documents that match an existing document fingerprint are also routed to a dedicated duplicate notification path.

The existing record is preserved and no new record is created.

## 12. Error Handling

The project includes a separate error-handling workflow for unexpected execution failures.

The error workflow uses an n8n Error Trigger and sends an email alert when a workflow execution fails unexpectedly.

The alert includes information such as:

- Workflow name
- Execution ID
- Last node executed
- Error message
- Execution URL

This provides an operational monitoring layer separate from normal business outcomes.

The workflow distinguishes between:

- Expected outcomes such as `VALID`
- Expected exceptions such as `REVIEW_REQUIRED`
- Duplicate detection
- Unexpected technical execution failures

This separation helps prevent normal invoice review conditions from being treated as system failures.

## Technology Stack

| Component | Purpose |
|---|---|
| n8n | Workflow automation and orchestration |
| OpenRouter | AI API routing |
| OpenAI GPT-4.1-nano | AI invoice extraction |
| Gmail | Invoice intake and email notifications |
| Supabase | Invoice data storage and duplicate checking |
| HEIF Converter | HEIC and HEIF image conversion |
| GitHub | Source control and project documentation |

## Repository Structure

The repository is organized to separate workflow exports, documentation, and configuration examples.

- `README.md` — Project overview and technical documentation
- `.gitignore` — Prevents local configuration, credentials, and runtime files from being committed
- `.env.example` — Example configuration template
- `workflows/` — Exported n8n workflows
- `docs/` — Detailed technical documentation

Documentation currently includes:

- Workflow architecture
- AI extraction
- Financial validation logic
- Duplicate detection
- Database storage
- Notifications and review handling
- Error handling
- Security and configuration

## Security & Configuration

Credentials and secrets should not be committed to the repository.

Examples include:

- Gmail credentials
- Supabase credentials
- OpenRouter API keys
- OAuth credentials
- API tokens
- Environment secrets

Actual credentials remain managed by the n8n instance.

The public error-handler workflow export uses a placeholder for the alert email rather than exposing a personal email address.

The `.env.example` file provides example configuration values for deployment planning and documentation. It does not contain real credentials.

## Project Highlights

| Area | Implementation |
|---|---|
| Document intake | Gmail-triggered invoice processing |
| File support | PDF, JPG, JPEG, PNG, HEIC, HEIF |
| AI extraction | OpenRouter with OpenAI GPT-4.1-nano |
| Validation | Deterministic financial validation |
| Retry handling | Bounded second extraction attempt |
| Duplicate detection | Normalized invoice fingerprint |
| HEIC/HEIF duplicates | Document-level fingerprint |
| Database | Supabase |
| Notifications | Gmail |
| Error monitoring | Dedicated n8n Error Trigger workflow |
| Source control | GitHub |

## Current Project Status

The core invoice-processing workflow has been developed and tested across multiple invoice scenarios.

Current project focus includes:

- Source control
- Workflow documentation
- Validation hardening
- Error handling
- Quality assurance testing
- Deployment preparation
- Production-readiness improvements

## Important Note

This project is an automation and document-processing system.

Automated validation does not replace appropriate accounting review.

Documents that fail validation, contain ambiguous information, or produce unexpected results should be reviewed by a qualified human before being used for final accounting or payment decisions.
