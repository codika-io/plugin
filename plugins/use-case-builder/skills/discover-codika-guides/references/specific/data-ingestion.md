# Data Ingestion Guide

This guide explains how data ingestion works in the Codika platform. Data ingestion is the pipeline that takes uploaded knowledge base (KB) documents, parses them, embeds them into a vector store (Pinecone), and makes them available for RAG (Retrieval-Augmented Generation) queries.

There are **two independent systems** for data ingestion, and they serve different purposes:

| System | Scope | KB Location | Namespace | Deployment |
|--------|-------|-------------|-----------|------------|
| **Process-level** | Shared across ALL instances | `processes/{processId}/knowledge_base/` | `process_{processId}` | `codika deploy process-data-ingestion` |
| **Instance-level** | Per-user, isolated | `process_instances/{instanceId}/knowledge_base/` | `{processInstanceId}` | Part of normal process deployment |

**Both systems coexist independently.** A single use case can have both process-level DI (for shared "Global Files") and instance-level DI (for per-user "My Files"), or just one, or neither.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Process-Level Data Ingestion](#2-process-level-data-ingestion)
3. [Instance-Level Data Ingestion](#3-instance-level-data-ingestion)
4. [N8n Workflow Structure](#4-n8n-workflow-structure)
5. [Codika Nodes for Data Ingestion](#5-codika-nodes-for-data-ingestion)
6. [The Callback System](#6-the-callback-system)
7. [Cost and Credit Deduction](#7-cost-and-credit-deduction)
8. [Archive and Unarchive Behavior](#8-archive-and-unarchive-behavior)
9. [Comparison Table](#9-comparison-table)
10. [Complete Examples](#10-complete-examples)

---

## 1. Architecture Overview

### How Data Ingestion Works (Both Levels)

The flow is the same for both levels, with different document paths and namespaces:

```
User uploads file to KB
        ↓
Platform parses file → extracts text (parsingData.status = 'completed')
        ↓
Firestore trigger detects parsing completion
        ↓
Trigger fetches webhook URL from parent document (denormalized)
        ↓
Trigger calls n8n data ingestion webhook with document content
        ↓
n8n workflow processes document:
  - Deduplication check (content hash)
  - Classification (LLM-based, optional)
  - Section extraction (optional)
  - Metadata extraction (optional)
  - Embed into Pinecone
        ↓
n8n workflow calls submitEmbeddingResult callback
        ↓
Platform updates document with embedding status
```

### Key Concepts

- **Webhook URLs**: The n8n data ingestion workflow exposes two webhooks: `/embed` (for new/updated documents) and `/embed-delete` (for removing embeddings when a document is archived).
- **Denormalization**: Webhook URLs are stored directly on the parent document (`Process` or `ProcessInstance`) for efficient Firestore trigger lookups.
- **Fire-and-forget**: The Firestore trigger calls the webhook without waiting for a response. The n8n workflow reports back via a callback URL.
- **Embedding secret**: A UUID generated per-embedding request, used to authenticate the callback. Prevents replay attacks.

---

## 2. Process-Level Data Ingestion

Process-level data ingestion is **transversal** — it is shared across ALL process instances. Documents in the process-level KB ("Global Files") are embedded into a single shared namespace (`process_{processId}`), so all users querying via RAG can access the same knowledge.

### 2.1 Deployment: Separate Lifecycle

Process-level DI has its **own deployment lifecycle**, completely independent from the process deployment. This means:

- Updating the DI workflow does NOT trigger "update available" notifications to users
- DI has its own versioning (major.minor, stored in `process_data_ingestions/{id}`)
- DI is deployed via `codika deploy process-data-ingestion <use-case-path>`, not via `codika deploy use-case`

**Why separate?** Data ingestion workflows evolve frequently (improving classification, adding metadata extraction). These changes are backend-only and shouldn't bother users with update prompts.

### 2.2 Configuration in config.ts

Process-level DI configuration uses a separate function `getDataIngestionConfig()`:

The process-level DI workflow lives in a dedicated `data-ingestion/` folder (not in `workflows/`). The CLI auto-discovers the single `.json` file in this folder during deployment.

```
my-use-case/
  config.ts                                    # Exports getDataIngestionConfig()
  data-ingestion/
    my-embedding-ingestion.json                # Exactly one workflow JSON file
  workflows/
    main-workflow.json                         # Regular workflows stay here
```

```typescript
import {
  loadAndEncodeWorkflow,
  type ProcessDataIngestionConfigInput,
} from 'codika';

/**
 * Get the data ingestion configuration for this use case
 *
 * Deploy with: codika deploy process-data-ingestion use-cases/your-use-case
 */
export function getDataIngestionConfig(): ProcessDataIngestionConfigInput {
  const ingestionWorkflowBase64 = loadAndEncodeWorkflow(
    join(__dirname, 'data-ingestion/my-embedding-ingestion.json')
  );

  return {
    workflowTemplateId: 'my-embedding-ingestion',
    workflowName: 'Embedding Ingestion',
    n8nWorkflowJsonBase64: ingestionWorkflowBase64,
    webhooks: {
      embed: '{{PROCDATA_PROCESS_ID_ATADCORP}}/embed',
      delete: '{{PROCDATA_PROCESS_ID_ATADCORP}}/embed-delete',
    },
    purpose: 'Embed KB documents into Pinecone for RAG retrieval',
    markdownInfo: getIngestionWorkflowMarkdown(),
    cost: 2, // 2 credits per successful embedding
  };
}
```

#### Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflowTemplateId` | `string` | Yes | Unique template ID for the DI workflow |
| `workflowName` | `string` | Yes | Display name |
| `n8nWorkflowJsonBase64` | `string` | Yes | Base64-encoded n8n workflow JSON |
| `webhooks.embed` | `string` | Yes | Webhook path for embedding (with placeholders) |
| `webhooks.delete` | `string` | Yes | Webhook path for deletion (with placeholders) |
| `purpose` | `string` | No | Description of what this DI does |
| `markdownInfo` | `string` | No | Markdown documentation |
| `cost` | `number` | No | Credit cost per successful embedding |

#### Webhook Path Placeholders

Process-level DI uses **process data placeholders** because the namespace is shared:

```
embed:  {{PROCDATA_PROCESS_ID_ATADCORP}}/embed
delete: {{PROCDATA_PROCESS_ID_ATADCORP}}/embed-delete
```

These get resolved to:
```
embed:  abc123processId/embed
delete: abc123processId/embed-delete
```

And the full webhook URL becomes:
```
https://n8n-base-url/webhook/abc123processId/embed
```

### 2.3 Deployment

Use cases with process-level DI are deployed via the `codika` CLI:

```bash
# Default (patch bump)
codika deploy process-data-ingestion use-cases/your-use-case

# Minor version bump
codika deploy process-data-ingestion use-cases/your-use-case --minor

# Major version bump
codika deploy process-data-ingestion use-cases/your-use-case --major

# Explicit version
codika deploy process-data-ingestion use-cases/your-use-case --version 2.0
```

The CLI reads `getDataIngestionConfig()` from `config.ts` and auto-discovers the workflow JSON file from the `data-ingestion/` folder (expects exactly one `.json` file).

### 2.4 Config Example

### 2.4 How Process-Level DI Gets Deployed (Backend Flow)

When `deployDataIngestion` is called:

1. The backend creates a `ProcessDataIngestion` record in `process_data_ingestions/{id}` with status `pending`
2. The workflow JSON is decoded from base64
3. Placeholders are replaced (`{{PROCDATA_PROCESS_ID_ATADCORP}}` → actual process ID, `{{FLEXCRED_*}}` → credentials, etc.)
4. A **new** n8n workflow is created (never updates in place — better audit trail)
5. Atomic transition: old workflow deactivated → new workflow activated (with rollback on failure)
6. On success, the `Process` document is updated with `activeDataIngestion`:

```typescript
// Denormalized on Process document for efficient Firestore trigger access
Process.activeDataIngestion = {
  dataIngestionId: 'di_abc123',
  version: '1.0',
  webhookUrls: {
    embed: 'https://n8n-base-url/webhook/processId/embed',
    delete: 'https://n8n-base-url/webhook/processId/embed-delete',
  },
};
```

### 2.5 Firestore Trigger: onProcessKnowledgeDocumentParsed

**Trigger path:** `processes/{processId}/knowledge_base/{docId}`

**Condition:** Document update where `parsingData.status` changes to `'completed'`

**Flow:**

1. Checks that parsing just completed and document has parsed text
2. Fetches `Process.activeDataIngestion.webhookUrls.embed`
3. Generates a UUID `embeddingSecret` and updates the document:
   ```
   embeddingData.status = 'pending'
   embeddingData.embeddingSecret = <uuid>
   embeddingData.triggeredAt = <serverTimestamp>
   ```
4. Builds payload and calls the embed webhook (fire-and-forget)

**Payload sent to the n8n workflow:**

```typescript
{
  doc_id: string;           // Knowledge document ID
  process_id: string;       // Process ID
  data_ingestion_id: string; // ProcessDataIngestion ID
  file_name: string;        // Original file name
  file_type: string;        // MIME type
  markdown_content: string;  // Parsed document text
  existing_tags: string[];   // Tags already on the document
  available_tags: string[];  // All tags from Process.tags (for auto-tagging)
  callback_url: string;      // submitEmbeddingResult URL
  embedding_secret: string;  // UUID for callback authentication
}
```

### 2.6 N8n Workflow: Pinecone Namespace

In the process-level DI workflow, the **Codika Init Data Ingestion** node uses:

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initDataIngestion",
    "namespace": "{{PROCDATA_NAMESPACE_ATADCORP}}"
  }
}
```

`{{PROCDATA_NAMESPACE_ATADCORP}}` resolves to `process_{processId}` — a shared namespace for all instances of this process.

### 2.7 Folder Structure

```
use-cases/
└── your-use-case/
    ├── config.ts                              # getConfiguration() + getDataIngestionConfig()
    └── workflows/
        ├── main-workflow.json                 # User-facing workflow
        ├── helper-subworkflow.json            # Optional sub-workflow
        └── my-embedding-ingestion.json        # Data ingestion workflow
```

---

## 3. Instance-Level Data Ingestion

Instance-level data ingestion provides **per-user isolation**. When a user uploads documents to their "My Files" KB, those documents are embedded into a namespace scoped to their process instance (`{processInstanceId}`). Other users cannot access these embeddings.

### 3.1 Deployment: Part of Normal Process Deployment

Unlike process-level DI, instance-level DI is **NOT deployed separately**. It is defined as a regular workflow with a `data_ingestion` trigger type inside the `workflows` array of `getConfiguration()`. When users install the process, the DI workflow is deployed alongside all other workflows.

**Why in-process?** Instance-level DI is per-user. Each user's instance needs its own DI workflow with its own webhook URLs. This is naturally handled by the per-instance deployment that already exists for user-facing workflows.

### 3.2 Configuration in config.ts

Instance-level DI is configured as a workflow inside `getConfiguration()`:

```typescript
import {
  loadAndEncodeWorkflow,
  type ProcessDeploymentConfigurationInput,
} from 'codika';

export function getConfiguration(): ProcessDeploymentConfigurationInput {
  // ... other workflows ...

  const workflows = [
    // User-facing workflows
    {
      workflowTemplateId: 'my-rag-workflow',
      // ... trigger, schema, etc.
    },

    // Instance-level data ingestion workflow
    {
      workflowTemplateId: 'instance-embedding-ingestion',
      workflowId: 'instance-embedding-ingestion',
      workflowName: 'Instance Embedding Ingestion',
      markdownInfo: getInstanceIngestionWorkflowMarkdown(),
      knowledgeBaseAccess: {
        processDocTags: [],
        processInstanceDocTags: ['*'], // Access to all instance KB docs
      },
      integrationUids: ['openai', 'pinecone'],
      triggers: [
        {
          triggerId: crypto.randomUUID(),
          type: 'data_ingestion' as const,
          title: 'Instance Document Embedding',
          description: 'Embeds instance KB documents into Pinecone for RAG retrieval',
          webhooks: {
            embed: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed',
            delete: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed-delete',
          },
          cost: 2, // 2 credits per successful embedding
        },
      ],
      outputSchema: [],
      n8nWorkflowJsonBase64: loadAndEncodeWorkflow(
        join(__dirname, 'workflows/instance-embedding-ingestion.json')
      ),
      cost: 0, // Cost is tracked via the trigger's cost field, not workflow cost
    },
  ];

  return {
    title: 'My Use Case',
    // ...
    workflows,
  };
}
```

### 3.3 The `data_ingestion` Trigger Type

The `data_ingestion` trigger is a new trigger type specifically for data ingestion workflows. Key characteristics:

- **NOT shown in the playground UI** — users cannot manually trigger it
- **NOT counted in trigger count** — the `getTriggerCount()` helper excludes it
- **Triggered internally** — by a Firestore trigger when KB documents finish parsing
- **Has webhook paths** — similar to HTTP triggers, but with different placeholders

#### Trigger Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `triggerId` | `string` | Yes | Unique identifier (`crypto.randomUUID()`) |
| `type` | `'data_ingestion'` | Yes | Must be `'data_ingestion'` |
| `title` | `string` | No | Display title |
| `description` | `string` | No | Description of what this DI does |
| `webhooks.embed` | `string` | Yes | Webhook path for embedding (with placeholders) |
| `webhooks.delete` | `string` | Yes | Webhook path for deletion (with placeholders) |
| `cost` | `number` | No | Credit cost per successful embedding |

### 3.4 Webhook Path Placeholders

Instance-level DI uses **user data placeholders** because each instance has its own namespace:

```
embed:  {{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed
delete: {{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed-delete
```

These get resolved at user installation time to:
```
embed:  abc123instanceId/instance-embed
delete: abc123instanceId/instance-embed-delete
```

**CRITICAL DISTINCTION:**
- Process-level uses `{{PROCDATA_PROCESS_ID_ATADCORP}}` → same for all users
- Instance-level uses `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` → unique per user

### 3.5 How Instance-Level DI Gets Deployed (Backend Flow)

When a user installs a process (creates a process instance):

1. All workflows (including the DI workflow) are deployed to n8n via `deployWorkflowsToN8n()`
2. During deployment, placeholders in webhook paths are replaced (e.g., `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` → actual instance ID)
3. After deployment, the backend scans for `data_ingestion` triggers in the deployed workflows
4. If found, it extracts the resolved webhook URLs and denormalizes them:

```typescript
// Stored on ProcessDeploymentInstance
ProcessDeploymentInstance.activeDataIngestion = {
  webhookUrls: {
    embed: 'https://n8n-base-url/webhook/instanceId/instance-embed',
    delete: 'https://n8n-base-url/webhook/instanceId/instance-embed-delete',
  },
  cost: 2,
  workflowTemplateId: 'instance-embedding-ingestion',
};

// Denormalized on ProcessInstance for Firestore trigger access
ProcessInstance.activeDataIngestion = {
  deploymentInstanceId: 'deployment_xyz',
  webhookUrls: {
    embed: 'https://n8n-base-url/webhook/instanceId/instance-embed',
    delete: 'https://n8n-base-url/webhook/instanceId/instance-embed-delete',
  },
  cost: 2,
};
```

This denormalization happens in:
- `createProcessInstance` — initial installation
- `updateProcessInstance` — version updates
- `redeployDeploymentInstance` — redeployments

### 3.6 Firestore Trigger: onProcessInstanceKnowledgeDocumentParsed

**Trigger path:** `process_instances/{processInstanceId}/knowledge_base/{docId}`

**Condition:** Document update where `parsingData.status` changes to `'completed'`

**Flow:**

1. Checks that parsing just completed and document has parsed text
2. Fetches `ProcessInstance.activeDataIngestion.webhookUrls.embed`
3. Generates a UUID `embeddingSecret` and updates the document:
   ```
   embeddingData.status = 'pending'
   embeddingData.embeddingSecret = <uuid>
   embeddingData.triggeredAt = <serverTimestamp>
   ```
4. Builds payload and calls the embed webhook (fire-and-forget)

**Payload sent to the n8n workflow:**

```typescript
{
  doc_id: string;                    // Knowledge document ID
  process_instance_id: string;       // Process Instance ID (NOT process ID)
  context_type: 'process_instance';  // Tells callback which path to use
  file_name: string;                 // Original file name
  file_type: string;                 // MIME type
  markdown_content: string;           // Parsed document text
  existing_tags: string[];            // Tags already on the document
  available_tags: string[];           // Empty for now (can be extended later)
  callback_url: string;               // submitEmbeddingResult URL
  embedding_secret: string;           // UUID for callback authentication
  cost?: number;                      // Cost for credit deduction (from trigger config)
}
```

**Key differences from process-level payload:**
- `process_instance_id` instead of `process_id`
- `context_type: 'process_instance'` field (tells the callback to look in `process_instances/` path)
- `cost` is sent directly in the payload (no `data_ingestion_id` to look up)
- No `data_ingestion_id` field (there is no separate `ProcessDataIngestion` entity)

### 3.7 N8n Workflow: Pinecone Namespace

In the instance-level DI workflow, the **Codika Init Data Ingestion** node uses:

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initDataIngestion",
    "namespace": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
  }
}
```

`{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` resolves to the user's process instance ID — a per-user isolated namespace.

**CRITICAL:** This is different from process-level which uses `{{PROCDATA_NAMESPACE_ATADCORP}}` (resolves to `process_{processId}`). Using the wrong placeholder would mix user data across instances or break isolation.

### 3.8 Folder Structure

The instance-level DI workflow lives alongside other workflows:

```
use-cases/
└── your-use-case/
    ├── config.ts                                    # getConfiguration() includes DI workflow
    └── workflows/
        ├── main-workflow.json                       # User-facing workflow
        ├── helper-subworkflow.json                  # Optional sub-workflow
        └── instance-embedding-ingestion.json        # Instance-level DI workflow
```

**No separate deployment needed** — the DI workflow deploys with the process via `codika deploy use-case`.

---

## 4. N8n Workflow Structure

Both process-level and instance-level DI workflows follow the same n8n workflow pattern. The only differences are the webhook paths and namespace placeholders.

### 4.1 Mandatory Pattern

```
Webhook Trigger (/embed)
        |
Codika Init Data Ingestion
(extracts: docId, markdownContent, contentHash, namespace, callbackUrl)
        |
Check Existing Hash (query Pinecone for doc_id)
        |
  +-----+-----+
  |           |
Unchanged   Changed/New
  |           |
Codika      [Optional: Delete existing vectors]
Ingestion       |
Callback    [Optional: Classify document type]
(skipped)       |
            [Optional: Extract sections]
                |
            [Optional: Extract metadata]
                |
            Prepare Documents (build doc array with metadata)
                |
            Loop Over Documents → Pinecone Vector Store (insert)
                |
            Codika Ingestion Callback (success)
```

### 4.2 Two Webhook Triggers

Every DI workflow needs **two** webhook triggers:

1. **Embed webhook** (`/embed`): Receives document content, processes and embeds it
2. **Delete webhook** (`/embed-delete`): Receives `doc_id`, deletes all vectors for that document

The delete webhook is called when a document is archived (soft-deleted from the KB).

### 4.3 Standard Nodes (MUST Keep Unchanged)

These nodes form the backbone and must be present:

| Node | Purpose |
|------|---------|
| **Webhook Trigger** | Receives document from Firestore trigger |
| **Codika Init Data Ingestion** | Extracts docId, contentHash, markdownContent, namespace |
| **Check Existing Hash** | Queries Pinecone by `doc_id` to check for duplicates |
| **Check Duplicate** (Code node) | SKIP/UPDATE/CREATE decision based on hash comparison |
| **Codika Ingestion Callback** | Reports status (success/skipped/failed) to platform |

### 4.4 Customization Points (Areas of Liberty)

These nodes can be customized per use case:

| Node | Purpose | Example |
|------|---------|---------|
| **Document Classification** | Classify document type using LLM | "proposal" vs "rfp" vs "unknown" |
| **Section Extraction** | Extract semantic sections | "methodology", "pricing", "team" |
| **Metadata Extraction** | Extract structured metadata | client name, thematic, duration |
| **Semantic Signature** | Generate similarity fingerprint | 8-field business context summary |
| **Prepare Documents** | Build document array for embedding | Multiple docs per input (sections + full + semantic) |

### 4.5 Check Duplicate Logic

The Code node implementing deduplication:

```javascript
const docId = $('Codika Init Data Ingestion').item.json.docId;
const contentHash = $('Codika Init Data Ingestion').item.json.contentHash;
const existingMatches = $('Check Existing Hash').first().json;

// Check if this document already exists in the vector store
const existingVectors = existingMatches?.matches || [];

if (existingVectors.length === 0) {
  // New document - create
  return [{ json: { action: 'CREATE', docId, contentHash } }];
}

// Check content hash
const existingHash = existingVectors[0]?.metadata?.content_hash;
if (existingHash === contentHash) {
  // Content unchanged - skip
  return [{ json: { action: 'SKIP', docId, reason: 'Content hash unchanged' } }];
}

// Content changed - update (delete old, re-embed)
return [{ json: { action: 'UPDATE', docId, contentHash } }];
```

### 4.6 Pinecone Configuration

| Setting | Value |
|---------|-------|
| **Embedding model** | `text-embedding-3-small` |
| **Dimensions** | 512 |
| **Max input length** | 8000 characters (truncate before embedding) |
| **Hash algorithm** | cyrb53 (provided by Codika Init Data Ingestion) |

#### Pinecone Credentials

**CRITICAL:** All Pinecone nodes — both LangChain nodes and direct HTTP request nodes — MUST use Flex Credentials (`FLEXCRED_PINECONE_*`). Never hardcode Pinecone credential IDs.

**LangChain Pinecone Vector Store node:**

```json
"credentials": {
  "pineconeApi": {
    "id": "{{FLEXCRED_PINECONE_ID_DERCXELF}}",
    "name": "{{FLEXCRED_PINECONE_NAME_DERCXELF}}"
  }
}
```

**HTTP Request nodes (query, delete, etc.):**

Use `predefinedCredentialType` with `pineconeApi` — do NOT use `genericCredentialType` with `httpHeaderAuth`:

```json
{
  "parameters": {
    "method": "POST",
    "url": "https://codika-default-index-1x4c51a.svc.gcp-europe-west4-de1d.pinecone.io/query",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "pineconeApi",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "=...",
    "options": {}
  },
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "pineconeApi": {
      "id": "{{FLEXCRED_PINECONE_ID_DERCXELF}}",
      "name": "{{FLEXCRED_PINECONE_NAME_DERCXELF}}"
    }
  }
}
```

Similarly, OpenAI embedding HTTP requests use `predefinedCredentialType` with `openAiApi`:

```json
"credentials": {
  "openAiApi": {
    "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
  }
}
```

### 4.7 Pinecone Metadata Constraints

- No nested objects in metadata
- Arrays must be serialized as comma-separated strings
- Each field must be `string`, `number`, or `boolean`
- Always include: `doc_id`, `content_hash`, `embedding_type` ('semantic' or 'raw')

### 4.8 Differences Between Process-Level and Instance-Level Workflows

| Aspect | Process-Level | Instance-Level |
|--------|---------------|----------------|
| **Webhook path** | `{{PROCDATA_PROCESS_ID_ATADCORP}}/embed` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed` |
| **Delete path** | `{{PROCDATA_PROCESS_ID_ATADCORP}}/embed-delete` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/instance-embed-delete` |
| **Namespace** | `{{PROCDATA_NAMESPACE_ATADCORP}}` → `process_{processId}` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` → `{instanceId}` |
| **Credentials** | `{{FLEXCRED_OPENAI_*}}`, `{{FLEXCRED_PINECONE_*}}` | Same — uses Flex Credentials |
| **Workflow settings** | Same `executionOrder: "v1"` and `errorWorkflow` | Same |
| **Codika Init Data Ingestion** | Present, with PROCDATA namespace | Present, with USERDATA namespace |

**The n8n workflow JSON is structurally identical** — the only things that change are the placeholder strings for webhook paths and namespace.

---

## 5. Codika Nodes for Data Ingestion

### 5.1 Codika Init Data Ingestion

The first node after the webhook trigger. Extracts document metadata from the webhook payload.

**Parameters:**

| Parameter | Required | Process-Level Value | Instance-Level Value |
|-----------|----------|---------------------|----------------------|
| `namespace` | No | `{{PROCDATA_NAMESPACE_ATADCORP}}` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` |

**Auto-extracted from webhook payload:**

| Field | Description |
|-------|-------------|
| `doc_id` / `docId` | Knowledge document ID |
| `markdown_content` / `markdownContent` | Parsed document text |
| `callback_url` / `callbackUrl` | URL for status reporting |
| `embedding_secret` / `embeddingSecret` | Authentication secret for callback |
| `process_id` / `processId` | Process ID (process-level DI) |
| `process_instance_id` / `processInstanceId` | Process Instance ID (instance-level DI) |
| `context_type` / `contextType` | `'process'` or `'process_instance'` — used by callback to resolve Firestore path |
| `cost` | Credit cost per embedding (instance-level DI — passed directly in payload) |
| `data_ingestion_id` / `dataIngestionId` | Data ingestion entity ID (process-level DI) |

**Output:**

```json
{
  "docId": "abc123",
  "documentId": "abc123_sanitized",
  "contentHash": "cyrb53_hash_string",
  "markdownContent": "# Document Title\n\nContent...",
  "namespace": "process_xyz or instanceId",
  "processId": "",
  "processInstanceId": "instanceId or empty",
  "callbackUrl": "https://api.codika.io/submitembeddingresult",
  "embeddingSecret": "uuid-v4",
  "hasCallback": true,
  "dataIngestionId": "",
  "contextType": "process or process_instance",
  "cost": 2,
  "_startTimeMs": 1700000000000
}
```

**Example (Process-Level):**

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initDataIngestion",
    "namespace": "{{PROCDATA_NAMESPACE_ATADCORP}}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init Data Ingestion"
}
```

**Example (Instance-Level):**

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initDataIngestion",
    "namespace": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init Data Ingestion"
}
```

### 5.2 Codika Ingestion Callback

Reports the embedding result back to the platform. Must be the last node on all paths.

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `status` | Yes | `'success'`, `'skipped'`, or `'failed'` |
| `extractedTags` | No | Comma-separated tags or JSON array |
| `skipReason` | Conditional | Required when `status = 'skipped'` |
| `errorMessage` | Conditional | Required when `status = 'failed'` |

**Auto-propagated fields (from Init Data Ingestion via execution context):**

The callback node automatically reads and forwards these fields to the `submitEmbeddingResult` endpoint. No configuration needed — they are set by the Init Data Ingestion node.

| Field in callback payload | Source | Description |
|---------------------------|--------|-------------|
| `doc_id` | Init Data Ingestion | Document ID |
| `process_id` | Init Data Ingestion | Process ID (process-level DI, omitted if empty) |
| `process_instance_id` | Init Data Ingestion | Process Instance ID (instance-level DI, omitted if empty) |
| `context_type` | Init Data Ingestion | `'process'` or `'process_instance'` (omitted if empty) |
| `data_ingestion_id` | Init Data Ingestion | ProcessDataIngestion entity ID (process-level DI, omitted if empty) |
| `cost` | Init Data Ingestion | Credit cost (instance-level DI, omitted if empty) |
| `embedding_secret` | Init Data Ingestion | Authentication secret |

**Example (Success):**

```json
{
  "parameters": {
    "resource": "dataIngestion",
    "operation": "ingestionCallback",
    "status": "success",
    "extractedTags": "={{ $json.extractedTags }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Callback"
}
```

**Example (Skipped):**

```json
{
  "parameters": {
    "resource": "dataIngestion",
    "operation": "ingestionCallback",
    "status": "skipped",
    "skipReason": "={{ $json.reason }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Skip Callback"
}
```

---

## 6. The Callback System

### 6.1 submitEmbeddingResult

Both process-level and instance-level DI use the same callback endpoint: `submitEmbeddingResult`.

**URL:** `https://api.codika.io/submitembeddingresult`

**Method:** POST

**How it works:**

1. The n8n workflow (via Codika Ingestion Callback) sends a POST to the callback URL
2. The callback validates the `embedding_secret` against the one stored on the document
3. Based on `status`, it updates the document's `embeddingData`:
   - `success` → `embeddingData.status = 'completed'`, tags merged
   - `skipped` → `embeddingData.status = 'skipped'`, skip reason stored
   - `failed` → `embeddingData.status = 'failed'`, error stored, retry count incremented

### 6.2 Context-Aware Document Resolution

The callback determines which Firestore path to use based on `context_type`:

```
context_type = 'process' (default):
  → processes/{process_id}/knowledge_base/{doc_id}

context_type = 'process_instance':
  → process_instances/{process_instance_id}/knowledge_base/{doc_id}
```

Process-level DI does NOT send `context_type` (defaults to `'process'` for backward compatibility). Instance-level DI always sends `context_type: 'process_instance'`.

### 6.3 Cost Deduction in Callback

Credits are deducted only on `status: 'success'`:

| Level | Cost Source | Entity Looked Up |
|-------|------------|------------------|
| Process-level | `ProcessDataIngestion.config.cost` | ProcessDataIngestion entity |
| Instance-level | `cost` field in callback payload | None (cost passed directly) |

---

## 7. Cost and Credit Deduction

### 7.1 Where Cost is Defined

- **Process-level:** In `getDataIngestionConfig().cost` → stored on `ProcessDataIngestion.config.cost`
- **Instance-level:** In the `data_ingestion` trigger's `cost` field → passed in webhook payload → forwarded to callback

### 7.2 When Deduction Happens

Credits are deducted only when the callback reports `status: 'success'`. Failed or skipped embeddings are free.

### 7.3 Cost Flow

```
Document uploaded → parsed → embedding triggered → n8n processes → callback
                                                                      ↓
                                                               status = 'success'?
                                                              Yes → Deduct credits
                                                              No  → No charge
```

---

## 8. Archive and Unarchive Behavior

### 8.1 Archive (Soft Delete)

When a KB document is archived:

1. The document is marked `archived: true`
2. If the document had `embeddingData.status === 'completed'`:
   - For **process** context: Calls `Process.activeDataIngestion.webhookUrls.delete` with `{ doc_id: knowledgeId }`
   - For **process_instance** context: Calls `ProcessInstance.activeDataIngestion.webhookUrls.delete` with `{ doc_id: knowledgeId }`
3. The delete webhook tells the n8n DI workflow to remove all vectors for that `doc_id` from Pinecone
4. `embeddingData` is cleared from the document

### 8.2 Unarchive (Restore)

When a KB document is unarchived:

1. The document is marked `archived: false`
2. If the document had `parsingData.status === 'completed'` (was previously parsed):
   - Re-embedding is automatically triggered (same flow as initial embedding)
   - A new `embeddingSecret` is generated
   - The embed webhook is called with the document content
   - This ensures the document is re-embedded after being restored

---

## 9. Comparison Table

| Aspect | Process-Level DI | Instance-Level DI |
|--------|------------------|-------------------|
| **Scope** | Shared across all instances | Per-user isolated |
| **KB path** | `processes/{id}/knowledge_base/` | `process_instances/{id}/knowledge_base/` |
| **UI label** | "Global Files" | "My Files" |
| **Pinecone namespace** | `process_{processId}` | `{processInstanceId}` |
| **Deployment** | `codika deploy process-data-ingestion <path>` | With process (`codika deploy use-case <path>`) |
| **Versioning** | Independent (ProcessDataIngestion entity) | Follows process deployment version |
| **Config location** | `getDataIngestionConfig()` | Inside `getConfiguration().workflows[]` |
| **Trigger type** | N/A (separate entity) | `data_ingestion` trigger |
| **Webhook path placeholder** | `{{PROCDATA_PROCESS_ID_ATADCORP}}` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` |
| **Namespace placeholder** | `{{PROCDATA_NAMESPACE_ATADCORP}}` | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` |
| **Denormalized on** | `Process.activeDataIngestion` | `ProcessInstance.activeDataIngestion` |
| **Cost entity** | `ProcessDataIngestion.config.cost` | Trigger `cost` field → payload |
| **Firestore trigger** | `onProcessKnowledgeDocumentParsed` | `onProcessInstanceKnowledgeDocumentParsed` |
| **Callback payload** | `process_id`, no `context_type` | `process_instance_id`, `context_type: 'process_instance'` |
| **User notification on DI update** | No (separate deployment) | Yes (part of process deployment) |

---

## 10. Complete Examples

### 10.1 Process-Level Only (Shared Knowledge Base)

A use case where admins upload shared documents that all users can query:

```typescript
// config.ts
export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'Company Knowledge Base',
    subtitle: 'Query company documents with AI',
    description: 'RAG-powered Q&A over shared company knowledge.',
    workflows: [
      {
        workflowTemplateId: 'knowledge-query',
        workflowId: 'knowledge-query',
        workflowName: 'Ask a Question',
        knowledgeBaseAccess: {
          processDocTags: ['*'],        // Access ALL process KB docs
          processInstanceDocTags: [],    // No instance docs
        },
        integrationUids: ['openai', 'anthropic', 'pinecone'],
        triggers: [/* HTTP trigger */],
        outputSchema: [/* ... */],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./workflows/knowledge-query.json'),
        cost: 3,
      },
    ],
    tags: ['policy', 'procedure', 'guide'],
    integrationUids: ['openai', 'anthropic', 'pinecone'],
  };
}

// Separate DI config
export function getDataIngestionConfig(): ProcessDataIngestionConfigInput {
  return {
    workflowTemplateId: 'knowledge-embedding',
    workflowName: 'Knowledge Embedding',
    n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./data-ingestion/knowledge-embedding.json'),
    webhooks: {
      embed: '{{PROCDATA_PROCESS_ID_ATADCORP}}/embed',
      delete: '{{PROCDATA_PROCESS_ID_ATADCORP}}/embed-delete',
    },
    purpose: 'Embed company documents for RAG retrieval',
    cost: 2,
  };
}
```

### 10.2 Instance-Level Only (Personal Knowledge Base)

A use case where each user has their own documents:

```typescript
// config.ts
export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'Personal Research Assistant',
    subtitle: 'Upload your documents and ask questions',
    description: 'AI assistant that answers questions based on YOUR uploaded documents.',
    workflows: [
      // User-facing workflow
      {
        workflowTemplateId: 'personal-query',
        workflowId: 'personal-query',
        workflowName: 'Ask My Documents',
        knowledgeBaseAccess: {
          processDocTags: [],             // No shared docs
          processInstanceDocTags: ['*'],  // Access ALL user's docs
        },
        integrationUids: ['openai', 'anthropic', 'pinecone'],
        triggers: [/* HTTP trigger */],
        outputSchema: [/* ... */],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./workflows/personal-query.json'),
        cost: 3,
      },
      // Instance-level DI workflow
      {
        workflowTemplateId: 'personal-embedding-ingestion',
        workflowId: 'personal-embedding-ingestion',
        workflowName: 'Personal Document Embedding',
        markdownInfo: '# Personal Embedding\n\nEmbeds your documents into your private knowledge base.',
        knowledgeBaseAccess: {
          processDocTags: [],
          processInstanceDocTags: ['*'],
        },
        integrationUids: ['openai', 'pinecone'],
        triggers: [
          {
            triggerId: crypto.randomUUID(),
            type: 'data_ingestion' as const,
            title: 'Personal Document Embedding',
            description: 'Embeds user documents into private Pinecone namespace',
            webhooks: {
              embed: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/personal-embed',
              delete: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/personal-embed-delete',
            },
            cost: 2,
          },
        ],
        outputSchema: [],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow(
          './workflows/personal-embedding-ingestion.json'
        ),
        cost: 0,
      },
    ],
    integrationUids: ['openai', 'anthropic', 'pinecone'],
  };
}

// No getDataIngestionConfig() needed — DI is inside getConfiguration()
```

### 10.3 Both Levels (Shared + Personal)

A use case with both shared global knowledge and per-user documents:

```typescript
// config.ts
export function getConfiguration(): ProcessDeploymentConfigurationInput {
  return {
    title: 'Legal Research Assistant',
    subtitle: 'Research using firm knowledge and your case files',
    description: 'AI legal assistant with access to shared firm knowledge and your personal case documents.',
    workflows: [
      // User-facing: query both KBs
      {
        workflowTemplateId: 'legal-research',
        workflowId: 'legal-research',
        workflowName: 'Legal Research',
        knowledgeBaseAccess: {
          processDocTags: ['*'],          // Access shared firm docs
          processInstanceDocTags: ['*'],  // Access personal case docs
        },
        integrationUids: ['openai', 'anthropic', 'pinecone'],
        triggers: [/* HTTP trigger */],
        outputSchema: [/* ... */],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./workflows/legal-research.json'),
        cost: 5,
      },
      // Instance-level DI for personal case documents
      {
        workflowTemplateId: 'case-embedding-ingestion',
        workflowId: 'case-embedding-ingestion',
        workflowName: 'Case Document Embedding',
        markdownInfo: '# Case Embedding\n\nEmbeds your case documents for personal RAG.',
        knowledgeBaseAccess: {
          processDocTags: [],
          processInstanceDocTags: ['*'],
        },
        integrationUids: ['openai', 'pinecone'],
        triggers: [
          {
            triggerId: crypto.randomUUID(),
            type: 'data_ingestion' as const,
            title: 'Case Document Embedding',
            webhooks: {
              embed: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/case-embed',
              delete: '{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/case-embed-delete',
            },
            cost: 2,
          },
        ],
        outputSchema: [],
        n8nWorkflowJsonBase64: loadAndEncodeWorkflow(
          './workflows/case-embedding-ingestion.json'
        ),
        cost: 0,
      },
    ],
    tags: ['contract', 'precedent', 'regulation', 'memo'],
    integrationUids: ['openai', 'anthropic', 'pinecone'],
  };
}

// Process-level DI for shared firm knowledge (deployed separately)
export function getDataIngestionConfig(): ProcessDataIngestionConfigInput {
  return {
    workflowTemplateId: 'firm-embedding-ingestion',
    workflowName: 'Firm Knowledge Embedding',
    n8nWorkflowJsonBase64: loadAndEncodeWorkflow('./data-ingestion/firm-embedding-ingestion.json'),
    webhooks: {
      embed: '{{PROCDATA_PROCESS_ID_ATADCORP}}/firm-embed',
      delete: '{{PROCDATA_PROCESS_ID_ATADCORP}}/firm-embed-delete',
    },
    purpose: 'Embed shared firm knowledge for RAG retrieval',
    cost: 2,
  };
}
```

In this example:
- Shared firm documents (contracts, precedents) go into `Global Files` → process-level DI → `process_{processId}` namespace
- Personal case documents go into `My Files` → instance-level DI → `{processInstanceId}` namespace
- The user-facing workflow queries both namespaces during RAG retrieval

---

## Related Documentation

- [use-case-guide.md](../use-case-guide.md) — Main use case creation guide
- [config-patterns.md](./config-patterns.md) — config.ts field reference
- [codika-nodes.md](./codika-nodes.md) — Codika node parameters (Init Data Ingestion, Ingestion Callback)
- [placeholder-patterns.md](./placeholder-patterns.md) — Complete placeholder reference
- [RAG_INTEGRATION_GUIDE.md](../../.claude/docs/features/rag_integration/RAG_INTEGRATION_GUIDE.md) — Deep RAG implementation guide with n8n workflow details
