---
name: get-project
description: Fetches project details including status, deployment version, stages, and process info. Use when the user asks to check a project, verify deployment state, or inspect project configuration.
---

# Get Project

Fetch detailed information about a specific project, including its status, deployment version, stage configuration, and whether it has a published process.

## When to Use

- The user wants to inspect a specific project's details
- Verifying deployment version and status before triggering workflows
- Checking if a project has a published process
- Viewing current stage information

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- An API key with `projects:read` scope
- A valid project ID (from `list-projects` or `create-project`)

## Typical Workflow

1. Use `list-projects` to find the project ID
2. Use `get-project` to inspect the project details
3. Use `get-instance` to inspect the live deployment instance

## Command

```bash
codika get project <projectId> [options]
```

### Arguments

| Argument | Description |
|----------|-------------|
| `<projectId>` | Project ID (required) |

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--api-url <url>` | Override API URL | — |
| `--api-key <key>` | Override API key | — |
| `--profile <name>` | Use a specific profile instead of the active one | — |
| `--json` | Output result as JSON | — |

## Examples

**Get project details:**

```bash
codika get project wIkjqoLC88abc123
```

**JSON output:**

```bash
codika get project wIkjqoLC88abc123 --json
```

## Expected Output

```
✓ Project

  Project ID:   wIkjqoLC88abc123
  Name:         Creafid Receipt Processor
  Status:       in_progress
  Published:    yes
  Process ID:   11fe8iPQ4pBlskVIknz4
  Deployment:   v1.50 (2026-03-31)
  Stages:       2 (current: 2)
  Created:      2026-03-27
```

## Access Control

- **Admin keys** (`cka_`): can access any project in any organization
- **Org owners/admins**: can access all projects in their organization
- **Regular members**: can only access projects they created

The response does not expose internal fields like `roles`, `stages` configuration, or `documentTags`.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| 403 / Missing scope | Key lacks `projects:read` scope | Update key with `projects:read` scope (see `update-organization-key` skill) |
| 403 / Permission denied | Member trying to access another user's project | Only the creator, org admins, and org owners can access a project |
| 404 / Project not found | Invalid project ID | Check the project ID with `list-projects` |

## Exit Codes

- `0` — Success
- `1` — API error or project not found
- `2` — CLI validation error

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file
