# Placeholder Patterns Guide

Placeholders are template strings in workflow JSON files that get replaced with actual values at deployment or runtime. This guide provides the complete reference for all placeholder patterns.

---

## 1. Placeholder Categories Overview

| Category | Prefix | Suffix | Replacement Time | Example |
|----------|--------|--------|------------------|---------|
| Process Data | `PROCDATA_` | `_ATADCORP` | Deployment | Process ID, namespace |
| User Data | `USERDATA_` | `_ATADRESU` | Instance deployment | User ID, org ID |
| System Credentials | `SYSCREDS_` | `_SDERCSYS` | Deployment | Codika-managed AI keys |
| Organization Credentials | `ORGCRED_` | `_DERCGRO` | Deployment | Org-level integrations |
| Organization Secrets | `ORGSECRET_` | `_TERCESORG` | Deployment | n8n base URL |
| User Credentials | `USERCRED_` | `_DERCRESU` | Instance deployment | User's OAuth tokens |
| Flex Credentials | `FLEXCRED_` | `_DERCXELF` | Deployment | AI providers with fallback |
| Member Secrets | `MEMSECRT_` | `_TRCESMEM` | Instance deployment | Execution auth |
| Installation Parameters | `INSTPARM_` | `_MRAPTSNI` | Instance deployment | User config values |
| Instance Credentials | `INSTCRED_` | `_DERCTSNI` | Instance deployment | Per-deployment credentials |
| Sub-Workflow | `SUBWKFL_` | `_LFKWBUS` | Deployment | Sub-workflow n8n IDs |

---

## 2. Pattern Naming Convention

Placeholder suffixes are reversed versions of the prefixes:

| Prefix | Suffix (Reversed) |
|--------|-------------------|
| `USERDATA` | `ATADRESU` |
| `PROCDATA` | `ATADCORP` |
| `USERCRED` | `DERCRESU` |
| `SYSCREDS` | `SDERCSYS` |
| `ORGCRED` | `DERCGRO` |
| `ORGSECRET` | `TERCESORG` |
| `FLEXCRED` | `DERCXELF` |
| `MEMSECRT` | `TRCESMEM` |
| `INSTPARM` | `MRAPTSNI` |
| `INSTCRED` | `DERCTSNI` |
| `SUBWKFL` | `LFKWBUS` |

---

## 3. Process-Specific Patterns (PROCDATA)

Auto-populated at deployment time.

| Pattern | Description | Example Value |
|---------|-------------|---------------|
| `{{PROCDATA_PROCESS_ID_ATADCORP}}` | Process ID | `ZxeKcJatlTM23W9Aihgd` |
| `{{PROCDATA_NAMESPACE_ATADCORP}}` | Pinecone namespace | `process_ZxeKcJatlTM23W9Aihgd` |

---

## 4. Sub-Workflow References (SUBWKFL)

Reference sub-workflows by their template ID. Replaced with actual n8n workflow ID at deployment.

| Pattern | Description |
|---------|-------------|
| `{{SUBWKFL_<TEMPLATE_ID>_LFKWBUS}}` | Sub-workflow n8n ID reference |

**Example:**
```json
{
  "parameters": {
    "workflowId": "={{$json.subWorkflowId}}",
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {}
    }
  },
  "type": "n8n-nodes-base.executeWorkflow"
}
```

---

## 5. User-Specific Patterns (USERDATA)

Populated at instance deployment when a user installs the process.

| Pattern | Description |
|---------|-------------|
| `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` | User's process instance ID |
| `{{USERDATA_USER_ID_ATADRESU}}` | User's ID |
| `{{USERDATA_ORGANIZATION_ID_ATADRESU}}` | User's organization ID |

---

## 6. System Credentials (SYSCREDS)

Codika-managed credentials for AI services.

| Pattern | Description |
|---------|-------------|
| `{{SYSCREDS_ANTHROPIC_ID_SDERCSYS}}` | Anthropic API credential ID |
| `{{SYSCREDS_ANTHROPIC_NAME_SDERCSYS}}` | Anthropic credential name `[Codika]` |
| `{{SYSCREDS_OPENAI_ID_SDERCSYS}}` | OpenAI API credential ID |
| `{{SYSCREDS_OPENAI_NAME_SDERCSYS}}` | OpenAI credential name |

