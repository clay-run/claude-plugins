# Testing Workflows

## Clay CLI

You have access to the `clay` CLI for running and inspecting workflow test runs.
Invoke it as `clay …` (no path prefix, no `python3`). In Claude Code it is on your
PATH automatically; in Codex/Cursor, run the `setup` skill once to install it.

Requires the `CLAY_API_KEY` environment variable. The workspace is resolved from
the key, so there is no workspace id to pass. If `clay whoami` fails, run `/setup`.

Every command prints JSON to stdout on success and a typed error envelope to
stderr on failure, with categorical exit codes (0 ok, 2 validation, 3 auth,
5 network, 6 not-found). Pipe stdout to `jq`.

## Commands

```bash
# Start a test run (input JSON on stdin via --input -; defaults to {})
echo '{"key":"value"}' | clay workflows runs start <workflowId> --input -
clay workflows runs start <workflowId>                 # no inputs

# Status / progress for a run
clay workflows runs get <workflowId> <runId>           # header + progress + map/reduce nodes
clay workflows runs get <workflowId> <runId> --nodes   # include every node
clay workflows runs get <workflowId> <runId> --verbose # + full inputs/outputs, mappings, entry steps
clay workflows runs get <workflowId> <runId> --node-id <nodeId>  # isolate one node, full map/reduce results

# List/filter the individual execution steps
clay workflows runs steps <workflowId> <runId>
clay workflows runs steps <workflowId> <runId> --status failed
clay workflows runs steps <workflowId> <runId> --node-id <nodeId>

# Pause / resume a run
clay workflows runs pause <workflowId> <runId>
clay workflows runs resume <workflowId> <runId>

# Download a file written by a code node from the run's sandbox.
# The sandbox cwd is /home/daytona (NOT /home/user).
clay workflows runs download-file <workflowId> <runId> --path /home/daytona/output.csv
clay workflows runs download-file <workflowId> <runId> --path /home/daytona/report.pdf --output ./report.pdf

# List all workflows
clay workflows list

# Get a workflow (returns { id, name, url })
clay workflows get <workflowId>

# Create a new workflow
clay workflows create --name "My Workflow"
```

If you create a new workflow for users, include the link from `clay workflows
create`/`clay workflows get` output (the `url` field) in your response.

## Watching a run to completion

There is no `watch` command — poll `runs get` until the run leaves a non-terminal
state. `status` is one of `pending` / `running` / `paused` / `waiting` /
`completed` / `failed`; `progress.percentage` tracks progress.

```bash
# Poll every 5s until the run finishes, then print final status
while :; do
  status=$(clay workflows runs get <workflowId> <runId> | jq -r '.status')
  echo "status: $status"
  case "$status" in completed|failed) break ;; esac
  sleep 5
done
```

## Inspecting what a run did (instead of "logs")

There is no `logs` command. The structured output of `runs get` and `runs steps`
is strictly better than grepping formatted text — filter it with `jq`:

```bash
# Full, untruncated inputs/outputs per node
clay workflows runs get <workflowId> <runId> --verbose | jq '.nodes'

# Just the failed nodes and their errors
clay workflows runs get <workflowId> <runId> --nodes | jq '.nodes[] | select(.status=="failed") | {nodeId, errors}'

# Errors across the failed steps (including each map entry)
clay workflows runs steps <workflowId> <runId> --status failed | jq '.data[].errors'

# One node's config + full map/reduce results
clay workflows runs get <workflowId> <runId> --node-id <nodeId> | jq '.nodes[0]'
```

## Example workflow

1. Start a test: `echo '{}' | clay workflows runs start wf_abc --input -`
2. Watch progress with the poll loop above until `status` is `completed`/`failed`.
3. Inspect failures: `clay workflows runs steps wf_abc wfr_xyz --status failed | jq '.data[].errors'`

## Testing & exploration MCP tools

- **execute_clay_action**: Run any Clay action to see its output before using it in a workflow
  - Provide `actionPackageId`, `actionKey`, and `inputs`
  - Returns raw action result — use this to understand output format before building nodes
  - Note: actions consume credits
- **run_code**: Run Python code in a sandbox to test logic before putting it in code nodes
  - Code must define `handler(context)` returning a dict
  - Supports `context.call_tool()` if tools are provided
  - Supports `context.get_input()` if inputs are provided
  - Optionally install pip packages

## Pro tips

- Poll `runs get` in a loop (above) to monitor a run while you work.
- Pipe to `jq` for filtering: `clay workflows runs steps <workflowId> <runId> | jq '.data[] | select(.status=="failed")'`
- Save output for later analysis: `clay workflows runs get <workflowId> <runId> --verbose > run.json`
- `--verbose` returns untruncated inputs/outputs; prefer it over reconstructing logs.
