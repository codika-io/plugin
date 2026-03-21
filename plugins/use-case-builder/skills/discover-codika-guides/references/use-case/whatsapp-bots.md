# WhatsApp Bot Development Guide

This guide documents best practices and lessons learned from building WhatsApp bots with AI agents, specifically the multi-role community bot pattern.

---

## 1. Architecture Overview

### Twilio Integration (Not Direct WhatsApp)

**Important:** We use **Twilio** as the messaging provider for WhatsApp, not direct WhatsApp Business API integration.

**Why Twilio?**
- Easier setup and management
- Unified API for SMS and WhatsApp
- Built-in n8n nodes (`n8n-nodes-base.twilio`)
- Handles WhatsApp Business API complexity

**Setup requirements:**
1. Twilio account with WhatsApp Sandbox or approved sender
2. Twilio credentials configured as organization credential (`ORGCRED_TWILIO`)
3. Webhook URL configured in Twilio to receive incoming messages

### Multi-Role Bot Pattern

A single WhatsApp number (via Twilio) can serve multiple user roles by routing messages through a main router:

```text
Incoming Message (Twilio Webhook) -> Main Router -> Role Detection -> Handler (Admin/Resident/Community/Not Found)
```

Each handler has its own:
- AI agent with role-specific system prompt
- Set of available tools
- Context documents (fetched from database)
- Memory window for conversation continuity

---

## 2. Handler Structure

### Standard Handler Flow

```text
Trigger -> Fetch Context -> Parse Input -> AI Agent -> Prepare Response -> Send via Twilio -> Log Message -> Return Result
```

### Parse Input Node Pattern

The Parse Input node should output ALL context needed by tools, including the caller's role:

```javascript
return [{
  json: {
    senderPhone: triggerInput.senderPhone,
    twilioPhone: triggerInput.twilioPhone,
    messageBody: triggerInput.messageBody,
    userData: userData,
    executionId: triggerInput.executionId,
    executionSecret: triggerInput.executionSecret,
    communityName: triggerInput.communityName || 'WAT',
    residentContext: contextDoc?.content_markdown || '',
    callerRole: 'resident',  // IMPORTANT: Hardcode the role for this handler
    startTimeMs: Date.now()
  }
}];
```

**Key insight:** Each handler knows its role (admin/resident/community), so hardcode `callerRole` in the Parse Input output. This makes it available to all tools.

---

## 3. AI Agent Tool Configuration (CRITICAL)

### The Problem: Null Parameters

When configuring AI agent tools that call sub-workflows, using the wrong parameter format causes values to arrive as `null`.

**BROKEN pattern** (do NOT use):
```json
"specifyInputSchema": true,
"inputSchema": "{ ... }",
"fields": {
  "values": [
    { "name": "caller_role", "value": "resident" }
  ]
}
```

### The Solution: workflowInputs Format

**CORRECT pattern** (always use this):
```json
"workflowInputs": {
  "mappingMode": "defineBelow",
  "value": {
    "caller_role": "={{ $('Parse Input').first().json.callerRole }}"
  },
  "matchingColumns": [],
  "schema": [
    { "id": "caller_role", "displayName": "caller_role", "required": true, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" }
  ],
  "attemptToConvertTypes": false,
  "convertFieldsToString": false
}
```

### Hardcode Context Parameters

**CRITICAL:** Parameters known from the handler context should be hardcoded programmatically, NOT provided by the AI.

| Parameter Type | Source | Example |
|---------------|--------|---------|
| `caller_phone` | From Parse Input | `$('Parse Input').first().json.senderPhone` |
| `caller_role` | From Parse Input | `$('Parse Input').first().json.callerRole` |
| `twilio_phone` | From Parse Input | `$('Parse Input').first().json.twilioPhone` |
| `event_title` | From AI | `$fromAI('event_title', ...)` |
| `participating` | From AI | `$fromAI('participating', ...)` |

**Why?**
1. **Security:** Prevents AI from spoofing user identity
2. **Reliability:** Context values are always correct, not dependent on AI extraction
3. **Simplicity:** AI only provides what it needs to (user intent), not redundant context

### Complete Tool Configuration Example