---

## 7. Organization Credentials (ORGCRED)

Credentials shared across all organization members.

| Pattern | Description |
|---------|-------------|
| `{{ORGCRED_<INTEGRATION>_ID_DERCGRO}}` | Org-level integration ID |
| `{{ORGCRED_<INTEGRATION>_NAME_DERCGRO}}` | Org-level integration name |

**Available Integrations:**

| Template Pattern | Integration UID | n8n Credential Type |
|------------------|-----------------|---------------------|
| `WHATSAPP` | whatsapp | whatsAppApi |
| `WHATSAPP_TRIGGER` | whatsapp_trigger | whatsAppTriggerApi |
| `SLACK` | slack | slackOAuth2Api |
| `FOLK` | folk | folkApi |
| `PIPEDRIVE` | pipedrive | pipedriveApi / pipedriveOAuth2Api |
| `TWILIO` | twilio | twilioApi |

**Examples:**

- `{{ORGCRED_WHATSAPP_ID_DERCGRO}}` - Organization's WhatsApp credential ID
- `{{ORGCRED_WHATSAPP_TRIGGER_ID_DERCGRO}}` - Organization's WhatsApp Trigger credential ID
- `{{ORGCRED_SLACK_ID_DERCGRO}}` - Organization's Slack credential ID
- `{{ORGCRED_FOLK_ID_DERCGRO}}` - Organization's Folk CRM credential ID
- `{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}` - Organization's Pipedrive credential ID

> **Custom integrations:** Custom integrations defined with `contextType: 'organization'` also use the `ORGCRED` pattern. For example, a custom integration with `id: 'cstm_acme_crm'` would use `{{ORGCRED_CSTM_ACME_CRM_ID_DERCGRO}}`.

---

## 8. Organization Secrets (ORGSECRET)

| Pattern | Description |
|---------|-------------|
| `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}` | n8n base URL for organization |
| `{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}` | n8n error workflow ID for error handling |

---

## 9. User Credentials (USERCRED)

Per-user credentials for integrations requiring personal OAuth tokens.

| Pattern | Description |
|---------|-------------|
| `{{USERCRED_<INTEGRATION>_ID_DERCRESU}}` | User's integration credential ID |
| `{{USERCRED_<INTEGRATION>_NAME_DERCRESU}}` | User's integration credential name |

**Available Integrations (Google):**

| Template Pattern | Integration UID |
|------------------|-----------------|
| `GOOGLE_CALENDAR` | google_calendar |
| `GOOGLE_CONTACTS` | google_contacts |
| `GOOGLE_DOCS` | google_docs |
| `GOOGLE_DRIVE` | google_drive |
| `GOOGLE_GMAIL` | google_gmail |
| `GOOGLE_SHEETS` | google_sheets |
| `GOOGLE_SLIDES` | google_slides |
| `GOOGLE_TASKS` | google_tasks |

**Available Integrations (Microsoft):**

| Template Pattern | Integration UID |
|------------------|-----------------|
| `MICROSOFT_EXCEL` | microsoft_excel |
| `MICROSOFT_ONEDRIVE` | microsoft_onedrive |
| `MICROSOFT_OUTLOOK` | microsoft_outlook |
| `MICROSOFT_SHAREPOINT` | microsoft_sharepoint |
| `MICROSOFT_TEAMS` | microsoft_teams |
| `MICROSOFT_TEAMS_ADVANCED` | microsoft_teams_advanced |
| `MICROSOFT_TO_DO` | microsoft_to_do |

**Available Integrations (Notion):**

| Template Pattern | Integration UID |
|------------------|-----------------|
| `NOTION` | notion |

**Available Integrations (Calendly):**

| Template Pattern | Integration UID |
|------------------|-----------------|
| `CALENDLY` | calendly |

**Example Usage:**

```json
"credentials": {
  "googleSheetsOAuth2Api": {
    "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
  }
}
```

