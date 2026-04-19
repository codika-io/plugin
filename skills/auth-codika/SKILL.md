---
name: auth-codika
description: "Bootstrap the codika CLI from scratch using OTP email signup/login — no browser required. Use when the agent needs to provision a fresh Codika account and `cko_` API key from the terminal, when `codika whoami` / any codika skill returns 401, or when the user asks how to get a Codika API key and can receive email. Runs `codika auth signup-request` / `signup-complete` (or login equivalents); falls back to pasting a dashboard key via `setup-codika` if the user prefers."
---

# Codika CLI OTP Authentication

Install the `codika` CLI and attach an organization API key to this machine using a 6-digit OTP the user reads from their inbox. After this, every other codika skill authenticates automatically: no browser, no env vars, no header juggling.

The preferred path is **OTP from the terminal** — two commands and the key is minted and saved. Dashboard-paste is available via `setup-codika` for users who already have a `cko_`.

## Prerequisites

- Node.js 20+
- An email address the user can check (for the OTP path)

## Step 1: Install the CLI

```bash
npm install -g codika
codika --version
```

If `npm install -g` is blocked, `npx codika ...` works for any command.

## Step 2: Pick a path

Ask the user which applies — don't guess:

- **No Codika account yet** → **Path A: OTP signup** (most common for first-time use).
- **Account exists, but this machine has no key (or the current key is bad)** → **Path B: OTP login**.
- **User already copied a `cko_...` from the dashboard** → **Path C: paste via `setup-codika`** (fallback).

If the user is unsure, start with Path A. The backend tells you (via `USER_ALREADY_HAS_ORGANIZATION`) if you should switch to Path B, and vice versa. Handle that branching automatically — don't make the user figure it out.

## Path A — OTP signup

```bash
codika auth signup-request --email $EMAIL --json
```

Expected success shape:

```json
{ "success": true, "data": { "email": "...", "expiresInSeconds": 600 } }
```

Tell the user "I sent a 6-digit code to `<email>` — paste it back when you have it" and wait. Then:

```bash
codika auth signup-complete --email $EMAIL --code $OTP --json
```

Optional flags for populating the new organization at the same time:

```bash
codika auth signup-complete --email $EMAIL --code $OTP --json \
  --company "Acme" \
  --description "We make widgets"
```

Expected success shape (v1 of the CLI):

```json
{
  "success": true,
  "data": {
    "profileName": "my-organization",
    "organizationId": "...",
    "organizationName": "My Organization",
    "isNewUser": true,
    "apiKey": {
      "keyId": "...",
      "keyPrefix": "cko_xxxx",
      "name": "CLI default key",
      "scopes": ["deploy:use-case","projects:create","workflows:trigger","executions:read","instances:read","instances:manage","skills:read","integrations:manage","api-keys:manage","projects:read"],
      "createdAt": "..."
    }
  }
}
```

The `cko_` key is saved to `~/.config/codika/config.json` and set as the active profile. Surface `organizationName`, `organizationId`, and `keyPrefix` to the user.

**Branching**: if `signup-request` or `signup-complete` returns `USER_ALREADY_HAS_ORGANIZATION`, switch to **Path B** (the user already has an account; OTP login gives them a fresh key).

## Path B — OTP login

```bash
codika auth login-request --email $EMAIL --json
codika auth login-complete --email $EMAIL --code $OTP --json
```

Same flow, same response shape. Each `login-complete` mints a **new** `cko_` key scoped to an existing organization; previous keys stay valid until revoked from the dashboard.

**Multi-org users**: Codika users can belong to multiple organizations. If the user has more than one, `login-complete` returns `MULTIPLE_ORGANIZATIONS` with `details.organizations = [{ id, name }, ...]`. The CLI will print the list to stderr; re-run with the target org:

```bash
codika auth login-complete --email $EMAIL --code $OTP --organization-id <id> --json
```

