# Supabase Integration Guide

Supabase provides a PostgreSQL database with REST API access for storing and retrieving data.

> **Implementation Reference:** [n8n Supabase Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Supabase)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `supabase` |
| **Node Type** | `n8n-nodes-base.supabase` |
| **Credential Type** | `supabaseApi` |
| **Credential Level** | INSTCRED (per-deployment database) |

### Credential Pattern

```json
"credentials": {
  "supabaseApi": {
    "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
    "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
  }
}
```

---

## 2. Codika-Specific Tips

### Node Output Flags

Different Supabase operations require different flags to prevent workflow interruption:

| Operation Type | Flag Needed | Why |
|----------------|-------------|-----|
| `getAll`, `get` (read) | `alwaysOutputData: true` | Query can return 0 rows → no output → workflow stops |
| `create`, `update`, `delete` (write) | `continueOnFail: true` | Operation can fail (constraints, permissions) → error thrown → workflow stops |

### Data Reference Pattern

The Codika Init node does not expose the original HTTP request input. Always reference the HTTP Trigger node directly:

```javascript
// WRONG - Codika Init output doesn't contain the body
const body = $json.body;

// CORRECT - Reference HTTP Trigger directly
const body = $('HTTP Trigger').first().json.body;
```

In Supabase node expressions:
```
={{ $('HTTP Trigger').first().json.body.user_id }}
```

---

## 3. Available Operations

| Operation | Value | Description |
|-----------|-------|-------------|
| Create | `create` | Create a new row |
| Delete | `delete` | Delete rows matching conditions |
| Get | `get` | Get a single row (requires at least one condition) |
| Get Many | `getAll` | Get many rows with optional filters |
| Update | `update` | Update rows matching conditions |

**IMPORTANT:** The operation is `"create"` NOT `"insert"`. Using `"insert"` will fail.

---

## 4. Parameter Reference (Complete)

This section documents **all** parameters available in the n8n Supabase node.

### Schema Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Use Custom Schema | `useCustomSchema` | boolean | No | `false` | Enable to use a non-public schema |
| Schema | `schema` | string | If useCustomSchema | `"public"` | Database schema name (requires API exposure in Supabase settings) |

