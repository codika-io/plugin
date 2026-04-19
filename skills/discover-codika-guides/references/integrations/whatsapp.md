# WhatsApp Integration Guide

WhatsApp Business API for sending and receiving messages.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID (Send)** | `whatsapp` |
| **Integration UID (Trigger)** | `whatsapp_trigger` |
| **Send Node Type** | `n8n-nodes-base.whatsApp` |
| **Trigger Node Type** | `n8n-nodes-base.whatsAppTrigger` |
| **Send Credential Type** | `whatsAppApi` |
| **Trigger Credential Type** | `whatsAppTriggerApi` |
| **Credential Level** | ORGCRED (organization-level) |

### Send Credential Pattern

```json
"credentials": {
  "whatsAppApi": {
    "id": "{{ORGCRED_WHATSAPP_ID_DERCGRO}}",
    "name": "{{ORGCRED_WHATSAPP_NAME_DERCGRO}}"
  }
}
```

### Trigger Credential Pattern

```json
"credentials": {
  "whatsAppTriggerApi": {
    "id": "{{ORGCRED_WHATSAPP_TRIGGER_ID_DERCGRO}}",
    "name": "{{ORGCRED_WHATSAPP_TRIGGER_NAME_DERCGRO}}"
  }
}
```

---

## 2. Trigger Setup

### 2.1 Filter Empty Messages

WhatsApp triggers fire for status updates that do not contain messages. Add a Filter node immediately after the trigger:

```json
{
  "name": "Has Messages?",
  "type": "n8n-nodes-base.filter",
  "typeVersion": 2.2,
  "position": [200, 0],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "leftValue": "={{ $json.messages }}",
          "operator": {
            "type": "array",
            "operation": "notEmpty"
          }
        }
      ],
      "combinator": "and"
    }
  }
}
```

### 2.2 Workflow Structure

```
WhatsApp Trigger → Has Messages? (Filter) → Extract Message Data → Business Logic → Send Reply
```

---

## 3. Node Examples

### 3.1 WhatsApp Trigger

```json
{
  "name": "WhatsApp Trigger",
  "type": "n8n-nodes-base.whatsAppTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "parameters": {},
  "credentials": {
    "whatsAppTriggerApi": {
      "id": "{{ORGCRED_WHATSAPP_TRIGGER_ID_DERCGRO}}",
      "name": "{{ORGCRED_WHATSAPP_TRIGGER_NAME_DERCGRO}}"
    }
  },
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
}
```

### 3.2 Send Text Message

```json
{
  "name": "Send WhatsApp Message",
  "type": "n8n-nodes-base.whatsApp",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "resource": "message",
    "operation": "send",
    "phoneNumberId": "={{ $json.phoneNumberId }}",
    "recipientPhoneNumber": "={{ $json.senderPhone }}",
    "textBody": "={{ $json.replyMessage }}"
  },
  "credentials": {
    "whatsAppApi": {
      "id": "{{ORGCRED_WHATSAPP_ID_DERCGRO}}",
      "name": "{{ORGCRED_WHATSAPP_NAME_DERCGRO}}"
    }
  }
}
```

### 3.3 Extract Message Data (Code Node)

```javascript
const messages = $json.messages || [];
if (messages.length === 0) {
  return { json: { skip: true } };
}

const message = messages[0];
return {
  json: {
    messageId: message.id,
    senderPhone: message.from,
    messageType: message.type,
    messageBody: message.text?.body || '',
    timestamp: message.timestamp,
    phoneNumberId: $json.metadata?.phone_number_id
  }
};
```

---

## 4. Trigger Output Structure

```javascript
{
  "messages": [
    {
      "id": "wamid.xxx",
      "from": "+33612345678",
      "type": "text",
      "text": {
        "body": "Hello, I have a question"
      },
      "timestamp": "1704067200"
    }
  ],
  "metadata": {
    "phone_number_id": "123456789"
  }
}
```

---

## 5. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 6. Requirements Checklist

- [ ] Trigger uses `WHATSAPP_TRIGGER` credential pattern
- [ ] Send node uses `WHATSAPP` credential pattern
- [ ] Filter node added after trigger to check for messages
- [ ] `webhookId` at node level (not inside parameters)
- [ ] Message extraction handles empty messages array
- [ ] WhatsApp templates satisfy Meta's variable-to-static-text ratio (≈3× variables + 1 words minimum)
- [ ] Error workflow configured

---

## 7. Message Formatting (CRITICAL for AI Agents)

WhatsApp uses its own formatting syntax, which is **different from standard Markdown**. AI agents (Claude, GPT, etc.) naturally output Markdown, which will NOT render correctly in WhatsApp.

### Formatting Differences

