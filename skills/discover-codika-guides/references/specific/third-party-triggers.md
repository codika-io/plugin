# Third-Party Triggers Guide

Workflows triggered by external services (Gmail, Calendly, WhatsApp, etc.) bypass the normal HTTP webhook flow. This guide covers the configuration and patterns for third-party trigger integrations.

---

## 1. The Problem

HTTP-triggered workflows receive execution metadata from the `triggerWebhook` Cloud Function. Third-party triggers bypass this entirely:

- They're initiated by the external service sending data to n8n
- No execution document exists until Codika Init creates one
- Execution time and status must be tracked via Codika nodes
- The incoming data format is determined by the external service, not Codika

---

## 2. The Solution

Use the **Codika Init** node immediately after the third-party trigger node. The node automatically:

1. Calls the `createWorkflowExecution` Cloud Function with member secret authentication
2. Returns execution credentials (executionId, executionSecret, callbackUrl)
3. Stores execution context for use by downstream Codika nodes (Submit Result, Report Error)

---

## 3. Architecture Overview

```text
Third-Party Trigger (Gmail/Calendly/WhatsApp/etc.)
        |
Codika Init (initWorkflow)
        |
Business Logic
(access execution data via $('Codika Init').first().json)
(access trigger data via $('Trigger Node Name').first().json)
        |
  +-----+-----+
  |           |
Success    Failure
  |           |
Codika     Codika
Submit     Report
Result     Error
```

---

## 4. Webhook ID Requirement

Many third-party trigger nodes require a `webhookId` property at the node level. This ID must be unique per process instance to ensure webhooks are properly routed.

### Why webhookId is Required

- External services register webhooks with n8n
- Each process instance needs its own webhook registration
- Without a unique ID, webhook collisions occur between installations

### Configuration Pattern

The `webhookId` must be:
- At the **node level** (sibling to `parameters`, not inside it)
- Set to `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`

```json
{
  "id": "trigger-node-id",
  "name": "Third Party Trigger",
  "type": "n8n-nodes-base.someTrigger",
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "parameters": {
    "authentication": "oAuth2",
    "events": ["event.created"]
  },
  "credentials": {
    "someOAuth2Api": {
      "id": "{{USERCRED_INTEGRATION_ID_DERCRESU}}",
      "name": "{{USERCRED_INTEGRATION_NAME_DERCRESU}}"
    }
  },
  "typeVersion": 1
}
```

### Triggers That Require webhookId

Check the integration-specific guide for each service. Common triggers requiring `webhookId`:
- Calendly Trigger
- Other webhook-based triggers that register with external services

---

## 5. Codika Init Configuration

Third-party triggers use placeholders directly for execution metadata:

### Parameters

| Parameter | Required | Value | Description |
|-----------|----------|-------|-------------|
| `resource` | Yes | `"initializeExecution"` | Resource type |
| `operation` | Yes | `"initWorkflow"` | Operation |
| `memberSecret` | Yes | `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}` | Member's execution auth secret |
| `organizationId` | Yes | `{{USERDATA_ORGANIZATION_ID_ATADRESU}}` | Organization ID placeholder |
| `userId` | Yes | `{{USERDATA_USER_ID_ATADRESU}}` | User ID placeholder |
| `processInstanceId` | Yes | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` | Process instance ID placeholder |
| `workflowId` | Yes | (literal string) | Workflow template ID |
| `triggerType` | Yes | (literal string) | Trigger source name |

### Trigger Type Values

Use a descriptive name matching the external service:

| Trigger Source | triggerType Value |
|---------------|-------------------|
| Gmail | `"gmail"` |
| Calendly | `"calendly"` |
| WhatsApp | `"whatsapp"` |
| Slack | `"slack"` |
| Pipedrive | `"pipedrive"` |
| Custom Webhook | `"webhook"` or service name |

### Example Configuration

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "{{USERDATA_ORGANIZATION_ID_ATADRESU}}",
    "userId": "{{USERDATA_USER_ID_ATADRESU}}",
    "processInstanceId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
    "workflowId": "email-processor",
    "triggerType": "gmail"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

---

## 6. Accessing Trigger Data

**IMPORTANT: The Codika Init node does NOT pass through its input data.**

To access the original trigger payload (from the external service) in downstream nodes, you must reference the trigger node directly using `$('Trigger Node Name').first().json`.

Do not use `$input.item.json` after Codika Init as it will only contain the Codika execution metadata, not your trigger data.

### Pattern for Accessing Both Data Sources

```javascript
// Get execution data from Codika Init
const initData = $('Codika Init').first().json;
const executionId = initData.executionId;
const startTimeMs = initData._startTimeMs;

