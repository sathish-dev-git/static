# 13 · MCP Tool Catalog — all 54 tools

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md). Server options, access levels, client setup: [12 · MCP server](12-mcp-server.md).
> Source: `ZA_CLI/src/main/java/com/zoho/analytics/cli/tools/**` (catalog, schemas, handlers), `mcp/**`, `chat/ToolExecutor.java`. Argument names, enums and result shapes below are taken verbatim from the code.

---

## 1. Catalog facts

| Fact | Value |
|---|---|
| Total tools | **54** — Metadata 13 · Rows 4 · Data 3 · Views 4 · Modelling 5 · Workspaces 15 · Administration 7 · Diagnostics 3 |
| Safety split | READ_ONLY **23** · MUTATING **25** · DESTRUCTIVE **6** |
| Exposed by `--access-level=read` (default) | 23 tools |
| Exposed by `--access-level=write` | 48 tools (never the 6 destructive) |
| Exposed by `--access-level=full` | 54 tools |
| Categories (for `--categories`) | `METADATA`, `ROW`, `DATA`, `VIEW`, `MODELLING`, `WORKSPACE`, `ADMIN`, `DIAGNOSTICS` (case-insensitive) |
| Schema style | `{"type":"object","properties":{…},"additionalProperties":false,"required":[…]}` |
| ID arguments | Always **strings** (Zoho IDs exceed double precision); numeric JSON is also accepted |
| Tool annotations sent to clients | `title`, `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint=false` |
| Result content | One MCP `text` content block; JSON pretty-printed with 2-space indent; plus `_meta.correlationId` and `_meta.cliCommand` (and `_meta.profile` in `--profiles` mode) |
| Result size cap | 1,000,000 chars, or `--max-result-tokens × 4` |
| Error signalling | `isError: true`; text is the JSON `{"error": "...", "errorCode": <n>}` (never TOON-encoded) |

### Shared argument fragments

| Argument | Type | Description (verbatim) | Present on |
|---|---|---|---|
| `workspaceId` | string | "Workspace ID (pass as a string)." / "Workspace ID (pass as a string to preserve precision)." | almost every tool |
| `viewId` | string | tool-specific, e.g. "View ID to inspect (pass as a string)." | view-level tools |
| `offset` | integer | "Number of items to skip before returning results (default: 0)." | 9 list tools |
| `limit` | integer | "Maximum number of items to return (default: all)." | 9 list tools |
| `all` | boolean | "Return every item, ignoring limit." | 9 list tools |
| `profile` | string | "Which configured profile/org this call should run against." | **every** tool, only when the server runs with `--profiles` (added and made required at startup) |

Pagination on list tools is **client-side**: the full list is fetched, then sliced; `all:true` beats `limit`.

### Standard error result

```json
{"error": "The specified workspace ID does not exist.", "errorCode": 7103}
```

`errorCode` is Zoho's own numeric code and is present only when the failure came from the Zoho API. Validation failures inside za-cli carry only `error`, for example:

```json
{"error": "Missing required field: workspaceId"}
{"error": "Field must not be empty: columnValues"}
{"error": "Read-only mode: mutating operations are disabled"}
{"error": "Refusing to delete with a match-all criteria. Set deleteAll to true to intentionally delete every row."}
```

The complete Zoho-code → friendly-message table is in [15 · Troubleshooting](15-troubleshooting-and-error-codes.md).

### Dry-run result (`za-cli mcp-server --dry-run …`)

Any MUTATING/DESTRUCTIVE call returns a **successful** result shaped:

```json
{
  "dryRun": true,
  "tool": "delete_view",
  "safety": "DESTRUCTIVE",
  "args": { "workspaceId": "2148712000000012345", "viewId": "2148712000000054321" },
  "message": "Dry run: no changes were made"
}
```

READ_ONLY tools are unaffected. On a `read`-level server a mutating call is *refused* (read-only error) before dry-run is considered.

### Truncation markers you may see

- JSON object over budget: `…\n(truncated: showing first N of T fields; refine your request to see the rest)`
- JSON array over budget: `…\n(truncated: showing first N of T items; refine your request to see the rest)`
- Plain text over budget: `…\n(truncated, showing first N characters)`
- `list_workspaces` / `list_views` summaries are additionally hard-capped at ~8,000 chars: `\n... (truncated, N shown of T total items)`

---

## 2. Tool index

