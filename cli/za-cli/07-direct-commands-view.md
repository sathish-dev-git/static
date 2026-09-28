# 07 · Direct Commands — `view` (views, tables, rows, sharing, modelling)

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Every `view` command takes the workspace with **`-w/--workspace`** (required) and, except `list` and the `create-*` commands, the view ID as a **positional**. All need a login and accept the common options ([04](04-global-options-and-output.md)).

```text
Usage: za-cli view [-hV] [COMMAND]
View/Table operations
Commands:
  list                List views/tables in a workspace
  get                 Get view details
  columns             Get column metadata for a view/table
  publish-config      Get the publish/share configuration for a view
  url                 Get the direct URL for a view
  embed-url           Get the embeddable URL for a view
  publish             Make a view publicly accessible
  share-info          Show who a view is shared with
  share               Share a view with users and permissions
  remove-share        Remove a view's sharing for users
  rename              Rename a view/table
  delete              Delete a view/table permanently
  add-column          Add a column to a table
  add-row             Add a row to a table
  update-row          Update rows matching a criteria
  delete-rows         Delete rows matching a criteria
  preview             Preview rows from a view
  import-rows         Import inline JSON rows into a table
  create-table        Create a table (--column "Name:TYPE" repeatable, or --json)
  create-query-table  Create a query table from a SQL SELECT
  create-chart        Create a chart (--x "col:op" --y "col:op", or --json)
  create-summary      Create a summary report (--group/--agg "col:table:op", or --json)
  create-pivot        Create a pivot report (--row/--col/--data "col:table:op", or --json)
```

| Command | Safety | Via catalog tool | `--dry-run` | Confirm (default) |
|---|---|---|---|---|
| `list` | READ ONLY | direct SDK (`list_views`) | — | — |
| `get`, `columns`, `publish-config`, `share-info` | READ ONLY | `get_view_info`, `get_view_columns`, `get_publish_config`, `get_view_share_details` | — | — |
| `url`, `embed-url` | READ ONLY | direct SDK (`get_view_url`, `get_embed_url`) | — | — |
| `publish` | CHANGES | direct SDK (`make_view_public`) | — | — |
| `share`, `remove-share` | CHANGES | `share_view`, `remove_view_share` | ✅ | — |
| `rename`, `delete` | CHANGES / **DELETES** | direct SDK (`rename_view`, `delete_view`) | — | — (no catalog prompt) |
| `add-row`, `update-row`, `delete-rows` | CHANGES / CHANGES / **DELETES** | direct SDK (`add_row`, `update_rows`, `delete_rows`) | — | — |
| `preview` | READ ONLY | `preview_data` | — | — |
| `add-column` | CHANGES | `add_column` | ✅ | — |
| `import-rows` | CHANGES | `import_rows` | ✅ | — |
| `create-table`, `create-query-table`, `create-chart`, `create-summary`, `create-pivot` | CHANGES | `create_table`, `create_query_table`, `create_chart`, `create_summary_report`, `create_pivot_report` | ✅ | — |

> **Important for scripts:** `view delete` and `view delete-rows` call the SDK directly in 1.0.0, so they do **not** prompt and `--dry-run` has no effect on them. Export the data first; use the interactive menu (which previews affected rows and asks twice) for one-off deletions.

Sample IDs used below: workspace `2148712000000012345`, table **Orders** `2148712000000054321` with columns `OrderId NUMBER`, `Region PLAIN`, `Revenue CURRENCY`, `OrderDate DATE`.

---

## 1. Discovery

### 1.1 `view list -w <ws> [--offset N] [--limit N] [--all]`

```text
$ za-cli view list -w 2148712000000012345

  ╭──────────────────────────────────────────────────────────────────╮
  │ ● Views in Workspace 2148712000000012345                         │
  ├─────────────────────┬────────────────────────┬───────────────────┤
  │ VIEW ID             │ VIEW NAME              │ VIEW TYPE         │
  ├─────────────────────┼────────────────────────┼───────────────────┤
  │ 2148712000000054321 │ Orders                 │ Table             │
  │ 2148712000000054322 │ Customers              │ Table             │
  │ 2148712000000054330 │ Revenue by Region      │ Chart             │
  │ 2148712000000054335 │ Revenue Summary        │ Summary           │
  │ 2148712000000054340 │ Sales Dashboard        │ Dashboard         │
  ╰─────────────────────┴────────────────────────┴───────────────────╯
    5 row(s)
```

