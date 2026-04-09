# Config.ts Patterns Guide

The `config.ts` file defines your process configuration including display metadata, workflows, triggers, and schemas. This guide provides the complete reference for structuring this file.

---

## 1. File Structure Overview

The use case folder contains `config.ts` alongside a `project.json` file:

```
use-cases/your-use-case/
├── config.ts
├── project.json          ← Target project ID
└── workflows/
    └── your-workflow.json
```

```typescript
// config.ts

// Imports
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
import { loadAndEncodeWorkflow, type ... } from 'codika';

// Path setup
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Configuration constants
export const WORKFLOW_FILES = [...];

// Schema functions
function getInputSchema(): FormInputSchema { ... }
function getOutputSchema(): FormOutputSchema { ... }

// Main configuration
export function getConfiguration(): ProcessDeploymentConfigurationInput { ... }

// Optional: Data ingestion for RAG
export function getDataIngestionConfig(): ProcessDataIngestionConfigInput { ... }
```

---

## 2. Required Imports

```typescript
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
import {
  loadAndEncodeWorkflow,
  type FormInputSchema,
  type FormOutputSchema,
  type HttpTrigger,
  type ScheduleTrigger,
  type ServiceEventTrigger,
  type SubworkflowTrigger,
  type ProcessDataIngestionConfigInput,
  type ProcessDeploymentConfigurationInput,
} from 'codika';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## 3. Configuration Constants

### project.json (Project ID)

The target Firebase project ID is stored in a separate `project.json` file alongside `config.ts` (not exported from config.ts):

```json
{
  "projectId": "your-project-id-here"
}
```

Create it manually or use the CLI:
```bash
codika project create --name "Your Use Case" --path .
```

> **Note:** For backward compatibility, the deployer still falls back to `PROJECT_ID` in config.ts if no `project.json` exists.

### WORKFLOW_FILES

```typescript
export const WORKFLOW_FILES = [
  join(__dirname, 'workflows/main-workflow.json'),
  join(__dirname, 'workflows/helper-workflow.json'),
];
```

List of all workflow JSON files to be archived during deployment.

### Data Ingestion Folder

For processes with RAG, the data ingestion workflow lives in a dedicated `data-ingestion/` folder (not in `workflows/`). The CLI auto-discovers the single `.json` file in this folder during `deploy process-data-ingestion`.

```
my-use-case/
  data-ingestion/
    my-embedding-ingestion.json    # Exactly one JSON file
```

---

## 4. Display Metadata

Every process must provide three mandatory display fields in `getConfiguration()`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `string` | Yes | Display title in process store (2-5 words) |
| `subtitle` | `string` | Yes | Short tagline describing the process |
| `description` | `string` | Yes | 1-2 sentences explaining capabilities |
| `releaseNotes` | `string` | No | Version-specific update notes |
| `icon` | `string` | No | Lucide icon name (e.g., `"Mail"`, `"FileText"`) |

### Guidelines

- **title**: Concise, focused on main value proposition
- **subtitle**: One short sentence describing key benefit
- **description**: What the process does and how it helps users

### Example

```typescript
return {
  title: 'Customer Feedback Intelligence',
  subtitle: 'Analyze feedback using AI-powered insights',
  description: 'A RAG-based system for analyzing customer feedback and generating actionable insights.',
  releaseNotes: 'Added support for multi-language feedback analysis',
  icon: 'MessageSquare',
  // ... rest of configuration
};
```

---

## 5. getConfiguration() Function

The main configuration function returns `ProcessDeploymentConfigurationInput`:

```typescript
export function getConfiguration(): ProcessDeploymentConfigurationInput {
  const webhookId = crypto.randomUUID();

  // URL pattern with placeholders
  const webhookUrl = `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-endpoint`;

  // Load and encode workflow
  const workflowBase64 = loadAndEncodeWorkflow(
    join(__dirname, 'workflows/main-workflow.json')
  );

  const workflows = [
    {
      workflowTemplateId: 'main-workflow',
      workflowId: 'main-workflow',
      workflowName: 'Main Workflow',
      markdownInfo: getWorkflowMarkdown(),
      knowledgeBaseAccess: {
        processDocTags: [],
        processInstanceDocTags: [],
      },
      integrationUids: ['openai', 'anthropic'],
      triggers: [
        {
          triggerId: webhookId,
          type: 'http' as const,
          url: webhookUrl,
          method: 'POST' as const,
          title: 'Run Workflow',
          description: 'Execute the main workflow',
          inputSchema: getInputSchema(),
        } satisfies HttpTrigger,
      ],
      outputSchema: getOutputSchema(),
      n8nWorkflowJsonBase64: workflowBase64,
    },
  ];

  return {
    title: 'Process Title',
    subtitle: 'Short description',
    description: 'Full description of the process.',
    workflows,
    processDeploymentMarkdown: getProcessMarkdown(),
    tags: ['tag1', 'tag2'],
    integrationUids: ['openai', 'anthropic'],
  };
}
```

---

## 6. Workflow Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflowTemplateId` | `string` | Yes | Unique identifier for the workflow template |
| `workflowId` | `string` | Yes | Workflow ID (used in Codika Init) |
| `workflowName` | `string` | Yes | Display name |
| `markdownInfo` | `string` | No | Workflow documentation |
| `knowledgeBaseAccess` | `object` | No | RAG document tag access |
| `integrationUids` | `string[]` | Yes | Required integrations |
| `triggers` | `Trigger[]` | Yes | Workflow triggers |
| `outputSchema` | `FormOutputSchema` | Yes | Output field definitions |
| `n8nWorkflowJsonBase64` | `string` | Yes | Base64-encoded workflow JSON |

