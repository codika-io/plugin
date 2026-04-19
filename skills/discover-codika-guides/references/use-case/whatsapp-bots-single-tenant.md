# WhatsApp Bots — Single-Tenant Pattern (One Process, Multiple Role-Based AI Agents)

This guide teaches the architecture used by `satellites/whatsapp-bots/instances/wat-dashboard/processes/wat-bot`. **Read this whole guide before writing any workflow JSON.** It is intentionally self-contained — every detail a reading agent needs is inline rather than behind a link.

If you're not sure which pattern you want, read `whatsapp-bots.md` first. Use this pattern when **one** organization / community has multiple user roles (admin, member, guest, …), each with their own system prompt and tool allowlist, but sharing a single DB and credential set.

---

## 1. When to use this pattern

- **One community / org.** Not multiple isolated clients.
- **Multiple user types** that need different AI agent behavior: e.g. admins can run privileged tools, residents can query events, community guests get a read-only assistant.
- **Shared Postgres schema** across every role (one members table, one events table, etc.).
- **One Twilio WhatsApp number** for the whole community.
- You want a **simpler ops story** than multi-tenant — one process to deploy, one set of credentials, one execution log to monitor.

"Multiple agents" in this pattern means **multiple AI agent nodes in role-specific handler sub-workflows within the same Codika process**. It does NOT mean multiple Codika processes — that's the multi-tenant pattern. If you need tenant isolation, read `whatsapp-bots-multi-tenant.md` instead.

---

## 2. Architecture at a glance

```
                   WhatsApp user (any role)
                             │
                             ▼
                    Twilio +XXXXXXXXXXX
                             │
                             │  com.twilio.messaging.inbound-message.received
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │  SINGLE CODIKA PROCESS (everything below lives here)     │
  │                                                          │
  │   main-router.json                                       │
  │    ├─ Twilio Trigger (service_event)                     │
  │    ├─ Codika Init                                        │
  │    ├─ Process Input  (strip `whatsapp:`, normalise phone)│
  │    ├─ Match User     (SELECT user + role by phone)       │
  │    ├─ Switch on role:                                    │
  │    │   ├─ admin       → Execute Workflow (handler-admin) │
  │    │   ├─ resident    → Execute Workflow (handler-resident)
  │    │   ├─ community   → Execute Workflow (handler-community)
  │    │   ├─ onboarding  → Execute Workflow (handler-onboarding) — user found but incomplete
  │    │   └─ not-found   → Execute Workflow (handler-not-found) — registration path
  │    └─ Codika Submit Result                               │
  │                                                          │
  │   handler-<role>.json (x5 or so)                         │
  │    ├─ executeWorkflowTrigger                             │
  │    ├─ AI Agent — role-specific system prompt             │
  │    │   ├─ ai_languageModel: Claude                       │
  │    │   ├─ ai_memory:         Postgres Chat Memory        │
  │    │   └─ ai_tool*:          role-scoped tool allowlist  │
  │    └─ Execute Workflow → sub-send-whatsapp               │
  │                                                          │
  │   tool-*.json (many)                                     │
  │    Each is a tiny sub-workflow that reads/writes DB      │
  │    and returns structured data to the agent.             │
  │                                                          │
  │   sub-send-whatsapp.json                                 │
  │    Twilio REST send, chunked.                            │
  │                                                          │
  │   http-provision-templates.json  +  sub-create-system-template.json
  │    +  scheduled-template-status-check.json               │
  │    (template approval pipeline)                          │
  └──────────────────────────────────────────────────────────┘
```

WAT's production deployment has 59 workflows: 2 routers (Twilio + Baileys), 5 role handlers, 30+ tool sub-workflows, a handful of scheduled jobs, plus the template pipeline. A minimal first cut can start with 10–15 workflows — one router, one or two handlers, three or four tools, send, template trio — and grow from there.

---

## 3. Prerequisites

### Credentials

| Credential | Context | Placeholder | Used in |
|---|---|---|---|
| Twilio (SID + auth token) | organization | `{{ORGCRED_TWILIO_ID_DERCGRO}}` | main-router (trigger), sub-send-whatsapp, sub-create-system-template |
| Postgres (Neon pooled) | process_instance | `{{INSTCRED_POSTGRES_ID_DERCTSNI}}` | user lookup in router, chat memory in handlers, tool sub-workflows |
| Anthropic (Claude) | flex | `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` | role handlers' AI agents |
| OpenAI (Whisper) | flex | `{{FLEXCRED_OPENAI_ID_*}}` | sub-transcribe-audio (voice messages) |

