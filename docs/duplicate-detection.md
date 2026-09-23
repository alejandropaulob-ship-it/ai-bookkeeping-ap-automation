# Invoice Fingerprinting and Duplicate Detection

## Overview

The duplicate-detection layer prevents the workflow from saving the same invoice or document more than once.

The workflow uses two related duplicate-detection approaches:

1. Invoice-level fingerprinting for the normal invoice-processing path.
2. Document-level fingerprinting for HEIC/HEIF documents.

Both approaches check the `invoices` table in Supabase before allowing a new record to be saved.

The duplicate-detection layer runs after the document has passed the extraction and validation stages required for the normal invoice path.

---

## Why Duplicate Detection Is Needed

Invoices may be submitted more than once.

For example, the same invoice may be:

- Forwarded to the invoice-processing email more than once.
- Attached to multiple emails.
- Resubmitted after a processing issue.
- Uploaded again by a user.
- Received in different file instances.

Without duplicate detection, the workflow could create multiple database records for the same invoice.

The duplicate-detection system provides a checkpoint before database insertion.

---

## Invoice Fingerprint

The normal invoice-processing path creates an `invoice_fingerprint` after the invoice has passed the extraction-status stage.

The node responsible for this is:

`Create Invoice Fingerprint`

The fingerprint is built from normalized invoice information.

The current fingerprint source contains:

- Vendor
- Invoice number
- Invoice date
- Currency
- Total amount

These values are normalized before being combined.

The normalized values are joined using the pipe character:

`|`

The resulting value is stored as:

`invoice_fingerprint`

---

## Fingerprint Structure

The current invoice fingerprint follows this structure:

`vendor|invoice_number|invoice_date|currency|total_amount`

The workflow does not use the raw extracted values directly.

Instead, the `Create Invoice Fingerprint` node first normalizes the relevant fields.

This helps reduce duplicate mismatches caused by differences in formatting.

For example, invoice numbers, dates, currencies, and amounts may have different representations in source documents even when they represent the same underlying value.

---

## Field Normalization

The fingerprint creation logic uses normalization functions for the fields included in the fingerprint.

The workflow normalizes:

- `vendor`
- `invoice_number`
- `invoice_date`
- `currency`
- `total_amount`

The amount is normalized to a consistent two-decimal representation when it is a valid numeric value.

The purpose is to make the fingerprint more consistent before it is compared against existing database records.

---

## Invoice Number Normalization

The invoice number is normalized before being included in the fingerprint.

This prevents formatting differences in the extracted invoice number from unnecessarily producing different fingerprint values.

The invoice number remains a source-document identifier and is not replaced with another identifier simply to create a fingerprint.

This is important because the extraction layer separately distinguishes the main invoice or document number from SOA numbers, reference numbers, transaction numbers, and other identifiers.

---

## Date Normalization

The invoice date is normalized before being included in the fingerprint.

This provides a consistent representation for comparison when the same invoice is processed more than once.

The fingerprint therefore uses the normalized invoice date rather than relying on the exact formatting originally returned by the AI extraction step.

---

## Currency Normalization

The currency is normalized before being included in the fingerprint.

This ensures that the fingerprint uses the normalized currency value produced by the extraction and validation pipeline.

Currency remains part of the fingerprint because the same vendor, invoice number, date, and numeric amount should not automatically be treated as identical when the currency differs.

---

## Total Amount Normalization

The total amount is normalized before fingerprint creation.

Valid numeric amounts are converted to a consistent two-decimal representation.

This reduces differences caused by formatting such as:

- Different decimal precision
- Numeric formatting differences
- Other representational differences

The fingerprint uses the normalized total amount rather than the original unformatted representation.

---

## Creating the Invoice Fingerprint

The `Create Invoice Fingerprint` node combines the normalized values:

`vendor`

`invoice_number`

`invoice_date`

`currency`

`total_amount`

These values are joined into a single string and stored as:

`invoice_fingerprint`

The node preserves the rest of the invoice data while adding the fingerprint field.

---

## Checking for an Existing Invoice

After the fingerprint is created, the workflow sends the data to:

`Check Duplicate Invoice`

This node queries the Supabase `invoices` table.

The query searches for an existing record where:

`invoice_fingerprint = current invoice_fingerprint`

The Supabase node is configured with `alwaysOutputData` so that the workflow can continue into the duplicate decision even when the search does not find an existing record.

---

## Duplicate Decision

The next node is:

`Duplicate Check`

This is an IF node.

It evaluates whether the Supabase search returned an existing record.

The current condition checks whether the returned JSON contains keys:

`Object.keys($json).length > 0`

This creates two possible paths.

