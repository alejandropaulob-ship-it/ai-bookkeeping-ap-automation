# AI Extraction and Re-Extraction

## Overview

The AI extraction layer is responsible for reading invoice and billing documents and converting their visible information into structured JSON.

The workflow uses two primary extraction paths:

1. Image-based AI extraction for JPG, JPEG, PNG, and converted HEIC/HEIF documents.
2. PDF text-based AI extraction for PDF documents.

The AI layer is responsible for extraction, not accounting correction.

Printed financial values are preserved as extracted whenever they are clearly visible. Mathematical validation and reconciliation are performed later by the deterministic validation layer.

---

## AI Model

The workflow uses OpenRouter as the AI API endpoint:

`https://openrouter.ai/api/v1/chat/completions`

The configured model is:

`openai/gpt-4.1-nano`

The extraction requests use:

`temperature: 0`

This keeps the extraction behavior deterministic and reduces unnecessary variation between requests.

The current token limits are:

- Image extraction: 800 tokens
- PDF extraction: 1000 tokens
- Second-pass image re-extraction: 1200 tokens

---

## Image-Based Invoice Extraction

The `AI Invoice Extraction` node processes invoice images after the document has been prepared for AI processing.

The workflow provides the invoice image to the model using an image URL containing the prepared image data.

The model is instructed to return only the requested JSON structure without:

- Markdown
- Explanations
- Comments
- Code fences
- Additional text

The image itself is treated as the source of truth for extraction.

---

## Strict Source Visibility Rules

The image extraction prompt contains strict rules intended to prevent the AI from inventing or reconstructing information.

The AI must:

- Extract only information that is actually visible and readable.
- Never guess missing information.
- Never infer values from business context.
- Never reconstruct incomplete values.
- Never complete partially visible dates.
- Never complete partially visible invoice numbers or other identifiers.
- Return `null` when a value cannot be reliably determined.
- Avoid using filenames or surrounding fields to fill missing information.
- Avoid using arithmetic to reconstruct missing or unreadable values.

A partially visible value is not treated as a valid complete value.

For example, if an identifier is visibly truncated, the workflow instructs the AI to return `null` rather than returning the visible portion or attempting to reconstruct the missing characters.

---

## Complete-Value Requirement

The extraction prompt requires the AI to verify that a returned value represents the complete value visible in the document.

This applies particularly to:

- Dates
- Invoice numbers
- Reference numbers
- Amounts
- Tax rates
- Customer names
- Vendor names
- Other identifiers

If the complete value cannot be confidently determined from the document, the expected extraction result is `null`.

This prevents downstream processing from treating partial or reconstructed values as authoritative source data.

---

## Vendor and Customer Identification

The workflow distinguishes between the document issuer and the entity being billed.

### Vendor

`vendor` represents the business, company, contractor, seller, or service provider that issued the document.

The AI is instructed to inspect:

- The document header
- Business name
- Logo text
- Business information
- Issuer information

The vendor should not be confused with the customer.

### Customer

`customer_name` represents the person, company, organization, or other entity being billed.

The workflow therefore keeps the issuer and billed party as separate fields.

---

## Financial Extraction Philosophy

The AI extraction layer does not correct invoice mathematics.

When a financial value is clearly printed on the document, the AI is instructed to preserve that printed value.

This applies to:

- Subtotal
- Discount
- Subtotal after discount
- Tax
- Tax rate
- Fees
- Surcharges
- Total
- Payment received
- Amount due

If the printed values do not mathematically reconcile, the extraction layer preserves the printed information.

The downstream validation logic is responsible for determining whether the extracted financial values reconcile.

---

## Total Amount and Amount Due

The workflow distinguishes between the full invoice total and the remaining amount owed.

### `total_amount`

`total_amount` represents the full invoice total before subtracting payments already received.

If the document clearly prints a full invoice total, the AI should extract that printed amount rather than calculating a replacement value.

### `amount_due`

`amount_due` represents the remaining balance after payment when a separate remaining-balance value is available.

The extraction prompt gives special attention to labels such as:

- Amount Due After Payment
- Balance Due
- Remaining Balance
- Balance After Payment
- Amount Remaining
- Current Balance

When a clearly printed final remaining balance exists, that printed value should be extracted as `amount_due`.

