# Notifications and Human Review

## Overview

The notification layer communicates the outcome of invoice processing to the sender of the original invoice email.

The workflow currently has four primary notification paths:

1. Valid invoice confirmation
2. Review-required notification
3. Duplicate invoice notification
4. Duplicate HEIC/HEIF document notification

These notifications communicate different workflow outcomes without silently discarding or accepting uncertain documents.

The notifications are sent through Gmail.

---

## Notification Service

The workflow uses Gmail nodes for outbound processing notifications.

The notification nodes send messages to the email address associated with the original invoice submission.

The workflow obtains the sender address from the incoming Gmail message:

`Receive Invoice Email`

This allows the processing result to be communicated directly to the sender who submitted the invoice.

---

## Notification Outcomes

The workflow communicates four primary outcomes.

### VALID

The invoice passed the current automated validation checks and was saved as a new invoice record.

Notification:

`Send Valid Confirmation`

### REVIEW_REQUIRED

The invoice did not pass the current automated validation requirements and requires additional review before normal processing can continue.

Notification:

`Send Review Required Email`

### DUPLICATE

The invoice already exists in the database based on the invoice fingerprint.

Notification:

`Send Duplicate Notification`

### DUPLICATE HEIC/HEIF DOCUMENT

The original HEIC/HEIF document already exists in the database based on its document fingerprint.

Notification:

`Send HEIC/HEIF Duplicate Notification`

---

## Valid Invoice Confirmation

The `Send Valid Confirmation` node sends a confirmation after:

`Save Invoice Record`

The normal flow is:

`Duplicate Check`

→ `Save Invoice Record`

→ `Send Valid Confirmation`

This means the confirmation is sent after the new invoice record has been saved.

---

## Valid Confirmation Recipient

The valid confirmation is sent to the sender of the original invoice email.

The recipient is taken from:

`Receive Invoice Email`

The workflow uses the sender's email address from the incoming Gmail message.

---

## Valid Confirmation Subject

The current subject is:

`Invoice Received – Successfully Processed`

This communicates that the invoice was received and completed the automated processing path.

---

## Valid Confirmation Information

The confirmation email includes an invoice-details section containing:

- Vendor
- Invoice number
- Invoice date
- Invoice total
- Payment received
- Final total due
- Status

The status is displayed as:

`VALID`

The invoice total is formatted using the processed currency and total amount.

Payment received and final total due are taken from the validated invoice data.

---

## Valid Confirmation Meaning

The current message explains that the invoice:

- Was successfully read.
- Passed the current automated validation checks.

The confirmation therefore represents the successful completion of the current automated processing path.

It does not mean that the system performed accounting work beyond the validations implemented in the workflow.

---

## Review Required Notification

The `Send Review Required Email` node handles invoices that do not pass the final extraction and validation decision.

The `Invoice Extraction Status` node has a separate review-required output that connects to:

`Send Review Required Email`

The flow is:

`Invoice Extraction Status`

→ `Send Review Required Email`

The invoice does not continue through the normal fingerprint and database-save path from this branch.

---

## Review Required Recipient

The review-required email is sent to the sender of the original invoice email.

The workflow obtains the sender address from:

`Receive Invoice Email`

This means the person or system that submitted the invoice receives the processing result.

---

## Review Required Subject

The current subject is:

`Invoice Received – Review Required`

This clearly distinguishes the message from the successful-processing notification.

---

## Review Required Invoice Details

The review-required email includes:

- Vendor
- Invoice number
- Invoice date
- Customer
- Printed amount due
- File
- Status

When a value is unavailable, the message displays:

`Not available`

This prevents missing extraction values from producing an empty-looking notification.

---

## Printed Amount Due

The review-required email specifically displays:

`Printed Amount Due`

The value is taken from:

`financial_validation.extracted_amount_due`

This is important because the workflow deliberately preserves the amount due printed on the invoice separately from the calculated expected amount due.

The notification therefore allows the recipient to see the source-document amount that was extracted.

---

## Reason for Review

The review-required email includes a:

`Reason for Review`

section.

It displays:

- Calculated amount due
- Extracted amount due from the invoice
- Extraction issues

The calculated amount comes from:

`financial_validation.calculated_expected_amount_due`

The extracted amount comes from:

`financial_validation.extracted_amount_due`

Any entries in:

`extraction_issues`

are also added to the notification.

---

## Why the Two Amount-Due Values Are Shown

Showing both values is intentional.

The workflow separates:

### Extracted Amount Due

The amount that was read from the invoice.

### Calculated Expected Amount Due

The amount independently calculated by the deterministic validation layer.

