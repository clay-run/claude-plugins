# Clay Actions & Tools

Clay actions are data enrichment tools that can be used in workflows. They provide capabilities like finding emails, looking up company info, phone numbers, and more.

There are two ways to use Clay actions:

1. **Enrich nodes** (`nodeType: "tool"`) — The simplest approach. Create an enrich node with exactly 1 tool. It executes the action directly, passing inputs from upstream nodes. No code or LLM needed.
2. **Code nodes** — Add tools to a code node and call them via `context.call_tool(actionKey, **inputs)` in Python. Use this when you need to process the results or call multiple actions with logic.

Note: Agent nodes do NOT use the `tools` field — they get tools via their Claygent configuration.

## Finding available actions

See the `execute_clay_action` MCP tool description — it documents which actions are permitted on this workspace and how to look up their schemas. The discovery surface is workspace-specific (configured tools, app accounts, credit costs, editorial metadata) and may be restricted by capability gates.

## Adding tools to nodes

Three ways to add a tool via `edit_node`:

**1. Reuse an existing configured tool** (preferred when available):
```json
{ "toolType": "clay_action", "toolId": "tct_abc123" }
```
Get `toolId` from the catalog's `configuredTools` array for each action.

**2. Create by actionKey + packageId:**
```json
{
  "toolType": "clay_action",
  "actionKey": "find-email-from-name",
  "actionPackageId": "56058efe-4757-4fe7-a44b-39c2d730c47a"
}
```
If the user has an app account for this action, it's auto-bound.

**3. Create with explicit app account** (for actions requiring API keys):
```json
{
  "toolType": "clay_action",
  "actionKey": "find-email-from-name",
  "actionPackageId": "56058efe-4757-4fe7-a44b-39c2d730c47a",
  "appAccountId": "app_xyz"
}
```
Get `appAccountId` from the catalog's `availableAppAccounts` for the action.

## Using tools in code nodes

After adding tools to a code node, call them from Python using:
```python
result = context.call_tool("actionKey", param1=value1, param2=value2)
```

To find an action's input parameters, see the `execute_clay_action` MCP tool description — it points to the discovery surface for input schemas.