| # | Tool | Category | Safety | Idempotent | CLI equivalent |
|---|---|---|---|---|---|
| 1 | `list_workspaces` | Metadata | READ_ONLY | yes | `workspace list` |
| 2 | `get_workspace_details` | Metadata | READ_ONLY | yes | `workspace get` |
| 3 | `list_views` | Metadata | READ_ONLY | yes | `view list` |
| 4 | `get_view_info` | Metadata | READ_ONLY | yes | `view get` |
| 5 | `get_view_columns` | Metadata | READ_ONLY | yes | `view columns` |
| 6 | `get_view_url` | Metadata | READ_ONLY | yes | `view url` |
| 7 | `get_embed_url` | Metadata | READ_ONLY | yes | `view embed-url` |
| 8 | `get_publish_config` | Metadata | READ_ONLY | yes | `view publish-config` |
| 9 | `list_datasources` | Metadata | READ_ONLY | yes | `workspace datasources` |
| 10 | `list_folders` | Metadata | READ_ONLY | yes | `workspace folders` |
| 11 | `list_groups` | Metadata | READ_ONLY | yes | `workspace groups` |
| 12 | `list_trash` | Metadata | READ_ONLY | yes | `workspace trash` |
| 13 | `search_metadata` | Metadata | READ_ONLY | yes | `search` |
| 14 | `preview_data` | Rows | READ_ONLY | yes | `view preview` |
| 15 | `add_row` | Rows | MUTATING | no | `view add-row` |
| 16 | `update_rows` | Rows | MUTATING | yes | `view update-row` |
| 17 | `delete_rows` | Rows | **DESTRUCTIVE** | no | `view delete-rows` |
| 18 | `export_data` | Data | MUTATING | yes | `export` |
| 19 | `import_data` | Data | MUTATING | no | `import` |
| 20 | `import_rows` | Data | MUTATING | no | `view import-rows` |
| 21 | `rename_view` | Views | MUTATING | yes | `view rename` |
| 22 | `add_column` | Views | MUTATING | no | `view add-column` |
| 23 | `make_view_public` | Views | MUTATING | yes | `view publish` |
| 24 | `delete_view` | Views | **DESTRUCTIVE** | no | `view delete` |
| 25 | `create_table` | Modelling | MUTATING | no | `view create-table` |
| 26 | `create_query_table` | Modelling | MUTATING | no | `view create-query-table` |
| 27 | `create_chart` | Modelling | MUTATING | no | `view create-chart` |
| 28 | `create_summary_report` | Modelling | MUTATING | no | `view create-summary` |
| 29 | `create_pivot_report` | Modelling | MUTATING | no | `view create-pivot` |
| 30 | `create_workspace` | Workspaces | MUTATING | no | `workspace create` |
| 31 | `rename_workspace` | Workspaces | MUTATING | yes | `workspace rename` |
| 32 | `create_folder` | Workspaces | MUTATING | no | `workspace create-folder` |
| 33 | `create_group` | Workspaces | MUTATING | no | `workspace create-group` |
| 34 | `add_workspace_users` | Workspaces | MUTATING | yes | `workspace users add` |
| 35 | `remove_workspace_users` | Workspaces | MUTATING | yes | `workspace users remove` |
| 36 | `get_workspace_share_info` | Workspaces | READ_ONLY | yes | `workspace share-info` |
| 37 | `get_view_share_details` | Workspaces | READ_ONLY | yes | `view share-info` |
| 38 | `share_view` | Workspaces | MUTATING | yes | `view share` |
| 39 | `remove_view_share` | Workspaces | MUTATING | yes | `view remove-share` |
| 40 | `restore_trash_view` | Workspaces | MUTATING | no | `workspace restore-trash` |
| 41 | `delete_workspace` | Workspaces | **DESTRUCTIVE** | no | `workspace delete` |
| 42 | `delete_folder` | Workspaces | **DESTRUCTIVE** | no | `workspace delete-folder` |
| 43 | `delete_group` | Workspaces | **DESTRUCTIVE** | no | `workspace delete-group` |
| 44 | `delete_trash_view` | Workspaces | **DESTRUCTIVE** | no | `workspace delete-trash` |
| 45 | `list_workspace_users` | Administration | READ_ONLY | yes | `workspace users` |
| 46 | `list_org_users` | Administration | READ_ONLY | yes | `org users` |
| 47 | `list_admins` | Administration | READ_ONLY | yes | `org admins` |
| 48 | `get_subscription` | Administration | READ_ONLY | yes | `org info` |
| 49 | `get_resource_usage` | Administration | READ_ONLY | yes | `org resources` |
| 50 | `add_org_users` | Administration | MUTATING | yes | `org users add` |
| 51 | `remove_org_users` | Administration | MUTATING | yes | `org users remove` |
| 52 | `get_telemetry_status` | Diagnostics | READ_ONLY | yes | `telemetry` |
| 53 | `get_telemetry_summary` | Diagnostics | READ_ONLY | yes | `telemetry` |
| 54 | `clear_telemetry` | Diagnostics | MUTATING | yes | `telemetry --clear` |

*Idempotent* = calling twice leaves the same state as calling once (safe for a client to retry after a network hiccup).

---

## 3. Metadata tools (13 · all READ_ONLY)

### 3.1 `list_workspaces`
**Title** List Workspaces · **CLI** `za-cli workspace list`
**Use case** List every workspace the authenticated user can access. Call it first to discover the `workspaceId` needed by almost every other tool.

| Arg | Type | Required | Description |
|---|---|---|---|
| `offset` | integer | no | skip N items |
| `limit` | integer | no | max items |
| `all` | boolean | no | ignore limit |

**Returns** plain text (not JSON), a numbered summary:

```text
Total: 3 items

1. Sales Analytics (id: 2148712000000012345)
2. Marketing (id: 2148712000000023456)
3. Finance – Shared (id: 2148712000000034567)
```

With pagination the header becomes `Showing 11-20 of 57 items`. Empty → `No items found. (count: 0)`.

Example call / result:

```json
// request
{"name":"list_workspaces","arguments":{"limit":2}}
// result text
"Showing 1-2 of 3 items\n\n1. Sales Analytics (id: 2148712000000012345)\n2. Marketing (id: 2148712000000023456)"
```

### 3.2 `get_workspace_details`
**Title** Get Workspace Details · **CLI** `za-cli workspace get <id>`

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |

**Returns** the raw Zoho workspace object, e.g.

```json
{
  "workspaceId": "2148712000000012345",
  "workspaceName": "Sales Analytics",
  "workspaceDesc": "Quarterly sales reporting",
  "createdBy": "admin@example.com",
  "createdTime": "2025-11-03 09:14:22",
  "isDefault": false,
  "orgId": "20087654321"
}
```

### 3.3 `list_views`
**Title** List Views · **CLI** `za-cli view list -w <ws>`

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |
| `offset` / `limit` / `all` | int / int / bool | no |

**Returns** numbered text with the view type in brackets:

```text
Total: 4 items

1. Orders (id: 2148712000000054321) [Table]
2. Customers (id: 2148712000000054322) [Table]
3. Revenue by Region (id: 2148712000000054330) [Chart]
4. Sales Dashboard (id: 2148712000000054340) [Dashboard]
```

### 3.4 `get_view_info`
**Title** Get View Details · **CLI** `za-cli view get -w <ws> <view>`

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |
| `viewId` | string | **yes** |

**Returns** the view object with noise removed. Stripped keys: `orgId, createdByName, createdByZuId, lastDesignModifiedByName, lastDesignModifiedByZuId, lastModifiedByName, lastModifiedByZuId`; inside a nested `columns` array also `dataTypeId, dataType, columnIndex, sortedIndex, sortedOrder`.

```json
{
  "viewId": "2148712000000054321",
  "viewName": "Orders",
  "viewType": "Table",
  "viewDesc": "",
  "folderId": "2148712000000040001",
  "createdTime": "2025-11-03 09:20:01",
  "lastModifiedTime": "2026-02-11 17:02:45",
  "columns": [
    {"columnId": "2148712000000054401", "columnName": "OrderId", "dataTypeName": "NUMBER"},
    {"columnId": "2148712000000054402", "columnName": "Region", "dataTypeName": "PLAIN"}
  ]
}
```

### 3.5 `get_view_columns`
**Title** Get View Columns · **CLI** `za-cli view columns -w <ws> <view>`
**Use case** Get column names and data types before adding/updating rows or writing criteria.

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |
| `viewId` | string | **yes** |

**Returns** a JSON **array** of column definitions (cleaned) — or the whole table metadata object if the response had no `columns` array.

```json
[
  {"columnId": "2148712000000054401", "columnName": "OrderId",   "dataTypeName": "NUMBER"},
  {"columnId": "2148712000000054402", "columnName": "Region",    "dataTypeName": "PLAIN"},
  {"columnId": "2148712000000054403", "columnName": "Revenue",   "dataTypeName": "CURRENCY"},
  {"columnId": "2148712000000054404", "columnName": "OrderDate", "dataTypeName": "DATE"}
]
```

### 3.6 `get_view_url` / 3.7 `get_embed_url`
**CLI** `view url` / `view embed-url`

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |
| `viewId` | string | **yes** |

**Returns**

```json
{"viewUrl": "https://analytics.zoho.com/workspace/2148712000000012345/view/2148712000000054321"}
```
```json
{"embedUrl": "https://analytics.zoho.com/open-view/2148712000000054321/…"}
```

### 3.8 `get_publish_config`
**CLI** `view publish-config`. Args `workspaceId`, `viewId` (both required). **Returns** the raw publish/share configuration object for the view (public flag, access mode, link details, embed options as reported by Zoho).

### 3.9 `list_datasources`
**CLI** `workspace datasources`. Args `workspaceId` (required) + `offset`/`limit`/`all`. **Returns** a JSON array of data-source descriptors (connection name, type, schedules, synced views).

### 3.10 `list_folders`
**CLI** `workspace folders`. Args `workspaceId` + pagination. **Returns**

```json
[
  {"folderId": "2148712000000040001", "folderName": "Default", "folderDesc": "", "isDefault": true, "parentFolderId": "-1"},
  {"folderId": "2148712000000040002", "folderName": "Q1 Reports", "folderDesc": "", "isDefault": false, "parentFolderId": "-1"}
]
```

### 3.11 `list_groups`
**CLI** `workspace groups`. Args `workspaceId` + pagination. **Returns**

```json
[
  {"groupId": "2148712000000067890", "groupName": "Sales team", "groupDesc": "", "members": ["alice@example.com","bob@example.com"]}
]
```