// Get original trigger data from the trigger node
const triggerData = $('Gmail Trigger').first().json;
const emailSubject = triggerData.subject;
const emailBody = triggerData.body;

// Use both in business logic
return [{
  json: {
    executionId,
    processedEmail: emailSubject,
    durationMs: Date.now() - startTimeMs
  }
}];
```

### Common Trigger Data Paths

Different services send data in different structures. Always inspect the trigger output to understand the data format:

```javascript
// Gmail
const email = $('Gmail Trigger').first().json;
const subject = email.subject;
const from = email.from;
const body = email.body;

// Calendar events
const event = $('Calendar Trigger').first().json;
const eventName = event.payload?.scheduled_event?.name;
const startTime = event.payload?.scheduled_event?.start_time;

// Slack
const slack = $('Slack Trigger').first().json;
const text = slack.text;           // Message text
const userId = slack.user;         // User ID
const channelId = slack.channel;   // Channel ID
const messageTs = slack.ts;        // Message timestamp

// Pipedrive
const pipedrive = $('Pipedrive Trigger').first().json;
const action = pipedrive.meta?.action;     // 'create', 'change', 'delete'
const entity = pipedrive.meta?.object;     // 'deal', 'person', 'organization', etc.
const entityId = pipedrive.meta?.id;       // ID of the affected entity
const current = pipedrive.current;         // Current state of the entity
const previous = pipedrive.previous;       // Previous values (on updates)

// Webhook (generic)
const webhook = $('Webhook Trigger').first().json;
const payload = webhook.body;
```

---

## 7. Complete Example: Email Processor

This example processes incoming emails and submits results back to Codika.

### Workflow Structure

```text
Gmail Trigger -> Codika Init -> Process Email -> Codika Submit Result
                                              -> Codika Report Error (on failure)
```

### 1. Gmail Trigger Node

```json
{
  "parameters": {
    "pollTimes": {
      "item": [{ "mode": "everyMinute" }]
    },
    "filters": {}
  },
  "name": "Gmail Trigger",
  "type": "n8n-nodes-base.gmailTrigger",
  "typeVersion": 1,
  "credentials": {
    "gmailOAuth2": {
      "id": "{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_GMAIL_NAME_DERCRESU}}"
    }
  }
}
```

### 2. Codika Init

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "{{USERDATA_ORGANIZATION_ID_ATADRESU}}",
    "userId": "{{USERDATA_USER_ID_ATADRESU}}",
    "processInstanceId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
    "workflowId": "email-processor",
    "triggerType": "gmail"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

### 3. Process Email (Code Node)

```javascript
// Access both trigger and execution data
const email = $('Gmail Trigger').first().json;
const initData = $('Codika Init').first().json;