```bash
codika integration set twilio \
  --context-type organization \
  --secret TWILIO_ACCOUNT_SID=ACxxx... \
  --secret TWILIO_AUTH_TOKEN=xxx... \
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

Simpler than the multi-tenant pattern because there's no `bots` / `phone_mappings` routing table. Typical starting shape:

```ts
// src/lib/db/schema.ts
export const members = pgTable('members', {
  id: uuid('id').primaryKey().defaultRandom(),
  phoneNumber: text('phone_number').notNull().unique(),  // stored WITH leading '+'
  displayName: text('display_name'),
  role: text('role').notNull(),                           // 'admin' | 'resident' | 'community' | 'onboarding'
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at').defaultNow(),
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
  category: text('category'),        // CHECK IN ('UTILITY', 'MARKETING', 'AUTHENTICATION')
  language: text('language'),
  status: text('status').notNull().default('pending'),
  rejectionReason: text('rejection_reason'),
  createdAt: timestamp('created_at').defaultNow()
});

// Plus whatever domain-specific tables your community needs (events, guest_passes, ...)
```

**Note on phone format:** in a single-tenant bot it's fine to store phones WITH the leading `+` (WAT does this — session keys and DB lookups align cleanly). In the multi-tenant pattern the convention is WITHOUT `+` because of legacy reasons. Pick one and stay consistent.

### Deployment parameters

A single-tenant bot typically needs one or more deployment params. WAT uses:

```ts
deploymentInputSchema: [
  { key: 'USER_BOT_PHONE', type: 'string', required: true, regex: '^\\+[0-9]+$',
    description: 'Twilio WhatsApp number the bot receives on' },
  { key: 'COMMUNITY_NAME', type: 'string', required: true,
    description: 'Human-readable community name used in prompts + templates' },
  { key: 'VOICE_ENABLED', type: 'boolean', defaultValue: 'false',
    description: 'Accept voice messages (requires OpenAI Whisper integration)' },
  // …optionally Baileys / Evolution API params if you need the non-Twilio path
],
defaultDeploymentParameters: {
  USER_BOT_PHONE: '+14436478971',
  COMMUNITY_NAME: 'My Community',
  VOICE_ENABLED: 'false',
}
```

Every `codika deploy use-case` re-applies `defaultDeploymentParameters`. Keep defaults in `config.ts` current, not in ad-hoc `codika redeploy --param` calls.

---

## 4. `config.ts` skeleton

```ts
export const WORKFLOW_FILES = [
  // leaf helpers first (so placeholder resolution finds them)
  join(__dirname, 'workflows/sub-send-whatsapp.json'),
  join(__dirname, 'workflows/sub-create-system-template.json'),
  // tools (extend freely)
  join(__dirname, 'workflows/tool-query-members.json'),
  join(__dirname, 'workflows/tool-list-events.json'),
  join(__dirname, 'workflows/tool-register-member.json'),
  // handlers (role-specific)
  join(__dirname, 'workflows/handler-admin.json'),
  join(__dirname, 'workflows/handler-resident.json'),
  join(__dirname, 'workflows/handler-community.json'),
  join(__dirname, 'workflows/handler-onboarding.json'),
  join(__dirname, 'workflows/handler-not-found.json'),
  // entry point (last, so all downstream SUBWKFL placeholders resolve)
  join(__dirname, 'workflows/main-router.json'),
  // management + templates
  join(__dirname, 'workflows/http-test-bot.json'),
  join(__dirname, 'workflows/http-provision-templates.json'),
  join(__dirname, 'workflows/scheduled-template-status-check.json'),
];

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'My Community WhatsApp Bot',
    subtitle: 'Role-based assistant for members',
    description: 'Multiple role-scoped AI agents sharing one process and DB.',
    workflows,
    tags: ['whatsapp', 'community'],
    integrationUids: ['twilio', 'anthropic', 'postgres'],
    icon: 'MessageCircle',
    processType: 'organizational',
    deploymentInputSchema: [...],
    defaultDeploymentParameters: {...},
  };
}
```

Each workflow declares its own `integrationUids`, `triggers`, and input/output schemas — see the per-workflow sections below.

---

## 5. `main-router` — the inbound entry point

Single workflow, Twilio-triggered, does phone parsing + user lookup + role dispatch. No AI here.

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

n8n auto-registers the Twilio Event Streams subscription on workflow activation using the credentials above. If your first activation happened with placeholder Twilio creds, **cycle the instance** after swapping to real ones: `codika instance deactivate <id>` then `codika instance activate <id>`. Credentials alone do not re-register the webhook.

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

Required immediately after any non-HTTP trigger so Codika can track, bill and error-report the execution.

### 5.3 Process Input (Code)

Normalise the Twilio event into a clean object every downstream node can rely on:

```js
const data = $input.item.json.data || $input.item.json;
const rawFrom = data.from || '';
const rawTo = data.to || '';
const senderPhone = rawFrom.replace(/^(whatsapp|sms):/, '');   // KEEP '+' — WAT convention
const botPhone   = rawTo.replace(/^(whatsapp|sms):/, '');
const orchestratorPhone = '{{INSTPARM_USER_BOT_PHONE_MRAPTSNI}}';

