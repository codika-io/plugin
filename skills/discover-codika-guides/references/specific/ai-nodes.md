# AI Nodes Guide

This guide explains how to use LangChain AI nodes in n8n workflows, including when to use `chainLlm` vs `agent` nodes and how to configure AI agent tools.

---

## 1. The Problem with Agent Nodes and Structured Output

When using LangChain nodes for LLM-based tasks, choosing the wrong node type will break structured output parsing.

### Why Agent Nodes Break Structured Output

The **Agent node** (`@n8n/n8n-nodes-langchain.agent`) is designed for complex multi-step reasoning with tool use. When processing a request, agents output verbose reasoning BEFORE the final answer:

```text
Based on the document content, I need to analyze the structure and authorship...

The document appears to be authored by Intys consultancy and addressed to GSK...

After careful analysis, this is clearly a proposal document because:
1. It contains pricing sections
2. It has team composition
3. It's addressed TO a client

{
  "documentType": "proposal",
  "confidence": "high",
  "reasoning": "Document is authored by Intys, contains pricing and team sections..."
}
```

The **Structured Output Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`) expects pure JSON, but receives reasoning text followed by JSON. This causes:

```text
Error: Model output doesn't fit required format
```

---

## 2. chainLlm vs agent: The Solution

For tasks requiring structured JSON output, use the **Basic LLM Chain** (`chainLlm`) instead of the Agent node.

### Decision Matrix

| Use Case | Node | Why |
|----------|------|-----|
| Document classification | `chainLlm` | Simple input → structured output |
| Metadata extraction | `chainLlm` | Direct extraction, no reasoning needed |
| Text categorization | `chainLlm` | Deterministic classification |
| JSON transformation | `chainLlm` | Straightforward mapping |
| Multi-step reasoning | `agent` | Needs tool use and deliberation |
| Complex research tasks | `agent` | Requires planning and iteration |
| Tool calling workflows | `agent` | Needs access to external tools |

**Rule of thumb:** If you need JSON output, use `chainLlm`. If you need tool access or complex reasoning, use `agent`.

---

## 3. LangChain Node Architecture (CRITICAL)

LangChain nodes in n8n require a **multi-node architecture**. The main node (`chainLlm` or `agent`) requires connected AI sub-nodes to function.

### Node Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                     LangChain Multi-Node Structure              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────┐                                           │
│   │ lmChatAnthropic │ ─── ai_languageModel ──►┐                 │
│   │ [has credentials]│                        │                 │
│   └─────────────────┘                         ▼                 │
│                                         ┌─────────────┐         │
│   ┌─────────────────┐                   │  chainLlm   │ ─► main │
│   │ outputParser    │ ─── ai_outputParser ──►│ (or agent)  │         │
│   │ Structured      │                   └─────────────┘         │
│   └─────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Rules

1. **Credentials go on the model node** (`lmChatAnthropic`), NOT on `chainLlm` or `agent`
2. **Connections use AI-specific types**: `ai_languageModel`, `ai_outputParser` (not `main`)
3. **Without the model sub-node**, the chainLlm/agent node shows as "?" in n8n

### AI Connection Types Reference

| Connection Type | Purpose | From Node → To Node |
|-----------------|---------|---------------------|
| `ai_languageModel` | Connect LLM model | lmChatAnthropic → chainLlm/agent |
| `ai_outputParser` | Connect output parser | outputParserStructured → chainLlm |
| `ai_memory` | Connect conversation memory | memoryBufferWindow → agent |
| `ai_tool` | Connect tools | toolCode/toolWorkflow → agent |
| `ai_embedding` | Connect embedding model | embeddingsOpenAi → vectorStore |
| `ai_document` | Connect document loader | documentDefaultDataLoader → vectorStore |

### LangChain Node Types Reference

| Node Type | typeVersion | Has Credentials |
|-----------|-------------|-----------------|
| `@n8n/n8n-nodes-langchain.chainLlm` | 1.7 | NO |
| `@n8n/n8n-nodes-langchain.agent` | 1.9 | NO |
| `@n8n/n8n-nodes-langchain.lmChatAnthropic` | 1.3 | YES (anthropicApi) |
| `@n8n/n8n-nodes-langchain.outputParserStructured` | 1.3 | NO |
| `@n8n/n8n-nodes-langchain.memoryBufferWindow` | 1.3 | NO |
| `@n8n/n8n-nodes-langchain.toolCode` | 1.1 | NO |
| `@n8n/n8n-nodes-langchain.toolWorkflow` | 2.1 | NO |
| `@n8n/n8n-nodes-langchain.vectorStorePinecone` | 1.3 | YES (pineconeApi) |
| `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | 1.2 | YES (openAiApi) |
| `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | 1.1 | NO |

---

## 4. Basic LLM Chain Configuration

A chainLlm setup requires **three nodes** and **special connections**.

### Node 1: LLM Chain (`chainLlm`)

```json
{
  "id": "chain-llm-001",
  "name": "Classify Document",
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "typeVersion": 1.7,
  "position": [620, 300],
  "parameters": {
    "promptType": "define",
    "text": "=Classify this document.\n\nDocument types:\n- \"proposal\": Business proposal sent to a client\n- \"report\": Internal or external report\n\nDocument content:\n{{ $json.content }}",
    "hasOutputParser": true,
    "options": {}
  }
}
```

**Note:** No `credentials` property on this node. Credentials go on the model node.

### Node 2: Claude Model (`lmChatAnthropic`) - HAS THE CREDENTIALS

```json
{
  "id": "claude-model-001",
  "name": "Claude Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "typeVersion": 1.3,
  "position": [572, 500],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "claude-haiku-4-5-20251001",
      "mode": "list",
      "cachedResultName": "Claude Haiku 4.5"
    },
    "options": {
      "maxTokensToSample": 1024,
      "temperature": 0.3
    }
  },
  "credentials": {
    "anthropicApi": {
      "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
      "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
    }
  }
}
```

### Node 3: Output Parser (`outputParserStructured`)

```json
{
  "id": "output-parser-001",
  "name": "Classification Parser",
  "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
  "typeVersion": 1.3,
  "position": [716, 500],
  "parameters": {
    "jsonSchemaExample": "{\n  \"documentType\": \"proposal\",\n  \"confidence\": \"high\"\n}"
  }
}
```

### Connections (CRITICAL)

```json
"connections": {
  "Previous Node": {
    "main": [[{"node": "Classify Document", "type": "main", "index": 0}]]
  },
  "Claude Model": {
    "ai_languageModel": [[{"node": "Classify Document", "type": "ai_languageModel", "index": 0}]]
  },
  "Classification Parser": {
    "ai_outputParser": [[{"node": "Classify Document", "type": "ai_outputParser", "index": 0}]]
  },
  "Classify Document": {
    "main": [[{"node": "Next Node", "type": "main", "index": 0}]]
  }
}
```

### Common Configuration Mistakes

| Mistake | What Happens | Correct Approach |
|---------|--------------|------------------|
| Credentials on chainLlm | Auth fails or ignored | Put credentials on lmChatAnthropic |
| Missing lmChatAnthropic node | "?" icon, node broken | Add model node with ai_languageModel connection |
| Using `main` for model connection | Model not linked | Use `ai_languageModel` connection type |
| Inventing placeholders like `FLEXCRED_ANTHROPIC_CLAUDE` | Deployment fails | Use documented placeholders: `FLEXCRED_ANTHROPIC_ID_DERCXELF` |

---

## 5. When to Use Agent Node

Only use the Agent node when you need:

1. **Tool access:** The workflow needs to call external APIs or tools
2. **Multi-step reasoning:** The task requires planning and iteration
3. **Complex deliberation:** The output depends on intermediate decisions

### Agent Node Example

The agent node also requires a model sub-node connected via `ai_languageModel`. The agent itself does NOT have credentials.

**Agent Node:**
```json
{
  "id": "agent-001",
  "name": "Research Agent",
  "type": "@n8n/n8n-nodes-langchain.agent",
  "typeVersion": 1.9,
  "position": [800, 0],
  "parameters": {
    "promptType": "define",
    "text": "={{ $json.userMessage }}",
    "options": {
      "systemMessage": "You are a helpful assistant with access to tools."
    }
  }
}
```

**Model Node (same as chainLlm):**
```json
{
  "id": "claude-model-001",
  "name": "Claude Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "typeVersion": 1.3,
  "position": [736, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "claude-haiku-4-5-20251001",
      "mode": "list"
    },
    "options": {
      "maxTokensToSample": 1000,
      "temperature": 0.2
    }
  },
  "credentials": {
    "anthropicApi": {
      "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
      "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
    }
  }
}
```

**Connections:**
```json
"Claude Model": {
  "ai_languageModel": [[{"node": "Research Agent", "type": "ai_languageModel", "index": 0}]]
}
```

**Note:** When using the Agent node, do NOT expect clean structured JSON output. The agent outputs text with reasoning. Process the agent's response separately if JSON is needed.

---

## 6. Best Practices

1. **Default to chainLlm** for any task that needs structured JSON output
2. **Only use agent** when you truly need multi-step reasoning or tool access
3. **Keep chainLlm prompts concise** - don't ask for explanations unless you need them in the output
4. **Use `includeRaw: true`** in the output parser options during development to debug parsing issues
5. **Test with real data** - ensure your prompts produce consistent JSON structure
6. **Never use SplitInBatches to loop over AI chain nodes** - see Section 6.1 below

### 6.1 Processing Multiple Items Through AI Chains

**Rule: Do NOT use SplitInBatches with chainLlm or agent nodes.** SplitInBatches v3 does not reliably iterate when the loop body contains langchain chain nodes — it skips the loop entirely and goes straight to the "done" output.

Instead, let n8n's native item-by-item processing handle it. When a `chainLlm` node receives multiple items, it processes each one individually.

```
❌ WRONG - SplitInBatches loop with AI chain:
Items → SplitInBatches → chainLlm → Code → loop back → SplitInBatches (done) → Aggregate