```json
{
  "parameters": {
    "name": "list_events",
    "description": "Get upcoming events. Use when user asks about events.",
    "workflowId": {
      "__rl": true,
      "value": "{{SUBWKFL_tool-list-events_LFKWBUS}}",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "caller_role": "={{ $('Parse Input').first().json.callerRole }}"
      },
      "matchingColumns": [],
      "schema": [
        { "id": "caller_role", "displayName": "caller_role", "required": true, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" }
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": false
    }
  },
  "id": "tool-list-events-001",
  "name": "List Events Tool",
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2.1,
  "position": [696, 272]
}
```

---

## 4. Tool Sub-Workflow Design

### Input Validation

Always validate inputs at the start of tool sub-workflows:

```javascript
const callerRole = $('When Executed by Another Workflow').item.json.caller_role;

if (!callerRole) {
  return [{ json: { error: 'No caller_role provided', success: false } }];
}
```

### Visibility-Based Filtering

For resources with visibility settings (events, documents, etc.), filter based on caller role:

```javascript
const visibleEvents = allEvents.filter(e => {
  // Admins see everything
  if (callerRole === 'admin') return true;

  const vis = e.visibility || 'all';

  // Public and 'all' events visible to everyone
  if (vis === 'public' || vis === 'all') return true;

  // Community & Residents events
  if (vis === 'community_and_residents' && ['community', 'resident'].includes(callerRole)) return true;

  // Residents-only events
  if (vis === 'residents_only' && callerRole === 'resident') return true;

  return false;
});
```

### Standard Response Format

**Success:**
```json
{
  "success": true,
  "message": "Operation completed",
  "data": { ... }
}
```

**Error:**
```json
{
  "success": false,
  "error": "Descriptive error message"
}
```

---

## 5. Common Issues & Solutions

### Issue: Tool parameters arrive as `null`

**Symptom:** Sub-workflow receives `null` for parameters like `caller_role` or `caller_phone`.

**Cause:** Using the broken `specifyInputSchema` + `fields.values` pattern.

**Solution:** Switch to `workflowInputs` with `mappingMode: "defineBelow"`.

### Issue: Events missing based on visibility

**Symptom:** Users don't see events they should have access to (e.g., residents not seeing `residents_only` events).

**Cause:** `caller_role` is `null` in the tool, so visibility filtering excludes role-specific events.

**Solution:** Ensure `caller_role` is hardcoded in Parse Input and passed correctly to tools.

### Issue: User identity not recognized

**Symptom:** Tool says "User not found" even for registered users.

**Cause:** `caller_phone` is `null` or incorrectly formatted.

**Solution:**
1. Verify `caller_phone` is passed from Parse Input
2. Check phone format (with/without `+` prefix) - see Phone Number Storage section below
3. Use `workflowInputs` pattern, not `fields.values`

---

## 5.1 Phone Number Storage Pattern (CRITICAL)

**Store phone numbers in database WITHOUT the `+` prefix, add `+` only when sending via Twilio.**

### Why Not Store the `+`?

The `+` character is special in URL query strings - it gets interpreted as a **space**!

```
❌ phone_number=eq.+32477047490  →  Interpreted as: phone_number=eq. 32477047490 (with space!)
✅ phone_number=eq.32477047490   →  Works correctly
```

Supabase filter strings use URL encoding, so storing `+` creates query failures.

### The Pattern

| Operation | What to do | Code |
|-----------|------------|------|
| **Query database** | Strip `+` before querying | `.replace('+', '')` |
| **Write to database** | Strip `+` before inserting | `.replace('+', '')` |
| **Send to DB phone** | Add `+` prefix | `=+{{ $json.phone_number }}` |
| **Reply to sender** | Use as-is (already has `+`) | `={{ $json.senderPhone }}` |

### Code Examples

**Database query (Supabase filter):**
```javascript
"filterString": "=phone_number=eq.{{ $json.caller_phone.replace('+', '') }}"
```

**Database write (message log):**
```javascript
{ "fieldId": "phone_number", "fieldValue": "={{ $json.senderPhone.replace('+', '') }}" }
```

**Twilio send to database phone (user from DB):**
```javascript
"to": "=+{{ $json.user.phone_number }}"
```

**Twilio reply to sender (already has `+`):**
```javascript
"to": "={{ $json.senderPhone }}"
```

### Key Distinction: Two Phone Sources

