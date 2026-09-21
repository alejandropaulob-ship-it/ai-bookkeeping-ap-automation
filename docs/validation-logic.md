# Invoice Validation Logic

## Overview

The `Validate Invoice Data` node is the deterministic validation layer of the AI Bookkeeping & AP Automation workflow.

The AI extraction nodes are responsible for reading invoice documents and returning structured data. The validation node then independently evaluates that extracted information using JavaScript calculations and rule-based checks.

This separation allows the workflow to distinguish between:

- What the invoice actually printed
- What the AI extracted
- What the workflow independently calculates
- Whether those values reconcile
- Whether the document should continue automatically or require review

The validation result is ultimately represented by:

- `extraction_valid`
- `extraction_status`
- `extraction_issues`
- `financial_validation`
- `financial_validation_issues`

## 1. Basic Document Validation

The validator first checks whether several required document fields are present.

It checks:

- Vendor
- Invoice number
- Invoice date

If any of these fields are missing, the validator adds a corresponding issue to `extraction_issues`.

Examples include:

- Missing vendor
- Missing invoice number
- Missing invoice date

## 2. Currency Normalization

The validator normalizes the currency value before continuing with financial validation.

If the extracted currency is empty, the current validator defaults the normalized currency to:

USD

This normalized value is then included in the workflow output.

## 3. Line-Item Validation

The validator checks whether the extraction contains a usable `line_items` array.

If:

- `line_items` is missing
- `line_items` is not an array
- `line_items` contains no entries

the validator adds:

No line items extracted

to the extraction issues.

When valid line items are available, the validator independently calculates the sum of their extracted `total` values.

If every line item contains a usable numeric total, the workflow calculates:

Line Item Total
=
Sum of all line-item totals

This becomes the calculation base when available.

## 4. Amount Breakdown Processing

The validator examines the `amount_breakdown` array to identify financial summary entries.

It looks for information such as:

- Subtotal
- Subtotal after discount
- Total
- Amount due
- Payment received
- Tax
- Discounts
- Fees
- Surcharges
- Other charges

The validator uses both the entry `type` and the entry description/label when identifying these values.

This allows the workflow to work with different descriptions used by different invoice formats.

## 5. Subtotal Validation

The validator attempts to identify the printed subtotal from the amount breakdown.

Recognized subtotal descriptions include examples such as:

- SUBTOTAL
- NET SALES
- NET SALE
- NET AMOUNT
- VATABLE SALES
- VATABLE SALE
- TAXABLE SALES
- TAXABLE SALE

When both the calculated line-item total and extracted subtotal are available, they are compared.

The comparison uses a tolerance of less than $0.01.

If they do not reconcile, the validator adds an issue describing the difference between:

- Calculated line-item total
- Extracted subtotal

The result is also stored as:

`line_items_match_subtotal`

## 6. Subtotal After Discount

The validator separately identifies a printed subtotal-after-discount value when present.

It recognizes descriptions containing terms such as:

- SUBTOTAL AFTER DISCOUNT
- SUBTOTAL AFTER DEDUCTION
- NET AFTER DISCOUNT
- NET AFTER DEDUCTION

The validator calculates its own subtotal after discount:

Calculated Subtotal After Discount
=
Calculation Base Subtotal
-
Discount Amount

If a printed subtotal-after-discount value exists, the calculated value is compared against it.

The result is stored as:

`subtotal_after_discount_reconciled`

A mismatch produces a validation issue.

## 7. Discount Detection

The validator identifies discounts using both the entry type and description.

It recognizes entries such as:

- DISCOUNT
- REBATE
- PROMO
- PROMOTIONAL

A subtotal-after-discount entry is excluded from discount detection so that it is not incorrectly treated as a discount itself.

The validator can determine the discount amount from:

1. A percentage shown in the discount entry
2. The difference between the calculation base subtotal and printed subtotal after discount
3. An explicit discount amount

The validator records the source of the discount calculation using:

`discount_amount_source`

Possible sources include:

- `reconciled_percentage_and_explicit`
- `calculated_from_percentage`
- `derived_due_to_explicit_discount_mismatch`
- `derived_from_authoritative_subtotal`
- `explicit_discount_entry`

The normalized discount amount is then used in subsequent financial calculations.

## 8. Additional Charges and Fees

The validator identifies additional charges from the amount breakdown.

Recognized charge types and descriptions include items such as:

- Fee
- Charge
- Surcharge
- Processing
- Handling
- Shipping
- Administrative
- Convenience
- Disposal
- Environmental
- Service Charge
- Other Charge

The validator sums recognized charge entries into:

`additional_charges`

These charges are included in the downstream financial calculation.

## 9. Tax Validation

Tax is identified from the amount breakdown and can also fall back to the extracted `tax` field.

The validator recognizes tax entries such as:

- VAT
- Value Added Tax
- Sales Tax
- State Tax
- City Tax
- County Tax
- Local Government Tax
- GST
- HST
- PST
- QST

The validator also attempts to identify a tax rate from the tax description when a percentage is present.

### Independent Tax Calculation

When an identifiable tax rate exists, the validator independently calculates tax.

The taxable base is:

Calculation Base Subtotal
-
Discount
+
Additional Charges

Then:

Calculated Tax
=
Taxable Base
×
Tax Rate

The independently calculated tax is compared against the extracted tax amount.

If the values do not reconcile within the validator's tolerance, the workflow records a tax validation issue.

The financial validation output includes:

- `tax_rate`
- `tax_base_after_discount`
- `calculated_tax`
- `extracted_tax`
- `tax_difference`
- `tax_reconciled`

## 10. Expected Total Calculation

The validator independently calculates an expected invoice total when a calculation base is available.

The calculation is:

Calculation Base Subtotal
-
Discount
+
Additional Charges
+
Tax
=
Calculated Expected Total

When a tax rate is available, the independently calculated tax is used.

When no identifiable tax rate is available, the extracted tax amount is used for the total calculation when available.

The result is stored as:

`calculated_expected_total`

The extracted invoice total is also retained separately as:

`extracted_total`

The validator records whether the two values reconcile as:

`total_reconciled`

## 11. Payment Received

The validator identifies payment entries from the amount breakdown.

Recognized payment descriptions include:

- PAYMENT RECEIVED
- AMOUNT PAID
- PAID
- PAYMENT MADE
- PAYMENTS RECEIVED

Payment entries are summed to determine:

`payment_received`

If no payment breakdown entry exists, the validator can use the extracted `payment_received` field.

The payment amount is normalized and used in the amount-due calculation.

## 12. Amount Due Validation

The validator separately evaluates the amount due.

It first preserves the extracted amount due as:

`extracted_amount_due`

It then independently calculates the expected remaining balance:

Calculated Expected Amount Due
=
Calculated Expected Total
-
Payment Received

The result is not allowed to become negative.

The independently calculated value is stored as:

`calculated_expected_amount_due`

The validator compares the extracted amount due against the calculated expected amount due.

The difference is stored as:

`amount_due_difference`

The reconciliation result is stored as:

`amount_due_reconciled`

## 13. Printed Amount Due vs. Calculated Amount Due

An important part of the validation architecture is that the workflow keeps the extracted printed amount due available for comparison.

The validation output contains both:

- `extracted_amount_due`
- `calculated_expected_amount_due`

This allows the review notification to show the amount printed on the invoice alongside the independently calculated amount.

The validator can therefore identify cases where the document's printed financial information does not agree with the workflow's independent calculation.

## 14. Gross-Total Amount-Due Handling

The validator contains specific handling for invoices where the extracted amount due appears to equal the full invoice total even though a payment has been recorded.

The validator checks whether:

- Payment received is greater than zero
- Extracted amount due is approximately equal to the extracted total

This situation is treated as a special case in the amount-due validation logic.

The workflow preserves this handling so that this specific document pattern does not automatically create a false-positive amount-due issue.

## 15. Suspicious Merged Financial Lines

The validator also checks for a specific extraction-quality problem.

It examines the names of extracted line items for multiple financial keywords.

The current keyword checks include:

- VAT
- VALUE ADDED TAX
- LOCAL GOVT
- LOCAL GOVERNMENT
- DOCUMENTARY
- PREMIUM
- OTHERS

If at least two of these financial keywords appear in the same line-item name, the workflow adds:

Multiple financial lines appear to have been combined into one line item

to `extraction_issues`.

This identifies a possible case where the AI incorrectly merged separate financial rows into one line item.

## 16. Financial Validation Summary

The validator creates a separate:

`financial_validation_issues`

array.

This array contains issues associated with financial calculations and reconciliation, including issues involving:

- Calculated values
- Amount mismatches
- Missing total information

The complete set of extraction issues remains available separately in:

`extraction_issues`

This allows downstream workflow nodes to distinguish broader extraction problems from financial validation issues.