The AI is specifically instructed not to substitute:

- The original invoice total
- A subtotal
- The tax amount
- The payment amount
- An unrelated identifier
- A mathematically expected amount

for the printed remaining balance.

---

## Payment Information

The extraction layer captures payment information when it is visible.

Relevant fields include:

- `payment_received`
- `payment_method`
- `payment_reference`

A payment reference should represent an actual payment, check, transaction, or reference number.

A payment date is not treated as a payment reference.

When both an invoice number and a separate payment transaction/reference number are present, the payment transaction/reference number takes priority for `payment_reference`.

Payment information is also kept separate from the original invoice total.

---

## Currency Extraction

Currency is extracted from the document.

The image extraction prompt contains a specific rule for dollar signs: when the invoice uses `$`, the workflow expects USD unless another currency is explicitly identified.

The workflow does not intentionally invent currency information when the document does not support it.

---

## Tax Extraction

Tax receives additional extraction safeguards because small visual digit differences can affect downstream financial validation.

The AI is instructed to:

- Read the printed tax amount directly from the tax line.
- Preserve the printed amount.
- Avoid calculating tax from the tax rate.
- Avoid replacing a printed tax amount with a mathematically calculated amount.
- Use the same exact printed tax amount for the `tax` field and its corresponding `amount_breakdown` entry.

The extraction prompt also requires visual verification of individual digits.

Particular attention is given to visually similar digits such as:

- 3
- 5
- 6
- 8
- 9

If a tax digit cannot be confidently distinguished, the expected result is `null` rather than a mathematically plausible replacement.

---

## Line-Item Extraction

Line items are extracted only when they are actually present on the document.

Each line item contains:

- `item_name`
- `quantity`
- `unit_cost`
- `total`

The AI is instructed not to invent missing values.

If quantity, unit cost, or total is not clearly available, that field should be `null`.

Printed line-item totals are preserved even when quantity multiplied by unit cost would produce a different result.

This allows the downstream validation layer to detect discrepancies instead of having the extraction layer silently correct them.

---

## Amount Breakdown

The `amount_breakdown` array preserves the document's visible financial structure.

Depending on the document, this may include:

- Subtotal
- Discount
- Subtotal after discount
- Tax
- Fees
- Surcharges
- Other charges
- Total
- Payment-related financial lines when applicable

A printed discount is represented as a negative amount because it is a deduction.

The extraction layer should not create financial breakdown entries that are not actually present on the document.

---

## Printed Values vs. Calculated Values

A central design principle of the workflow is the separation between extraction and validation.

The AI performs document reading.

The validation layer performs financial calculations.

For example, if an invoice visibly prints:

- Subtotal: one value
- Tax: another value
- Total: a value that does not mathematically reconcile

the AI should preserve the printed values.

The validator later calculates the expected result and determines whether the document requires review.

This prevents the AI from silently changing source-document information to make the numbers appear correct.

---

## PDF Extraction

PDF documents use a separate extraction path.

The `Extract PDF with AI` node sends extracted PDF text to the same OpenRouter endpoint using:

`openai/gpt-4.1-nano`

The current token limit is 1000 tokens.

The PDF extraction prompt follows the same core principles as image extraction:

- Extract only information present in the PDF text.
- Never guess or invent missing information.
- Return `null` when information cannot be reliably determined.
- Preserve printed financial values.
- Do not mathematically correct printed values.
- Preserve discounts, taxes, fees, surcharges, totals, and payment information.
- Extract line items from the available PDF text.
- Return only the requested JSON.

The PDF text is treated as the extraction source, while downstream workflow nodes perform validation and calculation.

---

## Parsing the AI Response

The `Parse Image Invoice` node converts the AI response into structured workflow data.

The parser:

1. Reads the AI response content.
2. Checks whether response content exists.
3. Removes Markdown code fences if the model included them.
4. Parses the response as JSON.
5. Determines whether the extraction was an initial pass or retry.
6. Removes total entries that were incorrectly placed inside `line_items`.
7. Restores relevant image/file metadata.
8. Sends the structured result to the cleanup and validation stages.

If the AI response cannot be parsed as valid JSON, the node creates a `REVIEW_REQUIRED` result and records:

`AI response could not be parsed as valid JSON`

