# Twilio Integration Guide

Twilio provides SMS, WhatsApp messaging, and voice call capabilities.

> **Implementation Reference:** [n8n Twilio Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Twilio)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `twilio` |
| **Trigger Node Type** | `n8n-nodes-base.twilioTrigger` |
| **Action Node Type** | `n8n-nodes-base.twilio` |
| **Credential Type** | `twilioApi` |
| **Credential Level** | ORGCRED (organization-level) |

### Credential Pattern

```json
"credentials": {
  "twilioApi": {
    "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
    "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
  }
}
```

---

## 2. Codika-Specific Tips

### Error Handling

Add `continueOnFail: true` to handle message/call failures gracefully without stopping the workflow.

### Phone Number Format

- **Trigger output**: Numbers have prefixes like `whatsapp:+1234567890`
- **Send node input**: Provide clean numbers like `+1234567890`
- **WhatsApp**: Set `toWhatsapp: true` and the node automatically adds the `whatsapp:` prefix

### Phone Number Storage Pattern (CRITICAL)

**Store phone numbers in database WITHOUT the `+` prefix, add `+` only when sending via Twilio.**

**Why?**
- The `+` character is a special URL character (interpreted as space in query strings)
- Supabase filter strings like `phone_number=eq.+32477047490` fail because `+` → space
- Storing without `+` keeps database queries clean and simple

**Pattern:**

| Operation | What to do | Example |
|-----------|------------|---------|
| **Receive from Twilio** | Strip `whatsapp:` prefix, keep `+` temporarily | `whatsapp:+32477047490` → `+32477047490` |
| **Query database** | Strip `+` before querying | `.replace('+', '')` → `32477047490` |
| **Write to database** | Strip `+` before inserting | `.replace('+', '')` → `32477047490` |
| **Send via Twilio** | Add `+` prefix to database phone | `=+{{ $json.phone_number }}` → `+32477047490` |
| **Reply to sender** | Use senderPhone as-is (already has `+`) | `={{ $json.senderPhone }}` |

**Code examples:**

```javascript
// Querying database - strip + for comparison
"filterString": "={{ 'phone_number=eq.' + $json.senderPhone.replace('+', '') }}"

// Writing to database - strip + before insert
"fieldValue": "={{ $json.senderPhone.replace('+', '') }}"

// Sending to database phone number - add + prefix
"to": "=+{{ $json.user.phone_number }}"

// Replying to sender - already has + from Twilio
"to": "={{ $json.senderPhone }}"
```

**Summary:**
- Database stores: `32477047490` (no `+`)
- Twilio needs: `+32477047490` (with `+`)
- Add `+` only at the Twilio send boundary

### Call Event Delay

The "New Call" trigger event can take **up to 30 minutes** to fire. For real-time call handling, use Twilio's TwiML webhooks directly instead.

---

## 3. Available Resources & Operations

### SMS Resource

| Operation | Value | Description |
|-----------|-------|-------------|
| Send | `send` | Send SMS, MMS, or WhatsApp message |

### Call Resource

| Operation | Value | Description |
|-----------|-------|-------------|
| Make | `make` | Make a phone call with voice message |

### Trigger Events (2 events)

| Event | Value | Description |
|-------|-------|-------------|
| New SMS | `com.twilio.messaging.inbound-message.received` | Incoming SMS/WhatsApp message |
| New Call | `com.twilio.voice.insights.call-summary.complete` | Call completed (up to 30 min delay) |

**IMPORTANT:** The trigger only supports these 2 events. Events like `message.delivered`, `message.sent`, `message.failed` are NOT supported.

---

## 4. Parameter Reference (Complete)

This section documents **all** parameters available in the n8n Twilio nodes.

### SMS Send Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| From | `from` | string | Yes | - | Sender phone number (E.164 format, e.g., `+14155238886`) |
| To | `to` | string | Yes | - | Recipient phone number (E.164 format) |
| To WhatsApp | `toWhatsapp` | boolean | No | `false` | Auto-prefix numbers with `whatsapp:` |
| Message | `message` | string | Yes | - | Message content (text only, MMS not supported) |
| Status Callback | `options.statusCallback` | string | No | - | Webhook URL for delivery status updates |

**Note:** The n8n Twilio node does **not** support MMS media attachments. For MMS, use the HTTP Request node with Twilio's API directly.

