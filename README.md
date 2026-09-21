# AI Bookkeeping & AP Automation

An AI-powered Accounts Payable (AP) automation workflow built with n8n, OpenAI, OpenRouter, Supabase, and Gmail.

The system receives invoices through email, extracts invoice data using AI, validates the extracted financial information, detects potential duplicates, stores invoice records, and automatically notifies the sender when an invoice requires review.

## Overview

The workflow follows this general process:

Invoice Email → Receive Invoice Email → Split Invoice Attachments → Route by File Type → Document Processing → AI Invoice Extraction → Parse & Normalize Extracted Data → Invoice Validation → Extraction Quality Check → Duplicate Invoice Check → Save Invoice Record / Review Required

Supported file types:

- PDF
- JPG
- JPEG
- PNG
- HEIC
- HEIF

## Core Capabilities

### Invoice Intake

Invoices are received through a Gmail-triggered workflow with attachments enabled.

The workflow identifies supported invoice attachments and routes them according to their file type.

### AI Invoice Extraction

The system uses AI to extract information including:

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

The extraction instructions require the AI to use the document as the source of truth and avoid guessing or inventing missing information.

## Financial Validation

Extracted invoice data is independently validated by downstream workflow logic.

The system distinguishes between:

- Printed invoice values
- Calculated expected values
- Payments already received
- Remaining amount due

Printed values are not automatically corrected during extraction.

If the printed financial information does not reconcile with the expected calculation, the invoice can be routed to REVIEW_REQUIRED instead of being automatically accepted.

This validation is particularly important for:

- Incorrect printed totals
- Incorrect tax amounts
- Incorrect remaining balances
- Discounts
- Fees and surcharges
- Payments already received
- Other financial inconsistencies

## Amount Due Handling

The workflow specifically distinguishes between the full invoice total and the remaining amount due.

When an invoice contains a clearly printed final balance after payment, that printed amount is preserved as the extracted amount_due.

The downstream validation layer can then compare the printed value against the expected financial calculation.

This prevents the AI extraction layer from silently correcting an invoice simply because its printed numbers do not mathematically reconcile.

## Tax Validation

Tax is extracted directly from the invoice.

The workflow is designed to preserve the printed tax amount rather than replacing it with a calculated value.

The extracted tax is also carried into the financial breakdown so that the same printed tax value can be validated downstream.

If the tax amount cannot be reliably read, the extraction can return null rather than guessing.

## Extraction Quality and Retry

The workflow includes an extraction-quality check for cases where the AI may have incorrectly merged multiple financial lines.

When the workflow detects "Multiple financial lines", it can perform a second extraction attempt.

The retry process increases the retry count and provides additional instructions to the AI to carefully re-read the document and preserve separate financial rows.

The workflow limits this retry behavior so that an invoice does not enter an uncontrolled extraction loop.

## Duplicate Invoice Detection

Before an invoice is saved as a new record, the workflow creates an invoice fingerprint using normalized invoice information.

The fingerprint uses values including:

- Vendor
- Invoice number
- Invoice date
- Currency
- Total amount

The fingerprint is then checked against existing invoice records in Supabase.

This allows the workflow to identify potential duplicate invoices before creating another invoice record.

## Database

Invoice records are stored in Supabase.

The workflow currently stores information including:

- Invoice fingerprint
- Vendor
- Invoice number
- Invoice date
- Customer
- Currency
- Total amount
- Tax
- Amount breakdown
- Source file
- Sender email
- Processing timestamp
- Extraction status

## Automated Notifications

### Valid Invoice

When an invoice passes the current automated validation checks, the workflow sends a confirmation email containing relevant invoice information and the validation status.

### Review Required

When an invoice fails validation or contains information requiring human review, the workflow sends a review-required notification.

The notification can include:

- Vendor
- Invoice number
- Invoice date
- Customer
- Printed amount due
- Calculated expected amount
- Extraction issues
- Source file

No final automated processing decision is made for invoices routed to review.

## HEIC and HEIF Processing

HEIC and HEIF invoice images are converted to JPEG before being passed to the image extraction stage.

The workflow uses a dedicated HEIF conversion service.

The converted image is normalized back into the workflow with a JPEG filename and MIME type.

## Error Handling

A separate error-handling workflow is included in the project.

The error workflow uses an n8n Error Trigger and sends an email notification when an execution fails.

The notification includes information such as:

- Workflow name
- Execution ID
- Last node executed
- Error message
- Execution URL

This provides an operational alerting layer for workflow failures.

## Technology Stack

| Component | Purpose |
|---|---|
| n8n | Workflow automation and orchestration |
| OpenAI | AI-based invoice extraction |
| OpenRouter | AI API routing |
| Gmail | Invoice intake and email notifications |
| Supabase | Invoice data storage and duplicate checking |
| HEIF Converter | HEIC and HEIF image conversion |
| GitHub | Source control and project documentation |

## Project Structure

The repository is intended to use the following structure:

- README.md
- .gitignore
- workflows/
- docs/

The workflows directory will contain the exported n8n workflows.

The docs directory will contain project documentation such as workflow architecture, invoice validation logic, and deployment instructions.

## Security

Credentials and secrets must not be committed to this repository.

Examples include:

- Gmail credentials
- Supabase credentials
- OpenRouter API keys
- API tokens
- OAuth credentials
- Environment secrets

Actual credentials should remain managed by the n8n instance.

## Current Project Status

The core invoice-processing workflow has been developed and tested across multiple invoice scenarios.

Current development focus:

- Source control
- Workflow documentation
- Validation hardening
- Error handling
- Testing
- Deployment preparation
- Production-readiness improvements

## Important Note

This project is an automation and document-processing system.

Automated validation does not replace appropriate accounting review. Documents that fail validation or contain ambiguous information should be reviewed by a qualified human before being used for final accounting or payment decisions.
