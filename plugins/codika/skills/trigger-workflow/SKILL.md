---
name: trigger-workflow
description: Triggers a deployed Codika workflow and optionally polls for results. Use when the user asks to trigger, run, execute, or test a deployed workflow. Supports inline JSON payloads, file payloads, polling with timeout, and auto-resolves processInstanceId from project.json.
---

# Trigger Workflow

Trigger a deployed Codika workflow and optionally poll for results.

## Prerequisites

- Use case is deployed (`project.json` contains `devProcessInstanceId`)
- Authenticated via `codika login` (check with `codika whoami`) or `CODIKA_API_KEY` env var

## Resolving the Workflow ID

The `workflowId` is the `workflowTemplateId` from `config.ts`. If the user provides a use case folder path, read `config.ts` to find the available workflow IDs and their trigger types.

**Supported trigger types:**
- **HTTP triggers** (`trigger.type === 'http'`): Triggered with optional payload
- **Schedule triggers** (`trigger.type === 'schedule'`): Triggered via the `manualTriggerUrl` configured in `config.ts`. The CLI creates an execution document upfront and passes it to the workflow, so `--poll` works exactly like HTTP triggers. No payload is needed.

## Using the CLI (Recommended)

### Basic trigger (fire-and-forget)

```bash
codika trigger <workflowId>
```

Returns immediately with an `executionId`. The `processInstanceId` is auto-resolved from `project.json` in the current directory.

### With payload

Use a heredoc with `--payload-file -` to pass JSON via stdin. This avoids shell escaping issues with inline JSON.

```bash
codika trigger <workflowId> --payload-file - <<'EOF'
{"field": "value", "nested": {"key": "no escaping needed"}}
EOF
```

For pre-existing JSON files, pass the path directly:

```bash
codika trigger <workflowId> --payload-file input.json
```

### Poll for results

```bash
# Wait for completion (default timeout: 120s)
codika trigger <workflowId> --poll --payload-file - <<'EOF'
{"field": "value"}
EOF

# Custom timeout
codika trigger <workflowId> --poll --timeout 60

# Save result to file
codika trigger <workflowId> --poll -o result.json
```

### All options

| Option                       | Description                                              |
| ---------------------------- | -------------------------------------------------------- |
| `--process-instance-id <id>` | Explicit process instance ID (overrides project file)    |
| `--path <path>`              | Path to use case folder with project file                |
| `--project-file <path>`      | Custom project file path (default: `project.json`)       |
| `--payload-file <path>`      | Read payload from a JSON file, or `-` for stdin          |
| `--poll`                     | Wait for execution to complete                           |
| `--timeout <seconds>`        | Max poll time (default: 120)                             |
| `-o, --output <path>`        | Save result to file (with --poll)                        |
| `--api-url <url>`            | Override API URL                                         |
| `--api-key <key>`            | Override API key                                         |
| `--profile <name>`           | Use a specific profile instead of the active one         |
| `--json`                     | Structured JSON output                                   |

### Process instance ID resolution

The command resolves `processInstanceId` automatically:

1. `--process-instance-id` flag (highest priority)
2. Project file (`--project-file` or default `project.json`) in `--path` directory
3. Project file (`--project-file` or default `project.json`) in current directory

### Examples

```bash
# From inside a use case folder (project.json auto-detected)
cd my-use-case/
codika trigger proposal-generation --poll --payload-file - <<'EOF'
{"transcript": "Meeting notes from client call..."}
EOF

# Explicit path
codika trigger proposal-generation --path ./my-use-case --poll

# CI/CD with JSON output
codika trigger proposal-generation \
  --payload-file test-input.json \
  --poll --timeout 60 --json

# Explicit process instance ID
codika trigger proposal-generation \
  --process-instance-id abc-123 \
  --poll --payload-file - <<'EOF'
{"text": "test"}
EOF

# Trigger a scheduled workflow manually (no payload needed)
codika trigger daily-report --poll
```

## Interpreting Results

- `status: "success"` — `resultData` contains workflow output
- `status: "failed"` — `errorDetails.message` describes the failure
- `status: "pending"` — still running, keep polling

## Key Notes

- Both HTTP and schedule triggers are supported
- Schedule triggers require a `manualTriggerUrl` in config.ts (with a matching Webhook node in the workflow JSON)
- `workflowId` is the `workflowTemplateId` from config.ts
- For HTTP triggers: request body wraps data in `{"payload": {...}}`
- For schedule triggers: no payload needed (the workflow runs its own logic)
- Execution is async — trigger returns immediately with an `executionId`
- `--poll` works for both trigger types
- Use `codika get execution <executionId>` for full n8n execution details

## Error Reference

| HTTP | Error              | Cause                             | Fix                                                                                |
| ---- | ------------------ | --------------------------------- | ---------------------------------------------------------------------------------- |
| 401  | Invalid API key    | Wrong or expired key              | Run `codika whoami` to check, then `codika login` to re-authenticate |
| 403  | Missing scope      | Key lacks `workflows:trigger`     | Create new key with scope                                                          |
| 403  | No access          | Process instance in different org | Switch profile with `codika use <profile>`                                  |
| 404  | Workflow not found | Wrong workflowId                  | Check `config.ts`                                                                  |
| 412  | Instance inactive  | Process paused or failed          | Re-deploy or check platform                                                        |
