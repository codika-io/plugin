# HTTP Triggers Guide

HTTP triggers expose workflows to user-initiated execution via the Codika UI. Users fill out a form (defined by `inputSchema`), submit it, and see results (defined by `outputSchema`).

---

## 1. Required Workflow Pattern

Every HTTP-triggered workflow must follow this structure:

```
HTTP Trigger → Codika Init → [Business Logic] → Codika Submit Result
                                             └→ Codika Report Error (on failure)
```

**Requirements:**

1. **Codika Init** must be the first node after the HTTP Trigger
2. **Codika Submit Result** or **Codika Report Error** must be the final node on every execution path
3. No execution path should end without calling one of these completion nodes

### Codika Init Node Configuration

```json
{
  "parameters": {
    "resource": "initializeExecution",
    "operation": "initWorkflow",
    "memberSecret": "{{MEMSECRT_EXECUTION_AUTH_TRCESMEM}}",
    "organizationId": "={{$json.body.executionMetadata.organizationId}}",
    "userId": "={{$json.body.executionMetadata.userId}}",
    "processInstanceId": "={{$json.body.executionMetadata.processInstanceId}}",
    "workflowId": "your-workflow-id",
    "triggerType": "http"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Init"
}
```

### Codika Submit Result Node Configuration

```json
{
  "parameters": {
    "resource": "workflowOutputs",
    "operation": "submitResult",
    "resultData": "={ \"field_key\": \"value\", \"another_field\": 123 }"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Submit Result"
}
```

### Codika Report Error Node Configuration

```json
{
  "parameters": {
    "resource": "errorHandling",
    "operation": "reportError",
    "errorMessage": "Description of what went wrong",
    "errorType": "validation_error"
  },
  "type": "n8n-nodes-codika.codika",
  "typeVersion": 1,
  "name": "Codika Report Error"
}
```

---

## 2. HTTP Trigger Configuration in config.ts

Define HTTP triggers in your workflow configuration using the `HttpTrigger` type.

### HttpTrigger Structure

```typescript
{
  triggerId: crypto.randomUUID(),  // Unique identifier
  type: 'http',
  url: webhookUrl,                 // URL with placeholders
  method: 'POST',                  // HTTP method
  title: 'Trigger Title',          // Display name in UI
  description: 'What this trigger does',
  inputSchema: getInputSchema(),   // Form fields (see Section 3)
} satisfies HttpTrigger
```

### URL Path Pattern

The webhook URL must use placeholders that get replaced at deployment:

```typescript
const webhookUrl = `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}/webhook/{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-endpoint`;
```

| Placeholder | Description |
|-------------|-------------|
| `{{ORGSECRET_N8N_BASE_URL_TERCESORG}}` | Organization's n8n base URL |
| `{{PROCDATA_PROCESS_ID_ATADCORP}}` | Process ID |
| `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}` | User's process instance ID |

The final segment (`your-endpoint`) is your custom endpoint name.

### HTTP Trigger Node in Workflow

The n8n Webhook node must match the URL pattern:

```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/your-endpoint",
    "responseMode": "lastNode"
  },
  "name": "HTTP Trigger",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2
}
```

**Important:** `responseMode` must be `"lastNode"`. This ensures the workflow completes before responding, allowing Codika Submit Result to properly return data.

---

## 3. Input Schema Definition

The `inputSchema` defines the form fields users see when triggering the workflow.

All field properties are defined at the top level of each field object.

### Field Type Reference

| Type | UI Component | Value Type | Description |
|------|--------------|------------|-------------|
| `string` | Text input | `string` | Single-line text field |
| `text` | Textarea | `string` | Multi-line text field with configurable rows |
| `number` | Number input | `number` | Numeric input (integer or double) |
| `boolean` | Checkbox/Switch | `boolean` | True/false toggle |
| `date` | Date picker | `string` (ISO) | Calendar date selector |
| `select` | Dropdown | `string` | Single selection from options list |
| `multiselect` | Checkboxes list | `string[]` | Multiple selections from options |
| `radio` | Radio buttons | `string` | Single selection with visible options |
| `file` | File upload | `string` (file ID) | Upload with optional preview |
| `array` | Repeatable input | `T[]` | List of primitive items |
| `object` | Nested form | `object` | Single structured object with nested fields |
| `objectArray` | Repeatable objects | `object[]` | Array of structured objects |

### Basic Structure

```typescript
function getInputSchema(): FormInputSchema {
  return [
    {
      type: 'section',
      title: 'Section Title',
      description: 'Optional section description',
      collapsible: true,       // Enable collapse/expand
      defaultOpen: true,       // Initial state when collapsible
      inputSchema: [
        // Fields go here
      ],
    },
  ];
}
```

---

### Complete Field Type Reference (All Parameters)

#### 3.1 String Field

Single-line text input.

```typescript
{
  // === Required Properties ===
  key: 'user_name',           // Unique field identifier
  type: 'string',             // Field type
  label: 'User Name',         // Display label

  // === Optional Properties ===
  description: 'Enter the user name',      // Help text below field
  tooltip: 'This will be used for display', // Hover tooltip
  placeholder: 'John Doe',                  // Placeholder text
  required: true,                           // Whether field is required

  // === String-Specific Properties ===
  defaultValue: '',           // Default value
  minLength: 2,               // Minimum character length
  maxLength: 100,             // Maximum character length
  regex: '^[a-zA-Z ]+$',      // Validation regex pattern
  regexError: 'Only letters and spaces allowed', // Custom regex error message
}
```

**Value Type:** `string`

---

#### 3.2 Text Field (Textarea)

Multi-line text input for longer content.

```typescript
{
  // === Required Properties ===
  key: 'description',
  type: 'text',
  label: 'Description',

  // === Optional Properties ===
  description: 'Enter a detailed description',
  tooltip: 'Provide as much detail as needed',
  placeholder: 'Describe your request...',
  required: false,

  // === Text-Specific Properties ===
  defaultValue: '',           // Default value
  minLength: 10,              // Minimum character length
  maxLength: 5000,            // Maximum character length
  rows: 6,                    // Number of visible rows (default: 4)
  regex: '',                  // Validation regex pattern
  regexError: '',             // Custom regex error message
}
```

**Value Type:** `string`

---

#### 3.3 Number Field

Numeric input with optional constraints.

```typescript
{
  // === Required Properties ===
  key: 'quantity',
  type: 'number',
  label: 'Quantity',

  // === Optional Properties ===
  description: 'Number of items to process',
  tooltip: 'Enter a whole number',
  placeholder: 'Enter quantity',
  required: true,

  // === Number-Specific Properties ===
  defaultValue: 10,           // Default value
  numberType: 'integer',      // 'integer' or 'double'
  min: 1,                     // Minimum value
  max: 100,                   // Maximum value
  step: 1,                    // Increment step (e.g., 0.1 for decimals)
}
```

**Value Type:** `number`

---

#### 3.4 Boolean Field

Toggle/checkbox for true/false values.

```typescript
{
  // === Required Properties ===
  key: 'is_urgent',
  type: 'boolean',
  label: 'Urgent Request',

  // === Optional Properties ===
  description: 'Check if this request is urgent',
  tooltip: 'Urgent requests are prioritized',
  required: false,

  // === Boolean-Specific Properties ===
  defaultValue: false,        // Default value (true or false)
  enabledLabel: 'Yes',        // Label when checked (default: "Enabled")
  disabledLabel: 'No',        // Label when unchecked (default: "Disabled")
}
```

**Value Type:** `boolean`

---

#### 3.5 Date Field

Date picker for calendar selection.

```typescript
{
  // === Required Properties ===
  key: 'due_date',
  type: 'date',
  label: 'Due Date',

  // === Optional Properties ===
  description: 'Select the due date',
  tooltip: 'Choose a date within the allowed range',
  required: true,

  // === Date-Specific Properties ===
  defaultValue: '2025-01-15', // Default value (ISO date string: YYYY-MM-DD)
  minDate: '2024-01-01',      // Minimum selectable date (ISO string)
  maxDate: '2025-12-31',      // Maximum selectable date (ISO string)
}
```

**Value Type:** `string` (ISO date format: `"2025-01-15"`)

---

#### 3.6 Select Field (Dropdown)

Single selection from a dropdown list.

```typescript
{
  // === Required Properties ===
  key: 'priority',
  type: 'select',
  label: 'Priority',
  options: [                  // REQUIRED: Array of options
    { value: 'low', label: 'Low Priority' },
    { value: 'medium', label: 'Medium Priority' },
    { value: 'high', label: 'High Priority' },
  ],

  // === Optional Properties ===
  description: 'Select the priority level',
  tooltip: 'Higher priority items are processed first',
  placeholder: 'Select priority...',
  required: true,

  // === Select-Specific Properties ===
  defaultValue: 'medium',     // Default selected value (must match an option value)
}
```

**Value Type:** `string` (the selected option's `value`)

---

#### 3.7 Multiselect Field

Multiple selection from checkboxes/tags.

```typescript
{
  // === Required Properties ===
  key: 'categories',
  type: 'multiselect',
  label: 'Categories',
  options: [                  // REQUIRED: Array of options
    { value: 'tech', label: 'Technology' },
    { value: 'finance', label: 'Finance' },
    { value: 'marketing', label: 'Marketing' },
    { value: 'sales', label: 'Sales' },
  ],

  // === Optional Properties ===
  description: 'Select all applicable categories',
  tooltip: 'You can select multiple options',
  placeholder: 'Select categories...',
  required: false,

  // === Multiselect-Specific Properties ===
  defaultValue: ['tech'],     // Default selected values (array of option values)
  minSelections: 1,           // Minimum required selections
  maxSelections: 3,           // Maximum allowed selections
}
```

**Value Type:** `string[]` (array of selected option values: `["tech", "finance"]`)

---

#### 3.8 Radio Field

Single selection with all options visible.

```typescript
{
  // === Required Properties ===
  key: 'contact_method',
  type: 'radio',
  label: 'Preferred Contact Method',
  options: [                  // REQUIRED: Array of options
    { value: 'email', label: 'Email' },
    { value: 'phone', label: 'Phone' },
    { value: 'sms', label: 'SMS' },
  ],

  // === Optional Properties ===
  description: 'How should we contact you?',
  tooltip: 'Select one option',
  required: true,

  // === Radio-Specific Properties ===
  defaultValue: 'email',      // Default selected value
}
```

**Value Type:** `string` (the selected option's `value`)

---

#### 3.9 File Field

File upload with optional preview.

```typescript
{
  // === Required Properties ===
  key: 'document',
  type: 'file',
  label: 'Upload Document',

  // === Optional Properties ===
  description: 'Upload a PDF or image file',
  tooltip: 'Maximum file size: 10MB',
  required: false,

  // === File-Specific Properties ===
  maxSize: 10 * 1024 * 1024,  // Maximum file size in bytes (10MB)
  allowedMimeTypes: [         // Allowed MIME types
    'application/pdf',
    'image/png',
    'image/jpeg',
    'image/*',                // Wildcard for all images
  ],
  showPreview: true,          // Show image preview after upload
  tags: ['input-document'],   // Tags for document organization
}
```

**Value Type:** Object with file metadata (see Section 6 for details)

---

#### 3.10 Array Field

Repeatable list of primitive values.

```typescript
{
  // === Required Properties ===
  key: 'email_list',
  type: 'array',
  label: 'Email Addresses',
  itemField: {                // REQUIRED: Full field definition for each item
    type: 'string',
    minLength: 5,
    maxLength: 100,
    regex: '^[^@]+@[^@]+\\.[^@]+$',
    regexError: 'Invalid email format',
    placeholder: 'Enter email address',
  },

  // === Optional Properties ===
  description: 'Add email addresses to notify',
  tooltip: 'Click + to add more emails',
  required: true,

  // === Array-Specific Properties ===
  defaultValue: ['admin@example.com'], // Default array values
  minItems: 1,                // Minimum number of items
  maxItems: 10,               // Maximum number of items
}
```

**The `itemField` property** defines the complete field configuration for each array item. It supports 7 primitive field types with their full property set:

| Type | Available Properties |
|------|---------------------|
| `string` | `defaultValue`, `minLength`, `maxLength`, `regex`, `regexError`, `placeholder` |
| `text` | `defaultValue`, `minLength`, `maxLength`, `rows`, `regex`, `regexError`, `placeholder` |
| `number` | `defaultValue`, `min`, `max`, `step`, `numberType`, `placeholder` |
| `boolean` | `defaultValue`, `enabledLabel`, `disabledLabel` |
| `date` | `defaultValue`, `minDate`, `maxDate` |
| `select` | `defaultValue`, `options` (required), `placeholder` |
| `radio` | `defaultValue`, `options` (required) |

> **Note:** `multiselect` is NOT supported as an array item type because it produces nested arrays (array of arrays), which Firestore cannot store.

**Example with text items:**

```typescript
{
  key: 'notes',
  type: 'array',
  label: 'Notes',
  itemField: {
    type: 'text',
    rows: 4,                  // Now you can specify rows!
    maxLength: 1000,
    placeholder: 'Enter a note...',
  },
  minItems: 0,
  maxItems: 5,
}
```

**Value Type:** Array of the item type (e.g., `string[]`, `number[]`)

---

#### 3.11 Object Field

Single structured object with nested fields.

```typescript
{
  // === Required Properties ===
  key: 'contact_info',
  type: 'object',
  label: 'Contact Information',
  schema: [                   // REQUIRED: Nested field definitions
    {
      key: 'name',
      type: 'string',
      label: 'Full Name',
      required: true,
      maxLength: 100,
    },
    {
      key: 'email',
      type: 'string',
      label: 'Email',
      required: true,
      regex: '^[^@]+@[^@]+\\.[^@]+$',
      regexError: 'Invalid email',
    },
    {
      key: 'phone',
      type: 'string',
      label: 'Phone',
      required: false,
    },
  ],

  // === Optional Properties ===
  description: 'Enter contact details',
  tooltip: 'All contact information for this person',
  required: true,

  // === Object-Specific Properties ===
  defaultValue: {             // Default object value
    name: '',
    email: '',
    phone: '',
  },
}
```

**Value Type:** `object` (e.g., `{ name: "John", email: "john@example.com", phone: "" }`)

---

#### 3.12 ObjectArray Field

Array of structured objects (repeatable forms).

```typescript
{
  // === Required Properties ===
  key: 'team_members',
  type: 'objectArray',
  label: 'Team Members',
  itemSchema: [               // REQUIRED: Schema for each object
    {
      key: 'name',
      type: 'string',
      label: 'Name',
      required: true,
      maxLength: 100,
    },
    {
      key: 'role',
      type: 'select',
      label: 'Role',
      required: true,
      options: [
        { value: 'developer', label: 'Developer' },
        { value: 'designer', label: 'Designer' },
        { value: 'manager', label: 'Manager' },
      ],
    },
    {
      key: 'email',
      type: 'string',
      label: 'Email',
      required: true,
    },
    {
      key: 'is_lead',
      type: 'boolean',
      label: 'Team Lead',
      defaultValue: false,
    },
  ],

  // === Optional Properties ===
  description: 'Add team members for this project',
  tooltip: 'Click + to add more members',
  required: true,

  // === ObjectArray-Specific Properties ===
  defaultValue: [             // Default array of objects
    {
      name: '',
      role: 'developer',
      email: '',
      is_lead: false,
    },
  ],
  minItems: 1,                // Minimum number of items
  maxItems: 10,               // Maximum number of items
}
```

**Value Type:** `object[]` (array of objects matching the schema)

---

### Section Configuration

Sections group related fields visually.

```typescript
{
  type: 'section',
  title: 'Basic Information',           // Section header
  description: 'Enter basic details',   // Optional description
  collapsible: true,                    // Enable collapse/expand
  defaultOpen: true,                    // Initial state when collapsible
  inputSchema: [
    // Fields go here...
  ],
}
```

---

## 4. Output Schema Definition

The `outputSchema` defines what results are displayed to users after workflow completion. Field properties are defined at the top level of each field object.

```typescript
function getOutputSchema(): FormOutputSchema {
  return [
    {
      key: 'status',
      type: 'string',
      label: 'Status',
      description: 'Operation status',
    },
    {
      key: 'result_text',
      type: 'text',
      label: 'Result',
      description: 'The processed output',
    },
    {
      key: 'items_processed',
      type: 'number',
      label: 'Items Processed',
      description: 'Number of items processed',
    },
    {
      key: 'success',
      type: 'boolean',
      label: 'Success',
      description: 'Whether the operation succeeded',
    },
    {
      key: 'output_file',
      type: 'file',
      label: 'Generated File',
      description: 'The generated output file',
    },
  ];
}
```

The output schema uses the same field types as the input schema. The workflow's `Codika Submit Result` node must return data matching these field keys.

---

## 5. Accessing Input Data in Workflows

### The executionMetadata Object

Every HTTP trigger payload includes `executionMetadata`:

```typescript
{
  executionMetadata: {
    executionId: string,
    executionSecret: string,
    callbackUrl: string,
    workflowId: string,
    processId: string,
    processInstanceId: string,
    userId: string,
    organizationId: string
  }
}
```

### Accessing Form Data

The Codika Init node does **not** pass through input data. Reference the HTTP Trigger node directly:

```javascript
// Get execution data from Codika Init
const initData = $('Codika Init').first().json;
const executionId = initData.executionId;
const startTimeMs = initData._startTimeMs;

// Get form data from HTTP Trigger
const triggerData = $('HTTP Trigger').first().json;
const formField = triggerData.body.field_key;
```

> **COMMON MISTAKE:** Form fields are stored directly in `body`, NOT in `body.inputData`.
>
> ```javascript
> // WRONG - will return undefined!
> const inputData = webhookData.body?.inputData || {};
> const value = inputData.my_field; // undefined
>
> // CORRECT - access fields directly from body
> const inputData = webhookData.body || {};
> const value = inputData.my_field; // works!
> ```

---

## 6. Input File Handling

Files uploaded via the form are enriched by the backend before reaching the workflow.

### Available File Properties

```javascript
const fileData = $('HTTP Trigger').first().json.body.document;
// {
//   downloadUrl: 'https://storage.googleapis.com/...?token=...',  // Signed URL
//   fileName: 'document.pdf',
//   fileType: 'pdf',
//   fileSize: 1024000,
//   mimeType: 'application/pdf',
//   text: '...'  // Parsed text content (for PDFs and documents)
// }
```

### Accessing Parsed Text

For documents with parsed text (PDFs):

```javascript
const parsedContent = triggerData.body.document?.text || '';
```

### Downloading Binary Files

For files that need binary processing (images, videos), use an HTTP Request node:

```text
HTTP Trigger → Codika Init → HTTP Request (GET downloadUrl) → Process Binary → ...
```

HTTP Request node configuration:

```json
{
  "parameters": {
    "url": "={{ $('HTTP Trigger').first().json.body.image.downloadUrl }}",
    "options": {}
  },
  "name": "Download Image",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4
}
```

---

## 7. Output File Handling

Output files (files generated by the workflow and returned to users) use a separate pattern.

> **📖 See [Output Files Guide](./OUTPUT_FILES_GUIDE.md)** for documentation on:
> - Codika Upload File node configuration
> - Returning files in workflow results
> - File storage and signed URLs

---

## 8. Complete Example

### config.ts

```typescript
function getInputSchema(): FormInputSchema {
  return [
    {
      type: 'section',
      title: 'Analysis Request',
      description: 'Configure your analysis',
      collapsible: false,
      inputSchema: [
        {
          key: 'input_text',
          type: 'text',
          label: 'Text to Analyze',
          description: 'Paste the text you want to analyze',
          placeholder: 'Enter text...',
          required: true,
          maxLength: 10000,
          rows: 6,
        },
        {
          key: 'analysis_type',
          type: 'select',
          label: 'Analysis Type',
          description: 'What type of analysis?',
          placeholder: 'Select type...',
          required: true,
          options: [
            { value: 'sentiment', label: 'Sentiment Analysis' },
            { value: 'summary', label: 'Summarization' },
            { value: 'keywords', label: 'Keyword Extraction' },
          ],
        },
        {
          key: 'include_metadata',
          type: 'boolean',
          label: 'Include Metadata',
          description: 'Include additional analysis metadata',
          defaultValue: false,
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
      label: 'Analysis Result',
      description: 'The analysis output',
    },
    {
      key: 'confidence',
      type: 'number',
      label: 'Confidence Score',
      description: 'Confidence of the analysis (0-1)',
    },
    {
      key: 'keywords',
      type: 'array',
      label: 'Keywords',
      description: 'Extracted keywords',
      itemField: { type: 'string' },
    },
  ];
}
```

### Workflow Structure

```text
HTTP Trigger → Codika Init → Get Input → AI Analysis → IF Success
                                                          ├── TRUE → Codika Submit Result
                                                          └── FALSE → Codika Report Error
```

### HTTP Trigger Configuration

```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "{{PROCDATA_PROCESS_ID_ATADCORP}}/{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}/analyze",
    "responseMode": "lastNode"
  },
  "name": "HTTP Trigger",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2
}
```

---

## 9. Calling HTTP Workflows Programmatically (from Dashboard or External Code)

HTTP-triggered workflows can also be called from application code (e.g., a dashboard backend) using the Public API.

### Payload Format

**Recommended:** Wrap user data in a `payload` key. The Codika trigger system extracts the `payload` fields and places them in `body` alongside `executionMetadata`.

```
POST {triggerBaseUrl}/{processInstanceId}/{workflowEndpoint}
Header: X-API-Key: ck_...
Header: Content-Type: application/json
Body: { "payload": { "my_field": "value", "another_field": 123 } }
```

**Raw body is also supported:** If no `payload` key is present, the entire request body is passed through as-is. This is useful for third-party webhooks (e.g., Evolution API, Stripe) that send their own payload format and cannot be configured to wrap data in a `payload` key.

```
// Third-party webhook sending raw body — works fine
Body: { "event": "messages.upsert", "instance": "bot", "data": { ... } }
```

Inside the workflow, access the data from the Webhook Trigger node:

```javascript
const input = $('Webhook Trigger').first().json;
const body = input.body || input;
const myField = body.my_field;      // "value" (wrapped format)
const event = body.event;           // "messages.upsert" (raw format)
```

### Trigger + Poll Pattern

The public API is asynchronous. The trigger returns an `executionId`, which you poll for completion:

```typescript
// 1. Trigger
const triggerRes = await fetch(`${TRIGGER_BASE_URL}/${workflowEndpoint}`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': API_KEY
  },
  body: JSON.stringify({
    payload: { phone_number: '32477123456', name: 'John' }
  })
});
const { executionId } = await triggerRes.json();

// 2. Poll for result
let result;
while (!result) {
  await new Promise(r => setTimeout(r, 1500));
  const statusRes = await fetch(`${STATUS_BASE_URL}/${executionId}`, {
    headers: { 'X-API-Key': API_KEY }
  });
  const { execution } = await statusRes.json();
  if (execution.status === 'success') result = execution.resultData;
  if (execution.status === 'failed') throw new Error(execution.errorDetails?.message);
}
```

### Best Practice

```javascript
// RECOMMENDED - fields wrapped in payload key (explicit, clean separation)
body: JSON.stringify({ payload: { phone_number: '123', name: 'John' } })

// ALSO WORKS - raw body (used by third-party webhooks that can't wrap in payload)
body: JSON.stringify({ phone_number: '123', name: 'John' })
```

> **Note:** When using the `payload` wrapper, only the contents of `payload` are passed to the workflow. When sending a raw body, the entire body (all top-level keys) is passed through. The `payload` wrapper is recommended for Codika-controlled callers to keep the format explicit.

---

## 10. Async API Calls with Callback (Fire-and-Wait Pattern)

When a workflow calls an external API that processes asynchronously and reports back via a callback URL, use the **fire-and-wait** pattern: HTTP Request → Wait for Webhook → process result.

### Pattern

```
[Previous Logic] → HTTP Request (fire) → Wait (webhook callback) → [Process Result]
```

The HTTP Request node sends the request (including a `callback_url` from `$execution.resumeUrl`), the API returns immediately (e.g., 200 with a `job_id`), and the Wait node pauses execution until the callback arrives.

### CRITICAL: Never use retryOnFail on fire-and-wait HTTP Request nodes

**`retryOnFail` must be `false` (or omitted) on the HTTP Request node that triggers the async API.** This is a hard rule with no exceptions.

**Why:** Even though n8n retries are designed for transient failures, they cause catastrophic issues with async APIs:

1. The API accepts the request and returns 200 — but if n8n considers anything about the response an error (timeout, unexpected format, network blip), it retries
2. Each retry spawns a **new** async job on the API side
3. All jobs run in parallel, consuming resources (API credits, compute) for duplicate work
4. Only one callback will match the Wait node — the others are wasted

**Real-world cost:** A single presentation generation with 3 retries at 5-second intervals spawns 3 parallel Claude agent sessions (~$1-3 each), tripling the cost per execution.

### Correct Configuration

```json
{
  "parameters": {
    "method": "POST",
    "url": "https://your-async-api.example.com/v1/generate",
    "sendBody": true,
    "contentType": "multipart-form-data",
    "bodyParameters": {
      "parameters": [
        {
          "parameterType": "formData",
          "name": "callback_url",
          "value": "={{ $execution.resumeUrl }}/my-callback"
        }
      ]
    },
    "options": {
      "timeout": 30000
    }
  },
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "onError": "continueErrorOutput"
}
```

Note: **No `retryOnFail`, `maxTries`, or `waitBetweenTries` properties.** The `onError: "continueErrorOutput"` routes failures to an error handler instead of retrying.

### Wait Node Configuration

```json
{
  "parameters": {
    "resume": "webhook",
    "httpMethod": "POST",
    "options": {
      "webhookSuffix": "my-callback",
      "limitWaitTime": true,
      "limitType": "afterTimeInterval",
      "amount": 20,
      "unit": "minutes"
    }
  },
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1
}
```

The `webhookSuffix` must match the suffix appended to `$execution.resumeUrl` in the HTTP Request node.

### When retryOnFail IS appropriate

Retries are safe on HTTP Request nodes that are **synchronous and idempotent**:

- Embedding API calls (OpenAI, Cohere) — same input produces same output
- GET requests to download files — no side effects
- Status polling endpoints — read-only
- LLM chain nodes (`chainLlm`) — see [ai-nodes.md](./ai-nodes.md)

Retries are **dangerous** on:

- Any API call that triggers a background job or async process
- Any API call that creates resources (records, files, sessions) as a side effect
- Any API call paired with a Wait node for callback

---

## 11. Checklist

Before deploying an HTTP-triggered workflow:

- [ ] HTTP Trigger node with correct path pattern
- [ ] Codika Init immediately after trigger with `triggerType: "http"`
- [ ] All execution paths end with Codika Submit Result or Codika Report Error
- [ ] Input schema field keys match what the workflow expects
- [ ] Output schema field keys match what Codika Submit Result returns
- [ ] File fields properly configured with `allowedMimeTypes` and `maxSize`
- [ ] Required fields marked with `required: true`
- [ ] HTTP Request nodes calling async APIs have `retryOnFail` disabled (or omitted)
- [ ] Fire-and-wait patterns use `onError: "continueErrorOutput"` instead of retries
