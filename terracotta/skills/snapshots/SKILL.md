---
name: snapshots
description: View workflow version history and restore to a previous state. Use when the user mentions snapshots, version history, or wants to see what changed. Also use when you need to undo an edit or the user wants you to undo one.
allowed-tools: Bash(clay *), Bash(jq *)
---

# Workflow Snapshots

Snapshots are immutable, point-in-time captures of the entire workflow graph — nodes, edges, prompts, scripts, tools, and positions.

## Key concepts

- **Automatic creation**: A snapshot is created automatically before every `edit_node` call and when a run starts
- **Content-addressed**: Each snapshot has a SHA-256 hash of its contents. Identical workflow states deduplicate to the same hash
- **Immutable**: Once created, a snapshot never changes
- **Run isolation**: Runs are pegged to a specific snapshot. Editing the workflow doesn't affect in-flight runs

## CLI reference

Use the `clay` CLI (on your PATH). It needs only `CLAY_API_KEY`; the workspace is
resolved from the key. Output is JSON — pipe it to `jq`.

### List recent snapshots

```bash
clay workflows snapshots list <workflowId>
```

Returns `{ data: [...] }`, newest first, each with `id`, `hash`, `createdAt`, and
`nodeCount`/`edgeCount`. So `data[0]` is the most recent snapshot.

### Show a snapshot (whole graph, or one node)

```bash
clay workflows snapshots get <workflowId> <snapshotId>
clay workflows snapshots get <workflowId> <snapshotId> --node-id <nodeId>
```

Returns the full captured graph: `nodes` (with types, prompts, code, tools) and
`edges`. `--node-id` narrows `nodes` to a single node (edges are left intact).

### Diff two snapshots

There is no built-in diff. Fetch both and compare with `jq`:

```bash
clay workflows snapshots get <workflowId> <oldSnapshotId> | jq '.nodes' > old.json
clay workflows snapshots get <workflowId> <newSnapshotId> | jq '.nodes' > new.json
diff <(jq -S . old.json) <(jq -S . new.json)
```

### Restore to a snapshot

```bash
clay workflows snapshots restore <workflowId> <snapshotId>
```

Restores the workflow to the exact state captured in the snapshot. This replaces
all current nodes, edges, prompts, scripts, and tools. Restore is destructive and
does NOT snapshot the current graph first — the pre-restore state is recoverable
only if it was already captured (snapshots are taken automatically before each
edit and at run start). If the current graph has unsnapshotted changes you might
want back, run `snapshots list` first and note the latest snapshot id.

## Common workflows

### Undo the last edit

Every `edit_node` call creates a snapshot before applying changes, so the most
recent snapshot (`data[0]`) is the state right before the last edit.

```bash
# Find the most recent snapshot, then restore to it
snap=$(clay workflows snapshots list <workflowId> | jq -r '.data[0].id')
clay workflows snapshots restore <workflowId> "$snap"
```

To undo multiple edits, pick an older snapshot id from the list instead.

### Compare current state to a previous version

List snapshots, then diff two of them with the `jq` recipe above.

### Review what a run executed

Since runs are pegged to snapshots, inspect the exact workflow state a run used by
looking up the run's snapshot id and passing it to `snapshots get`.
