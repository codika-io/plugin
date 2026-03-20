---
name: init-use-case
description: Scaffolds a new Codika use case folder with config.ts, workflow JSON templates, and version tracking. Use when the user asks to create, initialize, scaffold, or bootstrap a new use case from scratch. Generates HTTP-triggered, schedule-triggered, and sub-workflow templates.
---

# Init Use Case

Scaffold a new Codika use case folder with a ready-to-verify, ready-to-deploy template including `config.ts`, workflow JSON files, and version tracking.

## When to Use

- The user wants to start a new use case from scratch
- Creating the initial folder structure for an n8n workflow project
- Bootstrapping a new automation with the correct Codika patterns

## Prerequisites

- `codika` CLI installed (`npm install -g codika`)
- For project creation: authenticated via `codika login` (see `setup-codika` skill)
- Project creation is optional — scaffolding works without authentication

## Command

```bash
codika init <path> [options]
```

### Arguments

| Argument | Required | Description                         |
| -------- | -------- | ----------------------------------- |
| `<path>` | Yes      | Directory to create the use case in |

### Options

| Flag                   | Description                           | Default            |
| ---------------------- | ------------------------------------- | ------------------ |
| `--name <name>`        | Use case display name                 | Interactive prompt |
| `--description <desc>` | Use case description                  | Auto-generated     |
| `--icon <icon>`        | Lucide icon name                      | `Workflow`         |
| `--no-project`         | Skip project creation on the platform | Creates project    |
| `--project-id <id>`    | Use existing project ID (no API call) | —                  |
| `--no-install`         | Skip npm install after scaffolding    | Runs npm install   |
| `--project-file <path>` | Custom filename for the project file | `project.json`     |
| `--api-url <url>`      | Override API URL                      | —                  |
| `--api-key <key>`      | Override API key                      | —                  |
| `--json`               | Output result as JSON                 | —                  |

## Examples

**Scaffold with project creation (default):**

```bash
codika init ./my-use-case --name "My Automation"
```

**Scaffold without project creation:**

```bash
codika init ./my-use-case --name "My Automation" --no-project
```

**Scaffold with existing project ID:**

```bash
codika init ./my-use-case --name "My Automation" --project-id abc123
```

**JSON output for scripting:**

```bash
codika init ./my-use-case --name "My Automation" --no-project --json
```

## Generated Files

```
my-use-case/
  config.ts                         # Deployment configuration (3 workflows)
  version.json                      # Version tracking (starts at 1.0.0)
  project.json                      # Project ID (only if project created or --project-id). Filename changes if --project-file is used.
  package.json                      # Dependencies (codika)
  tsconfig.json                     # TypeScript config for IDE support
  .gitignore                        # Ignores node_modules/
  node_modules/                     # Installed after scaffolding (unless --no-install)
  workflows/
    main-workflow.json               # HTTP-triggered parent workflow
    scheduled-report.json            # Schedule-triggered workflow (Monday 9 AM)
    text-processor.json              # Sub-workflow (called by main workflow)
```

### Multiple use cases in a shared workspace

When the user manages multiple use cases, they can share a single `package.json` at the parent level instead of having one per use case. Set it up once, then scaffold use cases inside:

```bash
mkdir my-automations && cd my-automations
npm init -y && npm install codika
echo 'node_modules/' > .gitignore
codika init ./email-automation --name "Email Automation"
codika init ./report-generator --name "Report Generator"
```

```
my-automations/
  package.json              # Shared
  node_modules/             # Single copy
  email-automation/
    config.ts
    workflows/
  report-generator/
    config.ts
    workflows/
```

The CLI detects the existing `codika` dependency in a parent `package.json` and skips creating per-use-case dependency files. Each use case is still independently deployable.

### What the Template Demonstrates

- **HTTP trigger**: Webhook-based workflow with input validation and sub-workflow delegation
- **Schedule trigger**: Cron-based workflow with manual trigger webhook for on-demand execution
- **Sub-workflow**: Internal workflow called by the parent, with proper `SUBWKFL` placeholder
- **Codika patterns**: Init/Submit/Report nodes, placeholder system, error handling

## Expected Output

**Success:**

```
Creating use case "My Automation"...

  Scaffolded files:
    ✓ config.ts
    ✓ workflows/main-workflow.json
    ✓ workflows/scheduled-report.json
    ✓ workflows/text-processor.json
    ✓ version.json
    ✓ project.json
    ✓ package.json
    ✓ .gitignore
    ✓ tsconfig.json

  Project ID: abc123

  ✓ Dependencies installed

✓ Done! Next steps:

  1. Edit the workflow JSON files in ./my-use-case/workflows/
  2. Update config.ts with your schemas and metadata
  3. Run: codika verify use-case ./my-use-case
  4. Run: codika deploy use-case ./my-use-case
```

## Next Steps After Init

```bash
# 1. Validate the use case
codika verify use-case ./my-use-case

# 2. Deploy to the platform
codika deploy use-case ./my-use-case
```

If you used `--no-project`, create a project before deploying:

```bash
codika project create --name "My Automation" --path ./my-use-case
```

## Error Handling

| Error                                         | Cause                                     | Fix                                                        |
| --------------------------------------------- | ----------------------------------------- | ---------------------------------------------------------- |
| "Directory already contains a config.ts"      | Path already has a use case               | Use a different path or delete the existing folder         |
| "Use case name is required"                   | No `--name` flag and non-interactive mode | Provide `--name` flag                                      |
| "No API key found, skipping project creation" | Not authenticated (warning, not error)    | Run `codika login` first, or use `--no-project`     |
| Project creation failed                       | Invalid API key or network issue          | Check `codika whoami`, re-run `codika login` |

## Exit Codes

- `0` — Use case scaffolded successfully
- `1` — Runtime error (project creation failure is a warning, not exit 1)
- `2` — CLI validation error (missing path, folder already exists)
