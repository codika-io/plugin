---
name: trigger-workflow
description: Trigger a deployed Codika workflow and optionally poll for results
trigger:
  when: User asks to trigger, run, execute, or test a deployed workflow
  actions: [trigger, run, execute, test, fire]
  targets: [workflow, use case, process, automation]
---

# Trigger Workflow

Trigger a deployed Codika workflow and optionally poll for results.

## Prerequisites

- Use case is deployed (`project.json` contains `devProcessInstanceId`)
- Authenticated via `codika-helper login` (check with `codika-helper whoami`) or `CODIKA_API_KEY` env var

## Using the CLI (Recommended)

### Basic trigger (fire-and-forget)

```bash
codika-helper trigger <workflowId>
```

Returns immediately with an `executionId`. The `processInstanceId` is auto-resolved from `project.json` in the current directory.

### With payload

```bash
# Inline JSON
codika-helper trigger <workflowId> --payload '{"field": "value"}'

# From a JSON file
codika-helper trigger <workflowId> --payload-file input.json
```

### Poll for results

```bash
# Wait for completion (default timeout: 120s)
codika-helper trigger <workflowId> --payload-file input.json --poll

# Custom timeout
codika-helper trigger <workflowId> --poll --timeout 60

# Save result to file
codika-helper trigger <workflowId> --poll -o result.json
```

### All options

| Option                       | Description                                           |
| ---------------------------- | ----------------------------------------------------- |
| `--process-instance-id <id>` | Explicit process instance ID (overrides project.json) |
| `--path <path>`              | Path to use case folder with project.json             |
| `--payload <json>`           | Inline JSON payload string                            |
| `--payload-file <path>`      | Read payload from a JSON file                         |
| `--poll`                     | Wait for execution to complete                        |
| `--timeout <seconds>`        | Max poll time (default: 120)                          |
| `-o, --output <path>`        | Save result to file (with --poll)                     |
| `--api-url <url>`            | Override API URL                                      |
| `--api-key <key>`            | Override API key                                      |
| `--json`                     | Structured JSON output                                |

### Process instance ID resolution

The command resolves `processInstanceId` automatically:

1. `--process-instance-id` flag (highest priority)
2. `project.json` in `--path` directory
3. `project.json` in current directory

### Examples

```bash
# From inside a use case folder (project.json auto-detected)
cd my-use-case/
codika-helper trigger proposal-generation --payload '{"transcript": "..."}'

# Explicit path
codika-helper trigger proposal-generation --path ./my-use-case --poll

# CI/CD with JSON output
codika-helper trigger proposal-generation \
  --payload-file test-input.json \
  --poll --timeout 60 --json

# Explicit process instance ID
codika-helper trigger proposal-generation \
  --process-instance-id abc-123 \
  --payload '{"text": "test"}'
```

## Interpreting Results

- `status: "success"` — `resultData` contains workflow output
- `status: "failed"` — `errorDetails.message` describes the failure
- `status: "pending"` — still running, keep polling

## Key Notes

- Only workflows with `trigger.type === 'http'` can be triggered
- `workflowId` is the `workflowTemplateId` from config.ts
- Request body wraps data in `{"payload": {...}}`
- Execution is async — trigger returns immediately
- Use `codika-helper get execution <executionId>` for full n8n execution details

## Error Reference

| HTTP | Error              | Cause                             | Fix                                                                                |
| ---- | ------------------ | --------------------------------- | ---------------------------------------------------------------------------------- |
| 401  | Invalid API key    | Wrong or expired key              | Run `codika-helper whoami` to check, then `codika-helper login` to re-authenticate |
| 403  | Missing scope      | Key lacks `workflows:trigger`     | Create new key with scope                                                          |
| 403  | No access          | Process instance in different org | Switch profile with `codika-helper use <profile>`                                  |
| 404  | Workflow not found | Wrong workflowId                  | Check `config.ts`                                                                  |
| 412  | Instance inactive  | Process paused or failed          | Re-deploy or check platform                                                        |
