# Codika Custom Nodes Guide

Codika provides custom n8n nodes that handle execution tracking, result submission, file uploads, and error reporting. These nodes encapsulate the Codika platform integration logic.

---

## 1. Critical Workflow Rules

Every workflow MUST follow these rules:

**Rule 1: Every workflow MUST start with a Codika Init node**

- HTTP-triggered workflows: Codika Init extracts execution metadata from the webhook payload
- Schedule/third-party triggered workflows: Codika Init creates a new execution via API call

**Rule 2: Every workflow branch MUST end with either Codika Submit Result OR Codika Report Error**

- Success paths: Codika Submit Result
- Error/failure paths: Codika Report Error
- No workflow execution should complete without one of these terminal nodes

**Rule 3: NEVER use a Merge node to reconverge conditional branches before a Codika terminal node**

The n8n Merge node waits for ALL its inputs to receive data before executing. If only one conditional branch runs (e.g., "has emails" vs "no emails"), the Merge node stalls forever and the terminal node never triggers.

Instead, connect each branch directly to `Codika Submit Result`:

```text
IF ─┬→ Path A → ... → Codika Submit Result
    └→ Path B → ... → Codika Submit Result
```

If both paths produce the same output shape, connect them both to the same `Codika Submit Result` node (only one will run per execution). Use the Format/Code node output (not a downstream API response) as the input to `Codika Submit Result` so `$json` contains the result data.

```text
Trigger (HTTP/Schedule/Service)
        |
  Codika Init       <-- REQUIRED (first node after trigger)
        |
  Business Logic
        |
  +-----+-----+
  |           |
Success    Failure
  |           |
Codika     Codika
Submit     Report     <-- REQUIRED (all branches must end here)
Result     Error
```

---

## 2. Available Nodes

The Codika node provides 6 operations organized into 5 resource categories:

| Resource | Operation | Purpose |
|----------|-----------|---------|
| Initialize Execution | `initWorkflow` | Initialize workflow execution tracking |
| Initialize Execution | `initDataIngestion` | Initialize RAG data ingestion workflows |
| Workflow Outputs | `submitResult` | Submit successful workflow results |
| File Management | `uploadFile` | Upload files to Codika storage |
| Data Ingestion | `ingestionCallback` | Report RAG ingestion status |
| Error Handling | `reportError` | Report workflow errors |

---

## 3. Codika Init (initWorkflow)

Initializes execution tracking for standard workflows. Handles both HTTP-triggered and schedule/third-party triggered workflows.

### Behavior

- **HTTP triggers:** Automatically extracts `executionMetadata` from the webhook payload (passthrough mode)
- **Schedule/third-party triggers:** Creates a new execution by calling the `createWorkflowExecution` API

### Parameters (Schedule/Third-Party Triggers)

| Parameter | Required | Placeholder | Description |
|-----------|----------|-------------|-------------|
| `memberSecret` | Yes | `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}` | Member's execution auth secret |
| `organizationId` | Yes | `{{USERDATA_ORGANIZATION_ID_ATADRESU}}` | Organization ID |
| `userId` | Yes | `{{USERDATA_USER_ID_ATADRESU}}` | User ID |
| `processInstanceId` | Yes | `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` | Process instance ID |
| `workflowId` | Yes | (literal string) | Workflow template ID (e.g., `"gmail-draft-assistant"`) |
| `triggerType` | Yes | (literal string) | Trigger source (e.g., `"schedule"`, `"gmail"`) |

### Parameters (HTTP Triggers)

For HTTP triggers, metadata is extracted from the webhook payload:

| Parameter | Required | Value | Description |
|-----------|----------|-------|-------------|
| `memberSecret` | Yes | `{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}` | Member's execution auth secret |
| `organizationId` | Yes | `={{$json.body.executionMetadata.organizationId}}` | From webhook |
| `userId` | Yes | `={{$json.body.executionMetadata.userId}}` | From webhook |
| `processInstanceId` | Yes | `={{$json.body.executionMetadata.processInstanceId}}` | From webhook |
| `workflowId` | Yes | (literal string) | Workflow template ID |
| `triggerType` | Yes | `"http"` | Always "http" |

### Output

```json
{
  "executionId": "string",
  "executionSecret": "string",
  "callbackUrl": "string",
  "errorCallbackUrl": "string",
  "processId": "string",
  "processInstanceId": "string",
  "userId": "string (Firebase UID, NOT an email address)",
  "workflowId": "string",
  "_startTimeMs": "number",
  "_mode": "passthrough|create"
}
```

