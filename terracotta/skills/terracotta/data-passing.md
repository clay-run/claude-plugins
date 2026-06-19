# Data Passing in Clay Workflows

Two methods are available for moving data between nodes.

## Method 1: LLM Variable Filling (default)

When an agent node finishes, it picks one outgoing edge and fills in `{{variable_name}}` placeholders in the downstream node's prompt based on its understanding of the work it just did.

**Example:**

```
Node A prompt: "Research the company {{company_name}} and determine their industry."
Node B prompt: "Write an email to {{contact_name}} at {{company_name}} in the {{industry}} industry."
```

When Node A transitions to Node B, the LLM fills in `company_name`, `contact_name`, and `industry` based on its research.

**Best for:** Flexible, prompt-to-prompt text. Simple workflows where the immediately preceding agent node has all the context needed.

**Limitations:**
- Non-deterministic — the LLM may fill values slightly differently each run
- No type validation — everything is a string
- Only works node-to-immediate-successor — for data from 2+ hops back, use `inputRefs`

## Method 2: Explicit `inputRefs` (typed, deterministic)

Use `inputRefs` when:

- The data must be exact (a number, a boolean, a specific structured field)
- The data comes from a node that is NOT the immediate predecessor
- You want guarantees instead of LLM-mediated filling

### Setup

The **upstream node** declares an `outputSchema` describing its structured output:

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

The **downstream node** declares `inputRefs` mapping each input to a JSONPath into the upstream output:

```json
{
  "nodeConfig": {
    "inputRefs": {
      "company_name": { "sourceNodeId": "wfn_upstream", "path": "$.company_name" },
      "score":        { "sourceNodeId": "wfn_upstream", "path": "$.score" }
    }
  }
}
```

### Accessing referenced inputs

- **Agent nodes:** `{{company_name}}` in the prompt — populated from the resolved inputRef, not LLM-filled
- **Enrich nodes:** Inputs are passed straight to the underlying Clay action

### JSONPath syntax

- `$.fieldName` — top-level field
- `$.nested.field` — nested
- `$.array[0].name` — array element

## Choosing between the two

| Scenario | Method |
|----------|--------|
| Free-form text from the immediately preceding agent | LLM variable filling |
| Numeric scores, IDs, boolean flags | `inputRefs` |
| Data from 2+ hops back (or across branches once we have them) | `inputRefs` |
| Inputs into an enrich (tool) node | `inputRefs` whenever the upstream output is structured |

You can mix: use `inputRefs` for the critical typed fields and let the LLM fill in supplementary text variables in the same prompt.