### 3.12 `list_trash`
**CLI** `workspace trash`. Args `workspaceId` + pagination. **Returns** a JSON array of trashed views (`viewId`, `viewName`, `viewType`, deleted-by / deleted-time fields as reported).

### 3.13 `search_metadata`
**Title** Search Metadata · **CLI** `za-cli search <query> [-w <ws>]`
**Use case** "which table has churn data" — keyword/fuzzy name matching across workspace and view names org-wide, plus column names when `workspaceId` is given. Not semantic.

| Arg | Type | Required | Description |
|---|---|---|---|
| `query` | string | **yes** | Search text to match against names. |
| `workspaceId` | string | no | Limit to one workspace and also search its column names. |

**Returns**

```json
{
  "query": "churn",
  "totalMatches": 87,
  "matches": [
    {"score": 100, "matchedOn": "view name",   "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054350", "viewName": "churn"},
    {"score": 75,  "matchedOn": "column name", "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054322", "viewName": "Customers", "columnName": "customer_churn"},
    {"score": 25,  "matchedOn": "workspace name", "workspaceId": "2148712000000099999", "workspaceName": "Churn & Retention"}
  ],
  "note": "Showing the top 50 of 87 matches."
}
```

Scoring: **100** exact (case-insensitive) · **75** substring · **50** all query words present (multi-word queries) · **25** at least one word present. Sorted best-first, capped at **50** (`note` appears only when truncated). Column search skips chart/pivot/summary/report/kpi/analysisview/dashboard views. Between per-view metadata calls za-cli paces itself adaptively (0 → 200 ms → up to 2 s after a slow call) to respect rate limits.

---

## 4. Row tools (4)

### 4.1 `preview_data` · READ_ONLY
**CLI** `view preview`. **Use case** Preview the first rows of a view as CSV text.

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | |
| `limit` | integer | no | Maximum number of rows (default 20, max 100). |

**Returns** (at most 20 lines including the header, regardless of `limit`):

```json
{
  "format": "csv",
  "requestedLimit": 20,
  "shownLines": 4,
  "previewLines": [
    "OrderId,Region,Revenue,OrderDate",
    "1001,East,1200.50,2026-01-05",
    "1002,West,900.00,2026-01-06",
    "1003,East,300.25,2026-01-07"
  ]
}
```

### 4.2 `add_row` · MUTATING · not idempotent
**CLI** `view add-row`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | Table view ID |
| `columnValues` | object | **yes** | Column name → value pairs (non-empty) |

```json
// request
{"workspaceId":"2148712000000012345","viewId":"2148712000000054321","columnValues":{"Region":"West","Revenue":900}}
// result: Zoho's own response describing the inserted row (shape as returned by the API), e.g.
{"columns": {"OrderId": "1004", "Region": "West", "Revenue": "900"}}
```

### 4.3 `update_rows` · MUTATING · idempotent
**CLI** `view update-row`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | |
| `columnValues` | object | **yes** | new values |
| `criteria` | string | **yes** | e.g. `"\"Sales\".\"Region\"='East'"` |

Guard: a match-all criteria (blank, `true`, `1=1`, `'1'='1'`, or any OR-branch tautology) is refused: `Refusing to update with a match-all criteria. Provide a criteria that selects specific rows.`

```json
// result (Zoho response; updatedRows may be a string, sometimes nested under "result")
{"updatedRows": "42", "updatedColumns": "[]"}
```

### 4.4 `delete_rows` · **DESTRUCTIVE** · not idempotent
**CLI** `view delete-rows`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | |
| `criteria` | string | no | rows to delete |
| `deleteAll` | boolean | no | true = delete every row (criteria ignored) |

Guards: no criteria and no `deleteAll` → `criteria is required. To delete every row, set deleteAll to true.`; match-all criteria → `Refusing to delete with a match-all criteria. Set deleteAll to true to intentionally delete every row.`

```json
{"deletedRows": 17}
```

---

## 5. Data tools (3 · all MUTATING)

### 5.1 `export_data` · idempotent
**CLI** `export`. Prefer `csv` for tables, `html` for charts, `pdf` for dashboards.

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | |
| `format` | enum | **yes** | `csv, json, xml, xls, pdf, html, image` |
| `outputPath` | string | no | default `za-export-<workspaceId>-<viewId>.<format>` in the current directory |
| `force` | boolean | no | overwrite existing file |
| `criteria` | string | no | row filter |
| `columns` | string | no | comma-separated |
| `recordLimit` | integer | no | max records |
| `showHiddenCols` | boolean | no | |
| `showPersonalCols` | boolean | no | |

Output path **must be inside the current working directory** of the server process.

```json
{"status":"exported","workspaceId":2148712000000012345,"viewId":2148712000000054321,"format":"csv","outputPath":"/home/you/project/za-export-2148712000000012345-2148712000000054321.csv"}
```

Errors: `Unsupported export format: <f>. Allowed: csv, json, xml, xls, pdf, html, image.` · `Output file already exists: <path>. Set force to true to overwrite, or choose a different outputPath.` · `Unsafe export path: '<p>' is outside the current directory. Files must be within: <base>`

