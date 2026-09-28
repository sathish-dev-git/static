# 06 · Direct Commands — `workspace`

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Workspace commands take the workspace ID as a **bare positional** (no `-w`). All need a login and accept the common options ([04](04-global-options-and-output.md)).

```text
Usage: za-cli workspace [-hV] [COMMAND]
Workspace operations
Commands:
  list           List all accessible workspaces
  get            Get workspace details
  create         Create a new workspace
  rename         Rename a workspace
  delete         Delete a workspace permanently
  users          List and manage workspace users
  share-info     Show workspace-wide view-sharing details
  folders        List folders in a workspace
  create-folder  Create a folder in a workspace
  delete-folder  Delete a folder from a workspace
  groups         List user groups in a workspace
  create-group   Create a user group in a workspace
  delete-group   Delete a user group from a workspace
  trash          List views in the workspace trash
  restore-trash  Restore a view from the workspace trash
  delete-trash   Permanently delete a view from the workspace trash
  datasources    List data sources in a workspace
```

| Command | Safety | Via catalog tool | `--dry-run` | Confirm prompt (default policy) | Pagination |
|---|---|---|---|---|---|
| `list` | READ ONLY | direct SDK (`list_workspaces` in MCP) | — | — | ✅ (owned & shared independently) |
| `get <ws>` | READ ONLY | direct SDK (`get_workspace_details`) | — | — | — |
| `create "<name>"` | CHANGES | direct SDK (`create_workspace`) | — | — | — |
| `rename <ws> --name` | CHANGES | `rename_workspace` | ✅ | — | — |
| `delete <ws>` | **DELETES** | `delete_workspace` | ✅ | ✅ | — |
| `users [<ws>]` | READ ONLY | direct SDK (`list_workspace_users`) | — | — | ✅ |
| `users add <ws> <emails>` | CHANGES | direct SDK (`add_workspace_users`) | — | — | — |
| `users remove <ws> <emails>` | CHANGES | direct SDK (`remove_workspace_users`) | — | — | — |
| `share-info <ws>` | READ ONLY | `get_workspace_share_info` | — | — | — |
| `folders <ws>` | READ ONLY | `list_folders` | — | — | ✅ |
| `create-folder <ws> --name` | CHANGES | `create_folder` | ✅ | — | — |
| `delete-folder <ws> <folderId>` | **DELETES** | `delete_folder` | ✅ | ✅ | — |
| `groups <ws>` | READ ONLY | `list_groups` | — | — | ✅ |
| `create-group <ws> --name --emails` | CHANGES | `create_group` | ✅ | — | — |
| `delete-group <ws> <groupId>` | **DELETES** | `delete_group` | ✅ | ✅ | — |
| `trash <ws>` | READ ONLY | `list_trash` | — | — | ✅ |
| `restore-trash <ws> <viewId>` | CHANGES | `restore_trash_view` | ✅ | — | — |
| `delete-trash <ws> <viewId>` | **DELETES** | `delete_trash_view` | ✅ | ✅ | — |
| `datasources <ws> [-d]` | READ ONLY | direct SDK (`list_datasources`) | — | — | ✅ |

---

## 1. `workspace list`

```text
za-cli workspace list [--offset <N>] [--limit <N>] [--all]
```

Two tables — workspaces you own and workspaces shared with you (the second only if non-empty). Columns `WORKSPACE ID`, `WORKSPACE NAME`, `CREATED TIME`.

