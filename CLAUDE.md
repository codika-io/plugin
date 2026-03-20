# Codika Marketplace (Public)

Public Claude Code plugin marketplace by Codika. Users install individual plugins to teach Claude Code agents how to work with the Codika platform.

## Repository Structure

This is a **marketplace** — it contains multiple independent plugins, each with its own `.claude-plugin/plugin.json`, skills, and (optionally) agents, commands, and hooks.

```
.
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── plugins/
│   └── codika/                   # CLI helper plugin
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           ├── setup-codika/
│           ├── create-project/
│           ├── deploy-use-case/
│           ├── deploy-data-ingestion/
│           ├── deploy-documents/
│           ├── publish-use-case/
│           ├── redeploy-use-case/
│           ├── verify-use-case/
│           └── manage-integrations/
├── README.md
├── CLAUDE.md
└── LICENSE
```

## Plugins

### codika (`plugins/codika/`)

Skills for the `codika` CLI (`codika`). Pure documentation — no code, no dependencies, no secrets.

| Skill | Description |
|-------|-------------|
| `setup-codika` | Install the CLI globally and run `codika login` |
| `create-project` | Create a project via `codika project create` |
| `deploy-use-case` | Validate then deploy via `codika verify` + `codika deploy` |
| `deploy-data-ingestion` | Deploy data ingestion config via `codika deploy process-data-ingestion` |
| `deploy-documents` | Upload stage documents via `codika deploy documents` |
| `publish-use-case` | Publish a deployment to production via `codika publish` |
| `redeploy-use-case` | Redeploy with parameter changes via `codika redeploy` |
| `verify-use-case` | Validate workflows via `codika verify use-case` |
| `init-use-case` | Scaffold a new use case via `codika init` |
| `fetch-use-case` | Download a deployed use case via `codika get use-case` |
| `trigger-workflow` | Trigger a workflow via `codika trigger` |
| `get-execution` | Fetch execution details via `codika get execution` |
| `list-executions` | List recent executions via `codika list executions` |
| `manage-integrations` | Manage integrations (set, list, delete) via `codika integration` |

## Conventions

- Skills must not contain secrets, API keys, or internal URLs
- Skills should reference CLI commands and flags, not internal implementation details
- Skills should guide agents to use `setup-codika` when authentication fails
- Keep SKILL.md files self-contained — each should work independently
- Each plugin gets its own directory under `plugins/` with its own `.claude-plugin/plugin.json`

## Adding a New Plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`
2. Add skills under `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`
3. Register the plugin in `.claude-plugin/marketplace.json` under the `plugins` array
4. Update README.md with the new plugin's documentation

## Testing

```bash
# Add this marketplace
/plugin marketplace add codika-io/codika-marketplace

# Install a plugin
/plugin install codika@codika-marketplace

# Skills are namespaced
/codika:setup-codika
/codika:deploy-use-case
```