### 5.2 `import_data` · not idempotent
**CLI** `import`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | target table |
| `filePath` | string | **yes** | `.csv` or `.json`, inside the current directory |
| `importType` | enum | **yes** | `append, truncateadd, updateadd` |
| `matchingColumns` | string | no | for `updateadd` |
| `dateFormat` | string | no | |
| `delimiter` | enum | no | `comma, tab, semicolon, pipe` |
| `onError` | enum | no | `abort, skiprow, setcolumnempty` |
| `skipRows` | integer | no | |

```json
{"status":"imported","workspaceId":2148712000000012345,"viewId":2148712000000054321,"filePath":"/home/you/project/data.csv","importType":"append"}
```

Errors: `Import file not found: <p>` · `Import path is not a regular file: <p>` · `Unsupported import file type. Use .csv or .json files.` · cwd `SecurityException` as for export.

### 5.3 `import_rows` · not idempotent
**CLI** `view import-rows`. Prefer over repeated `add_row` when seeding several rows.

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `viewId` | string | **yes** | |
| `rows` | array of object | **yes** | each row = column → value |
| `importType` | enum | no | `append` (default), `truncateadd`, `updateadd` |

**Returns** Zoho's import response. The fields za-cli itself relies on are `importSummary.totalRowCount`, `importSummary.successRowCount` and `importSummary.warnings`; other keys are whatever the API returns.

```json
{
  "importSummary": {
    "totalRowCount": 3,
    "successRowCount": 3,
    "warnings": 0
  }
}
```

---

## 6. View tools (4)

| Tool | Args (all required) | Returns |
|---|---|---|
| `rename_view` · MUTATING · idempotent · `view rename` | `workspaceId`, `viewId`, `newName` | `{"status":"renamed","viewId":2148712000000054321,"newName":"Q1 Sales"}` |
| `add_column` · MUTATING · `view add-column` | `workspaceId`, `viewId`, `columnName`, `dataType` ∈ `PLAIN, NUMBER, DATE, EMAIL, CURRENCY, URL, POSITIVE_NUMBER, DECIMAL_NUMBER` | `{"status":"added","columnId":2148712000000054410,"columnName":"Status"}` |
| `make_view_public` · MUTATING · idempotent · `view publish` | `workspaceId`, `viewId` | `{"status":"published","publicUrl":"https://analytics.zoho.com/open-view/…"}` |
| `delete_view` · **DESTRUCTIVE** · `view delete` | `workspaceId`, `viewId` | `{"status":"deleted","viewId":2148712000000054321}` |

---

## 7. Modelling tools (5 · all MUTATING · not idempotent)

### 7.1 `create_table` · `view create-table`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `tableName` | string | **yes** | |
| `columns` | array | **yes** | items `{ "columnName": string, "dataType": PLAIN\|NUMBER\|DATE\|EMAIL\|CURRENCY\|URL\|POSITIVE_NUMBER\|DECIMAL_NUMBER }` |

```json
// request
{"workspaceId":"2148712000000012345","tableName":"Orders","columns":[{"columnName":"OrderId","dataType":"NUMBER"},{"columnName":"CustomerEmail","dataType":"EMAIL"}]}
// result
{"status":"created","tableId":2148712000000054321,"tableName":"Orders"}
```

Internally translated to the SDK design `{"TABLENAME": …, "COLUMNS": [{"COLUMNNAME": …, "DATATYPE": …}]}`.

### 7.2 `create_query_table` · `view create-query-table`

| Arg | Type | Required |
|---|---|---|
| `workspaceId` | string | **yes** |
| `tableName` | string | **yes** |
| `query` | string | **yes** — single MySQL-compatible SELECT |

→ `{"status":"created","tableId":2148712000000054360,"tableName":"TopCustomers"}`

### 7.3 `create_chart` · `view create-chart`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `tableName` | string | **yes** | base table name |
| `chartName` | string | **yes** | |
| `chartType` | enum | **yes** | `bar, line, pie, scatter, bubble` |
| `xAxis` | object | **yes** | `{ "columnName": string, "operation": string, "tableName"?: string }` |
| `yAxis` | object | **yes** | same shape |

Operation guidance: category axis → non-aggregate (`actual`, `dimension`); measure axis → aggregate (`sum`, `count`, `average`, …); date columns → `year`, `month`, `week`, …. `tableName` inside an axis only when the column comes from a related table.

```json
// request
{"workspaceId":"2148712000000012345","tableName":"Orders","chartName":"Revenue by region","chartType":"bar",
 "xAxis":{"columnName":"Region","operation":"actual"},"yAxis":{"columnName":"Revenue","operation":"sum"}}
// result
{"status":"created","reportType":"chart","reportId":2148712000000054330}
```