### Call Make Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| From | `from` | string | Yes | - | Caller phone number (must be a Twilio number) |
| To | `to` | string | Yes | - | Recipient phone number (E.164 format) |
| Use TwiML | `twiml` | boolean | No | `false` | If `true`, message is raw TwiML XML; if `false`, message is plain text wrapped in `<Say>` |
| Message | `message` | string | Yes | - | Text to speak (plain) or TwiML XML (if `twiml: true`) |
| Status Callback | `options.statusCallback` | string | No | - | Webhook URL for call status updates |

**Note:** When `twiml: false`, the message is automatically wrapped as: `<Response><Say>{message}</Say></Response>`

### Trigger Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Trigger On | `updates` | multiOptions | Yes | `[]` | Array of event types to listen for |

**Available Events (exhaustive list):**
- `com.twilio.messaging.inbound-message.received` - Incoming SMS/WhatsApp message
- `com.twilio.voice.insights.call-summary.complete` - Call completed (up to 30 min delay)

**Note:** These are the ONLY 2 events supported by the trigger. Events like `message.delivered`, `message.sent`, `message.failed` are NOT available in the n8n node.

---

## 5. Common Mistakes

| Wrong | Correct | Issue |
|-------|---------|-------|
| `updates: ["message.delivered"]` | `updates: ["com.twilio.messaging.inbound-message.received"]` | Invalid event type |
| `resource: "message"` | `resource: "sms"` | Wrong resource name |
| `operation: "call"` | `operation: "make"` | Wrong operation name |
| Missing `toWhatsapp: true` | Include for WhatsApp | WhatsApp prefix not added |
| `to: "whatsapp:+123..."` | `to: "+123...", toWhatsapp: true` | Don't add prefix manually |

---

## 6. Node Examples

### 6.1 Twilio Trigger (Receive Messages)

```json
{
  "parameters": {
    "updates": ["com.twilio.messaging.inbound-message.received"]
  },
  "id": "twilio-trigger-001",
  "name": "Twilio Trigger",
  "type": "n8n-nodes-base.twilioTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  }
}
```

### 6.2 Twilio Trigger (Both SMS and Call)

```json
{
  "parameters": {
    "updates": [
      "com.twilio.messaging.inbound-message.received",
      "com.twilio.voice.insights.call-summary.complete"
    ]
  },
  "id": "twilio-trigger-002",
  "name": "Twilio Trigger",
  "type": "n8n-nodes-base.twilioTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  }
}
```

### 6.3 Send SMS Message

