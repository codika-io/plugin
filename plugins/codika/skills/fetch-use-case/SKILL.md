---
name: fetch-use-case
description: Fetch and download a deployed Codika use case from the platform
trigger:
  when: User asks to download, fetch, pull, or restore a deployed use case
  actions: [fetch, download, pull, restore, get]
  targets: [use case, deployed use case, metadata documents]
---

# Fetch Use Case

Fetch a deployed use case from the Codika platform with all its metadata documents.

## When to Use

- The user wants to download a deployed use case from the platform
- Restoring a use case that was previously deployed
- Inspecting what documents are stored for a given project
- Pulling the latest deployed version of a use case locally

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)
- A project ID for the deployed use case

## Command

```bash
codika-helper get use-case <projectId> [outputPath] [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<projectId>` | Yes | The project ID of the deployed use case |
| `[outputPath]` | No | Directory to write files to (defaults to `./<projectId>`) |

### Options

| Flag | Description |
|------|-------------|
| `--version <version>` | Fetch a specific version instead of the latest |
| `--list` | List documents only without downloading |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

## Examples

**Fetch the latest version:**

```bash
codika-helper get use-case my-project-id ./restored
```

**Fetch a specific version:**

```bash
codika-helper get use-case my-project-id ./restored --version 1.0
```

**List documents only (no download):**

```bash
codika-helper get use-case my-project-id --list
```

**JSON output for scripting:**

```bash
codika-helper get use-case my-project-id --list --json
```

## Expected Output

**Download:**

```
Fetching use case my-project-id...

  Downloaded files:
    config.ts
    workflows/main-workflow.json
    workflows/sub-workflow.json
    version.json

  Total:       4 files
  Output:      ./restored
```

**List mode:**

```
Documents for my-project-id:

  config.ts
  workflows/main-workflow.json
  workflows/sub-workflow.json
  version.json

  Total: 4 documents
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "Project not found" | Invalid or non-existent project ID | Verify the project ID from the Codika dashboard or `project.json` |
| "Invalid version format" | Malformed `--version` value | Use semantic version format (e.g., `1.0`, `2.1.0`) |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika-helper login` |

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika-helper whoami` to check the current identity, or `codika-helper use <profile>` to switch profiles.
