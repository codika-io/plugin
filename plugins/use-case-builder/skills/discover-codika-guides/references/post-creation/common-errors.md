# Common n8n Workflow Errors

> This document catalogs common errors encountered during n8n workflow development on the Codika platform.
> Use this reference to avoid repeating mistakes and debug issues faster.

---

## Table of Contents

1. [Code Node Limitations](#1-code-node-limitations)
2. [Credential Configuration](#2-credential-configuration)
3. [API Integration Issues](#3-api-integration-issues)
4. [Node Reference Errors](#4-node-reference-errors)
5. [HTTP Request Node Issues](#5-http-request-node-issues)
6. [Execution & Testing Issues](#6-execution--testing-issues)
7. [Quick Reference - Error Codes](#7-quick-reference---error-codes)

---

## 1. Code Node Limitations

### Error: `Module 'crypto' is disallowed`

**Symptom:**
```
Module 'crypto' is disallowed [line X]
```

**Cause:**
n8n Code nodes block the Node.js `crypto` module for security reasons.

**Solution:**
- ❌ **Don't use:** `const crypto = require('crypto');`
- ✅ **Use instead:**
  - HTTP Request node with external API that handles crypto operations
  - Cloud Function for JWT generation, encryption, etc.
  - n8n's built-in credential system for authentication

**Example:**
```javascript
// ❌ WRONG - Will fail
const crypto = require('crypto');
const hash = crypto.createHash('sha256');

// ✅ CORRECT - Use Cloud Function
const response = await $http.request({
  url: 'https://your-domain.cloudfunctions.net/generateHash',
  method: 'POST',
  body: { data: 'string to hash' }
});
```

---

### Error: `fetch is not defined`

**Symptom:**
```
fetch is not defined [line X]
ReferenceError: fetch is not defined
```

**Cause:**
The global `fetch` API is not available in n8n Code nodes.

**Solution:**
- ❌ **Don't use:** `fetch(url, options)`
- ✅ **Use instead:**
  - HTTP Request node (preferred for most cases)
  - `$http.request()` helper (if available in your n8n version)
  - Restructure to use n8n's native nodes for HTTP calls

**Example:**
```javascript
// ❌ WRONG - fetch not available
const response = await fetch(pollUrl, {
  method: 'GET',
  headers: { 'Authorization': `Bearer ${token}` }
});

// ✅ CORRECT - Use HTTP Request node instead
// Replace Code node with HTTP Request node for API calls
```

**Note:** For polling/retry logic, use n8n's Loop node + HTTP Request instead of fetch in Code.

---

### Error: `$http is not defined`

**Symptom:**
```
$http is not defined [line X]
ReferenceError: $http is not defined
```

**Cause:**
The `$http` helper is not available in all n8n versions or execution contexts.

**Solution:**
- ❌ **Don't use:** `$http.request()` in Code nodes
- ✅ **Use instead:**
  - HTTP Request node (most reliable)
  - External API calls via Cloud Functions
  - Restructure workflow to use native n8n nodes

**Best Practice:**
Avoid complex HTTP operations in Code nodes. Use HTTP Request nodes for all external API calls.

---

### Error: `URL is not defined`

**Symptom:**
```
ReferenceError: URL is not defined at VmCodeWrapper
```

**Cause:**
The JavaScript `URL` class is not available in n8n's sandboxed Code node environment.

**Solution:**
Use manual string parsing instead of the URL class:

```javascript
// ❌ WRONG - URL class not available
const url = new URL(notionUrl);
const blockId = url.pathname.split('/').pop();

// ✅ CORRECT - Manual string parsing
let cleanUrl = notionUrl.split('?')[0].split('#')[0];
let pathPart = cleanUrl.replace(/^https?:\/\/[^\/]+/, '');
let pathParts = pathPart.split('/').filter(p => p);
const lastPart = pathParts[pathParts.length - 1];

// Extract ID using regex
const idMatch = lastPart.match(/[a-f0-9]{32}$/i);
const blockId = idMatch ? idMatch[0] : lastPart.replace(/-/g, '');
```

**Common use cases:**
- Parsing Notion page URLs to extract block IDs
- Extracting query parameters from callback URLs
- Parsing API endpoint URLs

---

### Error: `this.helpers is not defined` / HTTP in Code nodes

**Symptom:**
```
TypeError: Cannot read properties of undefined (reading 'httpRequestWithAuthentication')
```

Or HTTP calls inside Code nodes silently fail (try/catch swallows errors).

**Cause:**
The `this.helpers.httpRequestWithAuthentication` method is not available in n8n Code node sandbox.

**Solution:**
Use separate HTTP Request nodes instead of making HTTP calls inside Code nodes:

```
❌ WRONG - HTTP inside Code node:
┌──────────────────────────────────────────┐
│ Code Node                                │
│ for (const item of items) {              │
│   await this.helpers.httpRequest(...)    │  ← Fails silently
│ }                                        │
└──────────────────────────────────────────┘

✅ CORRECT - Use n8n's item processing:
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Prepare Items   │ ──► │  HTTP Request    │ ──► │ Combine Results  │
│ (outputs many)  │     │ (runs per item)  │     │ (aggregates)     │
└─────────────────┘     └──────────────────┘     └──────────────────┘
```

**Pattern for looping HTTP calls:**

1. **Code node** - Output multiple items (one per API call needed):
```javascript
const items = data.filter(item => item.needsFetch);
return items.map(item => ({
  json: { id: item.id, url: item.url }
}));
```

2. **HTTP Request node** - n8n automatically runs it for each item:
```json
{
  "url": "={{ $json.url }}",
  "method": "GET"
}
```

3. **Code node** - Combine all results:
```javascript
const allItems = $input.all();
const combined = allItems.map(item => item.json.data);
return [{ json: { results: combined } }];
```

---

## 2. Credential Configuration

### Error: `Credential with ID "..." does not exist for type "falAiApi"`

**Symptom:**
```
Credential with ID "NMqchpzEHmKXKGON" does not exist for type "falAiApi"
```

**Cause:**
Using a custom credential type that doesn't exist in n8n.

**Solution:**
For APIs without dedicated n8n credential types (like Fal.ai), use **generic credential types**:

- ✅ **Use:** `httpHeaderAuth` for API key in header
- ✅ **Use:** `httpBasicAuth` for username/password
- ✅ **Use:** `httpDigestAuth` for digest authentication
- ✅ **Use:** `oAuth2Api` for OAuth 2.0

**Example (Fal.ai):**
```json
{
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "httpHeaderAuth",
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"
    }
  }
}
```

**Reference:** See INTEGRATIONS_REFERENCE.md for all supported credential types.

---

### Error: `Header name must be a valid HTTP token`

**Symptom:**
```
Header name must be a valid HTTP token ["[Codika] fal AI API"]
```

**Cause:**
The Header Auth credential is configured with an invalid header name (contains brackets, spaces, or special characters).

**Solution:**
In the n8n Header Auth credential configuration:

- ❌ **Wrong:** Header Name = `[Codika] fal AI API` (the credential name)
- ✅ **Correct:** Header Name = `Authorization` (the actual HTTP header)

**Header Auth Configuration:**
```
Credential Name: [Codika] fal AI API  (display name, can be anything)
Header Name: Authorization               (must be valid HTTP header name)
Header Value: Key YOUR_API_KEY           (the actual credential value)
```

**Valid HTTP header names:**
- Only alphanumeric, hyphens, and underscores
- Examples: `Authorization`, `X-API-Key`, `X-Auth-Token`
- NOT: `[Codika] API`, `My Auth Header`, `Auth (Key)`

---

## 3. API Integration Issues

### Error: `Application 'model-name' not found` (404)

**Symptom:**
```
404 - "Application 'hedra-v1-image-to-video' not found"
```

**Cause:**
Incorrect model name or endpoint path for the API.

**Solution:**
1. **Verify the exact model name** from the API documentation
2. **Check the endpoint structure:**
   - Fal.ai uses: `fal-ai/model-name` (not `fal.ai/model-name`)
   - Some models use different paths: `veo3.1/reference-to-video` vs `veo3.1-reference-to-video`

3. **Test the endpoint** with curl first:
   ```bash
   curl https://fal.run/fal-ai/veo3.1/reference-to-video \
     -H "Authorization: Key YOUR_KEY" \
     -H "Content-Type: application/json" \
     -d '{"prompt":"test","image_urls":["url"],"duration":"8s"}'
   ```

**Example Mistake:**
```
❌ https://queue.fal.run/fal-ai/hedra-v1-image-to-video  (doesn't exist)
✅ https://fal.run/fal-ai/veo3.1/reference-to-video     (correct)
```

---

### Error: `unexpected value; permitted: 'X'`

**Symptom:**
```
422 - "unexpected value; permitted: '8s'"
```

**Cause:**
API parameter value doesn't match allowed values.

**Solution:**
1. **Check API documentation** for exact allowed values
2. **Common cases:**
   - Duration: Some models only support specific values (e.g., only "8s", not "6s")
   - Aspect ratio: Format matters ("16:9" vs "16_9" vs "landscape")
   - Enum values: Case-sensitive, exact match required

**Example (Veo 3.1):**
```json
{
  "duration": "6s"  // ❌ Not allowed
  "duration": "8s"  // ✅ Correct (only option)
}
```

**Best Practice:**
When integrating a new API, always test with the documented example values first.

---

### Error: `Cannot access application` (401 Authentication)

**Symptom:**
```
401 - "Cannot access application 'fal-ai/veo3.1'. Authentication is required"
```

**Cause:**
API key not properly formatted or credential not configured correctly.

**Solution:**
1. **Check header format** - Some APIs require prefixes:
   ```
   ❌ Authorization: YOUR_API_KEY
   ✅ Authorization: Key YOUR_API_KEY       (Fal.ai format)
   ✅ Authorization: Bearer YOUR_TOKEN      (Common OAuth format)
   ```

2. **Verify credential value** includes required prefix in Header Auth:
   - Header Name: `Authorization`
   - Header Value: `Key 1ef8e954-42c4-...` (NOT just the API key alone)

3. **Test credentials** outside n8n first with curl

---

## 4. Node Reference Errors

### Error: `Referenced node doesn't exist`

**Symptom:**
```
Error: Referenced node doesn't exist
Cannot assign to read only property 'name' of object 'Error: Referenced node doesn't exist'
```

**Cause:**
Code node references a node by name that has been renamed or deleted.

**Solution:**
When renaming nodes, update **all references** to that node in downstream Code nodes.

**Example:**
```javascript
// Node was renamed: "Poll Fal.ai Until Complete" → "Parse Fal.ai Response"

// ❌ WRONG - References old name
const result = $('Poll Fal.ai Until Complete').item.json;

// ✅ CORRECT - References new name
const result = $('Parse Fal.ai Response').item.json;
```

**Best Practice:**
- Search entire workflow JSON for old node name before renaming
- Use descriptive, stable names that won't need changing
- Keep a mapping of old → new names when refactoring

**Prevention:**
When renaming a node:
1. Note the old name
2. Search workflow for `$('Old Node Name')`
3. Update all references to new name
4. Validate workflow before deploying

---

## 5. HTTP Request Node Issues

### Error: `Method not allowed` (405)

**Symptom:**
```
405 - "Method Not Allowed"
```

**Cause:**
HTTP method (GET/POST/PUT/DELETE) doesn't match what the API endpoint expects.

**Solution:**
1. **Verify the API documentation** for correct HTTP method
2. **Common pattern:**
   - GET: Retrieve data, status checks
   - POST: Create resources, submit data
   - PUT: Update resources
   - DELETE: Remove resources

**Example:**
```json
// ❌ WRONG - Using GET for data submission
{
  "method": "GET",
  "url": "https://fal.run/fal-ai/veo3.1/reference-to-video"
}

// ✅ CORRECT - Use POST for submission
{
  "method": "POST",
  "url": "https://fal.run/fal-ai/veo3.1/reference-to-video",
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "=..."
}
```

**Common Mistake:**
When doing partial node updates with `n8n_update_partial_workflow`, the method can accidentally change if not included in the update. Always specify the full parameters object.

---

### Error: `Required property 'URL' cannot be empty`

**Symptom:**
```
Workflow validation error: Required property 'URL' cannot be empty
```

**Cause:**
HTTP Request node is missing the `url` parameter, often due to incomplete partial updates.

**Solution:**
When updating HTTP Request nodes with `n8n_update_partial_workflow`, **always include all required parameters**:

```javascript
// ❌ WRONG - Only updating body
{
  "type": "updateNode",
  "updates": {
    "parameters": {
      "jsonBody": "=..."  // Missing method, url, etc.
    }
  }
}

// ✅ CORRECT - Include all parameters
{
  "type": "updateNode",
  "updates": {
    "parameters": {
      "method": "POST",
      "url": "https://...",
      "authentication": "predefinedCredentialType",
      "nodeCredentialType": "httpHeaderAuth",
      "sendBody": true,
      "specifyBody": "json",
      "jsonBody": "=...",
      "options": {"timeout": 600000}
    }
  }
}
```

**Prevention:**
- Always get the current node parameters first
- Merge your changes with existing parameters
- Validate workflow after updates

---

## 6. Configuration & Hardcoded URLs

### Error: Wrong Cloud Functions Domain (404)

**Symptom:**
```
404 - Page not found (HTML response)
The resource you are requesting could not be found
```

When calling Cloud Functions that are confirmed to exist and be deployed.

**Cause:**
Using incorrect Cloud Functions domain in hardcoded URLs.

**Common Mistake:**
```javascript
// ❌ WRONG - Old domain
const uploadUrl = 'https://europe-west1-codika-app.cloudfunctions.net/uploadworkflowoutput';

// ✅ CORRECT - Use the API gateway
const uploadUrl = 'https://api.codika.io/uploadworkflowoutput';
```

**Domain Patterns:**
- ❌ `europe-west1-codika-app.cloudfunctions.net` (old, blocked by ingress)
- ✅ `api.codika.io` (correct — goes through LB + Cloud Armor)

**Solution:**
1. **Verify the project ID** in Firebase Console
2. **Check deployed functions:**
   ```bash
   gcloud functions list --region=europe-west1
   ```
3. **Use the API gateway domain:**
   ```
   https://api.codika.io/{functionName}
   ```

**Prevention:**
- Store Cloud Function URLs in environment variables or config
- Use ORGSECRET placeholders for dynamic URLs
- Don't mix project names (platform vs app)
- Verify URLs in Firebase Console before hardcoding

**Where This Appears:**
Common in Process Input nodes that set callback/upload URLs for workflow executions.

---

## 7. Execution & Testing Issues

### Error: `EXECUTION_NOT_PENDING` (Upload to completed execution)

**Symptom:**
```
404 - The resource you are requesting could not be found
```

When calling `uploadWorkflowOutput` with execution metadata from a previous run.

**Cause:**
The Cloud Function only accepts file uploads for executions with status `"pending"`. Once an execution is completed or failed, uploads are rejected.

**Code Location:**
`codika_app_platform/functions/src/features/process-executions/api/uploadWorkflowOutput.ts:119-122`

**Why This Design:**
- Prevents uploading to already-completed executions
- Avoids race conditions
- Ensures data consistency

**Solution for Testing:**

**Option A: Verify Data Structure Only**
```javascript
// In "Prepare Video Upload", verify output has:
console.log('Upload data ready:', {
  executionId: executionMetadata.executionId,
  executionSecret: executionMetadata.executionSecret,
  hasBinaryData: !!videoBinary,
  uploadUrl: uploadUrl
});
// Don't actually upload during testing
```

**Option B: Mock Upload Response**
Replace "Upload Video" node temporarily with Code node:
```javascript
// Mock successful upload for testing
return [{
  json: {
    success: true,
    documentId: "test-doc-" + Date.now(),
    storagePath: "test/videos/santa.mp4",
    fileName: "santa-video.mp4",
    fileSize: 2688061,
    mimeType: "video/mp4"
  },
  binary: $binary  // Pass through for downstream nodes
}];
```

**Option C: Use n8n Execution Replay**
1. Find a successful execution in n8n history
2. Click "Use execution data" or "Test workflow with data from..."
3. n8n will replay using cached data (no API calls)

**Best Practice:**
For expensive API operations ($3-5/call), structure workflows so you can:
1. Pin/mock expensive API responses
2. Test downstream logic separately
3. Do one final end-to-end test when ready

---

## 7. Quick Reference - Error Codes

| Error Code | Meaning | Common Cause | Quick Fix |
|------------|---------|--------------|-----------|
| `Module 'crypto' is disallowed` | Security restriction | Using crypto module | Use Cloud Function |
| `fetch is not defined` | API not available | Using fetch() | Use HTTP Request node |
| `$http is not defined` | Helper not available | Using $http helper | Use HTTP Request node |
| `Credential ... does not exist for type` | Wrong credential type | Custom type doesn't exist | Use httpHeaderAuth |
| `Header name must be a valid HTTP token` | Invalid header name | Special chars in header | Use simple names (Authorization) |
| `Application '...' not found` (404) | Wrong endpoint | Incorrect model name | Check API docs |
| `Method not allowed` (405) | Wrong HTTP method | Using GET instead of POST | Use correct method |
| `unexpected value; permitted` (422) | Invalid param value | Value not in allowed list | Check API docs for allowed values |
| `Cannot access application` (401) | Auth failed | Missing/wrong API key format | Check header prefix (Key/Bearer) |
| `Referenced node doesn't exist` | Broken reference | Node renamed | Update all `$('Node Name')` |
| `Required property 'URL' cannot be empty` | Missing parameter | Incomplete partial update | Provide full parameters object |
| `EXECUTION_NOT_PENDING` (404 upload) | Upload to old execution | Execution already completed | Can only upload to pending executions |
| 404 HTML "Page not found" | Cloud Function not found | Wrong domain (codika-app-platform vs codika-app) | Use correct project domain |
| "?" icon on LangChain node | Missing AI sub-node | No lmChatAnthropic connected | Add model node with ai_languageModel connection |
| Credentials ignored on chainLlm | Wrong node for credentials | Credentials on chainLlm instead of model | Move credentials to lmChatAnthropic node |
| Invalid FLEXCRED placeholder | Invented placeholder name | Using non-existent placeholder | Use documented: FLEXCRED_ANTHROPIC_ID_DERCXELF |
| Model not connected | Wrong connection type | Using `main` instead of `ai_languageModel` | Use ai_languageModel connection type |

---

## 8. API Endpoint Selection

### Error: Using async endpoint without polling infrastructure

**Symptom:**
Workflow uses `https://queue.fal.run/...` (async endpoint) but doesn't have proper polling logic, or polling fails due to Code node limitations.

**Cause:**
Async endpoints return immediately with `request_id`, requiring separate polling to get results. Polling requires HTTP calls that can't be done with `fetch()` in Code nodes.

**Solution:**

**Option A: Use Synchronous Endpoint** (Simplest)
```json
{
  "url": "https://fal.run/fal-ai/model-name",  // NOT queue.fal.run
  "options": {
    "timeout": 600000  // 10 minutes - endpoint waits for completion
  }
}
```

**Option B: Implement Polling with Native Nodes**
If you must use async endpoint:
1. Call `https://queue.fal.run/...` → get `request_id`
2. Loop:
   - Wait node (10 seconds)
   - HTTP Request to status endpoint
   - If node checks status
   - Loop back if not complete
3. Continue when status = "COMPLETED"

**Comparison:**
```
Async (queue.fal.run):
  Submit → Returns request_id immediately
  Poll every 10s until complete
  More complex, requires loop nodes

Sync (fal.run):
  Submit → Waits for completion → Returns result
  Simpler, just needs high timeout
  May fail if timeout too short
```

**Best Practice:**
Use synchronous endpoints for n8n workflows unless the operation takes longer than n8n's maximum timeout (usually 10 minutes).

---

## 9. Flexcreds & Placeholder Patterns

### Common Flexcred Mistakes

**Issue:** Using wrong placeholder pattern for credential type

**Patterns by Credential Level:**

```javascript
// FLEXCRED (AI providers, works out-of-box)
"credentials": {
  "httpHeaderAuth": {
    "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",      // ✅ Correct
    "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"   // ✅ Correct
  }
}

// ❌ WRONG - Don't use non-existent credential types
"credentials": {
  "falAiApi": {  // This type doesn't exist in n8n
    "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}"
  }
}

// ORGCRED (Organization-level)
"credentials": {
  "whatsAppApi": {
    "id": "{{ORGCRED_WHATSAPP_ID_DERCGRO}}",
    "name": "{{ORGCRED_WHATSAPP_NAME_DERCGRO}}"
  }
}

// USERCRED (Member OAuth)
"credentials": {
  "googleSheetsOAuth2Api": {
    "id": "{{USERCRED_GOOGLE_SHEETS_ID_DERCRESU}}",
    "name": "{{USERCRED_GOOGLE_SHEETS_NAME_DERCRESU}}"
  }
}
```

**For Raw API Keys in Code Nodes:**
```javascript
// When you need the actual key value (not credential ID)
const apiKey = '{{FLEXCRED_FAL_AI_KEY_DERCXELF}}';  // Gets replaced during deployment
```

**Reference:** See INTEGRATIONS_REFERENCE.md section 1 for complete patterns.

---

## 10. Partial Workflow Updates

### Losing Parameters During Updates

**Problem:** Using `n8n_update_partial_workflow` with incomplete parameters loses existing configuration

**Example Error:**
After updating just the `jsonBody`, the node loses `method`, `url`, and other required fields, causing validation errors.

**Solution:**
Always provide **complete parameters object** when updating nodes:

```javascript
// ❌ RISKY - May lose other parameters
{
  "type": "updateNode",
  "nodeId": "my-http-node",
  "updates": {
    "parameters": {
      "jsonBody": "=..."  // Only updating body - other params lost!
    }
  }
}

// ✅ SAFE - Complete parameters
{
  "type": "updateNode",
  "nodeId": "my-http-node",
  "updates": {
    "parameters": {
      "method": "POST",
      "url": "https://api.example.com/endpoint",
      "authentication": "predefinedCredentialType",
      "nodeCredentialType": "httpHeaderAuth",
      "sendBody": true,
      "specifyBody": "json",
      "jsonBody": "=...",
      "options": {"timeout": 600000}
    }
  }
}
```

**Best Practice:**
1. Get current node with `n8n_get_workflow(id, mode='full')`
2. Extract existing parameters
3. Merge your changes
4. Update with complete object

---

## 11. Debugging Checklist

When encountering an error:

1. ✅ **Check execution logs** in n8n UI for exact error message
2. ✅ **Get execution details** with `n8n_executions(action='get', id=executionId)`
3. ✅ **Validate workflow** with `n8n_validate_workflow(workflowId)`
4. ✅ **Review node configuration** - all required fields present?
5. ✅ **Check API documentation** - correct endpoint, method, parameters?
6. ✅ **Test credentials** - can you call the API directly with curl?
7. ✅ **Verify node references** - do all `$('Node Name')` references exist?
8. ✅ **Check for Code node limitations** - no fetch, crypto, or restricted modules?
9. ✅ **Review recent changes** - what was modified since last working version?

---

## 12. Testing Strategies for Expensive Operations

### Problem: API costs $3-5 per call, can't afford repeated testing

**Solutions:**

### A. n8n Execution Replay (Recommended)
```
1. Run workflow once successfully (or until expensive node completes)
2. In n8n UI: Executions → Select execution → "Use execution data"
3. Edit workflow and test downstream nodes with cached data
4. No API calls made, no cost incurred
```

### B. Pin Node Output
```
1. Successful execution → Right-click node → "Pin output data"
2. Pinned data persists in editor
3. Manual testing uses pinned data
4. Test downstream flow without API calls
```

### C. Mock Expensive Nodes
```
1. Disable expensive API node
2. Add Code node with mock response:

   return [{
     json: {
       video: { url: "https://test-video.mp4" }
     }
   }];

3. Connect to downstream flow
4. Test with mock data
5. Re-enable real node when ready
```

### D. Add Test Mode Flag
```javascript
// In Process Input:
const testMode = body.test_mode || false;

// Add If node after validation:
// TRUE branch → Mock Data node
// FALSE branch → Real API call

// Deploy with test mode for development
// Switch to production when validated
```

### E. Manual Trigger Strategy
```
1. Pin output at expensive node (from successful run)
2. Add Manual Trigger node before next node
3. Disable expensive API node during testing
4. Manually trigger downstream flow
5. Use pinned data for testing
6. Re-enable and remove manual trigger when done
```

**Best Practice:**
- Use execution replay for quick iterations
- Add permanent test mode for long-term development
- Structure workflows to isolate expensive operations
- One final end-to-end test before production release

---

---

## 13. LangChain Node Configuration Errors

### Error: Node shows as "?" icon in n8n

**Symptom:**
A LangChain node (`chainLlm`, `agent`) displays as a question mark icon in the n8n editor and cannot be executed.

**Cause:**
LangChain nodes require connected AI sub-nodes. The node appears broken when:
1. Missing required `lmChatAnthropic` (or other model) sub-node
2. Connection type is wrong (`main` instead of `ai_languageModel`)
3. Invalid node configuration or typeVersion

**Solution:**
Add the required AI model sub-node and connect properly:

```json
// 1. Add lmChatAnthropic node (with credentials)
{
  "name": "Claude Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "typeVersion": 1.3,
  "credentials": {
    "anthropicApi": {
      "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
      "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
    }
  }
}

// 2. Connect with ai_languageModel type (not main)
"Claude Model": {
  "ai_languageModel": [[{"node": "Your Chain Node", "type": "ai_languageModel", "index": 0}]]
}
```

**Reference:** See [ai-nodes.md](../specific/ai-nodes.md) Section 3 for complete multi-node architecture.

---

### Error: Credentials on wrong node

**Symptom:**
Workflow executes but LLM calls fail with authentication errors, or credentials are ignored silently.

**Cause:**
Credentials placed on the `chainLlm` or `agent` node instead of the model node (`lmChatAnthropic`).

**Wrong:**
```json
{
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "credentials": {
    "anthropicApi": { "id": "...", "name": "..." }
  }
}
```

**Correct:**
```json
// Credentials go on lmChatAnthropic, NOT chainLlm
{
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "credentials": {
    "anthropicApi": { "id": "...", "name": "..." }
  }
}
```

---

### Error: Invalid FLEXCRED placeholder

**Symptom:**
```
Placeholder '{{FLEXCRED_ANTHROPIC_CLAUDE}}' not recognized
```
Or deployment fails with unrecognized placeholder.

**Cause:**
Using invented/incorrect placeholder names.

**Common Mistakes:**
```json
"{{FLEXCRED_ANTHROPIC_CLAUDE}}"      // Not a real placeholder
"={{FLEXCRED_ANTHROPIC}}"            // Wrong format (extra =)
"{{FLEXCRED_CLAUDE_ID}}"             // Wrong naming pattern
```

**Correct Format:**
```json
"{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}"   // ID placeholder
"{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}" // Name placeholder
```

**Valid FLEXCRED Placeholders:**
- `FLEXCRED_ANTHROPIC_ID_DERCXELF` / `FLEXCRED_ANTHROPIC_NAME_DERCXELF`
- `FLEXCRED_OPENAI_ID_DERCXELF` / `FLEXCRED_OPENAI_NAME_DERCXELF`
- `FLEXCRED_PINECONE_ID_DERCXELF` / `FLEXCRED_PINECONE_NAME_DERCXELF`

**Reference:** See [placeholder-patterns.md](../specific/placeholder-patterns.md) for all valid placeholders.

---

### Error: Wrong connection type for AI nodes

**Symptom:**
Model node appears connected in the editor but chainLlm says "No model connected" or behaves as if no model is available.

**Cause:**
Using `main` connection type instead of AI-specific connection types.

**Wrong:**
```json
"Claude Model": {
  "main": [[{"node": "Chain Node", "type": "main", "index": 0}]]
}
```

**Correct:**
```json
"Claude Model": {
  "ai_languageModel": [[{"node": "Chain Node", "type": "ai_languageModel", "index": 0}]]
}
```

**AI Connection Types Reference:**

| Connection Type | Purpose |
|-----------------|---------|
| `ai_languageModel` | Connect model nodes (lmChatAnthropic, lmChatOpenAI) to chainLlm/agent |
| `ai_outputParser` | Connect output parsers (outputParserStructured) to chainLlm |
| `ai_tool` | Connect tools (toolCode, toolWorkflow) to agent |
| `ai_memory` | Connect memory (memoryBufferWindow) to agent |
| `ai_embedding` | Connect embedding models (embeddingsOpenAi) to vector stores |
| `ai_document` | Connect document loaders to vector stores |

---

## 14. SplitInBatches + LangChain Nodes

### Error: SplitInBatches skips loop body with AI chain nodes

**Symptom:**
SplitInBatches v3 immediately fires the "done" output (output 1) with only 1 item, while the "loop" output (output 0) is empty. The AI chain nodes in the loop body never execute.

**Cause:**
SplitInBatches v3 does not reliably iterate when the loop body contains LangChain chain nodes (`chainLlm`, `agent`). The node skips the loop entirely.

**Solution:**
Remove SplitInBatches and let n8n process items natively. The `chainLlm` node automatically processes each incoming item individually.

```
❌ WRONG:
Items → SplitInBatches → chainLlm → Code → loop back → SplitInBatches (done) → Aggregate

✅ CORRECT:
Items → chainLlm → Code (runOnceForEachItem) → Aggregate Code (runOnceForAllItems)
```

When combining AI output with original item data in a downstream Code node, use index-based matching with `$('Node').all()` (NOT paired items — `$('Node').item.json` does not work through langchain nodes):

```javascript
const aiResults = $input.all();
const originals = $('Source Node').all();
const results = [];
for (let i = 0; i < aiResults.length; i++) {
  const ai = aiResults[i].json.output || aiResults[i].json;
  const orig = originals[i]?.json || {};
  results.push({ json: { id: orig.id, category: ai.category } });
}
return results;
```

**Reference:** See [ai-nodes.md](../specific/ai-nodes.md) Section 6.1 for the complete pattern.

---

## 14. WhatsApp-Specific Errors

Three production-facing gotchas worth memorising when building any WhatsApp bot on this platform. Both the multi-tenant and single-tenant architectures have the same failure modes. See [whatsapp-bots.md](../use-case/whatsapp-bots.md) for architecture selection, then the matching deep guide ([whatsapp-bots-multi-tenant.md](../use-case/whatsapp-bots-multi-tenant.md) or [whatsapp-bots-single-tenant.md](../use-case/whatsapp-bots-single-tenant.md)) for full fix context.

### Error: `null value in column "id" of relation "n8n_chat_histories" violates not-null constraint`

**Symptom:**

The bot always replies with its fallback error text ("Oops, I'm having a small technical issue…"). `codika get execution <id> --deep --slim --json` shows the above error inside a Postgres Chat Memory sub-node.

**Cause:**

The `n8n_chat_histories.id` column is `integer NOT NULL` with no identity sequence. n8n's `@n8n/n8n-nodes-langchain.memoryPostgresChat` does not supply an `id` on insert — it expects Postgres to auto-generate one. In Bot-Factory-descended schemas this worked only because n8n's `CREATE TABLE IF NOT EXISTS` fallback had created the table with IDENTITY first; when Drizzle `db:push` creates the table first from a schema missing `.generatedAlwaysAsIdentity()`, the broken shape persists.

**Fix:**

```sql
ALTER TABLE n8n_chat_histories ALTER COLUMN id ADD GENERATED ALWAYS AS IDENTITY;
```

Apply to both prod + dev branches. Also update `src/lib/db/schema.ts`:

```ts
id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
```

See the matching WhatsApp guide §3 (prerequisites) for full context.

---

### Error: Welcome template silently reclassified `UTILITY → MARKETING`

**Symptom:**

You submitted a template with `category: 'UTILITY'`. After the daily status-check cron runs, `whatsapp_templates.category` shows `MARKETING` and your Twilio bill starts climbing ~5× faster than expected.

**Cause:**

Your `sub-create-system-template` workflow submits the Meta ApprovalRequest without `allow_category_change: false`. Twilio defaults `allow_category_change` to `true`, which lets Meta silently reclassify the template based on its auto-classifier reading the body. Invitational / greeting / "anytime" / "feel free to" language reliably scores MARKETING.

**Fix:**

In `sub-create-system-template.json`, the HTTP Request node that hits `POST /v1/Content/{sid}/ApprovalRequests`:

```json
"jsonBody": "={{ JSON.stringify({ name: $json.friendlyName, category: $json.category, allow_category_change: false }) }}"
```

Then delete the MARKETING row from `whatsapp_templates`, rewrite the body as transactional confirmation of a user action (e.g. "You've been added to {{2}} on Codika, {{1}}. This is your direct line for…"), redeploy the orchestrator, and re-trigger `http-provision-templates`. Meta will now either approve UTILITY as stated or reject outright — no silent flip.

---

### Error: `code=2388299 userMessage="Variables can't be at the start or end of the template"`

**Symptom:**

Meta rejects a template after the status-check poll. Rejection reason contains `code=2388299`.

**Cause:**

The template body begins or ends with a `{{n}}` variable. Meta's rule: variables must have non-variable text on at least one side — ideally both.

**Fix:**

Prefix or suffix the body with plain text:

- BAD:  `{{1}}, tu as été ajouté à {{2}}…`
- GOOD: `Bonjour {{1}}, tu as été ajouté à {{2}} sur Codika…`

Delete the rejected row from `whatsapp_templates`, update the copy in the `SYSTEM_TEMPLATES` array inside `http-provision-templates.json`, redeploy, re-trigger provisioning.

---

*This document should be updated whenever new common errors are discovered. Include the error message, cause, and solution with code examples.*
