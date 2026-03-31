---
name: list-projects
description: Lists all projects for an organization. Use when the user wants to see their projects, check project status, find a project ID, or audit what projects exist in the org.
---

# List Projects

Retrieve a list of all projects for the authenticated organization. Useful for finding project IDs, checking status, or auditing project state.

## When to Use

- The user wants to see all projects in their organization
- Finding a project ID to pass to `get-project` or `deploy-use-case`
- Checking which projects have published processes
- Auditing project status across the organization

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- An API key with `projects:read` scope

## Typical Workflow

1. List projects to find the target project
2. Use a returned project ID with `get-project` for detailed inspection
3. Use the project ID with `deploy-use-case` for deployments

## Command

```bash
codika list projects [options]
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--archived` | Show archived projects instead of active ones | Off |
| `--limit <n>` | Number of results to return (max 100) | `50` |
| `--api-url <url>` | Override API URL | — |
| `--api-key <key>` | Override API key | — |
| `--profile <name>` | Use a specific profile instead of the active one | — |
| `--json` | Output result as JSON | — |

## Examples

**List all projects:**

```bash
codika list projects
```

**List archived projects:**

```bash
codika list projects --archived
```

**JSON output for scripting:**

```bash
codika list projects --limit 10 --json
```

## Expected Output

**Table output:**

```
● Projects (xwk9CcT440...)

  Name                              Status         Published  Created
  ────────────────────────────────── ────────────── ────────── ──────────
  Creafid Receipt Processor          ● in_progress  yes        2026-03-27
  Growth Analytics                   ○ draft        no         2026-03-25

  Showing 2 projects
```

## Access Control

- **Admin keys** (`cka_`) and **org owners/admins**: see all projects in the organization
- **Regular members**: see only projects they created

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| 403 / Missing scope | Key lacks `projects:read` scope | Update key with `projects:read` scope (see `update-organization-key` skill) |

## Exit Codes

- `0` — Success
- `1` — API error
- `2` — CLI validation error

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika whoami` to check the current identity, or `codika use <profile>` to switch profiles.