**Note:** Custom schemas must be [exposed in the Supabase API settings](https://supabase.com/docs/guides/api/using-custom-schemas?queryGroups=language&language=curl#exposing-custom-schemas) before they can be used.

### Core Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Resource | `resource` | options | Yes | `"row"` | Always `"row"` (only resource available) |
| Operation | `operation` | options | Yes | `"create"` | `create`, `delete`, `get`, `getAll`, `update` |
| Table Name | `tableId` | options | Yes | - | Table name (loaded dynamically from Supabase) |

### Data Parameters (for `create` and `update`)

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Data to Send | `dataToSend` | options | Yes | `"defineBelow"` | `autoMapInputData` or `defineBelow` |
| Inputs to Ignore | `inputsToIgnore` | string | No | `""` | Comma-separated properties to exclude (only for autoMapInputData) |
| Fields | `fieldsUi` | fixedCollection | If defineBelow | `{}` | Container for field definitions |
| Field Values | `fieldsUi.fieldValues` | array | If defineBelow | `[]` | Array of field definitions |
| Field Name | `fieldsUi.fieldValues[].fieldId` | options | Yes | - | Column name - **Use `fieldId` NOT `fieldName`** |
| Field Value | `fieldsUi.fieldValues[].fieldValue` | string | Yes | - | Value to set (supports expressions) |

### Filter Parameters (for `get`, `getAll`, `update`, `delete`)

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Filter Type | `filterType` | options | Yes | `"manual"` | `none` (getAll only), `manual`, or `string` |
| Must Match | `matchType` | options | If manual | `"anyFilter"` | `anyFilter` (OR) or `allFilters` (AND) |
| Filters | `filters` | fixedCollection | If manual | `{}` | Container for conditions |
| Conditions | `filters.conditions` | array | If manual | `[]` | Array of filter conditions |
| Field Name | `filters.conditions[].keyName` | options | Yes | - | Column to filter on |
| Condition | `filters.conditions[].condition` | options | Yes | - | Filter operator (see operators below) |
| Search Function | `filters.conditions[].searchFunction` | options | If fullText | - | Full-text search function |
| Field Value | `filters.conditions[].keyValue` | string | Yes | - | Value to compare |
| Filters (String) | `filterString` | string | If string | `""` | Raw PostgREST filter syntax |

**Note for `get` operation:** The filter is simpler - only `keyName` and `keyValue` fields (no `condition` or `matchType`), and at least one condition is required.

> **CRITICAL for `update` and `delete` operations:** Always use `filterType: "string"` with `filterString`. The `filterType: "manual"` with `matchType`/`filters` does NOT work for update/delete operations in Supabase node v1 — it causes `"At least one select condition must be defined"`. Reserve `manual` filters for `getAll` read operations only.

### Pagination Parameters (for `getAll`)

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results or apply limit |
| Limit | `limit` | number | If !returnAll | `50` | Max results to return (min: 1) |

### Filter Operators

| Operator | Value | Description |
|----------|-------|-------------|
| Equals | `eq` | Exact equality match |
| Not Equals | `neq` | Not equal to value |
| Greater Than | `gt` | Greater than value |
| Greater Than or Equal | `gte` | Greater than or equal |
| Less Than | `lt` | Less than value |
| Less Than or Equal | `lte` | Less than or equal |
| LIKE | `like` | Pattern matching (use `*` instead of `%`) |
| ILIKE | `ilike` | Case-insensitive pattern matching |
| Is | `is` | For null, true, false, unknown |
| Full-Text | `fullText` | Full-text search (requires `searchFunction`) |

---

## 5. Common Mistakes

| Wrong | Correct | Error Message |
|-------|---------|---------------|
| `operation: "insert"` | `operation: "create"` | `The value "insert" is not supported!` |
| `operation: "insertRow"` | `operation: "create"` | `The value "insertRow" is not supported!` |
| `fieldName` | `fieldId` | `The value "" is not supported!` |
| `fieldsToSend` | `fieldsUi` | Fields show as empty in UI |
| `fieldList` | `fieldValues` | Fields show as empty in UI |

---

## 6. Node Examples

### 6.1 Create Row

Create a new row with explicit field definitions:

```json
{
  "parameters": {
    "operation": "create",
    "tableId": "users",
    "dataToSend": "defineBelow",
    "fieldsUi": {
      "fieldValues": [
        {
          "fieldId": "first_name",
          "fieldValue": "={{ $('HTTP Trigger').first().json.body.first_name }}"
        },
        {
          "fieldId": "last_name",
          "fieldValue": "={{ $('HTTP Trigger').first().json.body.last_name }}"
        },
        {
          "fieldId": "email",
          "fieldValue": "={{ $('HTTP Trigger').first().json.body.email }}"
        }
      ]
    }
  },
  "id": "supabase-create-001",
  "name": "Create User",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

### 6.2 Get Row (Single)

Get a single row by matching a specific condition (requires at least one condition):

```json
{
  "parameters": {
    "operation": "get",
    "tableId": "users",
    "filters": {
      "conditions": [
        {
          "keyName": "id",
          "keyValue": "={{ $('HTTP Trigger').first().json.body.user_id }}"
        }
      ]
    }
  },
  "id": "supabase-get-001",
  "name": "Get User By ID",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.3 Get All Rows

Get multiple rows with optional filtering:

```json
{
  "parameters": {
    "operation": "getAll",
    "tableId": "users",
    "returnAll": false,
    "limit": 100,
    "filterType": "none"
  },
  "id": "supabase-getall-001",
  "name": "Get All Users",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.4 Get All Rows with Filter

Get rows matching specific conditions:

```json
{
  "parameters": {
    "operation": "getAll",
    "tableId": "orders",
    "returnAll": true,
    "filterType": "manual",
    "matchType": "allFilters",
    "filters": {
      "conditions": [
        {
          "keyName": "status",
          "condition": "eq",
          "keyValue": "pending"
        },
        {
          "keyName": "created_at",
          "condition": "gte",
          "keyValue": "2024-01-01"
        }
      ]
    }
  },
  "id": "supabase-getall-filtered-001",
  "name": "Get Pending Orders",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.5 Update Row

Update rows matching a PostgREST filter string.

> **IMPORTANT:** Always use `filterType: "string"` with `filterString` for update operations. Do NOT use `filterType: "manual"` with `matchType`/`filters` for updates — it causes the error `"At least one select condition must be defined"`. The `manual` filter type only works reliably for `getAll` read operations.

```json
{
  "parameters": {
    "operation": "update",
    "tableId": "users",
    "filterType": "string",
    "filterString": "=id=eq.{{ $('HTTP Trigger').first().json.body.user_id }}",
    "fieldsUi": {
      "fieldValues": [
        {
          "fieldId": "last_name",
          "fieldValue": "={{ $('HTTP Trigger').first().json.body.new_last_name }}"
        },
        {
          "fieldId": "updated_at",
          "fieldValue": "={{ $now.toISO() }}"
        }
      ]
    }
  },
  "id": "supabase-update-001",
  "name": "Update User",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

**Filter string syntax:** The `filterString` uses raw PostgREST syntax. The leading `=` is required. Examples:
- Single filter: `=id=eq.{{ $json.id }}`
- Multiple filters: `=id=eq.{{ $json.id }}&status=eq.pending`
- Do NOT include the `id` field in `fieldsUi.fieldValues` — the filter handles the row match, the field values are only for columns to update.

### 6.6 Delete Row

Delete rows matching filter conditions:

```json
{
  "parameters": {
    "operation": "delete",
    "tableId": "sessions",
    "filterType": "manual",
    "matchType": "allFilters",
    "filters": {
      "conditions": [
        {
          "keyName": "user_id",
          "condition": "eq",
          "keyValue": "={{ $('HTTP Trigger').first().json.body.user_id }}"
        },
        {
          "keyName": "expired",
          "condition": "eq",
          "keyValue": "true"
        }
      ]
    }
  },
  "id": "supabase-delete-001",
  "name": "Delete Expired Sessions",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

---

## 7. Advanced Examples

### 7.1 Create with Auto-Map Input Data

Automatically map input properties to table columns:

```json
{
  "parameters": {
    "operation": "create",
    "tableId": "logs",
    "dataToSend": "autoMapInputData",
    "inputsToIgnore": "id, created_at"
  },
  "id": "supabase-create-automap-001",
  "name": "Log Entry",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

### 7.2 Filter with PostgREST String

Use raw PostgREST filter syntax for complex queries:

```json
{
  "parameters": {
    "operation": "getAll",
    "tableId": "products",
    "returnAll": true,
    "filterType": "string",
    "filterString": "price=gte.100&category=eq.electronics&in_stock=is.true"
  },
  "id": "supabase-getall-string-001",
  "name": "Get Expensive Electronics",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.3 Full-Text Search

Search text columns using PostgreSQL full-text search:

```json
{
  "parameters": {
    "operation": "getAll",
    "tableId": "articles",
    "returnAll": false,
    "limit": 20,
    "filterType": "manual",
    "matchType": "allFilters",
    "filters": {
      "conditions": [
        {
          "keyName": "content",
          "condition": "fullText",
          "searchFunction": "wfts",
          "keyValue": "machine learning AI"
        }
      ]
    }
  },
  "id": "supabase-fts-001",
  "name": "Search Articles",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

**Full-Text Search Functions:**
| Function | Value | Description |
|----------|-------|-------------|
| to_tsquery | `fts` | Standard text search query |
| plainto_tsquery | `plfts` | Plain text to query |
| phraseto_tsquery | `phfts` | Phrase search |
| websearch_to_tsquery | `wfts` | Web-style search syntax |

### 7.4 LIKE Pattern Matching

Search with wildcards (use `*` instead of `%`):

```json
{
  "parameters": {
    "operation": "getAll",
    "tableId": "users",
    "returnAll": true,
    "filterType": "manual",
    "matchType": "anyFilter",
    "filters": {
      "conditions": [
        {
          "keyName": "email",
          "condition": "ilike",
          "keyValue": "*@company.com"
        }
      ]
    }
  },
  "id": "supabase-like-001",
  "name": "Find Company Users",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.5 Custom Schema

Query tables in a non-public schema (requires schema exposure in Supabase API settings):

```json
{
  "parameters": {
    "useCustomSchema": true,
    "schema": "private_data",
    "operation": "getAll",
    "tableId": "sensitive_records",
    "returnAll": false,
    "limit": 50,
    "filterType": "none"
  },
  "id": "supabase-custom-schema-001",
  "name": "Get Private Schema Data",
  "type": "n8n-nodes-base.supabase",
  "typeVersion": 1,
  "position": [660, 0],
  "credentials": {
    "supabaseApi": {
      "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
      "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
    }
  },
  "alwaysOutputData": true
}
```

**Note:** Custom schemas must be exposed in your Supabase project's API settings before they can be accessed via the REST API.

---

## 8. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `The value "insert" is not supported!` | Wrong operation name | Use `"create"` instead of `"insert"` |
| `The value "insertRow" is not supported!` | Wrong operation name | Use `"create"` |
| `The value "" is not supported!` | Empty field name | Use `fieldId` not `fieldName` |
| `null value in column "X" violates not-null constraint` | Field values not being passed | Check `fieldId` spelling and data reference path |
| Workflow stops after `getAll` node | Query returned 0 rows | Add `alwaysOutputData: true` to node |
| Workflow stops after `create/update` node | Write operation failed | Add `continueOnFail: true` to node |
| Fields show as empty in UI | Wrong parameter structure | Use `fieldsUi.fieldValues` with `fieldId` |
| `At least one condition is required` | Get operation without filter | Add at least one condition to `filters.conditions` |
| `At least one select condition must be defined` | Update/delete using `filterType: "manual"` | Use `filterType: "string"` with `filterString` instead (see section 6.5) |
| `invalid input syntax for type uuid: ""` | Empty string `""` passed to a UUID column | Never use `\|\| ''` as fallback for UUID columns. Use `\|\| null` or omit the field entirely when the value may be absent |

### Nullable Field Handling

When a field value may be `null` or `undefined`, be careful with fallback expressions:

| Expression | Result when undefined | Use for |
|---|---|---|
| `={{ $json.field }}` | Sends `undefined` (field omitted) | Best default — safe for all column types |
| `={{ $json.field \|\| null }}` | Sends `null` | When you explicitly want SQL NULL |
| `={{ $json.field \|\| '' }}` | Sends empty string `""` | **ONLY** for text columns where empty string is acceptable. **NEVER** use for UUID, integer, or other typed columns |

For nullable UUID columns (like `organization_id`), the safest approach is to conditionally build the data in a Code node upstream and use `dataToSend: "autoMapInputData"`, so the field is simply absent when not applicable.

---

## 9. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 10. Requirements Checklist

- [ ] Credentials use INSTCRED pattern (`{{INSTCRED_SUPABASE_ID_DERCTSNI}}`)
- [ ] Operation uses valid value: `create`, `delete`, `get`, `getAll`, `update`
- [ ] **Never use `"insert"` as operation**
- [ ] Field definitions use `fieldId` (not `fieldName`)
- [ ] Data structure is `fieldsUi.fieldValues[]` (not `fieldsToSend`)
- [ ] Read operations include `alwaysOutputData: true`
- [ ] Write operations include `continueOnFail: true`
- [ ] **Update/delete operations use `filterType: "string"` with `filterString` (NOT `filterType: "manual"`)**
- [ ] **No `|| ''` fallback on UUID columns** — use `|| null` or omit the field
- [ ] HTTP Trigger data accessed via `$('HTTP Trigger').first().json.body`
- [ ] Error workflow configured