### Integration UIDs

Common integration UIDs:

| Integration | UID |
|-------------|-----|
| Anthropic | `anthropic` |
| OpenAI | `openai` |
| Google Sheets | `google_sheets` |
| Google Drive | `google_drive` |
| Gmail | `google_gmail` |
| Slack | `slack` |
| Pipedrive | `pipedrive` |
| Folk CRM | `folk` |

---

## 7. Trigger Configuration

### HTTP Trigger

```typescript
{
  triggerId: crypto.randomUUID(),
  type: 'http' as const,
  url: `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/endpoint`,
  method: 'POST' as const,
  title: 'Trigger Title',
  description: 'What this trigger does',
  inputSchema: getInputSchema(),
} satisfies HttpTrigger
```

### Schedule Trigger

```typescript
{
  triggerId: crypto.randomUUID(),
  type: 'schedule' as const,
  title: 'Scheduled Task',
  description: 'Runs on a schedule',
} satisfies ScheduleTrigger
```

> **Note:** Schedule triggers do not have `inputSchema` since users don't submit data.

### Service Event Trigger

For workflows triggered by external services (Slack, Gmail, WhatsApp, Pipedrive, etc.):

```typescript
{
  triggerId: crypto.randomUUID(),
  type: 'service_event' as const,
  service: 'slack' as const,
  eventType: 'message',
  title: 'New Slack Message',
  description: 'Triggered when a message is posted in the monitored channel',
} satisfies ServiceEventTrigger
```

**Valid `service` values:** `'telegram' | 'email' | 'slack' | 'pipedrive' | 'discord' | 'other'`

> **Note:** Service event triggers do not have `inputSchema` since input comes from the external service. See [third-party-triggers.md](./third-party-triggers.md) for workflow-level patterns (Codika Init configuration, data access, webhookId requirements).

### Subworkflow Trigger

For workflows called by other workflows:

```typescript
{
  triggerId: crypto.randomUUID(),
  type: 'subworkflow' as const,
  title: 'Called by Parent Workflow',
  description: 'Invoked as a sub-workflow',
  inputSchema: [{ key: 'text', type: 'string' }],
  calledBy: ['parent-workflow-id'],
} satisfies SubworkflowTrigger
```

**Optional fields:**
- `inputSchema` — Array of `{ key: string, type: 'string' | 'number' | 'boolean' | 'array' | 'object' }` matching the n8n `executeWorkflowTrigger` inputs
- `calledBy` — Array of `workflowTemplateId` values for parent workflows

