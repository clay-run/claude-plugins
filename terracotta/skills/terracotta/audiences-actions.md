# Audiences actions

Read this before configuring `upsert-audiences-record` (or any future audiences-related action) on a tool node.

## upsert-audiences-record

Creates or updates a contact or account via `inputMappingConfig`. The config is a flat object with pipe-namespaced keys (`<group>|<fieldId>`) — never dot-notation, never nested.

When attaching this action to a tool node, use:

- `actionKey: "upsert-audiences-record"`
- `actionPackageId: "b1ab3d5d-b0db-4b30-9251-3f32d8b103c1"`

### Lookup vs record fields

- **Lookup fields** are the keys used to find an existing audience record. For contacts these are fixed (`email`, `linkedin_url`, `phone`); for accounts (`domain`, `linkedin_url`). The values you pass under `lookupFields|<id>` are matched against existing records — if any match, the record is updated; otherwise a new one is created.
- **Record fields** are the data written onto the matched (or newly created) record. These are workspace-defined and vary per workspace; call `get_audience_schema` to discover the available ids.

### Required keys

- `entityType` (static) — `"ACCOUNT"` or `"CONTACT"`.
- `lookupFields|selectedLookupFields` (static array) — lookup-key field ids.
- `lookupFields|<id>` for every id above — value to match on.
- `recordFields|selectedRecordFields` (static array) — fields to write.
- `recordFields|<id>` for every id above — value to write.
- `recordFields|removeNullValues` (static bool) — `true` skips blanks; `false` overwrites with them.

Every id in a `selected*` array MUST have a matching `<group>|<id>` binding, or the action returns `ERROR_BAD_REQUEST`.

### Discover real field IDs

Call `get_audience_schema` with `entityType` first — schema ids and English names often differ (account "company name" is `org_name`). It returns `lookupFields` (fixed: CONTACT → `email`, `linkedin_url`, `phone`; ACCOUNT → `domain`, `linkedin_url`) and `recordFields` (workspace-defined).

### Worked example — account upsert

```json
{
  "entityType":                          { "type": "static",    "value": "ACCOUNT" },
  "lookupFields|selectedLookupFields":   { "type": "static",    "value": ["domain"] },
  "lookupFields|domain":                 { "type": "reference", "expression": "{{domain}}" },
  "recordFields|selectedRecordFields":   { "type": "static",    "value": ["org_name", "domain"] },
  "recordFields|org_name":               { "type": "reference", "expression": "{{org_name}}" },
  "recordFields|domain":                 { "type": "reference", "expression": "{{domain}}" },
  "recordFields|removeNullValues":       { "type": "static",    "value": true }
}
```