✅ CORRECT - Let items flow naturally:
Items → chainLlm → Code (runOnceForEachItem) → Aggregate Code (runOnceForAllItems)
```

**Pattern for processing multiple items with AI classification:**

1. **chainLlm** receives N items, processes each one individually (N calls to the LLM)
2. **Combine Code node** (default `runOnceForAllItems` mode) merges AI output with original data using index-based matching
3. **Aggregate Code node** (default `runOnceForAllItems` mode) collects all items with `$input.all()` and produces a single summary

**Important:** Paired item references (`$('Node').item.json`) do NOT work reliably through langchain chain nodes. Use `$('Node').all()` with index-based matching instead.

```json
{
  "name": "Combine AI Results with Original Data",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "jsCode": "const aiResults = $input.all();\nconst originals = $('Source Node').all();\n\nconst results = [];\nfor (let i = 0; i < aiResults.length; i++) {\n  const ai = aiResults[i].json.output || aiResults[i].json;\n  const orig = originals[i]?.json || {};\n  results.push({ json: { id: orig.id, category: ai.category } });\n}\nreturn results;"
  }
}
```

---

## 7. Error Handling and Retry for LangChain Nodes

### Node Architecture Recap

LangChain workflows use a parent-child node structure:

```
chainLlm (parent - MAIN EXECUTION)
├── lmChatAnthropic (child - via ai_languageModel)
└── outputParserStructured (child - via ai_outputParser)
```

**Key insight:** Child nodes (model, parser) don't execute independently - they're invoked by the parent `chainLlm` node. This affects how you should configure retry and error handling.

### Retry Configuration

Add retry **ONLY to the `chainLlm` node** (not child nodes):

```json
{
  "id": "chain-llm-001",
  "name": "My LLM Chain",
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "typeVersion": 1.7,
  "position": [600, 0],
  "parameters": {
    "promptType": "define",
    "text": "...",
    "hasOutputParser": true
  },
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 1000
}
```

This handles:
- **Transient API failures** - rate limits, 503 errors, timeouts
- **Unparseable LLM output** - retry may produce valid JSON on second attempt

**Don't add retry to:**
- `lmChatAnthropic` - sub-node, retry won't work independently
- `outputParserStructured` - deterministic, same invalid input = same parse failure

### Error Workflow vs Inline Error Handling

n8n provides two error handling mechanisms:

| Mechanism | Scope | When Error Workflow Triggers |
|-----------|-------|------------------------------|
| No `onError` (default) | Node stops, error propagates | **YES** - error workflow runs |
| `onError: "continueErrorOutput"` | Error routed to local branch | **NO** - handled inline |

**Recommended approach:** Use the default behavior (let errors propagate to error workflow) for most cases. The error workflow provides:
- Centralized error logging
- Consistent error handling
- Simpler workflow structure

Use `onError: "continueErrorOutput"` only when you need:
- Custom user-friendly error messages
- Different handling per error type
- Graceful degradation for batch processing

### Error Branch JSON Pattern (Red Connector)

When you need inline error handling on a node, you must configure two things:

1. **Node property:** Add `"onError": "continueErrorOutput"` to the node
2. **Connections:** Wire a **second output** (index 1) in the `main` connections array

**Which nodes need error branches:** HTTP Request, Google Sheets, Google Drive, Supabase, Execute Workflow, and other nodes that interact with external services or databases. These are flagged by the `ERROR-BRANCH-REQUIRED` validation rule.

**Which nodes do NOT need error branches:** LangChain sub-nodes (`outputParserStructured`, `lmChatAnthropic`, `memoryBufferWindow`, `embeddingsOpenAi`, `documentDefaultDataLoader`, `vectorStorePinecone`). These execute within their parent chain/agent node and cannot have independent error outputs.

#### Node with `onError`

```json
{
  "id": "http-001",
  "name": "Fetch Data",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [400, 0],
  "parameters": {
    "url": "https://api.example.com/data",
    "options": {}
  },
  "onError": "continueErrorOutput"
}
```

#### Connections with error branch (second output array)

The `main` array has two entries: index 0 is the success path, index 1 is the error path.

```json
"Fetch Data": {
  "main": [
    [{"node": "Process Result", "type": "main", "index": 0}],
    [{"node": "Handle Fetch Error", "type": "main", "index": 0}]
  ]
}
```

#### Complete example — HTTP Request with error branch

```json
{
  "nodes": [
    {
      "id": "http-001",
      "name": "Fetch Data",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [400, 0],
      "parameters": { "url": "https://api.example.com/data" },
      "onError": "continueErrorOutput"
    },
    {
      "id": "process-001",
      "name": "Process Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [650, -100],
      "parameters": { "jsCode": "return items;" }
    },
    {
      "id": "error-001",
      "name": "Handle Fetch Error",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [650, 100],
      "parameters": {
        "jsCode": "return [{ json: { success: false, error: $json.error?.message || 'API call failed' } }];"
      }
    }
  ],
  "connections": {
    "Fetch Data": {
      "main": [
        [{"node": "Process Result", "type": "main", "index": 0}],
        [{"node": "Handle Fetch Error", "type": "main", "index": 0}]
      ]
    }
  }
}
```

**Important:** Without `"onError": "continueErrorOutput"` on the node, the second output will never fire. The node property and the two-output connection pattern must both be present.

### Recommended Pattern

```
[Previous Node] → [chainLlm with retryOnFail] → [IF Success] → [Submit Result]
                                                           └→ [Report Error]
