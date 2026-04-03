---
name: manage-integrations
description: Manages integrations (API keys, credentials) for Codika organizations and process instances via the CLI. Supports create, list, and delete operations for all non-OAuth integrations.
---

## When to Use

Use this skill when the user asks to:
- Set up, configure, or connect an integration (e.g., "add my OpenAI API key", "connect Supabase")
- List configured integrations for an organization or process instance
- Delete or disconnect an integration
- Check which integrations are connected

## Prerequisites

- `codika` CLI installed and authenticated (`codika login`)
- API key with `integrations:manage` scope
- For process instance integrations: a deployed use case with `project.json`

## Commands

### Set (Create/Update) an Integration

```bash
# Simple API key integration
codika integration set openai --secret OPENAI_API_KEY=sk-xxx

# Multi-field integration
codika integration set supabase \
  --secret SUPABASE_HOST=https://xxx.supabase.co \
  --secret SUPABASE_SERVICE_ROLE_KEY=eyJ... \
  --process-instance-id <id>

# From JSON (best for agents)
codika integration set openai --secrets '{"OPENAI_API_KEY":"sk-xxx"}' --json

# From file (best for CI/CD)
codika integration set postgres --secrets-file ./secrets.json --process-instance-id <id>

# Force overwrite existing
codika integration set openai --secret OPENAI_API_KEY=sk-new --force

# Custom integration (cstm_* prefix) — auto-extracts schema from config.ts
codika integration set cstm_weather_api \
  --secret API_KEY=xxx \
  --path ./my-use-case \
  --json

# Custom integration with explicit schema file (fallback)
codika integration set cstm_weather_api \
  --secret API_KEY=xxx \
  --process-instance-id <id> \
  --custom-schema-file ./custom-schema.json
```

#### Secret Input Methods (merge priority: file < JSON < flags)
1. `--secrets-file path.json` — lowest priority
2. `--secrets '{"KEY":"VALUE"}'` — middle priority
3. `--secret KEY=VALUE` (repeatable) — highest priority

#### Custom Integrations (cstm_* prefix)

Custom integrations require a schema that defines their fields and n8n credential mapping. The CLI auto-extracts this schema from the `customIntegrations` array in `config.ts` when `--path` is provided (or when run from a use case folder).

**Recommended approach — use `--path`:**
```bash
codika integration set cstm_acme_crm \
  --secret API_KEY=xxx \
  --path ./my-use-case \
  --json
```

The CLI reads `config.ts`, finds the `cstm_acme_crm` entry in `customIntegrations`, and uses it as the schema automatically. No separate schema file needed.

**Fallback — explicit schema file** (when config.ts is not available):
```bash
codika integration set cstm_acme_crm \
  --secret API_KEY=xxx \
  --custom-schema-file ./schema.json \
  --process-instance-id <id>
```

### List Integrations

```bash
# List org-level integrations
codika integration list

# List process instance integrations
codika integration list --context-type process_instance --path .

# JSON output
codika integration list --json
```

### Delete an Integration

```bash
# Check dependencies first (no --confirm)
codika integration delete openai

# Delete immediately
codika integration delete openai --confirm

# Delete process instance integration
codika integration delete supabase --confirm --process-instance-id <id>
```

## Context Type Resolution

The CLI automatically resolves the context type from the integration registry:

| Integration | Default Context | Example |
|------------|----------------|---------|
| `openai`, `anthropic`, etc. | `organization` | Shared across org |
| `supabase`, `postgres`, `telegram` | `process_instance` | Per deployment |
| `cstm_*` (custom) | `process_instance` | Per deployment |

Override with `--context-type <type>` when needed.

## OAuth Integrations (Dashboard Only)

These integrations require OAuth and cannot be configured from the CLI:
- **Google**: gmail, calendar, drive, sheets, docs, slides, tasks, contacts
- **Microsoft**: teams, outlook, sharepoint, to_do, excel, onedrive
- **Other**: notion, calendly, slack, shopify

The CLI will print the dashboard URL when you try to set an OAuth integration.

## CLI-Configurable Integrations

| ID | Name | Context | Fields |
|----|------|---------|--------|
| `openai` | OpenAI | org | `OPENAI_API_KEY` |
| `anthropic` | Anthropic | org | `ANTHROPIC_API_KEY` |
| `google_gemini` | Google Gemini | org | `GOOGLE_GEMINI_API_KEY` |
| `mistral` | Mistral | org | `MISTRAL_API_KEY` |
| `deep_seek` | DeepSeek | org | `DEEP_SEEK_API_KEY` |
| `cohere` | Cohere | org | `COHERE_API_KEY` |
| `x_ai` | xAI | org | `X_AI_API_KEY` |
| `open_router` | OpenRouter | org | `OPEN_ROUTER_API_KEY` |
| `tavily` | Tavily | org | `TAVILY_API_KEY` |
| `pinecone` | Pinecone | org | `PINECONE_API_KEY` |
| `folk` | Folk | org | `FOLK_API_KEY` |
| `odoo` | Odoo | org | `ODOO_URL`, `ODOO_DATABASE`, `ODOO_USERNAME`, `ODOO_PASSWORD` |
| `twilio` | Twilio | org | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` |
| `pipedrive` | Pipedrive | org | `PIPEDRIVE_API_TOKEN` |
| `supabase` | Supabase | instance | `SUPABASE_HOST`, `SUPABASE_SERVICE_ROLE_KEY` |
| `postgres` | PostgreSQL | instance | `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DATABASE`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
| `telegram` | Telegram | instance | `TELEGRAM_ACCESS_TOKEN` |

## Agent Usage Pattern

When an agent needs to configure integrations after deploying a use case:

```bash
# 1. Deploy the use case
codika deploy use-case ./my-use-case --json

# 2. Set required integrations (built-in)
codika integration set openai --secrets '{"OPENAI_API_KEY":"sk-xxx"}' --json
codika integration set supabase \
  --secrets '{"SUPABASE_HOST":"https://xxx.supabase.co","SUPABASE_SERVICE_ROLE_KEY":"eyJ..."}' \
  --path ./my-use-case --json

# 3. Set custom integrations (schema auto-extracted from config.ts)
codika integration set cstm_acme_crm \
  --secrets '{"API_KEY":"acme_sk_xxx"}' \
  --path ./my-use-case --json

# 4. Redeploy to activate
codika redeploy --path ./my-use-case --force --json
```

## Error Handling

| Exit Code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | API error (server error, network failure) |
| 2 | Validation error (missing fields, OAuth integration, missing confirmation) |

## Authentication

Uses the standard Codika CLI authentication chain:
1. `--api-key` flag (highest priority)
2. `CODIKA_API_KEY` environment variable
3. Active profile from `codika login`

The API key must have the `integrations:manage` scope.
