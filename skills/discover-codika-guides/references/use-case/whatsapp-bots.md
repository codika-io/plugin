# WhatsApp Bots — Start Here

This is the landing page. It helps you pick between the two canonical WhatsApp-bot architectures on Codika. Once you've picked, go read the matching deep guide — it will give you everything you need to build, deploy, and debug.

## The two patterns

| Pattern | One-line summary | Canonical codebase | Go to |
|---|---|---|---|
| **Multi-tenant** | One orchestrator process routes inbound WhatsApp by phone → `bots.subworkflow_id` into **a separate Codika process per client**. Each tenant has isolated credentials + their own Postgres schema. | `satellites/codika-client-bots` (also `codika-bot-factory`) | [`whatsapp-bots-multi-tenant.md`](./whatsapp-bots-multi-tenant.md) |
| **Single-tenant** | One Codika process contains everything — a main-router + multiple role-based AI agent handlers (admin / resident / community / onboarding / …). Shared DB, shared credentials. "Multiple agents" = multiple AI agent roles in the same process, NOT multiple processes. | `satellites/whatsapp-bots/instances/wat-dashboard/processes/wat-bot` | [`whatsapp-bots-single-tenant.md`](./whatsapp-bots-single-tenant.md) |

## How to choose

Answer these questions in order:

1. **Will this bot eventually serve multiple distinct clients / companies / customers**, each of whom must NOT see each other's data? → **Multi-tenant**.
2. **Do different user types (admins, members, guests, …) need different tool access and different system prompts**, but they all belong to the same organization with shared data? → **Single-tenant**.
3. **Are you just prototyping one bot for one audience with no role distinctions?** → **Single-tenant**, start with one handler and grow.

Decision cheat-sheet:

| Criterion | Multi-tenant | Single-tenant |
|---|---|---|
| Number of distinct client orgs | Many | One |
| Credential isolation between tenants | Required | Not needed |
| Per-tenant Postgres schema | Required (`creafid`, `acme`, …) | One shared schema |
| Ops complexity | Higher — deploy.sh per tenant, `bots` + `phone_mappings` tables | Lower — one process, one deploy |
| Shared Twilio number | Yes (one number routes to many tenants) | Yes (one number for the whole community) |
| AI agent variety | Usually one agent per client bot | Multiple role-specific agents in one process |
| Starting cost | Heavier (orchestrator + first tenant) | Lighter (one process) |

When in doubt, **default to single-tenant**. You can promote to multi-tenant later by extracting a handler into its own process and adding an orchestrator — the handler contract is identical.

## Shared prerequisites (both patterns)

Both architectures need the same platform plumbing. Details are inside each deep guide, but here are the non-negotiables:

1. **Twilio** (org-level credential, `ORGCRED_TWILIO`). Submit real SID + auth token via `codika integration set twilio --context-type organization`. Configure the inbound-message webhook for your Twilio number to point at Codika's n8n endpoint (the `twilioTrigger` node auto-registers this on workflow activation, but cycle the instance via `codika instance deactivate` then `activate` if you changed credentials after first deploy).
2. **Postgres** (instance-level credential, `INSTCRED_POSTGRES`). Typically a Neon project with a pooled connection string.
3. **AI provider** — Anthropic (flex, `FLEXCRED_ANTHROPIC`) for the agent, plus OpenAI if you need Whisper transcription for voice messages.
4. **The `n8n_chat_histories` identity gotcha — read this or your agent will never reply.** If your dashboard's Drizzle schema defines the `n8n_chat_histories` table (used by `@n8n/n8n-nodes-langchain.memoryPostgresChat`), the integer `id` column MUST be declared with `.generatedAlwaysAsIdentity()`:

   ```ts
   id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
   ```

   Without it, `drizzle-kit push` creates the column as `integer NOT NULL` with no sequence, and every chat-memory write fails silently with:

   ```
   null value in column "id" of relation "n8n_chat_histories" violates not-null constraint
   ```

   The agent then falls through to its error handler and the user sees the fallback message instead of the real response. See `product-builder/skills/setup-neon/SKILL.md §6b` for the full story.

5. **Deployment parameter `USER_BOT_PHONE`** — the Twilio WhatsApp number the bot receives on, formatted `+14436478971`. Every `codika deploy use-case` re-applies `defaultDeploymentParameters` from `config.ts`, so set the default there rather than relying on `codika redeploy --param`.

## What the deep guides cover that this landing doesn't

- Full `config.ts` skeleton with the minimum workflow set.
- `main-router` node-by-node: trigger, phone parsing, DB lookup, role/bot dispatch, Codika Submit Result.
- Handler skeleton: `executeWorkflowTrigger`, AI agent + Chat Memory + tool sub-workflows, sanitize + reply.
- `sub-send-whatsapp` chunking pattern and WhatsApp formatting table.
- **WhatsApp template provisioning** — the `http-provision-templates` + `sub-create-system-template` + `scheduled-template-status-check` trio, including `allow_category_change: false` (critical — without it Meta silently flips UTILITY → MARKETING at ~5× the cost) and the Meta variable-position rule.
- Testing via `http-test-bot`, inspecting nested sub-workflow traces with `codika get execution --deep`.
- WhatsApp-specific common errors with fixes.

Pick your pattern above and read the matching guide end-to-end before writing any workflow JSON.
