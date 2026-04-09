# Integration Reference

Complete reference for all Codika platform integrations: built-in handlers, credential types, placeholder patterns, and CLI fields.

## Built-in vs Custom Integrations

There are two ways to connect an integration in a use case:

### Built-in Integrations (Codika Proxy)

The platform has a dedicated handler for each built-in integration. You just:
1. Add the integration ID to `integrationUids` in config.ts
2. Use the correct placeholder in your workflow JSON
3. Set credentials via `codika integration set <id> --secret KEY=VALUE`

The handler knows exactly which n8n credential type to create and which fields to send.

### Custom Integrations (`cstm_*`)

You define everything yourself in config.ts: the n8n credential type, field mapping, secret fields. Use this when:
- No built-in handler exists for the service you need
- The built-in handler is broken or missing fields
- You need a different n8n credential type than what the handler uses

See [config-patterns.md](./config-patterns.md#13-custom-integrations) for the full `CustomIntegrationSchema` reference.

### Key Differences

| | Built-in | Custom |
|---|---|---|
| Integration ID | `twilio` | `cstm_my_twilio` |
| config.ts | Just add to `integrationUids` | Define `customIntegrations` array with full schema |
| Workflow placeholder | `{{ORGCRED_TWILIO_ID_DERCGRO}}` | `{{ORGCRED_CSTM_MY_TWILIO_ID_DERCGRO}}` |
| n8n credential type | Hardcoded in handler | You choose (any valid type) |
| Field mapping | Handled by handler | You define `n8nCredentialMapping` |
| CLI setup | `codika integration set twilio` | `codika integration set cstm_my_twilio --path ./use-case` |
| What n8n receives | Same credential | Same credential |

Both approaches produce an n8n credential with a stored ID. The placeholder system resolves that ID at deployment time. From the workflow's perspective, they work identically.

### Switching from Built-in to Custom

If a built-in handler fails (e.g., missing fields), you can switch to a custom integration:

1. Use `codika integration schema <type>` to see what fields n8n needs
2. Define a `customIntegrations` entry in config.ts with the correct `n8nCredentialType` and `n8nCredentialMapping`
3. Update `integrationUids` to use the new `cstm_*` ID
4. Update all workflow JSON placeholders to use the new integration ID

---

## Placeholder Pattern Rules

The placeholder prefix depends on the **context type** (where the credential is stored), not the n8n credential type:

| Context Type | Placeholder Prefix | Suffix | Example |
|---|---|---|---|
| Organization | `ORGCRED` | `_DERCGRO` | `{{ORGCRED_TWILIO_ID_DERCGRO}}` |
| Member (OAuth) | `USERCRED` | `_DERCRESU` | `{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}` |
| Process Instance | `INSTCRED` | `_DERCTSNI` | `{{INSTCRED_SUPABASE_ID_DERCTSNI}}` |
| Flex (org/Codika fallback) | `FLEXCRED` | `_DERCXELF` | `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` |

Each placeholder has two variants:
- `_ID_` — replaced with the n8n credential ID (used in credential references)
- `_NAME_` — replaced with the credential name

The integration name in the placeholder is always **UPPER_CASE** (e.g., `GOOGLE_GMAIL`, `CSTM_MY_CRM`).

---

## Flex Credential Integrations (FLEXCRED)

Flex credentials let users choose between Codika's shared API keys (billed per usage) or their own organization keys. Use `FLEXCRED` in workflows when you want this flexibility.

| Integration ID | Name | n8n Credential Type | Placeholder | CLI Field |
|---|---|---|---|---|
| `anthropic` | Anthropic | `anthropicApi` | `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` | `ANTHROPIC_API_KEY` |
| `openai` | OpenAI | `openAiApi` | `{{FLEXCRED_OPENAI_ID_DERCXELF}}` | `OPENAI_API_KEY` |
| `tavily` | Tavily | `tavilyApi` | `{{FLEXCRED_TAVILY_ID_DERCXELF}}` | `TAVILY_API_KEY` |
| `google_gemini` | Google Gemini | `googlePalmApi` | `{{FLEXCRED_GOOGLE_GEMINI_ID_DERCXELF}}` | `GOOGLE_GEMINI_API_KEY` |
| `x_ai` | xAI | `xAiApi` | `{{FLEXCRED_X_AI_ID_DERCXELF}}` | `X_AI_API_KEY` |
| `open_router` | OpenRouter | `openRouterApi` | `{{FLEXCRED_OPEN_ROUTER_ID_DERCXELF}}` | `OPEN_ROUTER_API_KEY` |
| `mistral` | Mistral | `mistralCloudApi` | `{{FLEXCRED_MISTRAL_ID_DERCXELF}}` | `MISTRAL_API_KEY` |
| `cohere` | Cohere | `cohereApi` | `{{FLEXCRED_COHERE_ID_DERCXELF}}` | `COHERE_API_KEY` |
| `deep_seek` | DeepSeek | `deepSeekApi` | `{{FLEXCRED_DEEP_SEEK_ID_DERCXELF}}` | `DEEP_SEEK_API_KEY` |
| `fal_ai` | FAL AI | `falAiApi` | `{{FLEXCRED_FAL_AI_ID_DERCXELF}}` | — |
| `pinecone` | Pinecone | `pineconeApi` | `{{FLEXCRED_PINECONE_ID_DERCXELF}}` | `PINECONE_API_KEY` |

These integrations can also be used as `ORGCRED` if you want to force org-only credentials (no Codika fallback).

---

## Organization-Level Integrations (ORGCRED)

API key and credential-based integrations shared across the organization.

### Single API Key

| Integration ID | Name | n8n Credential Type | Placeholder | CLI Field |
|---|---|---|---|---|
| `openai` | OpenAI | `openAiApi` | `{{ORGCRED_OPENAI_ID_DERCGRO}}` | `OPENAI_API_KEY` |
| `anthropic` | Anthropic | `anthropicApi` | `{{ORGCRED_ANTHROPIC_ID_DERCGRO}}` | `ANTHROPIC_API_KEY` |
| `google_gemini` | Google Gemini | `googlePalmApi` | `{{ORGCRED_GOOGLE_GEMINI_ID_DERCGRO}}` | `GOOGLE_GEMINI_API_KEY` |
| `mistral` | Mistral | `mistralCloudApi` | `{{ORGCRED_MISTRAL_ID_DERCGRO}}` | `MISTRAL_API_KEY` |
| `deep_seek` | DeepSeek | `deepSeekApi` | `{{ORGCRED_DEEP_SEEK_ID_DERCGRO}}` | `DEEP_SEEK_API_KEY` |
| `cohere` | Cohere | `cohereApi` | `{{ORGCRED_COHERE_ID_DERCGRO}}` | `COHERE_API_KEY` |
| `x_ai` | xAI | `xAiApi` | `{{ORGCRED_X_AI_ID_DERCGRO}}` | `X_AI_API_KEY` |
| `open_router` | OpenRouter | `openRouterApi` | `{{ORGCRED_OPEN_ROUTER_ID_DERCGRO}}` | `OPEN_ROUTER_API_KEY` |
| `tavily` | Tavily | `tavilyApi` | `{{ORGCRED_TAVILY_ID_DERCGRO}}` | `TAVILY_API_KEY` |
| `pinecone` | Pinecone | `pineconeApi` | `{{ORGCRED_PINECONE_ID_DERCGRO}}` | `PINECONE_API_KEY` |
| `folk` | Folk CRM | `folkApi` | `{{ORGCRED_FOLK_ID_DERCGRO}}` | `FOLK_API_KEY` |
| `pipedrive` | Pipedrive | `pipedriveApi` | `{{ORGCRED_PIPEDRIVE_ID_DERCGRO}}` | `PIPEDRIVE_API_TOKEN` |

### Multi-Field Credentials

| Integration ID | Name | n8n Credential Type | Placeholder | CLI Fields |
|---|---|---|---|---|
| `twilio` | Twilio | `twilioApi` | `{{ORGCRED_TWILIO_ID_DERCGRO}}` | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` |
| `odoo` | Odoo | `odooApi` | `{{ORGCRED_ODOO_ID_DERCGRO}}` | `ODOO_URL`, `ODOO_DATABASE`, `ODOO_USERNAME`, `ODOO_PASSWORD` |
| `whatsapp` | WhatsApp | `whatsAppApi` | `{{ORGCRED_WHATSAPP_ID_DERCGRO}}` | `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_BUSINESS_ACCOUNT_ID` |
| `whatsapp_trigger` | WhatsApp Trigger | `whatsAppTriggerApi` | `{{ORGCRED_WHATSAPP_TRIGGER_ID_DERCGRO}}` | `WHATSAPP_CLIENT_ID`, `WHATSAPP_CLIENT_SECRET` |

### OAuth (Dashboard Only)

These require OAuth and cannot be configured from the CLI. Connect via the dashboard.

| Integration ID | Name | n8n Credential Type | Placeholder |
|---|---|---|---|
| `slack` | Slack | `slackOAuth2Api` | `{{ORGCRED_SLACK_ID_DERCGRO}}` |
| `shopify` | Shopify | `shopifyOAuth2Api` | `{{ORGCRED_SHOPIFY_ID_DERCGRO}}` |
| `calendly` | Calendly | `calendlyOAuth2Api` | `{{ORGCRED_CALENDLY_ID_DERCGRO}}` |

---

## Member-Level OAuth Integrations (USERCRED)

Per-user OAuth connections. Connected via the dashboard. Each member has their own credentials.

### Google

| Integration ID | Name | n8n Credential Type | Placeholder |
|---|---|---|---|
| `google_gmail` | Gmail | `gmailOAuth2` | `{{USERCRED_GOOGLE_GMAIL_ID_DERCRESU}}` |
| `google_calendar` | Google Calendar | `googleCalendarOAuth2Api` | `{{USERCRED_GOOGLE_CALENDAR_ID_DERCRESU}}` |
| `google_drive` | Google Drive | `googleDriveOAuth2Api` | `{{USERCRED_GOOGLE_DRIVE_ID_DERCRESU}}` |
| `google_sheets` | Google Sheets | `googleSheetsOAuth2Api` | `{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}` |
| `google_docs` | Google Docs | `googleDocsOAuth2Api` | `{{USERCRED_GOOGLE_DOCS_ID_DERCRESU}}` |
| `google_slides` | Google Slides | `googleSlidesOAuth2Api` | `{{USERCRED_GOOGLE_SLIDES_ID_DERCRESU}}` |
| `google_tasks` | Google Tasks | `googleTasksOAuth2Api` | `{{USERCRED_GOOGLE_TASKS_ID_DERCRESU}}` |
| `google_contacts` | Google Contacts | `googleContactsOAuth2Api` | `{{USERCRED_GOOGLE_CONTACTS_ID_DERCRESU}}` |

### Microsoft

| Integration ID | Name | n8n Credential Type | Placeholder |
|---|---|---|---|
| `microsoft_teams` | Teams | `microsoftTeamsOAuth2Api` | `{{USERCRED_MICROSOFT_TEAMS_ID_DERCRESU}}` |
| `microsoft_teams_advanced` | Teams Advanced | `microsoftTeamsOAuth2Api` | `{{USERCRED_MICROSOFT_TEAMS_ADVANCED_ID_DERCRESU}}` |
| `microsoft_outlook` | Outlook | `microsoftOutlookOAuth2Api` | `{{USERCRED_MICROSOFT_OUTLOOK_ID_DERCRESU}}` |
| `microsoft_sharepoint` | SharePoint | `microsoftSharePointOAuth2Api` | `{{USERCRED_MICROSOFT_SHAREPOINT_ID_DERCRESU}}` |
| `microsoft_onedrive` | OneDrive | `microsoftOneDriveOAuth2Api` | `{{USERCRED_MICROSOFT_ONEDRIVE_ID_DERCRESU}}` |
| `microsoft_to_do` | To Do | `microsoftToDoOAuth2Api` | `{{USERCRED_MICROSOFT_TO_DO_ID_DERCRESU}}` |
| `microsoft_excel` | Excel | `microsoftExcelOAuth2Api` | `{{USERCRED_MICROSOFT_EXCEL_ID_DERCRESU}}` |

### Other

| Integration ID | Name | n8n Credential Type | Placeholder |
|---|---|---|---|
| `notion` | Notion | `notionOAuth2Api` | `{{USERCRED_NOTION_ID_DERCRESU}}` |
| `calendly` | Calendly | `calendlyOAuth2Api` | `{{USERCRED_CALENDLY_ID_DERCRESU}}` |

---

## Process Instance-Level Integrations (INSTCRED)

Per-deployment credentials. Each process instance can have different credentials. Set via CLI with `--path` or `--process-instance-id`.

| Integration ID | Name | n8n Credential Type | Placeholder | CLI Fields |
|---|---|---|---|---|
| `supabase` | Supabase | `supabaseApi` | `{{INSTCRED_SUPABASE_ID_DERCTSNI}}` | `SUPABASE_HOST`, `SUPABASE_SERVICE_ROLE_KEY` |
| `postgres` | PostgreSQL | `postgres` | `{{INSTCRED_POSTGRES_ID_DERCTSNI}}` | `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DATABASE`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_SSL` (optional) |
| `telegram` | Telegram | `telegramApi` | `{{INSTCRED_TELEGRAM_ID_DERCTSNI}}` | `TELEGRAM_ACCESS_TOKEN` |

---

## n8n Credential Types Reference

Every credential type has a specific schema defining its required and optional fields. Use the CLI to discover them:

```bash
codika integration schema <credentialType>
```

### Common Credential Types

| n8n Credential Type | Used By | Fields |
|---|---|---|
| `openAiApi` | OpenAI | `apiKey` (required), `organizationId`, `url` |
| `anthropicApi` | Anthropic | `apiKey` (required) |
| `twilioApi` | Twilio | `authType`, `accountSid`, `authToken`, `allowedDomains` |
| `supabaseApi` | Supabase | `host` (required), `serviceRole` (required) |
| `postgres` | PostgreSQL | `host`, `port`, `database`, `user`, `password`, `ssl` |
| `telegramApi` | Telegram | `accessToken` (required) |
| `slackOAuth2Api` | Slack | OAuth fields (managed by platform) |
| `httpHeaderAuth` | Generic API key in header | `name`, `value` |
| `httpBasicAuth` | Generic username/password | `user`, `password` |
| `httpQueryAuth` | Generic API key in query param | `name`, `value` |
| `folkApi` | Folk CRM | `apiKey`, `allowedDomains` |
| `tavilyApi` | Tavily | `apiKey`, `allowedDomains`, `baseUrl` |
| `odooApi` | Odoo | `url`, `db`, `username`, `password` |
| `pineconeApi` | Pinecone | `apiKey` |
| `pipedriveApi` | Pipedrive | `apiToken` |
| `mistralCloudApi` | Mistral | `apiKey` |
| `deepSeekApi` | DeepSeek | `apiKey` |
| `cohereApi` | Cohere | `apiKey` |
| `xAiApi` | xAI | `apiKey` |
| `openRouterApi` | OpenRouter | `apiKey` |
| `googlePalmApi` | Google Gemini | `apiKey` |
| `whatsAppApi` | WhatsApp | `accessToken`, `businessAccountId` |
| `whatsAppTriggerApi` | WhatsApp Trigger | `clientId`, `clientSecret` |

For any credential type not listed here, run `codika integration schema <type>` to see the full schema with required fields and conditional rules.

---

## Related Documentation

- [config-patterns.md](./config-patterns.md#13-custom-integrations) — CustomIntegrationSchema definition and examples
- [placeholder-patterns.md](./placeholder-patterns.md) — All placeholder types and replacement rules
- [Integration guides](../integrations/) — Per-integration guides with node examples and configuration details