return [{
  json: {
    senderPhone, botPhone,
    isForUs: botPhone === orchestratorPhone,
    messageBody: data.body || '',
    messageSid: data.messageSid || data.MessageSid || '',
    numMedia: parseInt(data.numMedia || data.NumMedia || '0', 10),
    isWhatsApp: rawFrom.startsWith('whatsapp:'),
  }
}];
```

### 5.4 Match User (Postgres)

```sql
SELECT id, display_name, role, is_active
FROM members
WHERE phone_number = '{{ $('Process Input').first().json.senderPhone }}'
  AND is_active = TRUE
LIMIT 1
```

### 5.5 Switch on role (n8n Switch or chained IF)

```
┌─ role = 'admin'       ──▶ Execute Workflow: handler-admin
├─ role = 'resident'    ──▶ Execute Workflow: handler-resident
├─ role = 'community'   ──▶ Execute Workflow: handler-community
├─ role = 'onboarding'  ──▶ Execute Workflow: handler-onboarding
└─ no row found         ──▶ Execute Workflow: handler-not-found
```

Each Execute Workflow passes the same input shape:

```json
{
  "workflowId": { "__rl": true, "mode": "id", "value": "{{SUBWKFL_handler-admin_LFKWBUS}}" },
  "workflowInputs": {
    "mappingMode": "defineBelow",
    "value": {
      "senderPhone":           "={{ $('Process Input').first().json.senderPhone }}",
      "messageBody":           "={{ $('Process Input').first().json.messageBody }}",
      "userId":                "={{ $('Match User').first().json.id }}",
      "userRole":              "={{ $('Match User').first().json.role }}",
      "botPhone":              "={{ $('Process Input').first().json.botPhone }}",
      "sendWhatsappWorkflowId":"{{SUBWKFL_sub-send-whatsapp_LFKWBUS}}",
      "channel":               "twilio",
      "messageType":           "text",
      "mediaFileId":           "",
      "mediaContentType":      ""
    }
  },
  "options": { "waitForSubWorkflow": true }
}
```

### 5.6 Codika Submit Result

```json
{ "resource": "workflowOutputs", "operation": "submitResult", "resultData": "={{ $json }}" }
```

---

## 6. Role handlers (sub-workflows within the same process)

One handler per role. Structure is identical across roles; only the **system prompt** and the **set of tools wired into the agent** changes.

### 6.1 Input contract

```
senderPhone (string)      - sender with '+' prefix
messageBody (string)      - message text
userId (uuid/string)      - members.id, used as session scope root
userRole (string)         - 'admin' | 'resident' | ...
botPhone (string)         - Twilio receive number, '+14436478971'
sendWhatsappWorkflowId (string)  - n8n workflow ID for sub-send-whatsapp
channel (string)          - 'twilio' in prod, 'http-test' when testing
messageType (string)      - 'text' / 'image' / 'audio'
mediaFileId, mediaContentType (string)
```

### 6.2 Handler skeleton

```
executeWorkflowTrigger
  └─ Parse Input (Code)                   // sanity defaults
      └─ AI Agent (LangChain)             // ROLE-SPECIFIC system prompt
           ├─ ai_languageModel: Claude Haiku/Sonnet
           ├─ ai_memory:        Postgres Chat Memory
           └─ ai_tool*:         tool-query-members, tool-list-events, …  (role-allowlist)
      └─ Prepare Response (Code)          // sanitize markdown → WhatsApp syntax
           └─ Should Send? (IF channel !== 'http-test')
                ├─ YES: Execute Workflow (sendWhatsappWorkflowId)
                │        inputs: { twilioPhone: botPhone, senderPhone, messageText: responseText }
                └─ NO:  skip
           └─ Return Result (Code)
                returns { responseText, status: 'success' }
