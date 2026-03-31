---
name: get-instance
description: Fetches process instance details (deployment parameters, status, version, workflows). Use when the user asks to check instance status, view deployment parameters, inspect what's deployed, or verify a live instance.
---

# Get Instance

Retrieve the current state of a deployed process instance, including its deployment parameters (INSTPARM values), deployment status, version, and active workflows.

## When to Use

- The user wants to check what deployment parameters are set on a live instance
- Verifying the deployment status or version of an instance
- Checking which workflows are deployed and their n8n workflow IDs
- Confirming an instance is active before triggering workflows
- Debugging parameter issues after a redeploy

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A deployed process instance (via `deploy-use-case` skill or the dashboard)
- API key with `instances:read` scope

## Typical Workflow

1. Deploy a use case (see `deploy-use-case` skill)
2. Redeploy with parameters (see `redeploy-use-case` skill)
3. Use this command to verify the parameters are set correctly
4. If incorrect, redeploy with corrected parameters

## Command

```bash
codika get instance [processInstanceId] [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `[processInstanceId]` | No | Process instance ID (auto-resolved from project.json if omitted) |

### Options

| Flag | Description |
|------|-------------|
| `--path <path>` | Path to a use case folder containing the project file |
| `--project-file <path>` | Custom project file path (default: `project.json`) |
| `--environment <env>` | Environment: `dev` (default) or `prod` |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--profile <name>` | Use a specific profile |
| `--json` | Output result as JSON |

### Process Instance ID Resolution

The process instance ID is resolved in this order:

1. Positional argument (highest priority)
2. Project file in `--path` directory — uses `devProcessInstanceId` or `prodProcessInstanceId` based on `--environment`
3. Project file in current directory — same environment-aware selection

Use `--environment prod` to target the production instance.

## Examples

**From a use case folder (reads `project.json` automatically):**

```bash
cd my-use-case && codika get instance
```

**Production instance:**

```bash
codika get instance --path ./my-use-case --environment prod
```

**Explicit instance ID:**

```bash
codika get instance 019d312f-517c-726e-83ac-b678f2ad6afc
```

**JSON output for scripting:**

```bash
codika get instance --environment prod --json
```

## Expected Output

```
✓ Process Instance

  Instance ID:    019d312f-517c-726e-83ac-b678f2ad6afc
  Process ID:     11fe8iPQ4pBlskVIknz4
  Environment:    prod
  Status:         deployed (active)
  Version:        1.50
  Title:          Creafid Receipt Processor

  Deployment Parameters:
    TO_EMAILS: [] (empty)
    CC_EMAILS: [] (empty)

  Workflows:
    - http-process-receipt (n8n: NGC6dwX7WL1F29DI)
    - http-get-receipts (n8n: XE4RRqOIaJF1m8Fu)
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Process instance ID is required" | No argument and no `project.json` | Provide the ID as argument or run from a use case folder |
| "Process instance not found" | Invalid instance ID | Verify the ID from `project.json` or the dashboard |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami` to check, then `codika login` |
| 403 / Permission Denied | API key from different organization | Use the correct profile: `codika use <profile>` |

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