```text
$ za-cli workspace list

  ╭─────────────────────────────────────────────────────────────────────────────╮
  │ ● Owned Workspaces                                                          │
  ├─────────────────────┬───────────────────────┬───────────────────────────────┤
  │ WORKSPACE ID        │ WORKSPACE NAME        │ CREATED TIME                  │
  ├─────────────────────┼───────────────────────┼───────────────────────────────┤
  │ 2148712000000012345 │ Sales Analytics       │ 04 Mar 2026 11:22:31 IST      │
  │ 2148712000000023456 │ Marketing Warehouse   │ 17 Jun 2026 09:05:00 IST      │
  ╰─────────────────────┴───────────────────────┴───────────────────────────────╯
    2 row(s)

  ╭─────────────────────────────────────────────────────────────────────────────╮
  │ ● Shared Workspaces                                                         │
  ├─────────────────────┬───────────────────────┬───────────────────────────────┤
  │ WORKSPACE ID        │ WORKSPACE NAME        │ CREATED TIME                  │
  ├─────────────────────┼───────────────────────┼───────────────────────────────┤
  │ 2148712000000034567 │ Finance – Shared      │ 21 Jan 2026 14:40:12 IST      │
  ╰─────────────────────┴───────────────────────┴───────────────────────────────╯
    1 row(s)

  CLI: $ za-cli workspace list -P <passphrase>
```

Raw payload (`-o json`): the Zoho `/workspaces` object

```json
{
  "ownedWorkspaces": [
    {"workspaceId": "2148712000000012345", "workspaceName": "Sales Analytics", "workspaceDesc": "Quarterly sales reporting", "createdBy": "admin@example.com", "createdTime": "1772601751442", "isDefault": false},
    {"workspaceId": "2148712000000023456", "workspaceName": "Marketing Warehouse", "workspaceDesc": "", "createdBy": "admin@example.com", "createdTime": "1781683500000", "isDefault": false}
  ],
  "sharedWorkspaces": [
    {"workspaceId": "2148712000000034567", "workspaceName": "Finance – Shared", "workspaceDesc": "", "createdBy": "cfo@example.com", "createdTime": "1769006412000"}
  ]
}
```

Variants:

```bash
za-cli workspace list --limit 5                      # up to 5 owned AND up to 5 shared
za-cli workspace list -o csv --fields workspaceId,workspaceName
za-cli workspace list --filter "workspaceName~sales"
```

Interactive: Org menu → **Workspaces**. MCP: `list_workspaces` (returns a numbered text summary).

---

## 2. `workspace get <workspaceId>`

```text
$ za-cli workspace get 2148712000000012345

  ╭──────────────────────────────────────────────────────────────╮
  │ ● Details                                                    │
  ├───────────────────────┬──────────────────────────────────────┤
  │ Workspace ID          │ 2148712000000012345                  │
  │ Workspace Name        │ Sales Analytics                      │
  │ Workspace Description │ Quarterly sales reporting            │
  │ Org ID                │ 20087654321                          │
  │ Created By            │ admin@example.com                    │
  │ Created Time          │ 04 Mar 2026 11:22:31 IST             │
  │ Is Default            │ false                                │
  ╰───────────────────────┴──────────────────────────────────────╯
    7 field(s)
```

Wrong ID → `[ZA1004] Operation failed` / `The specified workspace ID does not exist.` (Zoho 7103). Writing `-w` here → `Unknown option: '-w'` (exit 2).

---

## 3. `workspace create "<workspaceName>"`

```text
$ za-cli workspace create "Sales Analytics 2027"
  ✓  Workspace created!
     ID:    2148712000000099001
     Name:  Sales Analytics 2027
```

`-o json` → `{"workspaceId": "2148712000000099001", "workspaceName": "Sales Analytics 2027"}`. Quote names containing spaces. Duplicate name → `A workspace with that name already exists. Please choose a different name.` (7101). A new workspace is empty and visible only to you until you add users.

---

## 4. `workspace rename <workspaceId> --name "<new-name>"`

```text
$ za-cli workspace rename 2148712000000012345 --name "Sales Analytics 2026"

  ╭────────────────────────────────────────────╮
  │ ● Details                                  │
  ├──────────────┬─────────────────────────────┤
  │ Status       │ renamed                     │
  │ Workspace ID │ 2148712000000012345         │
  │ New Name     │ Sales Analytics 2026        │
  ╰──────────────┴─────────────────────────────╯
    3 field(s)
```

