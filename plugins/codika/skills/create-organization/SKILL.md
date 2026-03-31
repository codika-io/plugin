---
name: create-organization
description: Creates a new organization on the Codika platform. Use when the user asks to create an organization, set up a new org, initialize an organization, or needs a new workspace for a team or client. Requires the organizations:create scope.
---

# Create Organization

Create a new organization on the Codika platform.

## When to Use

- The user wants to create a new organization
- Setting up a workspace for a new team or client
- Onboarding a new customer that needs their own org

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- Personal key (`ckp_`) or admin key (`cka_`) with `organizations:create` scope

> **Note:** Organization keys (`cko_`) cannot create organizations — they are scoped to a single org and lack the `organizations:create` scope. Use a personal key or admin key instead.

## Command

```bash
codika organization create --name "My Organization" [options]
```

### Required Options

| Flag | Description |
|------|-------------|
| `--name <name>` | Organization name (2-100 characters) |

### Optional Options

| Flag | Description |
|------|-------------|
| `--description <desc>` | Organization description |
| `--size <size>` | Organization size (`solo`, `2-10`, `11-50`, `51-200`, `201-1000`, `1000+`) |
| `--logo <path>` | Path to a logo image file (JPEG, PNG, or WebP, max 5MB). Uploaded to platform storage. |
| `--n8n-base-url <url>` | Self-hosted n8n instance URL (must be paired with `--n8n-api-key`) |
| `--n8n-api-key <key>` | Self-hosted n8n API key (must be paired with `--n8n-base-url`) |
| `--store-credential-copy` | Store encrypted credential backup in Codika (only with self-hosted n8n) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--profile <name>` | Use a specific profile instead of the active one |
| `--json` | Output result as JSON |

## Examples

**Basic organization creation:**

```bash
codika organization create --name "Acme Corp"
```

**With description and size:**

```bash
codika organization create \
  --name "Acme Corp" \
  --description "Main workspace for Acme team" \
  --size "11-50"
```

**With a logo:**

```bash
codika organization create \
  --name "Acme Corp" \
  --logo ./acme-logo.png
```

**With self-hosted n8n:**

```bash
codika organization create \
  --name "Enterprise Client" \
  --n8n-base-url "https://n8n.enterprise.com" \
  --n8n-api-key "n8n_api_xxxxx"
```

**With self-hosted n8n and credential backup:**

```bash
codika organization create \
  --name "Enterprise Client" \
  --n8n-base-url "https://n8n.enterprise.com" \
  --n8n-api-key "n8n_api_xxxxx" \
  --store-credential-copy
```

**JSON output (for scripting):**

```bash
codika organization create --name "Test Org" --json
```

## Expected Output

**Success:**

```
✓ Organization Created Successfully

  Organization ID: abc123-def456
  Request ID:      req-789
```

**JSON success:**

```json
{
  "success": true,
  "data": {
    "organizationId": "abc123-def456"
  },
  "requestId": "req-789"
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` or check `codika whoami` (see `setup-codika` skill) |
| "API key does not have organizations:create scope" | Key missing required scope | Create a new API key with the `organizations:create` scope in the dashboard |
| "Organization keys cannot create organizations" | Using an org key (`cko_`) | Switch to a personal key (`ckp_`) or admin key (`cka_`) with `organizations:create` scope |
| "Insufficient scopes" | Personal key missing required scope | Re-create your personal key with the `organizations:create` scope in the dashboard |
| "Name must be at least 2 characters" | Name too short | Provide a name with at least 2 characters |
| "Both n8nBaseUrl and n8nApiKey must be provided together" | Only one n8n field provided | Provide both `--n8n-base-url` and `--n8n-api-key` together |
| "Invalid n8n credentials" | n8n URL or API key is wrong | Verify your n8n instance URL is reachable and the API key is correct |
| "Maximum organization limit reached" | User has 50+ organizations | Archive unused organizations first |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika login` with a valid key |

## Exit Codes

- `0` — Organization created successfully
- `1` — API error (creation failed)
- `2` — CLI validation error (missing required flags)
