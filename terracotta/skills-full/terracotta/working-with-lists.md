# Working with Lists in Terracotta

There are two approaches to processing lists of items in Terracotta:

1. **Batch workflow runs** — start a separate, independent workflow run for each item (via CSV upload or API)
2. **Map/Reduce/Collect nodes** — process a list within a single workflow run using parallel processing primitives

## Approach 1: Batch Workflow Runs

Start one independent workflow run per item. Each run is fully isolated with its own inputs.

**When to use:**
- Each item needs the full workflow (all nodes, all branching logic)
- Items are completely independent (no aggregation needed)
- You want separate run logs per item
- Processing a CSV of leads, companies, etc.

**How it works:**
- Upload a CSV file via the batch run endpoint
- Each CSV row becomes one workflow run
- CSV column headers map to the workflow's initial `{{variables}}`
- Runs execute independently and can be monitored separately

**CLI example:**
```bash
# Start one run per item (input JSON on stdin via --input -)
echo '{"company_name": "Acme Corp"}' | clay workflows runs start wf_abc --input -
```

## Approach 2: Map/Reduce/Collect Nodes

Process a list of items within a single workflow run. Use this when you need to iterate over an array, optionally aggregate results, and continue the workflow with the collected output.

**Pipeline patterns:**
```
map → (next node)                (process each item, auto-gather results)
map → reduce → (next node)       (process items, group by key, auto-gather)
```

> **Note:** Map and Reduce nodes have built-in auto-gather that automatically collects results.

### Map Node (nodeType: "map")

Fans out over an array, processing each item independently. Supports three modes:

#### How the map node finds its input array

The map node resolves its input array in this order:

1. **`inputRefs.entries`** (preferred) — if the map node has an `inputRefs` with key `"entries"` AND an `inputSchema`, it resolves the referenced upstream output at the given path.
2. **Fallback: `entries` key in step inputs** — if no inputRefs are configured, the map node looks for a literal `entries` key in the upstream node's output.

**Important:** The inputRef key MUST be `"entries"` — this is the hardcoded key the map node looks for. Also, `inputSchema` must be set on the map config for inputRefs resolution to work.

Example: upstream code node outputs `{"products": [...]}`, map node needs:
```json
{
  "nodeType": "map",
  "inputRefs": {
    "entries": {
      "sourceNodeId": "wfn_upstream_node_id",
      "path": "products"
    }
  },
  "mapConfig": {
    "mode": "code",
    "inputSchema": { "type": "object", "properties": { "name": { "type": "string" } } }
  },
  "code": "def handler(context):\n    return context['input']"
}
```

If you skip inputRefs, the upstream node must output the array at the `entries` key directly (e.g., `return {"entries": [...]}`).


#### Code mode
Runs Python code per item. Fast, deterministic, no LLM cost.

> **Important:** Map and reduce node code is set via the top-level `code` field — the same field used by regular code nodes. It is NOT nested inside `mapConfig` or `reduceConfig`. When creating a map/reduce node, send `code` as a separate field update alongside `mapConfig`/`reduceConfig`.

Simple map (no reduce):
```json
{
  "nodeType": "map",
  "mapConfig": {
    "mode": "code",
    "inputSchema": { "type": "object", "properties": { "name": { "type": "string" } } },
    "maxConcurrency": 10
  },
  "code": "def handler(context):\n    item = context['input']\n    return {'name': item['name'].upper(), 'domain': item.get('domain', '')}"
}
```

Map with key for downstream reduce:
```json
{
  "nodeType": "map",
  "mapConfig": {
    "mode": "code",
    "inputSchema": { "type": "object", "properties": { "name": { "type": "string" }, "category": { "type": "string" } } },
    "outputKey": true,
    "keyExpressionPath": "category",
    "maxConcurrency": 10
  },
  "code": "def handler(context):\n    item = context['input']\n    return {'name': item['name'], 'category': item['category'], 'processed': True}"
}
```

**Python handler context in code mode:**
- `context['input']` — the current item from the array
- `context['index']` — position in the array (0-based)
- `context['key']` — grouping key (null unless `outputKey: true`)

#### Agent mode
Runs an LLM agent per item. Use when each item needs flexible reasoning, tool use, or creative output.

```json
{
  "nodeType": "map",
  "mapConfig": {
    "mode": "agent",
    "agentConfig": {
      "prompt": "Research this company and summarize their main product.",
      "modelId": "model-id",
      "outputSchema": { "type": "object", "properties": { "summary": { "type": "string" } } }
    },
    "maxChunkSize": 100,
    "maxSteps": 10
  }
}
```

#### Key map config fields