| Source | Has `+`? | When sending via Twilio |
|--------|----------|------------------------|
| `senderPhone` (from Twilio trigger) | Yes | Use directly: `={{ $json.senderPhone }}` |
| `phone_number` (from database) | No | Add prefix: `=+{{ $json.phone_number }}` |

### Summary

```
Database stores:  32477047490     (no +)
Twilio needs:     +32477047490    (with +)
senderPhone:      +32477047490    (already has +)
```

**Rule:** Strip `+` for database operations, add `+` only at Twilio send boundary (when using DB phone numbers)

---

## 6. WhatsApp Formatting

WhatsApp uses different markdown than standard:

| Format | WhatsApp Syntax | Standard Markdown |
|--------|-----------------|-------------------|
| Bold | `*text*` | `**text**` |
| Italic | `_text_` | `*text*` |
| Strikethrough | `~text~` | `~~text~~` |
| Monospace | `` `text` `` | `` `text` `` |

### Sanitize AI Output

AI models often output standard markdown. Sanitize before sending:

```javascript
// Convert **bold** to *bold* (WhatsApp uses single asterisks)
responseText = responseText.replace(/\*\*([^*]+)\*\*/g, '*$1*');

// Remove markdown headers (# ## ###) - WhatsApp doesn't support them
responseText = responseText.replace(/^#{1,6}\s+/gm, '');
```

---

## 7. Memory & Context

### Buffer Window Memory

Use `memoryBufferWindow` for conversation continuity:

```json
{
  "parameters": {
    "sessionIdType": "customKey",
    "sessionKey": "={{ $('Parse Input').first().json.senderPhone }}",
    "contextWindowLength": 20
  },
  "name": "Resident Memory",
  "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
  "typeVersion": 1.3
}
```

**Key:** Use phone number as session key for per-user conversation history.

### Context Documents

Fetch role-specific context documents from database:

```javascript
// In Parse Input, after fetching context document
residentContext: contextDoc?.content_markdown || '',
```

Include in system prompt:
```
{{ $json.residentContext }}
```

---

## 8. Checklist for New WhatsApp Bot Tools

When adding a new tool to a WhatsApp bot:

- [ ] Tool sub-workflow has proper input validation
- [ ] Tool returns `{ success: true/false, ... }` format
- [ ] Handler Parse Input outputs all context needed by tool (`callerRole`, etc.)
- [ ] Tool configuration uses `workflowInputs` pattern (NOT `fields.values`)
- [ ] Context parameters (phone, role) are hardcoded from Parse Input
- [ ] AI-provided parameters use `$fromAI()` syntax
- [ ] Tool is connected to AI agent in handler
- [ ] Tool description clearly explains when to use it

---

## 9. Testing WhatsApp Bots

### Test Each Role

Test the same queries from different user roles:
- Admin: Should see all events, can create/cancel
- Resident: Should see `all` + `residents_only` events
- Community: Should see `all` + `community_and_residents` events

### Verify Tool Outputs

Check n8n execution logs for:
- Input parameters received by tools (not `null`)
- Correct visibility filtering
- Proper error messages returned

### Common Test Cases

1. "What events are coming up?" - Test visibility filtering
2. "Register me for [event]" - Test caller_phone passing
3. "What am I registered for?" - Test my_registrations tool
4. "Submit a suggestion" - Test user lookup from phone

---

## 10. Reference: WAT Community Bot Architecture

```text
main-router.json
|-- handler-admin.json      (callerRole: 'admin')
|-- handler-resident.json   (callerRole: 'resident')
|-- handler-community.json  (callerRole: 'community')
+-- handler-not-found.json

Tools:
|-- tool-list-events.json        (needs: caller_role)
|-- tool-event-participation.json (needs: caller_phone, event_title, participating)
|-- tool-my-registrations.json   (needs: caller_phone)
|-- tool-event-creation.json     (needs: caller_phone + AI params)
|-- tool-event-cancellation.json (needs: caller_phone + event_title)
|-- tool-event-participants.json (needs: event_title only)
|-- tool-faq-lookup.json         (needs: question only)
|-- tool-submit-suggestion.json  (needs: caller_phone + content)
|-- tool-list-suggestions.json   (needs: status_filter only)
|-- tool-update-suggestion.json  (needs: AI params only - admin tool)
+-- tool-notion-sync.json        (needs: caller_phone, twilio_phone)
```
