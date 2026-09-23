# Error Handling and Workflow Recovery

## Overview

The workflow has a dedicated error-handling workflow for unexpected technical execution failures.

This error-handling layer is separate from the invoice validation and review logic.

The main invoice workflow handles document-level conditions such as:

- Missing information
- Financial discrepancies
- Extraction-quality problems
- Duplicate invoices
- Review-required results

The dedicated error handler handles unexpected workflow execution errors that cause an n8n execution to fail.

---

## Dedicated Error Handler Workflow

The error-handling workflow is named:

`Invoice Automation - Error Handler`

It is a separate n8n workflow from:

`AI Bookkeeping & AP Automation`

The dedicated workflow currently contains two nodes:

1. `Error Trigger`
2. `Send a message`

The connection is:

`Error Trigger`

→ `Send a message`

---

## Error Trigger

The first node is:

`Error Trigger`

This is an n8n Error Trigger node.

Its purpose is to receive information about failed executions from the workflow that is configured to use this error workflow.

The Error Trigger provides execution and workflow information that can then be used by the notification node.

---

## Main Workflow Error Configuration

The main:

`AI Bookkeeping & AP Automation`

workflow is configured with an error workflow:

`Invoice Automation - Error Handler`

This connects unexpected execution failures in the main workflow to the dedicated error-handling workflow.

The main workflow therefore has a separate operational failure path in addition to its normal invoice-processing branches.

---

## Error Handler Status

The current error-handler workflow is active.

This means it is configured as an active n8n workflow rather than existing only as an inactive exported workflow.

The main workflow also references the error-handler workflow through its workflow settings.

---

## Error Notification

The second node in the error-handler workflow is:

`Send a message`

This is a Gmail node.

Its purpose is to send an operational error notification when the Error Trigger receives a failed execution.

The notification is separate from the customer-facing invoice notifications in the main workflow.

---

## Error Notification Subject

The current subject is dynamically generated from the failed workflow name.

The subject follows this structure:

`🚨 Invoice Automation Error — [workflow name]`

This makes the affected workflow identifiable from the email subject.

---

## Error Notification Information

The error notification includes several pieces of execution information.

### Workflow

The failed workflow name is included using:

`workflow.name`

This identifies which workflow generated the error.

---

### Execution ID

The notification includes:

`execution.id`

This identifies the specific n8n execution that failed.

The execution ID can be used to locate the corresponding execution in n8n.

---

### Last Node Executed

The notification includes:

`execution.error.lastNodeExecuted`

This identifies the last node reported by the execution error information.

This helps narrow down where the workflow failure occurred.

---

### Error Message

The notification includes:

`execution.error.message`

This provides the error message associated with the failed execution.

The message gives the person investigating the issue an initial description of the failure.

---

### Execution URL

The notification includes the execution URL:

`execution.url`

The URL is inserted as a clickable link in the email.

This allows the recipient to open the failed execution directly in n8n for investigation.

---

## Error Notification Message

The current notification communicates that:

- An error occurred while processing an invoice.
- The workflow name is provided.
- The execution ID is provided.
- The last node executed is provided.
- The error message is provided.
- The execution URL is provided.
- The failed execution should be reviewed in n8n.

The notification therefore acts as an operational alert rather than an invoice-processing result.

---

## Error Handler Flow

The complete error-handler flow is:

`Error Trigger`

→ `Send a message`

The Error Trigger receives the failed execution information.

The Gmail node then sends the operational alert containing the available execution details.

There are currently no additional processing nodes between the error trigger and notification.

---

## Operational Errors vs. Invoice Validation

The workflow deliberately separates technical execution failures from invoice-level validation results.

### Invoice Validation

Validation determines whether the extracted invoice information satisfies the workflow's financial and document requirements.

Examples include:

- Missing required fields
- Tax discrepancies
- Total discrepancies
- Amount-due discrepancies
- Extraction-quality issues
- Other validation problems

