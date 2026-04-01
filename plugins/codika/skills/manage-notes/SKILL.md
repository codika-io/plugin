---
name: manage-notes
description: Manage project notes — a lightweight versioned knowledge base per project. Use when you need to persist context across sessions (project briefs, known issues, changelogs, debugging logs), read previous context for a project, or track what changed over time.
---

# Manage Project Notes

Create, update, and read versioned project documents that persist across sessions. Project notes are a lightweight knowledge base — each document type (e.g. `brief`, `known-issues`, `changelog`) is independently versioned with full history.

## When to Use

- You need to persist context about a project that should survive across sessions
- You want to record decisions, known issues, or deployment notes for future reference
- You need to read what a previous agent session documented about a project
- You want to track changes over time (every update creates an immutable version)

## How Agents Should Use This

Project notes are different from metadata documents (which are frozen deployment snapshots). Project notes are **living knowledge** — updated any time, independently versioned, meant to accumulate context over the project's lifetime.

### At the start of a session: read existing context

Before working on a project, always check if previous sessions left documentation:

```bash
codika notes list $PROJECT_ID
codika notes get $PROJECT_ID --type brief
codika notes get $PROJECT_ID --type known-issues
```

This gives you instant context — what the project does, what decisions were made, what's broken — without reverse-engineering from workflow JSON.

### After building or deploying: record what you did

```bash
codika notes upsert $PROJECT_ID --type brief \
  --content "Invoice processor for SMBs. HTTP trigger, Anthropic extraction, JSON output." \
  --summary "Initial brief after first deployment"

codika notes upsert $PROJECT_ID --type architecture \
  --content "Main workflow: HTTP trigger -> PDF parse -> Claude extraction -> Submit Result. Sub-workflow: email-parser." \
  --summary "Architecture decisions"
```

### After debugging: document the fix

```bash
codika notes upsert $PROJECT_ID --type debugging-log \
  --content "Timeout on PDFs > 5MB. Root cause: Anthropic token limit. Fix: added chunking." \
  --summary "Diagnosed and fixed PDF timeout issue"
```

### After each deployment: track changes

```bash
codika notes upsert $PROJECT_ID --type changelog \
  --content "v2: Added retry logic for API failures. Changed from 30s to 60s timeout." \
  --summary "v2 deployment notes" \
  --agent-id "use-case-builder"
```

### When diagnosing issues: check history

```bash
codika notes get $PROJECT_ID --type changelog --history
codika notes get $PROJECT_ID --type debugging-log
```

Compare what changed before and after an issue started — version history lets you diff the project's evolution.

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- A project ID (from `project.json`, `--project-id`, or `config.ts`)

## Commands

### Upsert a Document

Create a new document type or update an existing one. Creates v0.0.0 on first call, increments PATCH on subsequent calls.

```bash
codika notes upsert <projectId> --type <type> --summary <summary> [options]
```

| Flag | Description |
|------|-------------|
| `--type <type>` | **(Required)** Document type ID — lowercase with hyphens (e.g. `brief`, `known-issues`, `changelog`) |
| `--summary <summary>` | **(Required)** What changed in this version |
| `--title <title>` | Document title (defaults to type ID) |
| `--content <content>` | Document content (markdown) |
| `--file <path>` | Read content from a file instead of `--content` |

Content can also be piped via stdin: `echo "..." | codika notes upsert <projectId> --type <type> --summary <summary>`
| `--agent-id <id>` | Agent identifier (for tracking who wrote it) |
| `--major-change` | Bump MINOR version instead of PATCH |
| `--api-key <key>` | Override API key |
| `--json` | JSON output |

### List Documents

List all current project notes for a project.

```bash
codika notes list <projectId> [options]
```

| Flag | Description |
|------|-------------|
| `--type <type>` | Filter by document type ID |
| `--api-key <key>` | Override API key |
| `--json` | JSON output |

### Get a Document

Read the content of a specific document, optionally with version history.

```bash
codika notes get <projectId> --type <type> [options]
```

| Flag | Description |
|------|-------------|
| `--type <type>` | **(Required)** Document type ID |
| `--version <version>` | Get a specific version (e.g. `0.1.0`) |
| `--history` | Show all versions instead of just the current one |
| `--api-key <key>` | Override API key |
| `--json` | JSON output |

## Examples

**Record a project brief:**

```bash
codika notes upsert proj-abc123 \
  --type brief \
  --title "Project Brief" \
  --content "This project automates invoice processing..." \
  --summary "Initial brief"
```

**Update with a file:**

```bash
codika notes upsert proj-abc123 \
  --type known-issues \
  --file ./notes/known-issues.md \
  --summary "Added timeout issue with large PDFs"
```

**Read what was documented:**

```bash
codika notes get proj-abc123 --type brief
```

**Check version history:**

```bash
codika notes get proj-abc123 --type known-issues --history
```

**List all documents for a project:**

```bash
codika notes list proj-abc123
```

**Machine-readable output:**

```bash
codika notes list proj-abc123 --json
```

## Expected Output

**Upsert (new):**

```
Created brief -> v0.0.0
  Document ID: 019...
  Project:     proj-abc123
```

**Upsert (update):**

```
Updated known-issues -> v0.0.3
  Document ID: 019...
  Project:     proj-abc123
```

**List:**

```
Project notes for project proj-abc123:

  brief  v0.0.0  "Project Brief"
    Initial brief  (45 words)
  known-issues  v0.0.3  "Known Issues"
    Added timeout issue with large PDFs  (120 words)

2 document(s)
```

**Get:**

```
brief  v0.0.0  "Project Brief"
Summary: Initial brief
Words: 45  |  Status: current

---

This project automates invoice processing...
```

**Get with --history:**

```
Version history for "known-issues":

  * v0.0.3  current  "Added timeout issue with large PDFs"
    v0.0.2  superseded  "Documented rate limiting edge case"
    v0.0.1  superseded  "Fixed formatting"
    v0.0.0  superseded  "Initial known issues"

4 version(s)
```

## Suggested Document Types

These are conventions, not enforced — use any lowercase-hyphenated name:

| Type | Purpose |
|------|---------|
| `brief` | What the project does, who it's for, key decisions |
| `known-issues` | Bugs, edge cases, gotchas discovered during development |
| `changelog` | What changed and why across deployments |
| `debugging-log` | Diagnostic trail for ongoing issues |
| `deployment-notes` | What was deployed, configuration details, known state |
| `architecture` | Key architectural decisions and rationale |

## Versioning

- First upsert for a document type creates version `0.0.0`
- Subsequent upserts increment PATCH: `0.0.0` -> `0.0.1` -> `0.0.2`
- Use `--major-change` to bump MINOR: `0.0.5` -> `0.1.0`
- Previous versions are marked as `superseded` (immutable, queryable)
- Full version history is preserved and accessible via `--history`

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` (see `setup-codika` skill) |
| "Project not found" | Invalid project ID | Verify the project exists with `codika get project <id>` |
| "documentTypeId must be lowercase" | Invalid type format | Use lowercase with hyphens (e.g. `known-issues`, not `Known Issues`) |
| "content is required" | Missing content | Provide `--content` or `--file` |

## Exit Codes

- `0` — Operation succeeded
- `1` — Operation failed
