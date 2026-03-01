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
│           └── verify-use-case/
├── README.md
├── CLAUDE.md
└── LICENSE
```

## Plugins

### codika (`plugins/codika/`)

Skills for the `codika-helper` CLI (`@codika-io/helper-sdk`). Pure documentation — no code, no dependencies, no secrets.

| Skill | Description |
|-------|-------------|
| `setup-codika` | Install the CLI globally and run `codika-helper login` |
| `create-project` | Create a project via `codika-helper project create` |
| `deploy-use-case` | Validate then deploy via `codika-helper verify` + `codika-helper deploy` |
| `verify-use-case` | Validate workflows via `codika-helper verify use-case` |

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
