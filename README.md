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
npm install -g @codika-io/helper-sdk
codika login
```

## Available Plugins

| Plugin | Description | Skills |
|--------|-------------|--------|
| `codika` | CLI skills for creating projects, deploying and validating use cases | `setup-codika`, `create-project`, `deploy-use-case`, `verify-use-case` |

## Plugin: codika

Skills for the `codika` CLI. Once installed, Claude Code agents can:

| Skill | What it does |
|-------|-------------|
| `codika:setup-codika` | Install the CLI and authenticate |
| `codika:create-project` | Create a new project on the platform |
| `codika:deploy-use-case` | Validate and deploy a use case |
| `codika:verify-use-case` | Validate workflows without deploying |

## What is Codika?

Codika is a multi-tenant SaaS platform for building, deploying, and managing business process automations powered by n8n workflows. Users describe what they need, AI agents create the workflows, and the platform handles deployment, credential management, and execution tracking.

## Contributing

See `CLAUDE.md` for conventions and instructions on adding new plugins.

## License

MIT
