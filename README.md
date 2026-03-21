# Codika Marketplace

Public Claude Code plugin marketplace by [Codika](https://codika.io). Install plugins to let Claude Code agents create projects, deploy use cases, and manage workflows on the Codika platform.

## Quick Start

### 1. Add the Marketplace

```
/plugin marketplace add codika-io/codika-marketplace
```

### 2. Install a Plugin

```
/plugin install codika@codika-marketplace
```

### 3. Install the CLI

```bash
npm install -g codika
codika login
```

## Available Plugins

| Plugin | Description | Type |
|--------|-------------|------|
| `codika` | CLI skills for creating projects, deploying and validating use cases, managing integrations | Skills |
| `use-case-builder` | Autonomous agents that read docs to design, build, test, and deploy use cases from requirements | Agents |

## Plugin: codika

Skills for the `codika` CLI. Once installed, Claude Code agents can:

| Skill | What it does |
|-------|-------------|
| `codika:setup-codika` | Install the CLI and authenticate |
| `codika:create-project` | Create a new project on the platform |
| `codika:deploy-use-case` | Validate and deploy a use case |
| `codika:redeploy-use-case` | Redeploy an instance with different parameters |
| `codika:verify-use-case` | Validate workflows without deploying |
| `codika:manage-integrations` | Configure, list, and delete integrations (API keys, credentials) |

## Plugin: use-case-builder

Autonomous agents that read platform documentation to design, build, test, and fix Codika use cases. The user provides a goal — the agents study the guides to figure out the best architecture. Requires the `codika` plugin (agents use its CLI skills).

```
/plugin install use-case-builder@codika-marketplace
```

**Agents:**

| Agent | What it does |
|-------|-------------|
| `use-case-builder` | Reads platform documentation, designs architecture from user requirements, creates config.ts, delegates workflow creation, validates |
| `use-case-modifier` | Reads and understands an existing use case, reads the docs, then makes targeted modifications (add workflows, change triggers, add integrations) |
| `n8n-workflow-builder` | Builds individual n8n workflow JSON files following Codika patterns (called by builder/modifier or directly) |
| `use-case-tester` | Runs automated deploy-trigger-inspect-fix loops to test and debug use cases |

**Usage:**
- "Create a use case for invoice processing with PDF upload and AI extraction" → triggers `use-case-builder`
- "Add a Slack notification to the invoice-processor use case" → triggers `use-case-modifier`
- "Build an HTTP-triggered workflow for document classification" → triggers `n8n-workflow-builder`
- "Test and fix the invoice-processor use case" → triggers `use-case-tester`

**Self-contained:** The plugin ships with all platform documentation (`guides/` folder). No need to clone `codika-processes-lib` — agents read the bundled guides directly.

## What is Codika?

Codika is a multi-tenant SaaS platform for building, deploying, and managing business process automations powered by n8n workflows. Users describe what they need, AI agents create the workflows, and the platform handles deployment, credential management, and execution tracking.

## Contributing

See `CLAUDE.md` for conventions and instructions on adding new plugins.

## License

MIT