```

### 6.3 Role-specific system prompt (example: admin)

```
You are the <COMMUNITY_NAME> admin assistant. The current user is an admin
with full access. You have tools to query members, list and edit events,
approve guest passes, and manage invitations.

Guidelines:
- Be concise. WhatsApp replies should be 1–3 sentences unless listing items.
- For destructive actions (delete, deactivate), always confirm first.
- When listing items, format as *bold header* followed by _italic detail_.
- Never fabricate IDs or data. Only reference what tools returned.
- WhatsApp formatting: *bold* and _italic_ only. No **double asterisks**,
  no # headers, no Markdown links — plain URLs only.
```

Community / resident / onboarding handlers use the same skeleton but narrower prompts and a reduced tool list. `handler-not-found` skips the AI agent entirely and just sends a `welcome_en`/`welcome_fr` template response with a join URL.

### 6.4 Chat memory

```json
{
  "type": "@n8n/n8n-nodes-langchain.memoryPostgresChat",
  "parameters": {
    "sessionIdType": "customKey",
    "sessionKey": "={{ $('Parse Input').first().json.senderPhone }}",
    "tableName": "n8n_chat_histories",
    "contextWindowLength": 20
  },
  "credentials": { "postgres": { "id": "{{INSTCRED_POSTGRES_ID_DERCTSNI}}" } }
}
```

Session key is **`{+senderPhone}`** — simpler than the multi-tenant pattern because there's only one bot per phone. If you ever add a second independent bot in this same process (e.g. a health bot AND a community bot sharing a number, differentiated by keyword), key by `bot_name:senderPhone` instead.

**Identity reminder:** if you're seeing the fallback reply, first check `\d n8n_chat_histories` for `is_identity: YES`. If missing, `ALTER TABLE n8n_chat_histories ALTER COLUMN id ADD GENERATED ALWAYS AS IDENTITY;` on both branches and add `.generatedAlwaysAsIdentity()` in Drizzle.

### 6.5 Model + tool wiring

```json
// AI language model
{
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "parameters": {
    "model": { "__rl": true, "mode": "list", "value": "claude-haiku-4-5-20251001" },
    "options": { "maxTokensToSample": 1024, "temperature": 0.3 }
  },
  "credentials": { "anthropicApi": { "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}" } }
}

// Each tool — only include in the handlers where that tool is allowed
{
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "parameters": {
    "name": "query_members",
    "description": "Search members by name or phone. Returns display_name + role.",
    "workflowId": { "__rl": true, "mode": "id", "value": "{{SUBWKFL_tool-query-members_LFKWBUS}}" },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "query":   "={{ $fromAI('query', 'Name substring or phone number', 'string') }}",
        "caller_phone": "={{ $('Parse Input').first().json.senderPhone }}",
        "caller_role":  "={{ $('Parse Input').first().json.userRole }}"
      }
    }
  }
}
```

Role-filtered visibility: pass `caller_role` into every tool so the tool itself can decide what to return. Admins see PII, residents see public fields, community sees nothing sensitive.

---

## 7. Tool sub-workflows

Each tool is a tiny sub-workflow that does one thing. Triggered by `executeWorkflowTrigger` with a strongly-typed input schema.

### 7.1 Typical tool skeleton

```
executeWorkflowTrigger
  (inputs: query, caller_phone, caller_role)
  └─ Validate (Code) — fail fast on bad input
      └─ Query Postgres
          └─ Filter by caller_role (Code)
              └─ Return Result (Code)
                  returns { success: true, data: [...] }
```

### 7.2 Example: `tool-query-members`

```js
// Filter step
const role = $('executeWorkflowTrigger').first().json.caller_role;
const rows = $input.all().map(i => i.json);

