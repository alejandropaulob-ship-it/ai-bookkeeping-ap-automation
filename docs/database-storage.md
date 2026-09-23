# Database Storage and Record Management

## Overview

The database layer stores processed invoice information after the workflow has completed the required extraction, validation, quality checks, and duplicate detection steps.

The workflow uses Supabase as the database service.

The primary database table used by the workflow is:

`invoices`

The workflow has two database-save paths:

1. The normal invoice-processing path.
2. The HEIC/HEIF document-processing path.

Both paths write to the `invoices` table, but they store different sets of information based on the processing path.

---

## Database Service

The workflow uses Supabase nodes for database operations.

Supabase is used for:

- Checking existing invoice records.
- Checking existing document fingerprints.
- Saving new invoice records.
- Saving HEIC/HEIF document records.

The database therefore acts as both:

- A persistent storage layer.
- A duplicate-detection source.

---

## The `invoices` Table

The workflow stores processed records in:

`invoices`

The table is used by both the normal invoice path and the HEIC/HEIF document path.

The normal invoice path stores structured invoice information and processing metadata.

The HEIC/HEIF path stores document-level information needed to preserve the original file identity and processing history.

---

## Normal Invoice Database Path

The normal invoice record is saved by:

`Save Invoice Record`

This node is reached only after the invoice has passed through the downstream extraction and validation flow and has not been identified as a duplicate.

The normal path is:

`Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

→ `Save Invoice Record`

The database save is therefore downstream of duplicate detection.

---

## Fields Stored by the Normal Invoice Path

The `Save Invoice Record` node currently stores the following fields.

### Invoice Fingerprint

`invoice_fingerprint`

This contains the normalized invoice fingerprint created by:

`Create Invoice Fingerprint`

It allows future submissions to be compared against the stored invoice.

---

### Vendor

`vendor`

This stores the extracted vendor or issuer of the invoice.

The value comes from the invoice-processing data produced by the extraction and validation pipeline.

---

### Invoice Number

`invoice_number`

This stores the extracted primary invoice or document number.

The workflow's extraction rules distinguish this from other identifiers such as SOA numbers, reference numbers, and transaction numbers.

---

### Invoice Date

`invoice_date`

This stores the extracted invoice date.

The value comes from the processed invoice data after extraction and validation.

---

### Customer Name

`customer_name`

This stores the person, company, organization, or other entity being billed.

This remains separate from the vendor field.

---

### Currency

`currency`

This stores the currency associated with the invoice.

The value comes from the processed invoice data.

---

### Total Amount

`total_amount`

This stores the invoice total value produced by the validation pipeline.

The database therefore receives the downstream workflow result rather than relying on an unvalidated AI response.

---

### Status

`status`

The normal invoice path stores the workflow's:

`extraction_status`

This allows the stored record to retain the status determined by the extraction and validation process.

---

### Source File

`source_file`

This stores the original source filename associated with the processed invoice.

This provides a reference back to the document that produced the database record.

---

### Sender Email

`sender_email`

This stores the email address associated with the sender of the invoice-processing email.

The value is taken from the incoming Gmail message.

This provides additional traceability for where the invoice submission originated.

---

### Processed Timestamp

`processed_at`

This stores the time at which the invoice was processed.

The normal invoice path uses the workflow's current timestamp when creating the database record.

This provides a processing-history reference for the stored invoice.

---

### Tax

`tax`

This stores the tax value produced by the invoice extraction and validation pipeline.

The tax remains available as a dedicated database field in addition to the broader financial breakdown.

---

### Amount Breakdown

`amount_breakdown`

This stores the invoice's structured financial breakdown.

Depending on the document, the breakdown can contain entries such as:

- Products
- Services
- Subtotal
- Discount
- Tax
- Fees
- Charges
- Total
- Payment-related financial entries

The database therefore preserves more financial detail than the single `total_amount` field.

---

## Source of Normal Invoice Database Values

The normal database record is populated from:

`Create Invoice Fingerprint`

The node carries forward the processed invoice information and adds the fingerprint.

The `Save Invoice Record` node then references those processed values when creating the Supabase record.

This keeps the database save stage downstream from the extraction and validation process.

---

## Database Save After Duplicate Detection

The normal invoice record is not saved immediately after extraction.

The workflow first performs duplicate detection.

The sequence is:

`Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

If an existing record is found:

`Send Duplicate Notification`

If no existing record is found:

`Save Invoice Record`

This prevents the same normalized invoice fingerprint from being inserted again through the normal processing path.

---

## Confirmation After Database Storage

After the normal invoice record is saved, the workflow continues to:

`Send Valid Confirmation`

The confirmation email uses information from the processed invoice.

