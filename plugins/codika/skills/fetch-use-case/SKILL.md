---
name: fetch-use-case
description: Fetches and downloads a deployed Codika use case from the platform with all metadata documents. Use when the user asks to download, fetch, pull, or restore a deployed use case. Supports fetching specific versions and listing documents without downloading.
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

## Resolving the Project ID

The CLI requires a **project ID**, not a folder path. If the user provides a use case folder path instead, read the project file from that folder to get the `projectId`. The project file is `project.json` by default, but the user may have used `--project-file` during init or deploy to store it under a different name:

```bash
# Read the projectId from the use case folder (default project.json)
cat <use-case-path>/project.json
# Or from a custom project file
cat <use-case-path>/custom-project.json
# Then use the projectId value in the command below
```

## Command

```bash
codika-helper get use-case <projectId> [outputPath] [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<projectId>` | Yes | The project ID of the deployed use case (from `project.json`) |
| `[outputPath]` | No | Directory to write files to (defaults to `./<projectId>`) |

### Options

| Flag | Description |
|------|-------------|
| `--version <version>` | Fetch a specific version in `X.Y` format (fetches latest if omitted) |
| `--with-data-ingestion` | Include data ingestion workflow (default: true) |
| `--no-data-ingestion` | Exclude data ingestion workflow |
| `--di-version <version>` | Data ingestion version in `X.Y` format (latest if omitted) |
| `--list` | List documents only without downloading |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

**Note:** Use `--version` (not `--cli-version`). The global CLI version flag is `-V` / `--cli-version`, so `--version` on subcommands is reserved for specifying the use case version.

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

**Fetch with a specific data ingestion version:**

```bash
codika-helper get use-case my-project-id ./restored --di-version 1.2
```

**Fetch without data ingestion:**

```bash
codika-helper get use-case my-project-id ./restored --no-data-ingestion
```

**List with JSON output:**

```bash
codika-helper get use-case my-project-id --list --json
```

**Check CLI version (not use case version):**

```bash
codika-helper -V
```

## Expected Output

**Download:**

```
Fetching use case my-project-id...
  Output:  ./restored

  ✓ config.ts
  ✓ workflows/main-workflow.json
  ✓ workflows/sub-workflow.json
  ✓ data-ingestion/embedding-ingestion.json

✓ Use Case Downloaded Successfully

  Project:  my-project-id
  Version:  1.0
  DI Ver:   1.2
  Output:   ./restored
  Files:    4 file(s) downloaded
```

**List mode:**

```
✓ Found 4 document(s)

  Project:      my-project-id
  Version:      1.0
  DI Version:   1.2
  Organization: org-456

  Documents:
    config.ts  (8.1 KB, text/typescript)
    workflows/main-workflow.json  (5.8 KB, application/json)
    workflows/sub-workflow.json  (2.0 KB, application/json)
    data-ingestion/embedding-ingestion.json  (3.2 KB, application/json)
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "Project not found" | Invalid or non-existent project ID | Verify the project ID from the Codika dashboard or `project.json` |
| "Invalid version format" | Malformed `--version` value | Use `X.Y` format (e.g., `1.0`, `2.1`) |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika-helper login` |

## Authentication

The CLI handles authentication automatically. The API key is resolved in this order:

1. `--api-key` flag
2. `CODIKA_API_KEY` environment variable
3. Active profile in config file

Run `codika-helper whoami` to check the current identity, or `codika-helper use <profile>` to switch profiles.
