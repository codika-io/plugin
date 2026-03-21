# Mistral Integration Guide

Mistral provides open-weight and commercial AI models for text generation and analysis.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `mistral` |
| **Credential Type** | `mistralCloudApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "mistralCloudApi": {
    "id": "{{FLEXCRED_MISTRAL_ID_DERCXELF}}",
    "name": "{{FLEXCRED_MISTRAL_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatMistralCloud` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with Mistral model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with Mistral model) |

---

## 3. Node Examples

### 3.1 Mistral Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "Mistral Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatMistralCloud",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "mistral-large-latest",
      "mode": "list"
    },
    "options": {
      "maxTokens": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "mistralCloudApi": {
      "id": "{{FLEXCRED_MISTRAL_ID_DERCXELF}}",
      "name": "{{FLEXCRED_MISTRAL_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Available Models

| Model ID | Description |
|----------|-------------|
| `mistral-large-latest` | Mistral Large - Most capable |
| `mistral-medium-latest` | Mistral Medium - Balanced |
| `mistral-small-latest` | Mistral Small - Fast/cheap |
| `open-mixtral-8x22b` | Mixtral 8x22B - Open weights |

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

- [ ] Credentials use FLEXCRED pattern
- [ ] Credential type is `mistralCloudApi` (not `mistralApi`)
- [ ] Model ID specified using `__rl` pattern
- [ ] Error workflow configured