```

The `IF Success` check handles cases where the LLM returns unexpected output (e.g., missing required fields in the parsed JSON).

### Retry Settings Reference

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `retryOnFail` | boolean | `false` | Enable automatic retry on failure |
| `maxTries` | number | `3` | Maximum number of attempts (including first try) |
| `waitBetweenTries` | number | `0` | Milliseconds to wait between retries |

---

## 8. Troubleshooting chainLlm vs agent

### "Model output doesn't fit required format"

**Cause:** Using Agent node when chainLlm should be used.

**Fix:**
1. Change node type from `agent` to `chainLlm`
2. Simplify the prompt (remove reasoning instructions)
3. Ensure `hasOutputParser: true` is set

### Inconsistent JSON structure

**Cause:** Prompt not specific enough about output format.

**Fix:**
1. Add explicit JSON schema example in prompt
2. Be specific about required fields
3. Use Structured Output Parser with `fromJson` schema type

### Empty or partial output

**Cause:** LLM truncated response or prompt too long.

**Fix:**
1. Reduce input content length
2. Simplify expected output structure
3. Check model's context window limits

### LLM Chain output is nested in `output` object

**Cause:** When using `chainLlm` with a Structured Output Parser, the output structure is different than expected.

**Symptom:** Code node after LLM Chain reads `$json.answer` but gets `undefined`, while `$json.output.answer` has the value.

**Explanation:** The chainLlm node wraps the parsed output inside an `output` property:

```javascript
// LLM Chain + Output Parser outputs:
{
  "output": {
    "answer": "The WiFi password is WAT2027!"
  }
}

