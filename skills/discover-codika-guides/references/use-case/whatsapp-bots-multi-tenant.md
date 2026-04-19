# WhatsApp Bots — Multi-Tenant Pattern (Orchestrator + Per-Client Process)

This guide teaches the architecture used by `satellites/codika-bot-factory` and `satellites/codika-client-bots`. **Read this whole guide before writing any workflow JSON.** It is intentionally self-contained — every detail a reading agent needs is inline rather than behind a link.

If you're not sure which pattern you want, read `whatsapp-bots.md` first. Use this pattern when you're running WhatsApp bots for multiple distinct client organizations that must have isolated credentials + isolated Postgres schemas.

---

## 1. When to use this pattern

- You serve **multiple client organizations** (e.g. Creafid, Acme Corp, Pod Community), each with their own data, their own AI agent prompts, and their own per-client schema.
- One shared **Twilio WhatsApp number** (you don't want to provision one number per client).
- You route inbound messages by **sender phone → bot_id → subworkflow_id** using a single lookup table.
- You deploy each client bot as a **separate Codika process**, so their credentials and workflow code stay isolated.
- You have a **dashboard** where non-engineers can CRUD bots and phone mappings without editing code.

If instead you have one org with multiple user roles (admin/member/guest), read `whatsapp-bots-single-tenant.md`.

---

## 2. Architecture at a glance

```
                   WhatsApp user (any client)
                             │
                             ▼
                    Twilio +14436478971
                             │
                             │  com.twilio.messaging.inbound-message.received
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │  ORCHESTRATOR PROCESS (one per product)                  │
  │                                                          │
  │   main-router.json                                       │
  │    ├─ Twilio Trigger (service_event)                     │
  │    ├─ Codika Init                                        │
  │    ├─ Process Input    (strip `whatsapp:` + `+` prefix)  │
  │    ├─ Get Bot          (SQL JOIN: phone_mappings + bots) │
  │    ├─ Bot Found? ──NO──▶ handler-unknown (fallback)      │
  │    ├─ YES                                                │
  │    └─ Execute Workflow (workflowId = subworkflow_id)     │
  │                                │                         │
  │                                │ inputs: senderPhone,    │
  │                                │   messageBody, botId,   │
  │                                │   botPhone, channel,    │
  │                                │   sendWhatsappWorkflowId│
  │                                ▼                         │
  │                     ┌────────────────────────────────┐   │
  │                     │  PER-CLIENT PROCESS            │   │
  │                     │  (separate Codika deployment)  │   │
  │                     │                                │   │
  │                     │  creafid-handler.json          │   │
  │                     │   ├─ executeWorkflowTrigger    │   │
  │                     │   ├─ AI Agent (Claude Haiku)   │   │
  │                     │   │    ├─ Chat Memory          │   │
  │                     │   │    └─ capture_requirement  │   │
  │                     │   │         (tool sub-workflow)│   │
  │                     │   └─ returns { responseText,   │   │
  │                     │                 status }       │   │
  │                     └────────────────────────────────┘   │
  │                                │                         │
  │                                ▼ (callback with reply)   │
  │                       sub-send-whatsapp.json             │
  │                        (Twilio REST, chunked)            │
  │                                │                         │
  │    Codika Submit Result ◀──────┘                         │
  └──────────────────────────────────────────────────────────┘
                             │
                             ▼
                    WhatsApp user receives reply
```

Two things worth internalising:

- The orchestrator process **owns the `bots`, `phone_mappings`, `messages`, `n8n_chat_histories`, `whatsapp_templates` tables**. It's the single source of truth for routing.
- The per-client handler is a **sub-workflow in a different Codika process**. The orchestrator's `main-router` invokes it by its n8n workflow ID (stored in `bots.subworkflow_id`) via `Execute Workflow`. The handler calls back to the orchestrator's `sub-send-whatsapp` using the workflow ID passed in `sendWhatsappWorkflowId`.

---

## 3. Prerequisites

### Credentials

| Credential | Context | Placeholder | Used in |
|---|---|---|---|
| Twilio (SID + auth token) | organization | `{{ORGCRED_TWILIO_ID_DERCGRO}}` | main-router (trigger), sub-send-whatsapp, sub-create-system-template |
| Postgres (Neon pooled) | process_instance | `{{INSTCRED_POSTGRES_ID_DERCTSNI}}` | every DB read/write in orchestrator + handlers |
| Anthropic (Claude) | flex | `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` | per-client handler AI agents |
| OpenAI (Whisper) | flex | `{{FLEXCRED_OPENAI_ID_*}}` | sub-transcribe-audio (only if you accept voice notes) |

Set them with `codika integration set`:

```bash
codika integration set twilio \
  --context-type organization \
  --secret TWILIO_ACCOUNT_SID=ACxxxxxxxxxx... \
  --secret TWILIO_AUTH_TOKEN=xxxxxxxxxx... \
  --path . --environment dev

codika integration set postgres \
  --context-type process_instance \
  --process-instance-id <instance_id> \
  --secret POSTGRES_HOST=<pooler-host> \
  --secret POSTGRES_PORT=5432 \
  --secret POSTGRES_DATABASE=neondb \
  --secret POSTGRES_USER=neondb_owner \
  --secret POSTGRES_PASSWORD=<pwd> \
  --secret POSTGRES_SSL=require \
  --path . --environment dev
```

### Dashboard schema (Drizzle)

The dashboard defines and owns these tables via `drizzle-kit push`. Minimal shape:

```ts
// src/lib/db/schema.ts
export const bots = pgTable('bots', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  clientName: text('client_name').notNull(),
  description: text('description'),
  welcomeMessage: text('welcome_message'),
  processInstanceId: text('process_instance_id'),   // STABLE lookup key for updates
  subworkflowId: text('subworkflow_id'),             // n8n workflow ID of the handler
  status: text('status').notNull().default('inactive'),
  botPhone: text('bot_phone'),                       // informational only, not used for routing
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow()
});

export const phoneMappings = pgTable('phone_mappings', {
  id: uuid('id').primaryKey().defaultRandom(),
  phoneNumber: text('phone_number').notNull().unique(),  // stored WITHOUT leading '+'
  botId: uuid('bot_id').notNull().references(() => bots.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow()
});

// CRITICAL: id MUST be GENERATED ALWAYS AS IDENTITY.
// @n8n/n8n-nodes-langchain.memoryPostgresChat does not supply an id on insert —
// it expects Postgres to auto-generate it. Without identity, every chat-memory
// write fails with:
//   null value in column "id" of relation "n8n_chat_histories" violates not-null constraint
// and the agent falls through to its fallback handler. Keep `.generatedAlwaysAsIdentity()` forever.
export const n8nChatHistories = pgTable('n8n_chat_histories', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  sessionId: text('session_id').notNull(),
  message: jsonb('message').notNull()
});

export const whatsappTemplates = pgTable('whatsapp_templates', {
  id: uuid('id').primaryKey().defaultRandom(),
  contentSid: text('content_sid').notNull().unique(),
  templateKey: text('template_key').notNull().unique(),
  friendlyName: text('friendly_name'),
  templateBody: text('template_body').notNull(),
  category: text('category'),       // CHECK IN ('UTILITY', 'MARKETING', 'AUTHENTICATION')
  language: text('language'),
  status: text('status').notNull().default('pending'),
  rejectionReason: text('rejection_reason'),
  createdAt: timestamp('created_at').defaultNow()
});
```

Per-tenant schemas live in their OWN Postgres schemas so they don't collide, e.g. `creafid.requirements`, `acme.tickets`. Create those with plain SQL (outside of Drizzle) the first time you onboard a tenant.

### Deployment parameter

```ts
// processes/bot-orchestrator/config.ts
{
  key: 'USER_BOT_PHONE',
  type: 'string',
  label: 'Bot Phone Number',
  description: 'Twilio WhatsApp number the orchestrator receives on',
  required: true,
  regex: '^\\+[0-9]+$',
  defaultValue: '+14436478971',   // set your real default here
}
```

Every `codika deploy use-case` re-applies `defaultDeploymentParameters`. If you redeploy with `codika redeploy --param USER_BOT_PHONE=...`, a subsequent `deploy use-case` will overwrite that override. Keep the default in `config.ts` current.

---

## 4. Orchestrator `config.ts` skeleton

The minimum workflow set:

```ts
export const WORKFLOW_FILES = [
  join(__dirname, 'workflows/sub-send-whatsapp.json'),
  join(__dirname, 'workflows/sub-create-system-template.json'),
  join(__dirname, 'workflows/handler-unknown.json'),
  join(__dirname, 'workflows/main-router.json'),
  join(__dirname, 'workflows/http-manage-bot.json'),
  join(__dirname, 'workflows/http-manage-phone-mapping.json'),
  join(__dirname, 'workflows/http-test-bot.json'),
  join(__dirname, 'workflows/http-provision-templates.json'),
  join(__dirname, 'workflows/http-send-template.json'),
  join(__dirname, 'workflows/scheduled-template-status-check.json'),
];

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'Client Bots Orchestrator',
    subtitle: 'Routes WhatsApp messages to per-client bots',
    description: 'Phone → bot_id → subworkflow_id routing for WhatsApp.',
    workflows,
    tags: ['whatsapp', 'orchestrator'],
    integrationUids: ['twilio', 'anthropic', 'postgres'],
    icon: 'Bot',
    processType: 'organizational',
    deploymentInputSchema: [
      { key: 'USER_BOT_PHONE', type: 'string', required: true, regex: '^\\+[0-9]+$' },
    ],
    defaultDeploymentParameters: { USER_BOT_PHONE: '+14436478971' },
  };
}
```

Each workflow declares `integrationUids` for the credentials it uses so the platform knows what to resolve.

---

## 5. `main-router` — the inbound entry point

This is the only workflow triggered by Twilio. Node-by-node:

### 5.1 Twilio Trigger

```json
{
  "type": "n8n-nodes-base.twilioTrigger",
  "parameters": { "updates": ["com.twilio.messaging.inbound-message.received"] },
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}-main-router",
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  }
}
```

When the orchestrator's workflows activate, n8n calls Twilio's Event Streams API with these credentials to register the webhook. If your first activation happened with fake Twilio creds, you MUST cycle the instance (`codika instance deactivate <id>` then `codika instance activate <id>`) after updating the real creds — the webhook doesn't re-register automatically on credential change.

### 5.2 Codika Init

```json
{
  "type": "n8n-nodes-codika.codika",
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "{{USERDATA_ORGANIZATION_ID_ATADRESU}}",
    "userId": "{{USERDATA_USER_ID_ATADRESU}}",
    "processInstanceId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
    "workflowId": "main-router",
    "triggerType": "twilio"
  }
}
```

Required immediately after any non-HTTP trigger so the platform can bill, log, and error-report this execution.

### 5.3 Process Input (Code node)

Twilio sends phone numbers as `whatsapp:+32477047490`. Strip the prefix and `+` consistently so DB lookups match:

```js
const data = $input.item.json.data || $input.item.json;
const rawFrom = data.from || '';
const rawTo = data.to || '';
const senderPhone = rawFrom.replace(/^(whatsapp|sms):/, '').replace(/^\+/, '');  // "32477047490"
const botPhone   = rawTo.replace(/^(whatsapp|sms):/, '');                        // "+14436478971"
const orchestratorPhone = '{{INSTPARM_USER_BOT_PHONE_MRAPTSNI}}';
const isOrchestratorBot = botPhone === orchestratorPhone;

return [{
  json: {
    senderPhone, botPhone, isOrchestratorBot,
    messageBody: data.body || '',
    messageSid: data.messageSid || data.MessageSid || '',
    numMedia: parseInt(data.numMedia || data.NumMedia || '0', 10),
  }
}];
```

Phone numbers live in the DB **without** the `+`. That's an intentional convention — the orchestrator normalises at ingress once.

### 5.4 Get Bot (Postgres)

```sql
SELECT pm.phone_number,
       b.id            AS bot_id,
       b.name,
       b.subworkflow_id,
       b.status
FROM phone_mappings pm
JOIN bots b ON b.id = pm.bot_id
WHERE pm.phone_number = '{{ $('Process Input').first().json.senderPhone }}'
LIMIT 1
```

### 5.5 Bot Found? (IF)

Branches:

- **Yes, active:** continue to Execute Workflow.
- **No, or inactive:** route to `handler-unknown` which sends a fallback like "You're not registered yet — contact your account manager."

### 5.6 Execute Workflow (dynamic)

```json
{
  "type": "n8n-nodes-base.executeWorkflow",
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "={{ $('Get Bot').first().json.subworkflow_id }}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "senderPhone":           "={{ $('Process Input').first().json.senderPhone }}",
        "messageBody":           "={{ $('Process Input').first().json.messageBody }}",
        "botId":                 "={{ $('Get Bot').first().json.bot_id }}",
        "botPhone":              "={{ $('Process Input').first().json.botPhone }}",
        "sendWhatsappWorkflowId": "{{SUBWKFL_sub-send-whatsapp_LFKWBUS}}",
        "channel":               "twilio",
        "messageType":           "text",
        "mediaFileId":           "",
        "mediaContentType":      ""
      }
    },
    "options": { "waitForSubWorkflow": true }
  }
}
```

`sendWhatsappWorkflowId` is passed as an **INPUT** to the handler so the handler knows which workflow to call back for the reply. Never hardcode this inside the per-client handler — the orchestrator's sub-send-whatsapp n8n ID changes on every `deploy use-case` of the orchestrator.

### 5.7 Codika Submit Result

```json
{
  "resource": "workflowOutputs",
  "operation": "submitResult",
  "resultData": "={{ $json }}"
}
```

---

## 6. Per-client handler (in a separate Codika process)

Each client bot is its own Codika use case. The handler is a **sub-workflow** (no Codika Init, no Submit Result — the orchestrator owns those).

### 6.1 Input contract

Every handler must declare these inputs on its `executeWorkflowTrigger`:

| Input | Type | Notes |
|---|---|---|
| `senderPhone` | string | Stripped of `+` and `whatsapp:` prefix (e.g. `"32477047490"`). |
| `messageBody` | string | Raw text from the WhatsApp message. |
| `botId` | string (UUID) | The DB `bots.id`. Used as session scope. |
| `botPhone` | string | Twilio receive number, `+14436478971`. Used when sending replies. |
| `sendWhatsappWorkflowId` | string | n8n workflow ID to call back for the reply. |
| `channel` | string | `"twilio"` in production, `"http-test"` when the dashboard is testing (in which case you skip the outbound send and return responseText directly). |
| `messageType` | string | `"text"` / `"image"` / `"audio"` / … |
| `mediaFileId` | string | Optional. |
| `mediaContentType` | string | Optional. |

### 6.2 Handler skeleton

```
executeWorkflowTrigger
  └─ Parse Input (Code)           // sanity check, set defaults
      └─ AI Agent (LangChain)     // Claude Haiku / Sonnet
           ├─ ai_languageModel: Claude
           ├─ ai_memory:         Postgres Chat Memory
           └─ ai_tool*:           your tool sub-workflows
      └─ Prepare Response (Code)  // sanitize markdown to WhatsApp format
           └─ Should Send? (IF channel !== 'http-test')
                ├─ YES: Execute Workflow (by sendWhatsappWorkflowId)
                │        inputs: { twilioPhone: botPhone, senderPhone, messageText: responseText }
                └─ NO:  skip
           └─ Return Result (Code)
                returns { responseText, status: 'success' }
```

### 6.3 AI agent node

```json
{
  "type": "@n8n/n8n-nodes-langchain.agent",
  "parameters": {
    "promptType": "define",
    "text": "=Phone: {{ $json.senderPhone }}\nMessage: {{ $json.messageBody }}",
    "options": {
      "systemMessage": "You are the <Client> assistant. <role-specific prompt>..."
    }
  },
  "onError": "continueErrorOutput",
  "retryOnFail": true, "maxTries": 3
}
```

### 6.4 Chat memory (Postgres)

```json
{
  "type": "@n8n/n8n-nodes-langchain.memoryPostgresChat",
  "parameters": {
    "sessionIdType": "customKey",
    "sessionKey": "={{ $('Parse Input').first().json.botId }}:{{ $('Parse Input').first().json.senderPhone }}",
    "tableName": "n8n_chat_histories",
    "contextWindowLength": 20
  },
  "credentials": {
    "postgres": { "id": "{{INSTCRED_POSTGRES_ID_DERCTSNI}}" }
  }
}
```

**Why `botId:senderPhone` as the session key?** Because the orchestrator can re-map a phone to a different bot, and you don't want the new bot to inherit the previous bot's conversation context.

**Identity reminder:** if you're seeing the fallback reply, the first thing to check is whether `n8n_chat_histories.id` is `GENERATED ALWAYS AS IDENTITY`. `\d n8n_chat_histories` in psql — look for `is_identity: YES`. If it's not, `ALTER TABLE n8n_chat_histories ALTER COLUMN id ADD GENERATED ALWAYS AS IDENTITY;` on both branches and update your Drizzle schema with `.generatedAlwaysAsIdentity()`.

### 6.5 Tool sub-workflow (within the same per-client process)

```json
{
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "parameters": {
    "name": "capture_requirement",
    "description": "Record a client requirement. Call ONCE you have title + description + category + priority.",
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "{{SUBWKFL_sub-capture-requirement_LFKWBUS}}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "title":       "={{ $fromAI('title', 'Short title (max 8 words)', 'string') }}",
        "description": "={{ $fromAI('description', '1–3 sentence description', 'string') }}",
        "category":    "={{ $fromAI('category', 'feature_request | bug | question | other', 'string') }}",
        "priority":    "={{ $fromAI('priority', 'low | medium | high | urgent', 'string') }}",
        "caller_phone":"={{ $('Parse Input').first().json.senderPhone }}"
      }
    }
  }
}
```

### 6.6 Sending the reply — calling back the orchestrator

```json
{
  "type": "n8n-nodes-base.executeWorkflow",
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "={{ $('Parse Input').first().json.sendWhatsappWorkflowId }}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "twilioPhone": "={{ $('Parse Input').first().json.botPhone }}",
        "senderPhone": "={{ $('Parse Input').first().json.senderPhone }}",
        "messageText": "={{ $json.responseText }}"
      }
    },
    "options": { "waitForSubWorkflow": true }
  }
}
```

---

## 7. Bot + phone mapping management (from the dashboard)

Never mutate the `bots` table directly. Always go through the orchestrator's HTTP endpoints.

### 7.1 Create a bot (after the per-client process is first deployed)

```bash
codika trigger http-manage-bot --path processes/bot-orchestrator --poll --payload-file - <<'EOF'
{
  "action": "create",
  "name": "Creafid Requirements Bot",
  "client_name": "Creafid",
  "description": "Gathers change requests via WhatsApp.",
  "welcome_message": "Welcome message sent in WhatsApp invite templates",
  "process_instance_id": "<devProcessInstanceId from the client bot's project.json>",
  "subworkflow_id": "<handler n8n workflow ID from deploy output>",
  "status": "active",
  "bot_phone": "+14436478971"
}
EOF
```

`process_instance_id` is the **stable** lookup key for updates — never use the DB UUID. If you resend the same `create` payload with a different body, the handler recognizes it as an update.

### 7.2 Update bot (most common: subworkflow_id changed after a redeploy)

```bash
codika trigger http-manage-bot --path processes/bot-orchestrator --poll --payload-file - <<'EOF'
{
  "action": "update",
  "process_instance_id": "<devProcessInstanceId>",
  "subworkflow_id": "<NEW handler n8n workflow ID>"
}
EOF
```

### 7.3 Add a phone mapping

```bash
codika trigger http-manage-phone-mapping --path processes/bot-orchestrator --poll --payload-file - <<'EOF'
{
  "action": "add",
  "phone_number": "+32477047490",
  "bot_id": "<bot id returned from create>"
}
EOF
```

### 7.4 `deploy.sh` pattern (per-client, runs in each client's folder)

```bash
#!/usr/bin/env bash
# processes/bots/<client>/deploy.sh
# Deploy the client bot and auto-update subworkflow_id via the orchestrator.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ORCHESTRATOR_DIR="$(cd "$SCRIPT_DIR/../../bot-orchestrator" && pwd)"
PROCESS_INSTANCE_ID="<fixed devProcessInstanceId of THIS client bot>"

DEPLOY_OUTPUT=$(codika deploy use-case "$SCRIPT_DIR" ${1:-} 2>&1)
echo "$DEPLOY_OUTPUT"

HANDLER_ID=$(echo "$DEPLOY_OUTPUT" | grep "<client>-handler" | grep -oE '\(n8n: [A-Za-z0-9]+\)' | grep -oE '[A-Za-z0-9]{10,}')

[ -z "$HANDLER_ID" ] && { echo "ERROR: could not parse handler workflow ID"; exit 1; }

codika trigger http-manage-bot --path "$ORCHESTRATOR_DIR" --poll --payload-file - <<EOF
{ "action": "update", "process_instance_id": "$PROCESS_INSTANCE_ID", "subworkflow_id": "$HANDLER_ID" }
EOF
```

Why this exists: every `codika deploy use-case` generates **fresh n8n workflow IDs**. If you don't update the `bots` table, the orchestrator will try to call a deactivated workflow and the user will just see silence.

---

## 8. `sub-send-whatsapp` — the outbound workflow

One workflow, called by every handler via `sendWhatsappWorkflowId`. Inputs: `twilioPhone`, `senderPhone`, `messageText`.

### 8.1 Message chunking

WhatsApp accepts up to 1600 chars per message; Twilio charges per segment. Split long responses:

```js
// inside a Code node
const text = $input.item.json.messageText || '';
const MAX = 1500;
const chunks = [];
for (let i = 0; i < text.length; i += MAX) chunks.push(text.slice(i, i + MAX));
return chunks.map(c => ({ json: { chunk: c } }));
```

Then loop over `$items()` with a Twilio HTTP Request node (Twilio's v2 API endpoint, auth via the `twilioApi` credential):

```
POST https://api.twilio.com/2010-04-01/Accounts/{{ORGCRED_TWILIO_ACCOUNT_SID}}/Messages.json
  From: whatsapp:{{ $('Parse Input').first().json.twilioPhone }}
  To:   whatsapp:+{{ $('Parse Input').first().json.senderPhone }}
  Body: {{ $json.chunk }}
```

### 8.2 WhatsApp formatting

WhatsApp's markdown is **not** GitHub-flavored. Sanitize in the handler's "Prepare Response" node before the reply hits `sub-send-whatsapp`:

| Intent | GitHub / common | WhatsApp |
|---|---|---|
| Bold | `**bold**` | `*bold*` |
| Italic | `*italic*` or `_italic_` | `_italic_` |
| Strikethrough | `~~text~~` | `~text~` |
| Monospace | `` `code` `` | `` ```code``` `` |
| Bullet list | `- item` | `* item` or `- item` (both work) |
| Headings | `# Heading` | *not supported — remove* |
| Links | `[label](url)` | *plain URL only — Meta auto-linkifies* |

Minimum sanitizer:

```js
responseText = responseText
  .replace(/\*\*([^*]+)\*\*/g, '*$1*')          // **bold** → *bold*
  .replace(/^#{1,6}\s+/gm, '')                  // strip # headers
  .replace(/\[([^\]]+)\]\((https?:[^)]+)\)/g, '$1 $2');   // [label](url) → label url
```

---

## 9. WhatsApp template provisioning (outbound-initiated conversations)

**Why templates matter:** once a user has messaged you, you have a 24-hour "conversation window" to reply freely. Outside that window (or when YOU initiate the conversation), you must send a **Meta-approved template**. Templates live as Twilio Content Templates plus an ApprovalRequest; they're what you use for welcome messages, reminders, re-engagement, etc.

### 9.1 Meta categories — and why the choice matters

Meta requires every template to declare a category. Get this right or lose money.

| Category | When to use | Price impact | Examples |
|---|---|---|---|
| **UTILITY** | Confirmation of something the user did / account or transaction info | Cheap | "Your order shipped." "You've been added to Project X." "Your code is 1234." |
| **MARKETING** | Invitations, re-engagement, informational messages not tied to a specific user action | **~5× more expensive**, global opt-outs, stricter delivery caps | "Check out our new offer!" "We miss you." "Here's what's new this month." |
| **AUTHENTICATION** | OTPs only | Cheapest; must follow a strict format | "Your verification code is {{1}}." |

Meta auto-classifies. A "welcome" template is almost always scored MARKETING unless you frame it as **confirmation of an action that just occurred**. Real incident from `codika-client-bots`:

- FAIL — Meta silently flipped to MARKETING:
  ```
  Hi {{1}}! I'm the Codika Assistant for {{2}}. Send change requests, bug reports,
  or questions here anytime — the team is still reachable directly too.
  ```
  (Invitational greeting, "anytime", "reachable too" → Meta reads "marketing".)

- PASS — approved as UTILITY:
  ```
  You've been added to {{2}} on Codika, {{1}}. This is your direct line for change
  requests, bug reports, or questions about the project.
  ```
  (Opens with the action that just happened, transactional tone, no invitational language.)

### 9.2 Meta variable rule — variables can't be first or last

Meta will reject with `code=2388299, userMessage="Variables can't be at the start or end of the template."` if the body opens or closes with `{{n}}`. Fix by adding text either side:

- BAD: `{{1}}, tu as été ajouté à {{2}}…`
- GOOD: `Bonjour {{1}}, tu as été ajouté à {{2}} sur Codika…`

### 9.3 `allow_category_change: false` — the load-bearing flag

If you submit a template with `allow_category_change: true` (Twilio's default), Meta can silently reclassify your UTILITY template to MARKETING and you'll never know until your bill shows up. Always set it to `false` so Meta must either approve UTILITY as stated or reject outright.

Inside `sub-create-system-template`, the HTTP Request node that hits Twilio's ApprovalRequests endpoint:

```json
{
  "method": "POST",
  "url": "=https://content.twilio.com/v1/Content/{{ $json.contentSid }}/ApprovalRequests",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "twilioApi",
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "={{ JSON.stringify({ name: $json.friendlyName, category: $json.category, allow_category_change: false }) }}"
}
```

### 9.4 The provisioning trio

Three workflows do all the work:

1. **`http-provision-templates`** (HTTP trigger, dashboard calls it) — holds a `SYSTEM_TEMPLATES` array in a Code node. Diffs each entry against the `whatsapp_templates` table. For new or body-changed entries, calls `sub-create-system-template`.

   ```js
   const SYSTEM_TEMPLATES = [
     {
       template_key: 'welcome_en',
       friendly_name: 'welcome_en',
       template_body: "You've been added to {{2}} on Codika, {{1}}. This is your direct line for change requests, bug reports, or questions about the project.",
       variables: { "1": "Sophie", "2": "Creafid" },
       category: 'UTILITY',
       language: 'en',
       admin_intent: 'Welcome a new client user'
     },
     {
       template_key: 'welcome_fr',
       friendly_name: 'welcome_fr',
       template_body: "Bonjour {{1}}, tu as été ajouté à {{2}} sur Codika. Voici ton canal direct pour les demandes d'évolutions, bugs ou questions sur le projet.",
       variables: { "1": "Sophie", "2": "Creafid" },
       category: 'UTILITY',
       language: 'fr',
       admin_intent: 'Welcome a new client user (FR)'
     },
   ];
   ```

2. **`sub-create-system-template`** (sub-workflow) — creates the Content Template at Twilio (`POST /v1/Content`), then submits it for Meta approval (`POST /v1/Content/{sid}/ApprovalRequests` with `allow_category_change: false`), then writes the row to `whatsapp_templates` with `status = 'submitted_for_review'`.

3. **`scheduled-template-status-check`** (schedule trigger, daily cron `0 8 * * *`) — for every row still in `submitted_for_review` or `pending`, calls `GET /v1/Content/{sid}/ApprovalRequests` and maps Twilio's status back to one of `pending | approved | rejected`, plus writes the rejection reason if any. You can force a poll manually: `codika trigger scheduled-template-status-check --path . --poll --payload-file - <<< '{}'`.

### 9.5 Adding a new template

1. Edit the `SYSTEM_TEMPLATES` array in `http-provision-templates.json`. Follow the Meta variable rule and pick a category carefully (see 9.1).
2. `codika deploy use-case processes/bot-orchestrator --patch` — deploys the new workflow content. (n8n IDs change but nothing downstream references them.)
3. `codika trigger http-provision-templates --path . --poll --payload-file - <<< '{}'` — submits the new/changed templates to Twilio.
4. Either wait for tomorrow's 06:00 UTC cron, or trigger `scheduled-template-status-check` manually 30 min later to pick up Meta's verdict.

### 9.6 If Meta rejects

Common rejections and fixes:

| Reason | Fix |
|---|---|
| `code=2388299 Variables can't be at the start or end of the template` | Add text before `{{1}}` or after the last variable. |
| `code=2388022 Invalid parameter content` | Usually whitespace / newlines inside `{{n}}` or outside template_body; strip them. |
| Category silently flipped to MARKETING | Body too invitational. Reframe as confirmation of a user action. Also ensure `allow_category_change: false`. |
| "Content format violation" | Remove emojis in certain positions, remove URLs that look promotional, remove multiple `!` in a row. |

Delete the rejected row from `whatsapp_templates` and re-trigger `http-provision-templates` with revised copy — the diff logic will treat it as a fresh submission.

---

## 10. Testing

### 10.1 `http-test-bot` (bypasses Twilio)

The orchestrator's `http-test-bot` takes a phone + message body, runs the full routing + handler flow but sets `channel: "http-test"` so the handler returns `responseText` inline instead of sending over Twilio. Perfect for dashboard test UX and E2E smoke tests.

```bash
codika trigger http-test-bot --path processes/bot-orchestrator --poll --payload-file - <<'EOF'
{ "phone_number": "+32477047490", "message_body": "Hello" }
EOF
```

### 10.2 Triggering individual workflows

Any HTTP-triggered workflow in the orchestrator can be tested directly:

```bash
codika trigger http-manage-bot --path . --poll --payload-file - <<< '{"action":"list"}'
```

### 10.3 Inspecting nested traces

The orchestrator's `main-router` execution shows success even when the per-client handler errored internally. Use `--deep` to walk sub-workflow traces:

```bash
codika get execution <exec_id> --deep --slim --json
```

Look for `executionStatus: "error"` inside `_subExecutions.<nodeName>.data.resultData.runData.<nodeName>.N.error`.

---

## 11. WhatsApp-specific common errors

| Symptom | Root cause | Fix |
|---|---|---|
| Agent always replies with the fallback error message; executions show `null value in column "id" of relation "n8n_chat_histories"` deep in the trace | `n8n_chat_histories.id` is `integer NOT NULL` with no sequence | `ALTER TABLE n8n_chat_histories ALTER COLUMN id ADD GENERATED ALWAYS AS IDENTITY;` on both branches. Add `.generatedAlwaysAsIdentity()` in Drizzle. |
| Welcome template shows `category: "MARKETING"` in the DB after the poll | Submitted with `allow_category_change: true` (the default) and Meta auto-reclassified | Patch `sub-create-system-template` to send `allow_category_change: false`. Delete the MARKETING row, rewrite copy as UTILITY-compatible, re-provision. |
| Template rejected: `Variables can't be at the start or end of the template` (`code=2388299`) | Body begins or ends with `{{n}}` | Prefix/suffix with text (e.g. `Bonjour {{1}}, …` or `…{{2}}.`). |
| User sends a message and gets nothing back; `codika list executions` shows zero `main-router` runs | Twilio webhook was registered with fake creds, or credentials changed after activation | `codika instance deactivate <id>` then `codika instance activate <id>`. This re-registers Twilio Event Streams with current creds. |
| `isOrchestratorBot` always false inside main-router | `USER_BOT_PHONE` deployment parameter doesn't match the number Twilio is sending `to` | Update `defaultDeploymentParameters.USER_BOT_PHONE` in `config.ts` and redeploy, OR `codika redeploy --path . --param USER_BOT_PHONE=<real-number> --force`. |
| Outbound send fails with Twilio error `63016 Failed to send freeform message because you are outside the allowed window` | 24-hour conversation window has expired; you're trying to send non-template content to a cold user | Send an approved UTILITY template via `http-send-template` first; resume freeform only after the user replies. |
| Handler response reaches Twilio but user sees truncated text | Message >1600 chars and not chunked | Add the chunking loop from §8.1. |

See `post-creation/common-errors.md` for non-WhatsApp-specific n8n errors.

---

## 12. Reference implementations

- **`satellites/codika-client-bots/processes/bot-orchestrator/`** — current, minimal orchestrator. 14 workflows. Read its `config.ts` + the workflows listed in §4 for concrete examples of every snippet in this guide.
- **`satellites/codika-client-bots/processes/bots/creafid/`** — the first per-client handler built on this pattern. Handler + one tool sub-workflow (`sub-capture-requirement`) that INSERTs into `creafid.requirements`. Good starting shape for a new client.
- **`satellites/codika-bot-factory/`** — the progenitor. Bigger workflow set (media handling, Baileys support, more tooling) — useful as a reference but heavier than a new project needs.
- **Dashboard**: `satellites/codika-client-bots/src/` — SvelteKit + Drizzle + Better Auth + Neon. Shows how to wire the orchestrator's endpoints into admin UI.

## 13. When to cross this with the single-tenant pattern

If a single client has internal role distinctions (e.g. Creafid's admins see more tools than Creafid's members), **keep that inside the client's handler process** — a mini single-tenant router + role-specific AI agents within a single Codika deployment. The orchestrator only cares about phone → bot; what the bot does internally is its own business.

If you hit the opposite: multiple truly separate communities sharing one Twilio number, reach for this multi-tenant pattern. Don't try to fold multi-tenant routing into a single process.
