---
name: setup
description: Set up Clay API credentials for the workflow CLI. Use when CLAY_API_KEY is missing, when `clay whoami` fails, or when the user wants to configure their Clay credentials.
allowed-tools: Bash(clay *), Read, Edit, Write
---

# Clay Workflow Setup

First, check if credentials are already configured. This prints the CLI's real
exit code on its own line (`exit_code=N`) so a non-zero status is preserved for
you to read instead of being swallowed:

!`clay whoami; echo "exit_code=$?"`

Read the result by the printed **exit_code and JSON**, not by any status string:

- **exit_code=0** — setup is working. The JSON has `user.name` and `workspace.id`.
  Tell the user setup is working (name the workspace) and stop here.
- **exit_code=3** — auth problem. Parse `error.code` from the JSON on stderr:
  - `auth_missing_api_key` → not configured yet. Continue with the steps below.
  - `auth_invalid` / `auth_forbidden` → the key is wrong or lacks scope. Continue
    below to set a valid key.
- **exit_code=5** — `network_error` / `network_timeout`: a connection problem. Check
  `CLAY_API_URL` and the network; do not re-collect the key.

Only continue with the steps below if setup is not already working.

---

This skill configures your Clay API credentials so the `clay` CLI and the Clay
MCP server work without manual env var setup. Both read the same `CLAY_API_KEY`
environment variable, so there is a single credential to manage. The workspace is
resolved from the API key — there is no separate workspace id to set.

## What you need

1. **API Key** — create one at your workspace settings page (Account tab)
2. **API URL** — defaults to `https://api.clay.com`, only change for local development

## Steps

### 1. Get or create an API key

Direct the user to their workspace settings to create an API key:

> Go to **https://app.clay.com** → your workspace → **Settings → Account** and
> create a new API key. Copy it and paste it here.

### 2. Save credentials

Read the existing `.claude/settings.local.json` file in the project root, then
merge the credentials into the `env` key. Preserve any existing settings (like
`permissions`).

The file should look like:

```json
{
  "permissions": { ... },
  "env": {
    "CLAY_API_KEY": "<the api key>",
    "CLAY_API_URL": "https://api.clay.com"
  }
}
```

This file is gitignored (`.claude/settings.local.json` is in `.gitignore`), so
credentials won't be committed. The `clay` CLI reads `CLAY_API_KEY` directly from
the environment, so no separate `clay login` step is needed.

### 3. Restart and resume

Tell the user:

> Credentials saved! Environment variables are loaded at session startup, so you
> need to restart for them to take effect. Run `/exit`, then `claude --continue`
> to pick up right where you left off.

## Verifying setup

After restart, run the health check:

```bash
clay whoami
```

Exit 0 with a `user`/`workspace` JSON object means setup is complete. A non-zero
exit means the credentials still need fixing — re-read the exit-code guide above.
