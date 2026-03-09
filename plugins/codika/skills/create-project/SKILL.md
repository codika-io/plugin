---
name: create-project
description: Creates a new project on the Codika platform for deploying use cases. Use when the user asks to create a project, set up a new project, or initialize a project before deployment. Projects are required before deploying use cases.
---

# Create Project

Create a new project on the Codika platform.

## When to Use

- The user wants to create a new project for deploying use cases
- Before deploying a use case if no project exists yet

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)

## Command

```bash
codika project create --name "My Project" [options]
```

### Required Options

| Flag | Description |
|------|-------------|
| `--name <name>` | Project name |

### Optional Options

| Flag | Description |
|------|-------------|
| `--description <desc>` | Project description |
| `--template-id <id>` | Template ID (defaults to platform default) |
| `--organization-id <id>` | Organization ID (required for admin keys, derived from org API keys) |
| `--path <dir>` | Write the project file into this directory after creation (links the project to a use case folder) |
| `--project-file <path>` | Custom filename for project file (default: `project.json`) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

## Examples

**Create project and save `project.json` in the current directory (recommended):**

```bash
cd my-use-case
codika project create --name "Email Automations" --path .
```

This creates the project on the platform and writes `project.json` (containing the project ID and organization ID from the active profile) into the current directory. The `organizationId` in `project.json` enables automatic profile selection during deployment when you have multiple profiles.

**Create project and link to a specific use case folder:**

```bash
codika project create --name "Email Automations" --path ./my-use-case
```

**Without `--path` (just prints the project ID, does not write `project.json`):**

```bash
codika project create --name "Email Automations"
```

**With description and JSON output:**

```bash
codika project create \
  --name "Sales Pipeline" \
  --description "Automated sales workflow processes" \
  --json
```

## Expected Output

**Success (with --path):**

```
✓ Project Created Successfully

  Project ID: abc123-def456
  Request ID: req-789
  Wrote project.json to /path/to/my-use-case/project.json
```

**Success (without --path):**

```
✓ Project Created Successfully

  Project ID: abc123-def456
  Request ID: req-789
```

**JSON success:**

```json
{
  "success": true,
  "data": {
    "projectId": "abc123-def456"
  },
  "requestId": "req-789",
  "projectJsonPath": "/path/to/my-use-case/project.json"
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` or check `codika whoami` (see `setup-codika` skill) |
| "Organization ID is required" | Using an admin key without specifying org | Add `--organization-id <id>` |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika login` with a valid key |
| 403 / Forbidden | Key doesn't have project creation permission | Check API key permissions in the dashboard |

## Exit Codes

- `0` — Project created successfully
- `1` — API error (deployment failed)
- `2` — CLI validation error (missing required flags)
