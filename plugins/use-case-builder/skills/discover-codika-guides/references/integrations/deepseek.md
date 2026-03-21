# DeepSeek Integration Guide

DeepSeek provides cost-effective AI models with strong reasoning and coding capabilities.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `deep_seek` |
| **Credential Type** | `deepSeekApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "deepSeekApi": {
    "id": "{{FLEXCRED_DEEP_SEEK_ID_DERCXELF}}",
    "name": "{{FLEXCRED_DEEP_SEEK_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatDeepSeek` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with DeepSeek model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with DeepSeek model) |

---

## 3. Node Examples

### 3.1 DeepSeek Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "DeepSeek Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatDeepSeek",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "deepseek-chat",
      "mode": "list"
    },
    "options": {
      "maxTokens": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "deepSeekApi": {
      "id": "{{FLEXCRED_DEEP_SEEK_ID_DERCXELF}}",
      "name": "{{FLEXCRED_DEEP_SEEK_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Available Models

| Model ID | Description |
|----------|-------------|
| `deepseek-chat` | DeepSeek Chat - General purpose |
| `deepseek-coder` | DeepSeek Coder - Code-specialized |
| `deepseek-reasoner` | DeepSeek Reasoner - Enhanced reasoning |

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

- [ ] Credentials use FLEXCRED pattern with `DEEP_SEEK` (note underscore)
- [ ] Model ID specified using `__rl` pattern
- [ ] Error workflow configured