The raw AI response and parsing error are retained for troubleshooting.

---

## Total Entries and Line Items

The parser contains an additional safeguard for cases where a model incorrectly places a total inside `line_items`.

Entries whose item name is:

- `TOTAL AMOUNT`
- `TOTAL AMOUNT DUE`
- `GRAND TOTAL`
- `TOTAL`

are removed from `line_items`.

These values belong in the financial breakdown rather than being treated as products or services.

This prevents document totals from being incorrectly interpreted as purchased items.

---

## Cleaning AI Extraction Output

The `Clean AI Extraction Output` node performs additional normalization of AI responses before deterministic validation.

The node:

- Reads the AI response content.
- Preserves file metadata.
- Handles cases where the AI response is already an object.
- Removes Markdown code fences when necessary.
- Parses the cleaned response as JSON.
- Preserves filename and MIME type information.

The node also contains a compatibility step for simple arithmetic expressions that may occasionally appear inside numeric JSON fields.

For example, an expression such as:

`53.76 + 2.24 + 56.00 + 39.00`

can be converted into its numeric result so that the resulting JSON can continue through the workflow.

This is a response-cleanup mechanism. It does not replace the downstream financial validation logic.

---

## Extraction Quality Check

After parsing and cleanup, the workflow sends the extracted data to deterministic validation.

The validator can identify suspicious extraction patterns.

One important example is:

`Multiple financial lines`

This issue indicates that the AI may have combined separate printed financial rows instead of preserving them individually.

The workflow treats this as an extraction-quality problem.

When this specific issue is detected and no retry has yet been performed, the workflow sends the document through the second-pass extraction path.

---

## Second-Pass Re-Extraction

The workflow supports one additional AI extraction attempt when the extraction-quality check identifies merged financial lines.

The sequence is:

`Check Extraction Quality`

then:

`Prepare Re-Extraction`

then:

`AI Re-Extraction`

then:

`Restore Retry Metadata`

then:

`Parse Image Invoice`

The retry is limited to one additional attempt.

This prevents the workflow from repeatedly sending the same document to the AI when the extraction problem cannot be resolved automatically.

---

## Preparing the Retry

The `Prepare Re-Extraction` node adds retry metadata to the extracted document.

It:

- Increments `retry_count`.
- Sets `retry_required` to `true`.
- Sets `retry_reason` to `Merged financial lines detected`.
- Adds a specific retry instruction.

The retry instruction tells the AI to:

- Re-read the entire document.
- Avoid combining separate printed lines.
- Preserve every visible financial line separately.
- Preserve separate `amount_breakdown` entries.
- Preserve separate `line_items` when applicable.
- Carefully distinguish the main document number from SOA numbers.
- Distinguish reference numbers and other identifiers.
- Return `null` instead of guessing when information is unclear.

---

## Second-Pass Visual Extraction

The `AI Re-Extraction` node performs a second-pass visual extraction of the entire document.

The second-pass prompt specifically instructs the model to inspect the complete document again rather than focusing only on the suspected problem.

The model is instructed to:

- Read the entire image from the header through the bottom.
- Re-check document information.
- Inspect financial rows individually.
- Search the entire document for the final printed total.
- Check summary and bottom sections.
- Preserve every separate financial row.
- Avoid placing identifiers into `line_items`.
- Avoid calculating missing amounts.
- Return only the required JSON.

The second pass therefore acts as a targeted verification step while still re-extracting the complete document.

---

## Document Number Verification During Retry

The retry prompt gives additional attention to document identifiers.

The workflow distinguishes the main document number from other identifiers such as:

- SOA numbers
- Policy numbers
- Reference numbers
- PRN numbers
- Transaction numbers

When a main document number is clearly shown, it should be used as `invoice_number`.

Other identifiers should not replace the primary document number simply because they are easier for the AI to detect.

---

## Financial Row Verification During Retry

The second-pass extraction specifically addresses documents where financial rows are arranged in tables.

The AI is instructed to read the table row by row.

Each visible financial row should have its own `amount_breakdown` entry.

The model is instructed to match:

- The printed label
- The amount on the same visual row

rather than relying only on text reading order.

This is important for documents where labels and amounts are positioned in separate columns.

---

## Amount Classification During Retry