Row commands work on **Table** / **Query Table** views only. Filter: `--filter "viewType=Table"`. `-o json` returns the full array (`viewId, viewName, viewType, viewDesc, folderId, createdBy, createdTime, lastModifiedTime, …`).

### 1.2 `view get -w <ws> <viewId>`

```text
$ za-cli view get -w 2148712000000012345 2148712000000054321

  ╭────────────────────────────────────────────────────────────────╮
  │ ● Details                                                      │
  ├──────────────────────────┬─────────────────────────────────────┤
  │ View ID                  │ 2148712000000054321                 │
  │ View Name                │ Orders                              │
  │ View Type                │ Table                               │
  │ View Description         │ —                                   │
  │ Workspace ID             │ 2148712000000012345                 │
  │ Created By               │ admin@example.com                   │
  │ Created Time             │ 04 Mar 2026 11:30:01 IST            │
  │ Last Design Modified By  │ admin@example.com                   │
  │ Last Design Modified Time│ 11 Feb 2026 17:02:45 IST            │
  │ Folder ID                │ 2148712000000040001                 │
  │ No Of Rows               │ 1250                                │
  ╰──────────────────────────┴─────────────────────────────────────╯
```

Routed through the catalog, so internal noise fields (`orgId`, `createdByZuId`, `lastModifiedByName`, …) are stripped.

### 1.3 `view columns -w <ws> <viewId>`

```text
$ za-cli view columns -w 2148712000000012345 2148712000000054321

  ╭─────────────────────────────────────────────────────────────────────────────╮
  │ COLUMN NAME │ DATA TYPE NAME │ COLUMN ID           │ IS NULLABLE │ IS HIDDEN │
  ├─────────────┼────────────────┼─────────────────────┼─────────────┼───────────┤
  │ OrderId     │ NUMBER         │ 2148712000000054401 │ true        │ false     │
  │ Region      │ PLAIN          │ 2148712000000054402 │ true        │ false     │
  │ Revenue     │ CURRENCY       │ 2148712000000054403 │ true        │ false     │
  │ OrderDate   │ DATE           │ 2148712000000054404 │ true        │ false     │
  ╰─────────────┴────────────────┴─────────────────────┴─────────────┴───────────╯
    4 row(s)
```

Column names are **case-sensitive** in criteria and import headers. Compact form: `--fields columnName,dataTypeName -o csv`.

### 1.4 `view publish-config -w <ws> <viewId>`

Publish/share configuration object (public flag, access mode, link details) as a Details card or raw JSON. Use it to confirm a report is *not* public.

### 1.5 `view url` / `view embed-url` / `view publish`

```text
$ za-cli view url -w 2148712000000012345 2148712000000054330
  ╭───────────────────────────────────────────────────────────────────────╮
  │ ● View URL                                                            │
  ├───────────────────────────────────────────────────────────────────────┤
  │ https://analytics.zoho.com/workspace/2148712000000012345/view/2148712… │
  ╰───────────────────────────────────────────────────────────────────────╯

$ za-cli view embed-url -w 2148712000000012345 2148712000000054330 -o json
{"Embed URL": "https://analytics.zoho.com/open-view/2148712000000054330/…"}

$ za-cli view publish -w 2148712000000012345 2148712000000054340
  ╭───────────────────────────────────────────────────────────────╮
  │ ● Public URL                                                  │
  ├───────────────────────────────────────────────────────────────┤
  │ https://analytics.zoho.com/open-view/2148712000000054340/…    │
  ╰───────────────────────────────────────────────────────────────╯
```

`publish` makes the view viewable by **anyone with the link** — check the contents first. Embedding usually requires publishing first.

---

## 2. Sharing

### 2.1 `view share-info -w <ws> <viewId>`

```text
$ za-cli view share-info -w 2148712000000012345 2148712000000054321

  ╭──────────────────────────────────────────────────────────────────╮
  │ SHARED TO            │ SHARED BY          │ PERMISSION            │
  ├──────────────────────┼────────────────────┼───────────────────────┤
  │ alice@example.com    │ admin@example.com  │ Read Access           │
  │ Sales team           │ admin@example.com  │ Full Access           │
  │ Public Visitor       │ admin@example.com  │ Read Access           │
  ╰──────────────────────┴────────────────────┴───────────────────────╯
    3 row(s)
```

