# Anthropic Integration Guide

Anthropic provides Claude models for text generation, classification, and analysis.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `anthropic` |
| **Credential Type** | `anthropicApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "anthropicApi": {
    "id": "{{FLEXCRED_ANTHROPIC_ID_DERCXELF}}",
    "name": "{{FLEXCRED_ANTHROPIC_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatAnthropic` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with Anthropic model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with Anthropic model) |

---

## 3. Node Examples

### 3.1 Claude Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "Claude Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatAnthropic",
  "typeVersion": 1.3,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "claude-sonnet-4-20250514",
      "mode": "list"
    },
    "options": {
      "maxTokensToSample": 4096,
      "temperature": 0.7
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

### 3.2 Basic LLM Chain (for Structured Output)

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

Connect a Claude Chat Model sub-node to the chainLlm node.

---

## 4. Available Models

| Model ID | Description |
|----------|-------------|
| `claude-sonnet-4-20250514` | Claude Sonnet 4 - Best balance |
| `claude-opus-4-20250514` | Claude Opus 4 - Most capable |
| `claude-haiku-4-5-20251001` | Claude Haiku 4.5 - Fastest/cheapest |

---

## 5. When to Use chainLlm vs agent

| Use Case | Node | Why |
|----------|------|-----|
| Document classification | `chainLlm` | Direct structured output |
| Metadata extraction | `chainLlm` | No reasoning needed |
| Text categorization | `chainLlm` | Deterministic |
| Multi-step reasoning | `agent` | Needs tool use |
| Tool calling workflows | `agent` | External tool access |

**Rule:** Use `chainLlm` for JSON output. Use `agent` for tools or complex reasoning.

---

## 6. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 7. Requirements Checklist

- [ ] Credentials use FLEXCRED pattern
- [ ] Model ID specified using `__rl` pattern
- [ ] `chainLlm` used for structured JSON output (not `agent`)
- [ ] Output parser connected when using `hasOutputParser: true`
- [ ] Error workflow configured