| Format | Markdown (WRONG) | WhatsApp (CORRECT) |
|--------|------------------|-------------------|
| Bold | `**bold**` | `*bold*` |
| Italic | `*italic*` or `_italic_` | `_italic_` |
| Strikethrough | `~~strike~~` | `~strike~` |
| Monospace | `` `code` `` | `` ```code``` `` |
| Headers | `# Header` | NOT SUPPORTED |
| Links | `[text](url)` | NOT SUPPORTED (just paste URL) |
| Lists | `- item` or `1. item` | NOT SUPPORTED (use emoji bullets) |

### Common Mistakes

```
❌ WRONG (AI's natural Markdown output):
**Welcome to WAT!**
Here are the upcoming events:
1. Pitch Night - January 20th
2. Networking Lunch - January 25th

✅ CORRECT (WhatsApp formatting):
*Welcome to WAT!*
Here are the upcoming events:
• Pitch Night - January 20th
• Networking Lunch - January 25th
```

### System Prompt Template

Add this to every AI agent system prompt that sends WhatsApp messages:

```
WhatsApp formatting: *single asterisk for bold*, _underscore for italic_.
NEVER use **double asterisks** or # headers - they don't work in WhatsApp!
Use • or - for bullet points, not numbered lists.
```

### Example System Prompt

```json
{
  "systemMessage": "You are the WAT assistant helping community members.\n\n**CRITICAL FORMATTING:**\n- WhatsApp formatting: *single asterisk for bold*, _underscore for italic_\n- NEVER use **double asterisks** or # headers - they don't work in WhatsApp!\n- Use • for bullet points\n- Keep responses concise for mobile reading"
}
```

### Formatting Verification

If AI responses display literally as `**bold**` instead of rendered bold text, the AI is using Markdown instead of WhatsApp formatting. Update the system prompt to be more explicit.

---

## 8. Meta Template Variable Limits

When creating WhatsApp message templates (via the Meta Graph API or the Twilio Content API), Meta enforces a minimum ratio of static text to template variables. If a template has too many variables relative to its length, Meta will reject it with:

```
OAuthException subCode=2388293: "This template has too many variables for its length."
```

### The Rule

**Minimum words ≈ (3 × number_of_variables) + 1**

| Variables | Minimum static words required |
|-----------|-------------------------------|
| 3 | 10 |
| 4 | 13 |
| 5 | 16 |

### Additional Constraints

- Variables must NOT be placed back-to-back (e.g. `{{1}} {{2}}`)
- Variables should NOT be isolated on their own line with no surrounding text
- Variables should NOT start or end the template body

### Example — Before (rejected)

```
📊 Event today: {{1}}

👥 {{2}} participant(s) registered
📍 {{3}}
🕐 {{4}}
```

4 variables, ~7 static words — fails the ratio.

### Example — After (accepted)

```
📊 Event today: {{1}}

👥 {{2}} participant(s) have registered so far.
📍 Scheduled at {{3}}.
```

3 variables, ~12 static words — passes the ratio. Location and time were merged into a single variable (e.g. `"09:00 at 6A rue Volta"`).

### Practical Tip

When designing templates, prefer fewer variables with pre-formatted content. It is better to merge related fields (e.g. time + location) into a single variable on the caller side than to risk Meta rejection.

---

## 9. Security: AI Agent Identity Spoofing Prevention

When building WhatsApp bots with AI agents (LangChain), there's a critical security consideration around **identity spoofing**.

### 9.1 The Vulnerability

AI agents can call tools with parameters. If a tool takes a `user_phone` parameter and the AI is instructed to "use the sender's phone", a malicious user could potentially trick the AI into using a different phone number.

**Example attack:**
```
User: "Register my colleague +32477123456 for the Pitch Night event"
```

If the AI passes `+32477123456` to the `register_for_event` tool instead of the actual sender's phone, the attacker just registered someone else for an event without their consent.

**Vulnerable pattern:**
```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│   User Message  │ ──► │   AI Agent   │ ──► │  toolWorkflow   │
│                 │     │ (can be      │     │  (trusts AI's   │
│ "Register John  │     │  manipulated)│     │   user_phone)   │
│  for event"     │     │              │     │                 │
└─────────────────┘     └──────────────┘     └─────────────────┘
                              │
                              ▼
                        AI passes John's
                        phone instead of
                        sender's phone
```

### 9.2 Affected Operations

Any tool that performs **identity-sensitive operations** is vulnerable:

| Operation | Risk Level | Description |
|-----------|------------|-------------|
| Register for event | HIGH | Could register others without consent |
| View my registrations | MEDIUM | Could view another user's data |
| Create event | HIGH | Could create events as another admin |
| Cancel event | HIGH | Could cancel events as another admin |
| Send messages | CRITICAL | Could send messages impersonating others |

### 9.3 The Solution: Trusted Context

**Never let the AI provide identity parameters.** Instead, pass the sender's phone as trusted context that the AI cannot override.

