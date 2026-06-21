# Agent Plugins for Clay

Build with Clay in your AI coding agent — skills, MCP tools, and the Clay CLI.

## Installation

### Claude Code

```
/plugin marketplace add clay-run/agent-plugins
/plugin install clay@clay-plugins
```

### Codex

```
codex plugin marketplace add clay-run/agent-plugins
```

Then open **Plugins** and install **clay**.

### Cursor

Teams/Enterprise: Settings → Plugins → Add Marketplace → Import from Repo → `clay-run/agent-plugins`.

Otherwise (local install): the repo root is a *marketplace*, so clone it and copy the plugin itself — the `terracotta/` folder, which holds the plugin manifest — into your Cursor plugins dir, then reload Cursor:

```
git clone https://github.com/clay-run/agent-plugins.git
cp -R agent-plugins/terracotta ~/.cursor/plugins/local/clay
```

## Configuration

The `clay` CLI and the Clay MCP server both authenticate with a Clay API key. Create one in Clay under **Settings → Account** (the workspace is resolved from the key — there is no workspace id to set), then expose it as `CLAY_API_KEY`:

- **Claude Code** — run the bundled `setup` skill, which saves the key and verifies it with `clay whoami`.
- **Codex / Cursor** — export it in your shell so both the CLI and the MCP server read it:

  ```
  export CLAY_API_KEY="<your key>"
  ```

Verify with `clay whoami` — exit 0 prints your user and workspace; exit 3 means the key is missing or invalid.

## Using the `clay` CLI

In **Claude Code** the bundled `clay` CLI is on the agent's PATH automatically. **Codex and Cursor do not add a plugin's `bin/` to PATH** — the simplest fix is to ask the agent to run the bundled **`setup` skill**, which installs `clay` for you (no npm). To do it by hand, drop a forwarder — *not* a symlink, since the launcher locates its own files by path — into a directory on your PATH:

```
mkdir -p ~/.local/bin
launcher="$(find ~/.codex ~/.cursor ~/.claude -type f -path '*/bin/clay' 2>/dev/null | sort | tail -1)"
printf '#!/bin/sh\nexec "%s" "$@"\n' "$launcher" > ~/.local/bin/clay
chmod +x ~/.local/bin/clay
```

The CLI still needs `CLAY_API_KEY` in the environment (see above).
