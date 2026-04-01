# Use Case Creation Guide

This guide explains how to create use cases (processes) for the Codika platform. Read this guide first to understand the architecture, mandatory patterns, and key concepts. Specific guides are referenced for implementation details.

---

## 1. What You're Building

### What is a Process?

A **process** in Codika is a deployed use case containing n8n workflows. When you create a use case:

1. You define configuration (`config.ts`) + workflow JSON files
2. You deploy → creates a **Process** (discoverable in the store)
3. Users install → get their own **ProcessInstance**
4. Workflows execute via triggers (HTTP, schedule, service events)

### Components of a Use Case

| Component | Purpose |
|-----------|---------|
| `config.ts` | Defines workflows, triggers, input/output schemas |
| Workflow JSON files | n8n workflow definitions |

### Folder Structure

```
use-cases/
└── your-use-case/
    ├── config.ts              # Configuration
    ├── workflows/
    │   └── your-workflow.json # n8n workflow
    ├── skills/                # Optional — agent skills for triggerable workflows
    │   └── your-workflow/
    │       └── SKILL.md       # Claude-compatible skill file
    └── data-ingestion/        # Optional — process-level DI (deployed separately)
        └── embedding.json
```

---

## 2. Mandatory Workflow Pattern

**CRITICAL: Every parent workflow MUST follow this pattern:**

```
Trigger → Codika Init → [Your Business Logic] → Codika Submit Result / Codika Report Error
```

### Why This Pattern is Required

**Codika Init** (immediately after trigger):

- Registers the execution with the Codika platform
- Provides `executionId` and `executionSecret` for authentication
- Without it: The platform doesn't know the workflow is running

**Codika Submit Result** (end of success paths):

- Reports successful completion to the platform
- Sends result data back to the user
- Without it: Execution hangs as "pending" forever

**Codika Report Error** (end of error paths):

- Reports failure to the platform
- Provides error details for debugging
- Without it: User sees no feedback on failure

### What Happens If You Skip These Nodes

| Missing Node | Consequence |
|--------------|-------------|
| No Codika Init | Platform doesn't track execution, credentials unavailable |
| No Submit/Error | Execution shows as "pending" indefinitely |
| Submit on error path | User sees success when workflow failed |

### Exception: Sub-Workflows

Sub-workflows are called by other workflows and do NOT have Codika Init. They:

- Start with an Execute Workflow Trigger node
- Receive execution context from the parent (when using Codika nodes)
- Return output directly to the parent workflow

→ See [sub-workflows.md](./specific/sub-workflows.md) for sub-workflow implementation

---

## 3. The Placeholder System

### Why Placeholders Exist

Workflows cannot contain hardcoded secrets or user-specific values because:

- **Security**: API keys and secrets must never be in workflow JSON
- **Multi-tenancy**: Each user has different credentials and IDs
- **Deployment**: Process IDs aren't known until deployment time

Placeholders are special strings that get replaced with actual values.

### When Replacement Happens

