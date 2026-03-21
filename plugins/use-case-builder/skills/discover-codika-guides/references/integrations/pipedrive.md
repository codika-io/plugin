# Pipedrive CRM Integration Guide

Pipedrive is a sales-focused CRM platform for managing deals, contacts, organizations, activities, and sales pipelines. This integration uses the **official n8n Pipedrive node** and supports both **API Token** and **OAuth2** authentication methods.

> **API Documentation:** [Pipedrive REST API v1](https://developers.pipedrive.com/docs/api/v1)
>
> **Implementation Reference:** [n8n Pipedrive Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Pipedrive)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `pipedrive` |
| **Node Type** | `n8n-nodes-base.pipedrive` |
| **Trigger Node Type** | `n8n-nodes-base.pipedriveTrigger` |
| **Credential Type (API Token)** | `pipedriveApi` |
| **Credential Type (OAuth2)** | `pipedriveOAuth2Api` |
| **Credential Level** | ORGCRED (organization-level) |
| **API Base URL** | `https://api.pipedrive.com/v1` |

### Dual Authentication

Pipedrive supports two authentication methods. The credential type used depends on the `authMethod` stored in the integration metadata:

| Auth Method | Credential Type | Use Case |
|-------------|----------------|----------|
| API Token | `pipedriveApi` | Simple setup, single API token |
| OAuth2 | `pipedriveOAuth2Api` | User-authorized OAuth flow |

### Credential Patterns

**API Token:**

```json
"credentials": {
  "pipedriveApi": {
    "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
    "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
  }
}
```

**OAuth2:**

```json
"credentials": {
  "pipedriveOAuth2Api": {
    "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
    "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
  }
}
```

> **Note:** Both credential types use the same ORGCRED placeholder pattern. The actual credential type (`pipedriveApi` vs `pipedriveOAuth2Api`) is determined at runtime by the cloud function handler based on `metadata.authMethod`.

---

## 2. Codika-Specific Tips

### Error Handling Flags

| Scenario | Flag | Why |
|----------|------|-----|
| Read operations that may return empty | `alwaysOutputData: true` | Prevents workflow from stopping on 0 results |
| Write operations that may fail | `continueOnFail: true` | Allows graceful error handling |

### Custom Properties

Pipedrive uses custom fields extensively. Two special parameters help with custom properties:

| Parameter | Available On | Description |
|-----------|-------------|-------------|
| `resolveProperties` | Get, Get Many | Resolves custom field IDs to human-readable names and option IDs to values |
| `encodeProperties` | Update | Encodes custom property names to IDs and option values to IDs |

Set `resolveProperties: true` when reading data and `encodeProperties: true` when writing data with custom fields.

### Visibility Options

Most resources support a `visible_to` field:
- `'1'` = Owner and followers (private)
- `'3'` = Entire company (shared, default)

---

## 3. Pipedrive API Overview

### API Reference

| Resource | Documentation |
|----------|--------------|
| **Full API Reference** | [developers.pipedrive.com/docs/api/v1](https://developers.pipedrive.com/docs/api/v1) |
| **Activities** | [/docs/api/v1/Activities](https://developers.pipedrive.com/docs/api/v1/Activities) |
| **Deals** | [/docs/api/v1/Deals](https://developers.pipedrive.com/docs/api/v1/Deals) |
| **Leads** | [/docs/api/v1/Leads](https://developers.pipedrive.com/docs/api/v1/Leads) |
| **Organizations** | [/docs/api/v1/Organizations](https://developers.pipedrive.com/docs/api/v1/Organizations) |
| **Persons** | [/docs/api/v1/Persons](https://developers.pipedrive.com/docs/api/v1/Persons) |
| **Products** | [/docs/api/v1/Products](https://developers.pipedrive.com/docs/api/v1/Products) |
| **Notes** | [/docs/api/v1/Notes](https://developers.pipedrive.com/docs/api/v1/Notes) |
| **Files** | [/docs/api/v1/Files](https://developers.pipedrive.com/docs/api/v1/Files) |
| **Webhooks** | [/docs/api/v1/Webhooks](https://developers.pipedrive.com/docs/api/v1/Webhooks) |
| **Pipelines** | [/docs/api/v1/Pipelines](https://developers.pipedrive.com/docs/api/v1/Pipelines) |
| **Stages** | [/docs/api/v1/Stages](https://developers.pipedrive.com/docs/api/v1/Stages) |

### Key API Characteristics

- **Base URL:** `https://api.pipedrive.com/v1`
- **Pagination:** Cursor-based with `start` and `limit` parameters (max 500 per page)
- **Rate Limits:** Vary by plan; API returns `429 Too Many Requests` when exceeded
- **Custom Fields:** Identified by hash keys (e.g., `abc123def456...`). Use `resolveProperties` / `encodeProperties` to work with human-readable names.
- **Currencies:** 3-letter ISO codes (USD, EUR, GBP, etc.)

### API Capabilities Beyond n8n Node

The Pipedrive API offers additional endpoints not directly available as n8n node resources. Use an **HTTP Request** node for these:

| Resource | Endpoint | Description |
|----------|----------|-------------|
| Pipelines | `/v1/pipelines` | Manage sales pipelines |
| Stages | `/v1/stages` | Manage pipeline stages |
| Users | `/v1/users` | Manage workspace users |
| Goals | `/v1/goals` | Sales goals |
| Mail | `/v1/mailbox/mailMessages` | Email integration |
| Subscriptions | `/v1/subscriptions` | Recurring revenue |
| Call Logs | `/v1/callLogs` | Phone call tracking |
| Item Search | `/v1/itemSearch` | Cross-entity search |

---

## 4. Resources & Operations Overview

| Resource | Operations | Description |
|----------|------------|-------------|
| **Activity** | Create, Delete, Get, Get Many, Update | Tasks, calls, meetings, emails |
| **Deal** | Create, Delete, Duplicate, Get, Get Many, Search, Update | Sales opportunities |
| **Deal Activity** | Get Many | Activities linked to a deal |
| **Deal Product** | Add, Get Many, Remove, Update | Products attached to deals |
| **File** | Create, Delete, Download, Get, Update | File attachments |
| **Lead** | Create, Delete, Get, Get Many, Update | Pre-deal prospects |
| **Note** | Create, Delete, Get, Get Many, Update | Notes on entities |
| **Organization** | Create, Delete, Get, Get Many, Search, Update | Companies/accounts |
| **Person** | Create, Delete, Get, Get Many, Search, Update | Contacts |
| **Product** | Get Many | Product catalog |

---

## 5. Activity Resource

Activities represent scheduled tasks like calls, meetings, deadlines, and emails.

### 5.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/activities` | POST |
| Delete | `/v1/activities/{id}` | DELETE |
| Get | `/v1/activities/{id}` | GET |
| Get Many | `/v1/activities` | GET |
| Update | `/v1/activities/{id}` | PUT |

### 5.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `subject` | string | Yes | Subject/title of the activity |
| `done` | options | Yes | Whether activity is done: `'0'` (no) or `'1'` (yes) |
| `type` | string | Yes | Activity type (e.g., `call`, `meeting`, `task`, `deadline`, `email`, `lunch`) |
| `additionalFields.deal_id` | number | No | Deal to associate with |
| `additionalFields.due_date` | dateTime | No | Due date (YYYY-MM-DD) |
| `additionalFields.note` | string | No | Note content (HTML format supported) |
| `additionalFields.org_id` | options | No | Organization to associate with |
| `additionalFields.person_id` | number | No | Person to associate with |
| `additionalFields.user_id` | options | No | User to assign the activity to |
| `additionalFields.customProperties` | fixedCollection | No | Custom field name/value pairs |

### 5.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.done` | boolean | No | - | Filter by done status |
| `additionalFields.start_date` | dateTime | No | - | Filter start date |
| `additionalFields.end_date` | dateTime | No | - | Filter end date |
| `additionalFields.filterId` | options | No | - | Use a predefined Pipedrive filter |
| `additionalFields.type` | multiOptions | No | - | Filter by activity types |
| `additionalFields.user_id` | options | No | - | Filter by assigned user |

### 5.4 Update Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `activityId` | number | Yes | ID of the activity to update |
| `updateFields.subject` | string | No | Updated subject |
| `updateFields.done` | options | No | Done status: `'0'` or `'1'` |
| `updateFields.type` | string | No | Updated activity type |
| `updateFields.due_date` | dateTime | No | Updated due date |
| `updateFields.note` | string | No | Updated note content |
| `updateFields.deal_id` | number | No | Updated deal association |
| `updateFields.org_id` | options | No | Updated organization |
| `updateFields.person_id` | number | No | Updated person |
| `updateFields.user_id` | options | No | Updated assigned user |
| `updateFields.busy_flag` | boolean | No | Mark user as busy during activity |
| `updateFields.public_description` | string | No | Additional details for external calendar |
| `updateFields.customProperties` | fixedCollection | No | Updated custom properties |

### 5.5 Node Examples

#### Create Activity

```json
{
  "parameters": {
    "resource": "activity",
    "operation": "create",
    "subject": "Follow-up call with client",
    "done": "0",
    "type": "call",
    "additionalFields": {
      "deal_id": "={{ $json.dealId }}",
      "due_date": "={{ $now.plus({days: 1}).toISO() }}",
      "note": "<p>Discuss pricing proposal and next steps</p>",
      "person_id": "={{ $json.personId }}"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Activities

```json
{
  "parameters": {
    "resource": "activity",
    "operation": "getAll",
    "returnAll": false,
    "limit": 50,
    "additionalFields": {
      "done": false,
      "type": ["call", "meeting"],
      "start_date": "={{ $now.toISO() }}",
      "end_date": "={{ $now.plus({days: 7}).toISO() }}"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Update Activity (Mark as Done)

```json
{
  "parameters": {
    "resource": "activity",
    "operation": "update",
    "activityId": "={{ $json.id }}",
    "updateFields": {
      "done": "1",
      "note": "<p>Call completed. Client agreed to proceed.</p>"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 6. Deal Resource

Deals represent sales opportunities moving through your pipeline.

### 6.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/deals` | POST |
| Delete | `/v1/deals/{id}` | DELETE |
| Duplicate | `/v1/deals/{id}/duplicate` | POST |
| Get | `/v1/deals/{id}` | GET |
| Get Many | `/v1/deals` | GET |
| Search | `/v1/deals/search` | GET |
| Update | `/v1/deals/{id}` | PUT |

### 6.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | Yes | Deal title |
| `associateWith` | options | Yes | `'organization'` or `'person'` |
| `org_id` | number | If org | Organization ID to associate |
| `person_id` | number | If person | Person ID to associate |
| `additionalFields.value` | number | No | Deal monetary value (default: 0) |
| `additionalFields.currency` | string | No | 3-char currency code (default: `USD`) |
| `additionalFields.stage_id` | options | No | Pipeline stage ID |
| `additionalFields.status` | options | No | `open`, `won`, `lost`, `deleted` (default: `open`) |
| `additionalFields.probability` | number | No | Success probability (0-100) |
| `additionalFields.lost_reason` | string | No | Reason for loss (when status is `lost`) |
| `additionalFields.user_id` | options | No | Assigned user ID |
| `additionalFields.label` | options | No | Deal label |
| `additionalFields.visible_to` | options | No | `'1'` (private) or `'3'` (shared, default) |
| `additionalFields.customProperties` | fixedCollection | No | Custom field name/value pairs |

### 6.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `filters.status` | options | No | - | `all_not_deleted`, `deleted`, `lost`, `open`, `won` |
| `filters.stage_id` | options | No | - | Filter by pipeline stage |
| `filters.filter_id` | options | No | - | Use a predefined Pipedrive filter |
| `filters.user_id` | options | No | - | Filter by assigned user |

### 6.4 Search Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `term` | string | Yes | - | Search term (min 2 chars, or 1 with exact match) |
| `exactMatch` | boolean | Yes | false | Match full term exactly |
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.fields` | multiOptions | No | - | Fields to search: `custom_fields`, `notes`, `title` |
| `additionalFields.organizationId` | string | No | - | Filter by organization |
| `additionalFields.personId` | string | No | - | Filter by person |
| `additionalFields.status` | options | No | - | Status filter |

### 6.5 Node Examples

#### Create Deal

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "create",
    "title": "Enterprise License - Acme Corp",
    "associateWith": "organization",
    "org_id": "={{ $json.orgId }}",
    "additionalFields": {
      "value": 50000,
      "currency": "EUR",
      "stage_id": "={{ $json.stageId }}",
      "status": "open",
      "probability": 60,
      "visible_to": "3"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Deals (Open)

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "getAll",
    "returnAll": false,
    "limit": 100,
    "filters": {
      "status": "open"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Search Deals

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "search",
    "term": "={{ $json.searchQuery }}",
    "exactMatch": false,
    "returnAll": false,
    "limit": 25,
    "additionalFields": {
      "fields": ["title", "custom_fields"]
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Update Deal (Mark as Won)

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "update",
    "dealId": "={{ $json.dealId }}",
    "updateFields": {
      "status": "won",
      "value": 75000,
      "probability": 100
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Duplicate Deal

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "duplicate",
    "dealId": "={{ $json.dealId }}"
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 7. Deal Activity Resource

Retrieve activities linked to a specific deal.

### 7.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Get Many | `/v1/deals/{id}/activities` | GET |

### 7.2 Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `dealId` | options | Yes | - | Deal to get activities for |
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.done` | boolean | No | - | Filter by done status |
| `additionalFields.exclude` | string | No | - | Comma-separated activity IDs to exclude |

### 7.3 Node Example

```json
{
  "parameters": {
    "resource": "dealActivity",
    "operation": "getAll",
    "dealId": "={{ $json.dealId }}",
    "returnAll": false,
    "limit": 50,
    "additionalFields": {
      "done": false
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

---

## 8. Deal Product Resource

Manage products attached to deals, including pricing and quantities.

### 8.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Add | `/v1/deals/{id}/products` | POST |
| Get Many | `/v1/deals/{id}/products` | GET |
| Remove | `/v1/deals/{id}/products/{product_attachment_id}` | DELETE |
| Update | `/v1/deals/{id}/products/{product_attachment_id}` | PUT |

### 8.2 Add Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `dealId` | options | Yes | - | Deal to add product to |
| `productId` | options | Yes | - | Product from catalog |
| `item_price` | number | Yes | - | Price per unit (2 decimal precision) |
| `quantity` | number | Yes | 1 | Quantity (min: 1) |
| `additionalFields.comments` | string | No | - | Product description text |
| `additionalFields.discount_percentage` | number | No | - | Discount percentage (0-100) |
| `additionalFields.product_variation_id` | string | No | - | Product variation ID |

### 8.3 Node Examples

#### Add Product to Deal

```json
{
  "parameters": {
    "resource": "dealProduct",
    "operation": "add",
    "dealId": "={{ $json.dealId }}",
    "productId": "={{ $json.productId }}",
    "item_price": 299.99,
    "quantity": 5,
    "additionalFields": {
      "discount_percentage": 10,
      "comments": "Annual subscription - 10% volume discount"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Deal Products

```json
{
  "parameters": {
    "resource": "dealProduct",
    "operation": "getAll",
    "dealId": "={{ $json.dealId }}"
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

---

## 9. Person Resource

Persons represent individual contacts in your CRM.

### 9.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/persons` | POST |
| Delete | `/v1/persons/{id}` | DELETE |
| Get | `/v1/persons/{id}` | GET |
| Get Many | `/v1/persons` | GET |
| Search | `/v1/persons/search` | GET |
| Update | `/v1/persons/{id}` | PUT |

### 9.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Full name of the person |
| `additionalFields.email` | string[] | No | Email addresses (multiple values) |
| `additionalFields.phone` | string[] | No | Phone numbers (multiple values) |
| `additionalFields.org_id` | options | No | Organization to associate with |
| `additionalFields.label` | options | No | Person label |
| `additionalFields.owner_id` | options | No | Owner user ID |
| `additionalFields.marketing_status` | options | No | `no_consent`, `unsubscribed`, `subscribed`, `archived` |
| `additionalFields.visible_to` | options | No | `'1'` (private) or `'3'` (shared, default) |
| `additionalFields.customProperties` | fixedCollection | No | Custom field name/value pairs |

### 9.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.filterId` | options | No | - | Predefined Pipedrive filter |
| `additionalFields.firstChar` | string | No | - | Filter by first character of name |
| `additionalFields.sort` | string | No | - | Sort by field (e.g., `name ASC`) |

### 9.4 Search Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `term` | string | Yes | - | Search term (min 2 chars, or 1 with exact match) |
| `additionalFields.exactMatch` | boolean | No | false | Full exact matches only |
| `additionalFields.fields` | string | No | - | Comma-separated fields to search |
| `additionalFields.organizationId` | string | No | - | Filter by organization |

### 9.5 Node Examples

#### Create Person

```json
{
  "parameters": {
    "resource": "person",
    "operation": "create",
    "name": "John Doe",
    "additionalFields": {
      "email": ["john.doe@company.com", "john@personal.com"],
      "phone": ["+33612345678"],
      "org_id": "={{ $json.orgId }}",
      "label": "={{ $json.labelId }}",
      "marketing_status": "subscribed",
      "visible_to": "3"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Person

```json
{
  "parameters": {
    "resource": "person",
    "operation": "get",
    "personId": "={{ $json.personId }}",
    "resolveProperties": true
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Search Persons

```json
{
  "parameters": {
    "resource": "person",
    "operation": "search",
    "term": "={{ $json.email }}",
    "returnAll": false,
    "limit": 10,
    "additionalFields": {
      "exactMatch": true
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Update Person

```json
{
  "parameters": {
    "resource": "person",
    "operation": "update",
    "personId": "={{ $json.personId }}",
    "updateFields": {
      "name": "Jonathan Doe",
      "email": ["jonathan.doe@newcompany.com"],
      "phone": ["+33612345678", "+33698765432"],
      "org_id": "={{ $json.newOrgId }}"
    },
    "encodeProperties": true
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 10. Organization Resource

Organizations represent companies or accounts in your CRM.

### 10.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/organizations` | POST |
| Delete | `/v1/organizations/{id}` | DELETE |
| Get | `/v1/organizations/{id}` | GET |
| Get Many | `/v1/organizations` | GET |
| Search | `/v1/organizations/search` | GET |
| Update | `/v1/organizations/{id}` | PUT |

### 10.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Organization name |
| `additionalFields.label` | options | No | Organization label |
| `additionalFields.owner_id` | number | No | Owner user ID |
| `additionalFields.visible_to` | options | No | `'1'` (private) or `'3'` (shared, default) |
| `additionalFields.customProperties` | fixedCollection | No | Custom field name/value pairs |

### 10.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.firstChar` | string | No | - | Filter by first character of name |
| `additionalFields.filterId` | options | No | - | Predefined Pipedrive filter |

### 10.4 Search Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `term` | string | Yes | - | Search term (min 2 chars, or 1 with exact match) |
| `additionalFields.exactMatch` | boolean | No | false | Full exact matches only |
| `additionalFields.fields` | multiOptions | No | - | Fields to search: `address`, `custom_fields`, `name`, `notes` |

### 10.5 Node Examples

#### Create Organization

```json
{
  "parameters": {
    "resource": "organization",
    "operation": "create",
    "name": "Acme Corporation",
    "additionalFields": {
      "label": "={{ $json.labelId }}",
      "visible_to": "3"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Organizations

```json
{
  "parameters": {
    "resource": "organization",
    "operation": "getAll",
    "returnAll": false,
    "limit": 100,
    "resolveProperties": true
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Search Organizations

```json
{
  "parameters": {
    "resource": "organization",
    "operation": "search",
    "term": "Acme",
    "returnAll": false,
    "limit": 25,
    "additionalFields": {
      "fields": ["name", "custom_fields"]
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

---

## 11. Note Resource

Notes can be attached to deals, persons, organizations, or leads.

### 11.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/notes` | POST |
| Delete | `/v1/notes/{id}` | DELETE |
| Get | `/v1/notes/{id}` | GET |
| Get Many | `/v1/notes` | GET |
| Update | `/v1/notes/{id}` | PUT |

### 11.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `content` | string | Yes | Note content (HTML supported) |
| `additionalFields.deal_id` | number | No | Deal to attach note to |
| `additionalFields.lead_id` | number | No | Lead to attach note to |
| `additionalFields.org_id` | options | No | Organization to attach note to |
| `additionalFields.person_id` | number | No | Person to attach note to |

### 11.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `additionalFields.deal_id` | number | No | - | Filter by deal |
| `additionalFields.lead_id` | number | No | - | Filter by lead |
| `additionalFields.org_id` | options | No | - | Filter by organization |
| `additionalFields.person_id` | number | No | - | Filter by person |

### 11.4 Node Examples

#### Create Note on Deal

```json
{
  "parameters": {
    "resource": "note",
    "operation": "create",
    "content": "<h3>Meeting Notes</h3><ul><li>Discussed pricing structure</li><li>Client interested in annual plan</li><li>Follow-up scheduled for next week</li></ul>",
    "additionalFields": {
      "deal_id": "={{ $json.dealId }}"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Notes for a Person

```json
{
  "parameters": {
    "resource": "note",
    "operation": "getAll",
    "returnAll": false,
    "limit": 50,
    "additionalFields": {
      "person_id": "={{ $json.personId }}"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Update Note

```json
{
  "parameters": {
    "resource": "note",
    "operation": "update",
    "noteId": "={{ $json.noteId }}",
    "updateFields": {
      "content": "<p>Updated: Client confirmed the order. Proceeding with onboarding.</p>"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 12. Lead Resource

Leads are pre-deal prospects that haven't yet qualified for a deal.

### 12.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/leads` | POST |
| Delete | `/v1/leads/{id}` | DELETE |
| Get | `/v1/leads/{id}` | GET |
| Get Many | `/v1/leads` | GET |
| Update | `/v1/leads/{id}` | PATCH |

### 12.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | Yes | Lead name/title |
| `associateWith` | options | Yes | `'organization'` or `'person'` |
| `organization_id` | number | If org | Organization to associate |
| `person_id` | number | If person | Person to associate |
| `additionalFields.expected_close_date` | dateTime | No | Expected close date (ISO-8601) |
| `additionalFields.label_ids` | multiOptions | No | Lead label IDs |
| `additionalFields.owner_id` | options | No | Owner user ID |
| `additionalFields.value` | fixedCollection | No | Monetary value: `{ amount, currency }` |

### 12.3 Get Many Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |
| `filters.archived_status` | options | No | `all` | `all`, `archived`, `not_archived` |

### 12.4 Node Examples

#### Create Lead

```json
{
  "parameters": {
    "resource": "lead",
    "operation": "create",
    "title": "Inbound inquiry - Enterprise plan",
    "associateWith": "person",
    "person_id": "={{ $json.personId }}",
    "additionalFields": {
      "expected_close_date": "={{ $now.plus({months: 1}).toISO() }}",
      "value": {
        "valueProperties": {
          "amount": 10000,
          "currency": "EUR"
        }
      }
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Get All Active Leads

```json
{
  "parameters": {
    "resource": "lead",
    "operation": "getAll",
    "returnAll": true,
    "filters": {
      "archived_status": "not_archived"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

#### Update Lead

```json
{
  "parameters": {
    "resource": "lead",
    "operation": "update",
    "leadId": "={{ $json.leadId }}",
    "updateFields": {
      "title": "Qualified - Enterprise plan",
      "value": {
        "valueProperties": {
          "amount": 25000,
          "currency": "EUR"
        }
      }
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 13. File Resource

Manage file attachments associated with deals, persons, organizations, activities, or products.

### 13.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create (Upload) | `/v1/files` | POST |
| Delete | `/v1/files/{id}` | DELETE |
| Download | `/v1/files/{id}/download` | GET |
| Get | `/v1/files/{id}` | GET |
| Update | `/v1/files/{id}` | PUT |

### 13.2 Create (Upload) Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `binaryPropertyName` | string | Yes | `data` | Input binary property name |
| `additionalFields.deal_id` | number | No | - | Deal to associate |
| `additionalFields.person_id` | number | No | - | Person to associate |
| `additionalFields.org_id` | options | No | - | Organization to associate |
| `additionalFields.activity_id` | number | No | - | Activity to associate |
| `additionalFields.product_id` | number | No | - | Product to associate |

### 13.3 Download Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `fileId` | number | Yes | - | File ID to download |
| `binaryPropertyName` | string | Yes | `data` | Output binary property name |

### 13.4 Node Examples

#### Upload File to Deal

```json
{
  "parameters": {
    "resource": "file",
    "operation": "create",
    "binaryPropertyName": "data",
    "additionalFields": {
      "deal_id": "={{ $json.dealId }}"
    }
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Download File

```json
{
  "parameters": {
    "resource": "file",
    "operation": "download",
    "fileId": "={{ $json.fileId }}",
    "binaryPropertyName": "data"
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 14. Product Resource

Product catalog items that can be attached to deals.

### 14.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Get Many | `/v1/products` | GET |

### 14.2 Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `returnAll` | boolean | No | false | Return all results |
| `limit` | number | If !returnAll | 100 | Max results (1-500) |

### 14.3 Node Example

```json
{
  "parameters": {
    "resource": "product",
    "operation": "getAll",
    "returnAll": false,
    "limit": 100,
    "resolveProperties": true
  },
  "type": "n8n-nodes-base.pipedrive",
  "typeVersion": 1,
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

---

## 15. Pipedrive Trigger Node

The Pipedrive Trigger node uses webhooks to listen for real-time events from Pipedrive and starts the workflow when they occur.

### 15.1 Node Type

- **Type**: `n8n-nodes-base.pipedriveTrigger`
- **Type Versions**: `1` and `1.1` (default: `1.1`)
- **Category**: Trigger

### 15.2 Authentication Parameters

| Parameter | Type | Options | Default | Description |
|-----------|------|---------|---------|-------------|
| `authentication` | options | `apiToken`, `oAuth2` | `apiToken` | Authentication method for API calls |
| `incomingAuthentication` | options | `basicAuth`, `none` | `none` | Authentication for incoming webhook requests |

### 15.3 Available Actions (v1.1)

| Action | Value | Description |
|--------|-------|-------------|
| All | `*` | Triggers on any change |
| Created | `create` | Data got added |
| Changed | `change` | Data got changed |
| Deleted | `delete` | Data got deleted |

### 15.4 Available Entities (v1.1)

| Entity | Value | Description |
|--------|-------|-------------|
| All | `*` | Any entity type |
| Activity | `activity` | Activities (calls, meetings, etc.) |
| Activity Type | `activityType` | Activity type definitions |
| Deal | `deal` | Sales opportunities |
| Note | `note` | Notes on entities |
| Organization | `organization` | Companies/accounts |
| Person | `person` | Contacts |
| Pipeline | `pipeline` | Sales pipelines |
| Product | `product` | Product catalog |
| Stage | `stage` | Pipeline stages |
| User | `user` | Workspace users |

### 15.5 Webhook ID Requirement

The `webhookId` must be at the **node level** (sibling to `parameters`, not inside it):

```json
"webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
```

### 15.6 Recommended Workflow Structure

```text
Pipedrive Trigger → Codika Init → Business Logic → IF (success?)
                                                      ├─ Yes → Codika Submit Result
                                                      └─ No  → Codika Report Error
```

The Codika Init node must use `triggerType: "pipedrive"`. See [third-party-triggers.md](../specific/third-party-triggers.md) for full configuration details.

### 15.7 Trigger Output Structure

The webhook delivers event data in this format:

```json
{
  "v": 1,
  "matches_filters": null,
  "meta": {
    "action": "change",
    "object": "deal",
    "id": 12345,
    "company_id": 67890,
    "user_id": 11111,
    "host": "yourcompany.pipedrive.com",
    "timestamp": 1625000000,
    "permitted_user_ids": [11111, 22222],
    "change_source": "app",
    "is_bulk_update": false
  },
  "current": {
    "id": 12345,
    "title": "Updated Deal Name",
    "value": 50000,
    "currency": "EUR",
    "status": "open",
    "stage_id": 3,
    "pipeline_id": 1
  },
  "previous": {
    "title": "Old Deal Name",
    "value": 30000
  },
  "event": "updated.deal"
}
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `meta.action` | What happened: `create`, `change`, `delete` |
| `meta.object` | Entity type: `deal`, `person`, `organization`, etc. |
| `meta.id` | ID of the affected entity |
| `current` | Current state of the entity |
| `previous` | Previous values (only changed fields, on updates) |
| `event` | Event string (e.g., `updated.deal`, `added.person`) |

### 15.8 Trigger Node Examples

#### Listen for All Deal Changes

```json
{
  "parameters": {
    "authentication": "apiToken",
    "action": "*",
    "entity": "deal",
    "incomingAuthentication": "none"
  },
  "type": "n8n-nodes-base.pipedriveTrigger",
  "typeVersion": 1.1,
  "position": [250, 300],
  "id": "pipedrive-trigger-001",
  "name": "Pipedrive Trigger",
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Listen for New Persons Created

```json
{
  "parameters": {
    "authentication": "apiToken",
    "action": "create",
    "entity": "person",
    "incomingAuthentication": "none"
  },
  "type": "n8n-nodes-base.pipedriveTrigger",
  "typeVersion": 1.1,
  "position": [250, 300],
  "id": "pipedrive-trigger-002",
  "name": "New Person Trigger",
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

#### Listen for All Events (Any Entity)

```json
{
  "parameters": {
    "authentication": "apiToken",
    "action": "*",
    "entity": "*",
    "incomingAuthentication": "none"
  },
  "type": "n8n-nodes-base.pipedriveTrigger",
  "typeVersion": 1.1,
  "position": [250, 300],
  "id": "pipedrive-trigger-003",
  "name": "All Events Trigger",
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "pipedriveApi": {
      "id": "{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}",
      "name": "{{ORGCRED_PIPEDRIVE_NAME_DERCGRO}}"
    }
  }
}
```

---

## 16. Common Mistakes

| Wrong | Correct | Issue |
|-------|---------|-------|
| `operation: "list"` | `operation: "getAll"` | Wrong operation name for listing |
| `operation: "find"` | `operation: "search"` | Wrong operation name for search |
| `email: "john@example.com"` | `email: ["john@example.com"]` | Email must be an array |
| `phone: "+33612345678"` | `phone: ["+33612345678"]` | Phone must be an array |
| `done: true` | `done: "1"` | Done status is a string `'0'` or `'1'`, not boolean |
| `status: "closed"` | `status: "won"` | Status values are: `open`, `won`, `lost`, `deleted` |
| `associateWith` missing | Must specify `associateWith: 'organization'` or `'person'` | Required for deal/lead creation |
| Custom field by name | Use `encodeProperties: true` or hash key | Custom fields use hash IDs by default |
| `visible_to: 3` | `visible_to: "3"` | Value must be a string, not number |

---

## 17. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `401 Unauthorized` | Invalid or expired API token | Verify API token or refresh OAuth tokens |
| `403 Forbidden` | Insufficient permissions | Check user's permission level in Pipedrive |
| `404 Not Found` | Entity doesn't exist or was deleted | Verify entity ID before operations |
| `429 Too Many Requests` | Rate limit exceeded | Add delays between operations or reduce batch size |
| `400 Bad Request` | Invalid parameters | Check parameter types and required fields |
| Custom fields show as hash keys | `resolveProperties` not enabled | Add `resolveProperties: true` to get/getAll |
| Custom field updates ignored | Hash key not used | Add `encodeProperties: true` on update, or use hash key directly |
| Empty results from getAll | No matching entities | Add `alwaysOutputData: true` to prevent workflow stoppage |
| Deal creation fails | Missing `associateWith` | Must specify whether to associate with org or person |
| Webhook not firing | Webhook not registered | Activate workflow to register Pipedrive webhook |

---

## 18. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 19. Requirements Checklist

**Trigger workflows:**
- [ ] Trigger node has `webhookId` at node level (not inside parameters)
- [ ] `webhookId` value: `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] Trigger `action` and `entity` configured appropriately
- [ ] Trigger `typeVersion` set to `1.1`
- [ ] Codika Init immediately after trigger with `triggerType: "pipedrive"`

**Action nodes:**
- [ ] Credentials use ORGCRED pattern
- [ ] Correct credential type key based on auth method (`pipedriveApi` or `pipedriveOAuth2Api`)
- [ ] Read operations include `alwaysOutputData: true`
- [ ] Write operations include `continueOnFail: true`
- [ ] `resolveProperties: true` on get/getAll when reading custom fields
- [ ] `encodeProperties: true` on update when writing custom fields
- [ ] `done` field uses string values (`'0'` or `'1'`), not boolean
- [ ] `email` and `phone` fields use array format
- [ ] `associateWith` specified for deal and lead creation
- [ ] `visible_to` uses string values (`'1'` or `'3'`)
- [ ] Error workflow configured