Raw rows (`-o json`): `[{"viewId":"…","sharedTo":"alice@example.com","sharedBy":"admin@example.com","permissionString":"Read Access","isGroupShare":false,"criteria":"","permissions":{"read":true,"export":false,…}}, …]`. Not shared → `(no data)`.

### 2.2 `view share -w <ws> <viewId> --email <e>… --permission <p>…`

```text
Valid permissions: read, export, vud, drillDown, addRow, updateRow, deleteRow,
                   deleteAllRows, importAppend, importAddOrUpdate, importDeleteAllAdd, share, discussion
```

`--email` and `--permission` are both required and each may be repeated **or** given comma-separated.

```text
$ za-cli view share -w 2148712000000012345 2148712000000054321 \
    --email alice@example.com --email bob@example.com \
    --permission read,export

  ╭────────────────────────────────────────╮
  │ ● Details                              │
  ├─────────┬──────────────────────────────┤
  │ Status  │ shared                       │
  │ View ID │ 2148712000000054321          │
  │ Count   │ 2                            │
  ╰─────────┴──────────────────────────────╯
```

- Always include `read`; without it nothing else grants access.
- Recipients must already be org members.
- Unknown flag → `Unknown permission 'write'. Valid permissions: read, export, vud, …` (exit 1). No permission → `At least one --permission is required.`
- Group sharing is disabled in 1.0.0 (emails only).

Permission meanings: `read` view · `export` download · `vud` view underlying data behind a summary · `drillDown` · `addRow`/`updateRow`/`deleteRow`/`deleteAllRows` row edits · `importAppend`/`importAddOrUpdate`/`importDeleteAllAdd` imports · `share` re-share · `discussion` comments.

### 2.3 `view remove-share -w <ws> <viewId> --email <e>…`

```text
$ za-cli view remove-share -w 2148712000000012345 2148712000000054321 --email bob@example.com
  │ Status  │ removed              │
  │ View ID │ 2148712000000054321  │
  │ Count   │ 1                    │
```

Only affects this view; workspace-level access (`workspace users`) is separate.

---

## 3. Rename and delete

```text
$ za-cli view rename -w 2148712000000012345 2148712000000054321 -n "Orders 2026"
  │ View renamed to 'Orders 2026'          │

$ za-cli view delete -w 2148712000000012345 2148712000000054399
  │ View 2148712000000054399 deleted successfully.   │
```

`delete` removes a table with all its rows (dependent reports break). It goes to the workspace trash (`workspace trash` / `restore-trash`). Views with dependents → `This view has dependent views and cannot be deleted until they are removed.` (7277).

---

## 4. Rows

### Criteria syntax (used by `update-row`, `delete-rows`, `export`)

Zoho Analytics SQL-like filter: **column in double quotes, value in single quotes**.

```text
"Region" = 'East'
"Revenue" > '1000'
"Region" != 'Test' AND "OrderDate" >= '2026-01-01'
"Region" LIKE '%east%'
"Orders"."Region" = 'East'                       -- table-qualified form
```

Shell quoting: wrap in double quotes and escape the inner ones — `--criteria "\"Region\" = 'East'"` — or use single quotes on the outside when the value has no single quotes: `--criteria '"Revenue" > 1000'`.

### 4.1 `view preview -w <ws> <viewId> [--limit <1-100>]`

```text
$ za-cli view preview -w 2148712000000012345 2148712000000054321 --limit 3

  ╭──────────────────────────────────────────────────────╮
  │ ● Preview — 3 row(s) (limit 3)                       │
  ├─────────┬────────┬─────────┬────────────┤
  │ ORDERID │ REGION │ REVENUE │ ORDERDATE  │
  ├─────────┼────────┼─────────┼────────────┤
  │ 1001    │ East   │ 1200.50 │ 2026-01-05 │
  │ 1002    │ West   │ 900.00  │ 2026-01-06 │
  │ 1003    │ East   │ 300.25  │ 2026-01-07 │
  ╰─────────┴────────┴─────────┴────────────╯
```

Default 20 rows, max 100 (values outside fall back to 20). Other formats print the raw object:

```json
{"format":"csv","requestedLimit":3,"shownLines":4,"previewLines":["OrderId,Region,Revenue,OrderDate","1001,East,1200.50,2026-01-05","1002,West,900.00,2026-01-06","1003,East,300.25,2026-01-07"]}
```

For everything, use `export`.

### 4.2 `view add-row -w <ws> <viewId> -d '<json>'`

