---
name: clay-tables
description: Use this skill when users mention Clay tables, want to query table data, import table data, reference a table, or ask about table structure. Provides context about how Clay tables work and how to use the 'table' tools.
---

## About Clay Tables

Clay is a GTM (go-to-market) data and automation product. Clay tables are similar to Excel or Google Sheets. Each table contains:

- **Fields (columns)**: Can be basic fields (text, numbers, booleans), formula fields (JavaScript expressions), or action/enrichment fields (fetch data, call APIs, run AI agents)
- **Records (rows)**: Usually represent companies or people
- **Sources**: Add rows to tables from external data (APIs, CSV imports, webhooks, Clay's database of 850m+ people and 60m+ companies)

## Available MCP Tools

Both tools use the `table` MCP tool with a `mode` parameter.

### Schema mode

Get the schema and structure of a Clay table — field names, types, configurations, metadata.

```
table(tableId: "the-table-id", mode: "schema")
```

Returns XML with:

- Table metadata (name, row count, workspace)
- All fields with their types, configurations, and data profiles
- Source information if applicable
- Field group information (waterfalls, etc.)

### Query mode

Query a table using natural language. Generates and runs a SQL (ClayQL) query.

```
table(tableId: "the-table-id", mode: "query", taskDescription: "Show top 10 deals by ARR")
```

Capabilities:

- Filter, group, sort, count, sum, average any field
- Can only get up to 8 fields at a time, with 5 wheres, and 3 group_bys
- Returns up to 100 rows
- Single table only — no joins or CTEs

Examples:

- "How many contacts are in Boston?"
- "Show companies with more than 50 employees sorted by funding"
- "Average deal size by stage"
- "Count of rows where email is not null"

### Saving query results to CSV

After running the `table` tool (mode: query), save the results locally as a CSV file so the user can access them. Convert the JSON results array to CSV format and write it using the Write tool.

Example flow:

1. Run the `table` tool (mode: query) to get results
2. Extract column headers from the first result row
3. Convert each row to comma-separated values (quote fields containing commas)
4. Write to a local file like `./query-results.csv`

## Field Types in tables

1. **Basic fields**: Contain text, numbers, or boolean values
2. **Formula fields**: Single-line JavaScript expressions that auto-calculate (use Lodash as `_` and Moment.js)
3. **Action fields**: Run enrichments that fetch data, call APIs, or invoke AI. Each cell needs to be "run" and may cost credits
4. **AI fields**: Special action fields that run LLMs on data (OpenAI, Anthropic, Claygent for web research, Image Generation)
5. **Source fields**: Contain data imported from sources (Clay's company/people/jobs dataset, CSV, webhooks, Clay actions, signals)

## Key Concepts

### Sources

Sources add rows to tables. Types include:

- **Company / People / Jobs data**: Clay CPJ data
- **API Integration**: Clay actions that fetch data from external services
- **CSV Import**: Data imported from CSV files
- **Webhook**: Data received via webhook calls
- **Manual Entry**: Manually entered data
- **Event Monitor (Signals)**: Monitors for events like job changes, news, etc.

### Actions

Actions are enrichments that run on each row. They can:

- Fetch data from 100+ data providers (email, phone, firmographics, tech stack, funding)
- Call external APIs
- Run AI agents (Claygent can browse the web)
- Search Clay's database of 850m+ people and 60m+ companies

### Formulas

Formulas are single-line JavaScript expressions that:

- Use `_` for Lodash and `moment` for date handling
- Auto-calculate when referenced fields change
- Must use optional chaining (`?.`) for safe access
- Cannot use loops, if statements, spread syntax, or template literals

### Waterfalls

Waterfalls are groups of action fields that run in sequence:

- Each action runs only if previous ones failed
- Used to maximize data coverage (e.g., try multiple email providers)
- Have a merge field that combines results

## Rebuilding a Table as a Workflow

If a user wants to convert their table logic into a Clay workflow (e.g., to make it reusable, add branching, or run it on a schedule), use `/terracotta` to build a workflow that replicates the table's enrichment pipeline as connected nodes.