const filtered = rows.map(r => {
  const base = { display_name: r.display_name, role: r.role };
  if (role === 'admin')     return { ...base, phone_number: r.phone_number, is_active: r.is_active };
  if (role === 'resident')  return base;
  /* community, onboarding */ return { display_name: r.display_name };
});

return filtered.map(f => ({ json: f }));
```

Keep tool outputs minimal — LLMs waste tokens on fields they don't need. If a tool returns 200 rows, paginate; stream 10 with a `has_more: true` flag and let the agent decide whether to call again.

### 7.3 Tool allowlists per role

In each handler, ONLY include the tool nodes that role is allowed to use. The agent can't call a tool that isn't wired. Don't rely on tool-internal role checks as the only defense — both is safer.

---

## 8. Chat memory (quick reference)

- Table: `n8n_chat_histories` (public schema).
- Session key: `{+senderPhone}` for single-bot, `{bot_name}:{+senderPhone}` if multiple bots share a process.
- Context window: 20 messages is a good default; tune based on typical conversation length.
- **`id` MUST be `GENERATED ALWAYS AS IDENTITY`.** See §3 / §6.4. Without it, every write fails silently and the agent dies.

---

## 9. Sending replies — `sub-send-whatsapp`

One workflow, called by every handler. Inputs: `twilioPhone`, `senderPhone`, `messageText`.

### 9.1 Chunking (WhatsApp limit: 1600 chars)

```js
const text = $input.item.json.messageText || '';
const MAX = 1500;
const chunks = [];
for (let i = 0; i < text.length; i += MAX) chunks.push(text.slice(i, i + MAX));
return chunks.map(c => ({ json: { chunk: c } }));
```

Then loop over `$items()` with a Twilio HTTP Request node:

```
POST https://api.twilio.com/2010-04-01/Accounts/{{ORGCRED_TWILIO_ACCOUNT_SID}}/Messages.json
  From: whatsapp:{{ $('Parse Input').first().json.twilioPhone }}
  To:   whatsapp:{{ $('Parse Input').first().json.senderPhone }}
  Body: {{ $json.chunk }}
```

### 9.2 WhatsApp formatting

WhatsApp's markdown is not GitHub-flavored. Sanitize inside each handler's "Prepare Response" node before calling `sub-send-whatsapp`:

| Intent | GitHub / common | WhatsApp |
|---|---|---|
| Bold | `**bold**` | `*bold*` |
| Italic | `*italic*` or `_italic_` | `_italic_` |
| Strikethrough | `~~text~~` | `~text~` |
| Monospace | `` `code` `` | `` ```code``` `` |
| Bullet list | `- item` | `* item` or `- item` |
| Headings | `# Heading` | *not supported — remove* |
| Links | `[label](url)` | *plain URL only — Meta auto-linkifies* |

Minimum sanitizer:

```js
responseText = responseText
  .replace(/\*\*([^*]+)\*\*/g, '*$1*')
  .replace(/^#{1,6}\s+/gm, '')
  .replace(/\[([^\]]+)\]\((https?:[^)]+)\)/g, '$1 $2');
```

---

## 10. WhatsApp template provisioning (outbound-initiated conversations)

**Why templates matter:** when a user first messages you, you have a 24-hour "conversation window" for free-form replies. Outside that window (or when YOU initiate), you must send a **Meta-approved template**. Welcome messages, event reminders, re-engagement nudges — all templates.

### 10.1 Meta categories — choose carefully

| Category | When to use | Price impact | Examples |
|---|---|---|---|
| **UTILITY** | Confirmation of a user action / account or transaction info | Cheap | "You've been added to {{2}}." "Your booking is confirmed for {{1}}." |
| **MARKETING** | Invitations, re-engagement, informational messages not tied to a specific user action | **~5× more expensive**, global opt-outs, stricter delivery caps | "Check out our new offer!" "We miss you." |
| **AUTHENTICATION** | OTPs only, strict format | Cheapest | "Your verification code is {{1}}." |

Meta auto-classifies. A "welcome" template almost always scores MARKETING unless you frame it as confirmation of an action that just occurred. Real example from the `codika-client-bots` incident:

- FAIL — Meta silently flipped to MARKETING:
  ```
  Hi {{1}}! I'm the Codika Assistant for {{2}}. Send change requests, bug reports,
  or questions here anytime — the team is still reachable directly too.
  ```

