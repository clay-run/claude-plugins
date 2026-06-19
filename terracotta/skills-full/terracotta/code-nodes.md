# Code Nodes

Code nodes are used for deterministic, algorithmic tasks that don't need the flexibility of an agent node. They are not connected to LLMs — instead, they run Python code.

If you want the node to do any creative work (i.e., writing) or parsing largely unstructured data, use an agent node instead.

## Declaring inputs and outputs

When creating or editing a code node, declare its input and output schemas explicitly:

```json
[
  {"field": "code", "value": "def handler(context):\n    return {'greeting': f'Hello {context[\"name\"]}'}"},
  {"field": "inputSchema", "value": {"name": {"type": "string", "description": "Person name"}}},
  {"field": "outputSchema", "value": {"greeting": {"type": "string"}}}
]
```

Two schema formats are supported:

**Fields shorthand** (for flat schemas — renders as a table in the UI):
```json
{"name": {"type": "string"}, "count": {"type": "number"}, "active": {"type": "boolean"}}
```

**Full JSONSchema** (for nested/complex schemas):
```json
{
  "type": "object",
  "properties": {
    "companies": {
      "type": "array",
      "items": {"type": "object", "properties": {"name": {"type": "string"}, "revenue": {"type": "number"}}}
    }
  },
  "required": ["companies"]
}
```

## Python handler pattern

You must write a complete Python handler function that:

1. Accepts a `context` parameter containing inputs and utility methods
2. Processes the inputs according to the user's requirements
3. Returns a dictionary with string keys and appropriate values (REQUIRED)
4. Handles errors gracefully
5. Follows Python best practices

## Accessing inputs

Inputs are available as a dict on the `context` parameter:

- `context["key"]` — direct access (raises KeyError if missing)
- `context.get("key", default)` — with default value
- `context.key` — dot-notation access (also works)

```python
def handler(context):
    name = context["name"]
    count = context.get("count", 0)
    return {"result": f"Processed {count} items for {name}"}
```

## Available context methods

- `context.transition_to("node_name")` — Explicitly transition to a specific node
- `context.call_tool(tool_ref, **inputs)` — Execute a Clay action tool attached to this code node
  - `tool_ref` can be an `actionKey` (e.g., `"find-email-from-name"`) or a tool ID (e.g., `"tct_abc123"`)
  - When writing NEW code, use the `actionKey` from the actions catalog — it will be auto-converted to the tool ID when saved
  - When reading EXISTING code nodes, you may see tool IDs (`tct_...`) — these are valid and should be kept as-is
  - The tool must be added to the node's `tools` array (in the same edit or a prior edit)
  - Returns the tool execution result (usually a dict)
- If you need to know a Clay action's input schema, see the `execute_clay_action` MCP tool description — it documents the discovery surface for input schemas on this workspace.

## Return value

The handler function MUST return a dictionary with string keys. All returned data, as well as `print()` statements, are captured and readable by future nodes.

If `outputSchema` is set, the return value is validated against it.

Examples:

```python
return {"result": "success", "count": 42}
return {"name": user_name, "email": user_email, "valid": True}
return {"error": "Invalid input", "status": "failed"}
```

## Example — code node that finds emails using a tool

```python
# tools: [{ "toolType": "clay_action", "actionKey": "find-email-from-name",
#            "actionPackageId": "<packageId from actions catalog>" }]
def handler(context):
    result = context.call_tool(
        "find-email-from-name",
        full_name=context["name"],
        domain=context["domain"]
    )
    return {"email": result.get("email", ""), "confidence": result.get("confidence", 0)}
```

## Tips

- Access inputs via `context["key"]` or `context.get("key", default)`
- Always declare `inputSchema` when creating code nodes so upstream nodes know what to provide
- For logging and debugging, use standard Python `print()` statements
- Code can optionally explicitly transition to another node using `context.transition_to("node_name")`
- For map code handlers, use `context.item` or `context["input"]` to access the current entry
