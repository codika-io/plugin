---
name: verify-use-case
description: Validate a use case folder or workflow without deploying
trigger:
  when: User asks to verify, validate, lint, or check a use case
  actions: [verify, validate, check, lint, audit]
  targets: [use case, workflow, use-case structure, config]
---

# Verify Use Case

Validate a use case folder (or single workflow) without deploying.

## When to Use

- Before deploying to catch issues early
- After modifying workflow JSON files
- When the user asks to lint, validate, or check their use case
- As part of a CI pipeline

## Prerequisites

- `codika-helper` CLI installed (see `setup-codika` skill)
- No authentication needed — verification runs locally

## Commands

### Validate an entire use case

```bash
codika-helper verify use-case <path> [options]
```

### Validate a single workflow

```bash
codika-helper verify workflow <path> [options]
```

### Options

| Flag | Description |
|------|-------------|
| `--json` | Output findings as JSON (machine-readable) |
| `--fix` | Auto-fix fixable violations in place |
| `--dry-run` | Show what `--fix` would change without modifying files |
| `--strict` | Treat `should` severity as `must` (fail on warnings) |
| `--rules <ids>` | Only run specific rules (comma-separated) |
| `--exclude-rules <ids>` | Skip specific rules (comma-separated) |

## Examples

**Basic validation:**

```bash
codika-helper verify use-case ./use-cases/marketplace/my-use-case
```

**Auto-fix fixable issues:**

```bash
codika-helper verify use-case ./use-cases/marketplace/my-use-case --fix
```

**Preview fixes without applying:**

```bash
codika-helper verify use-case ./use-cases/marketplace/my-use-case --dry-run
```

**JSON output for CI:**

```bash
codika-helper verify use-case ./use-cases/marketplace/my-use-case --json
```

**Single workflow validation:**

```bash
codika-helper verify workflow ./workflows/main-workflow.json
```

## Expected Output

**All passing:**

```
✓ Use case validation passed

  0 must | 0 should | 0 nit
```

**With findings:**

```
✗ Use case validation failed

  workflows/main.json
    MUST  [CK-INIT] Missing Codika Init node
    MUST  [CK-SUBMIT] Missing Submit Result / Report Error nodes
    SHOULD [WF-SETTINGS] Missing errorWorkflow setting

  2 must | 1 should | 0 nit
```

**JSON output:**

```json
{
  "valid": false,
  "findings": [
    {
      "rule": "CK-INIT",
      "severity": "must",
      "path": "workflows/main.json",
      "message": "Missing Codika Init node",
      "fixable": false
    }
  ],
  "summary": { "must": 2, "should": 1, "nit": 0 }
}
```

## Common Validation Rules

| Rule | Severity | What it checks |
|------|----------|----------------|
| `CONFIG-EXPORTS` | must | config.ts exports required members. `PROJECT_ID` is optional when `project.json` exists |
| `CK-INIT` | must | Parent workflows must have a Codika Init node |
| `CK-SUBMIT` | must | Parent workflows must end with Submit Result / Report Error |
| `WF-SETTINGS` | must | Workflow settings (errorWorkflow, timezone, executionOrder) |
| `CK-PLACEHOLDERS` | must | Placeholder syntax follows the `{{TYPE_KEY_SUFFIX}}` pattern |
| `CK-CREDENTIALS` | must | Credentials use Codika patterns (FLEXCRED, USERCRED, etc.) |
| `CK-SUBWORKFLOW` | should | Sub-workflow references use `SUBWKFL` placeholders |

## Recommended Workflow

1. Run `codika-helper verify use-case <path>` to see all issues
2. Run `codika-helper verify use-case <path> --fix` to auto-fix what's possible
3. Manually fix remaining `must` violations
4. Run verification again to confirm clean
5. Deploy with `codika-helper deploy use-case <path>` (see `deploy-use-case` skill)

## Exit Codes

- `0` — All validations passed
- `1` — Validation failures found (at least one `must`)
