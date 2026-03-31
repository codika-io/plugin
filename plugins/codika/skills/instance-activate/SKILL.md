---
name: instance-activate
description: Activates or deactivates a Codika process instance. Use when the user wants to pause workflows, resume after maintenance, toggle between dev/prod environments, or temporarily disable a running instance without undeploying it.
---

# Activate / Deactivate Instance

Activate or deactivate a deployed process instance to control whether its workflows are running. Does NOT undeploy — use this for pausing and resuming.

## When to Use

- Pausing all workflows in an instance during maintenance or debugging
- Resuming workflows after maintenance
- Toggling between dev and prod environments
- Temporarily disabling a workflow without undeploying it

**Do NOT use for undeploying** — deactivating keeps the instance and its configuration intact. The instance can be reactivated at any time.

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A deployed process instance (`devProcessInstanceId` or `prodProcessInstanceId` in `project.json`)
- An API key with `instances:manage` scope

## Commands

```bash
codika instance activate [processInstanceId] [options]
codika instance deactivate [processInstanceId] [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `[processInstanceId]` | No | Process instance ID. If omitted, resolved from `project.json` |

### Options

| Flag | Description |
|------|-------------|
| `--path <path>` | Use case folder (resolves from project.json, default: cwd) |
| `--project-file <path>` | Custom project file (e.g., `project-client.json`) |
| `--environment <env>` | `dev` (default) or `prod` — which instance to target |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--profile <name>` | Use a specific profile instead of the active one |
| `--json` | Output result as JSON |

## Process Instance ID Resolution

The target process instance ID is resolved in this order:

1. Positional argument `[processInstanceId]` (highest priority)
2. `--path` + `--project-file` + `--environment`:
   - Reads `project.json` (or custom file via `--project-file`)
   - `--environment dev` (default) uses `devProcessInstanceId`
   - `--environment prod` uses `prodProcessInstanceId`

## Auto-Toggle Behavior

Activating a prod instance may automatically deactivate the corresponding dev instance (and vice versa). This prevents both environments from running simultaneously. The CLI reports when an auto-toggle occurs.

## Agent Usage

When automating, prefer the positional argument with `--json`:

```bash
codika instance activate pi_prod_789 --json
codika instance deactivate pi_dev_456 --json
```

From a use case folder:

```bash
codika instance activate --path /path/to/use-case --environment prod --json
```

## Examples

**Activate prod instance by ID:**

```bash
codika instance activate pi_prod_789
```

**Deactivate dev instance by ID:**

```bash
codika instance deactivate pi_dev_456
```

**Activate from use case folder:**

```bash
codika instance activate --environment prod
```

**Deactivate from use case folder:**

```bash
codika instance deactivate --environment dev
```

**Custom project file:**

```bash
codika instance activate --project-file project-client.json --environment prod
```

**JSON output:**

```bash
codika instance activate pi_prod_789 --json
```

## Expected Output

**Activate (human-readable):**

```
✓ Instance activated

  Instance:    pi_prod_789
  Environment: prod
  Status:      active
  Workflows:   3 activated
```

**Deactivate (human-readable):**

```
✓ Instance deactivated

  Instance:    pi_dev_456
  Environment: dev
  Status:      inactive
  Workflows:   3 deactivated
```

**With auto-toggle:**

```
✓ Instance activated

  Instance:    pi_prod_789
  Environment: prod
  Status:      active
  Workflows:   3 activated

  ⚠ Auto-deactivated dev instance pi_dev_456
```

**JSON success:**

```json
{
  "success": true,
  "processInstanceId": "pi_prod_789",
  "environment": "prod",
  "status": "active",
  "workflowsAffected": 3,
  "autoToggled": {
    "processInstanceId": "pi_dev_456",
    "environment": "dev",
    "status": "inactive"
  }
}
```

## Relationship to Other Commands

| Command | Purpose |
|---------|---------|
| `codika instance activate` | Resume workflows on an existing instance |
| `codika instance deactivate` | Pause workflows without undeploying |
| `codika redeploy` | Change parameters on an existing instance |
| `codika deploy use-case` | Deploy new code changes (creates new version) |

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Process instance not found" | Invalid instance ID or missing project.json | Check `project.json` has correct `devProcessInstanceId` / `prodProcessInstanceId` |
| "Instance already active/inactive" | Instance is already in the target state | No action needed |
| "Instance in failed state" | Cannot activate a failed instance | Fix with `codika redeploy` first |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |

## Exit Codes

- `0` — State change successful
- `1` — API error
- `2` — CLI validation error (missing flags or arguments)

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Profile matching `organizationId` from the project file
4. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