**Branching**: if `login-request` returns `USER_NOT_FOUND` or `USER_HAS_NO_ORGANIZATION`, switch to **Path A**.

## Path C — Paste a dashboard key (fallback)

Covered by the `setup-codika` skill. Use it only if the user explicitly has a key already (they say so, or they paste a `cko_...` at you). If they don't have one and ask where to get one, direct them back to Path A/B (30 seconds) rather than walking them through the dashboard.

## Step 3: Verify

```bash
codika whoami --json
```

Must return `success: true` with an `organizationName`, `keyPrefix` starting with `cko_`, and the 10 default scopes.

## Error-handling table

Every `codika auth ...` error payload carries a `nextAction` string — use it. Pass it through to the user verbatim when surfacing failures; its wording is chosen to be actionable.

| Code | What to do |
|---|---|
| `OTP_RESEND_COOLDOWN` (`details.retryInSeconds`) | Wait the number of seconds, then re-run the `-request`. |
| `OTP_EXPIRED`, `OTP_NOT_FOUND`, `OTP_ALREADY_USED`, `OTP_LOCKED_OUT` | Re-run the matching `-request` to get a fresh code. |
| `OTP_INVALID` (`details.attemptsRemaining`) | Ask the user to re-check the email; they have `attemptsRemaining` tries before lockout. |
| `OTP_PURPOSE_MISMATCH` | Happens if you crossed signup and login flows. Re-run the matching `-request`. |
| `USER_ALREADY_HAS_ORGANIZATION` | Switch from Path A → Path B. |
| `USER_NOT_FOUND` / `USER_HAS_NO_ORGANIZATION` | Switch from Path B → Path A. |
| `EMAIL_RATE_LIMITED` / `IP_RATE_LIMITED` | 20/hour per email or 60/hour per IP hit. Wait and retry. |
| `MULTIPLE_ORGANIZATIONS` (`details.organizations`) | Re-run `login-complete` with `--organization-id <id>` for the target org. |
| `ORGANIZATION_NOT_MEMBER` | The user is not in the organization they asked for. Re-run with an org id from `details.organizations`. |
| `MAX_API_KEYS_REACHED` | The org already has 20 active keys. Ask the user to revoke an unused one from the dashboard, then retry. |
| `COMPANY_NAME_TOO_LONG` | Shorten `--company` to ≤ 100 chars, or omit (defaults to "My Organization"). |
| `INVALID_EXPIRES_IN_DAYS` | Omit `--key-expires-in` or pass 1–365. |
| `ORGANIZATION_CREATION_FAILED` | Backend rejected the org creation. Surface the `message` verbatim and retry with different parameters. |
| `INTERNAL` | Retry in a few seconds. |

## Output checklist

- [ ] `codika --version` prints a version number.
- [ ] The active profile in `~/.config/codika/config.json` has a `cko_` key and the 10 default scopes.
- [ ] `codika whoami --json` returns `success: true`.
- [ ] Tell the user which organization they are logged into, the masked key prefix, and that other codika skills (deploy / project / trigger / etc.) will now work.

## Pitfalls

- **Starting interactively without an email.** The OTP path requires `--email`; don't try `codika auth signup-request` with no flags.
- **Falling back too fast.** If a signup hits `USER_ALREADY_HAS_ORGANIZATION`, that's a useful signal — switch to login, don't ask the user for a pasted key.
- **Leaking the code.** The 6-digit OTP arrives by email. Accept it from the user, but never echo it to shared logs.
- **Skipping org disambiguation.** For users in multiple orgs, `login-complete` without `--organization-id` intentionally fails with `MULTIPLE_ORGANIZATIONS`. Read the `details.organizations` list, let the user pick, then re-run — don't default to the first org silently.
- **CLI not on PATH after `npm install -g`.** Check `npm prefix -g` and ensure that bin dir is on PATH, or fall back to `npx codika ...`.
