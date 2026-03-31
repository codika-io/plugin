---
name: list-instances
description: Lists all process instances for an organization. Use when the user wants to see deployed instances, check instance status across environments, audit what's running, or find an instance ID for further operations like activate/deactivate or get-instance.
---

# List Instances

Retrieve a list of all process instances for the authenticated organization. Useful for auditing deployments, checking status across environments, or finding instance IDs.

## When to Use

- The user wants to see all deployed process instances
- Checking instance status across dev and prod environments
- Auditing what is currently running in an organization
- Finding an instance ID to pass to `get-instance`, `instance-activate`, or `list-executions`

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- An API key with `instances:read` scope

## Typical Workflow

1. List instances to get an overview of all deployments
2. Use a returned instance ID with `get-instance` for detailed inspection
3. Use `instance-activate` to activate/deactivate instances as needed
4. Use `list-executions` with an instance ID to check execution history

## Command

```bash
codika list instances [options]
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--environment <env>` | Filter by environment (`dev` or `prod`) | All environments |
| `--archived` | Include archived instances | Off |
| `--limit <n>` | Number of instances to return | `50` |
| `--api-url <url>` | Override API URL | — |
| `--api-key <key>` | Override API key | — |
| `--profile <name>` | Use a specific profile instead of the active one | — |
| `--json` | Output result as JSON | — |

## Examples

**List all instances:**

```bash
codika list instances
```

**Filter by environment:**

```bash
codika list instances --environment prod
```

**Include archived instances:**

```bash
codika list instances --archived
```

**JSON output for scripting:**

```bash
codika list instances --limit 10 --json
```

## Expected Output

**Table output:**

```
● Process Instances (org: My Organization)

  Title                    Env    Status       Version   Last Executed
  ──────────────────────── ────── ──────────── ───────── ───────────────────
  Email Automation         prod   ✓ active     v2.1.0    2026-03-30 14:30:00
  Email Automation         dev    ○ inactive   v2.2.0    2026-03-30 10:15:00
  Weekly Report            prod   ✓ active     v1.0.0    2026-03-29 08:00:00
  Data Ingestion Pipeline  dev    ✗ failed     v1.3.0    2026-03-28 16:45:00

  Showing 4 instances
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| 403 / Missing scope | Key lacks `instances:read` scope | Create key with `instances:read` scope |

Note: Zero instances is not an error — the CLI prints "No instances found." and exits with code 0.

## Exit Codes

- `0` — Success
- `1` — API error

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
