---
name: discover-codika-guides
description: Locates the use-case-builder plugin's guides/ folder which contains all Codika platform documentation for building and debugging use cases. Use before reading any Codika documentation.
---

# Discover Codika Guides

The `discover-codika-guides` skill ships with a `references/` folder containing all Codika platform documentation for building use cases. This skill helps you find and read the right guides.

## Discovery Strategy

The guides are bundled inside this skill's `references/` directory. Run these checks in order to find them:

1. **Search for the skill's references** — Use Glob to find: `**/discover-codika-guides/references/use-case-guide.md`
2. **Check common install paths** — Search for `**/use-case-builder/skills/discover-codika-guides/references/use-case-guide.md`

Once found, store the `references/` base path and use it for all subsequent reads.

If none found, tell the user: "I can't find the Codika guides. Make sure the `use-case-builder` plugin is installed from the Codika marketplace."

## Directory Structure

```
skills/discover-codika-guides/
├── SKILL.md                         ← This file
└── references/                      ← All documentation lives here
    ├── use-case-guide.md            # Main guide
    ├── specific/                    # Detailed implementation guides
    ├── integrations/                # Per-integration reference
    ├── plan-examples/               # Example build plans
    ├── post-creation/               # Post-creation validation
    └── use-case/                    # Use-case-specific guides
```

## Available Guides

All paths below are relative to the `references/` folder.

### Core Guides (read for every use case)

| Guide | Path | Content |
|-------|------|---------|
| **Main use-case guide** | `use-case-guide.md` | Overall architecture, mandatory patterns, placeholder system, trigger types, config structure, Codika nodes, input/output schemas |
| **Codika nodes** | `specific/codika-nodes.md` | Codika Init, Submit Result, Report Error, Upload File — node parameters and configuration |
| **Config patterns** | `specific/config-patterns.md` | config.ts structure, exports, workflow array, integration UIDs, display metadata |

### Trigger-Specific Guides (read based on trigger type)

| Guide | Path | When to read |
|-------|------|-------------|
| **HTTP triggers** | `specific/http-triggers.md` | Use case has HTTP/webhook triggers (user forms, API endpoints) |
| **Schedule triggers** | `specific/schedule-triggers.md` | Use case has cron/scheduled triggers |
| **Third-party triggers** | `specific/third-party-triggers.md` | Use case has Gmail, Slack, WhatsApp, Pipedrive, Calendly triggers |
| **Sub-workflows** | `specific/sub-workflows.md` | Use case needs reusable workflow logic called by parent workflows |

### Specialized Guides (read when applicable)

| Guide | Path | When to read |
|-------|------|-------------|
| **Placeholder patterns** | `specific/placeholder-patterns.md` | Complete reference for all 11 placeholder types (FLEXCRED, USERCRED, ORGCRED, etc.) |
| **AI nodes** | `specific/ai-nodes.md` | Use case uses LLM/AI nodes (chainLlm vs agent, tool workflows, $fromAI()) |
| **Process input schema** | `specific/process-input-schema.md` | Use case needs deployment parameters (INSTPARM, user-configurable at install) |
| **Data ingestion** | `specific/data-ingestion.md` | Use case has RAG/document embedding pipelines |
| **Agent skills** | `specific/agent-skills.md` | Creating SKILL.md files for workflow discoverability |

### Integration Guides

Located in `references/integrations/`. Read when the use case involves a specific integration:

**AI Providers (FLEXCRED):** `anthropic.md`, `openai.md`, `tavily.md`, `xai.md`, `openrouter.md`, `mistral.md`, `cohere.md`, `deepseek.md`, `fal-ai.md`

**Organization (ORGCRED):** `whatsapp.md`, `slack.md`, `twilio.md`, `folk.md`, `pipedrive.md`

**User (USERCRED):** `google.md`, `microsoft.md`, `calendly.md`, `notion.md`

**Instance (INSTCRED):** `supabase.md`, `postgres.md`

### Additional Resources

| Resource | Path | Content |
|----------|------|---------|
| **Plan examples** | `plan-examples/` | Example build plans for reference |
| **Common errors** | `post-creation/common-errors.md` | Common issues after creating workflows |
| **WhatsApp bots** | `use-case/whatsapp-bots.md` | WhatsApp bot-specific patterns |

## Recommended Reading Order

**For building a new use case:**
1. `use-case-guide.md` (always first)
2. `specific/config-patterns.md`
3. Trigger-specific guide (based on requirements)
4. `specific/codika-nodes.md`
5. `specific/placeholder-patterns.md` (if unsure about credentials)
6. `specific/ai-nodes.md` (if using AI)
7. Relevant integration guides

**For debugging/fixing a use case:**
1. `specific/codika-nodes.md`
2. Trigger-specific guide (based on the failing workflow's trigger)
3. `specific/placeholder-patterns.md`
4. `post-creation/common-errors.md`

**For creating a sub-workflow:**
1. `specific/sub-workflows.md`
2. `specific/ai-nodes.md` (if the sub-workflow is an AI tool)
