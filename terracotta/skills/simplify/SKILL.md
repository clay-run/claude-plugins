---
name: simplify
description: Analyze a Clay workflow and suggest simplifications. Reduces unnecessary complexity, merges redundant nodes, and replaces LLM nodes with deterministic alternatives where possible.
---

# Simplify Workflow

Analyze the current workflow and suggest concrete simplifications to reduce complexity, improve reliability, and lower costs.

## Process

1. **Read the workflow** using `read_workflow` (full mode) to get all node details
2. **Analyze each node** against the simplification checklist below
3. **Present findings** as a prioritized list of suggestions with specific changes
4. **Apply changes** after user approval, using `edit_node` for efficiency

## Simplification Checklist

### Replace LLM nodes with code nodes
Regular (LLM) nodes cost an LLM call per execution. Many can be replaced with deterministic code:

- **Data transformation** — extracting fields, reformatting JSON, string manipulation → code node
- **Simple routing** — if the decision can be expressed as rules on data fields → conditional node (rules mode)
- **Calculations** — math, aggregation, counting → code node
- **Template filling** — constructing strings from known fields → code node

**Ask:** "Does this node require creative reasoning, or could a Python function do the same thing?"

### Merge sequential nodes
Two adjacent nodes can often be combined into one if:
- Node A passes all its output to Node B, and Node B doesn't add new tools or branching
- Both nodes use the same model and could be described in a single prompt
- One node just reformats the other's output

**Ask:** "Would combining these prompts into one still produce the same result?"

### Remove unnecessary nodes
- Nodes that just pass data through without transformation
- Conditional nodes with only one possible outcome

### Use inputRefs instead of LLM variable filling
When a downstream node needs specific typed data from an upstream node:
- Add `outputSchema` to the upstream node
- Add `inputSchema` + `inputRefs` to the downstream node
- This is more reliable than relying on the LLM to fill `{{variables}}`

### Simplify tool usage
- If a node has tools it never uses, remove them (reduces prompt size and cost)
- If a node calls one tool and passes the result, consider making it a code node with `context.call_tool()`

### Use code mode for conditional and map nodes
- Conditional nodes: prefer `rules` or `code` mode over `agentic` mode when the decision logic is expressible programmatically
- Map nodes: prefer `code` mode over `agent` mode when processing is deterministic

## Output Format

Present suggestions as:
1. **What to change** — specific node(s) affected
2. **Why** — what complexity or cost this removes
3. **How** — the concrete edit (new node type, merged prompt, code snippet)

After presenting all suggestions, ask the user which ones to apply. Then execute them using `edit_node` and run `validate_workflow` with `prettier=true`.
