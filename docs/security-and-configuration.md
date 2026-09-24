# Security and Configuration

## Overview

AI Bookkeeping & AP Automation separates workflow logic from private operational configuration.

The public GitHub repository contains workflow structure, documentation, and sanitized configuration examples. Actual credentials and private operational values remain in the n8n instance.

This separation allows the workflow to be version-controlled and shared without exposing credentials or private configuration.

---

## Credential Management

The workflow uses n8n-managed credentials for external services.

### Gmail

Gmail credentials are configured through n8n's credential system.

The workflow exports may contain the credential reference and credential name, but the actual OAuth credentials are not stored in the workflow source.

### OpenRouter

OpenRouter is used for AI-based invoice extraction and re-extraction.

The workflow references the OpenRouter API endpoint and model configuration, while authentication is handled through the configured n8n credential.

The actual API key must remain in n8n and must not be committed to GitHub.

### Supabase

Supabase is used for invoice record storage and duplicate detection.

The workflow uses an n8n-managed Supabase credential for database operations.

Database credentials and private authentication values must remain in n8n and must not be committed to the repository.

---

## Environment Configuration

The repository includes an `.env.example` file as a configuration template.

The example file documents configuration values that may be required when deploying the project in an environment that uses environment variables.

The example file does not contain real secrets or private credentials.

The actual `.env` file, when used for a local deployment, must remain outside source control.

The repository `.gitignore` excludes:

- `.env`
- `.env.*`

while allowing `.env.example` to be committed.

### Important Configuration Note

The current n8n workflow primarily uses n8n-managed credentials rather than directly reading API keys from environment variables.

Therefore, `.env.example` should be treated as a deployment/configuration reference and not as evidence that the current workflow already depends on those environment variables.

Environment-variable-based configuration should only be introduced when it is intentionally configured and tested in the target deployment environment.

---

## Public Workflow Exports

Workflow JSON files are maintained in the `workflows/` directory.

Public workflow exports must be reviewed before being committed to the repository.

Private or deployment-specific values should be removed or replaced with safe placeholders.

For example, the Error Handler workflow uses an internal email recipient for technical error notifications. The public GitHub export uses a placeholder:

`YOUR_ALERT_EMAIL@example.com`

The live n8n workflow retains the actual configured recipient.

This allows the workflow logic to remain available for review without exposing a personal or deployment-specific email address.

---

## Secrets That Must Not Be Committed

The following values must never be committed to the public repository:

- API keys
- OAuth access tokens
- OAuth client secrets
- Database passwords
- Private authentication tokens
- Real `.env` files
- Private credential configuration
- Other deployment-specific secrets

Credential references or credential names in n8n workflow exports are not the same as the underlying credentials. The actual authentication secrets must remain protected by the n8n credential system or the target deployment's secure secret-management mechanism.

---

## `.gitignore` Protection

The repository `.gitignore` includes protections for common private configuration and runtime files.

Protected categories include:

- Environment files
- n8n local/runtime data
- SQLite databases
- Credential files
- Secret files
- Editor configuration
- Logs
- Temporary files

The purpose of these rules is to reduce the risk of accidentally committing local configuration or runtime data.

`.env.example` is intentionally allowed so that the repository can document expected configuration without exposing actual values.

---

## Docker Configuration

The local n8n deployment uses Docker Compose.

The current Compose application contains:

- `n8n`
- `heif-converter`

The n8n service uses the persistent `n8n_data` Docker volume for `/home/node/.n8n`.

The HEIC/HEIF conversion service is built locally from the `heif-converter` directory and exposes its internal conversion service on port `8000`.

The current Docker Compose configuration should not be changed solely to support the public GitHub workflow export.

Any future environment-variable configuration should be intentionally introduced, tested, and documented before becoming a dependency of the workflow.

---

## Error Handling Configuration

The project includes a dedicated n8n Error Handler workflow.

The Error Handler receives execution error information from the main automation and sends an internal notification containing information such as:

- Workflow name
- Execution ID
- Last node executed
- Error message
- Execution URL

The Error Handler is intended for unexpected technical execution failures.

It is separate from normal business-process outcomes such as:

- `VALID`
- `REVIEW_REQUIRED`
- Duplicate detection
- Extraction retry

These expected workflow outcomes should not be treated as technical workflow failures.

---

## Human Review and Security

The automation does not treat AI extraction as a replacement for validation or human review.

The workflow uses deterministic validation after AI extraction and can route invoices to `REVIEW_REQUIRED` when extraction or financial validation identifies an issue.

This design reduces the risk of silently accepting incorrect extracted financial data.

Human review remains part of the workflow for invoices that cannot be confidently processed automatically.

---

## Repository Security Practices

Before committing workflow changes:

1. Review the workflow export.
2. Check for API keys or authentication tokens.
3. Check for personal or deployment-specific information.
4. Check email recipients and notification destinations.
5. Confirm credentials remain n8n-managed.
6. Confirm `.env` and other private files are ignored.
7. Review the Git diff before committing.
8. Commit only the intended changes.

Public workflow exports should be treated as shareable source code and reviewed accordingly.

---

## Current Security Configuration

At the current project stage:

- GitHub repository is public.
- Workflow source is version-controlled.
- `.env.example` is committed as a configuration template.
- Actual credentials remain in n8n.
- The live Error Handler retains its actual internal alert recipient.
- The public Error Handler workflow export uses a placeholder recipient.
- `.env` is excluded by `.gitignore`.
- n8n runtime data is excluded from source control.
- No actual API keys or authentication secrets should be stored in the repository.

---

## Deployment Principle

The repository should contain the information required to understand, review, and reproduce the workflow architecture without containing the private credentials required to operate a specific deployment.

In practice:

**GitHub**
→ workflow logic, documentation, configuration examples, and sanitized exports

**n8n**
→ credentials, private notification recipients, and deployment-specific configuration

**Docker**
→ local/runtime infrastructure and persistent n8n data

This separation should be maintained as the project evolves.