### 7.4 `create_summary_report` · `view create-summary`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `tableName` | string | **yes** | |
| `reportName` | string | **yes** | |
| `groupBy` | array | **yes** | items `{ "columnName", "tableName", "operation" }` (use `actual`, `year`, …) |
| `aggregate` | array | **yes** | items same shape (use `sum`, `count`, `average`, … — never `actual`) |

→ `{"status":"created","reportType":"summary","reportId":2148712000000054335}`

### 7.5 `create_pivot_report` · `view create-pivot`

| Arg | Type | Required | Description |
|---|---|---|---|
| `workspaceId` | string | **yes** | |
| `tableName` | string | **yes** | |
| `reportName` | string | **yes** | |
| `row` | array | no | `{columnName, tableName, operation}` items |
| `column` | array | no | same |
| `data` | array | no | same (aggregate ops) |

At least one of `row`/`column`/`data` must be non-empty, else: `Provide at least one of 'row', 'column', or 'data' for the pivot report.`
→ `{"status":"created","reportType":"pivot","reportId":2148712000000054336}`

---

## 8. Workspace tools (15)

| Tool | Safety | Args | Returns |
|---|---|---|---|
| `create_workspace` · `workspace create` | MUTATING | `name` (req) | `{"workspaceId":2148712000000012399,"name":"Sales Analytics"}` |
| `rename_workspace` · `workspace rename` | MUTATING, idempotent | `workspaceId`, `newName` | `{"status":"renamed","workspaceId":…,"newName":"…"}` |
| `create_folder` · `workspace create-folder` | MUTATING | `workspaceId`, `name` | `{"status":"created","folderId":2148712000000040002,"name":"Q1 Reports"}` |
| `create_group` · `workspace create-group` | MUTATING | `workspaceId`, `name`, `emails` (array of string; a comma-separated string is also accepted) | `{"status":"created","groupId":2148712000000067890,"name":"Sales team"}` |
| `add_workspace_users` · `workspace users add` | MUTATING, idempotent | `workspaceId`, `emails` (array), `role` (opt, default viewer) | `{"status":"added","count":2}` |
| `remove_workspace_users` · `workspace users remove` | MUTATING, idempotent | `workspaceId`, `emails` | `{"status":"removed","count":1}` |
| `get_workspace_share_info` · `workspace share-info` | READ_ONLY | `workspaceId` | object with `sharedUsers`/`userPermissions`, `groupNames`/`groupPermissions`, `privateLinkPermissions`, `publicPermissions` (see below) |
| `get_view_share_details` · `view share-info` | READ_ONLY | `workspaceId`, `viewId` | JSON **array**, one entry per grant: user email, group, `"Public Visitor"` or `"Private Link"`, with exact permissions |
| `share_view` · `view share` | MUTATING, idempotent | `workspaceId`, `viewId`, `emails` (array), `permissions` (object of booleans) | `{"status":"shared","viewId":…,"count":1}` |
| `remove_view_share` · `view remove-share` | MUTATING, idempotent | `workspaceId`, `viewId`, `emails` | `{"status":"removed","viewId":…,"count":1}` |
| `restore_trash_view` · `workspace restore-trash` | MUTATING | `workspaceId`, `viewId` | `{"status":"restored","viewId":…}` |
| `delete_workspace` · `workspace delete` | **DESTRUCTIVE** | `workspaceId` | `{"status":"deleted","workspaceId":…}` |
| `delete_folder` · `workspace delete-folder` | **DESTRUCTIVE** | `workspaceId`, `folderId` | `{"status":"deleted","folderId":…}` |
| `delete_group` · `workspace delete-group` | **DESTRUCTIVE** | `workspaceId`, `groupId` | `{"status":"deleted","groupId":…}` |
| `delete_trash_view` · `workspace delete-trash` | **DESTRUCTIVE** | `workspaceId`, `viewId` | `{"status":"permanently_deleted","viewId":…}` |

### `share_view` permissions object

Boolean keys, same spelling as the read side: `read, export, vud, drillDown, addRow, updateRow, deleteRow, deleteAllRows, importAppend, importAddOrUpdate, importDeleteAllAdd, share, discussion`. Always set `read: true`; without it nothing else grants access. (The read side additionally reports `importDeleteUpdateAdd`, which cannot be set here.)

```json
{"workspaceId":"2148712000000012345","viewId":"2148712000000054321","emails":["alice@example.com"],
 "permissions":{"read":true,"export":true,"drillDown":true}}
```

### `get_workspace_share_info` result shape

Synthesised by za-cli from the SDK's `ShareInfo` (per-user and per-group lists of per-view permission entries). `userPermissions` is keyed by email, `groupPermissions` by group name; every entry carries all 14 permission flags (unset flags are `false`, never missing).

