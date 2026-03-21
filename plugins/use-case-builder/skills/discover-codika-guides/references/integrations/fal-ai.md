# fal AI Integration Guide

fal AI provides AI-powered image and video generation capabilities through their API.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `fal_ai` |
| **Node Type** | `n8n-nodes-base.httpRequest` (HTTP Request) |
| **Credential Type** | `httpHeaderAuth` (Header Auth) |
| **Credential Level** | FLEXCRED (works out-of-box) |

### Credential Pattern

```json
"credentials": {
  "httpHeaderAuth": {
    "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",
    "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"
  }
}
```

---

## 2. API Endpoints

fal AI does not have a dedicated n8n node. Use the HTTP Request node.

### Endpoint Types

| Type | URL Prefix | Behavior |
|------|------------|----------|
| Async (queue) | `https://queue.fal.run/` | Returns `request_id` to poll |
| Sync (direct) | `https://fal.run/` | Waits for completion |

### Common Endpoints

| Endpoint | Description |
|----------|-------------|
| `https://queue.fal.run/fal-ai/veo3.1/reference-to-video` | Google Veo 3.1 video generation (async) |
| `https://fal.run/fal-ai/flux/dev` | FLUX.1 [dev] image generation |
| `https://fal.run/fal-ai/flux/schnell` | FLUX.1 [schnell] (faster) |
| `https://fal.run/fal-ai/stable-diffusion-v3-medium` | Stable Diffusion v3 |
| `https://fal.run/fal-ai/flux-pro` | FLUX.1 [pro] (highest quality) |

---

## 3. Node Examples

### 3.1 Video Generation (Async)

```json
{
  "name": "Generate Video with Veo 3.1",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [600, 0],
  "parameters": {
    "method": "POST",
    "url": "https://queue.fal.run/fal-ai/veo3.1/reference-to-video",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"prompt\": \"{{ $json.textPrompt }}\",\n  \"image_urls\": [\"{{ $json.imageUrl }}\"],\n  \"duration\": \"6s\",\n  \"aspect_ratio\": \"16:9\",\n  \"generate_audio\": true,\n  \"resolution\": \"720p\"\n}",
    "options": {}
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"
    }
  }
}
```

### 3.2 Image Generation (Sync)

```json
{
  "name": "Generate Image with FLUX",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [600, 0],
  "parameters": {
    "method": "POST",
    "url": "https://fal.run/fal-ai/flux/dev",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"prompt\": \"{{ $json.prompt }}\",\n  \"image_size\": \"landscape_16_9\",\n  \"num_inference_steps\": 28,\n  \"guidance_scale\": 3.5\n}",
    "options": {}
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"
    }
  }
}
```

### 3.3 Poll Async Result

For queue endpoints, poll the status:

```json
{
  "name": "Poll Video Status",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [800, 0],
  "parameters": {
    "method": "GET",
    "url": "=https://queue.fal.run/fal-ai/veo3.1/reference-to-video/requests/{{ $json.request_id }}",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "options": {}
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{FLEXCRED_FAL_AI_ID_DERCXELF}}",
      "name": "{{FLEXCRED_FAL_AI_NAME_DERCXELF}}"
    }
  }
}
```

---

## 4. Response Formats

### 4.1 Async Response (Queue Endpoints)

Initial response:

```javascript
{
  "request_id": "abc123...",
  "status": "IN_QUEUE"
}
```

Poll result when completed:

```javascript
{
  "status": "COMPLETED",
  "video": {
    "url": "https://storage.googleapis.com/..."
  }
}
```

### 4.2 Sync Response (Direct Endpoints)

```javascript
{
  "images": [
    {
      "url": "https://fal.media/files/...",
      "width": 1024,
      "height": 768,
      "content_type": "image/jpeg"
    }
  ],
  "seed": 12345,
  "has_nsfw_concepts": [false],
  "prompt": "your prompt here"
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

- [ ] Credentials use FLEXCRED pattern with `FAL_AI` (note underscore)
- [ ] `nodeCredentialType` set to `httpHeaderAuth`
- [ ] Correct endpoint prefix (`queue.fal.run` vs `fal.run`)
- [ ] Poll mechanism implemented for async endpoints
- [ ] Error workflow configured
