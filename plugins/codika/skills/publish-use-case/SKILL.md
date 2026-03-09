---
name: publish-use-case
description: Publishes a deployed Codika use case to production, making it available to users. Use when the user asks to publish, promote, go live, or release a use case to production. Handles visibility settings, shared-with configuration, and optional dev/prod auto-toggle.
---

# Publish Use Case

Promote a deployed use case from dev to production. Creates a production process instance, activates the prod n8n workflow, and optionally pauses the dev instance when the prod instance runs.

## When to Use

- The user wants to promote a deployed use case from dev to production
- Making a process available to end users for the first time
- Setting visibility (private, organizational, public) on first publish
- Configuring dev/prod auto-toggle behavior

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A deployed use case with a `project.json` containing a `deployments` map (run `deploy-use-case` first)
- An API key with `deploy:use-case` scope

## Typical Workflow

1. Deploy a use case (see `deploy-use-case` skill) — produces `devProcessInstanceId` and the `deployments` map in `project.json`
2. Find the template ID in `project.json` under the `deployments` map
3. Publish using the template ID

```bash
# 1. Deploy
codika deploy use-case .

# 2. Check project.json for the template ID
# "deployments": { "1.0": { "templateId": "tmpl-abc123", ... } }

# 3. Publish
codika publish tmpl-abc123 --path .
```

## Command

```bash
codika publish <templateId> [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<templateId>` | Yes | Deployment template ID from the `deployments` map in `project.json` |

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--path <path>` | Path to the use case folder containing `project.json` | Current directory |
| `--project-file <path>` | Custom project file path | `project.json` |
| `--project-id <id>` | Override project ID (skips project file) | — |
| `--visibility <value>` | First publish only: `private`, `organizational`, or `public` | — |
| `--shared-with <value>` | For organizational processes: `owner_only`, `admins`, or `everyone` | — |
| `--auto-toggle-dev-prod` | Pause the dev instance when the prod instance is running | Off |
| `--skip-prod-instance` | Skip creating a production process instance | Off |
| `--api-url <url>` | Override API URL | — |
| `--api-key <key>` | Override API key | — |
| `--json` | Output result as JSON | — |

## Finding the Template ID

After a successful deployment, `project.json` contains a `deployments` map keyed by API version:

```json
{
  "projectId": "abc123",
  "organizationId": "org-456",
  "devProcessInstanceId": "pi-dev-789",
  "deployments": {
    "1.0": {
      "templateId": "tmpl-abc123",
      "createdAt": "2026-03-04T14:30:00.000Z"
    },
    "1.1": {
      "templateId": "tmpl-def456",
      "createdAt": "2026-03-04T15:00:00.000Z"
    }
  }
}
```

Look up the version you want to publish and use the `templateId` value — `tmpl-def456` to publish version 1.1 in this example.

## Examples

**Basic publish:**

```bash
codika publish tmpl-abc123 --path ./use-cases/marketplace/my-use-case
```

**With visibility and sharing settings:**

```bash
codika publish tmpl-abc123 --path . \
  --visibility organizational \
  --shared-with everyone
```

**With dev/prod auto-toggle:**

```bash
codika publish tmpl-abc123 --path . --auto-toggle-dev-prod
```

**JSON output for scripting:**

```bash
codika publish tmpl-abc123 --path . --json
```

## Expected Output

**Success:**

```
✓ Published successfully!

  Version:              1.0
  Template ID:          tmpl-abc123
  Prod Instance ID:     pi-prod-999
  Webhook URL:          https://n8n.example.com/webhook/abc-123
```

## What Happens on Publish

1. The template ID is resolved against the platform API
2. Visibility and sharing settings are applied (first publish only for visibility)
3. A production n8n workflow instance is activated
4. A prod process instance is created in Firestore
5. If `--auto-toggle-dev-prod` is set, the dev instance is configured to pause when prod is running
6. `prodProcessInstanceId` is saved to `project.json`

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Template not found" | Wrong template ID | Check the `deployments` map in `project.json` |
| "No deployments found in project file" | `project.json` has no `deployments` map | Deploy first with `codika deploy use-case .` |
| "Invalid visibility value" | Bad `--visibility` value | Use `private`, `organizational`, or `public` |
| "Invalid shared-with value" | Bad `--shared-with` value | Use `owner_only`, `admins`, or `everyone` |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| "Process already published" | Prod instance already exists | Use `codika deploy use-case .` to update the existing prod instance |

## Exit Codes

- `0` — Publish successful
- `1` — Publish failed (API error)
- `2` — CLI validation error (missing arguments, invalid flags)

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
