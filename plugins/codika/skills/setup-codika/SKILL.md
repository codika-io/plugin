# Setup Codika

Install the `codika-helper` CLI and authenticate with the Codika platform.

## When to Use

- The user wants to set up their environment for Codika
- A deploy or project command fails with "API key is required"
- First-time setup before using any other Codika skill
- The user wants to check their current identity or switch between organizations

## Steps

### 1. Check if already installed

```bash
codika-helper --version
```

If the command is not found, install it:

```bash
npm install -g @codika-io/helper-sdk
```

### 2. Check current authentication

```bash
codika-helper whoami
```

If logged in, this shows the current organization, key name, scopes, and profile. If not logged in, proceed to step 3.

### 3. Login

**Interactive mode** (prompts for API key with masked input):

```bash
codika-helper login
```

**Non-interactive mode** (for CI/CD or scripting):

```bash
codika-helper login --api-key <key>
```

**With a custom profile name:**

```bash
codika-helper login --name my-org
```

**With a custom base URL** (for dev/staging):

```bash
codika-helper login --api-key <key> --base-url <url>
```

On login, the CLI verifies the key against the platform and stores a profile with full metadata (organization name, key name, scopes, expiration).

### 4. Verify

```bash
codika-helper whoami
```

Shows the authenticated identity: organization name, organization ID, key name, scopes, and expiry.

## Managing Multiple Profiles

The CLI supports multiple API keys (profiles). Each profile stores a key and its associated metadata.

### Add another key

```bash
codika-helper login --api-key <second-key>
```

A new profile is created automatically, named after the organization. If a profile for the same organization already exists, it is replaced.

### List all profiles

```bash
codika-helper use
```

Shows all stored profiles with an active marker:

```
Profiles:
  * acme-corp        Acme Corp          cko_HAdu...
    other-org        Other Org Inc      cko_xyz9...
```

### Switch active profile

```bash
codika-helper use <profile-name>
```

### Remove a profile

```bash
codika-helper logout              # removes the active profile
codika-helper logout <name>       # removes a specific profile
```

### Show all configuration

```bash
codika-helper config show
```

Lists all profiles and the active one.

### Clear all configuration

```bash
codika-helper config clear                   # remove everything
codika-helper config clear --profile <name>  # remove one profile
```

## Where to Get an API Key

API keys are created from the Codika dashboard under Organization Settings > API Keys. The user needs to create an org-level API key.

## Configuration Storage

- Config file: `~/.config/codika-helper/config.json`
- File permissions: `0o600` (owner read/write only)
- Directory permissions: `0o700` (owner only)
- Format: multi-profile JSON with active profile pointer

## Resolution Priority

When commands need an API key or URL, they check in this order:

1. `--api-key` / `--api-url` flag
2. `CODIKA_API_KEY` / `CODIKA_BASE_URL` environment variable
3. Active profile in config file
4. Production default (base URL only)

For deploy commands, if the use case's `project.json` contains an `organizationId`, the CLI automatically selects the profile matching that organization — even if it's not the currently active profile.

## Error Handling

| Symptom | Action |
|---------|--------|
| `npm: command not found` | Node.js is not installed. Ask the user to install Node.js 18+. |
| `EACCES` on global install | Try `npm install -g @codika-io/helper-sdk --prefix ~/.npm-global` or suggest using `npx`. |
| Login succeeds but deploy fails | Run `codika-helper whoami` — the key may not have the right scopes. |
| Wrong organization | Run `codika-helper use` to list profiles and switch with `codika-helper use <name>`. |
