---
name: deploy-documents
description: Deploys use case documentation (stage markdown files) to the Codika platform. Use when the user asks to upload, deploy, or push documents for a use case. Auto-discovers stage files (1_*.md through 4_*.md) from the documents/ folder.
---

# Deploy Documents

Upload use case documentation (stage 1-4 markdown files) to the Codika platform. Auto-discovers document files from the use case's `documents/` folder.

## When to Use

- The user wants to upload or deploy use case documents
- After creating or updating stage markdown files in `documents/`
- When deploying documentation alongside a use case

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)
- A use case folder with a `documents/` subfolder containing stage markdown files
- A project to deploy to (via `project.json` or `PROJECT_ID` in `config.ts`)

## Document Folder Structure

```
my-use-case/
  project.json        # { "projectId": "abc123" }
  config.ts           # Optional — PROJECT_ID fallback
  documents/
    1_business_requirements.md
    2_solution_architecture.md
    3_detailed_design.md
    4_implementation_plan.md
```

### File Naming Convention

Files must follow the pattern `{stage}_{name}.md`:

| Pattern | Stage | Example Title |
|---------|-------|---------------|
| `1_*.md` | 1 | "Business Requirements" |
| `2_*.md` | 2 | "Solution Architecture" |
| `3_*.md` | 3 | "Detailed Design" |
| `4_*.md` | 4 | "Implementation Plan" |

Not all stages are required — the CLI uploads whichever stages exist.

## Deploy Command

```bash
codika-helper deploy documents <path> [options]
```

### Options

| Flag | Description |
|------|-------------|
| `--project-id <id>` | Override project ID |
| `--project-file <path>` | Custom project file path (default: `project.json`) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

### Project ID Resolution

1. `--project-id` flag (highest priority)
2. `project.json` file (default or `--project-file`)
3. `PROJECT_ID` export in `config.ts` (fallback)

## Examples

**Standard deployment:**

```bash
codika-helper deploy documents ./use-cases/marketplace/my-use-case
```

**With explicit project ID:**

```bash
codika-helper deploy documents ./use-cases/marketplace/my-use-case --project-id proj-abc123
```

**JSON output:**

```bash
codika-helper deploy documents ./use-cases/marketplace/my-use-case --json
```

## Expected Output

**Success:**

```
Reading document files...
  Stage 1: 1_business_requirements.md -> "Business Requirements" (2450 chars, ~380 words)
  Stage 2: 2_solution_architecture.md -> "Solution Architecture" (5120 chars, ~790 words)

Uploading 2 document(s)...

✓ Documents Deployed Successfully

  Stage 1: v1.0.0 (doc-abc123)
  Stage 2: v1.0.0 (doc-def456)

  Request ID: req-789
```

## How It Works

1. Scans `documents/` folder for files matching `{1,2,3,4}_*.md`
2. For each file:
   - Derives title from filename (e.g., `1_business_requirements.md` -> "Business Requirements")
   - Extracts summary from the first non-heading paragraph (max 200 chars)
   - Validates the file is not empty
3. Resolves project ID from `--project-id` > `project.json` > `config.ts`
4. Uploads all documents to the platform in a single API call
5. Reports per-stage results (document ID, version, status)

## Document Versioning

Documents are versioned on the platform:
- First upload for a stage creates version `1.0.0`
- Subsequent uploads increment the minor version (e.g., `1.0.0` -> `1.1.0`)
- Each document version is tracked in the platform's `version_history` collection

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "No documents/ folder found" | Missing folder | Create a `documents/` folder with stage files |
| "No stage files found" | No matching files | Add files named `1_*.md`, `2_*.md`, etc. |
| "file is empty" | Empty document | Add content to the empty file |
| "Could not determine project ID" | No project.json or config.ts | Add `project.json` or pass `--project-id` |

## Exit Codes

- `0` — Documents deployed successfully
- `1` — Deployment failed
