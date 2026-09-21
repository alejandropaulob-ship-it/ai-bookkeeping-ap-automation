# Workflow Architecture

## Overview

The AI Bookkeeping & AP Automation workflow is an n8n-based invoice processing system that receives invoices through email, extracts structured financial information using AI, validates the extracted data independently, detects duplicates, stores validated records in Supabase, and sends processing notifications.

The workflow is designed so that AI extraction is not treated as the final source of truth. Extracted financial information is independently checked by deterministic validation logic before an invoice is allowed to continue as VALID.

## High-Level Flow

<pre>
Gmail Invoice Trigger
        |
        v
Split Invoice Attachments
        |
        v
Route by File Type
   |        |        |        |        |
   v        v        v        v        v
  PDF      JPG      PNG     HEIC     HEIF
   |        |        |        |        |
   |        |        |        |        v
   |        |        |        |   HEIF Conversion
   |        |        |        |        |
   |        |        |        v        |
   |        |        |    HEIC Conversion
   |        |        |        |        |
   +--------+--------+--------+--------+
                    |
                    v
          AI Invoice Extraction
                    |
                    v
         Parse / Clean Extraction
                    |
                    v
          Validate Invoice Data
                    |
                    v
         Check Extraction Quality
              |             |
              |             |
           Retry         Continue
              |             |
              v             v
     Prepare Re-Extraction   Extraction
              |              Status
              v                |
      AI Re-Extraction          |
              |                 |
              v                 |
    Restore Retry Metadata      |
              |                 |
              +--------+--------+
                       |
                       v
              VALID / REVIEW
                 |       |
                 |       |
                 v       v
              VALID   REVIEW_REQUIRED
                 |       |
                 |       v
                 |   Review Email
                 |
                 v
          Invoice Fingerprint
                 |
                 v
          Duplicate Detection
             |           |
             |           |
        Duplicate       New
             |           |
             v           v
     Duplicate       Save Invoice
     Notification       Record
                         |
                         v
                  Valid Confirmation
</pre>

## 1. Invoice Intake

The workflow begins with a Gmail trigger that monitors incoming invoice emails.

Attachments are separated and processed individually so that an email containing multiple invoice documents can be handled as separate documents.

The workflow preserves important metadata such as:

- Original filename
- MIME type
- Sender email
- Original document data
- Retry metadata where applicable

## 2. File Type Routing

The workflow supports multiple document formats:

- PDF
- JPG
- JPEG
- PNG
- HEIC
- HEIF

PDF documents follow a text extraction path.

Image documents follow an image-based AI extraction path.

HEIC and HEIF files are converted to JPEG before AI processing while preserving the original document metadata needed by downstream processing.

## 3. AI Invoice Extraction

The workflow uses an OpenRouter API connection with an OpenAI GPT-4.1-nano model for invoice extraction.

The AI is instructed to extract structured invoice information without inventing or mathematically correcting values.

Important extraction fields include:

- Vendor
- Invoice number
- Invoice date
- Due date
- Customer
- Currency
- Payment method
- Payment reference
- Amount breakdown
- Line items
- Tax
- Payment information
- Amount due

The extraction instructions emphasize preserving values that are actually printed on the source document.

## 4. Extraction Parsing and Cleanup

The raw AI response is parsed into structured JSON.

The workflow removes unnecessary formatting such as Markdown code fences when necessary and prepares the extracted information for deterministic validation.

Large image payloads are also removed from the top-level data before downstream database processing.

## 5. Deterministic Invoice Validation

The Validate Invoice Data node performs independent validation after AI extraction.

This is an important architectural component because the AI extraction result is not automatically considered trustworthy simply because the AI returned valid JSON.

The validator checks areas including:

### Document Validation

- Vendor presence
- Invoice number presence
- Invoice date presence

### Line-Item Validation

The workflow checks whether usable line items were extracted.

It also looks for suspicious cases where multiple financial lines may have been incorrectly merged into a single line item.

### Financial Validation

The validator independently evaluates:

- Line-item totals
- Subtotals
- Discounts
- Taxes
- Additional charges
- Expected invoice total
- Payments received
- Expected amount due
- Remaining balance

### Tax Validation

When an identifiable tax rate is available, the workflow independently calculates the expected tax and compares it against the extracted tax amount.

### Total Validation

The workflow calculates an expected total from the available financial components and compares it against the extracted total.

### Payment and Amount-Due Validation

The workflow separately evaluates:

Invoice Total
-
Payment Received
=
Expected Amount Due

