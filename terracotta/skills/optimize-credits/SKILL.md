---
name: optimize-credits
description: Analyze a Clay workflow to reduce credit and LLM costs. Identifies expensive patterns and suggests cheaper alternatives.
---

# Optimize Credits

Analyze the current workflow and suggest changes to reduce credit consumption and LLM costs without sacrificing quality.

## Process

1. **Read the workflow** using `read_workflow` (full mode) to get all node details
2. **Identify cost drivers** — LLM calls, Clay action usage, model selection
3. **Present optimization opportunities** with estimated impact
4. **Apply changes** after user approval

## Cost Drivers in Clay Workflows

### LLM Calls (biggest cost driver)
Every regular (LLM) node makes at least one LLM call per execution. More capable models cost more.

**Optimizations:**
- **Replace with code nodes:** If a node does deterministic work (data transformation, filtering, formatting), replace it with a code node — zero LLM cost
- **Use cheaper models:** Simple tasks (parameter extraction, basic classification) can use smaller/faster models. Reserve powerful models for complex reasoning
- **Merge nodes:** Two sequential LLM nodes doing related work can often be one node with a combined prompt — cuts LLM calls in half
- **Pre-map action parameters:** When a node calls a Clay action, configure `static` or `reference` input mappings for parameters that don't need LLM inference. If ALL parameters are pre-mapped, the LLM parameter mapping call is skipped entirely

### Clay Action Credits
Each Clay action execution consumes credits based on the action's pricing tier.

**Optimizations:**
- **Avoid redundant enrichments:** If the same data was already fetched in an earlier node, reference it via inputRefs instead of calling the action again
- **Use conditional routing:** Skip expensive enrichments for items that don't need them (e.g., skip company research for companies you already have data on)
- **Choose cheaper alternatives:** Some actions have cheaper equivalents. See the `execute_clay_action` MCP tool description for the discovery surface to compare options
- **Fail fast in map nodes:** Use `failureMode: "fail_fast"` in collect nodes if one failure means the whole batch is invalid — avoids processing remaining items

### Model Selection
Different models have different cost/capability profiles.

**Optimizations:**
- **Match model to task complexity:**
  - Simple extraction/classification → use a smaller, faster model
  - Complex reasoning, multi-step analysis → use a more capable model
  - Creative writing, nuanced decisions → use the most capable model
- **Downgrade where safe:** Review each regular node's prompt. If the task is straightforward, try a cheaper model

### Map Node Efficiency
Map nodes multiply costs by the number of items processed.

**Optimizations:**
- **Code mode over agent mode:** If map processing is deterministic, use code mode (zero LLM cost per item)
- **Filter before mapping:** Add a code node before the map to filter out items that don't need processing
- **Reduce chunk size:** Smaller chunks mean less wasted work if a chunk fails
- **Use reduce to aggregate:** Instead of processing all items individually and collecting, use reduce to aggregate by key — fewer downstream processing steps

### Conditional Routing
Use conditional nodes to skip expensive branches for items that don't need them.

**Example:**
```
Instead of:  [All items] → [Expensive enrichment] → [Process]
Do this:     [All items] → [Conditional: has data?] → Yes → [Process]
                                                    → No  → [Enrichment] → [Process]
```

## Analysis Output Format

For each optimization opportunity, present:

1. **Node(s) affected** — which nodes to change
2. **Current cost pattern** — what's expensive and why
3. **Suggested change** — specific modification
4. **Estimated impact** — qualitative (high/medium/low) cost reduction
5. **Risk** — any quality trade-offs

Prioritize suggestions by impact (highest savings first).

After presenting all suggestions, ask the user which to apply. Execute using `edit_node` and validate with `validate_workflow`.