```json
{
  "sharedUsers": ["alice@example.com", "bob@example.com"],
  "userPermissions": {
    "alice@example.com": [
      {"viewId": 2148712000000054321, "viewName": "Orders", "sharedBy": "admin@example.com",
       "filterCriteria": "\"Region\" = 'East'",
       "permissions": {"read": true, "export": true, "vud": false, "addRow": false, "updateRow": false,
                       "deleteRow": false, "deleteAllRows": false, "importAppend": false, "importAddOrUpdate": false,
                       "importDeleteAllAdd": false, "importDeleteUpdateAdd": false, "drillDown": true,
                       "share": false, "discussion": false}}
    ]
  },
  "groupNames": ["Finance Team"],
  "groupPermissions": {
    "Finance Team": [ {"viewId": 2148712000000054330, "viewName": "Revenue by Region", "sharedBy": "admin@example.com", "permissions": {"read": true, "…": false}} ]
  },
  "publicPermissions":      [ {"viewId": 2148712000000054340, "viewName": "Public Dashboard", "permissions": {"read": true, "…": false}} ],
  "privateLinkPermissions": [ {"viewId": 2148712000000054331, "viewName": "Shared Report",    "permissions": {"read": true, "…": false}} ]
}
```

### `get_view_share_details` result shape

One flattened row per grant (a view shared with three people returns three rows):

```json
[
  {"viewId": "2148712000000054321", "sharedTo": "alice@example.com", "sharedBy": "admin@example.com",
   "permissionString": "Read Access", "isGroupShare": false, "criteria": "",
   "permissions": {"read": true, "export": false, "share": false}},
  {"viewId": "2148712000000054321", "sharedTo": "Sales team", "sharedBy": "admin@example.com",
   "permissionString": "Full Access", "isGroupShare": true, "criteria": "",
   "permissions": {"read": true, "export": true, "addRow": true, "updateRow": true, "deleteRow": true}}
]
```

`sharedTo` may also be `"Public Visitor"` or `"Private Link"`.

---

## 9. Administration tools (7)

| Tool | Safety | Args | Returns |
|---|---|---|---|
| `list_workspace_users` · `workspace users` | READ_ONLY | `workspaceId` + pagination | array of `{emailId, role, status, …}` |
| `list_org_users` · `org users` | READ_ONLY | pagination only | array of org users |
| `list_admins` · `org admins` | READ_ONLY | pagination only | array of admins |
| `get_subscription` · `org info` | READ_ONLY | none | subscription object (plan, limits, validity) |
| `get_resource_usage` · `org resources` | READ_ONLY | none | array of resource usage entries |
| `add_org_users` · `org users add` | MUTATING, idempotent | `emails` (array) | `{"status":"added","count":2}` |
| `remove_org_users` · `org users remove` | MUTATING, idempotent | `emails` | `{"status":"removed","count":1}` |

Sample `list_workspace_users` result:

```json
[
  {"emailId": "admin@example.com", "role": "admin",  "status": "active"},
  {"emailId": "alice@example.com", "role": "viewer", "status": "active"}
]
```

Sample `get_subscription` result (fields as reported by Zoho; illustrative):

```json
{
  "planName": "Premium",
  "planType": "PAID",
  "userCount": 15,
  "maxUsers": 25,
  "rowCount": 4200000,
  "maxRows": 10000000,
  "expiryDate": "2027-01-31",
  "orgId": "20087654321",
  "orgName": "Example Corp"
}
```

---

## 10. Diagnostics tools (3 · local machine only, never touch Zoho)

### 10.1 `get_telemetry_status` · READ_ONLY · `telemetry`
No args. Returns

```json
{"enabled":false,"settingKey":"telemetry.enabled","settingsFile":"/home/you/za-cli/config/settings.json","countsFile":"/home/you/za-cli/config/telemetry.json"}
```

### 10.2 `get_telemetry_summary` · READ_ONLY · `telemetry`

| Arg | Type | Required | Description |
|---|---|---|---|
| `topN` | integer | no | max entries per ranked list (default 10) |

```json
{
  "enabled": true,
  "totalCalls": 412, "totalCommandCalls": 300, "totalToolCalls": 112,
  "uniqueCommands": 17, "uniqueTools": 9,
  "topCommands": [{"name":"workspace list","count":120}],
  "topTools": [{"name":"list_views","count":44}],
  "rawCounts": {"command:workspace list":120, "tool:list_views":44}
}
```

All zeros when telemetry is off or empty — that is not an error.

### 10.3 `clear_telemetry` · MUTATING · idempotent · `telemetry --clear`
No args → `{"status":"cleared"}`. Only affects the local counts file.

---

## 11. Prompts (2)

Prompts return instructional text only; they never call Zoho. Each is **withheld** if any tool it references is outside the resolved scope, and **none are registered** in `--profiles` mode.

### 11.1 `audit_workspace` — "Review a workspace's structure, permissions, and stale resources."

| Argument | Required | Description |
|---|---|---|
| `workspaceName` | yes | The workspace name to audit (as seen in the ZA UI, not its ID). |

Uses only READ_ONLY tools: `list_workspaces, get_workspace_details, list_views, list_folders, list_groups, list_workspace_users, list_trash` → available at every access level.

Returned message (user role):