The extracted amount due is compared against the independently calculated balance when sufficient information is available.

## 6. Extraction Quality Check and Retry

After validation, the workflow checks for a specific extraction-quality problem:

Multiple financial lines appear to have been combined into one line item.

If this condition is detected and the invoice has not already been retried, the workflow performs a second extraction.

The retry counter prevents repeated retries.

The retry process is:

Check Extraction Quality
        |
        v
Prepare Re-Extraction
        |
        v
AI Re-Extraction
        |
        v
Restore Retry Metadata
        |
        v
Parse Image Invoice
        |
        v
Clean AI Extraction Output
        |
        v
Validate Invoice Data

The second-pass AI prompt specifically instructs the model to re-read the entire document and preserve separate financial rows instead of combining them.

## 7. Extraction Status

After validation, the workflow routes the document into one of two states.

### VALID

The invoice passed the current automated extraction and validation checks.

The document continues to duplicate detection and database processing.

### REVIEW_REQUIRED

The invoice contains one or more validation or extraction issues.

The workflow sends a review-required notification instead of continuing the invoice into normal processing.

This prevents invoices with unresolved validation issues from being treated as successfully processed.

## 8. Invoice Fingerprinting

For invoices that pass validation, the workflow creates an invoice fingerprint.

The fingerprint normalizes relevant invoice information so that equivalent invoice records can be compared consistently.

The fingerprint process considers fields such as:

- Vendor
- Invoice number
- Invoice date
- Currency
- Total amount

The fingerprint is then used for duplicate detection.

## 9. Duplicate Invoice Detection

The workflow queries the Supabase invoices table using the invoice fingerprint.

The duplicate check has two possible paths:

Duplicate Found
      |
      v
Duplicate Notification

or:

No Duplicate
      |
      v
Save Invoice Record

This prevents an invoice that has already been processed from being stored again as a new invoice through the normal processing path.

## 10. Database Storage

Validated, non-duplicate invoices are stored in the Supabase invoices table.

Stored information includes fields such as:

- Invoice fingerprint
- Vendor
- Invoice number
- Invoice date
- Customer name
- Currency
- Total amount
- Extraction status
- Source filename
- Sender email
- Processing timestamp
- Tax
- Amount breakdown

## 11. Notifications

The workflow provides several notification paths.

### Successful Processing

A confirmation email is sent when an invoice successfully passes the automated workflow.

The confirmation includes information such as:

- Vendor
- Invoice number
- Invoice date
- Invoice total
- Payment received
- Final total due
- Processing status

### Review Required

A separate email is sent when an invoice requires human review.

The review notification explains that the invoice was received but did not pass the current automated validation checks.

### Duplicate Invoice

A duplicate notification is sent when the invoice fingerprint already exists in the database.

## 12. Error Handling

The main workflow is configured with a separate n8n error-handling workflow.

The error workflow receives execution errors and sends an email notification containing information such as:

- Workflow name
- Execution ID
- Last executed node
- Error message
- Execution URL

This provides an operational alert path for unexpected workflow failures.

## 13. Design Principles

The workflow follows several important principles.

### AI Extraction Is Not the Final Validation Layer

The AI is responsible for reading and structuring the document.

Deterministic JavaScript validation is responsible for checking the extracted information.

### Printed Financial Values Should Be Preserved

The workflow is designed to preserve financial values visible on the source document rather than allowing the AI to silently correct them.

### Missing Information Should Not Be Invented

The extraction prompts instruct the AI to return null when information is genuinely unreadable or absent.

### Financial Rows Should Remain Separate

The workflow specifically detects and retries cases where multiple financial rows may have been merged during extraction.

### Duplicate Protection Occurs Before Normal Database Storage

Validated invoices are fingerprinted and checked against previously stored records before the normal save operation.

### Human Review Remains Available

Invoices that do not satisfy the automated validation requirements are routed to REVIEW_REQUIRED rather than being silently processed.

## Current Architecture Status

The workflow currently contains:

- Email-based invoice intake
- Multi-format document support
- PDF extraction
- Image extraction
- HEIC/HEIF conversion
- AI-based structured extraction
- Deterministic financial validation
- Tax validation
- Payment and amount-due reconciliation
- Extraction quality detection
- One-time AI re-extraction
- Invoice fingerprinting
- Duplicate detection
- Supabase storage
- Success notifications
- Review-required notifications
- Duplicate notifications
- Separate workflow error handling

This document describes the current exported workflow architecture and should be updated when major architectural changes are made.