**Notion:**
```json
"credentials": {
  "notionOAuth2Api": {
    "id": "{{USERCRED_NOTION_ID_DERCRESU}}",
    "name": "{{USERCRED_NOTION_NAME_DERCRESU}}"
  }
}
```

**Calendly:**
```json
"credentials": {
  "calendlyOAuth2Api": {
    "id": "{{USERCRED_CALENDLY_ID_DERCRESU}}",
    "name": "{{USERCRED_CALENDLY_NAME_DERCRESU}}"
  }
}
```

> **Custom integrations:** Custom integrations defined with `contextType: 'member'` also use the `USERCRED` pattern. For example, a custom integration with `id: 'cstm_personal_api'` would use `{{USERCRED_CSTM_PERSONAL_API_ID_DERCRESU}}`.

---

## 10. Flex Credentials (FLEXCRED)

AI providers with automatic fallback to Codika-managed credentials if users don't configure their own.

| Pattern | Description |
|---------|-------------|
| `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` | Anthropic API credential ID |
| `{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}` | Anthropic credential name |
| `{{FLEXCRED_OPENAI_ID_DERCXELF}}` | OpenAI API credential ID |
| `{{FLEXCRED_OPENAI_NAME_DERCXELF}}` | OpenAI credential name |
| `{{FLEXCRED_TAVILY_ID_DERCXELF}}` | Tavily API credential ID |
| `{{FLEXCRED_TAVILY_NAME_DERCXELF}}` | Tavily credential name |
| `{{FLEXCRED_GOOGLE_GEMINI_ID_DERCXELF}}` | Google Gemini API credential ID |
| `{{FLEXCRED_GOOGLE_GEMINI_NAME_DERCXELF}}` | Google Gemini credential name |
| `{{FLEXCRED_X_AI_ID_DERCXELF}}` | xAI API credential ID |
| `{{FLEXCRED_X_AI_NAME_DERCXELF}}` | xAI credential name |
| `{{FLEXCRED_OPEN_ROUTER_ID_DERCXELF}}` | OpenRouter API credential ID |
| `{{FLEXCRED_OPEN_ROUTER_NAME_DERCXELF}}` | OpenRouter credential name |
| `{{FLEXCRED_MISTRAL_ID_DERCXELF}}` | Mistral API credential ID |
| `{{FLEXCRED_MISTRAL_NAME_DERCXELF}}` | Mistral credential name |
| `{{FLEXCRED_COHERE_ID_DERCXELF}}` | Cohere API credential ID |
| `{{FLEXCRED_COHERE_NAME_DERCXELF}}` | Cohere credential name |
| `{{FLEXCRED_DEEP_SEEK_ID_DERCXELF}}` | DeepSeek API credential ID |
| `{{FLEXCRED_DEEP_SEEK_NAME_DERCXELF}}` | DeepSeek credential name |
| `{{FLEXCRED_FAL_AI_ID_DERCXELF}}` | fal AI API credential ID |
| `{{FLEXCRED_FAL_AI_NAME_DERCXELF}}` | fal AI credential name |
| `{{FLEXCRED_PINECONE_ID_DERCXELF}}` | Pinecone API credential ID |
| `{{FLEXCRED_PINECONE_NAME_DERCXELF}}` | Pinecone credential name |

### Usage Examples

**Anthropic:**
```json
"credentials": {
  "anthropicApi": {
    "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
    "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
  }
}
```

**OpenAI:**
```json
"credentials": {
  "openAiApi": {
    "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
  }
}
```

**Tavily:**
```json
"credentials": {
  "httpHeaderAuth": {
    "id": "{{FLEXCRED_TAVILY_ID_DERCXELF}}",
    "name": "{{FLEXCRED_TAVILY_NAME_DERCXELF}}"
  }
}
```

**Google Gemini:**
```json
"credentials": {
  "googlePalmApi": {
    "id": "{{FLEXCRED_GOOGLE_GEMINI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_GOOGLE_GEMINI_NAME_DERCXELF}}"
  }
}
```

**xAI:**
```json
"credentials": {
  "xAiApi": {
    "id": "{{FLEXCRED_X_AI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_X_AI_NAME_DERCXELF}}"
  }
}
```