return [{
  json: {
    email_subject: email.subject,
    email_from: email.from,
    processed_at: new Date().toISOString(),
    execution_id: initData.executionId
  }
}];
```

### 4. Codika Submit Result

```json
{
  "parameters": {
    "resource": "workflowOutputs",
    "operation": "submitResult",
    "resultData": "={{ $json }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Submit Result"
}
```

---

## 8. Filtering Trigger Events

Many third-party triggers fire for multiple event types. Add filtering logic to process only relevant events.

### Filter Node Pattern

Add a Filter or Code node after Codika Init to check event type:

```javascript
const triggerData = $('Third Party Trigger').first().json;

// Check if this is an event type we want to process
const eventType = triggerData.event;
const allowedEvents = ['invitee.created', 'message.received'];

if (!allowedEvents.includes(eventType)) {
  return { json: { skip: true, reason: 'Event type not processed' } };
}

// Continue processing
return { json: { skip: false, ...triggerData } };
```

### Using Installation Parameters for Filtering

Allow users to configure which events to process:

```typescript
// In config.ts deploymentInputSchema
{
  key: 'EVENT_FILTER',
  type: 'select',
  label: 'Events to Process',
  description: 'Select which events trigger the workflow',
  options: [
    { label: 'All Events', value: 'all' },
    { label: 'Created Only', value: 'created' },
    { label: 'Canceled Only', value: 'canceled' }
  ],
  required: true
}
```

Then filter in the workflow:

```javascript
const eventFilter = {{INSTPARM_EVENT_FILTER_MRAPTSNI}};  // No extra quotes - value includes quotes
const eventType = $('Trigger').first().json.event;

if (eventFilter !== 'all' && !eventType.includes(eventFilter)) {
  return { json: { skip: true } };
}
```

---

## 9. Config.ts for Third-Party Triggered Workflows

Third-party triggered workflows do not have an `inputSchema` since the input comes from the external service. Use the `ServiceEventTrigger` type:

```typescript
import { type ServiceEventTrigger } from 'codika';

// In the workflow configuration:
{
  workflowId: 'email-processor',
  workflowFile: 'email-processor-workflow.json',
  triggers: [{
    triggerId: crypto.randomUUID(),
    type: 'service_event' as const,
    service: 'email' as const,
    eventType: 'new_email',
    title: 'Process Emails',
    description: 'Processes incoming emails automatically',
  } satisfies ServiceEventTrigger],
  outputSchema: getOutputSchema(),
}
```

### Valid `service` Values

| Value | Use For |
|-------|---------|
| `'email'` | Gmail, Outlook, IMAP triggers |
| `'slack'` | Slack event triggers |
| `'telegram'` | Telegram bot triggers |
| `'discord'` | Discord bot triggers |
| `'pipedrive'` | Pipedrive CRM event triggers |
| `'other'` | WhatsApp (via Twilio), Calendly, Twilio, and any other service |

### Key Differences from HTTP Triggers

| Aspect | HTTP Trigger | Third-Party Trigger |
|--------|-------------|---------------------|
| `inputSchema` | Required (defines form) | Not applicable |
| `outputSchema` | Required | Required |
| Codika Init params | Extract from webhook | Use placeholders |
| `webhookId` | Not needed | May be required |
| User interaction | Form submission | Automatic on event |
| Data source | User form input | External service payload |

---

## 10. Error Workflow Configuration

Always configure an error workflow for third-party triggers:

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

This ensures that if the workflow fails unexpectedly, the error is captured and reported.

---

## 11. Checklist

**Trigger Node Configuration:**
- [ ] `webhookId` at node level if required by the trigger type
- [ ] `webhookId` value: `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] Credentials use appropriate pattern (USERCRED/ORGCRED)
- [ ] Events/filters configured appropriately

**Codika Init Configuration:**
- [ ] `resource`: `"initializeExecution"`
- [ ] `operation`: `"initWorkflow"`
- [ ] `memberSecret`: `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}`
- [ ] `organizationId`: `{{USERDATA_ORGANIZATION_ID_ATADRESU}}`
- [ ] `userId`: `{{USERDATA_USER_ID_ATADRESU}}`
- [ ] `processInstanceId`: `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] `workflowId`: matches your workflow configuration
- [ ] `triggerType`: describes the trigger source

**Workflow Structure:**
- [ ] Codika Init is immediately after the trigger
- [ ] All success paths end with Codika Submit Result
- [ ] All error paths end with Codika Report Error
- [ ] `errorWorkflow` setting configured in workflow JSON

**Data Access:**
- [ ] Access trigger data via `$('Trigger Node Name').first().json`
- [ ] Access execution data via `$('Codika Init').first().json`
- [ ] Do not rely on `$input.item.json` after Codika Init

**Event Filtering (if applicable):**
- [ ] Filter node/logic for relevant event types
- [ ] Handle edge cases (empty payloads, status updates)

---

## Related Documentation

- [http-triggers.md](./http-triggers.md) - HTTP trigger setup
- [schedule-triggers.md](./schedule-triggers.md) - Schedule trigger setup
- [codika-nodes.md](./codika-nodes.md) - Codika node reference
- [placeholder-patterns.md](./placeholder-patterns.md) - Placeholder patterns
- Integration-specific guides in `/integrations/` folder
