# Codika Plugin

Official agent plugin for the [Codika](https://codika.io) platform. Installs CLI skills + autonomous builder agents so any compatible coding agent (Claude Code, Cursor, …) can create projects, design n8n workflows, deploy use cases, and manage the platform end-to-end.

Conforms to the [Open Plugin Specification v1.0](https://github.com/vercel-labs/open-plugin-spec).

## Install

### Any Open-Plugin-compatible host (Claude Code, Cursor, …)

```bash
npx plugins add codika-io/plugin
```

The `plugins` CLI auto-detects which agent tools are installed and installs to all of them.

### Claude Code (native)

```
/plugin marketplace add codika-io/plugin
/plugin install codika@plugin
```

### CLI (required for deployment skills)

```bash
npm install -g codika
codika login
```

Then run `/codika:setup-codika` once per machine to complete authentication inside the agent.

## What's Inside

One plugin, `codika`, with 25 skills + 4 autonomous agents.

### Skills (`/codika:<skill>`)

Pure documentation-based skills that wrap the `codika` CLI. No code, no secrets.

**Setup & auth**
| Skill | What it does |
|---|---|
| `setup-codika` | Install the CLI and authenticate |
| `discover-codika-guides` | Locate and list the bundled platform documentation |

**Organizations & projects**
| Skill | What it does |
|---|---|
| `create-organization` | Create a new organization |
| `create-organization-key` | Create an organization API key |
| `update-organization-key` | Update an organization API key |
| `create-project` | Create a new project |
| `list-projects` | List all projects |
| `get-project` | Fetch project details |

**Use cases — build & deploy**
| Skill | What it does |
|---|---|
| `init-use-case` | Scaffold a new use case |
| `verify-use-case` | Validate workflows without deploying |
| `deploy-use-case` | Validate then deploy via `codika verify` + `codika deploy` |
| `redeploy-use-case` | Redeploy with parameter changes |
| `publish-use-case` | Publish a deployment to production |
| `fetch-use-case` | Download a deployed use case |
| `deploy-documents` | Upload stage documents |
| `deploy-data-ingestion` | Deploy data ingestion config |

**Execution & operations**
| Skill | What it does |
|---|---|
| `trigger-workflow` | Trigger a workflow |
| `get-execution` | Fetch execution details |
| `list-executions` | List recent executions |
| `list-instances` | List all process instances |
| `get-instance` | Fetch instance details |
| `instance-activate` | Activate or deactivate instances |
| `manage-integrations` | Configure, list, and delete integrations |
| `manage-notes` | Manage project notes |
| `get-skills` | List available platform skills |

### Agents (`Task` subagent_type `codika:<agent>`)

Autonomous agents that read the bundled platform documentation to design, build, test, and debug Codika use cases. The user provides a goal — the agents study the guides to figure out the best architecture.

| Agent | What it does |
|---|---|
| `use-case-builder` | Reads documentation, designs architecture from requirements, creates `config.ts`, delegates workflow creation, validates |
| `use-case-modifier` | Reads an existing use case (config.ts + all workflows), reads docs, makes targeted modifications |
| `n8n-workflow-builder` | Builds individual n8n workflow JSON files following Codika patterns |
| `use-case-tester` | Runs deploy-trigger-inspect-fix loops to test and debug use cases automatically |

**Usage examples:**
- "Create a use case for invoice processing with PDF upload and AI extraction" → invokes `codika:use-case-builder`
- "Add a Slack notification to the invoice-processor use case" → invokes `codika:use-case-modifier`
- "Build an HTTP-triggered workflow for document classification" → invokes `codika:n8n-workflow-builder`
- "Test and fix the invoice-processor use case" → invokes `codika:use-case-tester`

**Self-contained:** All platform documentation ships with the plugin under `skills/discover-codika-guides/references/` (main use-case guide + 11 specific guides + 19 integration guides + plan examples). Agents read these directly — no need to clone `codika-processes-lib`.

## What is Codika?

Codika is a multi-tenant SaaS platform for building, deploying, and managing business process automations powered by n8n workflows. Users describe what they need, AI agents create the workflows, and the platform handles deployment, credential management, and execution tracking.

## Contributing

See `CLAUDE.md` for conventions and instructions on adding new skills or agents.

## License

MIT
