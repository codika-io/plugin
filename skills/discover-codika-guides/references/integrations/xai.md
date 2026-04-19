# xAI Integration Guide

xAI provides Grok models for text generation, classification, and analysis.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `x_ai` |
| **Credential Type** | `xAiApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "xAiApi": {
    "id": "{{FLEXCRED_X_AI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_X_AI_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatGrok` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with xAI model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with xAI model) |

---

## 3. Node Examples

### 3.1 xAI Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "xAI Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatGrok",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "grok-3",
      "mode": "list"
    },
    "options": {
      "maxTokensToSample": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "xAiApi": {
      "id": "{{FLEXCRED_X_AI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_X_AI_NAME_DERCXELF}}"
    }
  }
}
```

### 3.2 Basic LLM Chain with xAI (for Structured Output)

Use `chainLlm` instead of `agent` when you need structured JSON output with a parser.

```json
{
  "name": "Classify Document",
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "typeVersion": 1.7,
  "position": [400, 0],
  "parameters": {
    "promptType": "define",
    "text": "=Classify this document.\n\nDocument types:\n- \"proposal\"\n- \"rfp\"\n- \"unknown\"\n\nDocument:\n{{ $json.content }}",
    "hasOutputParser": true,
    "options": {}
  }
}
```

Connect an xAI Chat Model sub-node to the chainLlm node.

---

## 4. Available Models

| Model ID | Description |
|----------|-------------|
| `grok-3` | Grok 3 - Most capable |
| `grok-3-mini` | Grok 3 Mini - Faster/cheaper |
| `grok-2` | Grok 2 - Previous generation |

---

## 5. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 6. Requirements Checklist

- [ ] Credentials use FLEXCRED pattern with `X_AI` (note underscore)
- [ ] Model ID specified using `__rl` pattern
- [ ] `chainLlm` used for structured JSON output (not `agent`)
- [ ] Error workflow configured
