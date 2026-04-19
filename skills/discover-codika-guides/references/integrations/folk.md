# Folk CRM Integration Guide

Folk CRM is a modern CRM platform for managing contacts, companies, deals, and interactions. This integration uses a **custom community node** (`n8n-nodes-folk`) since Folk does not have an official N8N integration.

---

## ⚠️ CRITICAL: Group ID Format

> **STOP! READ THIS BEFORE USING GROUP IDs**
>
> Folk group URLs and API group IDs have **DIFFERENT FORMATS**:
>
> | Source | Format | Example |
> |--------|--------|---------|
> | **URL in browser** | UUID only | `9d905ae1-e1c9-4904-8225-f2a5eb8e343a` |
> | **API expects** | `grp_` + UUID | `grp_9d905ae1-e1c9-4904-8225-f2a5eb8e343a` |
>
> **If the user provides a Folk group URL like:**
> ```
> https://app.folk.app/apps/contacts/network/.../groups/9d905ae1-e1c9-4904-8225-f2a5eb8e343a/view/...
> ```
>
> **You MUST prepend `grp_` to the extracted UUID:**
> ```javascript
> const groupIdMatch = folkGroupUrl.match(/groups\/([a-f0-9-]+)/i);
> const folkGroupId = groupIdMatch ? `grp_${groupIdMatch[1]}` : null;
> ```
>
> **Without the `grp_` prefix, you will get this error:**
> ```
> UNPROCESSABLE_ENTITY: String must contain exactly 40 characters and start with prefix "grp_"
> ```

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `folk` |
| **Node Type** | `n8n-nodes-folk.folk` |
| **Trigger Node Type** | `n8n-nodes-folk.folkTrigger` |
| **Credential Type** | `folkApi` |
| **Credential Level** | ORGCRED (organization-level) |
| **API Base URL** | `https://api.folk.app` |

### Credential Pattern

```json
"credentials": {
  "folkApi": {
    "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
    "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
  }
}
```

---

## 2. Resources & Operations Overview

| Resource | Operations | Description |
|----------|------------|-------------|
| **Person** | Create, Get, GetMany, Update, Delete | Contact management |
| **Company** | Create, Get, GetMany, Update, Delete | Organization management |
| **Deal** | Create, Get, GetMany, Update, Delete | Custom objects within groups |
| **Group** | GetMany, GetCustomFields | Group and custom field retrieval |
| **User** | Get, GetCurrent, GetMany | Workspace user information |
| **Note** | Create, Get, GetMany, Update, Delete | Notes attached to entities |
| **Reminder** | Create, Get, GetMany, Update, Delete | Recurring reminders |
| **Interaction** | Create | Log interactions (20 types) |
| **Webhook** | Create, Get, GetMany, Update, Delete | Webhook subscriptions |

---

## 3. Person Resource

### 3.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/people` | POST |
| Get | `/v1/people/{personId}` | GET |
| GetMany | `/v1/people` | GET |
| Update | `/v1/people/{personId}` | PATCH |
| Delete | `/v1/people/{personId}` | DELETE |

### 3.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `firstName` | string | No | First name of the person (max 500 characters) |
| `lastName` | string | No | Last name of the person (max 500 characters) |
| `additionalFields.birthday` | dateTime | No | Birthday (YYYY-MM-DD format) |
| `additionalFields.description` | string | No | Brief description |
| `additionalFields.fullName` | string | No | Full name (max 1000 chars) |
| `additionalFields.jobTitle` | string | No | Job title |
| `emails.emailValues[].email` | string | No | Email addresses (max 20) |
| `phones.phoneValues[].phone` | string | No | Phone numbers (max 20) |
| `groups.groupValues[].id` | string | No | Group IDs to add person to (max 100). **⚠️ Must use `grp_` prefix!** |
| `companies.companyValues[].value` | string | No | Company ID or name |
| `companies.companyValues[].isId` | boolean | No | Whether value is an ID (true) or name (false) |
| `addresses.addressValues[].address` | string | No | Physical addresses (max 20) |
| `urls.urlValues[].url` | string | No | URLs (max 20) |
| `customFieldValues` | json | No | Custom field values: `{ "groupId": { "fieldName": "value" } }`|

