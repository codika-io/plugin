# OpenRouter Integration Guide

OpenRouter provides access to multiple AI models through a unified API, including models from Anthropic, OpenAI, Meta, Google, and others.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `open_router` |
| **Credential Type** | `openRouterApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "openRouterApi": {
    "id": "{{FLEXCRED_OPEN_ROUTER_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPEN_ROUTER_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with OpenRouter model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with OpenRouter model) |

---

## 3. Node Examples

### 3.1 OpenRouter Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "OpenRouter Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "anthropic/claude-3.5-sonnet",
      "mode": "list"
    },
    "options": {
      "maxTokensToSample": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "openRouterApi": {
      "id": "{{FLEXCRED_OPEN_ROUTER_ID_DERCXELF}}",
      "name": "{{FLEXCRED_OPEN_ROUTER_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Available Models

OpenRouter provides access to models from multiple providers:

| Model ID | Description |
|----------|-------------|
| `anthropic/claude-3.5-sonnet` | Claude 3.5 Sonnet |
| `openai/gpt-4-turbo` | GPT-4 Turbo |
| `meta-llama/llama-3.1-405b-instruct` | Llama 3.1 405B |
| `google/gemini-pro-1.5` | Gemini Pro 1.5 |

See [OpenRouter models](https://openrouter.ai/models) for the full list.

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

- [ ] Credentials use FLEXCRED pattern with `OPEN_ROUTER` (note underscore)
- [ ] Model ID specified using `__rl` pattern
- [ ] Model ID uses provider/model format (e.g., `anthropic/claude-3.5-sonnet`)
- [ ] Error workflow configured
