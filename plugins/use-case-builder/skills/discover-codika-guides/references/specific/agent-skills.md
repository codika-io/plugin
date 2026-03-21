# Agent Skills Guide

This guide explains how to create agent skills for your use case. Skills make your workflows discoverable and usable by AI agents — they describe what each endpoint does, how to call it, and what to expect back.

---

## 1. Why Skills Exist

Codika use cases expose HTTP endpoints and scheduled workflows. Humans interact with them through the dashboard UI. But AI agents need a different interface — they need **documentation that explains what's available and how to use it programmatically**.

Skills bridge this gap. Each skill is a markdown file that describes one workflow: what it does, how to trigger it via the `codika` CLI, what input it expects, and what output it returns. Skills follow the official [Claude Agent Skills format](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview), so they work natively with Claude Code, the Claude API, and the Claude Agent SDK.

**The flow:**
1. You create workflows in your use case
2. You write a skill for each triggerable workflow
3. You deploy — skills travel with the deployment
4. An agent runs `codika get skills` to download them
5. The agent reads the skill files and knows exactly how to interact with your workflows

---

## 2. Which Workflows Get Skills

Create one skill per **triggerable** workflow:

| Workflow Type | Gets a Skill? | Why |
|---------------|---------------|-----|
| HTTP-triggered | **Yes** | User/agent-facing endpoint |
| Scheduled (with manual trigger URL) | **Yes** | Auto-runs but can be manually triggered for testing |
| Sub-workflow | **No** | Internal, called by other workflows |
| Data ingestion | **No** | Internal, triggered by document uploads |
| Service event (Twilio, Gmail webhook) | **Usually no** | Triggered by external services, not by agents directly |

**Exception:** If a service event workflow can also be triggered via `codika trigger` (e.g., `main-router-baileys` accepts webhook POSTs), you may create a skill for it — but this is rare.

---

## 3. Directory Structure

Skills live in a `skills/` folder alongside `workflows/`:

```
your-use-case/
├── config.ts
├── workflows/
│   ├── main-workflow.json
│   ├── scheduled-report.json
│   └── text-processor.json          # sub-workflow — no skill
└── skills/
    ├── main-workflow/                # one directory per skill
    │   └── SKILL.md
    └── scheduled-report/
        └── SKILL.md
```

Each skill is a **directory** containing a `SKILL.md` file. This matches Claude's expected format — agents can use these directories directly as Claude Code skills or upload them to the Claude API.

**Naming convention:** The directory name should match the `workflowTemplateId` or be a descriptive slug. It does not need to match exactly, but keeping them aligned reduces confusion.

---

## 4. SKILL.md Format

Every `SKILL.md` has two parts: YAML frontmatter and a markdown body.

### 4.1 Frontmatter (Required)

```yaml
---
name: my-use-case-process-text
description: Submits text for AI processing via the main-workflow HTTP endpoint. Returns processed text with a timestamp.
workflowTemplateId: main-workflow
---
```

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Max 64 characters. Lowercase letters, numbers, and hyphens only. Must not contain "anthropic" or "claude" (reserved words). |
| `description` | Yes | Max 1024 characters. Non-empty. **Third person** — "Submits text for..." not "Use this to submit..." |
| `workflowTemplateId` | Yes | Must match a `workflowTemplateId` from the workflows array in `config.ts`. This is how skills are linked to workflows. |

**Naming tips:**
- Prefix with your use case slug: `wat-direct-messaging`, `propale-generate-proposal`
- Use hyphens, not underscores: `my-skill-name` not `my_skill_name`
- Be specific: `wat-event-reminder-today` not `reminder`

**Description tips:**
- Write in third person: "Generates a weekly report..." not "I generate..." or "You can use this to..."
- Include both what the skill does AND when to use it
- Mention key integrations if relevant: "...using Twilio for WhatsApp delivery"

### 4.2 Optional Frontmatter Fields

You can add additional metadata to the frontmatter for documentation purposes. These are not required by Claude but help agents and humans understand the skill:

```yaml
trigger: http                    # or: scheduled
schedule: "55 8 * * *"          # cron expression (scheduled only)
manualTrigger: true              # whether manual triggering is possible
integrations:                    # external services used
  - Twilio
  - Supabase
cost: 3                          # credit cost per execution
input:                           # structured input schema
  - name: phone_number
    type: string
    required: true
output:                          # structured output schema
  - name: responseText
    type: string
```

These extra fields are purely informational — the platform only reads `name`, `description`, and `workflowTemplateId` from the frontmatter.

### 4.3 Markdown Body

The body should be **concise** (under 500 lines) and follow progressive disclosure. Include:

1. **Title** — H1 matching the workflow name
2. **One-line overview** — What this endpoint does
3. **How to trigger** — Exact `codika trigger` command with example payload
4. **Input** — Table or JSON schema of input parameters
5. **Output** — Example JSON response
6. **Notes** — Cost, limitations, edge cases

---

## 5. Writing Skills for HTTP Workflows