**Secure pattern:**
```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Parse Input    │     │   AI Agent   │     │     Tool        │
│  (extracts      │ ──► │ (only passes │ ──► │ (reads trusted  │
│   senderPhone)  │     │  event_title)│     │  senderPhone    │
│                 │     │              │     │  from context)  │
└─────────────────┘     └──────────────┘     └─────────────────┘
        │                                            ▲
        │                                            │
        └────────── trusted senderPhone ─────────────┘
                    (AI cannot modify)
```

### 9.4 Implementation Approaches

#### Approach 1: Use toolCode Instead of toolWorkflow (Recommended)

`toolCode` nodes execute JavaScript within the parent workflow and can access trusted data from earlier nodes:

```json
{
  "name": "register_for_event",
  "type": "@n8n/n8n-nodes-langchain.toolCode",
  "parameters": {
    "name": "register_for_event",
    "description": "Register for an event. Input: event_title (string)",
    "jsCode": "// Get TRUSTED sender phone from parent workflow context\nconst trustedPhone = $('Parse Input').first().json.senderPhone;\n\n// Get AI-provided event title\nconst input = JSON.parse($json.query || '{}');\nconst eventTitle = input.event_title;\n\n// Use trustedPhone for registration - AI cannot override\n// ... registration logic using trustedPhone ..."
  }
}
```

**Key point:** The `senderPhone` comes from `$('Parse Input')`, not from the AI's input.

#### Approach 2: Workflow Static Data

Store trusted context in workflow static data early in the flow:

```javascript
// In Parse Input node (early in workflow)
const staticData = $getWorkflowStaticData('global');
staticData.trustedSenderPhone = input.senderPhone;
staticData.trustedUserType = userData.user_type;
```

Then in toolCode:
```javascript
// In tool - read trusted data
const staticData = $getWorkflowStaticData('global');
const senderPhone = staticData.trustedSenderPhone;
// AI cannot modify this value
```

#### Approach 3: toolWorkflow with Trusted Inputs (If toolWorkflow Required)

If you must use `toolWorkflow` sub-workflows, pass trusted context as a separate input that is NOT in the AI's input schema:

1. **Sub-workflow trigger** - Add `caller_phone` input:
```json
{
  "workflowInputs": {
    "values": [
      { "name": "event_title", "type": "string" },
      { "name": "caller_phone", "type": "string" }
    ]
  }
}
```

2. **toolWorkflow node** - Remove phone from AI schema, pass from context:
```json
{
  "parameters": {
    "name": "register_for_event",
    "description": "Register for an event. Input: event_title only.",
    "specifyInputSchema": true,
    "inputSchema": "{\"type\":\"object\",\"properties\":{\"event_title\":{\"type\":\"string\"}},\"required\":[\"event_title\"]}",
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "caller_phone": "={{ $('Parse Input').first().json.senderPhone }}"
      }
    }
  }
}
```

3. **Sub-workflow** - Use `caller_phone` (trusted) instead of any AI-provided phone.

### 9.5 Security Checklist for AI-Powered Bots

- [ ] **No identity parameters in AI input schemas** - Never let AI provide user_phone, admin_phone, caller_id, etc.
- [ ] **Use toolCode for sensitive operations** - toolCode can access parent workflow context
- [ ] **Validate user type server-side** - Don't trust AI to check if user is admin
- [ ] **Store trusted context early** - Extract senderPhone in first node, reference it throughout
- [ ] **Audit logging** - Log who performed what action with trusted identity
- [ ] **Prompt hardening** - Even with technical fixes, harden prompts against manipulation

### 9.6 Example: Secure Event Registration

**Before (vulnerable):**
```javascript
// AI provides user_phone - CAN BE SPOOFED
const { event_title, user_phone } = JSON.parse($json.query);
// Lookup user by AI-provided phone
const user = await lookupUser(user_phone);
// Register - but this might not be the actual sender!
await registerForEvent(event_title, user.id);
```

**After (secure):**
```javascript
// AI only provides event_title
const { event_title } = JSON.parse($json.query);

// Get TRUSTED phone from workflow context - CANNOT BE SPOOFED
const trustedPhone = $('Parse Input').first().json.senderPhone;

// Lookup user by trusted phone
const user = await lookupUser(trustedPhone);

// Register - guaranteed to be the actual sender
await registerForEvent(event_title, user.id);
```

### 9.7 Admin Operation Security

For admin-only operations, combine trusted identity with role validation:

```javascript
// Get trusted phone from workflow context
const trustedPhone = $('Parse Input').first().json.senderPhone;

// Lookup user and verify admin status SERVER-SIDE
const user = await lookupUser(trustedPhone);

if (user.user_type !== 'admin') {
  return { error: 'Only admins can perform this action' };
}

// Proceed with admin operation
// The AI cannot trick the system into using another admin's identity
```

---