// NOT:
{
  "answer": "The WiFi password is WAT2027!"
}
```

**Fix:** In downstream Code nodes, access the nested structure:

```javascript
// ❌ WRONG
const answer = $json.answer;  // undefined

// ✅ CORRECT
const answer = $json.output?.answer || $json.answer || '';
```

**Why both patterns?** Using optional chaining with fallback handles both:
- LLM Chain output: `$json.output.answer`
- Direct structured data: `$json.answer`

---

## 9. Summary: chainLlm vs agent

| Scenario | Use |
|----------|-----|
| Need JSON output | `chainLlm` |
| Classification task | `chainLlm` |
| Extraction task | `chainLlm` |
| Need tool access | `agent` |
| Multi-step reasoning | `agent` |
| Complex research | `agent` |

---

## 10. AI Agent Tools (Sub-Workflows as Tools)

When using sub-workflows as tools in AI agents via the `toolWorkflow` node, use the `workflowInputs` object format.

### Parameter Configuration

```json
"workflowInputs": {
  "mappingMode": "defineBelow",
  "value": {
    "title": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('title', ``, 'string') }}",
    "event_date": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('event_date', ``, 'string') }}",
    "caller_phone": "={{ $('Parse Input').first().json.senderPhone }}"
  },
  "matchingColumns": [],
  "schema": [
    { "id": "title", "displayName": "title", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" },
    { "id": "event_date", "displayName": "event_date", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" },
    { "id": "caller_phone", "displayName": "caller_phone", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" }
  ],
  "attemptToConvertTypes": false,
  "convertFieldsToString": false
}
```

### Format Requirements

| Aspect | Format |
|--------|--------|
| Container | `workflowInputs.value` object |
| Schema | `workflowInputs.schema` array |
| $fromAI syntax | `$fromAI('key', ``, 'type')` with comment marker |
| Comment marker | `/*n8n-auto-generated-fromAI-override*/` (required) |

### Trusted Context Parameters

For parameters that should NOT come from AI (like user identity), use direct expressions:

```json
"caller_phone": "={{ $('Parse Input').first().json.senderPhone }}"
```

This prevents the AI from spoofing user identity.

### Complete toolWorkflow Node Example

```json
{
  "parameters": {
    "name": "create_event",
    "description": "Create a new community event.",
    "workflowId": {
      "__rl": true,
      "value": "{{SUBWKFL_tool-create-event_LFKWBUS}}",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "title": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('title', ``, 'string') }}",
        "event_date": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('event_date', ``, 'string') }}",
        "caller_phone": "={{ $('Parse Input').first().json.senderPhone }}"
      },
      "matchingColumns": [],
      "schema": [
        { "id": "title", "displayName": "title", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" },
        { "id": "event_date", "displayName": "event_date", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" },
        { "id": "caller_phone", "displayName": "caller_phone", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "string" }
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": false
    }
  },
  "id": "tool-create-event-001",
  "name": "Create Event Tool",
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2.1,
  "position": [992, 368]
}
```

### Troubleshooting AI Agent Tools

| Problem | Cause | Solution |
|---------|-------|----------|
| Parameters are `null` | Using `fields.values` format | Use `workflowInputs` format instead |
| AI can't fill parameters | Missing `$fromAI()` or wrong syntax | Use `$fromAI('key', ``, 'type')` with comment |
| Schema not recognized | Missing `schema` array | Add schema array with all parameters |
| `The value "" is not supported!` | Wrong field property name | Use `fieldId` not `fieldName` in sub-workflow Supabase nodes |

---

## 11. Tool Workflow Error Handling (CRITICAL)

### Why This Matters

When an AI agent calls a tool (sub-workflow), it expects a response to understand what happened. **If a tool fails silently (no response), the agent has no way to know the operation failed** and will assume it succeeded.

This leads to:
- Agent reporting success to users when operations actually failed
- Incorrect follow-up actions based on false assumptions
- Frustrated users who receive misleading information

### The Problem: Silent Failures

If a tool workflow has empty IF branches when validation fails:

```
❌ WRONG:
Validate Input → IF (success?) ─┬→ TRUE: Continue
                                └→ FALSE: [] (empty - no response!)