These conditions can lead to:

`REVIEW_REQUIRED`

and the corresponding review notification.

### Technical Execution Error

A technical execution error occurs when an n8n workflow execution fails unexpectedly.

The dedicated error handler receives these failures and sends an operational error notification.

This distinction prevents a normal validation result from being treated as a system failure.

---

## Review Required Is Not the Same as a Workflow Error

A `REVIEW_REQUIRED` result is an expected business-processing outcome.

The workflow intentionally routes the invoice to:

`Send Review Required Email`

when the invoice does not satisfy the current validation requirements.

This does not necessarily mean that n8n failed.

By contrast, the Error Handler is used for unexpected execution failures that cause the workflow to fail.

The two mechanisms therefore serve different purposes.

---

## Duplicate Detection Is Not a Workflow Error

Duplicate invoices are also expected workflow outcomes.

The normal invoice path checks for an existing fingerprint.

If a duplicate is found, the workflow intentionally routes the document to:

`Send Duplicate Notification`

instead of creating a new record.

This is not an execution failure.

Similarly, duplicate HEIC/HEIF documents are routed through:

`Send HEIC/HEIF Duplicate Notification`

rather than the technical error handler.

---

## Error Handling Does Not Replace Review Logic

The dedicated error handler does not determine whether an invoice is valid.

It does not:

- Recalculate invoice totals.
- Validate tax.
- Determine amount due.
- Retry AI extraction.
- Check invoice fingerprints.
- Save invoice records.
- Send review-required notifications.

Those responsibilities remain in the main invoice-processing workflow.

The error handler is focused on unexpected workflow execution failures.

---

## Error Handling Does Not Replace the AI Retry

The main workflow has a targeted retry mechanism for a specific extraction-quality issue:

`Multiple financial lines`

That retry follows:

`Check Extraction Quality`

→ `Prepare Re-Extraction`

→ `AI Re-Extraction`

→ `Restore Retry Metadata`

→ `Parse Image Invoice`

This is an expected recovery mechanism inside the invoice-processing workflow.

It is different from the dedicated Error Handler.

The AI retry attempts to improve extraction quality.

The Error Handler reports unexpected execution failures.

---

## Error Handling and Validation Flow

The normal invoice-processing workflow can reach:

`Validate Invoice Data`

→ `Check Extraction Quality`

→ `Invoice Extraction Status`

The invoice can then be routed to:

- Valid processing
- Review required

These are normal workflow decisions.

If an unexpected technical error causes the execution itself to fail, the dedicated error workflow handles that failure separately.

---

## Error Handling and Database Storage

The error handler is separate from the normal database-storage path.

The normal invoice path eventually reaches:

`Save Invoice Record`

when the invoice passes the required workflow checks and is not a duplicate.

If an unexpected technical failure prevents normal processing from completing, the dedicated error workflow provides an operational alert.

The error handler does not attempt to create or modify the invoice database record.

---

## Error Handling and Notifications

The main workflow contains customer-facing or processing-result notifications such as:

- `Send Valid Confirmation`
- `Send Review Required Email`
- `Send Duplicate Notification`
- `Send HEIC/HEIF Duplicate Notification`

The error handler contains:

- `Send a message`

The error notification is therefore distinct from the normal invoice-result notifications.

---

## Error Information Preserved for Investigation

The error notification preserves the key information needed to investigate a failed execution.

The available information includes:

- Workflow name
- Execution ID
- Last node executed
- Error message
- Execution URL

This provides a direct path from the alert email back to the failed n8n execution.

---

## Investigating a Workflow Error

When an error notification is received, the execution URL can be opened in n8n.

The failed execution can then be reviewed to determine:

1. Which workflow failed.
2. Which execution failed.
3. Which node was last executed.
4. What error message was reported.
5. What execution data is available for troubleshooting.

The error handler therefore provides notification and investigation context rather than attempting to automatically repair every possible technical failure.

