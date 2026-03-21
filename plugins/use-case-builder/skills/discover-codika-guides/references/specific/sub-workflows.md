# Sub-Workflows Guide

Sub-workflows allow one workflow to call another within the same process deployment.

---

## 1. Overview

### Use Cases

- Reuse common logic across multiple workflows
- Break complex workflows into smaller, manageable pieces
- Create helper functions callable by multiple parent workflows

### Key Characteristics

- Sub-workflows are **not visible in the UI** for direct user execution
- Sub-workflows are callable only via n8n's `executeWorkflow` node
- Sub-workflows are deployed **before** their parent workflows (automatic dependency ordering)

---

## 2. Sub-Workflow Configuration (config.ts)

Define a sub-workflow using the `subworkflow` trigger type:

```typescript
{
  workflowTemplateId: 'my-helper',
  workflowName: 'Helper Workflow',
  triggers: [
    {
      triggerId: crypto.randomUUID(),
      type: 'subworkflow',
      title: 'Process Data',
      description: 'Called by other workflows to process data.',
      inputSchema: [
        { name: 'text', type: 'string' },
        { name: 'count', type: 'number' },
      ],
      calledBy: ['main-workflow'],  // Optional: documents which workflows call this
    }
  ],
  outputSchema: [],  // Required: sub-workflows return output directly to parent
  n8nWorkflowJsonBase64: helperWorkflowBase64,
  cost: 0,  // Sub-workflows have no cost (parent workflow pays)
}
```

### Configuration Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Set to `'subworkflow'` |
| `inputSchema` | Yes | Array of `{ name: string, type: InputType }` entries |
| `calledBy` | No | Array of parent workflow template IDs (documentation only) |
| `outputSchema` | Yes | Set to `[]` (empty array) |
| `cost` | Yes | Set to `0` (parent pays) |

### Input Schema Types

- `string` - Text input
- `number` - Numeric input
- `boolean` - True/false
- `array` - List of items
- `object` - Nested object

---

## 3. Placeholder Pattern

Reference a sub-workflow in parent workflow JSON using this pattern:

| Pattern | Description | Example |
|---------|-------------|---------|
| `{{SUBWKFL_<TEMPLATE_ID>_LFKWBUS}}` | Sub-workflow n8n ID | `{{SUBWKFL_my-helper_LFKWBUS}}` |

**Pattern structure:**
- `SUBWKFL_` - Prefix identifying a sub-workflow reference
- `<TEMPLATE_ID>` - The `workflowTemplateId` of the target sub-workflow
- `_LFKWBUS` - Suffix (reversed SUBWKFL)

At deployment, the system replaces this placeholder with the actual n8n workflow ID.

---

## 4. Parent Workflow: Execute Workflow Node

Use n8n's Execute Workflow node to call a sub-workflow:

