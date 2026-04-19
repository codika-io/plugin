---
name: redeploy-use-case
description: Redeploys an existing Codika process instance with different parameters without creating a new template version. Use when the user wants to change deployment parameters (phone numbers, API keys, config values), retry a failed deployment, or switch configuration on an already-deployed instance.
---

# Redeploy Use Case

Redeploy an existing process instance with different parameters. Does NOT create a new template version — use `codika deploy use-case` for code changes instead.

## When to Use

- Changing deployment parameters (phone numbers, API keys, config values) on an existing instance
- Retrying a failed deployment
- Switching configuration after publishing (e.g., dev phone number changes after prod publish)

**Do NOT use for deploying new code changes** — use `codika deploy use-case` for that (see `deploy-use-case` skill).

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- An existing deployed use case with a `project.json` containing `devProcessInstanceId`
- An API key with `deploy:use-case` scope

## Command

```bash
codika redeploy [options]
```

## Options

| Flag | Description |
|------|-------------|
| `--process-instance-id <id>` | Target process instance ID (explicit) |
| `--path <path>` | Use case folder (resolves from project.json, default: cwd) |
| `--project-file <path>` | Custom project file (e.g., `project-wat.json`) |
| `--environment <env>` | `dev` (default) or `prod` — which instance to redeploy |
| `--param <KEY=VALUE>` | Set deployment parameter (repeatable) |
| `--params <json>` | JSON string with all parameters (agent-friendly) |
| `--params-file <path>` | JSON file with parameter overrides |
| `--force` | Force redeploy even if deployment is not in failed state |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |
| `--profile <name>` | Use a specific CLI profile instead of the active one |

## Process Instance Resolution

The target process instance ID is resolved in this order:

1. `--process-instance-id` flag (highest priority)
2. `--path` + `--project-file` + `--environment`:
   - Reads `project.json` (or custom file via `--project-file`)
   - `--environment dev` (default) uses `devProcessInstanceId`
   - `--environment prod` uses `prodProcessInstanceId`

## Parameter Input Methods

Three ways to provide parameters, merged with priority (highest first):

1. **`--param KEY=VALUE`** (repeatable) — best for humans, small changes
2. **`--params '{"key":"val"}'`** (inline JSON) — best for agents, programmatic use
3. **`--params-file path.json`** (file) — best for CI/CD, many parameters

**Important: Parameters are merged, not replaced.** You only need to pass the parameters you want to change — the backend merges them with the existing deployment parameters. Unchanged parameters are automatically preserved. For example, if an instance has 10 parameters and you only pass `--param USER_BOT_PHONE=+123`, only the phone number changes; all other 9 parameters stay the same.

If no parameters are provided at all, the existing deployment parameters are preserved (useful for retry-only scenarios).

## Agent Usage

When calling from agent automation, prefer `--params` with inline JSON and `--json` output:

```bash
codika redeploy --path /path/to/use-case \
  --params '{"USER_BOT_PHONE":"+1234567890","COMMUNITY_NAME":"My Community"}' \
  --force --json
```

With a custom project file:

```bash
codika redeploy --path /path/to/use-case \
  --project-file project-client.json \
  --params '{"USER_BOT_PHONE":"+1234567890"}' \
  --force --json
```

## Examples

**Change a single parameter:**

```bash
codika redeploy --path . --param USER_BOT_PHONE=+NEW --force
```

**Change multiple parameters:**

```bash
codika redeploy --path . --param KEY1=VAL1 --param KEY2=VAL2 --force
```

**Agent: pass all params as JSON:**

```bash
codika redeploy --path . --params '{"KEY1":"VAL1","KEY2":"VAL2"}' --force --json
```

**From file:**

```bash
codika redeploy --path . --params-file ./deploy-params.json --force
```

**Retry a failed deployment (no param changes):**

```bash
codika redeploy --path .
```

**Redeploy prod instance:**

```bash
codika redeploy --path . --environment prod --force
```

**Custom project file (multi-project use cases):**

```bash
codika redeploy --path . --project-file project-wat.json --force
```

## Expected Output

**Success (human-readable):**

```
✓ Redeployed successfully!

  Status:          deployed
  Instance ID:     <deploymentInstanceId>
  Workflows:       <count> deployed
```

**Success (JSON, with `--json` flag):**

```json
{
  "success": true,
  "data": {
    "deploymentStatus": "deployed",
    "deploymentInstanceId": "...",
    "n8nWorkflowIds": ["..."]
  },
  "requestId": "..."
}
```

**Error (human-readable):**

```
✗ Redeploy failed: <error_code> — <error_message>
```

## Relationship to Other Commands

| Command | Purpose |
|---------|---------|
| `codika deploy use-case` | Creates NEW template versions with code changes |
| `codika publish` | Promotes a dev template to production |
| `codika redeploy` | Changes PARAMETERS on existing instances (no new version) |

Use `deploy use-case` when workflows or `config.ts` changed. Use `redeploy` when only runtime parameters need updating.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Process instance not found" | Invalid instance ID or missing project.json | Check `project.json` has correct `devProcessInstanceId` / `prodProcessInstanceId` |
| "Deployment currently in progress" | Another deployment is running | Wait for it to complete, then retry |
| "Deployment is not in failed state" | Trying to redeploy a healthy instance without `--force` | Add `--force` flag |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| "No process instance ID found" | Missing `--process-instance-id` and no ID in project file | Deploy first with `codika deploy use-case .` or provide `--process-instance-id` |

## Exit Codes

- `0` — Redeployment successful
- `1` — Redeployment failed (API error)
- `2` — CLI validation error (missing flags, invalid parameters)

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Profile matching `organizationId` from the project file
4. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
