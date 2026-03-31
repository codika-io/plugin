---
name: create-organization-key
description: Creates an organization API key for a Codika organization. Use when the user needs an API key for deployment, triggering workflows, or managing a specific organization. Requires personal key or admin key with api-keys:manage scope.
---

# Create Organization Key

Create an organization API key (`cko_`) for a Codika organization.

## When to Use

- The user needs an API key scoped to a specific organization
- Setting up deployment credentials for CI/CD or agents
- Creating keys for triggering workflows or managing integrations within an org

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- Personal key (`ckp_`) or admin key (`cka_`) with `api-keys:manage` scope
- The target organization must already exist (see `create-organization` skill)

## Command

```bash
codika organization create-key --organization-id <id> --name <name> --scopes <scopes> [options]
```

### Required Options

| Flag | Description |
|------|-------------|
| `--organization-id <id>` | Organization ID to create the key for |
| `--name <name>` | Key name (for identification) |
| `--scopes <scopes>` | Comma-separated list of scopes (e.g. `deploy:use-case,workflows:trigger,integrations:manage`) |

### Available Scopes

`deploy:use-case`, `deploy:data-ingestion`, `projects:create`, `workflows:trigger`, `executions:read`, `instances:read`, `skills:read`, `integrations:manage`, `api-keys:manage`

### Optional Options

| Flag | Description |
|------|-------------|
| `--description <desc>` | Key description |
| `--expires-in-days <days>` | Number of days until the key expires (no expiry if omitted) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--profile <name>` | Use a specific profile instead of the active one |
| `--json` | Output result as JSON |

## Examples

**Basic key creation:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "CI Deploy Key" \
  --scopes "deploy:use-case,workflows:trigger"
```

**With description:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Agent Key" \
  --scopes "deploy:use-case,workflows:trigger,integrations:manage" \
  --description "Key for autonomous agent deployments"
```

**With expiry:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Temp Key" \
  --scopes "workflows:trigger" \
  --expires-in-days 30
```

**JSON output (for scripting):**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Script Key" \
  --scopes "deploy:use-case,workflows:trigger" \
  --json
```

## Expected Output

**Success:**

```
✓ Organization API Key Created Successfully

⚠ Save the API key below — it will not be shown again.

  API Key:     cko_xxxxxxxxxxxxxxxxxxxx
  Key Prefix:  cko_xxxxxxxx
  Key ID:      abc123-def456
  Name:        CI Deploy Key
  Scopes:      deploy:use-case, workflows:trigger
  Created:     3/30/2026
  Request ID:  req-789

  Saved as profile "org-api-key-ci-deploy-key" (now active)
```

> **Important:** The raw key is displayed only once at creation time. Store it securely — it cannot be retrieved later.

**JSON success:**

```json
{
  "success": true,
  "data": {
    "apiKey": "cko_xxxxxxxxxxxxxxxxxxxx",
    "keyPrefix": "cko_xxxxxxxx",
    "keyId": "abc123-def456",
    "name": "CI Deploy Key",
    "scopes": ["deploy:use-case", "workflows:trigger"],
    "organizationId": "xwk9CcT440Vupa8soIhY",
    "createdAt": "2026-03-30T12:00:00.000Z",
    "expiresAt": "2026-04-29T12:00:00.000Z"
  },
  "requestId": "req-789"
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` or check `codika whoami` (see `setup-codika` skill) |
| "API key does not have api-keys:manage scope" | Key missing required scope | Use a personal or admin key with `api-keys:manage` scope |
| "Organization not found" | Invalid organization ID | Verify the `--organization-id` value with `codika whoami` or the dashboard |
| "Only admins and owners can create API keys" | User lacks management role | Ensure you are an admin or owner of the organization |
| "Invalid scope" | Unrecognized scope value | Use scopes from the available scopes list above |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika login` with a valid key |

## Exit Codes

- `0` — Key created successfully
- `1` — API error (creation failed)
- `2` — CLI validation error (missing required flags)