```text
$ za-cli view add-row -w 2148712000000012345 2148712000000054321 \
    --data '{"Region":"West","Revenue":900,"OrderDate":"2026-09-22"}'

  ╭──────────────────────────────────────────────╮
  │ ● Details                                    │
  ├─────────┬────────────────────────────────────┤
  │ Columns │ {"OrderId":"1004","Region":"West",…│
  ╰─────────┴────────────────────────────────────╯
```

Wrap the JSON in single quotes so the shell keeps the double quotes. Omitted columns stay empty. Type mismatch → `A value does not match the data type of its column.` (7507); unknown column → `The column is not present in the specified table.` (7107).

### 4.3 `view update-row -w <ws> <viewId> -d '<json>' -c "<criteria>"`

```text
$ za-cli view update-row -w 2148712000000012345 2148712000000054321 \
    --data '{"Region":"East"}' --criteria "\"Region\" = 'Eastern'"

  ╭────────────────────────────────────────╮
  │ ● Details                              │
  ├─────────────────┬──────────────────────┤
  │ Updated Rows    │ 42                   │
  │ Updated Columns │ []                   │
  ╰─────────────────┴──────────────────────╯
```

Both `--data` and `--criteria` are required (no "update everything" shortcut). There is no affected-row preview here; check first with `export --criteria` using the same condition. Invalid criteria → `The specified criteria is invalid.` (8002) or `A column referenced in the criteria is not present in the table.` (8004).

### 4.4 `view delete-rows -w <ws> <viewId> (-c "<criteria>" | --all)`

```text
$ za-cli view delete-rows -w 2148712000000012345 2148712000000054321 --criteria "\"Region\" = 'Test'"
  │ 17 row(s) deleted.                     │

$ za-cli view delete-rows -w 2148712000000012345 2148712000000054321 --all
  │ 1250 row(s) deleted.                   │

$ za-cli view delete-rows -w 2148712000000012345 2148712000000054321
  ✗  [ZA1004] Operation failed
     Must specify --criteria or --all to delete rows. Use --criteria "<expression>" to delete matching rows, or --all to delete all rows in the view.
(exit 1)
```

Deleted rows cannot be recovered — export first. Because this command bypasses the catalog it does **not** prompt; the interactive **Delete Rows** flow shows the affected count and asks twice.

### 4.5 `view import-rows -w <ws> <viewId> --rows '<json array>' [--type append|truncateadd|updateadd]`

```text
$ za-cli view import-rows -w 2148712000000012345 2148712000000054321 \
    --rows '[{"Region":"East","Revenue":1200},{"Region":"West","Revenue":800}]'

  ╭──────────────────────────────────────────────╮
  │ ● Details                                    │
  ├────────────────┬─────────────────────────────┤
  │ Import Summary │ {"totalRowCount":2,"success…│
  ╰────────────────┴─────────────────────────────╯
```

`-o json` → `{"importSummary":{"totalRowCount":2,"successRowCount":2,"warnings":0}, …}`. Default mode `append`; `truncateadd` empties the table first; `updateadd` upserts (matching columns as configured by Zoho). For more than a handful of rows use `za-cli import` with a file.

---

## 5. Table structure

### 5.1 `view add-column -w <ws> <tableId> --name <col> --type <TYPE>`

Types: `PLAIN, NUMBER, DATE, EMAIL, CURRENCY, URL, POSITIVE_NUMBER, DECIMAL_NUMBER` (the interactive menu additionally offers `MULTI_LINE, PERCENTAGE, BOOLEAN, AUTO_NUMBER`).

```text
$ za-cli view add-column -w 2148712000000012345 2148712000000054321 --name Status --type PLAIN
  │ Status      │ added                │
  │ Column ID   │ 2148712000000054410  │
  │ Column Name │ Status               │
```

Duplicate → `A column with that name already exists in the table.` (7128). Columns cannot be removed or re-typed from the CLI.

### 5.2 `view create-table -w <ws> --name <n> --column "Name:TYPE"… | --json '<args>'`

```text
$ za-cli view create-table -w 2148712000000012345 --name Orders \
    --column "OrderId:NUMBER" --column "Region:PLAIN" --column "Revenue:CURRENCY" --column "OrderDate:DATE"

  │ Status     │ created              │
  │ Table ID   │ 2148712000000054321  │
  │ Table Name │ Orders               │
```

Equivalent with `--json` (full tool arguments; **overrides** every other flag):