**OpenRouter:**
```json
"credentials": {
  "openRouterApi": {
    "id": "{{FLEXCRED_OPEN_ROUTER_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPEN_ROUTER_NAME_DERCXELF}}"
  }
}
```

**Mistral:**
```json
"credentials": {
  "mistralCloudApi": {
    "id": "{{FLEXCRED_MISTRAL_ID_DERCXELF}}",
    "name": "{{FLEXCRED_MISTRAL_NAME_DERCXELF}}"
  }
}
```

**Cohere:**
```json
"credentials": {
  "cohereApi": {
    "id": "{{FLEXCRED_COHERE_ID_DERCXELF}}",
    "name": "{{FLEXCRED_COHERE_NAME_DERCXELF}}"
  }
}
```

**DeepSeek:**
```json
"credentials": {
  "deepSeekApi": {
    "id": "{{FLEXCRED_DEEP_SEEK_ID_DERCXELF}}",
    "name": "{{FLEXCRED_DEEP_SEEK_NAME_DERCXELF}}"
  }
}
```

**Pinecone:**
```json
"credentials": {
  "pineconeApi": {
    "id": "{{FLEXCRED_PINECONE_ID_DERCXELF}}",
    "name": "{{FLEXCRED_PINECONE_NAME_DERCXELF}}"
  }
}
```

---

## 11. Member Secrets (MEMSECRT)

Per-member secrets for authentication in schedule/third-party triggered workflows.

| Pattern | Description |
|---------|-------------|
| `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}` | Member's execution auth secret |

**Storage:** `organizations/{orgId}/members/{memberId}/secrets/executionAuth`

**Generation:** Automatically generated when a member is added to an organization.

**Usage:** Used in Codika Init node for schedule and third-party triggered workflows to authenticate back to the Codika platform.

---

## 12. Installation Parameters (INSTPARM)

Process instance parameterization - values users configure when installing a process.

### Placeholder Format

```text
{{INSTPARM_<PARAMETER_NAME>_MRAPTSNI}}
```

- `<PARAMETER_NAME>` must match the key in `deploymentInputSchema` (CONSTANT_CASE)
- Suffix `_MRAPTSNI` is "INSTPARM" reversed

### Value Type Usage

| Value Type | Serialized As | n8n Usage | Example |
|------------|---------------|-----------|---------|
| `string` | `"value"` (with quotes) | Unquoted | `{{INSTPARM_COMPANY_NAME_MRAPTSNI}}` |
| `text` | `"value"` (with quotes) | Unquoted | `{{INSTPARM_SYSTEM_PROMPT_MRAPTSNI}}` |
| `date` | `"2024-01-15"` (with quotes) | Unquoted | `{{INSTPARM_START_DATE_MRAPTSNI}}` |
| `select` | `"option_a"` (with quotes) | Unquoted | `{{INSTPARM_OUTPUT_FORMAT_MRAPTSNI}}` |
| `radio` | `"option_b"` (with quotes) | Unquoted | `{{INSTPARM_PRIORITY_MRAPTSNI}}` |
| `number` | `123` (no quotes) | Unquoted | `{{INSTPARM_MAX_ITEMS_MRAPTSNI}}` |
| `boolean` | `true` / `false` | Unquoted | `{{INSTPARM_ENABLE_FEATURE_MRAPTSNI}}` |
| `string[]` | `['a','b']` (single quotes) | Unquoted | `{{INSTPARM_ALLOWED_DOMAINS_MRAPTSNI}}` |
| `object` | `{'k':'v'}` (single quotes) | Unquoted | `{{INSTPARM_API_CONFIG_MRAPTSNI}}` |
| `object[]` | `[{'k':'v'}]` (single quotes) | Unquoted | `{{INSTPARM_TEAM_MEMBERS_MRAPTSNI}}` |

> **Note:** String types (string, text, date, select, radio) are serialized with JSON.stringify, which includes surrounding double quotes. This ensures proper handling of special characters including newlines in multiline text fields.

### Supported Field Types