The second-pass extraction uses a controlled set of amount types:

- `product`
- `service`
- `subtotal`
- `tax`
- `fee`
- `charge`
- `discount`
- `total`
- `payment`
- `other`

The retry prompt also provides classification guidance.

For example:

- A line should only be classified as `tax` when the document explicitly identifies it as tax or uses a clear tax designation.
- A percentage alone does not make a line a tax.
- `PREMIUM` is not automatically classified as tax.
- `DOCUMENTARY STAMPS` are normally classified as a fee, charge, or other unless explicitly identified as a tax.
- `TOTAL` and equivalent final-total labels are classified as `total` when they represent the final amount owed.
- `PAYMENT RECEIVED` is classified as `payment`.

The purpose is to preserve the document's actual financial meaning rather than inferring classifications from arithmetic or position alone.

---

## Retry Metadata Restoration

After the second-pass AI request, the `Restore Retry Metadata` node restores workflow information that must continue through the remaining pipeline.

The node restores:

- `retry_count`
- `file_name`
- `mime_type`
- `image_data_url`
- Original binary data when available

This allows the retry result to re-enter the normal parsing and validation pipeline without losing the original document context.

---

## Retry Result Flow

After metadata restoration, the retry result returns to:

`Parse Image Invoice`

From there it follows the same downstream path as the original extraction:

`Parse Image Invoice`

→ `Clean AI Extraction Output`

→ `Validate Invoice Data`

→ `Check Extraction Quality`

The workflow therefore does not create a separate validation system for retry results.

Both initial and retry extractions ultimately pass through the same deterministic validation layer.

---

## Extraction Status vs. Extraction Quality

The workflow separates two different concepts:

### Extraction Quality

This concerns whether the AI successfully represented the visible document structure.

Examples include:

- Merged financial rows
- Incorrect placement of totals
- Unparseable AI response

### Financial Validation

This concerns whether the extracted financial information mathematically reconciles.

Examples include:

- Tax mismatch
- Total mismatch
- Amount-due mismatch
- Other financial discrepancies

This separation is important because a document can have a structurally successful extraction while still containing printed financial values that do not mathematically reconcile.

Likewise, an extraction-quality issue can exist even before the financial calculations are evaluated.

---

## Design Principles

The AI extraction architecture follows several core principles.

### 1. Extract, do not correct

The AI should preserve clearly printed document values rather than silently correcting them.

### 2. Never invent missing information

Unclear or incomplete information should become `null` rather than a guessed value.

### 3. Preserve source-document evidence

Printed values remain available so downstream validation can compare them against independently calculated values.

### 4. Separate extraction from validation

AI handles document interpretation.

Deterministic code handles financial validation.

### 5. Retry targeted extraction problems

The workflow performs a second AI pass when a known extraction-quality problem is detected.

### 6. Limit retries

Only one retry is permitted for the current merged-financial-line condition.

### 7. Reuse the normal validation pipeline

Retry results return to the same parsing, cleanup, and validation stages used by the initial extraction.

---

## Current AI Extraction Architecture

The current extraction architecture is:

`Receive Invoice Email`

→ `Split Invoice Attachments`

→ `Route by File Type`

For image documents:

`Prepare Image for AI`

→ `AI Invoice Extraction`

→ `Parse Image Invoice`

→ `Clean AI Extraction Output`

→ `Validate Invoice Data`

For PDF documents:

`Extract PDF Text`

→ `Extract PDF with AI`

→ `PDF Code`

→ `Validate Invoice Data`

When merged financial lines are detected:

`Check Extraction Quality`

→ `Prepare Re-Extraction`

→ `AI Re-Extraction`

→ `Restore Retry Metadata`

→ `Parse Image Invoice`

The extracted information then proceeds through the same deterministic validation and review process.

---

## Current Architecture Status

The AI extraction layer is designed as an extraction and document-understanding component rather than an accounting decision-maker.

Its primary responsibilities are:

- Read the source document.
- Extract visible information.
- Preserve printed financial values.
- Structure document information as JSON.
- Detect and retry specific extraction-quality problems.
- Preserve document metadata.
- Pass the resulting data to deterministic validation.

Financial reconciliation, duplicate detection, storage, and final workflow routing remain downstream responsibilities.
