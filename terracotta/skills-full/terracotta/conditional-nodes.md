# Conditional Nodes

Conditional nodes (nodeType: `"conditional"`) route execution to exactly one downstream node. They do NOT process data — they decide where to go next.

Use the `conditionalConfig` field (not raw `nodeConfig`) to configure conditional nodes.

Conditional nodes are typically NOT initial or terminal. They sit between other nodes to route flow.

## Configuration

Three modes are supported:

### Rules mode (`mode: "rules"`)

Data-driven filter conditions evaluated top-to-bottom (first match wins).

Variables for rules come from the node's `inputSchema` + `inputRefs` (in `nodeConfig`), NOT from `rulesConfig`.
At runtime, `inputRefs` are resolved into the node's `stepInputs`, and rule conditions evaluate against those inputs.

```json
{
  "conditionalConfig": {
    "mode": "rules",
    "rulesConfig": {
      "rules": [
        {
          "condition": {
            "type": "GroupOp",
            "combinationMode": "And",
            "items": [
              { "type": "BinOp", "dataPath": ["score"], "operator": "GreaterThan", "value": 80 }
            ]
          },
          "targetNodeId": "wfn_high"
        },
        {
          "condition": {
            "type": "GroupOp",
            "combinationMode": "And",
            "items": [
              { "type": "BinOp", "dataPath": ["score"], "operator": "GreaterThan", "value": 50 }
            ]
          },
          "targetNodeId": "wfn_medium"
        }
      ],
      "defaultTargetNodeId": "wfn_low"
    }
  },
  "nodeConfig": {
    "inputSchema": { "type": "object", "properties": { "score": { "type": "number" } } },
    "inputRefs": { "score": { "sourceNodeId": "wfn_abc", "path": "$.score" } }
  }
}
```

### Agentic mode (`mode: "agentic"`)

An LLM evaluates the input and picks a route. Define named routes and an optional prompt to guide the decision. The LLM chooses one route based on the context.

```json
{
  "conditionalConfig": {
    "mode": "agentic",
    "agentConfig": {
      "prompt": "Decide which route to take based on the input data.",
      "routes": [
        { "id": "route_1", "name": "High priority" },
        { "id": "route_2", "name": "Low priority" }
      ]
    }
  }
}
```

Use `incomingEdges` with `routeId` on downstream nodes to wire edges to specific agentic routes.

### Code mode (`mode: "code"`)

Python code decides the target node. The code must call `context.transition_to("node_name", "condition_label")` to select the next node. The `condition_label` should describe the condition (e.g., `"is_urgent"`, `"score_above_80"`), not the target node name. Each unique `condition_label` creates a labeled output port on the node.

```json
{
  "conditionalConfig": {
    "mode": "code",
    "codeTransitions": [
      { "id": "is_urgent", "name": "Category is urgent" },
      { "id": "not_urgent", "name": "Category is not urgent" }
    ]
  }
}
```

Example code:
```python
def handler(context):
    if context.get_input('category') == 'urgent':
        context.transition_to('Handle Urgent', 'is_urgent')
    else:
        context.transition_to('Handle Normal', 'not_urgent')
```

The code is stored in the node's `code` field (via `scriptVersion`). `codeTransitions` are auto-populated from code analysis when the code is saved. See `code-nodes.md` for Python handler patterns.

## ConditionalExpressionGroup format

A condition is a tree of filter expressions. The top level is always a `GroupOp`.

### GroupOp (group of conditions combined with And/Or)

```json
{ "type": "GroupOp", "combinationMode": "And" | "Or", "items": [...expressions] }
```

### BinOp (binary comparison on a variable)

```json
{ "type": "BinOp", "dataPath": ["variableName"], "operator": "<op>", "value": <compareValue> }
```

- **dataPath**: Array path to the variable. These reference fields from the node's resolved inputs (defined via `inputSchema` + `inputRefs`). For top-level input fields, use `["fieldName"]`. For nested fields, use `["fieldName", "nestedField"]`.
- **operator**: One of: `"Equal"`, `"NotEqual"`, `"GreaterThan"`, `"GreaterThanOrEqual"`, `"LessThan"`, `"LessThanOrEqual"`, `"Contain"`, `"NotContain"`, `"StartsWith"`, `"NotStartsWith"`, `"EndsWith"`, `"NotEndsWith"`, `"Empty"`, `"NotEmpty"`, `"True"`, `"False"`
- **value**: The comparison value. Not needed for `"Empty"`, `"NotEmpty"`, `"True"`, `"False"` operators.

## Examples

Simple equality:
```json
{ "type": "BinOp", "dataPath": ["status"], "operator": "Equal", "value": "active" }
```

Number compare:
```json
{ "type": "BinOp", "dataPath": ["score"], "operator": "GreaterThan", "value": 80 }
```

String contains:
```json
{ "type": "BinOp", "dataPath": ["email"], "operator": "Contain", "value": "@gmail.com" }
```

Is not empty:
```json
{ "type": "BinOp", "dataPath": ["name"], "operator": "NotEmpty" }
```

Combined (AND):
```json
{
  "type": "GroupOp",
  "combinationMode": "And",
  "items": [
    { "type": "BinOp", "dataPath": ["score"], "operator": "GreaterThan", "value": 50 },
    { "type": "BinOp", "dataPath": ["status"], "operator": "Equal", "value": "verified" }
  ]
}
```
