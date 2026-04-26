# Brevo Integration Guide

Brevo (formerly Sendinblue) is a French/EU transactional email provider. Use it for OTP login emails, invitation emails, notifications, and any HTTP-based email send from a workflow.

> **Implementation Reference:** [Brevo API — sendTransacEmail](https://developers.brevo.com/reference/sendtransacemail)

Brevo is **not** a built-in n8n integration. It is consumed via the standard HTTP Request node using a custom (`cstm_brevo`) integration registered in the use case's `config.ts`.

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `cstm_brevo` (custom integration; you declare it in your `customIntegrations` array) |
| **Node Type** | `n8n-nodes-base.httpRequest` (no native Brevo node) |
| **n8n Credential Type** | `httpHeaderAuth` |
| **Credential Level** | `INSTCRED` (process_instance). Org-level (`ORGCRED`) is desired but not currently accepted by the platform deploy validator for `cstm_*` integrations — workaround: register the same key on each instance. |
| **API Endpoint** | `https://api.brevo.com/v3/smtp/email` |
| **Auth Header** | `api-key: xkeysib-...` (lowercase header name; raw key value with **no `Bearer` prefix**) |

### Custom Integration Declaration

In your `config.ts`, include this in `customIntegrations`:

```ts
{
  id: "cstm_brevo",
  name: "Brevo",
  description: "Brevo transactional email API (EU)",
  contextType: "process_instance",
  n8nCredentialType: "httpHeaderAuth",
  n8nCredentialMapping: { BREVO_API_KEY: "value", HEADER_NAME: "name" },
  secretFields: [
    {
      key: "BREVO_API_KEY",
      label: "Brevo API Key",
      type: "password",
      description: "Brevo API key (raw, no prefix). Found under SMTP & API → API Keys.",
      placeholder: "xkeysib-...",
      required: true,
    },
    {
      key: "HEADER_NAME",
      label: "Header Name",
      type: "string",
      description: "HTTP header name (Brevo expects 'api-key', lowercase).",
      placeholder: "api-key",
      required: true,
    },
  ],
  icon: "Mail",
  color: "#1F8FCE",
} satisfies CustomIntegrationSchema
```

And reference `"cstm_brevo"` in the relevant workflow's `integrationUids` array.

### Credential Pattern

```json
"credentials": {
  "httpHeaderAuth": {
    "id": "{{INSTCRED_CSTM_BREVO_ID_DERCTSNI}}",
    "name": "{{INSTCRED_CSTM_BREVO_NAME_DERCTSNI}}"
  }
}
```

At deploy time, Codika resolves these placeholders to the n8n credential ID storing the API key + header name. n8n then injects `api-key: <key>` into every request automatically.

---

## 2. Codika-Specific Tips

### CRITICAL: Empty `cc` arrays are rejected

**This is the most common Brevo bug, and it does not show up with Resend.**

Brevo treats `"cc": []` as a malformed field and rejects the request with:

```json
{
  "error": {
    "name": "NodeApiError",
    "message": "Bad request - please check your parameters",
    "description": "cc is missing"
  }
}
```

The error wording is misleading. Brevo's actual rule: when the `cc` field is **present**, it must contain at least one entry; otherwise the field must be **omitted entirely** from the request body.

Same logic applies to `bcc` and `replyTo` — present-but-empty arrays are invalid.

#### ❌ Wrong (will 400)

```json
{
  "sender": { "name": "Acme", "email": "noreply@acme.com" },
  "to": [{ "email": "client@example.com" }],
  "cc": [],
  "subject": "Hello",
  "htmlContent": "<p>Hi</p>"
}
```

#### ✅ Correct (omit `cc` when no recipients)

```json
{
  "sender": { "name": "Acme", "email": "noreply@acme.com" },
  "to": [{ "email": "client@example.com" }],
  "subject": "Hello",
  "htmlContent": "<p>Hi</p>"
}
```

#### Recommended pattern: build body conditionally in a Code node

Trying to omit a key conditionally inside the n8n HTTP node's `jsonBody` template is awkward. The clean pattern is to build the full Brevo body in an upstream Code node and conditionally include `cc` (and `bcc`, `replyTo`):

```javascript
const brevoBody = {
  sender: { name: 'Acme', email: 'noreply@acme.com' },
  to: toList.map(email => ({ email })),
  subject: `Subject for ${customerEmail}`,
  htmlContent: htmlBody,
  attachment: [{ name: 'document.pdf', content: pdfBase64 }]
};
if (ccEmails.length > 0) {
  brevoBody.cc = ccEmails.map(email => ({ email }));
}
if (bccEmails.length > 0) {
  brevoBody.bcc = bccEmails.map(email => ({ email }));
}

return [{ json: { brevoBody } }];
```

Then the HTTP node's `jsonBody` is a single expression:

```json
"jsonBody": "={{ JSON.stringify($json.brevoBody) }}"
```

This is far more reliable than per-field interpolation and gives you full conditional control over the request shape.

### Recipients are objects, not strings

Resend (and most JSON email APIs) accept `"to": ["email@example.com"]` as an array of strings. **Brevo requires `"to": [{ "email": "email@example.com" }]`** — array of objects with an `email` key (and optional `name`).

Same shape for `cc`, `bcc`, `replyTo`. Forgetting this gives a generic 400.

### Sender is an object, not a string

Resend uses `"from": "Acme <noreply@acme.com>"`. **Brevo uses `"sender": { "name": "Acme", "email": "noreply@acme.com" }`.** The field is named `sender` (not `from`) and it is an object.

### Attachments are `attachment` (singular), with `name` (not `filename`)

| Resend | Brevo |
|---|---|
| `"attachments": [{ "filename": "x.pdf", "content": "<base64>" }]` | `"attachment": [{ "name": "x.pdf", "content": "<base64>" }]` |

The field name is `attachment` (singular, even though it accepts an array), and inside each entry the file name field is `name` (not `filename`).

### Successful response uses `messageId`, not `id`

Resend returns `{ "id": "..." }`. **Brevo returns `{ "messageId": "..." }`.** When you store the email reference in your database or check whether the send succeeded, use `$json.messageId` (not `$json.id`).

```javascript
// Wrong (Resend pattern):
const emailSent = !!emailResult.id;

// Right (Brevo pattern):
const emailSent = !!emailResult.messageId;
```

### Region selection is for sending only, NOT for data residency

When you select `eu-west-1` (Ireland) in Brevo's domain settings, **only the SMTP relay endpoint moves to Ireland**. Account data, email metadata, recipient lists, and API logs are still stored in the United States, regardless of the region chosen. From Brevo's own docs and DPA:

> "Region selection controls where your emails are routed and sent from. It does not control where customer data is stored. All account data, including email metadata, logs, and API records, is stored in the United States regardless of the sending region you select."

> "Customer acknowledges that Company's primary processing operations take place in the United States, and that the transfer of Customer's Personal Data to the United States is necessary for the provision of the Services"

This matters for GDPR. If a use case has a strict EU data residency requirement, do not claim Brevo provides full residency — only "EU sending region with US data storage under SCC + DPF". For full residency, use AWS SES `eu-west-1` directly or an EU-native provider with documented EU-only storage.

### Domain verification

Brevo accepts any local-part (`noreply@`, `team@`, `bot@`, …) on a verified domain — you don't need to register specific senders. Verifying `mail.example.com` once lets you send from any `*@mail.example.com`. Verification is via DKIM + SPF DNS records on the sending domain.

### Free tier limits

Brevo's free tier allows ~300 emails/day. For higher volumes, requires a paid plan. Plenty for low-volume transactional use cases (OTP, invitations, status notifications). Insufficient for marketing-scale sends.

### Error handling

Always set `continueOnFail: true` on the HTTP node so a transient Brevo failure doesn't kill the whole workflow. Downstream you can inspect `emailResult.error` (n8n surfaces the API error as a structured object on the output) and write `email_sent: false` to your database for retry.

---

## 3. Request Body Reference

Field-by-field for `POST /v3/smtp/email`:

| Field | Type | Required | Notes |
|---|---|---|---|
| `sender` | object | Yes | `{ "name": "...", "email": "..." }`. Email must be on a verified domain. |
| `to` | array of objects | Yes | `[{ "email": "...", "name": "..." }]`. Array, even for a single recipient. |
| `cc` | array of objects | No | Same shape as `to`. **Omit entirely if no CCs — empty array is rejected.** |
| `bcc` | array of objects | No | Same shape as `to`. **Omit entirely if no BCCs.** |
| `replyTo` | object | No | `{ "email": "...", "name": "..." }`. **Omit entirely if not used.** |
| `subject` | string | Yes (when no `templateId`) | Email subject line. |
| `htmlContent` | string | Yes (when no `templateId`) | Raw HTML body. Brevo does not require pre-built dashboard templates. |
| `textContent` | string | No | Plain-text fallback. Recommended for deliverability. |
| `templateId` | number | No | Use a pre-built dashboard template instead of inline content. Mutually exclusive with `htmlContent` + `subject`. |
| `params` | object | No | Variable substitutions when using `templateId`. |
| `attachment` | array of objects | No | `[{ "name": "x.pdf", "content": "<base64>" }]`. Note: singular `attachment`, and `name` (not `filename`). |
| `headers` | object | No | Custom email headers (e.g., for List-Unsubscribe). |
| `tags` | array of strings | No | For Brevo dashboard analytics. |

---

## 4. Common Mistakes

| Wrong | Correct | Issue |
|---|---|---|
| `"cc": []` | omit `cc` field entirely | Brevo rejects empty arrays |
| `"to": ["a@b.com"]` | `"to": [{"email": "a@b.com"}]` | Brevo wants objects, not strings |
| `"from": "Acme <a@b.com>"` | `"sender": { "name": "Acme", "email": "a@b.com" }` | Brevo uses `sender` object |
| `"attachments": [{...}]` | `"attachment": [{...}]` | Singular field name |
| `{"filename": "x.pdf"}` | `{"name": "x.pdf"}` | Brevo uses `name` for attachment filename |
| `Authorization: Bearer xkeysib-...` | `api-key: xkeysib-...` | Lowercase header name, no `Bearer` prefix |
| `emailResult.id` | `emailResult.messageId` | Brevo returns `messageId` |
| Selecting `eu-west-1` for GDPR | Disclose US storage in DPA | Region affects sending, not data storage |

---

## 5. Node Examples

### 5.1 Build Body in Code Node (recommended)

```json
{
  "parameters": {
    "mode": "runOnceForAllItems",
    "jsCode": "const toList = $json.toEmails || [];\nconst ccEmails = $json.ccEmails || [];\n\nconst brevoBody = {\n  sender: { name: 'Acme', email: 'noreply@mail.acme.com' },\n  to: toList.map(email => ({ email })),\n  subject: `Welcome ${$json.customerName}`,\n  htmlContent: $json.htmlBody\n};\nif (ccEmails.length > 0) {\n  brevoBody.cc = ccEmails.map(email => ({ email }));\n}\n\nreturn [{ json: { brevoBody } }];"
  },
  "id": "build-brevo-body-001",
  "name": "Build Brevo Body",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [800, 0]
}
```

### 5.2 HTTP Send via Brevo

```json
{
  "parameters": {
    "method": "POST",
    "url": "https://api.brevo.com/v3/smtp/email",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify($json.brevoBody) }}",
    "options": {
      "timeout": 30000
    }
  },
  "id": "send-brevo-001",
  "name": "Send Email via Brevo",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1000, 0],
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_BREVO_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_BREVO_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true,
  "retryOnFail": true,
  "maxTries": 2,
  "waitBetweenTries": 2000
}
```

### 5.3 Inline Body (simple case, no conditional fields)

When you know `cc`/`bcc` are never set, you can interpolate inline. **Do not use this pattern when CC may be empty** — see section 2.

```json
{
  "parameters": {
    "method": "POST",
    "url": "https://api.brevo.com/v3/smtp/email",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"sender\": { \"name\": \"Acme\", \"email\": \"noreply@mail.acme.com\" },\n  \"to\": [{ \"email\": \"{{ $json.recipientEmail }}\" }],\n  \"subject\": \"{{ $json.subject }}\",\n  \"htmlContent\": \"{{ $json.html }}\"\n}",
    "options": { "timeout": 30000 }
  },
  "id": "send-brevo-simple-001",
  "name": "Send Brevo (simple)",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1000, 0],
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_BREVO_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_BREVO_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

### 5.4 Use a Dashboard Template

```json
{
  "parameters": {
    "method": "POST",
    "url": "https://api.brevo.com/v3/smtp/email",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"to\": [{ \"email\": \"{{ $json.recipientEmail }}\" }],\n  \"templateId\": 42,\n  \"params\": { \"firstName\": \"{{ $json.firstName }}\", \"otp\": \"{{ $json.otp }}\" }\n}",
    "options": { "timeout": 30000 }
  },
  "id": "send-brevo-template-001",
  "name": "Send Brevo (template)",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1000, 0],
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_BREVO_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_BREVO_NAME_DERCTSNI}}"
    }
  },
  "continueOnFail": true
}
```

When `templateId` is provided, `subject`, `htmlContent`, and `sender` come from the dashboard template — only `to` and `params` are needed in the request.

---

## 6. Reading the Response

Brevo's `200 OK` response shape:

```json
{
  "messageId": "<202604261453.93420934124@smtp-relay.mailin.fr>"
}
```

For batched sends (multiple `to` entries), the response is:

```json
{
  "messageIds": ["<...>", "<...>"]
}
```

Use `$('Send Email via Brevo').first().json.messageId` (or `messageIds`) to detect success and persist the reference for audit/debugging.

---

## 7. Migrating from Resend to Brevo

Common substitutions when converting an existing Resend-based workflow:

| Aspect | Resend | Brevo |
|---|---|---|
| URL | `https://api.resend.com/emails` | `https://api.brevo.com/v3/smtp/email` |
| Auth header | `Authorization: Bearer re_...` | `api-key: xkeysib-...` |
| Sender | `"from": "Name <a@b.com>"` (string) | `"sender": { "name": "Name", "email": "a@b.com" }` (object) |
| Recipients | `"to": ["a@b.com"]` (strings) | `"to": [{ "email": "a@b.com" }]` (objects) |
| Body field | `"html": "..."` | `"htmlContent": "..."` |
| Attachments | `"attachments": [{ "filename": "...", "content": "..." }]` | `"attachment": [{ "name": "...", "content": "..." }]` |
| Empty `cc` | Tolerated | Rejected — omit field instead |
| Response field | `id` | `messageId` |
| Header name | `Authorization` | `api-key` (lowercase) |
| Auth value | `Bearer re_...` | `xkeysib-...` (no prefix) |
| Custom integration | `cstm_resend` | `cstm_brevo` |

