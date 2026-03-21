# Google Services Integration Guide

Google services include Gmail, Sheets, Drive, Calendar, Docs, Slides, and Tasks. All use member-level OAuth credentials.

---

## 1. Integration Details

| Service | Integration UID | Node Type | Credential Type |
|---------|-----------------|-----------|-----------------|
| Gmail | `google_gmail` | `n8n-nodes-base.gmail` | `gmailOAuth2` |
| Sheets | `google_sheets` | `n8n-nodes-base.googleSheets` | `googleSheetsOAuth2Api` |
| Drive | `google_drive` | `n8n-nodes-base.googleDrive` | `googleDriveOAuth2Api` |
| Calendar | `google_calendar` | `n8n-nodes-base.googleCalendar` | `googleCalendarOAuth2Api` |
| Docs | `google_docs` | `n8n-nodes-base.googleDocs` | `googleDocsOAuth2Api` |
| Slides | `google_slides` | `n8n-nodes-base.googleSlides` | `googleSlidesOAuth2Api` |
| Tasks | `google_tasks` | `n8n-nodes-base.googleTasks` | `googleTasksOAuth2Api` |

**Credential Level:** USERCRED (member-level OAuth)

---

## 2. Credential Patterns

### Gmail

```json
"credentials": {
  "gmailOAuth2": {
    "id": "{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_GMAIL_NAME_DERCRESU}}"
  }
}
```

### Google Sheets

```json
"credentials": {
  "googleSheetsOAuth2Api": {
    "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
  }
}
```

### Google Drive

```json
"credentials": {
  "googleDriveOAuth2Api": {
    "id": "{{USERCRED_GOOGLE_DRIVE_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_DRIVE_NAME_DERCRESU}}"
  }
}
```

### Google Calendar

```json
"credentials": {
  "googleCalendarOAuth2Api": {
    "id": "{{USERCRED_GOOGLE_CALENDAR_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_CALENDAR_NAME_DERCRESU}}"
  }
}
```

---

## 3. Gmail Output Fields

The Gmail node returns email header fields in **PascalCase**: `From`, `To`, `Subject`, `Date`. Other fields use lowercase: `id`, `threadId`, `snippet`, `payload`, `labels`.

When referencing Gmail output in expressions or Code nodes, always use the correct casing:

```
✅  {{ $json.From }}       → "John Doe <john@example.com>"
✅  {{ $json.Subject }}    → "Meeting tomorrow"
✅  {{ $json.snippet }}    → "Hey, just wanted to..."
✅  {{ $json.id }}         → "19c33d0767444255"

❌  {{ $json.from }}       → undefined
❌  {{ $json.subject }}    → undefined
```

---

## 4. Gmail Examples

### 4.1 Get Authenticated User's Email

Codika Init returns `userId` which is a Firebase UID, **not** an email address. To get the user's actual email, call the Gmail profile API:

```json
{
  "name": "Get User Email",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "GET",
    "url": "https://www.googleapis.com/gmail/v1/users/me/profile",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "gmailOAuth2",
    "options": {}
  },
  "credentials": {
    "gmailOAuth2": {
      "id": "{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_GMAIL_NAME_DERCRESU}}"
    }
  }
}
```

Returns `{ "emailAddress": "user@example.com", ... }`. Reference with `$('Get User Email').first().json.emailAddress`.

### 4.2 Gmail Trigger

```json
{
  "name": "Gmail Trigger",
  "type": "n8n-nodes-base.gmailTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "parameters": {
    "pollTimes": {
      "item": [{ "mode": "everyMinute" }]
    },
    "filters": {}
  },
  "credentials": {
    "gmailOAuth2": {
      "id": "{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_GMAIL_NAME_DERCRESU}}"
    }
  }
}
```

### 4.3 Send Email (HTML)

The Gmail node sends plain text by default. **Always use `"emailType": "html"` and generate HTML content** — plain text and markdown render as a single unformatted line in email clients.

```json
{
  "name": "Send Email",
  "type": "n8n-nodes-base.gmail",
  "typeVersion": 2.1,
  "position": [600, 0],
  "parameters": {
    "resource": "message",
    "operation": "send",
    "sendTo": "={{ $json.recipientEmail }}",
    "subject": "={{ $json.emailSubject }}",
    "emailType": "html",
    "message": "={{ $json.emailBodyHtml }}"
  },
  "credentials": {
    "gmailOAuth2": {
      "id": "{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_GMAIL_NAME_DERCRESU}}"
    }
  }
}
```

### 4.4 HTML Email Formatting Rules

When a workflow sends emails, always generate HTML in a Code node upstream of the Gmail send node. Follow these rules:

1. **Use inline styles only** — email clients strip `<style>` blocks and ignore CSS classes
2. **Use `font-family` with web-safe fallbacks** — e.g., `-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif`
3. **Use `<table>` for layouts** — flexbox and grid are not supported in most email clients
4. **Wrap in a max-width container** — `<div style="max-width:600px;margin:0 auto">`
5. **Escape user-provided content** — prevent HTML injection from email subjects/senders