## 17. Final Validation Decision

The final validation decision is based on the complete `problems` array.

The validator sets:

`extraction_valid = true`

only when there are no remaining validation problems.

If there are one or more problems:

`extraction_valid = false`

The workflow then assigns:

### VALID

when:

`extraction_valid = true`

### REVIEW_REQUIRED

when:

`extraction_valid = false`

The status is stored as:

`extraction_status`

## 18. Validation Output

The validator returns the original extracted invoice data together with standardized and validation fields.

Important output values include:

### Standardized Financial Values

- `currency`
- `total_amount`
- `amount_due`
- `final_total_due`
- `payment_received`
- `tax`

### Financial Validation

- `calculated_product_service_total`
- `calculated_all_line_items_total`
- `extracted_subtotal`
- `calculation_base_subtotal`
- `calculated_subtotal_after_discount`
- `extracted_subtotal_after_discount`
- `subtotal_after_discount_reconciled`
- `line_items_match_subtotal`
- `tax_rate`
- `tax_base_after_discount`
- `tax_amount`
- `calculated_tax`
- `extracted_tax`
- `tax_difference`
- `tax_reconciled`
- `discount_amount`
- `discount_amount_source`
- `additional_charges`
- `calculated_expected_total`
- `extracted_total`
- `total_reconciled`
- `extracted_amount_due`
- `calculated_expected_amount_due`
- `amount_due_reconciled`
- `amount_due_difference`
- `payment_math_reconciled`
- `remaining_balance`

### Extraction Status

- `financial_validation_issues`
- `extraction_valid`
- `extraction_status`
- `extraction_issues`

## 19. Relationship With the Extraction Quality Retry

The validation node works together with the `Check Extraction Quality` node.

The retry condition checks whether:

`extraction_issues`

contains:

`Multiple financial lines`

and whether:

`retry_count < 1`

When both conditions are true, the workflow performs one second-pass extraction.

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

The retry instruction specifically tells the second-pass extraction to:

- Re-read the entire document
- Avoid combining separate printed lines
- Preserve visible financial lines separately
- Preserve separate line items when applicable
- Distinguish the main document number from SOA numbers and other references
- Return null instead of guessing unclear values

The retry counter prevents the workflow from repeatedly reprocessing the same extraction.

## 20. Validation and Human Review

After validation and the extraction-quality check, the workflow reaches the `Invoice Extraction Status` node.

Invoices marked:

`VALID`

continue toward:

- Invoice fingerprint creation
- Duplicate detection
- Database storage
- Valid confirmation

Invoices marked:

`REVIEW_REQUIRED`

are routed to the review-required notification instead of continuing through the normal automated storage path.

The review notification includes the printed amount due and the calculated amount due when those values are available.

This gives the reviewer visibility into the discrepancy rather than silently replacing the document's printed value.

## 21. Design Principles

The validation architecture follows several important principles.

### Separate Extraction From Validation

AI is used to read and structure the invoice.

Deterministic JavaScript is used to independently evaluate the extracted information.

### Preserve Source Values for Comparison

The workflow retains extracted financial values so that they can be compared against independently calculated values.

### Do Not Automatically Trust Reconciliations

A mathematically calculated value is treated as a validation result rather than automatically being assumed to be what the document printed.

### Detect Extraction Quality Problems

The workflow specifically looks for signs that multiple financial rows may have been incorrectly combined.

### Limit Automatic Retries

The current retry mechanism is limited to one second-pass extraction for the merged-financial-lines condition.

### Route Unresolved Issues to Review

Invoices that still contain validation problems are routed to `REVIEW_REQUIRED` instead of continuing through the normal automated processing path.

## Current Validation Architecture

The current validation layer contains:

- Required-field validation
- Currency normalization
- Line-item presence validation
- Line-item total calculation
- Subtotal extraction
- Subtotal reconciliation
- Subtotal-after-discount reconciliation
- Discount detection and calculation
- Additional charge detection
- Tax detection
- Tax-rate detection
- Independent tax calculation
- Expected invoice total calculation
- Payment detection
- Expected amount-due calculation
- Amount-due reconciliation
- Printed amount-due tracking
- Suspicious merged-financial-line detection
- Financial validation issue reporting
- VALID / REVIEW_REQUIRED status generation
- One-time extraction retry integration

This document describes the validation behavior of the currently exported workflow and should be updated whenever the `Validate Invoice Data` logic or its connected validation flow changes.
