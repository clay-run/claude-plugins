---
name: terracotta
description: This is the main skill that you should read to understand what Clay and Terracotta are, how to use the other skills & MCP tools, build automations and workflows in Clay, and more. You must always call it before using any of the Terracotta MCP tools.
---

# Terracotta Workflow Editor

You are an expert helping users build and edit Clay Terracotta workflows.

## Setup required

Anything that uses the `clay` CLI (running tests, searching actions, viewing snapshots, managing runs) requires Clay API key credentials. If `CLAY_API_KEY` is not set or `clay whoami` fails, tell the user to run `/setup` to configure their credentials. This only needs to be done once. The workspace is resolved from the key — there is no workspace id to set.

## Your capabilities

MCP Tool capabilities:

- Read workflows and nodes (`read`)
- Create, update, or delete nodes (`edit_node`)
- Validate workflows (`validate_workflow`)
- Get table schemas and query table data (`table`) — for detailed table help use `/clay-tables`
- Run any Clay action, either by calling it in Python or calling it directly one-off with `run_code` or `execute_clay_action`.

CLI capabilities (via the `clay` CLI):

- Start, poll, and inspect workflow runs (see `testing.md`)
- Pause runs and download files written by code nodes (`clay workflows runs pause` / `download-file`)
- Browse the Clay action catalog (`clay workflows actions`; see the `/discover-clay-actions` skill)
- Git-like version control for workflows — see past snapshots and restore to them (`clay workflows snapshots`; use the `snapshots` skill)

To find available Clay actions, integrations, and data providers, see the `execute_clay_action` MCP tool description — it documents the discovery surface available on this workspace.

## How Terracotta workflows work

Terracotta is a graph-based workflow builder. Nodes can have mapped inputs and outputs, but can also grep over all past results of nodes to find the information it needs. Most nodes are either an agent node or a code node. You can perform tasks with either.

An agent node is an LLM loop with a prompt, a set of tools it can use, and a model. The prompts can include inputs, which you can add by including `{{variable_name}}` in a prompt. These variables are filled in at runtime by an LLM reading the output of the **immediately preceding node only** — not nodes further back in the graph. If you need data from a node that isn't the direct predecessor (e.g., data from 2+ hops back, or across a conditional), use `inputRefs` to wire it explicitly. When the LLM finishes the task defined in a node, it will run in-parallel all nodes it has an edge to.

A code node is defined in Python, with a timeout, (optionally) packages, Clay actions & enrichments (which it can call with an SDK), similar to a lambda function. It is defined with its code, and its inputs are defined either with `context.get_input('name of input')` or explicitly in JSON. The inputs to a code node are also filled in by an LLM from the immediately preceding node's output.

An enrich node (`nodeType: "tool"`) executes a single Clay action directly — no code or LLM needed. It's the simplest way to call a data enrichment action (e.g., find an email, look up company info). Configure it with one tool in the `tools` field and it runs that action automatically, passing inputs from upstream nodes.

There are a few special types of nodes that change the flow of the graph.

- Conditional nodes take a condition, a prompt, or code, and only run one of their outgoing edges. Conditional code nodes can explicitly call `context.transition_to('node ID')`, while regular code nodes cannot.
- Map nodes take a list and can be either an agent or code node. The prompt or code is executed once per item in the list.
- Reduce and collect nodes work with the outputs of a map node.

Every workflow has a **trigger node** as its first node — this defines how the workflow gets launched (audience segment, webhook, Clay table, or manual). The trigger node's outputs become the inputs for the nodes it connects to. Workflows can be run either one-off, in bulk (as a batch), or set up as a stream (where you create a webhook or Clay table that automatically adds new runs to the workflow).

**Leaf nodes** (nodes with no downstream connections) are automatically treated as terminal — you do not need to mark them. There can be multiple leaf nodes per workflow. They have an explicit output schema, and when runs are created, users can define on a per-run basis where the output goes. Different runs in the same workflow can end with different output actions.

See `code-nodes.md` for more details on using the Clay SDK and how to call tools inside code nodes. Also use this before using `run_code` if needed.

## Creating your first workflow

```bash
# List all workflows
clay workflows list

# Create a new workflow
clay workflows create --name "My Workflow"
```

`clay` is on your PATH. For more commands to start, run, and inspect workflows, see `testing.md`.

## Required fields for new nodes

- `name`, `nodeType`, `incomingEdges`
- For agent nodes: `agentName`, `agentPrompt`, optionally `tools`, `agentModel`
- Any node can optionally have an `outputSchema` to define structured outputs
- For code nodes: `code`
- For enrich nodes (`nodeType: "tool"`): `tools` (exactly 1 tool)
- For conditional nodes: `conditionalConfig` (with `mode: "rules"`, `"agentic"`, or `"code"`)

Constraints:
- When creating or modifying agent nodes, always send `agentName`, `agentPrompt`, and `agentModel` together in a single `edit_node` call. Sending them separately can result in an agent with a blank prompt.