`-o json` → `{"status":"renamed","workspaceId":2148712000000012345,"newName":"Sales Analytics 2026"}`. `--name` is required (`Missing required option: '--name=<name>'`, exit 2). IDs never change, so aliases keep working.

---

## 5. `workspace delete <workspaceId>` — destructive

```text
$ za-cli workspace delete 2148712000000099001

  ⚠ Destructive operation requested: delete_workspace
     Arguments: {"workspaceId":"2148712000000099001"}
     Proceed? [y/N]: y

  ╭────────────────────────────────────────────╮
  │ ● Details                                  │
  ├──────────────┬─────────────────────────────┤
  │ Status       │ deleted                     │
  │ Workspace ID │ 2148712000000099001         │
  ╰──────────────┴─────────────────────────────╯
```

- The most destructive command za-cli has: every table, report and row inside is gone; **there is no trash for workspaces**.
- Prompts only when a console is attached and the policy is `destructive`/`mutating`; `--yes` skips; scripts never prompt.
- Preview first: `za-cli --dry-run workspace delete 2148712000000099001` → `{"dryRun":true,"tool":"delete_workspace","safety":"DESTRUCTIVE",…}`.
- System workspace → `The specified workspace is a system workspace and cannot be deleted.` (7165).

---

## 6. `workspace users [<workspaceId>]`

```text
za-cli workspace users <workspaceId> [--offset <N>] [--limit <N>] [--all]
```

```text
$ za-cli workspace users 2148712000000012345

  ╭──────────────────────────────────────────────────────────╮
  │ ● Workspace Users                                        │
  ├──────────────────────────┬──────────────┬────────────────┤
  │ EMAIL ID                 │ ROLE         │ STATUS         │
  ├──────────────────────────┼──────────────┼────────────────┤
  │ admin@example.com        │ Admin        │ Active         │
  │ alice@example.com        │ Viewer       │ Active         │
  ╰──────────────────────────┴──────────────┴────────────────╯
    2 row(s)
```

Omitting the ID → `Workspace ID is required: za-cli workspace users <workspaceId>` (exit 1).

### 6.1 `workspace users add <workspaceId> [-r <role>] <emails>…`

```text
$ za-cli workspace users add 2148712000000012345 alice@example.com bob@example.com -r viewer
  ╭────────────────────────────────────────╮
  │ ● Message                              │
  ├────────────────────────────────────────┤
  │ Added 2 user(s) as viewer              │
  ╰────────────────────────────────────────╯
```

`-r/--role` defaults to `viewer`; other values are passed to Zoho as-is (e.g. `admin`, `editor` — whatever roles your org defines). Users must already be in the organization (`org users add`).

### 6.2 `workspace users remove <workspaceId> <emails>…`

```text
$ za-cli workspace users remove 2148712000000012345 bob@example.com
  │ Removed 1 user(s) from workspace       │
```

Per-view shares granted directly (`view share`) survive this — check `view share-info`.

---

## 7. `workspace share-info <workspaceId>`

Workspace-wide sharing summary: which users and groups can see which views, plus public and private-link access. In table mode the nested structure is rendered as a Details card (nested maps summarised); use `-o json` for the full picture:

```json
{
  "sharedUsers": ["alice@example.com"],
  "userPermissions": {
    "alice@example.com": [
      {"viewId": 2148712000000054321, "viewName": "Orders", "sharedBy": "admin@example.com",
       "permissions": {"read": true, "export": true, "vud": false, "addRow": false, "updateRow": false, "deleteRow": false,
                       "deleteAllRows": false, "importAppend": false, "importAddOrUpdate": false, "importDeleteAllAdd": false,
                       "importDeleteUpdateAdd": false, "drillDown": true, "share": false, "discussion": false}}
    ]
  },
  "groupNames": ["Sales team"],
  "groupPermissions": { "Sales team": [ { "viewId": 2148712000000054330, "viewName": "Revenue by Region", "sharedBy": "admin@example.com", "permissions": { "read": true } } ] },
  "publicPermissions": [ { "viewId": 2148712000000054340, "viewName": "Sales Dashboard", "permissions": { "read": true } } ],
  "privateLinkPermissions": []
}
```

