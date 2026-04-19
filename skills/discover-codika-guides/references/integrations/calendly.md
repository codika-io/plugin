# Calendly Integration Guide

Calendly is a scheduling automation platform. This integration uses the n8n Calendly Trigger node to receive webhook notifications when meetings are scheduled or canceled.

> **Implementation Reference:** [n8n Calendly Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Calendly)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `calendly` |
| **Trigger Node Type** | `n8n-nodes-base.calendlyTrigger` |
| **Action Node Type** | None (use HTTP Request for API operations) |
| **Credential Type** | `calendlyOAuth2Api` |
| **Credential Level** | USERCRED (member-level OAuth) |

### Credential Pattern

```json
"credentials": {
  "calendlyOAuth2Api": {
    "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
    "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
  }
}
```

---

## 2. Codika-Specific Tips

### Always Use OAuth2

Codika always connects with OAuth2. Use `authentication: "oAuth2"` in all workflows.

### Webhook ID Requirement

The `webhookId` property is **required** at the node level (not inside parameters):

```json
"webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
```

### Error Handling

Trigger nodes don't use `continueOnFail`. Handle errors in downstream nodes with `alwaysOutputData: true` for reads.

### Scope Selection

- **User scope**: Triggers for events belonging to the authenticated user only
- **Organization scope**: Triggers for ALL events across the organization (requires org admin permissions)

---

## 3. Available Resources & Operations

### Trigger Events (2 events)

| Event | Value | Description |
|-------|-------|-------------|
| Event Created | `invitee.created` | New meeting scheduled |
| Event Canceled | `invitee.canceled` | Meeting canceled |

**Note:** Reschedules trigger both events: `invitee.canceled` for the old meeting, then `invitee.created` for the new one.

### API Operations (via HTTP Request)

For operations beyond triggering, use HTTP Request node with `calendlyOAuth2Api` credentials.

| Operation | Endpoint | Method |
|-----------|----------|--------|
| Get Scheduled Events | `/scheduled_events` | GET |
| Get Event | `/scheduled_events/{uuid}` | GET |
| Get Invitees | `/scheduled_events/{uuid}/invitees` | GET |
| Get Event Types | `/event_types` | GET |
| Get Current User | `/users/me` | GET |

---

## 4. Parameter Reference (Complete)

This section documents **all** parameters available in the n8n Calendly Trigger node.

### Trigger Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Authentication | `authentication` | options | Yes | `apiKey` | `oAuth2` (recommended) or `apiKey` - **always use `oAuth2`** |
| Scope | `scope` | options | Yes | `user` | `user` or `organization` - determines webhook scope |
| Events | `events` | multiOptions | Yes | `[]` | Array of event types to listen for |

**Available Events (exhaustive list):**
- `invitee.created` - New meeting scheduled by an invitee
- `invitee.canceled` - Meeting canceled by an invitee

**Scope Details:**
- `user` - Webhook triggers for events belonging to the authenticated user only. Requires `user` URI in webhook creation.
- `organization` - Webhook triggers for ALL events across the organization. Only requires `organization` URI. Requires org admin permissions.

**Note:** The n8n node does NOT expose any additional optional parameters. All webhook lifecycle (create, check, delete) is handled automatically by the node.

---

## 5. Common Mistakes

| Wrong | Correct | Issue |
|-------|---------|-------|
| `authentication: "apiKey"` | `authentication: "oAuth2"` | Use OAuth2 only |
| `events: "invitee.created"` | `events: ["invitee.created"]` | Events must be an array |
| `webhookId` inside parameters | `webhookId` at node level | Wrong placement |
| Missing `webhookId` property | Include `webhookId: "{{USERDATA_...}}"` | Webhook won't register |
| `scope: "org"` | `scope: "organization"` | Wrong scope value |
| `event: "created"` | `events: ["invitee.created"]` | Wrong event value |

---

## 6. Node Examples

### 6.1 Trigger - User Scope (Single Event)

