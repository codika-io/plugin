# Cohere Integration Guide

Cohere provides AI models optimized for enterprise use cases including text generation, embeddings, and reranking.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `cohere` |
| **Credential Type** | `cohereApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "cohereApi": {
    "id": "{{FLEXCRED_COHERE_ID_DERCXELF}}",
    "name": "{{FLEXCRED_COHERE_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Node Types

| Node Type | Use Case |
|-----------|----------|
| `@n8n/n8n-nodes-langchain.lmChatCohere` | LLM model for chains/agents |
| `@n8n/n8n-nodes-langchain.embeddingsCohere` | Text embeddings |
| `@n8n/n8n-nodes-langchain.chainLlm` | Basic LLM chain (use with Cohere model) |
| `@n8n/n8n-nodes-langchain.agent` | AI agent with tools (use with Cohere model) |

---

## 3. Node Examples

### 3.1 Cohere Chat Model (Sub-node)

Used as a sub-node connected to chains and agents:

```json
{
  "name": "Cohere Model",
  "type": "@n8n/n8n-nodes-langchain.lmChatCohere",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": {
      "__rl": true,
      "value": "command-r-plus",
      "mode": "list"
    },
    "options": {
      "maxTokens": 4096,
      "temperature": 0.7
    }
  },
  "credentials": {
    "cohereApi": {
      "id": "{{FLEXCRED_COHERE_ID_DERCXELF}}",
      "name": "{{FLEXCRED_COHERE_NAME_DERCXELF}}"
    }
  }
}
```

### 3.2 Cohere Embeddings (Sub-node)

Used for RAG and vector search:

```json
{
  "name": "Cohere Embeddings",
  "type": "@n8n/n8n-nodes-langchain.embeddingsCohere",
  "typeVersion": 1,
  "position": [600, 176],
  "parameters": {
    "model": "embed-english-v3.0",
    "options": {}
  },
  "credentials": {
    "cohereApi": {
      "id": "{{FLEXCRED_COHERE_ID_DERCXELF}}",
      "name": "{{FLEXCRED_COHERE_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Available Models

### Chat Models

| Model ID | Description |
|----------|-------------|
| `command-r-plus` | Command R+ - Most capable |
| `command-r` | Command R - Balanced |
| `command` | Command - Previous generation |

### Embedding Models

| Model ID | Description |
|----------|-------------|
| `embed-english-v3.0` | Embeddings (English) |
| `embed-multilingual-v3.0` | Embeddings (Multilingual) |

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
- [ ] Correct embedding model selected for language requirements
- [ ] Error workflow configured
