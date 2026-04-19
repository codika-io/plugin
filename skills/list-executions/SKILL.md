---
name: list-executions
description: Lists recent n8n workflow executions for a process instance. Use when the user wants to check recent execution status, find failed executions, monitor workflow runs, or get a list of execution IDs to pass to get-execution for deeper debugging.
---

# List Executions

Retrieve a paginated list of recent executions for a deployed process instance. Useful for monitoring workflow runs, spotting failures at a glance, or finding an execution ID to inspect in detail.

## When to Use

- The user wants to check the status of recent workflow runs
- Finding failed executions to debug
- Monitoring how often a workflow is triggered
- Getting an execution ID to pass to `get-execution` for node-level details

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A deployed process instance (`devProcessInstanceId` or `prodProcessInstanceId` in `project.json`)
- An API key with `executions:read` scope

## Typical Workflow

1. Deploy a use case (see `deploy-use-case` skill) — produces a `devProcessInstanceId` in `project.json`
2. Trigger the workflow one or more times (see `trigger-workflow` skill)
3. List executions to check overall status
4. Use a returned execution ID with `get-execution` for deeper debugging

## Command

```bash
codika list executions <processInstanceId> [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<processInstanceId>` | Yes | Process instance ID — dev or prod — from `project.json` |

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--workflow-id <id>` | Filter results to a specific workflow | All workflows |
| `--failed-only` | Return only failed executions | Off |
| `--limit <n>` | Number of executions to return (1–100) | `20` |
| `--api-url <url>` | Override API URL | — |
| `--api-key <key>` | Override API key | — |
| `--json` | Output result as JSON | — |

## Dev vs Prod

`project.json` contains both instance IDs after deploy and publish:

```json
{
  "devProcessInstanceId": "pi-dev-789",
  "prodProcessInstanceId": "pi-prod-999"
}
```

Use `devProcessInstanceId` to list dev executions and `prodProcessInstanceId` to list prod executions.

## Examples

**List recent executions for a dev instance:**

```bash
codika list executions pi-dev-789
```

**Filter by workflow:**

```bash
codika list executions pi-dev-789 --workflow-id wf-abc123
```

**Failed executions only:**

```bash
codika list executions pi-dev-789 --failed-only
```

**JSON output for scripting:**

```bash
codika list executions pi-dev-789 --limit 50 --json
```

## Expected Output

**Table output:**

```
● Recent Executions (pi-dev-789...)

  ID             Workflow             Status     Duration   Created
  ────────────── ──────────────────── ────────── ────────── ───────────────────
  exec_abc123    main-workflow        ✓ success  2.3s       2026-03-04 14:30:01
  exec_def456    main-workflow        ✓ success  1.8s       2026-03-04 13:15:42
  exec_ghi789    sub-workflow         ✗ failed   0.4s       2026-03-04 12:05:10
    └─ [HTTP Request] 403 Forbidden
  exec_jkl012    main-workflow        ✓ success  3.1s       2026-03-03 18:44:33

  Showing 4 executions
```

**Failed executions show the error message and failed node name on the next line.**

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Process instance not found" | Invalid process instance ID | Check `devProcessInstanceId` or `prodProcessInstanceId` in `project.json` |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |

Note: Zero executions is not an error — the CLI prints "No executions found." and exits with code 0.

## Exit Codes

- `0` — Success
- `1` — API error or instance not found
- `2` — CLI validation error (invalid flags or arguments)

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