```json
{
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "{{SUBWKFL_my-helper_LFKWBUS}}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "text": "={{ $json.text }}",
        "count": 5
      }
    },
    "options": {
      "waitForSubWorkflow": true
    }
  },
  "type": "n8n-nodes-base.executeWorkflow",
  "typeVersion": 1.3,
  "name": "Call Helper"
}
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `workflowId` | Uses `__rl` resource locator pattern with `SUBWKFL_*` placeholder |
| `workflowInputs.value` | Data passed to sub-workflow (key-value pairs) |
| `waitForSubWorkflow` | Set to `true` to wait for completion and receive output |

---

## 5. Sub-Workflow: Execute Workflow Trigger

Every sub-workflow starts with an Execute Workflow Trigger node:

```json
{
  "parameters": {
    "workflowInputs": {
      "values": [
        { "name": "text", "type": "string" },
        { "name": "count", "type": "number" }
      ]
    }
  },
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "typeVersion": 1.1,
  "name": "When Executed by Another Workflow"
}
```

Access input data in subsequent nodes via:
```javascript
$input.item.json.text
$input.item.json.count
```

Or reference the trigger node directly:
```javascript
$('When Executed by Another Workflow').first().json.text
```

---

## 6. Execution Metadata

### Parent Workflows vs Sub-Workflows

| Aspect | Parent Workflow | Sub-Workflow |
|--------|-----------------|--------------|
| Codika Init node | Required after trigger | **Not present** |
| Execution context | Auto-available | Must be passed explicitly |
| Codika nodes | Auto-retrieve credentials | Require override parameters |

### How Execution Context Works

**Parent workflows** have a Codika Init node immediately after the trigger. This node outputs:
- `executionId` - Unique execution identifier
- `executionSecret` - Authentication secret for API calls

Codika nodes (Upload File, Submit Result) automatically retrieve these values from the execution context.

**Sub-workflows** do not have a Codika Init node. The execution context is not available automatically.

### When Execution Metadata is Needed

Pass execution metadata to a sub-workflow **only when** the sub-workflow uses Codika nodes:
- `Codika Upload File` - Requires `executionIdOverride` and `executionSecretOverride`
- `Codika Submit Result` (if called from sub-workflow)

Sub-workflows that do not use Codika nodes do not need execution metadata.

### Passing Execution Metadata from Parent

In the parent workflow's Execute Workflow node, include `executionId` and `executionSecret` in the inputs:

```json
{
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "={{SUBWKFL_upload-subworkflow_LFKWBUS}}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "fileUrl": "={{ $json.fileUrl }}",
        "fieldKey": "generated_video",
        "fileName": "output.mp4",
        "executionId": "={{ $('Codika Init').first().json.executionId }}",
        "executionSecret": "={{ $('Codika Init').first().json.executionSecret }}"
      }
    },
    "options": {
      "waitForSubWorkflow": true
    }
  },
  "name": "Upload File Subworkflow",
  "type": "n8n-nodes-base.executeWorkflow",
  "typeVersion": 1.3
}
```

### Using Execution Metadata in Sub-Workflow

The sub-workflow trigger must declare these inputs:

```json
{
  "parameters": {
    "workflowInputs": {
      "values": [
        { "name": "fileUrl" },
        { "name": "fieldKey" },
        { "name": "fileName" },
        { "name": "executionId" },
        { "name": "executionSecret" }
      ]
    }
  },
  "name": "When Executed by Another Workflow",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "typeVersion": 1.1
}
```

Codika nodes use the override parameters:

```json
{
  "parameters": {
    "resource": "fileManagement",
    "operation": "uploadFile",
    "binaryPropertyName": "data",
    "fieldKey": "={{ $('When Executed by Another Workflow').first().json.fieldKey }}",
    "fileName": "={{ $('When Executed by Another Workflow').first().json.fileName }}",
    "executionIdOverride": "={{ $('When Executed by Another Workflow').first().json.executionId }}",
    "executionSecretOverride": "={{ $('When Executed by Another Workflow').first().json.executionSecret }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Upload File"
}
```

---

## 7. Deployment Order

The deployment system handles sub-workflow dependencies automatically:

1. Scans workflow JSON for `{{SUBWKFL_*_LFKWBUS}}` patterns
2. Builds a dependency graph of all workflows
3. Topologically sorts workflows (sub-workflows first, then dependents)
4. Deploys in order, replacing placeholders with actual n8n workflow IDs

**Circular dependencies are not allowed.** Deployment fails with an error message showing the cycle.

---

## 8. Complete Examples

### Example 1: Simple Sub-Workflow (No Codika Nodes)

A text transformation helper that does not need execution metadata.

**Sub-workflow config.ts:**

```typescript
{
  workflowTemplateId: 'text-transformer',
  workflowName: 'Text Transformer',
  triggers: [
    {
      triggerId: crypto.randomUUID(),
      type: 'subworkflow',
      title: 'Transform Text',
      description: 'Transforms text to uppercase and adds timestamp.',
      inputSchema: [
        { name: 'text', type: 'string' },
      ],
      calledBy: ['main-workflow'],
    }
  ],
  outputSchema: [],
  n8nWorkflowJsonBase64: transformerWorkflowBase64,
  cost: 0,
}
```

**Sub-workflow structure:**

```
When Executed by Another Workflow → Code (transform) → Return Output
```

**Code node:**

```javascript
const inputText = $input.item.json.text;

return [{
  json: {
    transformedText: inputText.toUpperCase(),
    processedAt: new Date().toISOString()
  }
}];
```

**Parent workflow call:**

```json
{
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "{{SUBWKFL_text-transformer_LFKWBUS}}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "text": "={{ $json.inputText }}"
      }
    },
    "options": {
      "waitForSubWorkflow": true
    }
  },
  "type": "n8n-nodes-base.executeWorkflow",
  "typeVersion": 1.3,
  "name": "Transform Text"
}
```

---

### Example 2: Sub-Workflow with File Upload (Execution Metadata Required)

A file upload helper that requires execution metadata.

**Sub-workflow config.ts:**

```typescript
{
  workflowTemplateId: 'file-uploader',
  workflowName: 'File Uploader',
  triggers: [
    {
      triggerId: crypto.randomUUID(),
      type: 'subworkflow',
      title: 'Upload File',
      description: 'Downloads a file from URL and uploads to Codika storage.',
      inputSchema: [
        { name: 'fileUrl', type: 'string' },
        { name: 'fieldKey', type: 'string' },
        { name: 'fileName', type: 'string' },
        { name: 'mimeType', type: 'string' },
        { name: 'executionId', type: 'string' },
        { name: 'executionSecret', type: 'string' },
      ],
      calledBy: ['video-generator', 'pdf-creator'],
    }
  ],
  outputSchema: [],
  n8nWorkflowJsonBase64: uploaderWorkflowBase64,
  cost: 0,
}
```

**Sub-workflow structure:**

```
When Executed by Another Workflow → Download File → Codika Upload File → Return Result
```

**Codika Upload File node (with override parameters):**

```json
{
  "parameters": {
    "resource": "fileManagement",
    "operation": "uploadFile",
    "binaryPropertyName": "data",
    "fieldKey": "={{ $('When Executed by Another Workflow').first().json.fieldKey }}",
    "fileName": "={{ $('When Executed by Another Workflow').first().json.fileName }}",
    "mimeType": "={{ $('When Executed by Another Workflow').first().json.mimeType }}",
    "executionIdOverride": "={{ $('When Executed by Another Workflow').first().json.executionId }}",
    "executionSecretOverride": "={{ $('When Executed by Another Workflow').first().json.executionSecret }}"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Upload File"
}
```

**Parent workflow call (passing execution metadata):**

```json
{
  "parameters": {
    "workflowId": {
      "__rl": true,
      "mode": "id",
      "value": "={{SUBWKFL_file-uploader_LFKWBUS}}"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "fileUrl": "={{ $json.videoUrl }}",
        "fieldKey": "generated_video",
        "fileName": "output.mp4",
        "mimeType": "video/mp4",
        "executionId": "={{ $('Codika Init').first().json.executionId }}",
        "executionSecret": "={{ $('Codika Init').first().json.executionSecret }}"
      }
    },
    "options": {
      "waitForSubWorkflow": true
    }
  },
  "name": "Upload Video File",
  "type": "n8n-nodes-base.executeWorkflow",
  "typeVersion": 1.3
}
```

---

## 9. Best Practices

1. **Use descriptive names**: Name sub-workflows clearly (e.g., `email-parser`, `data-validator`)
2. **Document inputs**: Use the `description` field in the subworkflow trigger to document expected input format
3. **Keep sub-workflows focused**: Each sub-workflow should do one thing well
4. **No cost for sub-workflows**: Set `cost: 0` for sub-workflows since the parent workflow pays
5. **Error handling**: Sub-workflow errors propagate to the parent workflow

---

## 10. Error Handling in Tool Workflows

### CRITICAL: Tool Workflows Must Return Errors Explicitly

When building sub-workflows used as **AI agent tools** (via `toolWorkflow`), you MUST ensure that validation failures and error conditions **always return an error response** to the calling agent. **Silent failures are unacceptable** - they cause agents to incorrectly assume operations succeeded.

### The Problem

If a tool workflow has an IF node with an empty false branch (`[]`), execution stops silently when validation fails:

```
❌ WRONG PATTERN:
Validate Input → IF (success?) ─┬→ TRUE: Continue to next step
                                └→ FALSE: [] (empty - silent failure!)