> **Note:** Sub-workflows do not include Codika Init/Submit/Report nodes. They start with `Execute Workflow Trigger`.

---

## 8. Input/Output Schema

> **See [http-triggers.md](./http-triggers.md)** for complete schema documentation including:
> - All field types (`string`, `text`, `number`, `boolean`, `date`, `select`, `multiselect`, `radio`, `file`, `array`, `object`, `objectArray`)
> - Field configuration options
> - Validation rules
> - Complete examples

### Output Schema Array Fields

When an output field has `type: 'array'`, it **must** include an `itemField` property specifying the type of each item in the array:

```typescript
{
  key: 'action_items',
  type: 'array',
  label: 'Action Items',
  description: 'List of action items extracted from the meeting',
  itemField: {
    type: 'string',
    label: 'Action Item',
  },
}
```

**Valid `itemField.type` values:** `'string' | 'text' | 'number' | 'boolean' | 'date' | 'select' | 'radio'`

For structured data (arrays of objects), use `string` type and return JSON-serialized data:

```typescript
{
  key: 'contacts',
  type: 'array',
  label: 'Contacts Found',
  description: 'List of contact records',
  itemField: {
    type: 'string',
    label: 'Contact (JSON)',
  },
}
```

### Quick Reference

FormInputSchema is an **array** of sections/fields. All field properties are defined at the top level of each field object.

```typescript
function getInputSchema(): FormInputSchema {
  return [
    {
      type: 'section',
      title: 'Input',
      description: 'Provide your input',
      collapsible: false,
      inputSchema: [
        {
          key: 'user_input',
          type: 'text',
          label: 'Your Input',
          description: 'Enter your text',
          placeholder: 'Type here...',
          required: true,
          maxLength: 5000,
          rows: 6,
        },
        {
          key: 'priority',
          type: 'select',
          label: 'Priority',
          required: true,
          options: [
            { value: 'low', label: 'Low' },
            { value: 'medium', label: 'Medium' },
            { value: 'high', label: 'High' },
          ],
        },
      ],
    },
  ];
}

function getOutputSchema(): FormOutputSchema {
  return [
    {
      key: 'result',
      type: 'text',
      label: 'Result',
      description: 'The processed output',
    },
    {
      key: 'status',
      type: 'string',
      label: 'Status',
    },
  ];
}
```

---

## 9. Data Ingestion Configuration (RAG)

For processes with RAG (Retrieval-Augmented Generation):

```typescript
export function getDataIngestionConfig(): ProcessDataIngestionConfigInput {
  const ingestionWorkflowBase64 = loadAndEncodeWorkflow(
    join(__dirname, 'data-ingestion/data-ingestion.json')
  );

  return {
    workflowTemplateId: 'data-ingestion',
    workflowName: 'Data Ingestion',
    n8nWorkflowJsonBase64: ingestionWorkflowBase64,
    webhooks: {
      embed: `{{PROCDATA_PROCESS_ID_ATADCORP}}/embed`,
      delete: `{{PROCDATA_PROCESS_ID_ATADCORP}}/embed-delete`,
    },
    purpose: 'Embed documents into Pinecone for RAG retrieval',
    markdownInfo: getIngestionWorkflowMarkdown(),
  };
}
```

### Data Ingestion Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflowTemplateId` | `string` | Yes | Unique template ID |
| `workflowName` | `string` | Yes | Display name |
| `n8nWorkflowJsonBase64` | `string` | Yes | Base64-encoded workflow |
| `webhooks.embed` | `string` | Yes | Embed webhook path |
| `webhooks.delete` | `string` | Yes | Delete webhook path |
| `purpose` | `string` | No | Description of ingestion purpose |
| `markdownInfo` | `string` | No | Workflow documentation |

---

## 10. Markdown Documentation Functions

Each workflow must have a `markdownInfo` field populated by a dedicated function, and the process must have a `processDeploymentMarkdown` field. These provide user-facing documentation displayed in the platform.

### Naming Convention