| Timing | Placeholder Type | Example |
|--------|------------------|---------|
| At deployment | Process data | `{{PROCDATA_PROCESS_ID_ATADCORP}}` |
| At user installation | User data | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` |
| At runtime | Credentials | `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` |

### Placeholder Categories

#### Organization Secrets (`ORGSECRET_`)

Organization-level configuration values:

- `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}` - n8n base URL
- `{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}` - Error workflow ID

Used in webhook URLs and error handling configuration.

#### Member Secrets (`MEMSECRT_`)

Per-member authentication secrets:

- `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}` - Execution authentication secret

Used in Codika Init for schedule/third-party triggers.

#### User Data (`USERDATA_`)

User-specific values, replaced at installation:

- `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` - User's process instance ID
- `{{USERDATA_USER_ID_ATADRESU}}` - User's ID
- `{{USERDATA_ORGANIZATION_ID_ATADRESU}}` - User's organization ID

Used in webhook paths and Codika Init parameters.

#### Process Data (`PROCDATA_`)

Process-specific values, replaced at deployment:

- `{{PROCDATA_PROCESS_ID_ATADCORP}}` - Process ID
- `{{PROCDATA_NAMESPACE_ATADCORP}}` - Pinecone namespace for RAG

Used in webhook paths and RAG configuration.

#### Flex Credentials (`FLEXCRED_`)

AI provider credentials with automatic fallback:

- `{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}` / `{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}`
- `{{FLEXCRED_OPENAI_ID_DERCXELF}}` / `{{FLEXCRED_OPENAI_NAME_DERCXELF}}`
- `{{FLEXCRED_PINECONE_ID_DERCXELF}}` / `{{FLEXCRED_PINECONE_NAME_DERCXELF}}`
- Other providers: Tavily, Google Gemini, etc.

Flex credentials allow processes to work out-of-the-box. If the user hasn't configured their own API key, Codika provides a default.

**Important:** For HTTP request nodes calling Pinecone or OpenAI APIs directly, use `predefinedCredentialType` with the appropriate `nodeCredentialType` (e.g., `pineconeApi`, `openAiApi`) — never hardcode credential IDs.

#### Organization Credentials (`ORGCRED_`)

Organization-level integration credentials:

- `{{ORGCRED_<INTEGRATION>_ID_DERCGRO}}` - Credential ID
- `{{ORGCRED_<INTEGRATION>_NAME_DERCGRO}}` - Credential name

For integrations like Slack, WhatsApp, Folk CRM, Pipedrive.

#### User Credentials (`USERCRED_`)

User-specific integration credentials:

- `{{USERCRED_<INTEGRATION>_ID_DERCRESU}}` - Credential ID
- `{{USERCRED_<INTEGRATION>_NAME_DERCRESU}}` - Credential name

For user-connected services like Gmail, Google Calendar.

#### Sub-Workflow References (`SUBWKFL_`)

References to sub-workflows within the same process:

- `{{SUBWKFL_<TEMPLATE_ID>_LFKWBUS}}` - n8n workflow ID

Replaced at deployment with actual n8n workflow IDs.

### Using Placeholders in Workflow JSON

**In credentials:**

```json
"credentials": {
  "anthropicApi": {
    "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
    "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
  }
}
```

**In webhook paths:**

```json
"path": "{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/analyze"
```

**In Codika Init parameters:**

```json
"processInstanceId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
```

→ See [placeholder-patterns.md](./specific/placeholder-patterns.md) for complete pattern reference

---

## 4. Codika Nodes

Codika provides custom n8n nodes for platform integration. These are NOT optional helper nodes - they ARE the integration with the Codika platform.

### Available Nodes

| Node | Operation | Purpose |
|------|-----------|---------|
| Codika Init | `initWorkflow` | Initialize execution tracking |
| Codika Init Data Ingestion | `initDataIngestion` | Initialize RAG ingestion workflows |
| Codika Submit Result | `submitResult` | Report successful completion |
| Codika Report Error | `reportError` | Report workflow failure |
| Codika Upload File | `uploadFile` | Upload files to knowledge base |
| Codika Ingestion Callback | `ingestionCallback` | Report RAG ingestion status |

### Codika Init (`initWorkflow`)

**When required:** First node after trigger in ALL parent workflows.

**What it does:**

- For HTTP triggers: Extracts `executionMetadata` from the webhook payload
- For schedule/third-party triggers: Creates a new execution via API

**What it outputs:**

- `executionId` - Unique identifier for this execution
- `executionSecret` - Authentication secret for Codika API calls
- `callbackUrl` - URL for submitting results
- `processInstanceId`, `userId`, `workflowId` - Context information

**Why required:** Without Codika Init, the platform cannot track the execution and Codika nodes cannot authenticate.

### Codika Submit Result (`submitResult`)

**When required:** End of every success path in parent workflows.

**What it does:**

- Sends result data back to the Codika platform
- Marks execution as successful
- Makes results available to the user in the UI

**Input:** `resultData` - JSON object matching the workflow's output schema.

**Why required:** Without Submit Result, the execution remains "pending" and the user never sees results.

### Codika Report Error (`reportError`)

**When required:** End of every error/failure path in parent workflows.

**What it does:**

- Reports the error to the Codika platform
- Marks execution as failed
- Provides error details for debugging

**Input:**

- `errorMessage` - Human-readable error description
- `errorType` - Category: `node_failure`, `validation_error`, `external_api_error`, `timeout`

**Why required:** Without Report Error, failures appear as hanging executions with no feedback.

### Codika Upload File (`uploadFile`)

**When required:** When the workflow generates files (videos, PDFs, images) that need to be stored.

**What it does:**

- Uploads binary file data to Codika storage
- Returns a `documentId` for referencing the file in results

**Note:** In sub-workflows, requires `executionIdOverride` and `executionSecretOverride` parameters since there's no Codika Init.

→ See [FILE_UPLOAD_GUIDE.md](../.claude/docs/features/file_upload/FILE_UPLOAD_GUIDE.md) for file upload implementation

### Codika Init Data Ingestion (`initDataIngestion`)

**When required:** First node in RAG data ingestion workflows.

**What it does:**

- Initializes execution tracking for document processing
- Handles batch processing of knowledge base documents

→ See [RAG_INTEGRATION_GUIDE.md](../.claude/docs/features/rag_integration/RAG_INTEGRATION_GUIDE.md) for RAG implementation

### Required vs Optional

| Node | When Required |
|------|---------------|
| Codika Init | ALL parent workflows (first after trigger) |
| Codika Submit Result | ALL parent workflows (success paths) |
| Codika Report Error | ALL parent workflows (error paths) |
| Codika Upload File | When generating files |
| Codika Init Data Ingestion | RAG ingestion workflows only |

→ See [codika-nodes.md](./specific/codika-nodes.md) for parameter tables and configuration examples

---

## 5. Trigger Types

Triggers define how workflows are started. Each trigger type has different setup requirements.

### HTTP Triggers

**Use case:** User-initiated workflows via the Codika UI.

**How it works:**

1. User fills out a form (defined by `inputSchema`)
2. Codika sends POST request to the webhook
3. Workflow executes and returns results to the user

**Characteristics:**

- User sees a form in the UI based on `inputSchema`
- Results displayed based on `outputSchema`
- Synchronous: user waits for completion

**Webhook pattern:**

```
{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-endpoint
```

**Webhook endpoint security:** All webhook nodes **must** use `authentication: "headerAuth"` with the Codika webhook auth credential. This ensures only the Codika platform can trigger n8n workflows — direct calls without the auth header are rejected with 403. The credential placeholders are:
- `{{ORGSECRET_WEBHOOK_AUTH_CRED_ID_TERCESORG}}` — credential ID
- `{{ORGSECRET_WEBHOOK_AUTH_CRED_NAME_TERCESORG}}` — credential name

→ See [http-triggers.md](./specific/http-triggers.md) for HTTP trigger implementation

#### Public API Access (External Triggering)

HTTP trigger workflows can also be triggered externally without Firebase authentication, using an API key. This enables integration with third-party systems, cron services, Zapier, Make, or any HTTP client.

**How it works:**

1. Each process instance gets a unique API key (auto-generated on creation, stored in `process_instances/{id}/secrets/apiKey`)
2. External systems call the public endpoint with the API key
3. The endpoint uses stable identifiers (`processInstanceId` + `workflowId`) that don't change across version updates

**Trigger endpoint:**

```
POST https://europe-west1-{project}.cloudfunctions.net/triggerWebhookPublic/{processInstanceId}/{workflowId}
Header: X-API-Key: ck_...
Body: { "payload": { "field1": "value1" } }
→ Returns: { "success": true, "executionId": "..." }
```

The `payload` wrapper is recommended but not required. If no `payload` key is present, the entire request body is passed through as-is (useful for third-party webhooks like Evolution API or Stripe).

**Status polling endpoint:**

```
GET https://europe-west1-{project}.cloudfunctions.net/getExecutionStatusPublic/{processInstanceId}/{executionId}
Header: X-API-Key: ck_...
→ Returns: { "execution": { "status": "success", "resultData": {...} } }
```

**Key points:**
- The `workflowId` is the `workflowTemplateId` from `config.ts` (e.g., `'my-workflow'`) — stable across versions
- When using the `payload` wrapper, the `payload` fields should match the workflow's `inputSchema` keys
- API keys use format `ck_<base64url>` (384 bits entropy)
- Keys can be regenerated from the UI (in the trigger panel's "Public API Access" section)
- The public URL, API key, and ready-to-use cURL commands are shown in the playground trigger panel

→ See the PROCESS_SYSTEM_ARCHITECTURE.md Section 7.8 for full technical details

### Schedule Triggers

**Use case:** Automated workflows running on a schedule (cron).

**How it works:**

1. n8n Schedule Trigger fires at configured times
2. Codika Init creates an execution (no incoming payload)
3. Workflow executes automatically

**Characteristics:**

- No user input (no `inputSchema` needed in trigger)
- Results stored for later viewing
- Asynchronous: runs in background

**Key difference:** Codika Init uses placeholders directly (not `$json.body.executionMetadata`) because there's no incoming HTTP payload.

→ See [schedule-triggers.md](./specific/schedule-triggers.md) for schedule trigger implementation

### Third-Party Triggers (Gmail, Calendly, etc.)

**Use case:** Workflows triggered by external services.

**How it works:**

1. External service (Gmail, Calendly) sends data to n8n
2. Codika Init creates an execution
3. Workflow processes the external data

**Characteristics:**

- Similar to schedule triggers (Codika Init creates execution)
- Input data comes from the external service
- May require user to connect their account (e.g., Gmail OAuth)
- Some triggers require a `webhookId` property for proper routing

**Examples:** Gmail trigger (new email), Calendly trigger (new booking), Webhook from external service.

→ See [third-party-triggers.md](./specific/third-party-triggers.md) for third-party trigger implementation

### Sub-Workflow Triggers

**Use case:** Workflows called by other workflows.

**How it works:**

1. Parent workflow calls sub-workflow via Execute Workflow node
2. Sub-workflow receives input from parent
3. Sub-workflow returns output to parent

**Characteristics:**

- NOT visible in UI (users can't trigger directly)
- No Codika Init node
- `cost: 0` (parent pays)

→ See [sub-workflows.md](./specific/sub-workflows.md) for sub-workflow implementation

### Data Ingestion Triggers

**Use case:** Instance-level data ingestion workflows that embed per-user KB documents into a vector store.

**How it works:**

1. User uploads a document to their "My Files" KB
2. Platform parses the document
3. Firestore trigger detects parsing completion and calls the DI webhook
4. n8n workflow embeds the document into Pinecone

**Characteristics:**

- NOT visible in UI (internal trigger only)
- Uses `Codika Init Data Ingestion` instead of `Codika Init`
- Uses `Codika Ingestion Callback` instead of `Codika Submit Result`
- Deployed with the process (not separately like process-level DI)

→ See [data-ingestion.md](./specific/data-ingestion.md) for complete data ingestion guide

### Choosing a Trigger Type

| Scenario | Trigger Type |
|----------|--------------|
| User fills form, sees results | HTTP |
| Runs automatically on schedule | Schedule |
| Responds to external events | Third-party |
| Reusable logic called by other workflows | Sub-workflow |
| Embed instance KB documents into vector store | Data Ingestion |

---

## 6. Configuration (config.ts)

The `config.ts` file defines your use case. It exports a `getConfiguration()` function that returns the full configuration.

### What config.ts Defines

| Field | Purpose |
|-------|---------|
| `title`, `subtitle`, `description` | Display metadata for the process store |
| `workflows[]` | Array of workflow configurations |
| `workflows[].workflowTemplateId` | Unique identifier for the workflow |
| `workflows[].triggers[]` | How the workflow can be triggered |
| `workflows[].triggers[].inputSchema` | Form fields for user input (HTTP triggers) |
| `workflows[].outputSchema` | Structure of workflow results |
| `workflows[].n8nWorkflowJsonBase64` | Base64-encoded workflow JSON |
| `workflows[].cost` | Credit cost per execution |

### Workflow Configuration

```typescript
import { loadAndEncodeWorkflow } from 'codika';

// In getConfiguration():
{
  workflowTemplateId: 'my-workflow',
  workflowName: 'My Workflow',
  triggers: [
    {
      triggerId: crypto.randomUUID(),
      type: 'http',
      title: 'Run Analysis',
      description: 'Analyze the provided text',
      url: webhookUrl,
      method: 'POST',
      inputSchema: getInputSchema(),
    }
  ],
  outputSchema: getOutputSchema(),
  n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./workflows/my-workflow.json'),
  cost: 1,
}
```

### Input Schema

Defines the form fields users see when triggering an HTTP workflow. All field properties are defined at the top level of each field object:

```typescript
function getInputSchema(): FormInputSchema {
  return [
    {
      type: 'section',
      title: 'Input',
      description: 'Provide input for analysis',
      collapsible: false,
      inputSchema: [
        {
          key: 'text_input',
          type: 'text',
          label: 'Text to analyze',
          description: 'Enter the text you want to analyze',
          placeholder: 'Paste your text here...',
          required: true,
          maxLength: 5000,
          rows: 6,
        },
        {
          key: 'analysis_mode',
          type: 'select',
          label: 'Analysis Mode',
          required: true,
          options: [
            { value: 'summary', label: 'Summarize' },
            { value: 'sentiment', label: 'Sentiment Analysis' },
          ],
        },
      ],
    },
  ];
}
```

**Field types:** `string`, `text`, `number`, `boolean`, `date`, `select`, `multiselect`, `radio`, `file`, `array`, `object`, `objectArray`

→ See [http-triggers.md](./specific/http-triggers.md) for complete field type reference with all parameters

### Output Schema

Defines the structure of results displayed to users:

```typescript
function getOutputSchema(): FormOutputSchema {
  return [
    {
      key: 'result',
      type: 'text',
      label: 'Analysis Result',
      description: 'The analysis output',
    },
    {
      key: 'confidence',
      type: 'number',
      label: 'Confidence Score',
      description: 'Confidence level (0-1)',
    },
  ];
}
```

The keys in output schema must match what `Codika Submit Result` sends in `resultData`.

### Sub-Workflow Configuration

Sub-workflows have a different trigger configuration:

```typescript
{
  workflowTemplateId: 'helper-workflow',
  triggers: [
    {
      triggerId: crypto.randomUUID(),
      type: 'subworkflow',
      title: 'Helper',
      description: 'Called by other workflows',
      inputSchema: [
        { key: 'input_text', type: 'string' },
      ],
      calledBy: ['main-workflow'],
    }
  ],
  outputSchema: [],  // Sub-workflows return to parent
  cost: 0,  // Parent pays
}
```

→ See [config-patterns.md](./specific/config-patterns.md) for complete field reference

---

## 7. Deployment Parameters (Process Installation)

### What Are Deployment Parameters?

Deployment parameters let users customize their process installation. When a user installs a process, they can configure settings that persist across all workflow executions.

**Important distinction:**

- **Trigger input schema** (`inputSchema` in triggers): Form fields users fill when **running** a workflow
- **Deployment input schema** (`getDeploymentInputSchema()`): Form fields users fill when **installing** the process

### When to Use

Use deployment parameters when:

- Process behavior should vary per installation (e.g., company name, limits)
- Settings should persist across all executions
- Users need to configure API endpoints, credentials references, or thresholds

### How It Works

1. Define schema in `config.ts` via `getDeploymentInputSchema()`
2. User sees installation form based on schema
3. Values stored in `ProcessInstance.deploymentParameters`
4. `INSTPARM_` placeholders replaced with actual values
5. Workflows access configured values at runtime

### Defining in config.ts

```typescript
export function getDeploymentInputSchema(): DeploymentInputSchema {
  return [
    {
      type: 'section',
      title: 'Settings',
      inputSchema: [
        {
          key: 'COMPANY_NAME',
          type: 'string',
          label: 'Company Name',
          required: true,
          defaultValue: 'Demo Corp',
        },
        {
          key: 'MAX_ITEMS',
          type: 'number',
          label: 'Max Items',
          defaultValue: 10,
          min: 1,
          max: 100,
        },
      ],
    },
  ];
}

export function getDefaultDeploymentParameters(): DeploymentParameterValues {
  return {
    COMPANY_NAME: 'Demo Corp',
    MAX_ITEMS: 10,
  };
}
```

### Using in Workflows (INSTPARM Placeholders)

**jsCode context** (inside Code node strings):

```javascript
// String values - NO extra quotes (value includes quotes via JSON.stringify)
const companyName = {{INSTPARM_COMPANY_NAME_MRAPTSNI}};

// Number/boolean values - unquoted
const maxItems = {{INSTPARM_MAX_ITEMS_MRAPTSNI}};
```

**JSON value context** (directly in node parameters):

```json
"parameters": {
  "value": {{INSTPARM_MAX_ITEMS_MRAPTSNI}}
}
```

> **Note:** The system automatically detects which context the placeholder is in and serializes appropriately. Use the same syntax in both contexts.

### Field Types

| Type | Description |
|------|-------------|
| `string` | Single-line text |
| `text` | Multi-line textarea (for long content, prompts, etc.) |
| `number` | Numeric with min/max/step |
| `boolean` | Toggle/checkbox |
| `select` | Dropdown single selection |
| `radio` | Radio button group |
| `array` | List of primitive items (7 types: string, text, number, boolean, date, select, radio) |
| `objectArray` | Array of structured objects |

→ See [process-input-schema.md](./specific/process-input-schema.md) for complete field reference including `itemField` configuration

---

## 8. Sub-Workflows

Sub-workflows allow breaking complex workflows into reusable pieces.

### When to Use Sub-Workflows

- Reuse common logic across multiple workflows
- Break a large workflow into manageable pieces
- Create helper functions (e.g., file upload, data transformation)

### How Sub-Workflows Differ from Parent Workflows

| Aspect | Parent Workflow | Sub-Workflow |
|--------|-----------------|--------------|
| Codika Init | Required | Not present |
| Trigger type | HTTP, schedule, third-party | `subworkflow` |
| Visible in UI | Yes | No |
| Cost | Defined (e.g., `cost: 1`) | `cost: 0` |
| Execution context | Auto-available | Must be passed from parent |

### Execution Metadata in Sub-Workflows

Sub-workflows don't have Codika Init, so execution context (`executionId`, `executionSecret`) is not automatically available.

**When a sub-workflow uses Codika nodes** (like Upload File), the parent must pass execution metadata:

```json
// Parent workflow - Execute Workflow node
"workflowInputs": {
  "value": {
    "executionId": "={{ $('Codika Init').first().json.executionId }}",
    "executionSecret": "={{ $('Codika Init').first().json.executionSecret }}",
    // ... other inputs
  }
}
```

The sub-workflow then uses override parameters:

```json
// Sub-workflow - Codika Upload File node
"executionIdOverride": "={{ $('When Executed by Another Workflow').first().json.executionId }}",
"executionSecretOverride": "={{ $('When Executed by Another Workflow').first().json.executionSecret }}"
```

→ See [sub-workflows.md](./specific/sub-workflows.md) for complete sub-workflow guide

### CRITICAL: Error Handling in Tool Workflows

When sub-workflows are used as **AI agent tools** (via `toolWorkflow`), they MUST return errors explicitly. **Silent failures are unacceptable** - if validation fails and no error is returned, the AI agent assumes the operation succeeded.

**Every IF node checking for errors must have BOTH branches connected:**

- TRUE branch → Continue to next step
- FALSE branch → Return error node with `{ success: false, error: "message" }`

→ See [sub-workflows.md#10-error-handling-in-tool-workflows](./specific/sub-workflows.md#10-error-handling-in-tool-workflows) for implementation patterns

---

## 9. Common Features

### File Uploads

When workflows generate files (videos, PDFs, images), use Codika Upload File to store them.

**Use cases:**

- AI-generated videos (Fal.ai, Runway)
- PDF generation (reports, proposals)
- Image generation

**Pattern:**

1. Generate/download file (binary data)
2. Upload via Codika Upload File node
3. Reference `documentId` in result data
4. Define `type: "file"` in output schema

→ See [FILE_UPLOAD_GUIDE.md](../.claude/docs/features/file_upload/FILE_UPLOAD_GUIDE.md)

### RAG Integration

RAG (Retrieval-Augmented Generation) enables workflows to retrieve context from a knowledge base.

**Components:**

- Data ingestion workflow (embeds documents into Pinecone)
- RAG-enabled workflow (retrieves relevant context)

**Two levels of data ingestion:**

- **Process-level** (shared "Global Files"): Deployed separately, shared across all instances
- **Instance-level** (per-user "My Files"): Deployed with the process, per-user isolation

**When to use:**

- Process needs to reference uploaded documents
- Answers should be grounded in specific knowledge

→ See [data-ingestion.md](./specific/data-ingestion.md) for data ingestion setup (process-level and instance-level)
→ See [RAG_INTEGRATION_GUIDE.md](../.claude/docs/features/rag_integration/RAG_INTEGRATION_GUIDE.md) for n8n workflow implementation details

### Google Sheets as Database

Use Google Sheets for lightweight data storage within workflows.

**Use cases:**

- Store workflow state between executions
- Simple data tables the user can edit
- Avoid database setup for simple needs

→ See [GOOGLE_SHEETS_DATABASE_GUIDE.md](../.claude/docs/features/google_sheet_as_database/GOOGLE_SHEETS_DATABASE_GUIDE.md)

---

## 10. Creating a New Use Case

### Step-by-Step Process

1. **Create folder structure**

   ```
   use-cases/your-use-case/
   ├── config.ts
   ├── project.json
   ├── workflows/
   │   └── your-workflow.json
   └── data-ingestion/          # Optional — for RAG processes
       └── embedding.json
   ```

2. **Create project.json**
   - Create it manually: `{"projectId": "your-project-id"}`
   - Or use the CLI: `codika project create --name "Your Use Case" --path .`

3. **Define config.ts**
   - Define `getConfiguration()` with workflows, triggers, schemas
   - Use appropriate placeholders

4. **Create workflow JSON**
   - Start with trigger node
   - Add Codika Init immediately after trigger
   - Implement business logic
   - End all paths with Codika Submit Result or Report Error

5. **Define triggers and schemas**
   - Input schema: form fields for user input
   - Output schema: result structure
   - Choose appropriate trigger type

### The codika Package

All types and utilities for use case configuration come from the `codika` npm package:

```typescript
import {
  loadAndEncodeWorkflow,
  type FormInputSchema,
  type FormOutputSchema,
  type HttpTrigger,
  type ScheduleTrigger,
  type ProcessDeploymentConfigurationInput,
} from 'codika';
```

---

## 11. AI Nodes (LLM Chain, Agent, Tools)

When using LangChain nodes for LLM tasks, choosing the correct node type and configuration is critical.

### chainLlm vs agent

| Use Case | Node | Why |
|----------|------|-----|
| Document classification | `chainLlm` | Direct structured output |
| Metadata extraction | `chainLlm` | No reasoning needed |
| Text categorization | `chainLlm` | Deterministic |
| Multi-step reasoning | `agent` | Needs tool use |
| Tool calling workflows | `agent` | External tool access |

**Rule:** If you need JSON output, use `chainLlm`. If you need tools or complex reasoning, use `agent`.

### AI Agent Tools (Sub-Workflows as Tools)

When using sub-workflows as tools in AI agents, use the `workflowInputs` format with `$fromAI()`:

```json
{
  "parameters": {
    "name": "create_event",
    "description": "Create a new event",
    "workflowId": { "__rl": true, "value": "{{SUBWKFL_tool-create-event_LFKWBUS}}", "mode": "id" },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "title": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('title', ``, 'string') }}",
        "caller_phone": "={{ $('Parse Input').first().json.senderPhone }}"
      },
      "schema": [
        { "id": "title", "displayName": "title", "type": "string", ... },
        { "id": "caller_phone", "displayName": "caller_phone", "type": "string", ... }
      ]
    }
  },
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2.1
}
```

**Key points:**

- Use `workflowInputs.value` (NOT the old `fields.values` format)
- AI-filled params use `$fromAI('key', ``, 'type')` with the comment marker
- Trusted context params (like user identity) use direct expressions
- **Tool workflows MUST return errors explicitly** - silent failures cause agents to report incorrect success

→ See [ai-nodes.md](./specific/ai-nodes.md) for complete examples, troubleshooting, error handling, and common mistakes

---

## 12. Final Verification Checklist

Before committing workflow JSON files, verify they are properly sanitized.

### Required Properties

Every workflow JSON must have these properties:

```json
{
  "name": "Your Workflow Name",
  "nodes": [...],
  "connections": {...},
  "settings": {...}
}
```

### Properties to Remove

Remove these n8n-generated properties before committing:

| Property | Why Remove |
|----------|------------|
| `id` | n8n internal ID (changes on import) |
| `versionId` | Version tracking ID |
| `meta` | Metadata (templateCredsSetupCompleted, instanceId) |
| `active` | Workflow active state |
| `tags` | n8n tag associations |
| `pinData` | Pinned execution data for testing |

### Required Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

**`executionOrder: "v1"`** - Ensures consistent node execution order.

**`errorWorkflow`** - Routes unhandled errors to the organization's error workflow.

### Quick Sanitization Check

```bash
# Check for properties that should be removed
grep -E '"(id|versionId|meta|active|tags|pinData)"' workflows/*.json
```

If any results appear, remove those properties from the workflow JSON.

### Retry Safety Check

**CRITICAL:** Before committing, verify that `retryOnFail` is NOT set on any HTTP Request node that calls an async/callback API. Retries on async APIs spawn duplicate jobs, wasting resources and money.

```bash
# Find all HTTP Request nodes with retryOnFail enabled
grep -B5 '"retryOnFail": true' workflows/*.json | grep -i "http\|api\|call"
```

**Safe to retry:** Embedding APIs, file downloads, status polling, LLM chain nodes (idempotent operations).

**Never retry:** API calls that trigger background jobs, create resources, or use a callback/webhook pattern (fire-and-wait). Use `"onError": "continueErrorOutput"` instead.

→ See [http-triggers.md Section 10](./specific/http-triggers.md) for the complete fire-and-wait pattern guide.

---

## 13. Agent Skills (Making Workflows Agent-Accessible)

Skills are Claude-compatible documentation files that describe how to interact with your use case's workflows. They make your HTTP endpoints and scheduled workflows **discoverable and usable by AI agents**.

When you deploy a use case with skills, agents can run `codika get skills` to download them and immediately understand how to trigger your workflows — input schemas, output schemas, example payloads, and all.

**Key points:**
- One skill per triggerable workflow (HTTP or scheduled with manual trigger)
- Each skill is a directory with a `SKILL.md` file following the [Claude Agent Skills format](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- Skills are automatically collected during `codika deploy` and stored with the deployment
- Downloaded skills can be placed in `.claude/skills/` for Claude Code auto-discovery or uploaded to the Claude API

> See [agent-skills.md](./specific/agent-skills.md) for the complete guide: directory structure, frontmatter spec, writing best practices, validation rules, CLI commands, and real-world examples.

---

## 14. Integration Guides

Codika provides integration guides for external services. Consult these guides when building use cases that connect to these services.

### AI Providers (FLEXCRED)

These integrations use Flex Credentials with automatic fallback to Codika-managed API keys.

| Integration | Guide | Description |
|-------------|-------|-------------|
| Anthropic | [anthropic.md](./integrations/anthropic.md) | Claude AI models |
| OpenAI | [openai.md](./integrations/openai.md) | GPT models and embeddings |
| Tavily | [tavily.md](./integrations/tavily.md) | AI-optimized web search |
| xAI | [xai.md](./integrations/xai.md) | Grok models |
| OpenRouter | [openrouter.md](./integrations/openrouter.md) | Multi-provider AI gateway |
| Mistral | [mistral.md](./integrations/mistral.md) | Mistral AI models |
| Cohere | [cohere.md](./integrations/cohere.md) | Enterprise AI and embeddings |
| DeepSeek | [deepseek.md](./integrations/deepseek.md) | Cost-effective AI models |
| fal AI | [fal-ai.md](./integrations/fal-ai.md) | Image and video generation |

### Organization-Specific Integrations (ORGCRED)

These integrations use organization-level credentials.

| Integration | Guide | Description |
|-------------|-------|-------------|
| WhatsApp | [whatsapp.md](./integrations/whatsapp.md) | WhatsApp Business messaging |
| Slack | [slack.md](./integrations/slack.md) | Slack workspace integration |
| Twilio | [twilio.md](./integrations/twilio.md) | SMS and WhatsApp via Twilio |
| Folk CRM | [folk.md](./integrations/folk.md) | CRM contact management |
| Pipedrive | [pipedrive.md](./integrations/pipedrive.md) | Sales CRM and pipeline management |

### User-Specific Integrations (USERCRED)

These integrations use member-level OAuth credentials.

| Integration | Guide | Description |
|-------------|-------|-------------|
| Google | [google.md](./integrations/google.md) | Gmail, Sheets, Drive, Calendar |
| Microsoft | [microsoft.md](./integrations/microsoft.md) | Outlook, Teams, Excel, OneDrive |
| Calendly | [calendly.md](./integrations/calendly.md) | Scheduling automation |
| Notion | [notion.md](./integrations/notion.md) | Databases, pages, and blocks |

### Per-Installation Integrations (INSTCRED)

These integrations use credentials configured during process installation.

| Integration | Guide | Description |
|-------------|-------|-------------|
| Supabase | [supabase.md](./integrations/supabase.md) | Supabase database |
| PostgreSQL | [postgres.md](./integrations/postgres.md) | PostgreSQL database |

---

## 15. Custom Integrations

Custom integrations let use cases define their own API connections dynamically in `config.ts`, without requiring any hardcoded integration files in the platform codebase. They are ideal for simple API-key or token-based services that are specific to a single use case.

### Key Concepts

- Custom integration IDs **must** start with `cstm_` to prevent collisions with built-in integrations
- IDs must be `snake_case` (e.g., `cstm_acme_crm`, `cstm_weather_api`)
- They support all three context types: `organization`, `member`, and `process_instance`
- The platform auto-generates the dashboard UI (credential form, status display) from the schema
- Custom integration IDs are automatically merged into `integrationUids` during deployment

### Defining in config.ts

Add a `customIntegrations` array to your `getConfiguration()` return value:

```typescript
import type {
  ProcessDeploymentConfigurationInput,
  CustomIntegrationSchema,
} from 'codika';

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'Acme CRM Sync',
    subtitle: 'Sync contacts with Acme CRM',
    description: 'Automatically syncs contacts from Acme CRM using their REST API.',
    customIntegrations: [
      {
        id: 'cstm_acme_crm',
        name: 'Acme CRM',
        description: 'Connect your Acme CRM API key',
        contextType: 'organization',
        n8nCredentialType: 'httpHeaderAuth',
        n8nCredentialMapping: {
          API_KEY: 'value',
        },
        secretFields: [
          {
            key: 'API_KEY',
            label: 'API Key',
            type: 'password',
            description: 'Your Acme CRM API key',
            placeholder: 'acme_key_...',
            required: true,
          },
        ],
        metadataFields: [
          {
            key: 'BASE_URL',
            label: 'Base URL',
            type: 'url',
            description: 'Acme CRM API base URL',
            placeholder: 'https://api.acme.com/v2',
          },
        ],
        icon: 'Building2',
        color: '#4F46E5',
      },
    ],
    workflows: [/* ... */],
    integrationUids: ['cstm_acme_crm', 'anthropic'],
  };
}
```

### Using in Workflow JSON

Custom integrations use the same credential placeholder patterns as built-in integrations. The placeholder type depends on `contextType`:

| contextType | Placeholder Pattern | Example |
|-------------|--------------------|---------|
| `process_instance` | `INSTCRED` | `{{INSTCRED_CSTM_ACME_CRM_ID_DERCTSNI}}` |
| `organization` | `ORGCRED` | `{{ORGCRED_CSTM_ACME_CRM_ID_DERCGRO}}` |
| `member` | `USERCRED` | `{{USERCRED_CSTM_ACME_CRM_ID_DERCRESU}}` |

**Workflow JSON example:**

```json
"credentials": {
  "httpHeaderAuth": {
    "id": "{{ORGCRED_CSTM_ACME_CRM_ID_DERCGRO}}",
    "name": "{{ORGCRED_CSTM_ACME_CRM_NAME_DERCGRO}}"
  }
}
```

### Supported n8n Credential Types

| Type | Use Case |
|------|----------|
| `httpHeaderAuth` | API key in HTTP header (most common) |
| `httpBasicAuth` | Username + password |
| `httpQueryAuth` | API key as query parameter |
| `none` | No n8n credential (metadata only) |

> See [placeholder-patterns.md](./specific/placeholder-patterns.md) for complete placeholder reference including custom integration examples.

---

## 16. Guide Index

### Specific Guides (Detailed Implementation)

| Guide | Description |
|-------|-------------|
| [http-triggers.md](./specific/http-triggers.md) | HTTP trigger configuration and form schemas |
| [schedule-triggers.md](./specific/schedule-triggers.md) | Schedule (cron) trigger setup |
| [third-party-triggers.md](./specific/third-party-triggers.md) | Third-party trigger setup (Gmail, Calendly, etc.) |
| [sub-workflows.md](./specific/sub-workflows.md) | Sub-workflow creation and execution metadata |
| [placeholder-patterns.md](./specific/placeholder-patterns.md) | Complete placeholder pattern reference |
| [codika-nodes.md](./specific/codika-nodes.md) | Codika node parameters and configuration |
| [config-patterns.md](./specific/config-patterns.md) | config.ts field reference |
| [process-input-schema.md](./specific/process-input-schema.md) | Deployment parameters and INSTPARM placeholders |
| [ai-nodes.md](./specific/ai-nodes.md) | LLM chains, agents, and AI agent tools configuration |
| [data-ingestion.md](./specific/data-ingestion.md) | Data ingestion (process-level and instance-level) |
| [agent-skills.md](./specific/agent-skills.md) | Agent skills for making workflows accessible to AI agents |

### Feature Guides

| Guide | Description |
|-------|-------------|
| [FILE_UPLOAD_GUIDE.md](../.claude/docs/features/file_upload/FILE_UPLOAD_GUIDE.md) | File upload implementation |
| [RAG_INTEGRATION_GUIDE.md](../.claude/docs/features/rag_integration/RAG_INTEGRATION_GUIDE.md) | RAG setup and data ingestion |
| [GOOGLE_SHEETS_DATABASE_GUIDE.md](../.claude/docs/features/google_sheet_as_database/GOOGLE_SHEETS_DATABASE_GUIDE.md) | Google Sheets as data storage |
| [PDF_GENERATION_GUIDE.md](../.claude/docs/features/pdf_generation/PDF_GENERATION_GUIDE.md) | PDF generation patterns |

### Reference

| Guide | Description |
|-------|-------------|
| [common-errors.md](post-creation/common-errors.md) | Troubleshooting common issues |
| Node Positioning | Use the `node-positioner` skill (`.claude/skills/node-positioner/`) |