| Type | Description |
|------|-------------|
| `string` | Single-line text input |
| `text` | Multi-line textarea (for long content, prompts) |
| `number` | Numeric input with min/max/step |
| `boolean` | Toggle/checkbox |
| `select` | Dropdown selection |
| `radio` | Radio button group |
| `array` | List of strings |
| `objectArray` | Array of structured objects |

### Common Mistakes

1. **Wrong format**: `{{DEPLOYMENT_PARAM_X}}` - INCORRECT
2. **Missing suffix**: `{{INSTPARM_X}}` - INCORRECT, must include `_MRAPTSNI`
3. **Wrong case**: `{{instparm_name_mraptsni}}` - INCORRECT, use UPPERCASE
4. **Adding extra quotes for strings**: `const x = '{{INSTPARM_NAME_MRAPTSNI}}'` - INCORRECT, string values already include quotes
5. **Adding quotes for arrays/objects**: `const arr = '{{INSTPARM_ITEMS_MRAPTSNI}}'` - INCORRECT, arrays/objects are unquoted but don't need extra quotes

### Context-Aware Serialization

INSTPARM placeholders work in **two contexts**, and the system automatically detects which one to use:

#### JSON Value Context
When the placeholder appears **directly in JSON** (outside quotes):
```json
"parameters": {
  "phoneNumber": {{INSTPARM_PHONE_MRAPTSNI}},
  "maxRetries": {{INSTPARM_MAX_RETRIES_MRAPTSNI}},
  "enableFeature": {{INSTPARM_ENABLE_FEATURE_MRAPTSNI}}
}
```

**Result** (after deployment):
```json
"parameters": {
  "phoneNumber": "+1234567890",
  "maxRetries": 5,
  "enableFeature": true
}
```

#### jsCode/String Context
When the placeholder appears **inside a JSON string** (inside quotes):
```json
"jsCode": "const phone = {{INSTPARM_PHONE_MRAPTSNI}}; const max = {{INSTPARM_MAX_RETRIES_MRAPTSNI}};"
```

**Result** (after deployment):
```json
"jsCode": "const phone = \"+1234567890\"; const max = 5;"
```

The double-escaping ensures the code is valid JavaScript after JSON parsing.

#### How It Works

The system counts unescaped quotes before the placeholder position:
- **Even count** → JSON value context → simple serialization
- **Odd count** → Inside string → escaped serialization

**Key point:** Use the same `{{INSTPARM_X_MRAPTSNI}}` syntax everywhere. The system handles the escaping automatically.

> **See:** [process-input-schema.md](./process-input-schema.md) for complete field type documentation.

---

## 13. Process Instance Credentials (INSTCRED)

Per-deployment credentials for complete data isolation between process installations.

| Pattern | Description |
|---------|-------------|
| `{{INSTCRED_<INTEGRATION>_ID_DERCTSNI}}` | Process instance integration credential ID |
| `{{INSTCRED_<INTEGRATION>_NAME_DERCTSNI}}` | Process instance integration credential name |

**Available Integrations:**

| Template Pattern | Integration UID | n8n Credential Type | Use Case |
|------------------|-----------------|---------------------|----------|
| `SUPABASE` | supabase | supabaseApi | Per-deployment Supabase connection |
| `POSTGRES` | postgres | postgres | Per-deployment PostgreSQL connection |
| `CSTM_*` | `cstm_*` | varies (httpHeaderAuth, httpBasicAuth, httpQueryAuth) | Per-deployment custom API connection |

**Usage (Supabase):**
```json
"credentials": {
  "supabaseApi": {
    "id": "{{INSTCRED_SUPABASE_ID_DERCTSNI}}",
    "name": "{{INSTCRED_SUPABASE_NAME_DERCTSNI}}"
  }
}
```

**Usage (PostgreSQL):**
```json
"credentials": {
  "postgres": {
    "id": "{{INSTCRED_POSTGRES_ID_DERCTSNI}}",
    "name": "{{INSTCRED_POSTGRES_NAME_DERCTSNI}}"
  }
}
```

**Usage (Custom Integration — e.g., cstm_acme_crm with contextType 'process_instance'):**
```json
"credentials": {
  "httpHeaderAuth": {
    "id": "{{INSTCRED_CSTM_ACME_CRM_ID_DERCTSNI}}",
    "name": "{{INSTCRED_CSTM_ACME_CRM_NAME_DERCTSNI}}"
  }
}
```