| Scope | Function name | Config field |
|-------|--------------|--------------|
| Per workflow | `get<WorkflowName>WorkflowMarkdown()` | `markdownInfo` |
| Per sub-workflow | `get<SubworkflowName>WorkflowMarkdown()` | `markdownInfo` |
| Process-level | `getProcessMarkdown()` | `processDeploymentMarkdown` |

### Workflow Markdown Structure

```typescript
function getMyFeatureWorkflowMarkdown(): string {
  return `# Workflow Display Name

## What It Does
One or two sentences describing the workflow's purpose.

## How It Works

1. **Step One** - What happens first
2. **Step Two** - What happens next
3. **Step Three** - Final result`;
}
```

Add optional sections only when relevant: `## Configuration` (for workflows with deployment parameters), `## Example Interactions` (for AI agent workflows), `## Supported Formats` (for file-processing workflows).

### Sub-Workflow Markdown Structure

Keep sub-workflow documentation minimal:

```typescript
function getMyHelperSubworkflowMarkdown(): string {
  return `# Sub-Workflow Name

## Purpose
One sentence describing what it does.

## Called By
- parent-workflow-id (context of usage)`;
}
```

### Process Markdown Structure

```typescript
function getProcessMarkdown(): string {
  return `# Use Case Title

Brief description of the overall system.

## Overview
- **Workflow A** - What it does
- **Workflow B** - What it does

## Quick Start
1. **Configure** - What to set up
2. **Use** - How to use it`;
}
```

The process markdown should give users a high-level understanding of the entire use case: what workflows are available, how they relate to each other, and how to get started.

---

## 11. Complete Example

```typescript
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
import {
  loadAndEncodeWorkflow,
  type FormInputSchema,
  type FormOutputSchema,
  type HttpTrigger,
  type ProcessDeploymentConfigurationInput,
} from 'codika';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Project ID is in project.json: {"projectId": "production-project"}

export const WORKFLOW_FILES = [
  join(__dirname, 'workflows/main-workflow.json'),
];

// Input schema definition
function getInputSchema(): FormInputSchema {
  return [
    {
      type: 'section',
      title: 'Your Question',
      description: 'Ask anything and get AI-powered answers',
      collapsible: false,
      inputSchema: [
        {
          key: 'query',
          type: 'text',
          label: 'Your Question',
          description: 'Enter your question',
          placeholder: 'What would you like to know?',
          required: true,
          maxLength: 5000,
          rows: 4,
        },
        {
          key: 'response_format',
          type: 'select',
          label: 'Response Format',
          description: 'How should the answer be formatted?',
          required: false,
          defaultValue: 'detailed',
          options: [
            { value: 'brief', label: 'Brief' },
            { value: 'detailed', label: 'Detailed' },
            { value: 'bullet_points', label: 'Bullet Points' },
          ],
        },
      ],
    },
  ];
}

// Output schema definition
function getOutputSchema(): FormOutputSchema {
  return [
    {
      key: 'answer',
      type: 'text',
      label: 'Answer',
      description: 'The AI-generated response',
    },
    {
      key: 'confidence',
      type: 'number',
      label: 'Confidence',
      description: 'Confidence score (0-1)',
    },
    {
      key: 'sources_used',
      type: 'boolean',
      label: 'Sources Used',
      description: 'Whether external sources were consulted',
    },
  ];
}

// Markdown documentation
function getAiAssistantWorkflowMarkdown(): string {
  return `# AI Assistant

## What It Does
Answers user questions using advanced language models.

## How It Works

1. **Submit Question** - User enters a question and selects response format
2. **AI Processing** - Claude analyzes the question and generates a response
3. **Return Result** - Answer returned with confidence score`;
}

function getProcessMarkdown(): string {
  return `# AI Assistant

An AI-powered assistant that answers questions using advanced language models.

## Overview
- **Ask Question** - Submit any question and get an AI-generated answer