```json
{
  "parameters": {
    "resource": "sms",
    "operation": "send",
    "from": "={{ $json.twilioPhone }}",
    "to": "={{ $json.senderPhone }}",
    "toWhatsapp": false,
    "message": "Your reply message here",
    "options": {}
  },
  "id": "twilio-sms-001",
  "name": "Send SMS",
  "type": "n8n-nodes-base.twilio",
  "typeVersion": 1,
  "position": [400, 0],
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 6.4 Send WhatsApp Message

```json
{
  "parameters": {
    "resource": "sms",
    "operation": "send",
    "from": "={{ $json.twilioPhone }}",
    "to": "={{ $json.senderPhone }}",
    "toWhatsapp": true,
    "message": "={{ $json.replyMessage }}",
    "options": {}
  },
  "id": "twilio-whatsapp-001",
  "name": "Send WhatsApp",
  "type": "n8n-nodes-base.twilio",
  "typeVersion": 1,
  "position": [400, 0],
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 6.5 Make Voice Call (Simple Text)

When `twiml: false`, the message text is automatically wrapped in TwiML:

```json
{
  "parameters": {
    "resource": "call",
    "operation": "make",
    "from": "+14155238886",
    "to": "={{ $json.recipientPhone }}",
    "twiml": false,
    "message": "Hello! This is an automated call from our system.",
    "options": {}
  },
  "id": "twilio-call-001",
  "name": "Make Call",
  "type": "n8n-nodes-base.twilio",
  "typeVersion": 1,
  "position": [400, 0],
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

This generates: `<Response><Say>Hello! This is an automated call...</Say></Response>`

### 6.6 Make Voice Call (TwiML)

When `twiml: true`, provide raw TwiML for advanced control:

```json
{
  "parameters": {
    "resource": "call",
    "operation": "make",
    "from": "+14155238886",
    "to": "={{ $json.recipientPhone }}",
    "twiml": true,
    "message": "<Response><Say voice=\"alice\">Hello!</Say><Pause length=\"1\"/><Say>Goodbye.</Say></Response>",
    "options": {}
  },
  "id": "twilio-call-twiml-001",
  "name": "Make Call with TwiML",
  "type": "n8n-nodes-base.twilio",
  "typeVersion": 1,
  "position": [400, 0],
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 6.7 Send with Status Callback

```json
{
  "parameters": {
    "resource": "sms",
    "operation": "send",
    "from": "+14155238886",
    "to": "={{ $json.recipientPhone }}",
    "toWhatsapp": false,
    "message": "Your message here",
    "options": {
      "statusCallback": "https://your-webhook-url.com/status"
    }
  },
  "id": "twilio-status-001",
  "name": "Send with Callback",
  "type": "n8n-nodes-base.twilio",
  "typeVersion": 1,
  "position": [400, 0],
  "credentials": {
    "twilioApi": {
      "id": "{{ORGCRED_TWILIO_ID_DERCGRO}}",
      "name": "{{ORGCRED_TWILIO_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

---

## 7. WhatsApp Message Formatting (CRITICAL for AI Agents)

When sending WhatsApp messages via Twilio, the message formatting is **different from standard Markdown**. AI agents (Claude, GPT, etc.) naturally output Markdown, which will NOT render correctly in WhatsApp.

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

Add this to every AI agent system prompt that sends WhatsApp messages via Twilio:

```
WhatsApp formatting: *single asterisk for bold*, _underscore for italic_.
NEVER use **double asterisks** or # headers - they don't work in WhatsApp!
Use • or - for bullet points, not numbered lists.
```

### Formatting Verification

If AI responses display literally as `**bold**` instead of rendered bold text, the AI is using Markdown instead of WhatsApp formatting. Update the system prompt to be more explicit.

---

## 8. Meta Template Variable Limits (WhatsApp via Twilio Content API)

When creating WhatsApp message templates via the Twilio Content API, Meta enforces a minimum ratio of static text to template variables. If a template has too many variables relative to its length, Meta will reject it with:

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

## 9. Phone Number Parsing

Twilio trigger outputs prefixed phone numbers. Parse them before using in Send node:

### Process Input Node (Code)

```javascript
const input = $input.item.json;
const data = input.data || {};

// Parse the "whatsapp:+number" format to extract clean numbers
const rawFrom = data.from || '';  // e.g., "whatsapp:+1YYYYYYYYYY"
const rawTo = data.to || '';      // e.g., "whatsapp:+1XXXXXXXXXX"

// Extract clean phone numbers (remove prefix)
const senderPhone = rawFrom.replace(/^(whatsapp|sms):/, '');  // "+1YYYYYYYYYY"
const twilioPhone = rawTo.replace(/^(whatsapp|sms):/, '');    // "+1XXXXXXXXXX"

// Determine if this is WhatsApp or SMS
const isWhatsApp = rawFrom.startsWith('whatsapp:');

return [{
  json: {
    senderPhone,      // Who sent the message (reply TO this number)
    twilioPhone,      // Your Twilio number (send FROM this number)
    isWhatsApp,       // true if WhatsApp, false if SMS
    messageBody: data.body || '',
    messageSid: data.messageSid,
    timestamp: data.timestamp
  }
}];
```

---

## 10. Trigger Output Structure

### SMS/WhatsApp Message

```json
{
  "specversion": "1.0",
  "type": "com.twilio.messaging.inbound-message.received",
  "source": "/2010-04-01/Accounts/{accountSid}/Messages/{messageSid}.json",
  "id": "EZxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "time": "2025-12-16T19:50:37.000Z",
  "data": {
    "numMedia": 0,
    "timestamp": "2025-12-16T19:50:37.000Z",
    "accountSid": "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "to": "whatsapp:+1XXXXXXXXXX",
    "numSegments": 1,
    "messageSid": "SMxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "body": "The message content here",
    "from": "whatsapp:+1YYYYYYYYYY"
  }
}
```

| Field | Path | Description |
|-------|------|-------------|
| Message Body | `data.body` | Text content |
| Sender | `data.from` | Sender phone with prefix |
| Recipient | `data.to` | Your Twilio number with prefix |
| Message SID | `data.messageSid` | Unique message ID |

---

## 11. Receiving Media Messages (Voice, Images, etc.)

The Twilio CloudEvents webhook (`twilioTrigger` node) does **NOT** include `MediaContentType0` or `MediaUrl0` fields — those only exist in Twilio's legacy webhook format. The CloudEvents payload only includes `numMedia` (count).

### What the Trigger Gives You

```json
{
  "data": {
    "numMedia": 1,
    "body": "",
    "messageSid": "MMxxxxxxxx",
    "accountSid": "ACxxxxxxxx",
    "from": "whatsapp:+32477047490"
  }
}
```

When `numMedia >= 1`, there is media attached — but no content type or URL.

### Fetching Media Details via Twilio REST API

To get the actual media (type + download URL), call the Twilio Messages Media API:

```
GET https://api.twilio.com/2010-04-01/Accounts/{accountSid}/Messages/{messageSid}/Media.json
```

Use Twilio credentials (`predefinedCredentialType: twilioApi`) for authentication.

**Response structure:**

```json
{
  "media_list": [
    {
      "content_type": "audio/ogg",
      "sid": "MExxxxxxxx",
      "parent_sid": "MMxxxxxxxx",
      "uri": "/2010-04-01/Accounts/ACxxx/Messages/MMxxx/Media/MExxx.json"
    }
  ]
}
```

**Key:** `media_list` is a direct array (not nested under `.media`).

### Building the Download URL

```javascript
const mediaSid = firstMedia.sid;
const mediaUrl = `https://api.twilio.com/2010-04-01/Accounts/${accountSid}/Messages/${messageSid}/Media/${mediaSid}`;
```

Download this URL with Twilio API credentials (basic auth).

### Common Media Content Types

| Type | `content_type` | Source |
|------|---------------|--------|
| Voice note | `audio/ogg` | WhatsApp voice messages (Opus-encoded OGG) |
| Image | `image/jpeg` | Photos and screenshots |
| Video | `video/mp4` | Video messages |
| Document | `application/pdf` | PDFs and documents |

### Complete Pattern: Detect and Fetch Media in n8n

```
Process Input (extract numMedia, hasMedia flag)
  → Is Media? (IF: hasMedia === true)
    → TRUE: HTTP Request GET .../Media.json (Twilio auth)
             → Code: extract content_type + build download URL
             → Route by type (audio → transcribe, image → OCR, etc.)
    → FALSE: continue with text message
```

**Important:** Always reference a node *after* the media resolution when accessing `messageBody` downstream. If you reference `$('Process Input')` directly, you'll get the original empty body, not the processed content. Use a node that sits after both the text and media paths merge (e.g., the IF node both paths flow into).

---

## 12. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `The operation "call" is not known!` | Wrong operation name | Use `operation: "make"` for calls |
| `The resource "message" is not known!` | Wrong resource name | Use `resource: "sms"` |
| Message not sent to WhatsApp | Missing flag | Set `toWhatsapp: true` |
| Call not receiving text | TwiML issue | Set `twiml: false` for plain text |
| Invalid TwiML | Malformed XML | Validate TwiML structure |
| Trigger events not working | Wrong event names | Use exact event strings |
| Call trigger delayed | Expected behavior | Call events take up to 30 minutes |
| `OAuthException subCode=2388293: "This template has too many variables for its length."` | Too many variables relative to static text | Reduce variable count or add more surrounding static words — see section 8 |

---

## 13. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 14. Requirements Checklist

- [ ] Credentials use ORGCRED pattern
- [ ] Resource is `sms` or `call` (not `message`)
- [ ] Operation is `send` for SMS or `make` for calls
- [ ] Phone numbers in clean format (no prefix) for send operations
- [ ] `toWhatsapp: true` set when sending WhatsApp messages
- [ ] `twiml: false` for plain text calls, `twiml: true` for TwiML
- [ ] Trigger uses valid event types (only 2 supported)
- [ ] Write operations include `continueOnFail: true`
- [ ] Process Input node parses phone prefixes from trigger
- [ ] WhatsApp templates satisfy Meta's variable-to-static-text ratio (≈3× variables + 1 words minimum)
- [ ] Media messages: use Messages Media API to fetch content type and download URL (CloudEvents trigger doesn't include them)
- [ ] Media messages: reference a post-merge node for `messageBody`, not `$('Process Input')` directly
- [ ] Error workflow configured