---

## Error Recovery Philosophy

The current architecture uses different recovery mechanisms depending on the type of problem.

### Extraction-Quality Problem

Use the targeted AI retry.

### Validation Problem

Route the invoice to review.

### Duplicate Invoice

Notify the sender and preserve the existing record.

### Unexpected Technical Failure

Trigger the dedicated error workflow and send an operational alert.

This keeps recovery behavior aligned with the type of failure encountered.

---

## Why a Dedicated Error Workflow Is Useful

Separating technical error handling from the main workflow provides several benefits.

### Independent Error Notification

The main workflow does not need to contain a separate error-notification branch after every node.

### Centralized Operational Alerting

Unexpected execution failures can be handled by one dedicated workflow.

### Better Troubleshooting Context

The error notification includes execution information that can be used to investigate the failure.

### Separation of Responsibilities

The main workflow focuses on invoice processing.

The error workflow focuses on execution failures.

---

## Current Error Handler Architecture

The dedicated error workflow is:

`Error Trigger`

→ `Send a message`

The main workflow is configured to use:

`Invoice Automation - Error Handler`

as its error workflow.

The error notification includes:

`workflow.name`

`execution.id`

`execution.error.lastNodeExecuted`

`execution.error.message`

`execution.url`

---

## Relationship Between All Recovery Layers

The complete workflow has multiple levels of protection.

### Level 1: Extraction Protection

The AI extraction prompts require visible, complete source values and prohibit guessing.

### Level 2: Extraction Retry

A targeted second AI pass is available for the `Multiple financial lines` extraction-quality condition.

### Level 3: Deterministic Validation

The workflow independently validates financial relationships and identifies discrepancies.

### Level 4: Review Required

Invoices that do not satisfy the current validation decision are routed to the review notification.

### Level 5: Duplicate Detection

Previously stored invoices and HEIC/HEIF documents are detected before new records are created.

### Level 6: Technical Error Handling

Unexpected n8n execution failures trigger the dedicated error workflow.

These layers address different failure modes rather than relying on one mechanism for every problem.

---

## What the Error Handler Does Not Currently Do

The current error-handler workflow is intentionally simple.

It does not currently:

- Retry failed n8n executions automatically.
- Modify the failed workflow.
- Update the invoice database.
- Attempt to reprocess the invoice.
- Perform additional AI extraction.
- Determine whether the invoice itself is valid.
- Send a review-required invoice notification.
- Resolve the underlying technical problem automatically.

Its current responsibility is to notify the appropriate recipient and provide execution information for investigation.

---

## Design Principles

The error-handling architecture follows several principles.

### 1. Separate technical failures from business decisions

A workflow execution failure is different from an invoice failing validation.

### 2. Preserve execution context

The alert includes the workflow name, execution ID, last node, error message, and execution URL.

### 3. Centralize unexpected-error handling

The dedicated error workflow provides one operational alerting mechanism for the main workflow.

### 4. Keep expected recovery inside the main workflow

Extraction retries, validation, review, and duplicate handling remain part of the normal invoice-processing architecture.

### 5. Do not silently ignore technical failures

Unexpected execution failures generate an operational notification.

### 6. Keep the error handler simple

The current error workflow focuses on alerting and investigation rather than attempting automatic remediation.

---

## Current Architecture Status

The error-handling layer provides a dedicated operational safety net for unexpected n8n execution failures.

Its primary responsibilities are:

- Receive failed execution information through `Error Trigger`.
- Identify the failed workflow.
- Identify the failed execution.
- Identify the last node executed.
- Capture the reported error message.
- Provide a direct execution URL.
- Send an operational Gmail notification.
- Separate technical failures from invoice validation outcomes.
- Provide enough information for manual troubleshooting in n8n.

The current architecture therefore combines automated invoice processing with targeted extraction recovery, deterministic validation, human review, duplicate protection, and dedicated technical error alerting.