For one view use `view share-info`. Interactive: read-only **Share Info** exists per view, not per workspace.

---

## 8. Folders

```text
za-cli workspace folders <workspaceId> [--offset <N>] [--limit <N>] [--all]
za-cli workspace create-folder <workspaceId> --name "<folder-name>"
za-cli workspace delete-folder <workspaceId> <folderId>
```

```text
$ za-cli workspace folders 2148712000000012345

  ╭─────────────────────────────────────────────────────────────────╮
  │ FOLDER ID           │ FOLDER NAME   │ IS DEFAULT │ PARENT FOLDER ID │
  ├─────────────────────┼───────────────┼────────────┼──────────────────┤
  │ 2148712000000040001 │ Default       │ true       │ -1               │
  │ 2148712000000040002 │ Q1 Reports    │ false      │ -1               │
  ╰─────────────────────┴───────────────┴────────────┴──────────────────╯
    2 row(s)
```

(Columns are derived from the payload; fields blank in every row are dropped.)

```text
$ za-cli workspace create-folder 2148712000000012345 --name "Q2 Reports"
  │ Status    │ created              │
  │ Folder ID │ 2148712000000040003  │
  │ Name      │ Q2 Reports           │

$ za-cli workspace delete-folder 2148712000000012345 2148712000000040003

  ⚠ Destructive operation requested: delete_folder
     Arguments: {"workspaceId":"2148712000000012345","folderId":"2148712000000040003"}
     Proceed? [y/N]: y
  │ Status    │ deleted              │
  │ Folder ID │ 2148712000000040003  │
```

Folders cannot be renamed from the CLI. Unknown folder → `The specified folder is not present in the workspace.` (7144).

---

## 9. Groups

```text
za-cli workspace groups <workspaceId> [--offset <N>] [--limit <N>] [--all]
za-cli workspace create-group <workspaceId> --name "<group-name>" --emails <a@x.com,b@x.com>
za-cli workspace delete-group <workspaceId> <groupId>
```

```text
$ za-cli workspace create-group 2148712000000012345 --name "Sales team" --emails alice@example.com,bob@example.com
  │ Status   │ created                 │
  │ Group ID │ 2148712000000067890     │
  │ Name     │ Sales team              │
```

`--name` and `--emails` are both required; `--emails` is one comma-separated string (an empty string creates an empty group). Members cannot be edited from the CLI afterwards (delete and recreate). Duplicate → `A group with that name already exists.` (7282). `delete-group` prompts (destructive) and removes any access the group conferred.

---

## 10. Trash

```text
za-cli workspace trash <workspaceId> [--offset <N>] [--limit <N>] [--all]
za-cli workspace restore-trash <workspaceId> <viewId>
za-cli workspace delete-trash <workspaceId> <viewId>
```

```text
$ za-cli workspace trash 2148712000000012345

  ╭──────────────────────────────────────────────────────────────────────╮
  │ VIEW ID             │ VIEW NAME     │ VIEW TYPE │ DELETED BY          │
  ├─────────────────────┼───────────────┼───────────┼─────────────────────┤
  │ 2148712000000054399 │ Old Orders    │ Table     │ alice@example.com   │
  ╰─────────────────────┴───────────────┴───────────┴─────────────────────╯
    1 row(s)

$ za-cli workspace restore-trash 2148712000000012345 2148712000000054399
  │ Status  │ restored             │
  │ View ID │ 2148712000000054399  │

$ za-cli workspace delete-trash 2148712000000012345 2148712000000054399

  ⚠ Destructive operation requested: delete_trash_view
     Arguments: {"workspaceId":"2148712000000012345","viewId":"2148712000000054399"}
     Proceed? [y/N]: y
  │ Status  │ permanently_deleted  │
  │ View ID │ 2148712000000054399  │
```