```json
{
  "parameters": {
    "authentication": "oAuth2",
    "scope": "user",
    "events": ["invitee.created"]
  },
  "id": "calendly-trigger-001",
  "name": "Calendly Trigger",
  "type": "n8n-nodes-base.calendlyTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "calendlyOAuth2Api": {
      "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
      "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
    }
  }
}
```

### 6.2 Trigger - User Scope (Both Events)

```json
{
  "parameters": {
    "authentication": "oAuth2",
    "scope": "user",
    "events": ["invitee.created", "invitee.canceled"]
  },
  "id": "calendly-trigger-002",
  "name": "Calendly Trigger",
  "type": "n8n-nodes-base.calendlyTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "calendlyOAuth2Api": {
      "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
      "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
    }
  }
}
```

### 6.3 Trigger - Organization Scope

Use organization scope to receive events from ALL users in the organization. Requires organization admin permissions.

```json
{
  "parameters": {
    "authentication": "oAuth2",
    "scope": "organization",
    "events": ["invitee.created", "invitee.canceled"]
  },
  "id": "calendly-trigger-org-001",
  "name": "Calendly Org Trigger",
  "type": "n8n-nodes-base.calendlyTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "calendlyOAuth2Api": {
      "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
      "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
    }
  }
}
```

### 6.4 HTTP Request - Get User Info