If these values differ, the notification provides both values so the recipient can see the discrepancy rather than having the system silently replace the printed amount.

---

## Review Instructions

The current review-required email asks the recipient to:

- Review the invoice.
- Provide a corrected document or missing information if necessary.

The message also states:

`No final processing decision has been made yet.`

This communicates that the workflow has stopped short of treating the invoice as successfully processed.

---

## What Review Required Does Not Do

The current review-required path does not automatically save the invoice through:

`Save Invoice Record`

The normal database-save path begins only from the valid branch of:

`Invoice Extraction Status`

The review-required branch instead sends the notification.

This keeps unresolved invoice conditions outside the normal completed-processing path.

---

## Sources of Review Conditions

The review-required outcome can result from problems detected by the deterministic validation and extraction pipeline.

Examples include:

- Missing required invoice information
- Financial calculation discrepancies
- Tax mismatches
- Total mismatches
- Amount-due mismatches
- Extraction issues
- Other validation problems recorded in the workflow

The exact reasons are represented in the workflow's validation output.

---

## Extraction Quality and Review

The workflow performs an extraction-quality check before the final extraction-status decision.

One specific extraction-quality issue is:

`Multiple financial lines`

When this issue is detected and a retry is still available, the workflow performs its second-pass AI extraction.

The retry flow is:

`Check Extraction Quality`

→ `Prepare Re-Extraction`

→ `AI Re-Extraction`

→ `Restore Retry Metadata`

→ `Parse Image Invoice`

The resulting extraction then returns to the normal validation pipeline.

---

## Review After Retry

The retry does not bypass validation.

After the second-pass extraction, the document again goes through:

`Parse Image Invoice`

→ `Clean AI Extraction Output`

→ `Validate Invoice Data`

→ `Check Extraction Quality`

This means the second AI attempt must still satisfy the same downstream validation process.

If the resulting data still fails the final validation decision, the workflow can reach:

`REVIEW_REQUIRED`

and send the review notification.

---

## Duplicate Invoice Notification

The `Send Duplicate Notification` node handles invoices that already exist in the database.

The normal duplicate path is:

`Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

If an existing record is found:

`Send Duplicate Notification`

The invoice is not saved as a new invoice record.

---

## Duplicate Invoice Recipient

The duplicate invoice notification is sent to the sender of the original invoice email.

This allows the submitter to know that the invoice was already found in the system.

---

## Duplicate Invoice Subject

The current subject is:

`⚠️ Duplicate Invoice Detected — [invoice number]`

The invoice number is dynamically included in the subject.

---

## Duplicate Invoice Information

The duplicate invoice email includes:

- Vendor
- Invoice number
- Invoice date
- Customer
- Total amount
- Current status
- Original file
- Existing record ID

The message explains that the invoice was already found in the system and was not processed as a new invoice.

---

## Existing Record Preservation

The duplicate invoice notification explicitly states that:

- The existing invoice record was preserved.
- No new invoice record was created.

This reflects the actual workflow behavior.

The duplicate branch ends with the notification instead of continuing to:

`Save Invoice Record`

---

## HEIC/HEIF Duplicate Notification

HEIC and HEIF documents use a separate duplicate-notification node:

`Send HEIC/HEIF Duplicate Notification`

This node is reached from:

`Duplicate Check1`

when an existing document fingerprint is found.

The flow is:

`Check Duplicate Document`

→ `Duplicate Check1`

→ `Send HEIC/HEIF Duplicate Notification`

---

## HEIC/HEIF Duplicate Recipient

The HEIC/HEIF duplicate notification is sent to the sender of the original invoice email.

The recipient is obtained from:

`Receive Invoice Email`

---

## HEIC/HEIF Duplicate Subject

The current subject is:

`⚠️ Duplicate HEIC/HEIF Document Detected — [source file]`

The source filename is dynamically included in the subject.

---

## HEIC/HEIF Duplicate Information

The notification includes:

- Original file
- Existing record ID
- Current status
- Original sender
- Previously processed timestamp
- Document fingerprint

The message explains that the document was already found in the system and was not saved as a new document.

---

## HEIC/HEIF Existing Record Preservation

The HEIC/HEIF duplicate notification states that:

- The existing document record was preserved.
- No new record was created.

The duplicate branch therefore prevents the HEIC/HEIF document from reaching:

`Save Invoice Record1`

---

## Notification Routing

The notification layer is connected to specific workflow decisions.

### Valid

`Save Invoice Record`

→ `Send Valid Confirmation`

### Review Required

`Invoice Extraction Status`

→ `Send Review Required Email`

### Duplicate Invoice

`Duplicate Check`

→ `Send Duplicate Notification`

### Duplicate HEIC/HEIF Document

`Duplicate Check1`

→ `Send HEIC/HEIF Duplicate Notification`

Each notification therefore corresponds to a distinct processing outcome.

---

## Human-in-the-Loop Design

The current workflow includes a human-review mechanism through the review-required notification.

When the automated validation process cannot establish a valid result, the workflow does not simply treat the invoice as successfully processed.

Instead, it:

1. Stops the normal valid-processing path.
2. Communicates the review condition.
3. Provides the extracted invoice information.
4. Shows relevant financial comparison values.
5. Lists available extraction issues.
6. Requests review or corrected information.

This creates a human checkpoint when the automated system cannot establish the required result.

---

## Why Human Review Matters

The workflow is designed to avoid silently accepting uncertain financial information.

AI extraction can encounter documents with:

- Poor image quality
- Ambiguous digits
- Complex financial tables
- Multiple financial rows
- Missing information
- Conflicting printed values

The deterministic validation layer provides an additional check.

When the resulting data does not satisfy the current validation requirements, the notification layer communicates that condition rather than treating the invoice as successfully completed.

---

## Notification Layer and Source Evidence

The notifications preserve important source-document evidence.

For review-required invoices, the email can show:

- Printed amount due
- Calculated expected amount due
- Extraction issues
- Invoice identifiers
- Source filename

This allows the recipient to understand why the invoice did not continue through the normal processing path.

---

## Notification Layer and Database Storage

The notification layer is positioned around database storage decisions.

### Valid Path

The invoice is saved first:

`Save Invoice Record`

then the sender receives:

`Send Valid Confirmation`

### Duplicate Path

The database is checked first.

If the invoice already exists:

`Send Duplicate Notification`

No new record is saved.

### Review Path

The invoice does not proceed to the normal save path.

Instead:

`Send Review Required Email`

This separation ensures that the notification accurately reflects the database-processing outcome.

---

## Notification Layer and Duplicate Detection

Duplicate notifications are generated from the result of database duplicate checks.

The normal invoice path checks:

`invoice_fingerprint`

The HEIC/HEIF path checks:

`document_fingerprint`

Only when a matching existing record is found does the corresponding duplicate notification execute.

---

## Notification Layer and Validation

The review-required notification is downstream from the validation and extraction-quality logic.

The workflow therefore does not send a review notification simply because an AI extraction occurred.

The notification represents a result of the workflow's current validation decision.

---

## Notification Layer and AI Extraction

The AI extraction layer is responsible for reading the document.

The notification layer does not perform additional AI extraction or financial calculations.

Instead, it communicates the results already produced by:

- AI extraction
- Parsing
- Cleanup
- Deterministic validation
- Extraction-quality checks
- Duplicate detection

This keeps notification logic separate from document interpretation and financial validation.

---

## Notification Design Principles

The notification architecture follows several principles.

### 1. Communicate the processing result

Each notification corresponds to a specific workflow outcome.

### 2. Do not silently discard uncertain invoices

Review-required documents receive an explicit notification.

### 3. Preserve source values

The review notification exposes the printed amount due separately from the calculated expected amount due.

### 4. Prevent duplicate records

Duplicate notifications occur instead of creating another database record.

### 5. Preserve existing records

Duplicate notifications explicitly communicate that the existing record was preserved.

### 6. Send results to the original sender

The current notification nodes use the sender address from the original invoice email.

### 7. Keep notification logic separate

Gmail notification nodes communicate results but do not replace extraction, validation, duplicate detection, or database storage logic.

---

## Current Notification Architecture

The valid notification flow is:

`Save Invoice Record`

→ `Send Valid Confirmation`

The review-required flow is:

`Invoice Extraction Status`

→ `Send Review Required Email`

The normal duplicate flow is:

`Duplicate Check`

→ `Send Duplicate Notification`

The HEIC/HEIF duplicate flow is:

`Duplicate Check1`

→ `Send HEIC/HEIF Duplicate Notification`

---

## Current Architecture Status

The notification layer provides the communication and human-review boundary for the invoice-processing workflow.

Its primary responsibilities are:

- Confirm successfully processed invoices.
- Communicate review-required conditions.
- Show relevant validation information.
- Notify senders when an invoice is already stored.
- Notify senders when an HEIC/HEIF document is already stored.
- Preserve visibility into duplicate conditions.
- Keep uncertain invoices outside the normal completed-processing path.
- Communicate workflow results without replacing the underlying extraction, validation, duplicate-detection, or database logic.

The current system therefore combines automated processing with explicit notifications when the workflow reaches a valid, review-required, or duplicate outcome.
