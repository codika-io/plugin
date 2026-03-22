# Schedule Triggers Guide

Workflows triggered by schedule (cron) bypass the normal HTTP webhook flow. This guide covers the configuration and patterns for schedule-triggered workflows.

---

## 1. The Problem

HTTP-triggered workflows receive execution metadata from the `triggerWebhook` Cloud Function. Schedule triggers bypass this entirely:

- They're initiated by n8n directly based on a cron expression
- No execution document exists until Codika Init creates one
- Execution time and status must be tracked via Codika nodes
- There is no incoming payload with user data

---

## 2. The Solution

Use the **Codika Init** node immediately after the Schedule Trigger. The node automatically:

1. Calls the `createWorkflowExecution` Cloud Function with member secret authentication
2. Returns execution credentials (executionId, executionSecret, callbackUrl)
3. Stores execution context for use by downstream Codika nodes (Submit Result, Report Error)

---

## 3. Architecture Overview

```text
Schedule Trigger (cron)
        |
Codika Init (initWorkflow)
        |
Business Logic
(access execution data via $('Codika Init').first().json)
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

## 4. Schedule Trigger Configuration

### Schedule Trigger Node

```json
{
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "cronExpression",
          "expression": "*/5 * * * *"
        }
      ]
    }
  },
  "name": "Schedule Trigger",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2
}
```

### Common Cron Expressions

| Expression | Description |
|------------|-------------|
| `*/5 * * * *` | Every 5 minutes |
| `*/15 * * * *` | Every 15 minutes |
| `0 * * * *` | Every hour (at minute 0) |
| `0 */2 * * *` | Every 2 hours |
| `0 9 * * *` | Daily at 9:00 AM |
| `0 9,18 * * *` | Twice daily at 9:00 AM and 6:00 PM |
| `0 9 * * 1` | Every Monday at 9:00 AM |
| `0 9 * * 1-5` | Weekdays at 9:00 AM |
| `0 0 1 * *` | First day of every month at midnight |
| `0 0 * * 0` | Every Sunday at midnight |

### Cron Expression Format

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * *
```

---

## 5. Codika Init Configuration for Schedule Triggers

Schedule triggers use placeholders directly for execution metadata:

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
| `triggerType` | Yes | `"schedule"` | Always "schedule" |

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
    "workflowId": "daily-report",
    "triggerType": "schedule"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

---

## 6. Complete Example: Daily Report Generator

This example generates a daily report and submits it back to Codika.

### Workflow Structure

```text
Schedule Trigger -> Codika Init -> Generate Report -> Codika Submit Result
```

### 1. Schedule Trigger

```json
{
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "cronExpression",
          "expression": "0 9 * * *"
        }
      ]
    }
  },
  "name": "Schedule Trigger",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2
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
    "workflowId": "daily-report",
    "triggerType": "schedule"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

### 3. Generate Report (Code Node)

```javascript
const initData = $('Codika Init').first().json;

// Generate report data
const report = {
  generated_at: new Date().toISOString(),
  execution_id: initData.executionId,
  report_content: 'Daily summary...'
};

return [{ json: report }];
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

## 7. Accessing Execution Data

Schedule triggers have no incoming payload. The Codika Init output provides execution context:

```javascript
// Get execution data from Codika Init
const initData = $('Codika Init').first().json;
const executionId = initData.executionId;
const startTimeMs = initData._startTimeMs;
const processInstanceId = initData.processInstanceId;

// Use in business logic
const durationMs = Date.now() - startTimeMs;

return [{
  json: {
    executionId,
    processInstanceId,
    durationMs,
    completedAt: new Date().toISOString()
  }
}];
```

---

## 8. Config.ts for Schedule Workflows

Schedule-triggered workflows do not have an `inputSchema` since there's no user input:

```typescript
{
  workflowId: 'daily-report',
  workflowFile: 'daily-report-workflow.json',
  triggers: [{
    triggerId: crypto.randomUUID(),
    type: 'schedule',
    title: 'Generate Daily Report',
    description: 'Generates a report every day at 9 AM',
  } satisfies ScheduleTrigger],
  outputSchema: getOutputSchema(),
}
```

### Key Differences from HTTP Triggers

| Aspect | HTTP Trigger | Schedule Trigger |
|--------|-------------|------------------|
| `inputSchema` | Required (defines form) | Not applicable |
| `outputSchema` | Required | Required |
| Codika Init params | Extract from webhook | Use placeholders |
| User interaction | Form submission | None (automatic) |
| Execution timing | On user action | On cron schedule |

---

## 9. Workflow Timezone Configuration

Add `timezone` to workflow settings so cron expressions use local time instead of UTC:

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}",
    "timezone": "Europe/Brussels"
  }
}
```

With `"timezone": "Europe/Brussels"`, cron `0 9 * * *` runs at **9:00 AM Brussels time** (handles DST automatically).

### IANA Timezone Format

Use `{Region}/{City}` format. Common examples:

| Region | Examples |
|--------|----------|
| Africa | `Africa/Cairo`, `Africa/Johannesburg`, `Africa/Lagos`, `Africa/Nairobi` |
| America | `America/New_York`, `America/Chicago`, `America/Los_Angeles`, `America/Sao_Paulo`, `America/Mexico_City` |
| Asia | `Asia/Tokyo`, `Asia/Shanghai`, `Asia/Singapore`, `Asia/Dubai`, `Asia/Kolkata`, `Asia/Seoul` |
| Australia | `Australia/Sydney`, `Australia/Melbourne`, `Australia/Perth`, `Australia/Brisbane` |
| Europe | `Europe/London`, `Europe/Paris`, `Europe/Berlin`, `Europe/Brussels`, `Europe/Vienna`, `Europe/Moscow` |
| Pacific | `Pacific/Auckland`, `Pacific/Honolulu`, `Pacific/Fiji` |
| UTC | `UTC`, `Etc/UTC` |

> ⚠️ **Warning**: `Etc/GMT` zones have **inverted signs**. `Etc/GMT+5` = UTC-5. Use region-based timezones instead.

Full list: [IANA Time Zone Database](https://www.iana.org/time-zones)

---

## 10. Error Workflow Configuration

Always configure an error workflow:

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}",
    "timezone": "Europe/Brussels"
  }
}
```