### 3.3 Node Examples

#### Create Person

```json
{
  "parameters": {
    "resource": "person",
    "operation": "create",
    "firstName": "John",
    "lastName": "Doe",
    "additionalFields": {
      "jobTitle": "Software Engineer",
      "description": "Key contact for technical discussions"
    },
    "emails": {
      "emailValues": [
        { "email": "john.doe@company.com" },
        { "email": "john@personal.com" }
      ]
    },
    "phones": {
      "phoneValues": [
        { "phone": "+33612345678" }
      ]
    },
    "groups": {
      "groupValues": [
        { "id": "grp_abc123" }
      ]
    },
    "companies": {
      "companyValues": [
        { "value": "comp_xyz789", "isId": true }
      ]
    },
    "addresses": {
      "addressValues": [
        { "address": "123 Main St, Paris, France" }
      ]
    },
    "urls": {
      "urlValues": [
        { "url": "https://linkedin.com/in/johndoe" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
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
    "personId": "={{ $json.personId }}"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many People

```json
{
  "parameters": {
    "resource": "person",
    "operation": "getMany",
    "returnAll": false,
    "limit": 50
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Update Person

Update supports all create fields plus emails/phones/groups/companies/addresses/urls/customFieldValues outside updateFields:

```json
{
  "parameters": {
    "resource": "person",
    "operation": "update",
    "personId": "={{ $json.personId }}",
    "updateFields": {
      "firstName": "Jonathan",
      "fullName": "Jonathan Doe",
      "jobTitle": "Senior Software Engineer"
    },
    "emails": {
      "emailValues": [
        { "email": "jonathan.doe@newcompany.com" }
      ]
    },
    "groups": {
      "groupValues": [
        { "id": "grp_new123" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Delete Person

```json
{
  "parameters": {
    "resource": "person",
    "operation": "delete",
    "personId": "={{ $json.personId }}"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 4. Company Resource

### 4.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/companies` | POST |
| Get | `/v1/companies/{companyId}` | GET |
| GetMany | `/v1/companies` | GET |
| Update | `/v1/companies/{companyId}` | PATCH |
| Delete | `/v1/companies/{companyId}` | DELETE |

### 4.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Company name (must be unique across workspace) |
| `additionalFields.description` | string | No | Company description |
| `additionalFields.employeeRange` | options | No | See values below |
| `additionalFields.foundationYear` | string | No | Year founded (YYYY format) |
| `additionalFields.fundingRaised` | number | No | Total funding raised in USD |
| `additionalFields.industry` | string | No | Industry/sector |
| `additionalFields.lastFundingDate` | dateTime | No | Last funding round date (YYYY-MM-DD) |
| `emails.emailValues[].email` | string | No | Company email addresses (max 20) |
| `phones.phoneValues[].phone` | string | No | Company phone numbers (max 20) |
| `urls.urlValues[].url` | string | No | Company website URLs (max 20) |
| `groups.groupValues[].id` | string | No | Group IDs to add company to (max 100). **⚠️ Must use `grp_` prefix!** |
| `addresses.addressValues[].address` | string | No | Physical addresses (max 20) |
| `customFieldValues` | json | No | Custom field values: `{ "groupId": { "fieldName": "value" } }` |

**Employee Range Values:** `1-10`, `11-50`, `51-200`, `201-500`, `501-1000`, `1001-5000`, `5001-10000`, `10000+`

### 4.3 Node Examples

#### Create Company

```json
{
  "parameters": {
    "resource": "company",
    "operation": "create",
    "name": "Acme Corporation",
    "additionalFields": {
      "description": "Technology consulting firm",
      "employeeRange": "51-200",
      "foundationYear": "2015",
      "industry": "Technology"
    },
    "emails": {
      "emailValues": [
        { "email": "contact@acme.com" }
      ]
    },
    "urls": {
      "urlValues": [
        { "url": "https://acme.com" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Companies

```json
{
  "parameters": {
    "resource": "company",
    "operation": "getMany",
    "returnAll": true
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Update Company

Update supports all create fields plus emails/phones/urls/groups/addresses/customFieldValues outside updateFields:

```json
{
  "parameters": {
    "resource": "company",
    "operation": "update",
    "companyId": "={{ $json.companyId }}",
    "updateFields": {
      "description": "Updated company description",
      "employeeRange": "201-500",
      "fundingRaised": 5000000,
      "lastFundingDate": "2024-06-15"
    },
    "emails": {
      "emailValues": [
        { "email": "info@company.com" }
      ]
    },
    "groups": {
      "groupValues": [
        { "id": "grp_abc123" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 5. Deal Resource

Deals are **custom objects** organized within Groups. Each group can have different object types for deals.

> **⚠️ REMINDER:** The `groupId` parameter requires the `grp_` prefix! See [Group ID Format](#️-critical-group-id-format).

### 5.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/groups/{groupId}/{objectType}` | POST |
| Get | `/v1/groups/{groupId}/{objectType}/{dealId}` | GET |
| GetMany | `/v1/groups/{groupId}/{objectType}` | GET |
| Update | `/v1/groups/{groupId}/{objectType}/{dealId}` | PATCH |
| Delete | `/v1/groups/{groupId}/{objectType}/{dealId}` | DELETE |

### 5.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `groupId` | options | Yes | The group to create the deal in. **⚠️ Must use `grp_` prefix format!** |
| `objectType` | options | Yes | Custom object type for deals in this group (depends on groupId) |
| `name` | string | No | Name of the deal (max 1000 characters) |
| `additionalFields.companies` | string | No | Comma-separated company IDs to associate (max 20) |
| `additionalFields.people` | string | No | Comma-separated person IDs to associate (max 20) |
| `additionalFields.customFieldValues` | json | No | Custom field values: `{ "fieldName": "value" }` |

### 5.3 Node Examples

#### Create Deal

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "create",
    "groupId": "grp_abc123",
    "objectType": "deal",
    "name": "New Enterprise Contract",
    "additionalFields": {
      "companies": "comp_xyz789",
      "people": "pers_def456, pers_ghi012"
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Deals

```json
{
  "parameters": {
    "resource": "deal",
    "operation": "getMany",
    "groupId": "grp_abc123",
    "objectType": "deal",
    "returnAll": false,
    "limit": 50
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 6. Group Resource

Groups are containers for organizing deals and custom objects.

> **⚠️ IMPORTANT: Group ID Format**
>
> When working with groups, remember:
> - **URL format:** `https://app.folk.app/.../groups/9d905ae1-e1c9-4904-8225-f2a5eb8e343a/...`
> - **API format:** `grp_9d905ae1-e1c9-4904-8225-f2a5eb8e343a`
>
> Always prepend `grp_` to the UUID extracted from Folk URLs!

### 6.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| GetMany | `/v1/groups` | GET |
| GetCustomFields | `/v1/groups/{groupId}/custom-fields/{entityType}` | GET |

### 6.2 GetCustomFields Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `groupId` | string | Yes | The ID of the group. **⚠️ Must use `grp_` prefix!** |
| `entityType` | string | Yes | The entity type to get custom fields for. Can be "person", "company", or a custom object type name |
| `returnAll` | boolean | No | Whether to return all results (handles pagination) |
| `limit` | number | No | Maximum number of results to return (when returnAll=false) |

### 6.3 Node Examples

#### Get Many Groups

```json
{
  "parameters": {
    "resource": "group",
    "operation": "getMany",
    "returnAll": true
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Custom Fields for Persons

```json
{
  "parameters": {
    "resource": "group",
    "operation": "getCustomFields",
    "groupId": "grp_abc123",
    "entityType": "person",
    "returnAll": true
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Custom Fields for Companies

```json
{
  "parameters": {
    "resource": "group",
    "operation": "getCustomFields",
    "groupId": "grp_abc123",
    "entityType": "company"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 7. User Resource

Retrieve information about workspace users.

### 7.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Get | `/v1/users/{userId}` | GET |
| GetCurrent | `/v1/users/me` | GET |
| GetMany | `/v1/users` | GET |

### 7.2 Node Examples

#### Get Current User

```json
{
  "parameters": {
    "resource": "user",
    "operation": "getCurrent"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Users

```json
{
  "parameters": {
    "resource": "user",
    "operation": "getMany",
    "returnAll": true
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 8. Note Resource

Notes can be attached to persons or companies, with markdown support.

### 8.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/notes` | POST |
| Get | `/v1/notes/{noteId}` | GET |
| GetMany | `/v1/notes` | GET |
| Update | `/v1/notes/{noteId}` | PATCH |
| Delete | `/v1/notes/{noteId}` | DELETE |

### 8.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityId` | string | Yes | ID of person or company to attach note to |
| `content` | string | Yes | Note content (supports markdown, max 100,000 characters) |
| `visibility` | options | Yes | `public` (default) or `private` |
| `additionalFields.parentNoteId` | string | No | ID of parent note (for creating reply/threaded notes) |

### 8.3 Node Examples

#### Create Note

```json
{
  "parameters": {
    "resource": "note",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "content": "## Meeting Summary\n\n- Discussed project timeline\n- Agreed on next steps\n- Follow-up scheduled for next week",
    "visibility": "public"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Reply Note (Threaded)

```json
{
  "parameters": {
    "resource": "note",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "content": "Follow-up: Confirmed the meeting details.",
    "visibility": "public",
    "additionalFields": {
      "parentNoteId": "note_abc123"
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Get Many Notes

```json
{
  "parameters": {
    "resource": "note",
    "operation": "getMany",
    "returnAll": false,
    "limit": 100
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 9. Reminder Resource

Reminders use RFC 5545 ICalendar recurrence rules.

### 9.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/reminders` | POST |
| Get | `/v1/reminders/{reminderId}` | GET |
| GetMany | `/v1/reminders` | GET |
| Update | `/v1/reminders/{reminderId}` | PATCH |
| Delete | `/v1/reminders/{reminderId}` | DELETE |

### 9.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Reminder name (max 255 characters) |
| `entityId` | string | Yes | ID of person or company to attach reminder to |
| `recurrenceRule` | string | Yes | RFC 5545 recurrence rule (includes DTSTART with timezone and RRULE) |
| `visibility` | options | Yes | `public` or `private`. Public reminders are visible to all workspace users |
| `assignedUsers.userValues[].identifierType` | options | No | `id` (User ID) or `email` (Email address) |
| `assignedUsers.userValues[].value` | string | No | The user ID or email address (required for public reminders, 1-50 users) |

### 9.3 Recurrence Rule Examples

| Rule | Description |
|------|-------------|
| `FREQ=DAILY;INTERVAL=1` | Daily |
| `FREQ=WEEKLY;INTERVAL=1` | Weekly |
| `FREQ=WEEKLY;INTERVAL=2` | Every 2 weeks |
| `FREQ=MONTHLY;INTERVAL=1` | Monthly |
| `FREQ=YEARLY;INTERVAL=1` | Yearly |
| `FREQ=WEEKLY;BYDAY=MO,WE,FR` | Every Monday, Wednesday, Friday |

### 9.4 Node Examples

#### Create Reminder with User IDs

```json
{
  "parameters": {
    "resource": "reminder",
    "operation": "create",
    "name": "Weekly follow-up call",
    "entityId": "={{ $json.personId }}",
    "recurrenceRule": "DTSTART;TZID=Europe/Paris:20250717T090000\\nRRULE:FREQ=WEEKLY;INTERVAL=1;BYDAY=MO",
    "visibility": "public",
    "assignedUsers": {
      "userValues": [
        { "identifierType": "id", "value": "user_abc123" },
        { "identifierType": "id", "value": "user_def456" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Reminder with Emails

```json
{
  "parameters": {
    "resource": "reminder",
    "operation": "create",
    "name": "Monthly check-in",
    "entityId": "={{ $json.personId }}",
    "recurrenceRule": "DTSTART;TZID=Europe/Paris:20250801T100000\\nRRULE:FREQ=MONTHLY;INTERVAL=1",
    "visibility": "public",
    "assignedUsers": {
      "userValues": [
        { "identifierType": "email", "value": "john@company.com" },
        { "identifierType": "email", "value": "jane@company.com" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Private Reminder

```json
{
  "parameters": {
    "resource": "reminder",
    "operation": "create",
    "name": "Personal follow-up",
    "entityId": "={{ $json.personId }}",
    "recurrenceRule": "DTSTART;TZID=Europe/Paris:20250720T140000\\nRRULE:FREQ=DAILY;INTERVAL=1",
    "visibility": "private"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 10. Interaction Resource

Log interactions with contacts. **Create only** - no read/update/delete operations.

### 10.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/interactions` | POST |

### 10.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityId` | string | Yes | ID of person or company |
| `title` | string | Yes | Interaction title (max 255 characters) |
| `dateTime` | dateTime | Yes | When the interaction took place |
| `typeMode` | options | Yes | `predefined` (use built-in type) or `custom` (use emoji) |
| `type` | options | Yes (when typeMode=predefined) | Interaction type (see below) |
| `customEmoji` | string | Yes (when typeMode=custom) | A single emoji to use as the interaction type |
| `content` | string | Yes | Notes/content (max 100,000 characters) |

### 10.3 Predefined Interaction Types

| Type Value | Display Name |
|------------|--------------|
| `call` | Call |
| `coffee` | Coffee |
| `discord` | Discord |
| `drink` | Drink |
| `event` | Event |
| `fbMessenger` | Facebook Messenger |
| `hangout` | Hangout |
| `iMessage` | iMessage |
| `linkedin` | LinkedIn |
| `lunch` | Lunch |
| `meeting` | Meeting |
| `message` | Message |
| `signal` | Signal |
| `skype` | Skype |
| `slack` | Slack |
| `telegram` | Telegram |
| `twitter` | Twitter |
| `viber` | Viber |
| `wechat` | WeChat |
| `whatsapp` | WhatsApp |

### 10.4 Custom Emoji Types

You can use any single emoji as a custom interaction type. Examples: `🎉`, `🎂`, `✈️`, `🏆`, `📞`, `💼`

### 10.5 Node Examples

#### Create Meeting Interaction (Predefined Type)

```json
{
  "parameters": {
    "resource": "interaction",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "title": "Product Demo Meeting",
    "dateTime": "2024-01-15T14:00:00.000Z",
    "typeMode": "predefined",
    "type": "meeting",
    "content": "Presented the new features of the platform.\n\nKey points discussed:\n- Pricing structure\n- Implementation timeline\n- Technical requirements\n\nNext steps: Send proposal by Friday"
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Call Interaction

```json
{
  "parameters": {
    "resource": "interaction",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "title": "Discovery Call",
    "dateTime": "={{ $now.toISO() }}",
    "typeMode": "predefined",
    "type": "call",
    "content": "Initial discovery call to understand requirements."
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Interaction with Custom Emoji

```json
{
  "parameters": {
    "resource": "interaction",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "title": "Birthday celebration",
    "dateTime": "={{ $now.toISO() }}",
    "typeMode": "custom",
    "customEmoji": "🎂",
    "content": "Sent birthday wishes and a small gift."
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create WhatsApp Interaction

```json
{
  "parameters": {
    "resource": "interaction",
    "operation": "create",
    "entityId": "={{ $json.personId }}",
    "title": "Quick question via WhatsApp",
    "dateTime": "={{ $now.toISO() }}",
    "typeMode": "predefined",
    "type": "whatsapp",
    "content": "Contact asked about delivery timeline. Confirmed 2-week turnaround."
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 11. Webhook Resource

Programmatically manage Folk webhook subscriptions.

### 11.1 Operations

| Operation | API Endpoint | Method |
|-----------|--------------|--------|
| Create | `/v1/webhooks` | POST |
| Get | `/v1/webhooks/{webhookId}` | GET |
| GetMany | `/v1/webhooks` | GET |
| Update | `/v1/webhooks/{webhookId}` | PATCH |
| Delete | `/v1/webhooks/{webhookId}` | DELETE |

### 11.2 Create Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Webhook name (max 255 characters) |
| `targetUrl` | string | Yes | URL where webhook events will be sent |
| `subscribedEvents.eventValues[].eventType` | options | Yes | Event types to subscribe to (1-20 events) |
| `subscribedEvents.eventValues[].filter` | json | No | Optional filter for this event (see filter format below) |
| `additionalFields.status` | options | No | `active` (default) or `inactive` |

### 11.3 Event Types

| Event Type | Description |
|------------|-------------|
| `person.created` | When a person is created |
| `person.updated` | When a person is updated |
| `person.deleted` | When a person is deleted |
| `company.created` | When a company is created |
| `company.updated` | When a company is updated |
| `company.deleted` | When a company is deleted |
| `object.created` | When a custom object (deal) is created |
| `object.updated` | When a custom object (deal) is updated |
| `object.deleted` | When a custom object (deal) is deleted |

### 11.4 Event Filters

Filters allow you to narrow down which events trigger the webhook. Filter format depends on the event type:

```json
{
  "groupId": "grp_xxx",     // Filter by group - ⚠️ Must include grp_ prefix!
  "path": ["status"],        // Filter by field path
  "value": "won"             // Filter by field value
}
```

> **⚠️ REMINDER:** The `groupId` in filters must use the `grp_` prefix format!

### 11.5 Update Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `webhookId` | string | Yes | The ID of the webhook to update |
| `updateFields.name` | string | No | New webhook name |
| `updateFields.targetUrl` | string | No | New target URL |
| `updateFields.status` | options | No | `active` or `inactive` |
| `subscribedEvents.eventValues[].eventType` | options | No | Event types (replaces existing subscribed events) |
| `subscribedEvents.eventValues[].filter` | json | No | Optional filter for this event |

### 11.6 Node Examples

#### Create Webhook

```json
{
  "parameters": {
    "resource": "webhook",
    "operation": "create",
    "name": "Contact Updates Webhook",
    "targetUrl": "https://my-app.com/webhooks/folk",
    "subscribedEvents": {
      "eventValues": [
        { "eventType": "person.created" },
        { "eventType": "person.updated" },
        { "eventType": "company.created" },
        { "eventType": "company.updated" }
      ]
    },
    "additionalFields": {
      "status": "active"
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Create Webhook with Filters

```json
{
  "parameters": {
    "resource": "webhook",
    "operation": "create",
    "name": "Won Deals Webhook",
    "targetUrl": "https://my-app.com/webhooks/folk/won-deals",
    "subscribedEvents": {
      "eventValues": [
        {
          "eventType": "object.updated",
          "filter": "{\"groupId\": \"grp_abc123\", \"path\": [\"status\"], \"value\": \"won\"}"
        }
      ]
    },
    "additionalFields": {
      "status": "active"
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

#### Update Webhook

```json
{
  "parameters": {
    "resource": "webhook",
    "operation": "update",
    "webhookId": "whk_abc123",
    "updateFields": {
      "name": "Updated Webhook Name",
      "status": "inactive"
    },
    "subscribedEvents": {
      "eventValues": [
        { "eventType": "person.created" },
        { "eventType": "person.deleted" }
      ]
    }
  },
  "type": "n8n-nodes-folk.folk",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  }
}
```

---

## 12. Folk Trigger Node

The Folk Trigger node listens for real-time events from Folk CRM. It automatically creates and manages webhook subscriptions in Folk.

### 12.1 Node Type

- **Type**: `n8n-nodes-folk.folkTrigger`
- **Category**: Trigger

### 12.2 Available Events

| Event | Description |
|-------|-------------|
| `person.created` | Triggered when a person is created |
| `person.updated` | Triggered when a person is updated |
| `person.deleted` | Triggered when a person is deleted |
| `company.created` | Triggered when a company is created |
| `company.updated` | Triggered when a company is updated |
| `company.deleted` | Triggered when a company is deleted |
| `object.created` | Triggered when a custom object is created |
| `object.updated` | Triggered when a custom object is updated |
| `object.deleted` | Triggered when a custom object is deleted |

### 12.3 Node Example

```json
{
  "parameters": {
    "events": [
      "person.created",
      "person.updated",
      "company.created",
      "company.updated"
    ]
  },
  "type": "n8n-nodes-folk.folkTrigger",
  "typeVersion": 1,
  "credentials": {
    "folkApi": {
      "id": "{{ORGCRED_FOLK_ID_DERCGRO}}",
      "name": "{{ORGCRED_FOLK_NAME_DERCGRO}}"
    }
  },
  "webhookId": "auto-generated"
}
```

### 12.4 Trigger Behavior

1. **On Activation**: Automatically creates a webhook in Folk pointing to the N8N webhook URL
2. **On Event**: Receives the event payload and passes it to the workflow
3. **On Deactivation**: Automatically deletes the webhook from Folk

---

## 13. Response Formats

### 13.1 Person Response

```json
{
  "data": {
    "id": "pers_abc123",
    "firstName": "John",
    "lastName": "Doe",
    "birthday": "1990-05-15",
    "description": "Key contact for technical discussions",
    "jobTitle": "Software Engineer",
    "emails": ["john.doe@company.com", "john@personal.com"],
    "phones": ["+33612345678"],
    "createdAt": "2024-01-01T10:00:00.000Z",
    "updatedAt": "2024-01-15T14:30:00.000Z"
  }
}
```

### 13.2 Company Response

```json
{
  "data": {
    "id": "comp_xyz789",
    "name": "Acme Corporation",
    "description": "Technology consulting firm",
    "employeeRange": "51-200",
    "foundationYear": "2015",
    "industry": "Technology",
    "emails": ["contact@acme.com"],
    "phones": ["+33100000000"],
    "urls": ["https://acme.com"],
    "createdAt": "2024-01-01T10:00:00.000Z",
    "updatedAt": "2024-01-15T14:30:00.000Z"
  }
}
```

### 13.3 List Response (GetMany)

```json
{
  "data": {
    "items": [
      { "id": "...", "..." : "..." },
      { "id": "...", "..." : "..." }
    ],
    "nextCursor": "cursor_token_for_pagination"
  }
}
```

### 13.4 Webhook Event Payload

```json
{
  "eventType": "person.updated",
  "data": {
    "id": "pers_abc123",
    "firstName": "John",
    "lastName": "Doe"
  },
  "timestamp": "2024-01-15T14:30:00.000Z"
}
```

---

## 14. Best Practices

### 14.1 Entity ID References

When creating notes, reminders, or interactions, use the entity ID from a previous operation:

```javascript
// Using expression to reference person ID from previous node
"entityId": "={{ $json.data.id }}"

// Or from specific node output
"entityId": "={{ $('Create Person').item.json.data.id }}"
```

### 14.2 Pagination with GetMany

For large datasets, use pagination:

```json
{
  "parameters": {
    "resource": "person",
    "operation": "getMany",
    "returnAll": false,
    "limit": 100
  }
}
```

Set `returnAll: true` only when you need all records (the node handles pagination automatically).

### 14.3 Recurrence Rules

Common RFC 5545 patterns:

```
FREQ=DAILY;INTERVAL=1              // Every day
FREQ=WEEKLY;INTERVAL=1;BYDAY=MO    // Every Monday
FREQ=MONTHLY;INTERVAL=1;BYMONTHDAY=1  // First of every month
FREQ=YEARLY;INTERVAL=1;BYMONTH=1;BYMONTHDAY=1  // January 1st every year
```

### 14.4 Markdown in Notes

Notes support full markdown:

```markdown
## Meeting Notes

**Attendees:** John, Jane, Bob

### Key Points
- Discussed Q1 roadmap
- Agreed on budget allocation

### Action Items
1. [ ] Send proposal
2. [ ] Schedule follow-up
```

### 14.5 Error Handling

Folk API returns standard HTTP error codes:
- `400` - Bad Request (invalid parameters)
- `401` - Unauthorized (invalid/expired API key)
- `404` - Not Found (entity doesn't exist)
- `429` - Rate Limited
- `500` - Server Error

### 14.6 Company Name Uniqueness

Company names must be unique across the workspace. If you try to create a company with an existing name, the API will return an error. Consider checking if the company exists first with GetMany.

---

## 15. FINAL CHECKLIST - Before Using Folk Integration

> **⚠️ STOP AND VERIFY BEFORE DEPLOYING ANY FOLK WORKFLOW**

### Group ID Format Check

If your workflow uses **ANY** of these operations, you MUST verify the group ID format:

| Operation | Parameter to Check |
|-----------|-------------------|
| Person Create/Update | `groups.groupValues[].id` |
| Company Create/Update | `groups.groupValues[].id` |
| Deal Create/Get/Update/Delete | `groupId` |
| Group GetCustomFields | `groupId` |
| Webhook Create with filters | `filter.groupId` |
| customFieldValues | Keys are group IDs |

### The Rule

```
┌─────────────────────────────────────────────────────────────────────────┐
│  URL:  https://app.folk.app/.../groups/9d905ae1-e1c9-4904-8225.../...   │
│                                         └──────────────┬─────────────┘  │
│                                                        │                │
│                                                        ▼                │
│  API:  grp_9d905ae1-e1c9-4904-8225-f2a5eb8e343a                        │
│        └─┬─┘└──────────────────┬───────────────┘                        │
│          │                     │                                        │
│       PREFIX              UUID from URL                                 │
│       (always)           (extracted)                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Code Pattern for URL → Group ID Conversion

```javascript
// If user provides a Folk group URL
const folkGroupUrl = '{{USER_PROVIDED_FOLK_GROUP_URL}}';

// Extract UUID and prepend grp_ prefix
const groupIdMatch = folkGroupUrl.match(/groups\/([a-f0-9-]+)/i);
const folkGroupId = groupIdMatch ? `grp_${groupIdMatch[1]}` : null;

if (!folkGroupId) {
  throw new Error('Could not extract Folk group ID from URL: ' + folkGroupUrl);
}

// Use folkGroupId in your Folk node parameters
```

### Quick Validation

A valid Folk group ID:
- ✅ Starts with `grp_`
- ✅ Is exactly 40 characters long
- ✅ Example: `grp_9d905ae1-e1c9-4904-8225-f2a5eb8e343a`

An INVALID group ID (will cause errors):
- ❌ `9d905ae1-e1c9-4904-8225-f2a5eb8e343a` (missing `grp_` prefix)
- ❌ `grp_abc123` (too short)
- ❌ `group_9d905ae1-e1c9-4904-8225-f2a5eb8e343a` (wrong prefix)