| Field | Purpose |
|-------|---------|
| `inputSchema` | JSON Schema describing each array item |
| `outputKey` | Set `true` to emit a key per item (required for reduce). **Must also set `keyExpressionPath`** |
| `keyExpressionPath` | JSONPath to extract grouping key from each output (e.g., `"key"` or `"$.category"`). **Required when `outputKey: true`** |
| `maxConcurrency` | Number of parallel workers (1-100) |
| `maxChunkSize` | Max entries per processing chunk (agent mode, default 100) |
| `maxRetries` | Retry attempts per chunk (0-10) |
| `gatherResults` | Whether to gather results into a combined output for downstream nodes (default: `true`). When false, downstream nodes will not have access to the combined results |
| `errorHandling` | How to handle entry failures during gathering: `"fail_node_on_any_error"` (default) or `"ignore_errors"` |
| `maxGatherSize` | Max entries to gather in memory (default: 10000, max: 100000) |

Use the `code` field with `mode: "replace"` for string-replace edits to the handler code.

### Reduce Node (nodeType: "reduce")

Groups map outputs by key and aggregates each group. Must receive input from a map node with `outputKey: true`.

```json
{
  "nodeType": "reduce",
  "reduceConfig": {
    "mode": "code",
    "batchSize": 100
  },
  "code": "def handler(context):\n    total = sum(e['output']['count'] for e in context['entries'])\n    return {'key': context['key'], 'total': total}"
}
```

**Python handler context:**
- `context['key']` — the grouping key
- `context['entries']` — array of entries, each with: `input`, `output`, `index`, `key`
- `context['count']` — number of entries in this group

#### Key reduce config fields

| Field | Purpose |
|-------|---------|
| `batchSize` | Number of keys to process per batch (1-10000) |
| `gatherResults` | Whether to gather results into a combined output for downstream nodes (default: `true`). When false, downstream nodes will not have access to the combined results |
| `errorHandling` | How to handle entry failures during gathering: `"fail_node_on_any_error"` (default) or `"ignore_errors"` |
| `maxGatherSize` | Max entries to gather in memory (default: 10000, max: 100000) |

Use the `code` field with `mode: "replace"` for string-replace edits to the handler code.


### Wiring the Pipeline

#### Map → Reduce wiring

Reduce nodes receive the raw DataList from a map node. Use `inputRefs` with `path: "dataList"`:

```json
{
  "nodeConfig": {
    "nodeType": "reduce",
    "inputRefs": {
      "dataList": {
        "sourceNodeId": "wfn_map_node_id",
        "path": "dataList"
      }
    }
  }
}
```

#### Map/Reduce → downstream node wiring

When `gatherResults` is enabled (the default), map and reduce nodes output a `results` array containing all gathered entries. Downstream nodes access this via:

**Option A: Agent node** — automatically accesses prior outputs via context memory search, no explicit wiring needed. Best when the downstream node needs to interpret, summarize, or flexibly reason about results.

**Option B: Code node with inputRefs** — use `inputRefs` with `path: "results"`:

```json
{
  "nodeConfig": {
    "nodeType": "code",
    "inputRefs": {
      "items": {
        "sourceNodeId": "wfn_reduce_node_id",
        "path": "results"
      }
    }
  }
}
```

The code node then accesses the data via `context.get_input('items')`, which returns the array of gathered results.

> **Warning — code node pitfalls with map/reduce results:**
> - Do NOT use `path: "dataList"` or `path: "dataListId"` — these return internal IDs, not the actual data. Always use `path: "results"`.
> - The `codeInputSchema` must exactly match the actual data type. If the reduce outputs an array, the schema must have `"type": "array"` for that field — a `"type": "string"` mismatch will cause a validation error.
> - If you encounter persistent type mismatch errors with code nodes, switch to an agent node instead — it's more forgiving and usually the better choice for consuming aggregated results.

#### Output keys available from map/reduce nodes (with gatherResults enabled)

| Key | Type | Description |
|-----|------|-------------|
| `results` | array | Gathered entries — use this for downstream data access |
| `totalEntries` / `totalKeys` | number | Total items processed |
| `successCount` | number | Successfully processed items |
| `failedCount` | number | Failed items |

## Choosing the Right Approach

| Need | Use |
|------|-----|
| Process each item through the full workflow | Batch runs |
| Transform/enrich a list within a workflow | Map (code mode) |
| Run an LLM agent on each item in a list | Map (agent mode) |
| Group and aggregate results by key | Map → Reduce |
| Collect all results and continue the workflow | Map with `gatherResults: true` (default) |
| Handle failures gracefully | Set `errorHandling: "ignore_errors"` on Map or Reduce |
| Summarize/interpret map or reduce results | Use an agent node downstream (reads results via context search) |
| Pass map/reduce results to a code node | Use `inputRefs` with `path: "results"` (NOT `"dataList"`) |