The workflow therefore follows this sequence:

`Save Invoice Record`

→ `Send Valid Confirmation`

The database save is completed before the normal success notification is sent.

---

## HEIC/HEIF Database Path

HEIC and HEIF files use a separate database-processing path.

These documents first preserve their original metadata and binary file information before being converted for AI processing.

The original document information is retained so that the workflow can create a document-level fingerprint.

The duplicate-detection flow then determines whether the original document has already been stored.

---

## HEIC/HEIF Database Flow

The HEIC/HEIF path reaches the database through:

`Prepare Document for Fingerprint`

→ `Build Document Fingerprint Source`

→ `Crypto`

→ `Check Duplicate Document`

→ `Duplicate Check1`

If a duplicate is found:

`Send HEIC/HEIF Duplicate Notification`

If no duplicate is found:

`Merge3`

→ `Prepare HEIC/HEIF for Database`

→ `Save Invoice Record1`

---

## HEIC/HEIF Document Fingerprint

The HEIC/HEIF path generates:

`document_fingerprint`

using the original document binary.

The `Crypto` node is configured to use:

`original_file`

as its binary input.

The resulting fingerprint is stored as:

`document_fingerprint`

This allows the database to retain an identity for the original HEIC/HEIF document.

---

## HEIC/HEIF Database Preparation

After a HEIC/HEIF document passes the duplicate check, the workflow reaches:

`Prepare HEIC/HEIF for Database`

This node restores the generated document fingerprint into the data that will be saved.

The node carries forward the document data and adds:

`document_fingerprint`

before sending the record to the database.

---

## HEIC/HEIF Fields Stored

The `Save Invoice Record1` node currently stores the following fields for this path.

### Invoice Fingerprint Field

`invoice_fingerprint`

For the HEIC/HEIF save path, this field receives the generated:

`document_fingerprint`

This allows the existing database field to carry the document fingerprint for this processing path.

---

### Status

`status`

The HEIC/HEIF database path currently stores:

`manual_review`

This reflects the current behavior of this separate document-processing path.

The database record therefore identifies the document as requiring manual review rather than representing it as a fully processed normal invoice record.

---

### Source File

`source_file`

This stores:

`original_file_name`

The original HEIC/HEIF filename is preserved rather than using only the converted JPEG filename.

---

### Sender Email

`sender_email`

This stores the sender email address from the original incoming Gmail message.

This maintains traceability to the source submission.

---

### Document Fingerprint

`document_fingerprint`

This stores the fingerprint generated from the original document binary.

It provides the value used for document-level duplicate detection.

---

### Processed Timestamp

`processed_at`

This stores the current workflow timestamp when the HEIC/HEIF record is saved.

The current HEIC/HEIF database path uses the workflow timestamp in ISO format.

---

## Why the HEIC/HEIF Record Is Different

The HEIC/HEIF path has a different database record structure because the document follows a specialized processing path.

The workflow:

- Preserves the original HEIC/HEIF metadata.
- Converts the file for image processing.
- Preserves the original binary document.
- Creates a document fingerprint.
- Checks the original document against existing records.
- Saves a manual-review record when no duplicate is found.

The current database save node for this path does not populate the same full invoice field set as the normal `Save Invoice Record` node.

---

## Database Storage and Duplicate Detection

The database is directly involved in duplicate prevention.

The normal path queries:

`invoice_fingerprint`

using:

`Check Duplicate Invoice`

The HEIC/HEIF path queries:

`document_fingerprint`

using:

`Check Duplicate Document`

Both checks use the `invoices` table.

The database therefore serves as the persistent reference point for previously processed records.

---

## Database Storage and Validation

Database storage occurs after the financial validation process.

The normal path is:

`AI Extraction`

→ `Parse`

→ `Clean`

→ `Validate Invoice Data`

→ `Check Extraction Quality`

→ `Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Save Invoice Record`

The database therefore stores the downstream workflow result rather than directly storing the raw AI response.

---

## Database Storage and Review Required

The normal workflow separates invoices that require review from invoices that continue toward storage.

The `Invoice Extraction Status` node has two primary paths:

### Valid Path

The valid path continues to:

`Create Invoice Fingerprint`

and then through duplicate detection and database storage.

### Review Required Path

The review-required path continues to:

`Send Review Required Email`

instead of proceeding directly to the normal invoice database save path.

This keeps unresolved extraction or financial validation issues visible to a human rather than silently treating them as completed records.

---

## Database Storage and Duplicate Submissions

When a duplicate invoice is detected, the workflow does not proceed to:

`Save Invoice Record`

Instead, it sends:

`Send Duplicate Notification`

The existing database record remains available.