### Existing Record Found

If an existing record is returned, the workflow treats the invoice as a duplicate.

The workflow sends the document to:

`Send Duplicate Notification`

The existing record is not replaced by the new submission.

### No Existing Record Found

If no existing record is found, the workflow continues to:

`Save Invoice Record`

The invoice can then be stored as a new record.

---

## Duplicate Notification

When an invoice-level duplicate is detected, the workflow sends a duplicate notification instead of saving another invoice record.

The purpose of this notification is to make the duplicate condition visible to the appropriate recipient.

The duplicate path therefore stops the normal database-insertion process for that submission.

The existing invoice record remains the stored record.

---

## Database Protection

The duplicate check occurs before:

`Save Invoice Record`

This ordering is important.

The workflow does not first create a database record and then attempt to determine whether it was a duplicate.

Instead, it follows:

`Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

→ either duplicate notification or database storage

This makes duplicate detection a gate before normal invoice insertion.

---

## Relationship With Validation

Duplicate detection occurs after the invoice has passed the extraction-status decision.

The normal flow is:

`Validate Invoice Data`

→ `Check Extraction Quality`

→ `Invoice Extraction Status`

The valid path continues to:

`Create Invoice Fingerprint`

The review-required path does not continue directly into normal invoice fingerprinting and storage.

This means duplicate checking is part of the downstream processing path for invoices that have reached the normal storage stage.

---

## Normal Invoice Flow

The complete normal invoice duplicate-detection flow is:

`Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

If duplicate:

`Send Duplicate Notification`

If not duplicate:

`Save Invoice Record`

→ `Send Valid Confirmation`

This keeps duplicate handling separate from the financial validation logic.

---

## HEIC/HEIF Document Fingerprinting

HEIC and HEIF documents use a separate document-level duplicate-detection path.

This exists because those file types have their own preparation and conversion workflow.

The HEIC/HEIF duplicate path includes:

`Prepare Document for Fingerprint`

→ `Build Document Fingerprint Source`

→ `Crypto`

→ `Check Duplicate Document`

→ `Duplicate Check1`

---

## Preparing the HEIC/HEIF Document

The `Prepare Document for Fingerprint` node acts as a preparation checkpoint.

It passes the document data and binary files through unchanged.

This preserves the original document information and binary file for the fingerprinting stage.

---

## Building Document Fingerprint Metadata

The `Build Document Fingerprint Source` node adds document fingerprint source metadata.

The metadata includes:

- Original file name
- Original MIME type
- Original file format or extension

The resulting object is stored as:

`document_fingerprint_source`

This provides contextual information about the original HEIC/HEIF document.

The binary file itself remains available for the following fingerprinting step.

---

## Binary Document Fingerprint

The `Crypto` node creates the HEIC/HEIF document fingerprint.

The node is configured to:

- Use binary data
- Read the `original_file` binary property
- Store the resulting fingerprint as `document_fingerprint`

Unlike the normal invoice fingerprint, this path is based on the original document binary.

The purpose is to identify the same underlying file even when the document has gone through the HEIC/HEIF processing path.

---

## Checking the Document Fingerprint

After the `Crypto` node creates:

`document_fingerprint`

the workflow sends the result to:

`Check Duplicate Document`

This node queries the same Supabase `invoices` table.

The lookup searches for an existing record where:

`document_fingerprint = current document_fingerprint`

The result is then passed to:

`Duplicate Check1`

---

## HEIC/HEIF Duplicate Decision

`Duplicate Check1` uses the same basic empty-result logic as the normal duplicate check.

If an existing record is returned, the document is treated as a duplicate.

The workflow sends it to:

`Send HEIC/HEIF Duplicate Notification`

If no existing record is found, the workflow continues through the HEIC/HEIF database-preparation path.

---

## HEIC/HEIF Duplicate Notification

When a duplicate HEIC/HEIF document is detected, the workflow sends a dedicated duplicate notification.

The notification includes information such as:

- Original file
- Existing record ID
- Current status
- Original sender
- Previously processed timestamp
- Document fingerprint

The notification also indicates that the existing document record was preserved and that no new record was created.

---

## Preparing a New HEIC/HEIF Record

When the HEIC/HEIF document is not a duplicate, the workflow continues to:

`Merge3`

then:

`Prepare HEIC/HEIF for Database`

The preparation node restores the generated:

`document_fingerprint`

into the document data before the record is saved.

The workflow then sends the document to:

`Save Invoice Record1`

---

## Two Duplicate-Detection Strategies

The workflow therefore uses two different fingerprint concepts.

### Invoice Fingerprint

The normal invoice fingerprint is a normalized composite value based on:

