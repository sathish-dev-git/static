# 09 · `search` — find workspaces, views and columns by name

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).

```text
Usage: za-cli search [-hVy] [--dry-run] [--envelope] [--connect-timeout=<n>] [--fields=<f>] [--filter=<f>]
                     [-o=<outputFormat>] [-P=<passphrase>] [--proxy=<proxy>] [--read-timeout=<n>]
                     [--require-confirmation=<p>] [-w=<workspaceId>] <query>
Search workspace/table/column names for a keyword
      <query>      Search text to match against names
  -w, --workspace=<workspaceId>
                   Limit the search to one workspace and also search its column names
```

READ ONLY · catalog tool `search_metadata` · interactive: Workspace menu → **Search** (workspace-scoped only).

---

## 1. What it searches

| Mode | Scope | Matches on | Cost |
|---|---|---|---|
| `za-cli search <query>` | whole organization | workspace names + view names (tables, charts, reports, dashboards) | one call per workspace's view list |
| `za-cli search <query> -w <ws>` | one workspace | view names **and column names** of table-like views | one call per table for column metadata |

Column search is deliberately opt-in per workspace: scanning every column of every table org-wide would mean one request per table and could trip your account's rate limit. za-cli also paces itself adaptively (0 → 200 ms → up to 2 s between calls after a slow response).

This is **text matching, not semantic search**: `churn` finds `customer_churn` but not `cancellation_reason`. Try shorter fragments or synonyms people actually typed.

Non-table view types (`chart`, `pivot`, `summary`, `report`, `kpi`, `analysisview`, `dashboard`) are never scanned for columns.

## 2. Scoring and ranking

| Score | Rule |
|---|---|
| **100** | exact match (case-insensitive) |
| **75** | query is a substring of the name |
| **50** | every word of a multi-word query appears somewhere in the name |
| **25** | at least one query word appears |
| 0 | excluded |

Results are sorted best-first and **capped at 50**, with a `note` when more existed. Workspace/view/column listings are served from the in-process cache for `cache.ttlSeconds` (default 60 s).

## 3. Examples

```text
$ za-cli search churn

  ╭──────────────────────────────────────────────────────────────────────────────────╮
  │ ● Details                                                                        │
  ├───────────────┬──────────────────────────────────────────────────────────────────┤
  │ Query         │ churn                                                            │
  │ Total Matches │ 3                                                                │
  │ Matches       │ [{"score":100,"matchedOn":"view name","workspaceId":"2148712000… │
  ╰───────────────┴──────────────────────────────────────────────────────────────────╯
```

The result is a single object, so the table view summarises the `matches` array; use JSON for a readable list:

```bash
$ za-cli search churn -o json
{
  "query": "churn",
  "totalMatches": 3,
  "matches": [
    {"score": 100, "matchedOn": "view name", "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054350", "viewName": "Churn"},
    {"score": 75,  "matchedOn": "view name", "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054351", "viewName": "Monthly churn by plan"},
    {"score": 25,  "matchedOn": "workspace name", "workspaceId": "2148712000000099999", "workspaceName": "Churn & Retention Lab"}
  ]
}
```

Workspace-scoped, with column hits:

```bash
$ za-cli search revenue -w 2148712000000012345 -o json
{
  "query": "revenue",
  "totalMatches": 4,
  "matches": [
    {"score": 100, "matchedOn": "column name", "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054321", "viewName": "Orders", "columnName": "Revenue"},
    {"score": 75,  "matchedOn": "view name",   "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054330", "viewName": "Revenue by Region"},
    {"score": 75,  "matchedOn": "view name",   "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054335", "viewName": "Revenue Summary"},
    {"score": 75,  "matchedOn": "column name", "workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "viewId": "2148712000000054322", "viewName": "Customers", "columnName": "LifetimeRevenue"}
  ]
}
```

Many hits:

```json
{"query": "sales", "totalMatches": 87, "matches": [ … 50 items … ], "note": "Showing the top 50 of 87 matches."}
```

No hits: `{"query":"xyzzy","totalMatches":0,"matches":[]}`.

Piping to the next command:

```bash
view=$(za-cli -q search "Orders" -w 2148712000000012345 -o json | jq -r '.matches[] | select(.matchedOn=="view name") | .viewId' | head -1)
za-cli view preview -w 2148712000000012345 "$view"
```

## 4. Result fields

| Field | Present on | Meaning |
|---|---|---|
| `query` | always | the text searched |
| `totalMatches` | always | matches before the 50 cap |
| `matches[].score` | always | 100 / 75 / 50 / 25 |
| `matches[].matchedOn` | always | `workspace name` · `view name` · `column name` |
| `matches[].workspaceId`, `workspaceName` | always | where it lives |
| `matches[].viewId`, `viewName` | view/column matches | |
| `matches[].columnName` | column matches only | |
| `note` | only when truncated | `Showing the top 50 of N matches.` |

## 5. Errors and edge cases

- Missing query → `Missing required parameter: '<query>'` (exit 2).
- Unknown `-w` → `The specified workspace ID does not exist.` (exit 1).
- A single table whose metadata call fails is skipped silently so one bad view never aborts the search.
- Per-workspace failures during an org-wide search are also swallowed; the rest of the results are returned.

## 6. MCP

`search_metadata` with `{"query": "...", "workspaceId": "..."}` (workspaceId optional) returns the same object. Agents should call it *before* guessing IDs and prefer the scoped form when they already know the workspace.
