---
name: deploy-data-ingestion
description: Deploys a process-level data ingestion configuration to the Codika platform. Use when the user asks to deploy, push, or update data ingestion workflows. Handles version bumping, deployment archiving, and project.json tracking. Data ingestion is separate from use case deployment and does NOT trigger "update available" notifications.
---

# Deploy Data Ingestion

Deploy a process-level data ingestion configuration to the Codika platform. Data ingestion deployments are independent from use case deployments — updating data ingestion does NOT trigger "update available" notifications to users.

## When to Use

- The user wants to deploy a data ingestion workflow (document embedding pipeline)
- After creating or modifying data ingestion configuration in `config.ts`
- The use case has a `getDataIngestionConfig()` export in `config.ts`

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A use case folder with `config.ts` exporting `getDataIngestionConfig()` and a `data-ingestion/` folder with exactly one workflow JSON file
- A project to deploy to (via `project.json`)

## Use Case Folder Structure

```
my-use-case/
  project.json        # { "projectId": "abc123", "organizationId": "org-456" }
  config.ts           # getDataIngestionConfig()
  data-ingestion/
    <workflow>.json   # Exactly one workflow JSON file (auto-discovered)
  version.json        # Auto-managed — tracks dataIngestionVersion separately
  deployments/        # Auto-managed
    {projectId}/
      project-info.json
      data-ingestion/
        {apiVersion}/
          deployment-info.json
          config-snapshot.json
          <workflow>.json
```

## Deploy Command

```bash
codika deploy process-data-ingestion <path> [options]
```

### Version Flags

| Flag | API Strategy | Local Bump | Description |
|------|-------------|------------|-------------|
| _(default)_ / `--patch` | `minor_bump` | patch | Patch bump (1.0.0 → 1.0.1) |
| `--minor` | `minor_bump` | minor | Minor bump (1.0.1 → 1.1.0) |
| `--major` | `major_bump` | major | Major bump (1.1.0 → 2.0.0) |
| `--target-version <X.Y>` | `explicit` | patch | Deploy to explicit API version |

### Options

| Flag | Description |
|------|-------------|
| `--patch` | Patch version bump (default) |
| `--minor` | Minor version bump |
| `--major` | Major version bump |
| `--target-version <version>` | Deploy to explicit API version (e.g., `3.0`) |
| `--project-id <id>` | Override project ID |
| `--project-file <path>` | Custom project file path (default: `project.json`) |
| `--profile <name>` | Use a specific profile instead of the active one |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

## Examples

**Standard deployment (minor bump):**

```bash
codika deploy process-data-ingestion ./use-cases/marketplace/my-use-case
```

**Major version bump:**

```bash
codika deploy process-data-ingestion ./use-cases/marketplace/my-use-case --major
```

**Explicit version:**

```bash
codika deploy process-data-ingestion ./use-cases/marketplace/my-use-case --target-version 3.0
```

**JSON output:**

```bash
codika deploy process-data-ingestion ./use-cases/marketplace/my-use-case --json
```

## Expected Output

**Success:**

```
✓ Data Ingestion Deployment Successful

  Data Ingestion ID:  di-abc123
  API Version:        1.2
  Local Version:      1.0.0 -> 1.1.0
  Project ID:         proj-456
  Status:             active
  Webhook (embed):    https://n8n.example.com/webhook/embed-xxx
  Webhook (delete):   https://n8n.example.com/webhook/delete-xxx
```

## What Happens on Deploy

1. `config.ts` is loaded — `getDataIngestionConfig()` is read and the workflow JSON is auto-discovered from `data-ingestion/`
2. Project ID is resolved from `--project-id` > `project.json`
3. Configuration is sent to the platform
4. On success:
   - `version.json` is updated with the new `dataIngestionVersion` (separate from process version)
   - Deployment is archived in `deployments/{projectId}/data-ingestion/{apiVersion}/`
   - `project-info.json` is updated with the version mapping in `versionMappings.dataIngestion`
   - `project.json` is updated with `dataIngestionDeployments` map (version -> {dataIngestionId, createdAt, webhookUrls})

## Version Tracking

Data ingestion has its **own version line**, separate from use case deployments:

### version.json

```json
{
  "version": "1.0.4",
  "dataIngestionVersion": "1.1.0"
}
```

### project.json

```json
{
  "projectId": "abc123",
  "deployments": { ... },
  "dataIngestionDeployments": {
    "1.0": {
      "dataIngestionId": "di-abc123",
      "createdAt": "2025-01-20T14:30:00.000Z",
      "webhookUrls": {
        "embed": "https://n8n.example.com/webhook/embed-xxx",
        "delete": "https://n8n.example.com/webhook/delete-xxx"
      }
    }
  }
}
```

## Key Differences from Use Case Deployment

| Aspect | Use Case Deploy | Data Ingestion Deploy |
|--------|----------------|----------------------|
| Command | `deploy use-case` | `deploy process-data-ingestion` |
| Scope | Per-user instance (dev/prod) | Per-process (shared by all users) |
| Versioning | Semantic (X.Y.Z) + API (X.Y) | Simple (X.Y) + local (X.Y.Z) |
| Notifications | Triggers "update available" | Does NOT trigger notifications |
| Version flags | `--patch`, `--minor`, `--major`, `--target-version` | `--patch`, `--minor`, `--major`, `--target-version` |
| Tracking key | `deployments` in project.json | `dataIngestionDeployments` in project.json |

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "config.ts must export getDataIngestionConfig" | Missing export | Add `getDataIngestionConfig()` to config.ts |
| "No data-ingestion/ folder found" | Missing folder | Create `data-ingestion/` with one workflow JSON file |
| "Multiple JSON files found" | Too many files | Keep exactly one `.json` file in `data-ingestion/` |
| "No project ID found" | No project.json | Run `codika project create --name "..." --path <path>` |

## Exit Codes

- `0` — Deployment successful
- `1` — Deployment failed