- Vendor
- Invoice number
- Invoice date
- Currency
- Total amount

It is stored as:

`invoice_fingerprint`

### Document Fingerprint

The HEIC/HEIF document fingerprint is generated from the original binary file.

It is stored as:

`document_fingerprint`

These fingerprints serve different purposes and should not be treated as the same type of identifier.

---

## Why Both Approaches Exist

The normal invoice path has already converted the document into structured invoice information.

At that stage, the workflow can compare meaningful invoice attributes.

The HEIC/HEIF path also preserves the original binary document, allowing the workflow to create a document-level fingerprint before the file is processed through the database path.

This provides duplicate protection appropriate to each processing path.

---

## Relationship With Database Storage

The fingerprint fields are carried into the database record.

The normal `Save Invoice Record` node stores:

`invoice_fingerprint`

along with invoice information such as:

- Vendor
- Invoice number
- Invoice date
- Customer name
- Currency
- Total amount
- Status
- Source file
- Sender email
- Processing timestamp
- Tax
- Amount breakdown

The HEIC/HEIF database path also preserves the document fingerprint.

This allows future submissions to be compared against previously stored records.

---

## Duplicate Detection Does Not Modify the Existing Record

When a duplicate is detected, the workflow does not overwrite the existing invoice record.

Instead, the duplicate submission is routed to a notification path.

The existing record remains preserved in the database.

This prevents a repeated submission from unintentionally replacing previously processed information.

---

## Duplicate Detection vs. Financial Validation

Duplicate detection and financial validation solve different problems.

### Financial Validation

Determines whether the extracted financial information mathematically reconciles.

Examples include:

- Tax mismatches
- Total mismatches
- Amount-due mismatches
- Other financial discrepancies

### Duplicate Detection

Determines whether the invoice or document has already been stored.

Examples include:

- Same normalized invoice fingerprint
- Same original HEIC/HEIF document fingerprint

A document can therefore be financially valid and still be rejected as a duplicate.

Likewise, a unique document can contain financial discrepancies and still require review.

These are separate workflow decisions.

---

## Duplicate Detection vs. Extraction Quality

Extraction quality is evaluated before the normal invoice fingerprinting stage.

For example, if the validator detects:

`Multiple financial lines`

the workflow can perform its one allowed AI re-extraction before continuing.

The duplicate-detection stage therefore operates on the result of the completed extraction and validation pipeline rather than attempting to determine duplicates from incomplete AI output.

---

## Design Principles

The duplicate-detection architecture follows several principles.

### 1. Check before saving

The workflow checks for an existing record before creating a new invoice record.

### 2. Use normalized invoice data

The normal invoice fingerprint uses normalized fields to reduce formatting-related differences.

### 3. Preserve the original document for HEIC/HEIF fingerprinting

The HEIC/HEIF path retains the original binary document for document-level fingerprinting.

### 4. Keep fingerprint types separate

`invoice_fingerprint` and `document_fingerprint` serve different purposes and are generated differently.

### 5. Preserve existing records

A duplicate submission does not overwrite the existing database record.

### 6. Notify instead of silently discarding

Duplicate submissions are routed to dedicated notification nodes so the condition is visible.

### 7. Keep duplicate detection separate from financial validation

Duplicate detection answers whether the record already exists.

Financial validation answers whether the extracted numbers reconcile.

These decisions remain independent.

---

## Current Duplicate-Detection Architecture

The normal invoice path is:

`Invoice Extraction Status`

→ `Create Invoice Fingerprint`

→ `Check Duplicate Invoice`

→ `Duplicate Check`

→ duplicate notification

or

→ `Save Invoice Record`

→ `Send Valid Confirmation`

The HEIC/HEIF document path is:

`Prepare Document for Fingerprint`

→ `Build Document Fingerprint Source`

→ `Crypto`

→ `Check Duplicate Document`

→ `Duplicate Check1`

→ duplicate notification

or

→ `Merge3`

→ `Prepare HEIC/HEIF for Database`

→ `Save Invoice Record1`

---

## Current Architecture Status

The duplicate-detection layer provides a database-level safeguard against repeated invoice and document submissions.

Its primary responsibilities are:

- Create a normalized invoice fingerprint.
- Check the fingerprint against existing invoice records.
- Create a binary document fingerprint for HEIC/HEIF files.
- Check the document fingerprint against existing records.
- Route duplicates to notification paths.
- Prevent duplicate records from being created.
- Preserve existing database records.
- Keep duplicate detection separate from financial validation and extraction-quality decisions.

The duplicate-detection system is therefore the final protection layer between completed invoice processing and creation of a new database record.