This prevents a repeated submission from creating another normal invoice record.

The same principle applies to HEIC/HEIF documents through:

`Send HEIC/HEIF Duplicate Notification`

---

## Traceability

The database records include information that helps connect a stored record to its source and processing history.

Depending on the processing path, this includes:

- Source filename
- Sender email
- Processing timestamp
- Invoice fingerprint
- Document fingerprint
- Invoice number
- Invoice date
- Vendor
- Customer
- Currency
- Status

This allows stored records to retain context about where they came from and how they were processed.

---

## Financial Detail Preservation

The normal invoice record stores both summary and detailed financial information.

The summary field is:

`total_amount`

The detailed financial structure is:

`amount_breakdown`

The record also stores:

`tax`

This allows downstream systems or users to access both the overall invoice amount and the extracted financial components.

---

## Database as the Persistence Layer

The workflow itself performs document processing during an execution.

Supabase provides persistence after the execution completes.

The database therefore allows the workflow to retain:

- Processed invoice records
- Duplicate fingerprints
- Source information
- Processing timestamps
- Status information
- Financial information

Without persistent storage, duplicate detection would not be able to compare a new submission against previously processed records.

---

## Database Storage Does Not Replace Validation

The database does not perform the invoice's financial calculations.

The deterministic validation layer performs those calculations before the normal invoice reaches database storage.

The database stores the resulting workflow data.

This keeps responsibilities separated:

### AI Extraction

Reads the source document.

### Deterministic Validation

Checks financial relationships and extraction quality.

### Duplicate Detection

Checks whether the invoice or document already exists.

### Database Storage

Persists the processed record.

---

## Database Storage Does Not Replace Duplicate Detection

Saving a record and detecting duplicates are separate operations.

The workflow first queries the database for an existing matching fingerprint.

Only when the duplicate check does not find an existing record does the normal path proceed to:

`Save Invoice Record`

This ordering is important because it prevents duplicate records from being created through the normal invoice path.

---

## Normal Database Architecture

The normal database architecture is:

`Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

If duplicate:

`Send Duplicate Notification`

If new:

`Save Invoice Record`

→ `Send Valid Confirmation`

The database save therefore occurs only after the invoice has passed the preceding workflow gates.

---

## HEIC/HEIF Database Architecture

The HEIC/HEIF database architecture is:

`Prepare Document for Fingerprint`

→ `Build Document Fingerprint Source`

→ `Crypto`

→ `Check Duplicate Document`

→ `Duplicate Check1`

If duplicate:

`Send HEIC/HEIF Duplicate Notification`

If new:

`Merge3`

→ `Prepare HEIC/HEIF for Database`

→ `Save Invoice Record1`

The HEIC/HEIF record is currently stored with:

`manual_review`

status.

---

## Design Principles

The database architecture follows several principles.

### 1. Store after processing

The normal invoice record is saved after extraction, validation, extraction-quality checks, and duplicate detection.

### 2. Use persistent storage

Supabase provides the persistent database layer required for record retention and future duplicate checks.

### 3. Preserve traceability

Source filename, sender email, and processing timestamp are stored to provide processing context.

### 4. Preserve financial detail

The normal invoice record stores both summary financial information and the detailed amount breakdown.

### 5. Separate database responsibilities

The database stores workflow results but does not replace AI extraction, deterministic validation, or duplicate detection logic.

### 6. Prevent duplicate insertion

Duplicate checks occur before the normal invoice database save.

### 7. Preserve specialized document handling

HEIC/HEIF documents retain their original document fingerprint and are stored through their dedicated database path.

---

## Current Database Architecture

The normal invoice database path is:

`Validate Invoice Data`

→ `Check Extraction Quality`

→ `Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

→ `Save Invoice Record`

→ `Send Valid Confirmation`

The HEIC/HEIF database path is:

`Prepare Document for Fingerprint`

→ `Build Document Fingerprint Source`

→ `Crypto`

→ `Check Duplicate Document`

→ `Duplicate Check1`

→ `Prepare HEIC/HEIF for Database`

→ `Save Invoice Record1`

Both paths use the Supabase:

`invoices`

table.

---

## Current Architecture Status

The database layer provides persistent storage for processed invoice and document records.

Its primary responsibilities are:

- Store processed invoice information.
- Store invoice fingerprints.
- Store HEIC/HEIF document fingerprints.
- Preserve source and sender information.
- Preserve processing timestamps.
- Store invoice financial information.
- Support duplicate detection.
- Preserve existing records when duplicates are submitted.
- Maintain specialized storage behavior for HEIC/HEIF documents.

The database therefore serves as the persistent record layer connecting the document-processing workflow with future duplicate checks and stored invoice information.