> **Note:** Custom integrations defined with `contextType: 'process_instance'` use the `INSTCRED` placeholder pattern. Custom integrations with `contextType: 'organization'` use `ORGCRED` (see Section 7), and those with `contextType: 'member'` use `USERCRED` (see Section 9). The integration ID in the placeholder is the `cstm_*` ID converted to UPPER_CASE.

**Storage Path:** `process_instances/{processInstanceId}/integrations/{integrationId}`

**n8n Credential Name Format:** `INSTCRED_{INTEGRATION_20}_{HASH_25}_DERCTSNI`

**How it works:**

1. User installs a process that requires a process instance-level integration
2. During installation, a dialog prompts for credentials
3. Credentials are encrypted and stored at `process_instances/{id}/integrations/{integration}`
4. Workflow is deployed with the unique credential reference
5. Each process instance operates with its own isolated connection

---

## 14. Credential Template Resolution

How credential placeholders are located and replaced at deployment time.

### Resolution Process

1. **Extract** integration type from placeholder using regex:
   ```javascript
   const CREDENTIAL_PATTERN = /\{\{USERCRED_([A-Z_]+)_(ID|NAME)_DERCRESU\}\}/g;
   ```

2. **Convert** to snake_case to match integration UID:
   ```javascript
   'GOOGLE_SHEETS' → 'google_sheets'
   ```

3. **Lookup** user's credential for that integration in Firestore

4. **Generate** n8n credential name:
   - **Member-level**: `USERCRED_{INTEGRATION_20}_{HASH_25}_DERCRESU` (64 chars max)
   - **Org-level**: `WVYZQWVY_{INTEGRATION_20}_{HASH_25}_YVWQZYVW`
   - Integration name trimmed to 20 characters max, converted to UPPERCASE
   - Hash is first 25 characters of SHA-256 of `orgId+memberId`

### Complete Example

Workflow template:
```json
{
  "credentials": {
    "googleSheetsOAuth2Api": {
      "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
      "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
    }
  }
}
```

After deployment:
- `{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}` → `"123"` (actual n8n credential ID)
- `{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}` → `"USERCRED_GOOGLE_SHEETS_a1b2c3d4e5f6g7h8i9j0k1l2m_DERCRESU"`

---

## 15. WhatsApp Workflow Best Practices

WhatsApp triggers fire for status updates and other events without actual messages. Add a Filter node after the WhatsApp Trigger:

```json
{
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "leftValue": "",
        "typeValidation": "loose",
        "version": 2
      },
      "conditions": [
        {
          "id": "has-messages",
          "leftValue": "={{ $json.messages }}",
          "rightValue": "",
          "operator": {
            "type": "array",
            "operation": "notEmpty"
          }
        }
      ],
      "combinator": "and"
    },
    "options": {}
  },
  "type": "n8n-nodes-base.filter",
  "typeVersion": 2.2,
  "name": "Has Messages?"
}
```

**Workflow structure:**
```text
WhatsApp Trigger → Has Messages? (Filter) → Extract Message Data → ...
```

---

## 16. Checklist

- [ ] Use correct prefix and suffix for each placeholder type
- [ ] Match CONSTANT_CASE in templates to snake_case integration UIDs
- [ ] Quote string INSTPARM values, leave arrays/objects unquoted
- [ ] Use FLEXCRED for AI providers (not ORGCRED)
- [ ] Use USERCRED for Google/Microsoft OAuth integrations
- [ ] Use ORGCRED for organization-level integrations (Slack, WhatsApp, Folk, Pipedrive)
- [ ] Use INSTCRED for per-deployment database connections
- [ ] Include MEMSECRT in Codika Init for schedule/third-party triggers
- [ ] Add message filter after WhatsApp Trigger nodes

---

## Related Documentation

- [http-triggers.md](./http-triggers.md) - HTTP trigger setup
- [schedule-triggers.md](./schedule-triggers.md) - Schedule trigger setup
- [process-input-schema.md](./process-input-schema.md) - INSTPARM field types
