---
name: deploy-use-case
description: Validate and deploy a use case to the Codika platform
trigger:
  when: User asks to deploy, push, release, or publish a use case to Codika
  actions: [deploy, push, release, publish, ship]
  targets: [use case, process, workflow]
---

# Deploy Use Case

Validate and deploy a use case to the Codika platform. Automatically tracks local versions, archives deployments, and maintains version history.

## When to Use

- The user wants to deploy a use case (n8n workflows + config) to Codika
- After creating or modifying workflow files
- After a successful `verify-use-case` check

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)
- A use case folder with `config.ts` and `workflows/` directory
- A project to deploy to (either via `project.json` or `PROJECT_ID` in `config.ts`)

## Setting Up a New Use Case

The full flow from scratch:

```bash
cd my-use-case

# 1. Create a project and write project.json in the current folder
codika-helper project create --name "My Project" --path .

# 2. Write config.ts with getConfiguration() (no PROJECT_ID needed — it's in project.json)
# 3. Create workflows in workflows/

# 4. Validate
codika-helper verify use-case .

# 5. Deploy
codika-helper deploy use-case .
```

The `--path .` flag tells the CLI to write `project.json` (containing the project ID) into the current directory. Without `--path`, the project ID is only printed — not saved.

## Use Case Folder Structure

```
my-use-case/
  project.json        # { "projectId": "abc123", "organizationId": "org-456" }
  config.ts           # getConfiguration(), optionally WORKFLOW_FILES
  workflows/
    main-workflow.json
    sub-workflow.json
  version.json        # Auto-managed — tracks local semver (X.Y.Z)
  deployments/        # Auto-managed — deployment archives and version history
    {projectId}/
      project-info.json          # Version mapping history
      process/
        {apiVersion}/
          deployment-info.json   # Deployment metadata
          config-snapshot.json   # Config at time of deploy
          workflows/*.json       # Workflow snapshots
```

### Project ID Resolution

The project ID (deployment target) is resolved in this order:

1. `--project-id` flag (highest priority)
2. `project.json` file in the use case folder

### API Key Resolution (Org-Aware)

When `project.json` contains an `organizationId`, the CLI automatically selects the profile matching that organization — even if a different profile is currently active. This means you can work across multiple organizations without manually switching profiles.

1. `--api-key` flag (highest priority)
2. `CODIKA_API_KEY` environment variable
3. Profile matching `organizationId` from `project.json`
4. Active profile in config file

When auto-selecting, the CLI prints: `Using profile "acme-corp" (matches project organization)`

## Recommended Flow

Always validate before deploying:

```bash
# Step 1: Validate
codika-helper verify use-case <path>

# Step 2: Deploy (only if validation passes)
codika-helper deploy use-case <path>
```

## Deploy Command

```bash
codika-helper deploy use-case <path> [options]
```

### Version Flags

| Flag | API Strategy | Local Bump | Description |
|------|-------------|------------|-------------|
| _(default)_ / `--patch` | `minor_bump` | patch | Patch bump (1.0.0 → 1.0.1) |
| `--minor` | `minor_bump` | minor | Minor bump (1.0.1 → 1.1.0) |
| `--major` | `major_bump` | major | Major bump (1.1.0 → 2.0.0) |
| `--version <X.Y>` | `explicit` | patch | Deploy to explicit API version |

### Other Options

| Flag | Description |
|------|-------------|
| `--project-id <id>` | Override project ID (skips `project.json` and `config.ts`) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--additional-file <abs:rel>` | Add extra file (repeatable). Format: `absolutePath:relativePath` |
| `--json` | Output result as JSON |

## Examples

**Standard deployment (patch bump):**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case
```

**Minor version bump:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --minor
```

**Major version bump:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --major
```

**Deploy to explicit API version:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --version 2.0
```

**With additional metadata file:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case \
  --additional-file "/path/to/prd.md:docs/prd.md"
```

**JSON output for scripting:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --json
```

## Expected Output

**Success:**

```
✓ Deployment Successful

  Template ID:   tmpl-abc123
  API Version:   1.0
  Local Version: 1.0.4
  Process ID:    proc-def456
  Status:        deployed

  Workflows:
    - main-workflow: active (n8n: 42)
    - sub-workflow: active (n8n: 43)
```

## What Happens on Deploy

1. **version.json** is read (defaults to `1.0.0` if missing)
2. Local version is bumped based on flags (`--patch`, `--minor`, `--major`)
3. API version strategy is resolved and sent to the platform
4. On success:
   - `version.json` is updated with the new local version
   - Deployment is archived in `deployments/{projectId}/process/{apiVersion}/`
   - `project-info.json` is updated with the API→local version mapping
   - `project.json` is updated with `devProcessInstanceId`

## Version Tracking

### version.json

Auto-managed file tracking the local semver version:

```json
{ "version": "1.0.4" }
```

### project-info.json

Stores version mapping history per project:

```json
{
  "projectId": "abc123",
  "createdAt": "2025-01-15T10:00:00.000Z",
  "lastDeployedAt": "2025-01-20T14:30:00.000Z",
  "versionMappings": {
    "process": {
      "1.0": { "useCaseVersion": "1.0.4", "deployedAt": "2025-01-20T14:30:00.000Z" },
      "1.1": { "useCaseVersion": "1.1.0", "deployedAt": "2025-01-18T09:00:00.000Z" }
    },
    "dataIngestion": {}
  }
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` or `codika-helper use <profile>` (see `setup-codika` skill) |
| "Use case path does not exist" | Wrong path | Check the path points to a folder with `config.ts` |
| Validation errors in response | Workflow issues | Run `codika-helper verify use-case <path> --fix` first (see `verify-use-case` skill) |
| "No project ID found" | No project.json or PROJECT_ID | Run `codika-helper project create --name "..." --path <path>` or add `project.json` |
| "config.ts must export getConfiguration" | Missing export | Ensure `config.ts` exports a `getConfiguration()` function |
| 401 / Unauthorized | Invalid API key | Re-run `codika-helper login` or check `codika-helper whoami` |
| "Invalid version format" | Bad `--version` value | Use `X.Y` format (e.g., `--version 1.0`) |

## Exit Codes

- `0` — Deployment successful
- `1` — Deployment failed (API error or workflow-level failure)
- `2` — CLI validation error (missing path, invalid flags)