## Quick Start
1. **Install** the process
2. **Ask a question** via the trigger form`;
}

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  const webhookId = crypto.randomUUID();
  const webhookUrl = `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/ask`;

  const workflowBase64 = loadAndEncodeWorkflow(
    join(__dirname, 'workflows/main-workflow.json')
  );

  return {
    title: 'AI Assistant',
    subtitle: 'Get answers to your questions',
    description: 'An AI-powered assistant that answers your questions using advanced language models.',

    workflows: [
      {
        workflowTemplateId: 'ai-assistant',
        workflowId: 'ai-assistant',
        workflowName: 'AI Assistant',
        markdownInfo: getAiAssistantWorkflowMarkdown(),
        integrationUids: ['anthropic'],
        triggers: [
          {
            triggerId: webhookId,
            type: 'http' as const,
            url: webhookUrl,
            method: 'POST' as const,
            title: 'Ask Question',
            description: 'Submit a question to the AI assistant',
            inputSchema: getInputSchema(),
          } satisfies HttpTrigger,
        ],
        outputSchema: getOutputSchema(),
        n8nWorkflowJsonBase64: workflowBase64,
      },
    ],

    processDeploymentMarkdown: getProcessMarkdown(),
    integrationUids: ['anthropic'],
  };
}
```

---

## 12. Checklist

**Required Files and Exports:**
- [ ] `project.json` - Contains target project ID (`{"projectId": "..."}`)
- [ ] `WORKFLOW_FILES` - Array of workflow file paths (in config.ts)
- [ ] `getConfiguration()` - Main configuration function (in config.ts)

**Display Metadata:**
- [ ] `title` - Process title (2-5 words)
- [ ] `subtitle` - Short tagline
- [ ] `description` - Full description

**Workflow Configuration:**
- [ ] `workflowTemplateId` - Unique identifier
- [ ] `workflowId` - Matches Codika Init workflowId
- [ ] `markdownInfo` - Workflow documentation via `get<Name>WorkflowMarkdown()`
- [ ] `triggers` - At least one trigger
- [ ] `outputSchema` - Output field definitions
- [ ] `n8nWorkflowJsonBase64` - Encoded workflow

**Markdown Documentation:**
- [ ] `get<WorkflowName>WorkflowMarkdown()` for each workflow
- [ ] `getProcessMarkdown()` for process-level `processDeploymentMarkdown`

**For RAG Processes:**
- [ ] `data-ingestion/` folder with exactly one workflow JSON file
- [ ] `getDataIngestionConfig()` function in `config.ts`

---

## 13. Custom Integrations

The `customIntegrations` field in `ProcessDeploymentConfigurationInput` lets use cases define their own API connections without requiring hardcoded integration files in the platform.

### CustomIntegrationSchema Interface

```typescript
interface CustomIntegrationSchema {
  /** Unique ID, snake_case. Must start with 'cstm_' (e.g., 'cstm_acme_crm') */
  id: string;
  /** Display name shown in dashboard (e.g., 'Acme CRM') */
  name: string;
  /** Help text for users */
  description?: string;
  /** Where this integration is stored */
  contextType: 'organization' | 'member' | 'process_instance';
  /**
   * n8n credential type to create.
   * Any valid n8n credential type (e.g. 'httpHeaderAuth', 'twilioApi', 'openAiApi')
   * or 'none' for no n8n credential.
   * Use `codika integration schema <type>` to discover available types and their fields.
   */
  n8nCredentialType: string;
  /** Maps secret/metadata field keys to n8n credential data field names */
  n8nCredentialMapping?: Record<string, string>;
  /** Secret fields (encrypted, stored securely) */
  secretFields: CustomIntegrationFieldSchema[];
  /** Non-secret metadata fields (optional, for display/config) */
  metadataFields?: CustomIntegrationFieldSchema[];
  /** Lucide icon name for dashboard display (e.g., 'Building2') */
  icon?: string;
  /** URL to a custom logo image */
  logoUrl?: string;
  /** Primary color (hex) for dashboard display */
  color?: string;
}

interface CustomIntegrationFieldSchema {
  /** Field key, UPPER_SNAKE_CASE (e.g., 'API_KEY', 'BASE_URL') */
  key: string;
  /** Human-readable label */
  label: string;
  /** Field input type */
  type: 'password' | 'string' | 'url' | 'number';
  /** Help text for the field */
  description?: string;
  /** Placeholder text for the input */
  placeholder?: string;
  /** Whether the field is required (default: true for secrets, false for metadata) */
  required?: boolean;
}
```

