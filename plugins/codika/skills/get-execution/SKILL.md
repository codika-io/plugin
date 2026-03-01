---
name: get-execution
description: Fetch full n8n workflow execution details for debugging
trigger:
  when: User asks to debug, inspect, or get details of a workflow execution
  actions: [get, fetch, inspect, debug, view]
  targets: [execution, workflow run, execution details, execution logs]
---

# Get Execution

Retrieve full n8n workflow execution data (node-by-node) for debugging a triggered workflow.

## When to Use

- The user wants to debug a workflow execution that failed or returned unexpected results
- Inspecting node-level input/output for a specific execution
- Fetching sub-workflow execution details recursively

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)
- A workflow that has been deployed and triggered
- An execution ID from the trigger response (see `trigger-workflow` skill)

## Typical Workflow

1. Deploy a use case (see `deploy-use-case` skill)
2. Trigger a workflow (see `trigger-workflow` skill) -- get `executionId` from the response
3. Use this command to get full execution details

## Command

```bash
codika-helper get execution <executionId> [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<executionId>` | Yes | The Codika execution ID returned by the trigger response |

### Options

| Flag | Description |
|------|-------------|
| `--process-instance-id <id>` | Explicit process instance ID |
| `--path <path>` | Path to a use case folder containing `project.json` |
| `--deep` | Recursively fetch sub-workflow executions |
| `--slim` | Strip noise (`pairedItem`, `workflowData`) for cleaner output |
| `-o <path>` / `--output <path>` | Save output to a file instead of stdout |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

### Process Instance ID Resolution

The process instance ID is resolved in this order:

1. `--process-instance-id` flag (highest priority)
2. `project.json` in the `--path` directory (uses `devProcessInstanceId`)
3. `project.json` in the current directory

## Examples

**From a use case folder (reads `project.json` automatically):**

```bash
cd my-use-case && codika-helper get execution abc-123-def
```

**With explicit process instance ID:**

```bash
codika-helper get execution abc-123-def --process-instance-id pi-456
```

**Deep + slim for debugging:**

```bash
codika-helper get execution abc-123-def --deep --slim
```

**Save to file:**

```bash
codika-helper get execution abc-123-def --deep --slim -o /tmp/execution.json
```

## Expected Output

**Summary:**

```
Execution abc-123-def

  Status:           success
  n8n Execution ID: 12345
  Node Count:       8
  Duration:         2.3s

  Nodes:
    1. Webhook           success
    2. Set Variables      success
    3. HTTP Request       success
    4. IF                 success
    5. Format Output      success
```

## Deep Mode

When `--deep` is used, the CLI recursively fetches sub-workflow executions and attaches them as `_subExecutions` on the parent node that triggered them. This gives a complete picture of the full execution tree.

## Slim Mode

When `--slim` is used, the CLI strips noisy fields (`pairedItem`, `workflowData`) from the execution data. This produces cleaner, more readable output for debugging purposes.

Combine both for the best debugging experience:

```bash
codika-helper get execution abc-123-def --deep --slim
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "No process instance ID found" | No `--process-instance-id` flag and no `project.json` | Provide `--process-instance-id` or run from a use case folder with `project.json` |
| "Execution not found" | Invalid execution ID or execution expired | Verify the execution ID from the trigger response |
| "n8n execution ID not available" | Old workflow without execution tracking | Re-deploy the use case with the latest CLI version |
| 401 / Unauthorized | Invalid or expired API key | Run `codika-helper whoami` to check, then `codika-helper login` to re-authenticate |

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika-helper whoami` to check the current identity, or `codika-helper use <profile>` to switch profiles.