`delete-trash` is the point of no return. Deleted **views** go to the trash; deleted **workspaces** do not.

---

## 11. `workspace datasources <workspaceId> [-d|--detail]`

Lists the external connections feeding a workspace.

```text
$ za-cli workspace datasources 2148712000000012345

  ╭────────────────────────────────────────────────────────────────────────────────────────╮
  │ ● Data Sources                                                                         │
  ├──────────────┬──────────────────────────┬───────────────────┬────────┬──────────────────────┤
  │ CONNECTOR    │ SOURCE                   │ STATUS            │ SYNCS  │ LAST SYNC            │
  ├──────────────┼──────────────────────────┼───────────────────┼────────┼──────────────────────┤
  │ Google Drive │ sales.csv                │ success           │ 3/10   │ 2026-06-10 09:00:00  │
  │ Zoho CRM     │ —                        │ 2 schedule groups │ —      │ —                    │
  │ Web          │ https://static.zohocdn.… │ success           │ 1/10   │ 2026-06-11 10:00:00  │
  ╰──────────────┴──────────────────────────┴───────────────────┴────────┴──────────────────────╯
    3 row(s)
```

With `--detail`, each data source gets a heading, a **Schedules** table (`SCHEDULE / STATUS / SYNCS / NEXT SYNC / TABLES`) and one **Views — <schedule>** table (`VIEW / SOURCE / STATUS / LAST SYNC`) per sync interval:

```text
$ za-cli workspace datasources 2148712000000012345 -d

  Zoho CRM  —  crm.zoho.com

  ╭─────────────────────────────────────────────────────────────────╮
  │ ● Schedules                                                     │
  ├──────────┬─────────┬───────┬─────────────────────┬────────┤
  │ SCHEDULE │ STATUS  │ SYNCS │ NEXT SYNC           │ TABLES │
  ├──────────┼─────────┼───────┼─────────────────────┼────────┤
  │ Daily    │ Success │ 4/30  │ 2026-06-12 10:00:00 │ 1      │
  │ Hourly   │ Success │ 22/30 │ 2026-06-11 11:00:00 │ 2      │
  ╰──────────┴─────────┴───────┴─────────────────────┴────────╯

  ╭─────────────────────────────────────────────────────────╮
  │ ● Views — Daily                                         │
  ├────────┬──────────────┬─────────┬─────────────────────┤
  │ VIEW   │ SOURCE       │ STATUS  │ LAST SYNC           │
  ├────────┼──────────────┼─────────┼─────────────────────┤
  │ Orders │ orders_sheet │ Success │ 2026-06-11 10:00:00 │
  ╰────────┴──────────────┴─────────┴─────────────────────╯
```

Raw payload (`-o json`): array of `{datasourceName, datasourceId, source, sourceName, fileType, totalSyncAllowed, syncIntervals:[{schedule, lastDataSyncStatus, lastDataSyncTime, syncUsed, nextScheduleTime, tableDetails:[{viewName, sourceName, syncStatus, lastSyncTime}]}]}`. No sources → `No data sources configured.` Interactive: Workspace menu → **Datasources** (3-level drill-down).

---

## 12. Common failure samples

```text
$ za-cli workspace get 1
  ✗  [ZA1004] Operation failed
     The specified workspace ID does not exist.
     Run with -v for the full stack trace.
(exit 1)

$ za-cli workspace rename 2148712000000012345
Missing required option: '--name=<name>'
(exit 2)

$ za-cli workspace delete 2148712000000012345        # user answers "n"
  ✗  [ZA2001] Operation failed
     User cancelled the operation
(exit 1)
```