HTTP workflows are the most common skill type. The agent sends a payload and gets a response.

### Template

````markdown
---
name: {usecase-slug}-{action}
description: {Third-person description of what the endpoint does and returns.}
workflowTemplateId: {workflow-template-id}
---

# {Workflow Name}

{One-line description including integrations used.}

## How to trigger

```bash
codika trigger {workflow-template-id} --payload-file - <<'EOF'
{
  "field1": "value1",
  "field2": "value2"
}
EOF
```

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| field1 | string | yes | Description |
| field2 | number | no | Description |

## Output

```json
{
  "result": "example value",
  "timestamp": "2025-01-01T00:00:00.000Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| result | string | Description |
| timestamp | string | ISO 8601 timestamp |

## Notes

- Cost: {N} credit(s) per execution
- {Any limitations, max sizes, important behaviors}
````

### Real Example: WAT Bot Direct Messaging

````markdown
---
name: wat-direct-messaging
description: Sends a WhatsApp message to a list of phone numbers via Twilio. Accepts up to 500 recipients and returns delivery statistics.
workflowTemplateId: http-direct-messaging
---

# Direct Messaging

Sends a WhatsApp message to up to 500 phone numbers in a single batch via Twilio.

## How to trigger

```bash
codika trigger http-direct-messaging --payload-file - <<'EOF'
{
  "phone_numbers": ["32477123456", "33612345678"],
  "message_content": "Hello from WAT community!"
}
EOF
```

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| phone_numbers | array of strings | yes | Phone numbers (8-15 digits, no + prefix). Max 500. |
| message_content | text | yes | Message body. Max 4096 characters. |

## Output

```json
{
  "success": true,
  "total_sent": 2,
  "total_failed": 0,
  "sent_at": "2025-03-15T10:30:00.000Z"
}
```

## Notes

- Cost: 1 credit per execution
- Uses: Twilio (WhatsApp), Supabase (audit logging)
- Phone numbers must be digits only — no `+`, spaces, or dashes
````

### Real Example: Propale AI Generate Proposal

````markdown
---
name: propale-generate-proposal
description: Generates a professional proposal using RAG-based retrieval from historical proposals. Returns an HTML file with sections, sources count, and timestamp.
workflowTemplateId: proposal-generation
---

# Generate Proposal

Generates a professional HTML proposal using RAG retrieval from historical proposals stored in Pinecone.

## How to trigger

```bash
codika trigger proposal-generation --payload-file - <<'EOF'
{
  "proposal_type": "project",
  "proposal_language": "french",
  "style": "professional",
  "notes": ["Client prefers agile methodology"],
  "guidelines": "Emphasize prior fintech experience."
}
EOF
```

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| proposal_type | select | yes | "project", "advisory", or "staffing" |
| requirements_file | file | no | PDF or Word doc (max 50 MB) |
| proposal_language | select | yes | "english", "french", "dutch", "german", "spanish" |
| style | select | no | "professional", "minimalist", "tech", "elegant", "bold" |
| notes | array of text | no | Additional context notes (max 10 items, 10000 chars each) |
| guidelines | text | no | Specific guidelines (max 5000 chars) |

## Output

```json
{
  "proposal_html_file": "<documentId>",
  "sections_generated": 6,
  "sources_used": 4,
  "generated_at": "2025-03-15T10:30:00.000Z"
}
```

## Notes

- Cost: 100 credits per execution
- Uses: OpenAI (embeddings), Pinecone (vector search)
- The output HTML file is uploaded to Codika storage and referenced by document ID
````

---

## 6. Writing Skills for Scheduled Workflows

Scheduled workflows run automatically but can often be triggered manually for testing. The skill should explain both behaviors.

### Template

````markdown
---
name: {usecase-slug}-{action}
description: {Third-person description. Mention the schedule AND that it can be manually triggered for testing.}
workflowTemplateId: {workflow-template-id}
---

# {Workflow Name}

Runs automatically on a schedule ({human-readable schedule}). No user input required.

## Schedule

{Cron expression} — {Human-readable description} ({timezone})

## Manual trigger (for testing)

```bash
codika trigger {workflow-template-id}
```

No payload required.

## Output

| Field | Type | Description |
|-------|------|-------------|
| field1 | type | Description |

## Notes

- Cost: {N} credit(s) per execution
- {What the scheduled job does, any side effects}
````

### Real Example: WAT Bot Event Weekly Digest

````markdown
---
name: wat-event-weekly-digest
description: Sends a personalized weekly digest of upcoming events to all community members every Monday at 9 AM via WhatsApp. Can be manually triggered for testing.
workflowTemplateId: scheduled-event-weekly-digest
---

# Event Weekly Digest

Sends a personalized weekly digest of upcoming events to all community members, respecting event visibility rules.

## Schedule

`0 9 * * 1` — Every Monday at 9:00 AM (Europe/Brussels)

## Manual trigger (for testing)

```bash
codika trigger scheduled-event-weekly-digest
```

No payload required.

## Notes

- Cost: 1 credit per execution
- Uses: Supabase (event data, member list), Twilio (WhatsApp delivery)
- Each member receives a personalized digest based on event visibility rules
- Events without confirmed participants may be excluded from the digest
````

---

## 7. Validation Rules

The `codika verify use-case` command checks skills automatically. The validation rules are:

| Rule | Severity | What it checks |
|------|----------|----------------|
| `SKILL-STRUCTURE` | must | Every subdirectory in `skills/` contains a `SKILL.md` file |
| `SKILL-FRONTMATTER` | must | `SKILL.md` has valid YAML frontmatter with required fields |
| `SKILL-NAME-FORMAT` | must | `name` is max 64 chars, lowercase+numbers+hyphens, no reserved words |
| `SKILL-DESCRIPTION` | should | `description` is non-empty, max 1024 chars |
| `SKILL-WORKFLOW-REF` | must | `workflowTemplateId` matches an existing workflow file in `workflows/` |
| `SKILL-DUPLICATE` | must | No duplicate `name` or `workflowTemplateId` across skills |

**The skills folder is optional.** If no `skills/` directory exists, no validation errors are raised. This maintains backward compatibility with existing use cases.

---

## 8. Deployment & Retrieval

### Deploying skills

Skills are automatically collected during `codika deploy use-case`. The CLI:

1. Scans `skills/*/SKILL.md` in the use case directory
2. Parses frontmatter and validates each skill
3. Sends skill documents as part of the deployment request
4. Skills are stored on the `ProcessDeploymentInstance` in Firestore

No changes to `config.ts` are needed. Skills are discovered from the filesystem.

### Downloading skills

```bash
# From inside a use case folder (resolves processInstanceId from project.json)
codika get skills

# With explicit process instance ID
codika get skills <processInstanceId>

# Save to a specific directory
codika get skills --output .claude/skills

# JSON output (for scripting)
codika get skills --json

# Print to stdout
codika get skills --stdout
```

Downloaded skills are written as proper Claude-compatible directories:
```
./skills/
├── wat-direct-messaging/
│   └── SKILL.md
├── wat-test-bot/
│   └── SKILL.md
└── wat-event-weekly-digest/
    └── SKILL.md
```

### Using downloaded skills

**With Claude Code:** Copy to `.claude/skills/` in any project. Claude Code auto-discovers them.

```bash
codika get skills --output .claude/skills
```

**With the Claude API:** Upload via the Skills API.

```python
from anthropic.lib import files_from_dir

skill = client.beta.skills.create(
    display_title="Direct Messaging",
    files=files_from_dir("./skills/wat-direct-messaging"),
    betas=["skills-2025-10-02"],
)
```

---

## 9. Best Practices

### Content

- **Be concise.** Under 500 lines per SKILL.md. Claude is smart — only explain what it can't infer.
- **Show exact payloads.** Include real `codika trigger` commands with copy-pasteable JSON.
- **Document both input and output.** Tables for input params, JSON examples for output.
- **Mention integrations.** "Uses Salesforce and Google Sheets" helps agents understand dependencies.
- **Include cost.** Agents should know the credit cost before triggering expensive workflows.

### Naming

- **Prefix with use case slug:** `wat-direct-messaging`, `propale-generate-proposal`
- **Use hyphens:** `my-skill-name` not `my_skill_name`
- **Be specific:** `wat-event-reminder-today` not `reminder`
- **Max 64 characters** for the `name` field

### Descriptions

- **Third person:** "Sends a message to..." not "Use this to send..."
- **Include what AND when:** Both what the skill does and when an agent should use it
- **Max 1024 characters**

### For scheduled workflows

- **Always explain the automatic schedule** in the body
- **Always show the manual trigger command** so agents can test
- **State that no payload is required** for manual triggers (unless one is needed)

### Progressive disclosure

For complex skills, keep SKILL.md concise and add reference files:

```
my-complex-skill/
├── SKILL.md              # Overview + trigger command
├── INPUT-REFERENCE.md    # Detailed input schema with all options
└── EXAMPLES.md           # Multiple usage examples
```

Claude reads SKILL.md first, then loads additional files only when needed.

---

## 10. Checklist

Before deploying a use case with skills:

- [ ] One skill per triggerable workflow (HTTP + scheduled with manual trigger)
- [ ] No skills for sub-workflows or data ingestion workflows
- [ ] Each skill is a directory with a `SKILL.md` file
- [ ] Frontmatter has `name`, `description`, and `workflowTemplateId`
- [ ] `name` follows Claude constraints (lowercase, hyphens, max 64 chars, no reserved words)
- [ ] `description` is third person, non-empty, max 1024 chars
- [ ] `workflowTemplateId` matches an existing workflow in `workflows/`
- [ ] Body includes "How to trigger" with a `codika trigger` command
- [ ] Body includes input schema (table or JSON) for HTTP workflows
- [ ] Body includes output example (JSON) for HTTP workflows
- [ ] `codika verify use-case .` passes with no skill-related errors