```

The AI agent calling this tool receives no response and will assume the operation succeeded.

### The Correct Pattern

Every IF node that checks for errors MUST have BOTH branches connected:

```
✅ CORRECT PATTERN:
Validate Input → IF (success?) ─┬→ TRUE: Continue to next step
                                └→ FALSE: Return Error Node → outputs { success: false, error: "message" }
```

### How to Implement

**1. Validation Code Node** - Return a structured object with `success` flag:

```javascript
// Validate inputs
const title = (input.title || '').trim();

if (!title) {
  return [{ json: { error: 'Title is required', success: false } }];
}

return [{ json: { title, success: true } }];
```

**2. IF Node** - Check the `success` field:

```json
{
  "conditions": {
    "conditions": [
      {
        "leftValue": "={{ $json.success }}",
        "rightValue": true,
        "operator": { "type": "boolean", "operation": "equals" }
      }
    ]
  }
}
```

**3. Error Return Node** - Always connect the FALSE branch to an error return node:

```javascript
// Return validation error to caller
const input = $('Validate Input').first().json;
return [{ json: input }];  // Returns the { error: '...', success: false } object
```

### Complete Connections Example

```json
"Validation Check?": {
  "main": [
    [{ "node": "Continue Processing", "type": "main", "index": 0 }],
    [{ "node": "Return Validation Error", "type": "main", "index": 0 }]
  ]
}
```

**NOT this:**

```json
"Validation Check?": {
  "main": [
    [{ "node": "Continue Processing", "type": "main", "index": 0 }],
    []  ← WRONG: Silent failure!
  ]
}
```

### Standard Error Response Format

Tool workflows should return errors in this format:

```json
{
  "success": false,
  "error": "Human-readable error message explaining what went wrong"
}
```

Success responses should include:

```json
{
  "success": true,
  "data": { /* result data */ }
}
```

### Checklist for Tool Workflows

- [ ] Every IF node has BOTH true and false branches connected (no empty `[]` arrays)
- [ ] Validation code nodes return `{ success: false, error: "message" }` for failures
- [ ] Error return nodes are connected to all false branches
- [ ] Error messages are descriptive and help the agent understand what went wrong

---

## 11. Input Parameter Requirements

**IMPORTANT:** Every subworkflow MUST have at least 1 input parameter.

The n8n platform enforces `minRequiredFields: 1` on the `ExecuteWorkflowTrigger` node. Subworkflows with zero parameters will fail with the error:

```
At least 1 field is required.
```

### Why This Requirement Exists

- **Explicit Interface**: Forces workflow authors to define clear input contracts
- **Documentation**: Makes it obvious what data flows between parent and child workflows
- **Validation**: Prevents silent data loss from unspecified inputs

### What If You Don't Need Input?

If your subworkflow truly needs no input, add a meaningful context parameter such as:

- `caller_role` - Useful for filtering/access control
- `execution_context` - Useful for logging/tracing
- `timestamp` - Useful for time-sensitive operations

**Bad Example (will fail):**
```json
"workflowInputs": {
  "values": []
}
```

**Good Example:**
```json
"workflowInputs": {
  "values": [
    { "name": "caller_role", "type": "string" }
  ]
}
```

---

## 12. Checklist

### Sub-workflow configuration (config.ts)

- [ ] `type: 'subworkflow'` trigger
- [ ] `inputSchema` array with **at least 1** `{ name, type }` entry (REQUIRED)
- [ ] `description` documenting the sub-workflow purpose
- [ ] `calledBy` array listing parent workflows (optional but recommended)
- [ ] `outputSchema: []` (required, empty)
- [ ] `cost: 0` (parent pays)

### Sub-workflow JSON

- [ ] Starts with `executeWorkflowTrigger` node
- [ ] `workflowInputs.values` has **at least 1 parameter** (REQUIRED - zero params will fail)
- [ ] Each parameter in `workflowInputs.values` has `{ name, type }`
- [ ] Returns data to parent (last node's output is returned)
- [ ] Has `errorWorkflow` setting
- [ ] If using Codika nodes: includes `executionId` and `executionSecret` in trigger inputs
- [ ] If using Codika nodes: uses `executionIdOverride` and `executionSecretOverride` parameters

### Parent workflow JSON

- [ ] Uses `executeWorkflow` node with `{{SUBWKFL_<TEMPLATE_ID>_LFKWBUS}}` placeholder
- [ ] `waitForSubWorkflow: true` if output is needed
- [ ] If sub-workflow uses Codika nodes: passes `executionId` and `executionSecret` in `workflowInputs`
- [ ] Handles sub-workflow output appropriately

---

## Related Documentation

- [File Upload Guide](../../.claude/docs/features/file_upload/FILE_UPLOAD_GUIDE.md) - Detailed file upload patterns including sub-workflow examples