See `code-nodes.md` and `conditional-nodes.md` for details on specific node types.

## Adding tools to nodes

The `tools` field works on enrich nodes (`nodeType: "tool"`) and code nodes. Agent nodes get tools via their Claygent configuration, not this field.

- **Enrich nodes**: Set exactly 1 tool. The node executes that action directly with inputs from upstream.
- **Code nodes**: Add tools so the code can call them via `context.call_tool(actionKey, **inputs)`.

Use the `tools` field with an array:

```json
[
  {
    "toolType": "clay_action",
    "actionKey": "<actionKey>",
    "actionPackageId": "<packageId>"
  }
]
```

To find available actions on this workspace, see the `execute_clay_action` MCP tool description — it documents which actions are permitted and how to look up their schemas.

See `clay-actions.md` for full details on adding and using Clay actions.

## Input references (inputRefs)

Nodes can explicitly reference structured output from any upstream node using `inputRefs`. This enables typed data passing between nodes (as opposed to relying on the LLM to fill in `{{variables}}`).

**When to use inputRefs (required):**
- The data comes from a node that is NOT the immediate predecessor (e.g., separated by a conditional)
- The data is a non-string type (number, boolean) that must be preserved exactly
- You want deterministic, reliable data passing instead of LLM-mediated filling

**When LLM variable filling is fine:**
- The data comes from the immediately preceding node
- The data is a string
- Approximate filling is acceptable

Setting up inputRefs:

Use the `inputRefs` field on `edit_node` to wire upstream outputs to this node's inputs:

```json
{
  "inputRefs": {
    "score": { "sourceNodeId": "wfn_abc", "path": "$.lead_score" },
    "name": { "sourceNodeId": "wfn_abc", "path": "$.contact_name" }
  }
}
```

The path uses JSONPath syntax (e.g., `$.fieldName`, `$.nested.field`).

- Code nodes access referenced inputs via `context.get_input("localInputName")`
- Regular (LLM) nodes can access referenced inputs as `{{localInputName}}` in their prompt

## Testing & exploration

- `execute_clay_action`: Run any Clay action to see its output before using it in a workflow
- `run_code`: Run Python code in a sandbox to test logic before putting it in code nodes

In general, you should always use these before building workflows. See `testing.md` for CLI commands reference.

## Recommended workflow for building

1. Find relevant actions for the user's request (see the `execute_clay_action` MCP tool description for discovery)
2. Test actions with `execute_clay_action` to understand output format
3. Prototype logic with `run_code`
4. Build workflow nodes
5. Start a test run after

## Best practices

1. Always read the workflow first to understand current state
2. Create nodes sequentially with `edit_node`, using `incomingEdges` to wire them to existing nodes
3. Validate after making changes
4. After building or significantly modifying a workflow, run `validate_workflow` with `prettier=true` to auto-layout nodes
5. Use string-replace mode for small edits to prompts/code
6. Provide clear explanations of your changes
7. When adding tools for data enrichment (e.g., finding emails, company info, phone numbers), try adding 2-3 alternative tools that can provide similar data as fallbacks
8. After completing a workflow build, suggest running test runs to verify it works correctly
9. Always validate workflows before declaring them complete
11. After making a mistake or if the user asks to undo, use `/snapshots` to revert to the previous snapshot

## Reference docs

For detailed information on specific topics, read these files from this skill's directory:

### Core concepts

- `data-passing.md` — **How data flows between nodes.** Covers all five methods: LLM variable filling (`{{variables}}`), explicit inputRefs with outputSchema, context memory search (semantic search over prior outputs), stdout indexing for code nodes, and edge metadata. Includes guidance on when to use each method
- `code-nodes.md` — How to write Python handler functions for code nodes: `context.get_input()`, `context.call_tool()`, `context.transition_to()`, return dict format, and examples

### Patterns & features

- `working-with-lists.md` — **Processing lists of items.** Two approaches: (1) batch workflow runs (one independent run per item via batches), and (2) map/reduce/collect nodes for parallel processing within a single run. Covers map code mode vs agent mode, reduce grouping by key, collect gather vs stream mode, and how to wire the pipeline together
- `conditional-nodes.md` — The specific schema of conditional configurations `conditionalConfig`, and examples of simple and compound conditions

### Tools & testing

- `clay-actions.md` — Adding Clay action tools to nodes. The `tools` field format, action properties, workspace configuration, and calling actions from code nodes via `context.call_tool()`
- `testing.md` — `clay` CLI commands for running and inspecting workflow test runs: start, get (status/summary), steps, polling for completion, pause/resume, download files from sandbox

### Related skills

- `/simplify` — Analyze a workflow and suggest simplifications: replace LLM nodes with code, merge redundant nodes, remove unnecessary complexity
- `/optimize-credits` — Analyze a workflow for cost optimization: reduce LLM calls, use cheaper models for simple tasks, pre-map action parameters, add conditional routing to skip expensive steps
- `/snapshots` — View workflow version history, diff between versions, restore to a previous state, undo edits
