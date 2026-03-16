---
name: get-skills
description: Fetches agent skill documents from a deployed Codika process instance. Use when the user wants to discover, download, or inspect the available workflow endpoints and their documentation for a process instance.
---

# Get Skills

Fetch Claude-compatible skill documents from a deployed process instance. Skills describe how to interact with a use case's HTTP endpoints and scheduled workflows.

## When to Use

- User asks to discover what endpoints are available for a process instance
- User wants to download skills to use with Claude Code or the Claude API
- User asks to inspect or list the available workflows and their documentation
- Agent needs to understand how to interact with a deployed use case

## Prerequisites

- Authenticated with `codika login` or `CODIKA_API_KEY` environment variable
- A deployed process instance (with `devProcessInstanceId` in project.json, or known ID)

## Command

```bash
codika get skills [processInstanceId] [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `processInstanceId` | No | Process instance ID. If omitted, resolves from project.json |

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `--process-instance-id <id>` | — | Alternative to positional argument |
| `--path <path>` | — | Path to use case folder (to resolve from project.json) |
| `--project-file <path>` | `project.json` | Custom project file name |
| `-o, --output <dir>` | `./skills` | Output directory for downloaded skill files |
| `--stdout` | `false` | Print skill content to stdout instead of writing files |
| `--api-url <url>` | Production | Override API URL |
| `--api-key <key>` | Active profile | Override API key |
| `--profile <name>` | Active profile | Use a specific profile instead of the active one |
| `--json` | `false` | Structured JSON output |

## Examples

### From inside a use case folder

```bash
codika get skills
```

Resolves `devProcessInstanceId` from `project.json` and downloads skills to `./skills/`.

### Typical agent usage (from anywhere, all explicit)

```bash
codika get skills 019c8fb3-6bb2-773d-bf90-540c35bf8457 --profile satelites --output .claude/skills
```

This is the most common pattern for agents: explicit process instance ID + explicit profile + explicit output directory. Works from any folder — no project.json needed.

### From inside a use case folder (shorthand)

```bash
codika get skills
```

Only works when `project.json` exists in the current directory with `devProcessInstanceId`. Uses the active profile.

### Download to Claude Code skills directory

```bash
codika get skills --output .claude/skills
```

Downloads skills directly into `.claude/skills/` for Claude Code auto-discovery.

### With a specific profile (multi-org)

```bash
codika get skills --profile wat
```

Uses the `wat` profile's API key instead of the active one. To find the right profile for an org, run `codika use --json` to list all profiles with their `organizationId`, then match against the project's `organizationId`.

### JSON output for scripting

```bash
codika get skills --json
```

### Print to stdout

```bash
codika get skills --stdout
```

## Expected Output

### Human-readable (default)

```
✓ Downloaded 3 skill(s) to ./skills/

  process-text/SKILL.md
    Submits text for AI processing via HTTP POST
    Trigger: codika trigger main-workflow

  scheduled-report/SKILL.md
    Generates a weekly report automatically
    Trigger: codika trigger scheduled-report

  search-contacts/SKILL.md
    Searches CRM contacts by name or email
    Trigger: codika trigger search-contacts

To use with Claude Code, copy to .claude/skills/
```

### JSON output

```json
{
  "success": true,
  "processInstanceId": "abc123def456",
  "skillCount": 2,
  "skills": [
    {
      "name": "process-text",
      "description": "Submits text for AI processing via HTTP POST",
      "workflowTemplateId": "main-workflow",
      "contentMarkdown": "---\nname: process-text\n...",
      "relativePath": "skills/main-workflow/SKILL.md"
    }
  ]
}
```

## How It Works

1. Resolves process instance ID (flag > project.json > current directory)
2. Calls `getProcessSkillsPublic` cloud function with API key auth
3. Downloads skill documents from the deployment instance
4. Writes each skill as a Claude-compatible directory: `{name}/SKILL.md`

## Using Downloaded Skills

### With Claude Code

Copy downloaded skills into your project's `.claude/skills/` directory. Claude Code will auto-discover them:

```bash
codika get skills --output .claude/skills
```

### With the Claude API

Upload skills via the Skills API:

```python
from anthropic.lib import files_from_dir
skill = client.beta.skills.create(
    display_title="Process Text",
    files=files_from_dir("./skills/process-text"),
    betas=["skills-2025-10-02"],
)
```

### Triggering a workflow

After reading a skill, trigger the workflow it describes:

```bash
codika trigger <workflowTemplateId> --payload-file input.json --poll
```

## Process Instance ID Resolution

Priority order:
1. Positional argument (highest)
2. `--process-instance-id` flag
3. `devProcessInstanceId` from project.json in `--path` directory
4. `devProcessInstanceId` from project.json in current directory

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `Process instance ID is required` | No ID found from any source | Provide ID or ensure project.json has `devProcessInstanceId` |
| `API key is required` | No API key found | Run `codika login` or set `CODIKA_API_KEY` |
| `No skills found` | Process instance has no skills deployed | Deploy with skills: add `skills/` folder and redeploy |
| `401 Unauthorized` | Invalid API key | Run `codika login` to refresh credentials |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error (API failure, auth failure) |
