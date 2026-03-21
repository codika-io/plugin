# Microsoft Services Integration Guide

Microsoft services include Outlook, Teams, Excel, OneDrive, SharePoint, and To Do. All use member-level OAuth credentials.

---

## 1. Integration Details

| Service | Integration UID | Node Type | Credential Type |
|---------|-----------------|-----------|-----------------|
| Outlook | `microsoft_outlook` | `n8n-nodes-base.microsoftOutlook` | `microsoftOutlookOAuth2Api` |
| Teams | `microsoft_teams` | `n8n-nodes-base.microsoftTeams` | `microsoftTeamsOAuth2Api` |
| Excel | `microsoft_excel` | `n8n-nodes-base.microsoftExcel` | `microsoftExcelOAuth2Api` |
| OneDrive | `microsoft_onedrive` | `n8n-nodes-base.microsoftOneDrive` | `microsoftOneDriveOAuth2Api` |
| SharePoint | `microsoft_sharepoint` | `n8n-nodes-base.microsoftSharePoint` | `microsoftSharePointOAuth2Api` |
| To Do | `microsoft_to_do` | `n8n-nodes-base.microsoftToDo` | `microsoftToDoOAuth2Api` |

**Credential Level:** USERCRED (member-level OAuth)

---

## 2. Credential Patterns

### Outlook

```json
"credentials": {
  "microsoftOutlookOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_OUTLOOK_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_OUTLOOK_NAME_DERCRESU}}"
  }
}
```

### Teams

```json
"credentials": {
  "microsoftTeamsOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_TEAMS_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_TEAMS_NAME_DERCRESU}}"
  }
}
```

### Excel

```json
"credentials": {
  "microsoftExcelOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_EXCEL_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_EXCEL_NAME_DERCRESU}}"
  }
}
```

### OneDrive

```json
"credentials": {
  "microsoftOneDriveOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_ONEDRIVE_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_ONEDRIVE_NAME_DERCRESU}}"
  }
}
```

### SharePoint

```json
"credentials": {
  "microsoftSharePointOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_SHAREPOINT_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_SHAREPOINT_NAME_DERCRESU}}"
  }
}
```

### To Do

```json
"credentials": {
  "microsoftToDoOAuth2Api": {
    "id": "{{USERCRED_MICROSOFT_TO_DO_ID_DERCRESU}}",
    "name": "{{USERCRED_MICROSOFT_TO_DO_NAME_DERCRESU}}"
  }
}
```

---

## 3. Node Examples

### 3.1 Send Outlook Email

```json
{
  "name": "Send Outlook Email",
  "type": "n8n-nodes-base.microsoftOutlook",
  "typeVersion": 2,
  "position": [600, 0],
  "parameters": {
    "resource": "message",
    "operation": "send",
    "toRecipients": "={{ $json.recipientEmail }}",
    "subject": "={{ $json.emailSubject }}",
    "bodyContent": "={{ $json.emailBody }}"
  },
  "credentials": {
    "microsoftOutlookOAuth2Api": {
      "id": "{{USERCRED_MICROSOFT_OUTLOOK_ID_DERCRESU}}",
      "name": "{{USERCRED_MICROSOFT_OUTLOOK_NAME_DERCRESU}}"
    }
  }
}
```

### 3.2 Post Teams Message

```json
{
  "name": "Post Teams Message",
  "type": "n8n-nodes-base.microsoftTeams",
  "typeVersion": 2,
  "position": [600, 0],
  "parameters": {
    "resource": "chatMessage",
    "operation": "create",
    "chatId": "={{ $json.chatId }}",
    "message": "={{ $json.message }}"
  },
  "credentials": {
    "microsoftTeamsOAuth2Api": {
      "id": "{{USERCRED_MICROSOFT_TEAMS_ID_DERCRESU}}",
      "name": "{{USERCRED_MICROSOFT_TEAMS_NAME_DERCRESU}}"
    }
  }
}
```

### 3.3 Read Excel Data

```json
{
  "name": "Read Excel Data",
  "type": "n8n-nodes-base.microsoftExcel",
  "typeVersion": 2,
  "position": [600, 0],
  "parameters": {
    "operation": "getRows",
    "workbook": {
      "__rl": true,
      "value": "={{ $json.workbookId }}",
      "mode": "id"
    },
    "worksheet": {
      "__rl": true,
      "value": "={{ $json.worksheetId }}",
      "mode": "id"
    },
    "options": {}
  },
  "credentials": {
    "microsoftExcelOAuth2Api": {
      "id": "{{USERCRED_MICROSOFT_EXCEL_ID_DERCRESU}}",
      "name": "{{USERCRED_MICROSOFT_EXCEL_NAME_DERCRESU}}"
    }
  }
}
```

### 3.4 Upload to OneDrive

```json
{
  "name": "Upload to OneDrive",
  "type": "n8n-nodes-base.microsoftOneDrive",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "operation": "upload",
    "fileName": "={{ $json.fileName }}",
    "binaryPropertyName": "data"
  },
  "credentials": {
    "microsoftOneDriveOAuth2Api": {
      "id": "{{USERCRED_MICROSOFT_ONEDRIVE_ID_DERCRESU}}",
      "name": "{{USERCRED_MICROSOFT_ONEDRIVE_NAME_DERCRESU}}"
    }
  }
}
```

---

## 4. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 5. Requirements Checklist

- [ ] Credentials use USERCRED pattern with `MICROSOFT_` prefix
- [ ] Credential type matches the service
- [ ] Resource IDs use `__rl` pattern with `mode: "id"` where applicable
- [ ] Error workflow configured
