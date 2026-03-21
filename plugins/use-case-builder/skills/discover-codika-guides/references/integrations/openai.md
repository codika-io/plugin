# OpenAI Integration Guide

OpenAI provides GPT models and text embeddings.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `openai` |
| **Credential Type** | `openAiApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "openAiApi": {
    "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatOpenAi` | GPT chat model |
| `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | Text embeddings |
| `n8n-nodes-base.openAi` | Direct OpenAI API |

---

## 3. Node Examples

### 3.1 GPT Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "GPT Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
  "typeVersion": 1.2,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "gpt-4-turbo",
      "mode": "list"
    },
    "options": {
      "maxTokens": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "openAiApi": {
      "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
    }
  }
}
```

### 3.2 Embeddings (for RAG)

Used as a sub-node for vector store operations:

```json
{
  "name": "Get Embedding",
  "type": "@n8n/n8n-nodes-langchain.embeddingsOpenAi",
  "typeVersion": 1.2,
  "position": [400, 176],
  "parameters": {
    "model": "text-embedding-3-small",
    "options": {
      "dimensions": 512
    }
  },
  "credentials": {
    "openAiApi": {
      "id": "{{FLEXCRED_OPENAI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_OPENAI_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Available Models

### Chat Models

| Model ID | Description |
|----------|-------------|
| `gpt-4-turbo` | GPT-4 Turbo - Best balance |
| `gpt-4o` | GPT-4o - Latest multimodal |
| `gpt-4o-mini` | GPT-4o Mini - Faster/cheaper |
| `gpt-3.5-turbo` | GPT-3.5 - Budget option |

### Embedding Models

| Model ID | Description |
|----------|-------------|
| `text-embedding-3-small` | Efficient embeddings (recommended) |
| `text-embedding-3-large` | Higher quality embeddings |
| `text-embedding-ada-002` | Legacy embeddings |

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
- [ ] Model ID specified using `__rl` pattern (chat models)
- [ ] Embedding dimensions configured for RAG use cases
- [ ] Error workflow configured