---

## 11. Manual Trigger URL (Run Now Feature)

Schedule-triggered workflows can be manually triggered via the "Run Now" button in the UI. This requires configuring a `manualTriggerUrl` and adding a Webhook node to the workflow.

### Why Manual Trigger?

- Test workflows without waiting for the scheduled time
- Debug workflow execution on-demand
- Trigger immediate sync/refresh operations

### Config.ts Configuration

Add `manualTriggerUrl` to your schedule trigger definition:

```typescript
{
  triggerId: crypto.randomUUID(),
  type: 'schedule' as const,
  cronExpression: '*/10 * * * *',
  timezone: 'Europe/Brussels',
  humanReadable: 'Every 10 minutes',
  manualTriggerUrl: `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-workflow-manual`,
  title: 'Your Scheduled Workflow',
  description: 'Runs on schedule or manually via Run Now button',
} satisfies ScheduleTrigger
```

### Workflow JSON: Add Webhook Node

You must add a Webhook node to your workflow JSON that listens at the same path as `manualTriggerUrl`:

```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "={{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-workflow-manual",
    "responseMode": "onReceived",
    "options": {}
  },
  "id": "manual-trigger-webhook-001",
  "name": "Manual Trigger Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [0, 200],
  "webhookId": "unique-uuid-here"
}
```

### Connection: Both Triggers → Codika Init

Both the Schedule Trigger and Manual Trigger Webhook should connect to the same Codika Init node:

```
Schedule Trigger ─────────────────────────────→ Codika Init → Business Logic
                                                    ↑
Manual Trigger Webhook ─────────────────────────────┘
```

In the workflow JSON connections:

```json
{
  "connections": {
    "Schedule Trigger": {
      "main": [[{ "node": "Codika Init", "type": "main", "index": 0 }]]
    },
    "Manual Trigger Webhook": {
      "main": [[{ "node": "Codika Init", "type": "main", "index": 0 }]]
    }
  }
}
```

### Important: Path Must Match

The `path` in the Webhook node must match the path portion of `manualTriggerUrl`:

| Config.ts `manualTriggerUrl` | Webhook Node `path` |
|------------------------------|---------------------|
| `{{ORGSECRET}}/webhook/{{PROCDATA}}/{{USERDATA}}/sync-manual` | `={{PROCDATA}}/{{USERDATA}}/sync-manual` |

Note: The Webhook node path omits the base URL and `/webhook/` prefix - it only includes the unique path portion.

### CLI Triggering

Scheduled workflows with `manualTriggerUrl` can be triggered from the CLI:

```bash
codika trigger <workflowId> --poll
```

The CLI creates an execution document upfront and passes `executionMetadata` in the POST body to the `manualTriggerUrl`. The Codika Init node detects the pre-created execution and enters **passthrough mode** (skips calling `createWorkflowExecution`). This means:

- `--poll` works exactly like HTTP triggers
- The CLI returns a real `executionId` immediately
- The execution is tracked in Firestore from the start

When the same workflow runs on its cron schedule (no incoming body), Codika Init enters **create mode** as usual — no behavior change for automatic runs.

---

## 12. Checklist

**Schedule Trigger Node:**
- [ ] Cron expression is valid
- [ ] Schedule timing is appropriate for the use case
- [ ] Timezone configured in workflow settings (e.g., `Europe/Brussels`)

**Codika Init Configuration:**
- [ ] `resource`: `"initializeExecution"`
- [ ] `operation`: `"initWorkflow"`
- [ ] `memberSecret`: `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}`
- [ ] `organizationId`: `{{USERDATA_ORGANIZATION_ID_ATADRESU}}`
- [ ] `userId`: `{{USERDATA_USER_ID_ATADRESU}}`
- [ ] `processInstanceId`: `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] `workflowId`: matches your workflow configuration
- [ ] `triggerType`: `"schedule"`

**Workflow Structure:**
- [ ] Codika Init is immediately after the Schedule Trigger
- [ ] All success paths end with Codika Submit Result
- [ ] All error paths end with Codika Report Error
- [ ] `errorWorkflow` setting configured in workflow JSON

**Data Access:**
- [ ] Access execution data via `$('Codika Init').first().json`

**Manual Trigger (Optional):**
- [ ] `manualTriggerUrl` configured in config.ts with correct path
- [ ] Webhook node added to workflow JSON with matching path
- [ ] Webhook node connects to same Codika Init node as Schedule Trigger

---

## 13. Related Documentation

- [http-triggers.md](./http-triggers.md) - HTTP trigger setup
- [third-party-triggers.md](./third-party-triggers.md) - Third-party trigger setup
- [codika-nodes.md](./codika-nodes.md) - Codika node reference
- [placeholder-patterns.md](./placeholder-patterns.md) - Placeholder patterns
