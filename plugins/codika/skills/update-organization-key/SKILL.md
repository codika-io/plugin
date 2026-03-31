---
name: update-organization-key
description: Updates an existing organization API key's scopes, name, or description. Use when the user needs to change key permissions, rename a key, or update a key's description without revoking and recreating it.
---

# Update Organization Key

Update the scopes, name, or description of an existing organization API key. This avoids having to revoke and recreate the key when permissions need to change.

## When to Use

- The user needs to add or change scopes on an existing key (e.g., add `projects:read`)
- Renaming a key for better identification
- Updating the description of a key
- The user had to manually edit Firestore to change scopes (this replaces that workflow)

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- An API key with `api-keys:manage` scope
- Caller must be an admin or owner in the organization
- The key ID of the target key

## Command

```bash
codika organization update-key [options]
```

### Options

| Flag | Required | Description | Default |
|------|----------|-------------|---------|
| `--key-id <id>` | Yes | Key ID to update | — |
| `--scopes <scopes>` | No | New comma-separated scopes (replaces existing) | — |
| `--name <name>` | No | New key name | — |
| `--description <desc>` | No | New key description | — |
| `--api-url <url>` | No | Override API URL | — |
| `--api-key <key>` | No | Override API key | — |
| `--profile <name>` | No | Use a specific profile instead of the active one | — |
| `--json` | No | Output result as JSON | — |

At least one of `--scopes`, `--name`, or `--description` must be provided.

### Available Scopes

`deploy:use-case`, `deploy:data-ingestion`, `projects:create`, `projects:read`, `workflows:trigger`, `executions:read`, `instances:read`, `instances:manage`, `skills:read`, `integrations:manage`, `api-keys:manage`

## Examples

**Update scopes:**

```bash
codika organization update-key \
  --key-id abc123 \
  --scopes "deploy:use-case,instances:read,projects:read"
```

**Rename a key:**

```bash
codika organization update-key \
  --key-id abc123 \
  --name "Production Deploy Key"
```

**Multiple fields at once:**

```bash
codika organization update-key \
  --key-id abc123 \
  --name "Full Access" \
  --scopes "deploy:use-case,instances:read,instances:manage,projects:read"
```

**JSON output:**

```bash
codika organization update-key --key-id abc123 --scopes deploy:use-case --json
```

## Expected Output

**Success:**

```
✓ API key updated

  Key ID:       abc123
  Name:         Production Deploy Key
  Scopes:       deploy:use-case, instances:read, projects:read
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| 401 / Unauthorized | Invalid or expired API key | Run `codika whoami`, then `codika login` |
| 403 / Missing scope | Key lacks `api-keys:manage` scope | Use a key with `api-keys:manage` scope |
| 403 / Not admin | User is not admin/owner in the org | Contact an org admin |
| 404 / Key not found | Invalid key ID | Check the key ID in the platform dashboard |
| 400 / Revoked | Cannot update a revoked key | Create a new key instead |

## Exit Codes

- `0` — Success
- `1` — API error
- `2` — CLI validation error

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file
