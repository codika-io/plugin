# Tavily Integration Guide

Tavily provides AI-optimized web search, content extraction, and web crawling capabilities.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `tavily` |
| **Node Type** | `@tavily/n8n-nodes-tavily.tavily` |
| **Credential Type** | `tavilyApi` |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "tavilyApi": {
    "id": "{{FLEXCRED_TAVILY_ID_DERCXELF}}",
    "name": "{{FLEXCRED_TAVILY_NAME_DERCXELF}}"
  }
}
```

---

## 2. Available Resources

| Resource | Description |
|----------|-------------|
| `search` (default) | Web search with optional AI answer |
| `extract` | Extract content from URLs |
| `crawl` | Crawl websites |

---

## 3. Node Examples

### 3.1 Search (Default)

```json
{
  "name": "Search Tavily",
  "type": "@tavily/n8n-nodes-tavily.tavily",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "query": "={{ $json.searchQuery }}",
    "options": {
      "searchDepth": "basic",
      "maxResults": 10,
      "includeAnswer": true
    }
  },
  "credentials": {
    "tavilyApi": {
      "id": "{{FLEXCRED_TAVILY_ID_DERCXELF}}",
      "name": "{{FLEXCRED_TAVILY_NAME_DERCXELF}}"
    }
  }
}
```

### 3.2 Search Options

| Option | Type | Description |
|--------|------|-------------|
| `searchDepth` | `"basic"` \| `"advanced"` | Search depth (basic is faster) |
| `maxResults` | number | Maximum results to return (default: 10) |
| `includeAnswer` | boolean | Include AI-generated answer summary |
| `topic` | `"general"` \| `"news"` | Search topic focus |
| `timeRange` | `"day"` \| `"week"` \| `"month"` \| `"year"` | Time range filter (for news) |

### 3.3 Extract

```json
{
  "name": "Extract Content",
  "type": "@tavily/n8n-nodes-tavily.tavily",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "resource": "extract",
    "urls": "={{ $json.urlsToExtract }}",
    "options": {}
  },
  "credentials": {
    "tavilyApi": {
      "id": "{{FLEXCRED_TAVILY_ID_DERCXELF}}",
      "name": "{{FLEXCRED_TAVILY_NAME_DERCXELF}}"
    }
  }
}
```

### 3.4 Crawl

```json
{
  "name": "Crawl Website",
  "type": "@tavily/n8n-nodes-tavily.tavily",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "resource": "crawl",
    "url": "={{ $json.websiteUrl }}",
    "options": {}
  },
  "credentials": {
    "tavilyApi": {
      "id": "{{FLEXCRED_TAVILY_ID_DERCXELF}}",
      "name": "{{FLEXCRED_TAVILY_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Response Format (Search)

```javascript
{
  "query": "search query",
  "answer": "AI-generated summary answer",
  "results": [
    {
      "title": "Result Title",
      "url": "https://example.com/page",
      "content": "Snippet of the page content...",
      "score": 0.95
    }
  ]
}
```

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
- [ ] Resource specified (`search`, `extract`, or `crawl`)
- [ ] Query/URLs provided as input
- [ ] Error workflow configured