### Complete config.ts Example with Custom Integrations

```typescript
import {
  loadAndEncodeWorkflow,
  type ProcessDeploymentConfigurationInput,
  type HttpTrigger,
  type FormInputSchema,
  type FormOutputSchema,
} from 'codika';

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  const webhookUrl = `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/sync`;

  return {
    title: 'CRM Contact Sync',
    subtitle: 'Sync contacts with your custom CRM',
    description: 'Automatically pulls contacts from your CRM API.',

    customIntegrations: [
      {
        id: 'cstm_my_crm',
        name: 'My CRM',
        description: 'Connect your CRM API credentials',
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
            description: 'Your CRM API key',
            required: true,
          },
        ],
        metadataFields: [
          {
            key: 'BASE_URL',
            label: 'API Base URL',
            type: 'url',
            placeholder: 'https://api.mycrm.com/v1',
          },
        ],
        icon: 'Users',
        color: '#6366F1',
      },
    ],

    workflows: [
      {
        workflowTemplateId: 'sync-contacts',
        workflowId: 'sync-contacts',
        workflowName: 'Sync Contacts',
        integrationUids: ['cstm_my_crm', 'anthropic'],
        triggers: [
          {
            triggerId: crypto.randomUUID(),
            type: 'http' as const,
            url: webhookUrl,
            method: 'POST' as const,
            title: 'Sync Now',
            description: 'Pull latest contacts from CRM',
            inputSchema: [],
          } satisfies HttpTrigger,
        ],
        outputSchema: [],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./workflows/sync-contacts.json'),
      },
    ],

    integrationUids: ['cstm_my_crm', 'anthropic'],
  };
}
```

### Integration UIDs Note

Custom integration IDs listed in `customIntegrations` are automatically merged into `integrationUids` during deployment. You can also explicitly list them in `integrationUids` for clarity, but it is not required.

### Using Native n8n Credential Types

Custom integrations can use **any n8n credential type**, not just HTTP auth types. Use `codika integration schema <type>` to discover which fields a credential type requires, then map them with `n8nCredentialMapping`:

```bash
# Discover the fields for twilioApi
codika integration schema twilioApi
```

Example with a native n8n type:

```typescript
customIntegrations: [
  {
    id: 'cstm_my_twilio',
    name: 'My Twilio',
    contextType: 'organization',
    n8nCredentialType: 'twilioApi',  // Any valid n8n credential type
    n8nCredentialMapping: {
      ACCOUNT_SID: 'accountSid',
      AUTH_TOKEN: 'authToken',
      AUTH_TYPE: 'authType',
      ALLOWED_DOMAINS: 'allowedDomains',
    },
    secretFields: [
      { key: 'ACCOUNT_SID', label: 'Account SID', type: 'string', required: true },
      { key: 'AUTH_TOKEN', label: 'Auth Token', type: 'password', required: true },
    ],
    metadataFields: [
      { key: 'AUTH_TYPE', label: 'Auth Type', type: 'string', placeholder: 'authToken' },
      { key: 'ALLOWED_DOMAINS', label: 'Allowed Domains', type: 'string', placeholder: '' },
    ],
    icon: 'Phone',
    color: '#F22F46',
  },
],
```

---

## Related Documentation

- [integration-reference.md](./integration-reference.md) - Complete integration reference: all built-in integrations, credential types, placeholder patterns, CLI fields, and built-in vs custom comparison
- [http-triggers.md](./http-triggers.md) - Trigger and schema details
- [schedule-triggers.md](./schedule-triggers.md) - Schedule trigger config
- [third-party-triggers.md](./third-party-triggers.md) - Service event trigger workflows (Gmail, Slack, Calendly, etc.)
- [placeholder-patterns.md](./placeholder-patterns.md) - URL placeholders
- [process-input-schema.md](./process-input-schema.md) - Deployment parameters for process installation