### Example (Schedule Trigger)

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "{{USERDATA_ORGANIZATION_ID_ATADRESU}}",
    "userId": "{{USERDATA_USER_ID_ATADRESU}}",
    "processInstanceId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
    "workflowId": "poem-generator",
    "triggerType": "schedule"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

### Example (HTTP Trigger)

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "={{$json.body.executionMetadata.organizationId}}",
    "userId": "={{$json.body.executionMetadata.userId}}",
    "processInstanceId": "={{$json.body.executionMetadata.processInstanceId}}",
    "workflowId": "success-error-demo",
    "triggerType": "http"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

---

## 4. Codika Submit Result (submitResult)

Submits successful workflow completion with output data back to the Codika platform.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `resultData` | Yes | JSON object containing workflow results. Must match the workflow's output schema. |

### Output

```json
{
  "success": true,
  "executionId": "string",
  "submittedAt": "ISO string",
  "executionTimeMs": "number"
}
```

### Example

```json
{
  "parameters": {
    "resource": "workflowOutputs",
    "operation": "submitResult",
    "resultData": "={{ $json }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Submit Result"
}
```

### Accessing Previous Node Data

```javascript
// Reference specific node output
"resultData": "={{ $('Extract Poem').first().json }}"

// Build custom result object
"resultData": "={ \"status\": \"success\", \"message\": \"{{ $json.result }}\", \"timestamp\": \"{{ new Date().toISOString() }}\" }"
```

---

## 5. Codika Report Error (reportError)

Reports errors that occurred during workflow execution to the Codika platform.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `errorMessage` | Yes | Descriptive error message |
| `errorType` | Yes | Error category: `node_failure`, `validation_error`, `external_api_error`, `timeout` |
| `failedNodeName` | No | Name of the node that caused the failure |
| `lastExecutedNode` | No | Name of the last successfully executed node |

### Example

```json
{
  "parameters": {
    "resource": "errorHandling",
    "operation": "reportError",
    "errorMessage": "Demo error: The succeed parameter was set to false",
    "errorType": "validation_error"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Report Error"
}
```

### Dynamic Error Message

```json
{
  "parameters": {
    "resource": "errorHandling",
    "operation": "reportError",
    "errorMessage": "={{ $json.error?.message || 'Unknown error occurred' }}",
    "errorType": "node_failure",
    "failedNodeName": "={{ $json.failedNode || '' }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Report Error"
}
```

---

## 6. Codika Upload File (uploadFile)

Uploads files to Codika storage during workflow execution. Returns a `documentId` to include in the final result.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `binaryPropertyName` | Yes | Name of the binary property containing the file (default: `"data"`) |
| `fieldKey` | No | Output schema field key (e.g., `"generated_video"`) |
| `fileName` | No | Custom filename (uses original if not provided) |
| `mimeType` | No | MIME type (auto-detected if not provided) |
| `executionIdOverride` | No | Override execution ID (required for subworkflows) |
| `executionSecretOverride` | No | Override execution secret (required for subworkflows) |

### Output

```json
{
  "success": true,
  "documentId": "string",
  "storagePath": "string",
  "fileName": "string",
  "fileSize": "number",
  "mimeType": "string"
}
```

### Example (Parent Workflow)

In parent workflows, the execution context is automatically retrieved from Codika Init:

```json
{
  "parameters": {
    "resource": "fileManagement",
    "operation": "uploadFile",
    "binaryPropertyName": "data",
    "fieldKey": "generated_pdf",
    "fileName": "report.pdf",
    "mimeType": "application/pdf"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Upload File"
}
```

### Example (Subworkflow with Overrides)

In subworkflows, execution credentials must be passed from the parent and used via overrides:

```json
{
  "parameters": {
    "resource": "fileManagement",
    "operation": "uploadFile",
    "binaryPropertyName": "data",
    "fieldKey": "={{ $('When Executed by Another Workflow').first().json.fieldKey }}",
    "fileName": "={{ $('When Executed by Another Workflow').first().json.fileName }}",
    "mimeType": "application/pdf",
    "executionIdOverride": "={{ $('When Executed by Another Workflow').first().json.executionId }}",
    "executionSecretOverride": "={{ $('When Executed by Another Workflow').first().json.executionSecret }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Upload File"
}
```

### Using documentId in Submit Result

After uploading, include the `documentId` in the result data:

```javascript
// Prepare result with document ID
const uploadResult = $('Codika Upload File').first().json;

return [{
  json: {
    generated_pdf: uploadResult.documentId,  // Just the document ID
    generated_at: new Date().toISOString()
  }
}];
```

---

## 7. Codika Init Data Ingestion (initDataIngestion)

Initializes RAG data ingestion workflows for document embedding.

### Parameters

| Parameter | Required | Placeholder | Description |
|-----------|----------|-------------|-------------|
| `namespace` | No | `{{PROCDATA_NAMESPACE_ATADCORP}}` | Pinecone namespace for vector storage |

### Auto-extracted from Webhook Payload

| Field | Required | Description |
|-------|----------|-------------|
| `doc_id` / `docId` | Yes | Unique document identifier |
| `markdown_content` / `markdownContent` | Yes | Document content to embed |
| `callback_url` / `callbackUrl` | No | URL to report ingestion status |
| `embedding_secret` / `embeddingSecret` | No | Secret for callback authentication |
| `process_id` / `processId` | No | Process ID (process-level DI) |
| `process_instance_id` / `processInstanceId` | No | Process Instance ID (instance-level DI) |
| `context_type` / `contextType` | No | `'process'` or `'process_instance'` |
| `cost` | No | Credit cost per embedding (instance-level DI) |
| `data_ingestion_id` / `dataIngestionId` | No | Data ingestion entity ID (process-level DI) |

### Output

```json
{
  "docId": "string",
  "documentId": "string (sanitized)",
  "contentHash": "string (cyrb53 hash)",
  "markdownContent": "string",
  "namespace": "string",
  "processId": "string",
  "processInstanceId": "string",
  "callbackUrl": "string",
  "embeddingSecret": "string",
  "hasCallback": "boolean",
  "dataIngestionId": "string",
  "contextType": "string",
  "cost": "number | undefined",
  "_startTimeMs": "number"
}
```

### Example

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

---

## 8. Codika Ingestion Callback (ingestionCallback)

Reports data ingestion completion status back to the Codika platform.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `status` | Yes | Status: `success`, `skipped`, or `failed` |
| `extractedTags` | No | Comma-separated tags or JSON array extracted from document |
| `skipReason` | Conditional | Reason for skipping (required when `status=skipped`) |
| `errorMessage` | Conditional | Error message (required when `status=failed`) |

### Example (Success)

```json
{
  "parameters": {
    "resource": "dataIngestion",
    "operation": "ingestionCallback",
    "status": "success"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Callback"
}
```

### Example (Skipped - Content Unchanged)

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

## 9. Data Ingestion Workflow Pattern

RAG data ingestion workflows follow this pattern:

```text
Webhook Trigger (document)
        |
Codika Init Data Ingestion
(extracts: docId, markdownContent, contentHash, callbackUrl)
        |
Check Existing Hash (query Pinecone for existing document)
        |
  +-----+-----+
  |           |
Unchanged   Changed/New
  |           |
Codika      Process &
Ingestion   Embed
Callback    Document
(skipped)       |
            Store in
            Pinecone
                |
            Codika
            Ingestion
            Callback
            (success)
```

---

## 10. Accessing Execution Data in Downstream Nodes

Reference Codika Init output in downstream nodes:

```javascript
// Get execution ID
const executionId = $('Codika Init').first().json.executionId;

// Get start time for duration calculation
const startTimeMs = $('Codika Init').first().json._startTimeMs;
const durationMs = Date.now() - startTimeMs;

// Get callback URL (if needed for custom HTTP calls)
const callbackUrl = $('Codika Init').first().json.callbackUrl;
```

---

## 11. Checklist

- [ ] Codika Init is the first node after the trigger
- [ ] All success paths end with Codika Submit Result
- [ ] All error paths end with Codika Report Error
- [ ] resultData matches the output schema defined in config.ts
- [ ] File uploads use documentId in the final result
- [ ] Subworkflows pass executionId and executionSecret for file uploads
- [ ] Schedule/third-party triggers use placeholders for user/org data
- [ ] HTTP triggers extract metadata from webhook payload

---

## Related Documentation

- [http-triggers.md](./http-triggers.md) - HTTP trigger setup
- [schedule-triggers.md](./schedule-triggers.md) - Schedule trigger setup
- [sub-workflows.md](./sub-workflows.md) - Sub-workflow patterns
- [placeholder-patterns.md](./placeholder-patterns.md) - Placeholder reference