```text
Audit the workspace named "<workspaceName>" using only read-only tools (list_workspaces, get_workspace_details, list_views, list_folders, list_groups, list_workspace_users, list_trash):
1. Call list_workspaces to find the workspace ID whose name matches "<workspaceName>" exactly. If zero or
   multiple workspaces match, stop and ask the user to clarify/pick the correct
   workspace before continuing -- never guess.
2. Call get_workspace_details to see the workspace's own metadata (owner, creation date).
3. Call list_views and list_folders to inventory every table/chart/dashboard and its folder.
4. Call list_workspace_users and list_groups to review who has access and what groups exist -- flag any
   group whose membership looks broader than the workspace's purpose warrants.
5. Call list_trash to check for views awaiting permanent deletion that may need a decision.
Summarize findings as: structure overview, access/permission concerns, and stale
(trashed) resources. Do not modify anything -- this is a read-only review.
```

### 11.2 `build_report` — "Design and create a summary, pivot, or chart report from an existing table."

| Argument | Required | Description |
|---|---|---|
| `workspaceName` | yes | The workspace name containing the source table (as seen in the ZA UI, not its ID). |
| `goal` | yes | What the report should show, e.g. "monthly revenue by region". |

Uses `list_workspaces, list_views, get_view_columns, preview_data, create_summary_report, create_pivot_report, create_chart` → **withheld under `--access-level=read`**, present under `write`/`full`.

```text
Build a report in the workspace named "<workspaceName>" for this goal: "<goal>"
1. Call list_workspaces to find the workspace ID whose name matches "<workspaceName>" exactly. If zero or
   multiple workspaces match, stop and ask the user to clarify/pick the correct
   workspace before continuing -- never guess.
2. Call list_views to find the candidate base table(s) in this workspace.
3. Call get_view_columns on the most likely table to see its columns and data types --
   confirm the columns the goal needs actually exist before proceeding.
4. Call preview_data on that table to sanity-check the data shape (row count, sample values).
5. Choose the right report type for the goal: create_summary_report for a single group-by +
   aggregate, create_pivot_report for row/column/data cross-tabulation, or create_chart for a visual
   comparison. Call the matching tool with the confirmed table/column names.
Explain which report type you chose and why before creating it.
```

Missing argument → `Missing required prompt argument: <name>`.

---

## 12. Result encodings (json / toon / auto)

`--result-encoding` changes **only the result text**; schemas, arguments and JSON-RPC framing stay JSON. Errors are never TOON-encoded.

**TOON applies only when** the result is a non-empty JSON array of flat objects that all share the same key set, keys match `[A-Za-z_][A-Za-z0-9_]*`, and no value is nested. Columns are sorted alphabetically.

Wire format: `[<rowCount>]{<col1>,<col2>,…}:` then one comma-joined row per line. Numbers/booleans bare; `null` bare; text quoted only when it is empty, has edge whitespace, equals `null`/`true`/`false`, *looks like a number*, or contains `, : " \ { } [ ]` or control characters (escaped with `\`).

```json
[
  {"groupId": 101, "groupName": "Analysts",  "memberCount": 4,  "isDefault": false},
  {"groupId": 102, "groupName": "Ops, EMEA", "memberCount": 12, "isDefault": true},
  {"groupId": 103, "groupName": "2024",      "memberCount": 0,  "isDefault": false}
]
```
becomes
```text
[3]{groupId,groupName,isDefault,memberCount}:
101,Analysts,false,4
102,"Ops, EMEA",true,12
103,"2024",false,0
```

**auto** = TOON only when the JSON is **> 1000 characters**, table-shaped, **and** the TOON text is actually shorter; otherwise unchanged JSON. Results that are objects (`get_*`, every `{"status":…}` confirmation, `preview_data`) or plain text (`list_workspaces`, `list_views`) are never converted.

---

## 13. Calling conventions and gotchas for agent authors

1. **Discover IDs first**: `list_workspaces` → `list_views` → `get_view_columns`. Never guess IDs; copy them whole.
2. **Pass IDs as strings** (`"2148712000000012345"`), exactly as returned.
3. **Criteria syntax** is Zoho SQL-like: column in double quotes, value in single quotes: `"Region" = 'East'`, or fully qualified `"Orders"."Region"='East'`. Match-all criteria are refused on update/delete.
4. **Files** (`export_data.outputPath`, `import_data.filePath`) must be inside the server's working directory; the MCP client's launch directory decides that.
5. **Idempotency hints** tell you what is safe to retry. `add_row`, `import_*`, `create_*` may duplicate on retry.
6. **Dry-run servers** return `{"dryRun": true, …}` as a *success*; check for that field before claiming a change happened.
7. **Every result's `_meta.cliCommand`** (e.g. `view share`) lets you tell the human what you did in CLI terms; **`_meta.correlationId`** lets them find the log line.
8. In `--profiles` mode every call needs `"profile": "<name>"`; wrong/missing names return `isError` text such as `Missing "profile" argument. Configured profiles: [acme-prod, acme-staging]` or `Unknown "profile" argument: "gamma". Configured profiles: […]`.
9. Calls are **serialized** inside the server (one at a time) because the SDK token-refresh state is mutable; parallel tool calls from a client are queued, not rejected.