- PASS — approved as UTILITY:
  ```
  You've been added to {{2}} on Codika, {{1}}. This is your direct line for change
  requests, bug reports, or questions about the project.
  ```

### 10.2 Meta variable rule

Meta rejects templates whose body starts or ends with a `{{n}}` variable — error `code=2388299, userMessage="Variables can't be at the start or end of the template."` Fix by prefixing / suffixing with text:

- BAD:  `{{1}}, tu as été ajouté à {{2}}…`
- GOOD: `Bonjour {{1}}, tu as été ajouté à {{2}} sur Codika…`

### 10.3 `allow_category_change: false` — the load-bearing flag

If you don't set this, Twilio defaults to `true` and Meta can silently reclassify UTILITY → MARKETING. You'll pay the 5× bill with no warning. Always set it explicitly in `sub-create-system-template`:

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

Meta then either approves UTILITY as stated or rejects outright.

### 10.4 The provisioning trio

Three workflows:

1. **`http-provision-templates`** — HTTP-triggered, holds a `SYSTEM_TEMPLATES` array in a Code node. Diffs each entry against the `whatsapp_templates` table. For new or body-changed entries, calls `sub-create-system-template`. Example entry:

   ```js
   const SYSTEM_TEMPLATES = [
     {
       template_key: 'welcome_en',
       friendly_name: 'welcome_en',
       template_body: "You've been added to {{2}}, {{1}}. Message this number any time for help.",
       variables: { "1": "Sophie", "2": "Community" },
       category: 'UTILITY',
       language: 'en',
       admin_intent: 'Welcome a newly onboarded member',
     },
     {
       template_key: 'event_reminder_today',
       friendly_name: 'event_reminder_today',
       template_body: "Reminder, {{1}}: {{2}} starts today at {{3}}.",
       variables: { "1": "Alex", "2": "Monthly mixer", "3": "18:00" },
       category: 'UTILITY',
       language: 'en',
       admin_intent: 'Day-of event reminder',
     },
   ];
   ```

2. **`sub-create-system-template`** — creates the Content Template at Twilio (`POST /v1/Content`), submits it for Meta approval with `allow_category_change: false` (`POST /v1/Content/{sid}/ApprovalRequests`), writes the row to `whatsapp_templates` with `status = 'submitted_for_review'`.

3. **`scheduled-template-status-check`** — daily cron `0 8 * * *`. For every row still in `submitted_for_review` or `pending`, calls `GET /v1/Content/{sid}/ApprovalRequests`, maps Twilio's status back to `pending | approved | rejected`, writes the rejection reason if any. Force a manual poll any time:

   ```bash
   codika trigger scheduled-template-status-check --path . --poll --payload-file - <<< '{}'
   ```

### 10.5 Adding a new template

1. Edit `SYSTEM_TEMPLATES` in `http-provision-templates.json`. Follow the variable rule (10.2) and pick a category carefully (10.1).
2. `codika deploy use-case . --patch`.
3. `codika trigger http-provision-templates --path . --poll --payload-file - <<< '{}'`.
4. Wait for the cron or trigger a manual poll ~30 minutes later.

### 10.6 Handling rejections

Common rejections and fixes:

| Reason | Fix |
|---|---|
| `code=2388299 Variables can't be at the start or end of the template` | Add text before `{{1}}` or after the last variable. |
| `code=2388022 Invalid parameter content` | Whitespace / newlines inside `{{n}}` or outside the body — strip them. |
| Category silently MARKETING | Body too invitational. Reframe as confirmation of user action. Ensure `allow_category_change: false`. |
| "Content format violation" | Remove emojis in certain positions, remove URLs that look promotional, remove multiple `!` in a row. |

Delete the rejected row from `whatsapp_templates`, revise the copy, re-trigger `http-provision-templates` — the diff logic treats it as fresh.

---

## 11. Testing

### 11.1 `http-test-bot` (bypasses Twilio)

Triggers `main-router` with a hand-crafted payload and sets `channel: "http-test"` so handlers return `responseText` inline instead of sending over Twilio. Perfect for dashboard test UX.

```bash
codika trigger http-test-bot --path . --poll --payload-file - <<'EOF'
{ "phone_number": "+32477047490", "message_body": "What events are on this week?" }
EOF
```

### 11.2 Triggering individual tools

Tool sub-workflows can be called directly with `codika trigger` to validate inputs/outputs independently:

```bash
codika trigger tool-query-members --path . --poll --payload-file - <<'EOF'
{ "query": "sophie", "caller_phone": "+32477047490", "caller_role": "admin" }
EOF
```

### 11.3 Inspecting traces

`main-router` executions look successful even when a handler errored internally. Use `--deep` to walk sub-workflow traces:

```bash
codika get execution <exec_id> --deep --slim --json
```

Look for `executionStatus: "error"` inside `_subExecutions.<nodeName>.data.resultData.runData.<nodeName>.N.error`.

---

## 12. WhatsApp-specific common errors

| Symptom | Root cause | Fix |
|---|---|---|
| Agent always replies with the fallback error message; traces show `null value in column "id" of relation "n8n_chat_histories"` | `n8n_chat_histories.id` is `integer NOT NULL` without a sequence | `ALTER TABLE n8n_chat_histories ALTER COLUMN id ADD GENERATED ALWAYS AS IDENTITY;` on both branches. Add `.generatedAlwaysAsIdentity()` in Drizzle. |
| Welcome template ends up `category: "MARKETING"` in the DB after the poll | Submitted with `allow_category_change: true` (the default) and Meta auto-reclassified | Patch `sub-create-system-template` to send `allow_category_change: false`. Delete the MARKETING row, rewrite copy as UTILITY-compatible, re-provision. |
| Template rejected with `Variables can't be at the start or end of the template` (`code=2388299`) | Body begins or ends with `{{n}}` | Prefix/suffix with text (`Bonjour {{1}},…` or `…{{2}}.`). |
| User sends a message and gets nothing back; `codika list executions` shows zero `main-router` runs | Twilio webhook registered with fake creds, or creds changed after activation | `codika instance deactivate <id>` then `codika instance activate <id>`. Re-registers Twilio Event Streams with current creds. |
| Role routing sends everyone to the wrong handler | `Match User` SQL query doesn't match because phone format differs (`+` vs no `+`) | Pick ONE convention for `members.phone_number` and match it in `Process Input`. Don't mix. |
| Outbound send fails with Twilio `63016 Failed to send freeform message because you are outside the allowed window` | 24-hour conversation window expired; trying to send non-template content to a cold user | Send an approved UTILITY template via `http-send-template` first; resume free-form only after the user replies. |
| Handler response reaches Twilio but user sees truncated text | Message >1600 chars and not chunked | Add the chunking loop from §9.1. |
| Agent calls a tool the user shouldn't have access to | Tool was wired into every handler | Remove the tool node from that role's handler. Role-filter inside the tool as a second line of defense. |

See `post-creation/common-errors.md` for non-WhatsApp-specific n8n errors.

---

## 13. Reference implementation

- **`satellites/whatsapp-bots/instances/wat-dashboard/processes/wat-bot/`** — the canonical single-tenant build. 59 workflows covering:
  - 2 routers: `main-router.json` (Twilio), `main-router-baileys.json` (Evolution API variant).
  - 5 role handlers: `handler-admin.json`, `handler-resident.json`, `handler-community.json`, `handler-onboarding.json`, `handler-not-found.json`.
  - 30+ tool sub-workflows: `tool-*.json` for members, events, guest passes, templates, Notion docs, etc.
  - Shared helpers: `sub-send-whatsapp.json`, `sub-transcribe-audio.json`, `sub-match-user.json`, `sub-create-system-template.json`.
  - Template pipeline: `http-provision-templates.json` + `sub-create-system-template.json` + `scheduled-template-status-check.json` (18 system templates defined).
  - Scheduled jobs: Notion sync, event reminders, digests, invitation nudges, guest pass cleanup.
- **Dashboard**: `satellites/whatsapp-bots/instances/wat-dashboard/src/` — SvelteKit + Drizzle + Better Auth + Neon.

WAT is a good study for "how big can this pattern get" (answer: big). Your starting bot doesn't need 59 workflows — begin with 10–15 and grow as tools are needed.

---

## 14. When to reach for multi-tenant instead

If you end up running this bot for **more than one distinct client organization** — separate credentials, separate data schemas, dashboard CRUD for onboarding new clients — stop and read `whatsapp-bots-multi-tenant.md`. The migration path is clean: extract a handler into its own Codika process, keep the same input contract, add an orchestrator-side `bots` + `phone_mappings` lookup. You don't need to rewrite handlers to migrate.