When updating an existing custom integration's `n8nCredentialMapping` and `secretFields`:
- Rename `RESEND_API_KEY` → `BREVO_API_KEY`
- Update `HEADER_NAME` placeholder/default from `Authorization` to `api-key`
- Update the API key field's description: stop including the `Bearer ` prefix in the value

---

## 8. Troubleshooting

| Error | Cause | Solution |
|---|---|---|
| `cc is missing` | Empty `cc: []` in body | Omit `cc` field entirely when no CCs (build body in Code node, see section 2) |
| `bcc is missing` | Empty `bcc: []` | Same — omit when empty |
| `Bad request - please check your parameters` (no `description`) | Various: `to` as array of strings, `from` instead of `sender`, `attachments` instead of `attachment` | Verify shape against section 3 |
| `401 Unauthorized` | API key wrong, or `Bearer ` prefix mistakenly included in the credential value | Use raw key only, no prefix; confirm header name is `api-key` (lowercase) |
| `400 Sender not allowed` | Sending domain not verified in Brevo dashboard | Add and verify the sender domain (DKIM + SPF) under Senders & IP → Domains |
| `email_sent` always false in DB | Reading `$json.id` instead of `$json.messageId` | Brevo returns `messageId`; update DB write logic |
| 24h+ wait for verification | DNS records not propagated | Confirm DKIM + SPF records are live (`dig TXT`); some registrars take hours |
| Free-tier daily limit reached | >300 emails/day on free plan | Upgrade plan or batch send via the `messageVersions` field for shared content |

