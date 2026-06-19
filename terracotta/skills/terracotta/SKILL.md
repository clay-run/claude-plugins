---
name: terracotta
description: This is the main skill that you should read to understand what Clay workflows are, how to use the other skills and MCP tools, and how to build automations and workflows in Clay. Always read it before using any of the workflow MCP tools.
---

# Clay Workflow Editor

You are an expert helping users build and edit Clay workflows.

You build workflows out of two kinds of nodes:

- **Claygent (agent) nodes** — LLM loops with prompts. The default building block for reasoning, drafting, summarizing, and classifying.
- **Enrich nodes** — single-Clay-action nodes for data lookups (find an email, look up a company, etc.). Pick the action from the workspace's available action set.

You should also understand **triggers** — how a workflow gets launched (audience segments, webhooks, Clay tables). Triggers are configured by the user in the Clay Workflows UI; you don't create or edit them via MCP, but you should be aware of them so you can design workflows around how they'll be invoked.

## Setup required

Anything that uses the `clay` CLI (running tests, searching actions, viewing snapshots, managing runs) requires Clay API key credentials. If `CLAY_API_KEY` is not set or `clay whoami` fails, tell the user to run `/setup`. This only needs to be done once. The workspace is resolved from the key — there is no workspace id to set.

## Your capabilities

MCP tools:

- **Read workflows and nodes** (`read`)
- **Create, update, or delete nodes** (`edit_node`) — for `agent` and `tool` (enrich) node types
- **Validate workflows** (`validate_workflow`)
- **Execute Clay actions one-off** (`execute_clay_action`)

CLI capabilities (via the `clay` CLI):

- Start, poll, and inspect workflow runs (see `testing.md`)
- Browse the Clay action catalog (`clay workflows actions`; see the `/discover-clay-actions` skill)
- Snapshots / version history (`clay workflows snapshots`; use the `/snapshots` skill)

## How a Clay workflow is structured

Clay workflows are graph-based. A workflow is a directed graph of nodes connected by edges. The two node types are:

### Agent nodes (Claygents)

An agent node is an LLM loop with a prompt and a model. When the run reaches the node, the LLM:

1. Reads the prompt (with `{{variable}}` placeholders filled in from the immediately preceding node's output)
2. Does whatever the prompt asks
3. Picks one of its outgoing edges and transitions to that next node, filling that node's variables in the process

Agent nodes can have tools attached via the **Claygent configuration** (this is separate from the `tools` field that enrich nodes use). Unfortunately, you cannot create Claygents with tools directly - but the user can do this in the UI themselves if they edit the Claygent directly. Treat agent nodes as Claygent prompts that do reasoning, summarization, drafting, classification, etc., and let enrich nodes do the data lookup work.

### Enrich nodes (`nodeType: "tool"`)

An enrich node executes a single Clay action directly — no LLM reasoning. Configure it with exactly one tool in the `tools` field and it runs that action with inputs filled from upstream.

Ask the user which Clay action they want to use. To learn an action's exact input/output shape, run it once with `execute_clay_action` before wiring it into the node — that confirms both that the workspace has the action available and what fields it expects.

### Trigger nodes and leaf nodes

- Every workflow has a **trigger node** as its first node — this defines how the workflow gets launched (audience segment, webhook, Clay table, or manual). The trigger node's outputs become the inputs for the nodes it connects to.
- **Leaf nodes** are nodes with no downstream connections. They are automatically treated as terminal — you do not need to mark them.

## Triggers — how workflows get launched

A workflow doesn't run by itself. It runs because a **trigger** kicks off a run. Triggers are configured by the user in the Clay Workflows UI; you do not create or edit triggers via MCP. Be aware of them so you can explain workflow shape:

- **Audience segment trigger** — every record in a Clay audience segment becomes a run input. Useful for batch-style enrichment over a known list.
- **Webhook trigger** — an external system POSTs to a URL and each request becomes a run.
- **Clay table trigger** — new rows added to a specific Clay table create runs automatically.
- **One-off / batch test runs** — the user (or the `clay` CLI) launches a single run or a batch for testing.

When designing a workflow, ask the user how the workflow will be triggered, because that determines:

- What the trigger node's outputs look like (a row from a table? a webhook body? an audience record?)
- Whether the workflow should be optimized for one-at-a-time or high-volume runs
- Whether leaf node output goes back to a Clay table, a webhook response, etc.

If the user hasn't picked a trigger, recommend the simplest option that fits their use case and tell them where to configure it in the UI after the workflow is built.

## Required fields for new nodes

For every node:

- `name`, `nodeType`, `incomingEdges`

For agent nodes (`nodeType: "agent"`):

- `agentName`, `agentPrompt`, `agentModel`
- Always send `agentName`, `agentPrompt`, and `agentModel` together in a single `edit_node` call. Sending them separately can result in an agent with a blank prompt.

For enrich nodes (`nodeType: "tool"`):

- `tools` — exactly one entry. The `actionKey` is the Clay action you want to invoke (confirm it via `execute_clay_action` first)

## Adding tools to enrich nodes

Use the `tools` field with a single-element array:

```json
[
  {
    "toolType": "clay_action",
    "actionKey": "<actionKey>",
    "actionPackageId": "<packageId>"
  }
]
```

Or reuse a workspace-configured tool by id:

```json
[{ "toolType": "clay_action", "toolId": "tct_abc123" }]
```

The user can tell you which `actionKey` and `actionPackageId` to use, or which existing `toolId` to reuse. Test the action with `execute_clay_action` before adding it to confirm it works on this workspace and to see its real output shape.

## Passing data between nodes

Two methods are available:

### `{{variable}}` filling (default)

Put `{{variable_name}}` in an agent node's prompt, and the upstream LLM fills it in when transitioning. Works node-to-immediate-successor only. Best for free-form text.

### `inputRefs` (typed, deterministic)

For data that needs to be exact (numbers, structured fields, data from 2+ hops back), declare an `outputSchema` on the upstream node and `inputRefs` on the downstream node:

```json
{
  "inputRefs": {
    "company_name": {
      "sourceNodeId": "wfn_upstream",
      "path": "$.company_name"
    },
    "score": { "sourceNodeId": "wfn_upstream", "path": "$.score" }
  }
}
```

Path syntax is JSONPath (`$.field`, `$.nested.field`). Agent nodes access referenced inputs as `{{company_name}}` in the prompt (populated from the ref, not LLM-filled).

See `data-passing.md` for a fuller reference.

## Recommended workflow for building

1. Ask the user what trigger they'll use (or recommend one) so you understand the initial node's inputs
2. Ask the user which Clay action(s) they want each enrich node to call, then test them with `execute_clay_action` to confirm output shape before wiring
3. Build the workflow node-by-node with `edit_node`, wiring `incomingEdges` as you go
4. Run `validate_workflow` with `prettier=true` to auto-layout and catch issues
5. Suggest the user kicks off a test run

## Best practices

1. Always read the workflow first to understand current state before editing
2. Create nodes sequentially with `edit_node`, using `incomingEdges` to wire them to existing nodes
3. Validate after making changes
4. After building or significantly modifying a workflow, run `validate_workflow` with `prettier=true`
5. Use string-replace mode for small edits to prompts
6. When adding enrichment tools, try 2-3 alternative actions as fallbacks if the primary one might miss
7. After completing a workflow, suggest a test run
8. If you make a mistake or the user asks to undo, use `/snapshots` to revert

## Reference docs in this skill

- `data-passing.md` — How `{{variables}}` and `inputRefs` work in detail
- `testing.md` — `clay` CLI commands for running and inspecting workflow runs
- `clay <command> --help` — Per-command JSON shape, flags, and error codes

## Related skills

- `/snapshots` — View workflow version history, diff between versions, restore a previous state
