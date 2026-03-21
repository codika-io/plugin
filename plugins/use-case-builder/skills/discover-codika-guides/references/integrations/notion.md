# Notion Integration Guide

Notion provides workspace management for databases, pages, blocks, and collaboration.

> **Implementation Reference:** [n8n Notion Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Notion)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `notion` |
| **Node Type** | `n8n-nodes-base.notion` |
| **Trigger Node Type** | `n8n-nodes-base.notionTrigger` |
| **Credential Type** | `notionOAuth2Api` |
| **Credential Level** | USERCRED (member-level OAuth) |
| **Node Version** | 2.2 |

### Credential Pattern

```json
"credentials": {
  "notionOAuth2Api": {
    "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
    "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
  }
}
```

---

## 2. Codika-Specific Tips

### Error Handling Flags

| Scenario | Flag | Why |
|----------|------|-----|
| Read operations that may return empty | `alwaysOutputData: true` | Prevents workflow from stopping on 0 results |
| Write operations that may fail | `continueOnFail: true` | Allows graceful error handling |

### Connection Requirement

In Notion, make sure to [add your connection](https://www.notion.so/help/add-and-manage-connections-with-the-api) to the pages you want to access. Without this, the API cannot access the pages/databases.

### Simplify Option

Most Notion operations have a `simple` option (default: `true`). When enabled, it returns a simplified version of the response. Disable it when you need the full raw Notion API response.

### Nested Blocks Limitation (CRITICAL)

The Notion API's `/blocks/{id}/children` endpoint **only returns direct children**, not recursively nested blocks.

**Example FAQ structure in Notion:**
```
- "What's the WiFi password?" (bulleted_list_item, has_children: true)
    - "The password is WAT2027!" (nested child - NOT returned by parent call)
- "Can I bring my dog?" (bulleted_list_item, has_children: true)
    - "Small dogs are allowed. Be mindful of others." (nested child)
```

**Problem:** A single API call to get the page's blocks only returns the top-level items (questions), not the nested answers.

**Solution:** Loop over blocks with `has_children: true` and fetch their children separately:

```
┌─────────────────────┐     ┌──────────────────────┐
│ Get Top-Level Blocks│     │ Prepare Parent IDs   │
│ (HTTP Request)      │ ──► │ (Code: outputs one   │
│                     │     │  item per parent)    │
└─────────────────────┘     └──────────────────────┘
                                      │
                                      ▼
                            ┌──────────────────────┐
                            │ Get Children         │
                            │ (HTTP - runs for     │ ──► Combine All Blocks
                            │  each parent)        │
                            └──────────────────────┘
```

**Code to identify parents with children:**
```javascript
const blocks = response.results || [];
const parentsWithChildren = blocks.filter(b => b.has_children === true);

// Output one item per parent for HTTP node to fetch
return parentsWithChildren.map(parent => ({
  json: { parentId: parent.id }
}));
```

**n8n Node Alternative:** The n8n Notion node has a `fetchNestedBlocks` option in `block:getAll`, but when using HTTP Request nodes directly (e.g., for more control), you must implement the loop yourself.

**Complete Example - FAQ Lookup Workflow:**

```
Execute Trigger
      │
      ▼
Extract Block ID (Code)     ← Parse Notion URL to get page ID
      │
      ▼
Get Top-Level Blocks (HTTP) ← GET /blocks/{pageId}/children
      │
      ▼
Prepare Parent IDs (Code)   ← Filter blocks with has_children: true
      │                        Output one item per parent
      ▼
  ┌───┴───┐
  │  IF   │ ── skipChildFetch: true ──► Skip Passthrough (Code)
  └───┬───┘                                      │
      │ false                                    │
      ▼                                          │
Get Children (HTTP)         ← Runs for EACH parent automatically
      │                        GET /blocks/{parentId}/children
      ▼                                          │
Combine All Blocks (Code)   ← Merge top-level + all children
      │                                          │
      ◄──────────────────────────────────────────┘
      │
      ▼
Convert to Markdown (Code)  ← Transform blocks to readable text
      │
      ▼
Answer Question (LLM Chain) ← Use FAQ content to answer
```

**Key insight:** The HTTP Request node automatically runs once per input item. So if "Prepare Parent IDs" outputs 5 items (5 parents with children), "Get Children" runs 5 times automatically - no loop node needed.

---

## 3. Available Resources & Operations

### Block (2 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Append After | `append` | Append blocks to a parent block |
| Get Child Blocks | `getAll` | Get many child blocks |

### Database (3 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Get | `get` | Get a database |
| Get Many | `getAll` | Get many databases |
| Search | `search` | Search databases using text search |

### Database Page (4 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Create | `create` | Create a page in a database |
| Get | `get` | Get a page in a database |
| Get Many | `getAll` | Get many pages in a database |
| Update | `update` | Update pages in a database |

### Page (3 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Archive | `archive` | Archive a page |
| Create | `create` | Create a page |
| Search | `search` | Text search of pages |

### User (2 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Get | `get` | Get a user |
| Get Many | `getAll` | Get many users |

### Trigger Events (2 events - polling trigger)

| Event | Value | Description |
|-------|-------|-------------|
| Page Added to Database | `pageAddedToDatabase` | Trigger when page is added |
| Page Updated in Database | `pagedUpdatedInDatabase` | Trigger when page is updated |

---

## 4. Parameter Reference (Complete)

This section documents **all** parameters available in the n8n Notion node.

### Common Patterns

#### Database Selection (resourceLocator)

```json
"databaseId": {
  "mode": "id",
  "value": "ab1545b247fb49fa92d6f4b49f4d8116"
}
```

**Available modes:**
- `list` - Select from dropdown
- `url` - Notion URL (e.g., `https://www.notion.so/0fe2f7de558b471eab07e9d871cdf4a9?v=...`)
- `id` - Database ID (e.g., `ab1545b247fb49fa92d6f4b49f4d8116`)

#### Page Selection (resourceLocator)

```json
"pageId": {
  "mode": "url",
  "value": "https://www.notion.so/My-Page-b4eeb113e118403aa450af65ac25f0b9"
}
```

**Available modes:**
- `url` - Notion page URL
- `id` - Page ID

#### Block Selection (resourceLocator)

```json
"blockId": {
  "mode": "id",
  "value": "ab1545b247fb49fa92d6f4b49f4d8116"
}
```

**Available modes:**
- `url` - Block URL (use "Copy link to block" in Notion)
- `id` - Block ID

---

### Block Resource Parameters

#### block:append

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Block | `blockId` | resourceLocator | Yes | - | The block to append to |
| Blocks | `blockUi` | fixedCollection | Yes | - | Blocks to append |

**Block Types:**
- `paragraph`, `heading_1`, `heading_2`, `heading_3`
- `bulleted_list_item`, `numbered_list_item`, `to_do`
- `toggle`, `code`, `quote`, `callout`, `divider`

#### block:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Block | `blockId` | resourceLocator | Yes | - | The block to get children from |
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |
| Also Fetch Nested Blocks | `fetchNestedBlocks` | boolean | No | false | Fetch nested blocks recursively |
| Simplify Output | `simplifyOutput` | boolean | No | true | Simplify response |

---

### Database Resource Parameters

#### database:get

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Database | `databaseId` | resourceLocator | Yes | - | The database to get |
| Simplify | `simple` | boolean | No | true | Simplify response |

#### database:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |
| Simplify | `simple` | boolean | No | true | Simplify response |

#### database:search

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Search Text | `text` | string | No | - | Text to search for |
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |
| Simplify | `simple` | boolean | No | true | Simplify response |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Sort | `sort.sortValue` | fixedCollection | Sort direction and timestamp |

---

### Database Page Resource Parameters

#### databasePage:create

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Database | `databaseId` | resourceLocator | Yes | - | The database to create page in |
| Title | `title` | string | No | - | Page title |
| Simplify | `simple` | boolean | No | true | Simplify response |
| Properties | `propertiesUi` | fixedCollection | No | - | Page properties |

**Property Types:**

| Type | Value Field | Description |
|------|-------------|-------------|
| Title | `title` | Page title property |
| Rich Text | `textContent` or `richText` blocks | Text content |
| Phone | `phoneValue` | Phone number (no format enforced) |
| Multi-Select | `multiSelectValue` | Multiple option selection |
| Select | `selectValue` | Single option selection |
| Status | `statusValue` | Status property |
| Email | `emailValue` | Email address |
| URL | `urlValue` | Web address |
| People | `peopleValue` | User references |
| Relation | `relationValue` | Related database page IDs |
| Checkbox | `checkboxValue` | Boolean true/false |
| Number | `numberValue` | Numeric value |
| Date | `date` / `dateStart` + `dateEnd` | Date or date range |
| Files | `fileUrls` | External file URLs |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Icon Type | `iconType` | options | `emoji` | `emoji` or `file` |
| Icon | `icon` | string | - | Emoji or file URL |

#### databasePage:get

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Database Page | `pageId` | resourceLocator | Yes | - | The page to get |
| Simplify | `simple` | boolean | No | true | Simplify response |

#### databasePage:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Database | `databaseId` | resourceLocator | Yes | - | The database to query |
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |
| Simplify | `simple` | boolean | No | true | Simplify response |
| Filter Match | `filterType` | options | No | `none` | `none`, `and`, or `or` |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Download Files | `downloadFiles` | boolean | Download files from file properties |
| Sort | `sort.sortValue` | fixedCollection | Sort by property or timestamp |

#### databasePage:update

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Database Page | `pageId` | resourceLocator | Yes | - | The page to update |
| Simplify | `simple` | boolean | No | true | Simplify response |
| Properties | `propertiesUi` | fixedCollection | No | - | Properties to update |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Icon Type | `iconType` | options | `emoji` | `emoji` or `file` |
| Icon | `icon` | string | - | Emoji or file URL |

---

### Page Resource Parameters

#### page:archive

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Page | `pageId` | resourceLocator | Yes | - | The page to archive |
| Simplify | `simple` | boolean | No | true | Simplify response |

#### page:create

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Parent Page | `pageId` | resourceLocator | Yes | - | Parent page for new page |
| Title | `title` | string | Yes | - | Page title |
| Simplify | `simple` | boolean | No | true | Simplify response |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Icon Type | `iconType` | options | `emoji` | `emoji` or `file` |
| Icon | `icon` | string | - | Emoji or file URL |

#### page:search

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Search Text | `text` | string | No | - | Text to search for |
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |
| Simplify | `simple` | boolean | No | true | Simplify response |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Filter | `filter.filters` | fixedCollection | Filter by object type (`page` or `database`) |
| Sort | `sort.sortValue` | fixedCollection | Sort direction and timestamp |

---

### User Resource Parameters

#### user:get

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| User ID | `userId` | string | Yes | - | The user ID to get |

#### user:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | false | Return all results |
| Limit | `limit` | number | No | 50 | Max results (1-100) |

---

### Trigger Parameters

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Event | `event` | options | Yes | `pageAddedToDatabase` | Event type to watch |
| Database | `databaseId` | resourceLocator | Yes | - | Database to watch |
| Simplify | `simple` | boolean | No | true | Simplify response |

**Event Options:**
- `pageAddedToDatabase` - Page Added to Database
- `pagedUpdatedInDatabase` - Page Updated in Database

---

## 5. Common Mistakes

| Wrong | Correct | Issue |
|-------|---------|-------|
| `databaseId: "abc123"` | `databaseId: { mode: "id", value: "abc123" }` | Must use resourceLocator |
| `pageId: "https://..."` | `pageId: { mode: "url", value: "https://..." }` | Must use resourceLocator |
| `operation: "query"` | `operation: "getAll"` | Wrong operation name for database pages |
| `resource: "databaseRow"` | `resource: "databasePage"` | Wrong resource name |
| Missing connection in Notion | Add integration to page | API cannot access unshared pages |

---

## 6. Node Examples

### 6.1 Get Database

```json
{
  "parameters": {
    "resource": "database",
    "operation": "get",
    "databaseId": {
      "mode": "id",
      "value": "={{ $json.databaseId }}"
    },
    "simple": true
  },
  "id": "notion-get-database-001",
  "name": "Get Database",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.2 Search Databases

```json
{
  "parameters": {
    "resource": "database",
    "operation": "search",
    "text": "={{ $json.searchQuery }}",
    "returnAll": false,
    "limit": 10,
    "simple": true,
    "options": {}
  },
  "id": "notion-search-databases-001",
  "name": "Search Databases",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.3 Create Database Page

```json
{
  "parameters": {
    "resource": "databasePage",
    "operation": "create",
    "databaseId": {
      "mode": "id",
      "value": "={{ $json.databaseId }}"
    },
    "title": "={{ $json.pageTitle }}",
    "simple": true,
    "propertiesUi": {
      "propertyValues": [
        {
          "key": "Status|status",
          "statusValue": "In Progress"
        },
        {
          "key": "Email|email",
          "emailValue": "={{ $json.email }}"
        }
      ]
    },
    "options": {}
  },
  "id": "notion-create-page-001",
  "name": "Create Database Page",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "continueOnFail": true
}
```

### 6.4 Get All Database Pages

```json
{
  "parameters": {
    "resource": "databasePage",
    "operation": "getAll",
    "databaseId": {
      "mode": "id",
      "value": "={{ $json.databaseId }}"
    },
    "returnAll": false,
    "limit": 50,
    "simple": true,
    "filterType": "none",
    "options": {}
  },
  "id": "notion-get-all-pages-001",
  "name": "Get All Database Pages",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.5 Update Database Page

```json
{
  "parameters": {
    "resource": "databasePage",
    "operation": "update",
    "pageId": {
      "mode": "id",
      "value": "={{ $json.pageId }}"
    },
    "simple": true,
    "propertiesUi": {
      "propertyValues": [
        {
          "key": "Status|status",
          "statusValue": "Done"
        }
      ]
    },
    "options": {}
  },
  "id": "notion-update-page-001",
  "name": "Update Database Page",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "continueOnFail": true
}
```

### 6.6 Create Page

```json
{
  "parameters": {
    "resource": "page",
    "operation": "create",
    "pageId": {
      "mode": "id",
      "value": "={{ $json.parentPageId }}"
    },
    "title": "={{ $json.pageTitle }}",
    "simple": true,
    "options": {
      "iconType": "emoji",
      "icon": "="
    }
  },
  "id": "notion-create-child-page-001",
  "name": "Create Page",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "continueOnFail": true
}
```

### 6.7 Search Pages

```json
{
  "parameters": {
    "resource": "page",
    "operation": "search",
    "text": "={{ $json.searchQuery }}",
    "returnAll": false,
    "limit": 25,
    "simple": true,
    "options": {
      "filter": {
        "filters": {
          "property": "object",
          "value": "page"
        }
      }
    }
  },
  "id": "notion-search-pages-001",
  "name": "Search Pages",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.8 Archive Page

```json
{
  "parameters": {
    "resource": "page",
    "operation": "archive",
    "pageId": {
      "mode": "id",
      "value": "={{ $json.pageId }}"
    },
    "simple": true
  },
  "id": "notion-archive-page-001",
  "name": "Archive Page",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "continueOnFail": true
}
```

### 6.9 Get Block Children

```json
{
  "parameters": {
    "resource": "block",
    "operation": "getAll",
    "blockId": {
      "mode": "id",
      "value": "={{ $json.blockId }}"
    },
    "returnAll": true,
    "fetchNestedBlocks": false,
    "simplifyOutput": true
  },
  "id": "notion-get-blocks-001",
  "name": "Get Block Children",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.10 Get User

```json
{
  "parameters": {
    "resource": "user",
    "operation": "get",
    "userId": "={{ $json.userId }}"
  },
  "id": "notion-get-user-001",
  "name": "Get User",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "position": [400, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  },
  "alwaysOutputData": true
}
```

### 6.11 Notion Trigger (Polling)

```json
{
  "parameters": {
    "event": "pageAddedToDatabase",
    "databaseId": {
      "mode": "id",
      "value": "={{ $json.databaseId }}"
    },
    "simple": true
  },
  "id": "notion-trigger-001",
  "name": "Notion Trigger",
  "type": "n8n-nodes-base.notionTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "credentials": {
    "notionOAuth2Api": {
      "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
      "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
    }
  }
}
```

---

## 7. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `Could not find database` | Database not shared with integration | Add integration to database in Notion settings |
| `Could not find page` | Page not shared with integration | Add integration to page or parent in Notion |
| `Invalid request URL` | Wrong database/page ID format | Use ID without dashes or the full URL |
| `Request failed with status code 400` | Invalid property type or value | Check property key format (`name|type`) |
| `Unauthorized` | Invalid or expired OAuth token | Re-authenticate Notion connection |
| Empty result from getAll | No matching items | Add `alwaysOutputData: true` to continue workflow |

---

## 8. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 9. Requirements Checklist

- [ ] Credentials use USERCRED pattern (`notionOAuth2Api`)
- [ ] Database/Page/Block references use resourceLocator with `mode` property
- [ ] Integration is connected to pages/databases in Notion
- [ ] Simplify option set appropriately for use case
- [ ] Read operations include `alwaysOutputData: true`
- [ ] Write operations include `continueOnFail: true`
- [ ] Property keys use format `name|type` when setting properties
- [ ] Error workflow configured