```javascript
// Example: Format Code node that generates HTML for email
const esc = (s) => (s || '').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');

let html = '<div style="font-family:-apple-system,BlinkMacSystemFont,\'Segoe UI\',Roboto,sans-serif;max-width:600px;margin:0 auto;color:#1a1a1a">';
html += '<h1 style="font-size:22px;border-bottom:2px solid #e5e5e5;padding-bottom:12px">Title</h1>';
html += '<table style="width:100%;border-collapse:collapse">';
for (const item of items) {
  html += `<tr style="border-bottom:1px solid #f0f0f0"><td style="padding:8px 0;font-weight:600;font-size:14px">${esc(item.label)}</td><td style="padding:8px 0;font-size:14px;color:#444">${esc(item.value)}</td></tr>`;
}
html += '</table></div>';

return [{ json: { emailBodyHtml: html } }];
```

---

## 5. Google Sheets Examples

### 5.1 Create Spreadsheet

```json
{
  "name": "Create Google Sheet",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "position": [400, 0],
  "parameters": {
    "resource": "spreadsheet",
    "operation": "create",
    "title": "={{ $json.sheetName }}"
  },
  "credentials": {
    "googleSheetsOAuth2Api": {
      "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
    }
  }
}
```

### 5.2 Append Data

```json
{
  "name": "Append to Sheet",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "position": [600, 0],
  "parameters": {
    "operation": "append",
    "documentId": {
      "__rl": true,
      "value": "={{ $json.spreadsheetId }}",
      "mode": "id"
    },
    "sheetName": {
      "__rl": true,
      "value": "gid=0",
      "mode": "id"
    },
    "columns": {
      "mappingMode": "autoMapInputData",
      "value": {},
      "matchingColumns": [],
      "schema": []
    },
    "options": {}
  },
  "credentials": {
    "googleSheetsOAuth2Api": {
      "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
    }
  }
}
```

### 5.3 Read Data

```json
{
  "name": "Read from Sheet",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "position": [600, 0],
  "parameters": {
    "operation": "read",
    "documentId": {
      "__rl": true,
      "value": "={{ $json.spreadsheetId }}",
      "mode": "id"
    },
    "sheetName": {
      "__rl": true,
      "value": "gid=0",
      "mode": "id"
    },
    "options": {
      "outputFormatting": true
    }
  },
  "credentials": {
    "googleSheetsOAuth2Api": {
      "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
    }
  }
}
```

---

## 6. Google Sheets Lazy Initialization Pattern

For workflows that log data to Google Sheets, use workflow static data to persist the sheet ID across executions.

### 6.1 Flow

```
Trigger → Check Init → [Needs Init?]
                           │
           ┌───────────────┴───────────────┐
           ↓ YES                           ↓ NO
   [Search Sheet]                          │
           │                               │
   [Sheet Exists?]                         │
           │                               │
 ┌─────────┴─────────┐                     │
 ↓ NO                ↓ YES                 │
[Create Sheet]   [Use Existing]            │
 │                   │                     │
[Add Headers]        │                     │
 │                   │                     │
[Store Init Data]    │                     │
 │                   │                     │
 └─────────┬─────────┘                     │
           ↓                               │
 [Merge Init Branches]                     │
           │                               │
           └───────────────┬───────────────┘
                           ↓
                 [Merge to Main Flow]
                           │
                           ↓
                 [Append/Read Sheet]
```

### 6.2 Check Initialization (Code Node)

```javascript
const CONFIG = {
  PROCESS_INSTANCE_UID: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}',
  SHEET_NAME: 'My_Workflow_Data_{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}'
};

const STORAGE_KEY = `init_data_${CONFIG.PROCESS_INSTANCE_UID}`;
const staticData = $getWorkflowStaticData('global');
const initData = staticData[STORAGE_KEY] || {};
const sheetId = initData.sheetId;

const inputData = $input.item.json;

return [{
  json: {
    ...inputData,
    _init: {
      needsInit: !sheetId,
      sheetId: sheetId || null,
      config: CONFIG,
      storageKey: STORAGE_KEY
    }
  }
}];
```

### 6.3 Search for Existing Sheet (HTTP Request)

Use Google Sheets credentials to access Drive API:

```json
{
  "parameters": {
    "method": "GET",
    "url": "https://www.googleapis.com/drive/v3/files",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "googleSheetsOAuth2Api",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        {
          "name": "q",
          "value": "=name='{{ $json._init.config.SHEET_NAME }}' and mimeType='application/vnd.google-apps.spreadsheet' and trashed=false"
        },
        {
          "name": "fields",
          "value": "files(id, name)"
        }
      ]
    }
  },
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "googleSheetsOAuth2Api": {
      "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
    }
  }
}
```

### 6.4 Store Initialization Data (Code Node)

```javascript
const inputData = $('Check Initialization').item.json;
const sheetData = $('Add Header Row').item.json;
const sheetId = sheetData.spreadsheetId;

const STORAGE_KEY = inputData._init.storageKey;
const staticData = $getWorkflowStaticData('global');
staticData[STORAGE_KEY] = { sheetId: sheetId };

return [{
  json: {
    ...inputData,
    _init: {
      ...inputData._init,
      sheetId: sheetId,
      initialized: true
    }
  }
}];
```

---

## 7. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 8. Requirements Checklist

- [ ] Credentials use USERCRED pattern with `GOOGLE_` prefix
- [ ] Credential type matches the service (e.g., `gmailOAuth2` for Gmail)
- [ ] Gmail fields use PascalCase (`From`, `To`, `Subject`) not lowercase
- [ ] Gmail send nodes use `"emailType": "html"` with HTML content (not markdown/plain text)
- [ ] Document/Sheet ID uses `__rl` pattern with `mode: "id"`
- [ ] Lazy initialization pattern used for sheet-based workflows
- [ ] Storage key includes process instance UID for isolation
- [ ] Error workflow configured