```

The agent receives no output and proceeds as if the operation succeeded.

### The Solution: Always Return Errors

Every validation IF node MUST have both branches connected:

```
✅ CORRECT:
Validate Input → IF (success?) ─┬→ TRUE: Continue to operation
                                └→ FALSE: Return Error → { success: false, error: "message" }
```

### Standard Tool Response Format

**Success Response:**
```json
{
  "success": true,
  "message": "Event created successfully",
  "data": { "id": "123", "title": "Community Meetup" }
}
```

**Error Response:**
```json
{
  "success": false,
  "error": "Event title is required"
}
```

### How Agents Interpret Tool Responses

1. **With error response:** Agent sees `success: false` and `error` message, informs user of the failure and may attempt to fix the issue
2. **With no response (silent failure):** Agent has no feedback, assumes success, and provides incorrect information to the user

### Example: Correct Error Return Node

```javascript
// Return validation error to caller
const input = $('Validate Input').first().json;
return [{ json: input }];  // Returns { success: false, error: "..." }
```

### Checklist for AI Tool Workflows

- [ ] Every IF node checking for errors has BOTH branches connected
- [ ] No empty `[]` arrays in IF node connections
- [ ] All error conditions return `{ success: false, error: "descriptive message" }`
- [ ] Success responses include `{ success: true, ... }`

See [sub-workflows.md](./sub-workflows.md#11-error-handling-in-tool-workflows) for detailed implementation patterns.

---

## 12. Voice Message Transcription with OpenAI Whisper

For workflows that receive WhatsApp voice messages, use OpenAI's Whisper API to transcribe audio to text before passing it to AI agents or handlers.

### When to Use

- WhatsApp bot receives voice notes (audio/ogg)
- Any workflow that needs speech-to-text conversion
- Multilingual audio — Whisper auto-detects the language (no hint needed)

### Architecture

Create a reusable **sub-workflow** (`sub-transcribe-audio`) that handles both Twilio and Baileys audio sources:

```
[Trigger] → [Route by Source]
  → Twilio: HTTP GET media URL (Twilio auth, binary response)
  → Baileys: Decode base64 to binary (Code node)