```json
{
  "parameters": {
    "method": "GET",
    "url": "https://api.calendly.com/users/me",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "calendlyOAuth2Api",
    "options": {}
  },
  "id": "get-calendly-user-001",
  "name": "Get Calendly User",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [400, 0],
  "credentials": {
    "calendlyOAuth2Api": {
      "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
      "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.5 HTTP Request - Get Scheduled Events

```json
{
  "parameters": {
    "method": "GET",
    "url": "https://api.calendly.com/scheduled_events",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "calendlyOAuth2Api",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        { "name": "user", "value": "={{ $json.resource.uri }}" },
        { "name": "status", "value": "active" }
      ]
    },
    "options": {}
  },
  "id": "get-events-001",
  "name": "Get Scheduled Events",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [600, 0],
  "credentials": {
    "calendlyOAuth2Api": {
      "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
      "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

---

## 7. Webhook Lifecycle

The n8n Calendly Trigger manages webhooks automatically through three lifecycle methods.

### How It Works

1. **checkExists**: When workflow activates, n8n calls `GET /webhook_subscriptions` to see if a matching webhook already exists
2. **create**: If no matching webhook found, n8n calls `POST /webhook_subscriptions` to create one
3. **delete**: When workflow deactivates, n8n calls `DELETE` on the webhook URI to remove it

### Scope Behavior

| Scope | Organization Parameter | User Parameter |
|-------|------------------------|----------------|
| `user` | Required (from `/users/me`) | Required (from `/users/me`) |
| `organization` | Required (from `/users/me`) | Not sent |

### Webhook State Storage

The node stores webhook state in `workflowStaticData`:
- `webhookURI`: The full URI of the created webhook (used for deletion)

---

## 8. Trigger Output Structure

```javascript
{
  "event": "invitee.created",
  "payload": {
    "email": "john.doe@example.com",
    "name": "John Doe",
    "first_name": "John",
    "last_name": "Doe",
    "timezone": "Europe/Brussels",
    "status": "active",
    "rescheduled": false,
    "cancel_url": "https://calendly.com/cancellations/XXXX",
    "reschedule_url": "https://calendly.com/reschedulings/XXXX",
    "questions_and_answers": [],
    "scheduled_event": {
      "name": "30 Minute Meeting",
      "start_time": "2025-01-07T15:00:00.000000Z",
      "end_time": "2025-01-07T15:30:00.000000Z",
      "event_type": "https://api.calendly.com/event_types/XXXX",
      "location": {
        "type": "google_conference",
        "join_url": "https://meet.google.com/xxx-xxx-xxx"
      }
    }
  }
}
```

### Key Fields

| Field | Path | Description |
|-------|------|-------------|
| Email | `payload.email` | Invitee's email |
| Full Name | `payload.name` | Invitee's full name |
| First Name | `payload.first_name` | May be null |
| Last Name | `payload.last_name` | May be null |
| Event Name | `payload.scheduled_event.name` | Event type name |
| Start Time | `payload.scheduled_event.start_time` | ISO 8601 format |
| End Time | `payload.scheduled_event.end_time` | ISO 8601 format |
| Event Type URI | `payload.scheduled_event.event_type` | For filtering |
| Is Rescheduled | `payload.rescheduled` | Boolean |

---

## 9. Advanced Examples

### 9.1 Filtering by Event Type

Use a Code node to filter for specific Calendly event types:

```javascript
const calendlyEventUrl = {{INSTPARM_CALENDLY_EVENT_URL_MRAPTSNI}};  // No extra quotes - value includes quotes
const eventTypeUri = $json.payload?.scheduled_event?.event_type || '';

const normalizeUrl = (url) => {
  return url.replace(/^https?:\/\//, '').replace(/\/+$/, '').toLowerCase();
};

const configuredUrl = normalizeUrl(calendlyEventUrl);
const eventUrl = normalizeUrl(eventTypeUri);

if (!eventUrl.includes(configuredUrl) && !configuredUrl.includes(eventUrl)) {
  return { json: { skip: true, reason: 'Event type does not match filter' } };
}

return {
  json: {
    skip: false,
    email: $json.payload.email,
    firstName: $json.payload.first_name || $json.payload.name.split(' ')[0],
    lastName: $json.payload.last_name || $json.payload.name.split(' ').slice(1).join(' '),
    fullName: $json.payload.name,
    eventName: $json.payload.scheduled_event.name,
    startTime: $json.payload.scheduled_event.start_time
  }
};
```

#### Deployment Input Schema

```typescript
{
  key: 'CALENDLY_EVENT_URL',
  type: 'string',
  label: 'Calendly Event URL',
  description: 'URL of the specific Calendly event type to process',
  placeholder: 'https://calendly.com/your-name/event-name',
  required: true,
  config: { type: 'string' }
}
```

### 9.2 Handling Reschedules

When processing both `invitee.created` and `invitee.canceled`:

```javascript
const isReschedule = $json.payload.rescheduled === true;
const eventType = $json.event;

if (eventType === 'invitee.canceled' && isReschedule) {
  // Skip canceled event that's part of a reschedule
  // The new invitee.created event will follow
  return { json: { skip: true, reason: 'Reschedule cancellation - ignoring' } };
}

return $input.item;
```

### 9.3 Name Parsing

Handle cases where `first_name`/`last_name` may be null:

```javascript
const fullName = $json.payload.name || '';
const firstName = $json.payload.first_name || fullName.split(' ')[0] || 'Unknown';
const lastName = $json.payload.last_name || fullName.split(' ').slice(1).join(' ') || '';

return {
  json: {
    firstName,
    lastName,
    fullName
  }
};
```

---

## 10. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| Webhook not registering | Missing `webhookId` at node level | Add `webhookId: "{{USERDATA_...}}"` outside parameters |
| Webhook not registering | `webhookId` inside parameters | Move `webhookId` to node level (sibling to `parameters`) |
| No events received | Wrong scope | Use `"user"` scope unless org admin access needed |
| Events from other users | Organization scope | Change to `"user"` scope to filter by authenticated user |
| Empty `first_name`/`last_name` | Calendly allows booking without names | Parse from `name` field as fallback |
| Duplicate events | Both created and canceled for reschedule | Check `rescheduled: true` to filter |

---

## 11. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 12. Requirements Checklist

- [ ] `authentication` set to `"oAuth2"`
- [ ] Credentials use USERCRED pattern (`calendlyOAuth2Api`)
- [ ] `webhookId` at node level (NOT inside parameters)
- [ ] `webhookId` value is `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] `scope` specified (`"user"` or `"organization"`)
- [ ] `events` is an array of event strings
- [ ] Error workflow configured
- [ ] Event type filtering implemented if needed
- [ ] Reschedule handling if processing both events
- [ ] Name parsing handles null `first_name`/`last_name`

---

## 13. Reference Implementation

See `use-cases/calendly-to-folk/` for a complete working example.
