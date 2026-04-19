# Codika Plugin

Public agent plugin for the Codika platform. Published as a single [Open Plugin v1](https://github.com/vercel-labs/open-plugin-spec)-conformant repo, installable via `npx plugins add codika-io/plugin` or Claude Code's `/plugin marketplace add` flow.

## Repository Structure

The repo root IS the plugin root — no `plugins/` wrapper, no marketplace layer. This matches `vercel/vercel-plugin` and `slideless-ai/plugin`, the recommended Open Plugin layout for single-plugin repos.

```
.
├── .plugin/
│   └── plugin.json                       # Vendor-neutral manifest (Open Plugin v1)
├── .claude-plugin/
│   └── plugin.json                       # Vendor-prefixed manifest (preferred by Claude Code)
├── skills/                               # 25 skills, all namespaced as `/codika:<name>`
│   ├── setup-codika/
│   ├── create-organization/
│   ├── create-organization-key/
│   ├── update-organization-key/
│   ├── create-project/
│   ├── list-projects/
│   ├── get-project/
│   ├── init-use-case/
│   ├── verify-use-case/
│   ├── deploy-use-case/
│   ├── redeploy-use-case/
│   ├── publish-use-case/
│   ├── fetch-use-case/
│   ├── deploy-documents/
│   ├── deploy-data-ingestion/
│   ├── trigger-workflow/
│   ├── get-execution/
│   ├── list-executions/
│   ├── list-instances/
│   ├── get-instance/
│   ├── instance-activate/
│   ├── manage-integrations/
│   ├── manage-notes/
│   ├── get-skills/
│   └── discover-codika-guides/
│       ├── SKILL.md
│       └── references/                   # Bundled platform docs
│           ├── use-case-guide.md
│           ├── specific/                 # 11 implementation guides
│           ├── integrations/             # 19 integration guides
│           ├── use-case/                 # Use case examples
│           └── plan-examples/
├── agents/                               # 4 agents, invokable as Task subagent_type `codika:<name>`
│   ├── use-case-builder.md
│   ├── use-case-modifier.md
│   ├── n8n-workflow-builder.md
│   └── use-case-tester.md
├── README.md
├── CLAUDE.md
├── SECURITY.md
├── LICENSE
└── .gitignore
```

## Manifests

Two manifests are kept intentionally in sync:

- `.plugin/plugin.json` — canonical Open Plugin v1 manifest. Read by `npx plugins`, Cursor, and any other Open-Plugin-compatible host.
- `.claude-plugin/plugin.json` — Claude Code vendor-prefixed manifest. Per Open Plugin §5.1, Claude Code prefers this when both are present.

When editing manifest metadata (`version`, `description`, `keywords`, …), update BOTH files. Both ship the same schema.

## History

Until v2.0.0 this repo was a two-plugin marketplace (`codika` + `use-case-builder`). Merged in 2026-04 following the `vercel/vercel-plugin` + `slideless-ai/plugin` single-plugin pattern. The 4 `use-case-builder` agents and the `discover-codika-guides` skill moved into the unified `codika` plugin — every skill and agent is now namespaced `/codika:*`.

## Conventions

- Skills must not contain secrets, API keys, or internal URLs.
- Skills should reference CLI commands and flags, not internal implementation details.
- Skills should guide agents to use `setup-codika` when authentication fails.
- Keep SKILL.md files self-contained — each should work independently.
- Agents (autonomous Task-invocable) live in `agents/<name>.md`. Skills (slash-command, interactive) live in `skills/<name>/SKILL.md`.

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`).
2. The plugin loader picks it up automatically from the default `skills/` discovery location (Open Plugin §7.1) — no manifest edits needed.
3. Document it in `README.md`.

## Adding a New Agent

1. Create `agents/<agent-name>.md` with YAML frontmatter (`name`, `description`, optional `tools`).
2. The agent is immediately invokable via Task with `subagent_type: "codika:<agent-name>"`.
3. Document it in `README.md`.

## Versioning

Follow semantic versioning. Breaking changes (skill rename, agent rename, skill/agent removal, manifest name change) require a major bump and a note in the README's History section. Additive changes (new skills, new agents, new references docs) only need a minor bump.

## Testing

```bash
# Test local install against any Open-Plugin-compatible host:
npx plugins add /Users/romainpattyn/.codika/core/app/codika-plugin

# Or Claude Code native (from this repo once pushed):
/plugin marketplace add codika-io/plugin
/plugin install codika@plugin

# All skills/agents namespaced as /codika:*
/codika:setup-codika
/codika:deploy-use-case
# Task tool subagent_type: "codika:use-case-builder"
```
