# Data Passing & Referencing in Terracotta

Terracotta workflows support multiple ways to pass data between nodes. Understanding when to use each method is key to building effective workflows.

## Method 1: LLM Variable Filling (Default)

The simplest method. When a regular (LLM) node finishes, it picks the next edge and fills in `{{variable_name}}` placeholders in the downstream node's prompt.

**How it works:**
- Define variables in a node's prompt using `{{variable_name}}` syntax
- The upstream LLM reads the downstream node's prompt and fills in the variables
- Variables are filled based on the LLM's understanding of the work done so far

**Example:**
```
Node A prompt: "Research the company {{company_name}} and determine their industry."
Node B prompt: "Write an email to {{contact_name}} at {{company_name}} in the {{industry}} industry."
```

When Node A transitions to Node B, the LLM fills in `company_name`, `contact_name`, and `industry` based on its research.

**Best for:** Flexible, creative data passing where the LLM decides what to fill in. Simple workflows where you don't need strict typing.

**Limitations:** Non-deterministic. The LLM might fill variables differently each time. No type validation.

## Method 2: Explicit Input/Output Refs (Structured)

For typed, reliable data passing between nodes. Uses `inputRefs` and `outputSchema` in `nodeConfig`.

**How it works:**
1. **Upstream node** declares an `outputSchema` — a JSON Schema describing its structured output
2. **Downstream node** declares an `inputSchema` and `inputRefs` mapping each input to an upstream output field

**Setting up the upstream node:**
```json
{
  "nodeConfig": {
    "outputSchema": {
      "type": "object",
      "properties": {
        "company_name": { "type": "string" },
        "industry": { "type": "string" },
        "score": { "type": "number" }
      }
    }
  }
}
```

**Setting up the downstream node:**
```json
{
  "nodeConfig": {
    "inputSchema": {
      "type": "object",
      "properties": {
        "company_name": { "type": "string" },
        "score": { "type": "number" }
      }
    },
    "inputRefs": {
      "company_name": {
        "sourceNodeId": "wfn_upstream_node_id",
        "path": "$.company_name"
      },
      "score": {
        "sourceNodeId": "wfn_upstream_node_id",
        "path": "$.score"
      }
    }
  }
}
```

**Accessing resolved inputs:**
- **Code nodes:** `context.get_input("company_name")`
- **Regular nodes:** `{{company_name}}` in the prompt (populated from the resolved inputRef, not from LLM filling)

**Path syntax:** JSONPath — `$.fieldName`, `$.nested.field`, `$.array[0].name`

**Best for:** Deterministic data passing. When you need type safety, when downstream nodes need specific fields from specific upstream nodes (not just the immediate predecessor).

## Method 3: Context Memory Search (Semantic)

Every node can search over all previous tool results and outputs from earlier nodes in the run. This is powered by a vector search index.

**How it works:**
- As each node executes, its outputs and tool results are indexed as "context droplets"
- Any downstream node can search this index to find relevant previous work
- The LLM in regular nodes does this automatically when it needs information
- Search considers relevance, recency, and content type trust scores

**Content types indexed:**
- `tool_result` — outputs from Clay action tool calls
- `code_output` — return values from code nodes
- `handoff_boundary` — edge transition context
- `user_prompt` — initial user inputs

**Best for:** When a node needs to reference work done several steps ago, or when you want the LLM to find the most relevant prior context automatically. Useful for complex workflows where not all data paths are known upfront.

## Method 4: Stdout Indexing for Code Nodes

Code nodes can use `print()` statements to output text that gets indexed and is searchable by downstream nodes.

**How it works:**
- Code nodes with `shouldIndexStdout: true` (the default) capture all `print()` output
- This output is indexed alongside structured outputs
- Downstream regular (LLM) nodes can access this via context memory search
- Useful for logging intermediate results or debug information that downstream nodes might need

**Example:**
```python
def handler(context):
    companies = context.get_input("companies")
    print(f"Processing {len(companies)} companies")
    for c in companies:
        print(f"Company: {c['name']}, Domain: {c.get('domain', 'unknown')}")
    return {"count": len(companies), "names": [c['name'] for c in companies]}
```

The `print()` output becomes searchable context for later nodes.

## Method 5: Edge Metadata & Routing Criteria

Edges between nodes can carry routing criteria and handoff configuration that influence how context is passed.

**How it works:**
- Each outgoing edge can have `routingCriteria` — a description of when to take this edge
- Edges also carry `handoffConfig` controlling how much context the next node receives

**Best for:** Controlling the flow of execution and what context each branch sees.

## Choosing the Right Method

| Scenario | Recommended method |
|----------|-------------------|
| Simple prompt-to-prompt data flow | LLM variable filling (`{{variables}}`) |
| Typed data from a specific upstream node | inputRefs with outputSchema |
| Code node needs data from a specific node | inputRefs with `context.get_input()` |
| Node needs to find relevant prior work | Context memory search (automatic for LLM nodes) |
| Debugging / logging intermediate results | `print()` with stdout indexing |
| Conditional routing based on data | inputRefs on conditional node + rules |

## Tips

- **Combine methods:** You can use inputRefs for critical typed data and let the LLM use context search for supplementary information in the same node.
- **inputRefs are precise:** They pull a specific field from a specific node. Use them when you need guarantees.
- **Context search is flexible:** It works across the entire execution history. Use it when you need broad access to prior work.
- **Code nodes are best with inputRefs:** Since code is deterministic, give it explicit typed inputs rather than relying on LLM-mediated variable filling.
