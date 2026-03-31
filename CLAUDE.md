# Codika Marketplace (Public)

Public Claude Code plugin marketplace by Codika. Users install individual plugins to teach Claude Code agents how to work with the Codika platform.

## Repository Structure

This is a **marketplace** — it contains multiple independent plugins, each with its own `.claude-plugin/plugin.json`, skills, and (optionally) agents, commands, and hooks.

```
.
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── plugins/
│   ├── codika/                   # CLI helper plugin
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       ├── setup-codika/
│   │       ├── create-project/
│   │       ├── create-organization/
│   │       ├── create-organization-key/
│   │       ├── deploy-use-case/
│   │       ├── deploy-data-ingestion/
│   │       ├── deploy-documents/
│   │       ├── publish-use-case/
│   │       ├── redeploy-use-case/
│   │       ├── verify-use-case/
│   │       └── manage-integrations/
│   └── use-case-builder/         # Autonomous use case agents
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── agents/
│       │   ├── use-case-builder.md
│       │   ├── use-case-modifier.md
│       │   ├── n8n-workflow-builder.md
│       │   └── use-case-tester.md
│       └── skills/
│           └── discover-codika-guides/
│               ├── SKILL.md
│               └── references/   # Bundled platform documentation
│                   ├── use-case-guide.md
│                   ├── specific/         # 11 implementation guides
│                   └── integrations/     # 19 integration guides
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
| `create-organization` | Create an organization via `codika organization create` |
| `create-organization-key` | Create an organization API key via `codika organization create-key` |
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
| `get-instance` | Fetch instance details via `codika get instance` |
| `list-executions` | List recent executions via `codika list executions` |
| `list-instances` | List all process instances via `codika list instances` |
| `instance-activate` | Activate or deactivate instances via `codika instance activate/deactivate` |
| `manage-integrations` | Manage integrations (set, list, delete) via `codika integration` |

### use-case-builder (`plugins/use-case-builder/`)

Autonomous agents that read platform documentation to design, build, test, and deploy Codika use cases. The user provides a goal — the agents study the guides to figure out the best architecture. Requires the `codika` plugin for CLI skills.

All platform documentation is bundled in the `discover-codika-guides` skill's `references/` folder (main use-case guide, 11 specific guides, 19 integration guides, plan examples, common errors). Agents use the skill to discover and read these guides at runtime.

**Agents:**

| Agent | Description |
|-------|-------------|
| `use-case-builder` | Reads platform documentation, designs the architecture from user requirements, creates config.ts, delegates workflow creation, validates |
| `use-case-modifier` | Reads and understands an existing use case (config.ts + all workflows), reads the docs, then makes targeted modifications |
| `n8n-workflow-builder` | Builds individual n8n workflow JSON files following Codika patterns. Called by builder/modifier or directly |
| `use-case-tester` | Runs deploy-trigger-inspect-fix loops to test and debug use cases automatically |

**Skills:**

| Skill | Description |
|-------|-------------|
| `discover-codika-guides` | Locates the codika-processes-lib repository and lists available documentation guides |

**Dependency:** Users must also install the `codika` plugin — the agents use its CLI skills (deploy, verify, trigger, get-execution, etc.).

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
