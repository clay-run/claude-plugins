---
name: discover-clay-actions
description: Fetches the live action catalog for this workspace. Search and discover available Clay actions for Clay workflows — email lookup, company enrichment, phone finders, etc. Includes commands for getting action input schemas.
allowed-tools: Bash(clay *), Bash(grep *), Bash(cat *), Bash(wc *), Bash(jq *), Read, Grep
---

# Discover Clay Actions

This skill helps you find available Clay actions for use in Clay workflow nodes,
via the `clay` CLI (on your PATH).

Not to be confused with `clay tools` — that lists workspace *function tools*, a
different concept. For workflow building blocks, use `clay workflows actions`.

## Actions catalog

The catalog is fetched live from the workspace's action catalog API. It includes all available actions with workspace-specific configuration (configured tools, app accounts, credit costs).

Each catalog entry has:
- `actionKey` — unique identifier for the action
- `packageId` — the package this action belongs to
- `displayName` / `name` — human-readable names
- `description` — what the action does
- `outputParameters` — what data the action returns
- `creditCost` — credits per execution (based on workspace billing plan)
- `dataStrengths` — what this action is best at (editorial metadata)
- `whyUseful` — when to use this action
- `configuredTools` — existing tool instances in this workspace, each with:
  - `toolId` — pass this to `edit_node` tools field to reuse an existing tool
  - `appAccountId` / `appAccountName` — bound credentials
- `availableAppAccounts` — app accounts the user has connected (for actions requiring API keys)
- `priorityTier` — lower is better (0 = functions, 1 = Clay first-party, 2 = has app account, 3 = Clay credits, 4 = requires key)

The catalog has no input field names. **Before wiring any input, fetch the real
names with `clay workflows actions schema` (see below) — never guess them**, or
the node will silently fail to bind.

### Using catalog data when adding tools

When adding a tool to a node, you can either:
1. **Reuse an existing tool** — pass `toolId` from `configuredTools`:
   ```json
   { "toolType": "clay_action", "toolId": "tct_abc123" }
   ```
2. **Create a new tool** — pass `actionKey` + `actionPackageId`:
   ```json
   { "toolType": "clay_action", "actionKey": "find-email-from-name", "actionPackageId": "..." }
   ```
3. **Bind specific credentials** — pass `appAccountId` from `availableAppAccounts`:
   ```json
   { "toolType": "clay_action", "actionKey": "...", "actionPackageId": "...", "appAccountId": "app_xyz" }
   ```

The catalog is auto-fetched below into `catalog.json`. If the fetch fails (e.g.
missing credentials), run `/setup` first.

```bash
clay workflows actions list > ${CLAUDE_SKILL_DIR}/catalog.json
```

!`clay workflows actions list > ${CLAUDE_SKILL_DIR}/catalog.json && echo "Wrote $(jq '.data | length' ${CLAUDE_SKILL_DIR}/catalog.json) actions to catalog.json" || echo "Fetch failed — run /setup if credentials are not configured"`

## How to search

The catalog is one big JSON object (`{ "data": [...] }`), kept fully greppable.
Search it for actions matching the user's request (`$ARGUMENTS`) with grep, or
filter structurally with jq:

```bash
grep -i "email" ${CLAUDE_SKILL_DIR}/catalog.json
jq -r '.data[] | select(.name | test("email";"i")) | "\(.priorityTier) \(.packageId) \(.actionKey) — \(.displayName)"' ${CLAUDE_SKILL_DIR}/catalog.json | sort
```

Prefer actions with lower `priorityTier` values and existing `configuredTools`.

## Getting action input schemas

Run this for any action whose inputs you'll bind, and use the exact `name`
values it returns:

```bash
clay workflows actions schema <packageId> <actionKey>
```

Example:

```bash
clay workflows actions schema 56058efe-4757-4fe7-a44b-39c2d730c47a find-email-from-name
```

This returns the action's `packageId`, `actionKey`, `displayName`, and
`inputParameters` (the input parameter schema). Pipe to `jq '.inputParameters'`
to see just the parameters.