---

## 9. Settings

Standard Codika error-workflow setting on the workflow:

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 10. Requirements Checklist

- [ ] `cstm_brevo` declared in `customIntegrations` with `httpHeaderAuth` credential type
- [ ] `n8nCredentialMapping`: `{ BREVO_API_KEY: "value", HEADER_NAME: "name" }`
- [ ] `BREVO_API_KEY` secret field stores the **raw** key (no `Bearer` prefix)
- [ ] `HEADER_NAME` field defaults to `api-key` (lowercase)
- [ ] Sending domain verified in Brevo dashboard (DKIM + SPF DNS records live)
- [ ] HTTP node URL: `https://api.brevo.com/v3/smtp/email`
- [ ] HTTP node uses `nodeCredentialType: "httpHeaderAuth"` with the `INSTCRED_CSTM_BREVO_*` placeholders
- [ ] Body construction uses upstream Code node when `cc`/`bcc`/`replyTo` may be empty
- [ ] Empty `cc` arrays are **never** sent — field is omitted entirely when no CCs
- [ ] Recipients are objects (`[{ "email": "..." }]`), not strings
- [ ] Sender is an object (`{ "name": "...", "email": "..." }`)
- [ ] Attachment field is `attachment` (singular), with `name` (not `filename`)
- [ ] Downstream code reads `messageId` (not `id`) from the response
- [ ] HTTP node has `continueOnFail: true` and `retryOnFail: true`
- [ ] If GDPR strict residency is required, US data storage is disclosed in the DPA (region setting alone does not satisfy this)