```bash
za-cli view create-table --json '{
  "workspaceId": "2148712000000012345",
  "tableName": "Orders",
  "columns": [
    {"columnName": "OrderId", "dataType": "NUMBER"},
    {"columnName": "Region", "dataType": "PLAIN"},
    {"columnName": "Revenue", "dataType": "CURRENCY"},
    {"columnName": "OrderDate", "dataType": "DATE"}
  ]
}'
```

Duplicate table → `A table with that name already exists in the workspace.` (7111). The table starts empty — load it with `import`.

### 5.3 `view create-query-table -w <ws> --name <n> --query "SELECT …" | --json`

```text
$ za-cli view create-query-table -w 2148712000000012345 --name TopCustomers \
    --query "SELECT \"Region\", SUM(\"Revenue\") AS \"Total\" FROM \"Orders\" GROUP BY \"Region\""
  │ Status     │ created              │
  │ Table ID   │ 2148712000000054360  │
  │ Table Name │ TopCustomers         │
```

Single MySQL-compatible `SELECT`; table/column names must match exactly (case included). Syntax error → `The SQL query could not be parsed. Please check its syntax.` (7403).

---

## 6. Reports

Common spec grammar:

| Flag | Shape | Meaning |
|---|---|---|
| `--x`, `--y` (chart) | `Column:operation` | axis column and operation |
| `--group`, `--agg` (summary), `--row`, `--col`, `--data` (pivot) | `Column:Table:operation` | column, the table it belongs to, operation; repeatable |

Operations are free-form strings passed to Zoho. Non-aggregate for categories: `actual`, `dimension`; date buckets: `year`, `quarter`, `month`, `week`, `day`, …; aggregates for measures: `sum`, `count`, `average`, `min`, `max`, `distinctCount`, …. Wrong operation for a type → `Invalid operation for the column's data type. Use sum/average/min/max/count for numbers, actual/count/distinctCount for text, and year/month/week/etc. for dates.` (8166).

### 6.1 `view create-chart`

```text
za-cli view create-chart -w <ws> --base-table <table> --name <n> --type bar|line|pie|scatter|bubble --x "col:op" --y "col:op"
```

```text
$ za-cli view create-chart -w 2148712000000012345 --base-table Orders \
    --name "Revenue by region" --type bar --x "Region:actual" --y "Revenue:sum"
  │ Status      │ created              │
  │ Report Type │ chart                │
  │ Report ID   │ 2148712000000054330  │
```

`--json` form:

```bash
za-cli view create-chart --json '{"workspaceId":"2148712000000012345","tableName":"Orders","chartName":"Revenue by region","chartType":"bar",
  "xAxis":{"columnName":"Region","operation":"actual"},"yAxis":{"columnName":"Revenue","operation":"sum"}}'
```

Missing axis → `Missing required field: xAxis`-style error (exit 1).

### 6.2 `view create-summary`

```text
$ za-cli view create-summary -w 2148712000000012345 --base-table Orders \
    --name "Revenue Summary" --group "Region:Orders:actual" --group "OrderDate:Orders:year" --agg "Revenue:Orders:sum" --agg "OrderId:Orders:count"
  │ Status      │ created              │
  │ Report Type │ summary              │
  │ Report ID   │ 2148712000000054335  │
```

Never use `actual` on an aggregate entry.

### 6.3 `view create-pivot`

```text
$ za-cli view create-pivot -w 2148712000000012345 --base-table Orders \
    --name "Region by quarter" --row "Region:Orders:actual" --col "OrderDate:Orders:quarter" --data "Revenue:Orders:sum"
  │ Status      │ created              │
  │ Report Type │ pivot                │
  │ Report ID   │ 2148712000000054336  │
```

At least one of `--row/--col/--data` is required: `Provide at least one of 'row', 'column', or 'data' for the pivot report.`

Verify any creation with `view list -w <ws>`; get a link with `view url`.

---

## 7. Failure samples

```text
$ za-cli view list
Missing required option: '--workspace=<workspaceId>'
(exit 2)

$ za-cli view get -w 2148712000000012345 999
  ✗  [ZA2001] Operation failed
     The specified view/table ID does not exist.  (code 7104)
(exit 1)

$ za-cli view share -w 2148712000000012345 2148712000000054321 --email a@x.com --permission write
  ✗  [ZA1004] Operation failed
     Unknown permission 'write'. Valid permissions: read, export, vud, drillDown, addRow, updateRow, deleteRow, deleteAllRows, importAppend, importAddOrUpdate, importDeleteAllAdd, share, discussion
(exit 1)
```