→ [Whisper API] → [Return { success, transcribedText }]
```

The parent workflow (main-router) calls this sub-workflow when it detects a media message, then injects the transcription as `[Voice message]: <text>` into the normal `messageBody` field. All downstream handlers work unchanged.

### Whisper API via HTTP Request Node

Use the `httpRequest` node (not the n8n OpenAI node) for more control over multipart form data:

```json
{
  "id": "transcribe-audio-001",
  "name": "Transcribe with Whisper",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "https://api.openai.com/v1/audio/transcriptions",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "openAiApi",
    "sendBody": true,
    "contentType": "multipart-form-data",
    "bodyParameters": {
      "parameters": [
        { "name": "model", "value": "whisper-1" },
        { "name": "response_format", "value": "json" },
        { "parameterType": "formBinaryData", "name": "file", "inputDataFieldName": "data" }
      ]
    },
    "options": { "timeout": 60000 }
  },
  "credentials": {
    "openAiApi": {
      "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
    }
  },
  "continueOnFail": true
}
```

**Key points:**
- The `file` parameter uses `parameterType: "formBinaryData"` with `inputDataFieldName: "data"` to send the binary audio from the previous node
- `model: "whisper-1"` is the only available Whisper model
- No `language` parameter needed — Whisper auto-detects, which is better for multilingual bots
- `response_format: "json"` returns `{ "text": "transcribed content" }`
- Set `continueOnFail: true` to handle transcription errors gracefully

### Credential Pattern

Whisper uses OpenAI API credentials (FLEXCRED level):

```json
"credentials": {
  "openAiApi": {
    "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
  }
}
```

### Downloading Audio from Twilio

Twilio media requires authenticated download. Use `predefinedCredentialType: twilioApi`:

```json
{
  "method": "GET",
  "url": "={{ $json.mediaUrl }}",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "twilioApi",
  "options": {
    "response": {
      "response": { "responseFormat": "file", "outputPropertyName": "data" }
    }
  }
}
```

The `responseFormat: "file"` ensures the audio is stored as binary data in the `data` property, which the Whisper node then reads via `inputDataFieldName: "data"`.

### Converting Base64 to Binary (Baileys/Evolution API)

For Baileys-sourced audio (base64 string from Evolution API's `getBase64FromMediaMessage` endpoint):

```javascript
const base64Data = $json.mediaBase64;
const mimeType = $json.mimeType || 'audio/ogg';
const buffer = Buffer.from(base64Data, 'base64');
const binaryData = await this.helpers.prepareBinaryData(buffer, 'audio.ogg', mimeType);
return [{ json: { ready: true }, binary: { data: binaryData } }];
```

### Injecting Transcription into Message Flow

After transcription, merge the result back into the message data:

```javascript
const original = $('Process Input').first().json;
const transcription = $input.first().json;

let messageBody;
if (transcription.success && transcription.transcribedText) {
  messageBody = `[Voice message]: ${transcription.transcribedText}`;
} else {
  messageBody = '[Voice message]: (Could not transcribe voice message)';
}

return [{ json: { ...original, messageBody, isVoiceMessage: true } }];
```

**Critical:** Downstream nodes must reference the node *after* the transcription merge, not `$('Process Input')` directly — otherwise they'll get the original empty `messageBody`. See [twilio.md](../integrations/twilio.md#11-receiving-media-messages-voice-images-etc) for details.

### Whisper Limits

| Constraint | Value |
|-----------|-------|
| Max file size | 25 MB |
| Supported formats | OGG, MP3, MP4, M4A, WAV, WEBM |
| Typical WhatsApp voice note | < 1 MB (well within limits) |
| Response time | 1-5 seconds for typical voice notes |
