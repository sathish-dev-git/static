# 04 · Global Options, Output Formats and Result Shaping

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Source: `ZaCli`, `command/BaseCommand`, `command/PaginationOptions`, `output/*`, `core/FieldProjection`, `core/RowFilter`, `core/Pagination`, `core/ErrorHandler`, plus captured runs of the shipped jar.

---

## 1. Where options go

| Option | Position | Notes |
|---|---|---|
| `-p/--profile`, `-v/--verbose`, `-q/--quiet`, `--no-color` | **Before** the command name (root only) | `za-cli -p staging workspace list` ✅ · `za-cli workspace list -p staging` ❌ `Unknown option: '-p'` (exit 2). `login`/`logout` accept their own `-p` after the name. |
| Every other common option | Before **or** after the command | `za-cli -o json workspace list` ≡ `za-cli workspace list -o json` |

Commands that accept **no** common options (local, no login): all `profile …`, all `alias …`, `completion`, `doctor`, `telemetry`, and the bare group names `workspace`, `view`, `org`, `profile`, `alias`.

Where the **workspace ID** goes:

| Family | How | Example |
|---|---|---|
| `workspace …` | bare positional | `za-cli workspace get 2148712000000012345` |
| `view …`, `import`, `export`, `search` | `-w` / `--workspace` | `za-cli view list -w 2148712000000012345` |

`Missing required option: '--workspace=<workspaceId>'` → you are on a view command and forgot `-w`. `Unknown option: '-w'` → you are on a workspace command; drop the flag.

Two `--limit`s: on the nine list commands it only trims what is **printed**; on `export` and `view preview` it is a real record cap sent to Zoho. `-o` (output format) vs `-O` (export file path) are different options.

---

## 2. Common options reference

```text
      --connect-timeout=<n>   Connection timeout in seconds (default: 5, or settings network.connectTimeoutSeconds)
      --dry-run               Preview a mutating/destructive tool call instead of running it; read-only calls are unaffected
      --envelope              With --output json, wrap results in the versioned {data, meta, error} envelope (opt-in; bare JSON is deprecated)
      --fields=<a,b,c>        Comma-separated field-name allowlist applied to the rendered result, regardless of --output
      --filter=<conds>        Comma-separated field=value/field!=value/field~value conditions (ANDed) narrowing an array result
  -h, --help                  Show help
      --no-color              Disable ANSI color output (same as NO_COLOR env var)                       [root only]
  -o, --output=<fmt>          table | json | csv | tsv | yaml | ndjson (default: table, or settings output.format)
  -p, --profile=<name>        Use a specific profile instead of the active one                          [root only]
  -P, --passphrase=<value>    Passphrase (or env:NAME / keychain:account reference) — avoids the prompt
      --proxy=<proxy>         host:port or http://[user:pass@]host:port (fallback: HTTPS_PROXY/https_proxy, then settings network.proxy)
  -q, --quiet                 Suppress deprecation notices and -vv preamble                              [root only]
      --read-timeout=<n>      Read timeout in seconds (default: 0 = none, or settings network.readTimeoutSeconds)
      --require-confirmation=<destructive|mutating|none>   Confirmation threshold (default destructive)
  -v, --verbose               Stack traces on error; -vv adds a per-command diagnostic preamble         [root only]
  -V, --version               Print version
  -y, --yes                   Skip confirmation prompts entirely (= --require-confirmation=none)
```

### 2.1 `-p` / `--profile`

Runs this one command against another saved profile without switching the active one. Unknown name:

```text
  ✗  [ZA1002] Profile 'nope' does not exist.
     Run 'za-cli profile list' to see available profiles.
(exit 1)
```

### 2.2 `-P` / `--passphrase` and `ZA_CLI_PASSPHRASE`

See [03 § 6](03-authentication-and-profiles.md#6-supplying-the-passphrase-without-a-prompt). Prefer the environment variable or a `keychain:` reference; `-P` is visible to other processes.

Precedence, highest first: `-P` on the **subcommand** → `-P` at the **root** → `ZA_CLI_PASSPHRASE` → interactive prompt. So `za-cli -P a workspace list -P b` uses `b`, and a `-P` at either level overrides the environment variable. za-cli reads `ZA_CLI_PASSPHRASE` exactly once from its process environment and never reads shell rc files — see [03 § 6.1](03-authentication-and-profiles.md#61-only-one-environment-variable-is-ever-read).

### 2.3 `-v` / `-vv` / `-q`

- `-v`: on error print the full stack trace instead of `Run with -v for the full stack trace.`
- `-vv`: additionally print a preamble to **stderr** before the command runs (captured):

```text
  [verbose] command=ListCmd profile= outputFormat=table correlationId=aa2fd063-cc52-4495-acde-089c6f68d85d
```

- `-q`: suppresses deprecation notices and the `-vv` preamble. Never suppresses data output or errors.

### 2.4 `--no-color` / `NO_COLOR`

Disables ANSI colours everywhere (also switches box characters to ASCII on terminals without Unicode). Useful when capturing output to files, though `-o csv|json` output never contains colour codes anyway.

### 2.5 `--proxy`, `--connect-timeout`, `--read-timeout`

Precedence for the proxy: `--proxy` → `ZA_CLI_SETTING_NETWORK_PROXY` → settings `network.proxy` → settings `network.proxy.host/.port/.username/.password` → `HTTPS_PROXY`/`https_proxy` → none.

```bash
za-cli --proxy proxy.corp.example:8080 workspace list
za-cli --proxy http://svc:pa55@proxy.corp.example:3128 login
za-cli --connect-timeout 15 --read-timeout 300 export -w … -v … -O big.csv
```

Invalid value is rejected before any request: `Invalid proxy '<v>': expected host:port or http://[user:pass@]host:port`. If the password contains `@`, `:` or `/`, use the four separate settings keys instead of a URL ([14](14-settings-files-and-logs.md)).

### 2.6 `--yes` and `--require-confirmation`

When a real console is attached, DESTRUCTIVE catalog calls prompt before running:

```text
  ⚠ Destructive operation requested: delete_workspace
     Arguments: {"workspaceId":"2148712000000012345"}
     Proceed? [y/N]: 
```

(MUTATING calls under `--require-confirmation=mutating` show `⚠ Confirm operation: <tool>` instead; import/export prompts also print the resolved `File:` path; argument keys containing `pass`, `secret`, `token` or `key` are redacted.) Only `y`/`yes` proceeds; anything else, EOF or no console counts as "no". Policy precedence: `--yes` > `--require-confirmation` > settings `confirmation.policy` > default `destructive`.

| Policy | Prompts for |
|---|---|
| `destructive` (default) | delete workspace / folder / group / trash view / rows / view |
| `mutating` | the above **plus** every create/rename/share/import/add/update call |
| `none` (= `--yes`) | nothing |

Scripts, CI, cron, pipes and redirects never prompt (no console) — which is why `--dry-run` matters when writing them. A declined prompt yields `{"error":"User cancelled the operation"}` → `[ZA2001]` exit 1. The interactive TUI has its own separate confirmations, unaffected by this policy.

```bash
za-cli workspace delete 2148712000000012345                              # prompts
za-cli --yes workspace delete 2148712000000012345                        # no prompt
za-cli --require-confirmation=mutating workspace create "New WS"         # prompt even for a create
```

### 2.7 `--dry-run`

Previews any MUTATING/DESTRUCTIVE **catalog** call without contacting Zoho and without prompting. Read-only calls run normally.

```bash
$ za-cli --dry-run view delete 2148712000000054321 -w 2148712000000012345
```
```text
  ╭──────────────────────────────────────────────────────────╮
  │ ● Details                                                │
  ├───────────┬──────────────────────────────────────────────┤
  │ Dry Run   │ true                                         │
  │ Tool      │ delete_view                                  │
  │ Safety    │ DESTRUCTIVE                                  │
  │ Args      │ {"workspaceId":"2148712000000012345","viewId… │
  │ Message   │ Dry run: no changes were made                │
  ╰───────────┴──────────────────────────────────────────────╯
    5 field(s)
```

With `-o json`:

```json
{"dryRun": true, "tool": "delete_view", "safety": "DESTRUCTIVE", "args": {"workspaceId": "2148712000000012345", "viewId": "2148712000000054321"}, "message": "Dry run: no changes were made"}
```

**Not covered** (these talk to the SDK directly, not through the catalog): `workspace list/get/create/users/users add/users remove/datasources`, `view list/url/embed-url/publish/rename/delete/add-row/update-row/delete-rows`, `import`, `export`, `org *`, `login`, `logout`. For those, `--dry-run` is accepted but has no effect — test with a read-only equivalent (e.g. `export --criteria` before `update-row`).

---

## 3. Output formats (`-o`)

| Format | Payload | Shape | Best for |
|---|---|---|---|
| `table` (default) | curated | Box-drawn tables / detail cards, colours, `N row(s)` footer, a `CLI:` hint line | humans |
| `json` | **full** | Pretty JSON (2-space). Bare payload is **deprecated**; add `--envelope` | scripts (`jq`) |
| `csv` | curated | RFC 4180, header row of human labels | spreadsheets |
| `tsv` | curated | same as csv, tab-delimited | `cut`, `awk` |
| `yaml` | full | block YAML, keys in the same order as the table | humans/config |
| `ndjson` | full | one compact JSON value per line (**one line per array element**) | streaming, `jq -c` |

"Curated" = the same columns the table shows (e.g. users → `Email ID`, `Role`, `Status`; workspaces → `Workspace ID`, `Workspace Name`, `Created Time`). "Full" = the complete, unmodified API payload.

Unknown format stops the command (exit 1) instead of falling back to table. The default can be changed with `output.format` in `settings.json` or `ZA_CLI_SETTING_OUTPUT_FORMAT`.

### 3.1 `table`

Rendering rules (from `TableFormatter`):

- Lists → bordered grid with a title bar (`● Owned Workspaces`), **UPPERCASED** human labels (`workspaceId` → `WORKSPACE ID`), max 50 chars per column (longer cells wrap), footer `N row(s)`.
- Single objects → two-column **Details** card (`● Details`), fields ordered: well-known keys first (`workspaceId, workspaceName, workspaceDesc, orgId, viewId, viewName, viewType, …, createdBy, createdTime, lastModifiedTime …`), the rest alphabetically by label. Footer `N field(s)`.
- Empty list → `  (no data)`.
- Missing/null values → `—`. Time fields (`createdTime`, `lastModifiedTime`, `modifiedTime`, `lastDesignModifiedTime`) as epoch millis/seconds or ISO are rendered `dd MMM yyyy HH:mm:ss z` (e.g. `04 Mar 2026 11:22:31 IST`).
- Nested objects of booleans (permission maps) collapse to the true keys (`Read, Export, Drill Down`); scalar arrays join with `, `; other nested values are compact one-line JSON.
- Inactive users are dimmed with the status cell highlighted.
- Every cell is sanitised (ANSI escapes and control characters stripped).
- A `CLI:` hint line follows many tables, e.g. `CLI: $ za-cli workspace list -P <passphrase>`.

Example:

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

### 3.2 `json` (bare — deprecated)

```bash
$ za-cli -o json workspace get 2148712000000012345
```
```text
  ⚠ DEPRECATED  --output json without --envelope returns the bare payload; pass --envelope to opt into the stable {data, meta, error} shape that becomes the default in the next major version.
```
(stderr) then on stdout:
```json
{
  "workspaceId": "2148712000000012345",
  "workspaceName": "Sales Analytics",
  "workspaceDesc": "Quarterly sales reporting",
  "orgId": "20087654321",
  "createdBy": "admin@example.com",
  "createdTime": "1772601751442"
}
```

Messages become `{"message": "…"}`; key/value results (URLs) become `{"View URL": "https://…"}`.

### 3.3 `json --envelope` (recommended for scripts)

```bash
$ za-cli -o json --envelope view list -w 2148712000000012345
```
```json
{
  "envelopeVersion": 1,
  "data": [
    {"viewId": "2148712000000054321", "viewName": "Orders", "viewType": "Table", "viewDesc": "", "createdTime": "1772602201000"},
    {"viewId": "2148712000000054330", "viewName": "Revenue by Region", "viewType": "Chart", "viewDesc": "", "createdTime": "1774150921491"}
  ],
  "meta": {
    "timestamp": "2026-09-22T10:14:02.117Z",
    "correlationId": "3f9c1d2e-8a41-4b77-9c1e-0d5a6f7b8c9d"
  },
  "error": null
}
```

- `envelopeVersion` is `1`. `data` is an object, array or string. `meta.correlationId` matches the log line for the run.
- **Errors are not enveloped**: failures still go to stderr with a non-zero exit code, so `stdout` is either a complete envelope or empty. (The `Envelope.failure` shape `{"data":null,"error":{"message":…,"code":…}}` exists in the library for future use.)
- `--envelope` has no effect on `csv`, `tsv`, `yaml`, `ndjson`.

```bash
za-cli -o json --envelope workspace list | jq -r '.data.ownedWorkspaces[] | "\(.workspaceId)\t\(.workspaceName)"'
```

### 3.4 `csv` / `tsv`

```bash
$ za-cli org users -o csv
Email ID,Role,Status
admin@example.com,Org Admin,Active
alice@example.com,User,Active
old.account@example.com,User,Deactivated
```

- Header uses human labels; single objects print a header line + one value line; empty arrays print nothing; primitive arrays print one value per line without a header.
- Quoting only when needed (`"Smith, John"`, `"6"" pipe"`); nulls → empty; nested values → compact JSON.
- **Formula-injection guard**: values starting with `=`, `+`, `-`, `@` are prefixed with `'` (`'=SUM(A1:A2)`).
- Messages → `message` header + text; key/values → `<key>` header + value.

### 3.5 `yaml`

```bash
$ za-cli workspace folders 2148712000000012345 -o yaml
- folderId: "2148712000000040001"
  folderName: Default
  isDefault: true
- folderId: "2148712000000040002"
  folderName: Q1 Reports
  isDefault: false
```

Strings that look like numbers/booleans/null or contain special characters are quoted (`zip: "007"`, `flag: "true"`); empty object `{}`, empty array `[]`, nulls `null`.

### 3.6 `ndjson`

```bash
$ za-cli view list -w 2148712000000012345 -o ndjson
{"viewId":"2148712000000054321","viewName":"Orders","viewType":"Table"}
{"viewId":"2148712000000054330","viewName":"Revenue by Region","viewType":"Chart"}
```

Ideal with `jq -c` or line-oriented tools; a single object is one line.

### 3.7 Special case: `view preview`

In `table` mode the CSV preview lines are parsed into a grid titled `Preview — 12 row(s) (limit 20)`; in every other format the raw `{format, requestedLimit, shownLines, previewLines[]}` object is printed.

---

## 4. `--fields` — column projection

- Comma-separated **top-level** key names (no dot paths), **case-sensitive**, applied after any `--filter`, to both objects and arrays, in **every** output format.
- Output keys follow the `--fields` order. A field the data lacks is **silently skipped**. In table/CSV it also replaces the curated column set.

```bash
za-cli workspace list --fields workspaceId,workspaceName -o csv
za-cli view columns -w 2148712000000012345 2148712000000054321 --fields columnName,dataTypeName
za-cli org users --fields emailId -o ndjson | jq -r .emailId
```

```text
$ za-cli workspace list --fields workspaceId,workspaceName -o csv
Workspace ID,Workspace Name
2148712000000012345,Sales Analytics
2148712000000023456,Marketing Warehouse
```

---

## 5. `--filter` — row selection

Comma-separated clauses, all must match (AND). Applies to **array results only** (single objects are untouched). Missing fields compare as empty strings.

| Clause | Operator | Semantics |
|---|---|---|
| `field=value` | equals | case-insensitive exact match |
| `field!=value` | not equals | negation (matches rows lacking the field) |
| `field~value` | contains | case-insensitive substring |

The parser takes the **first** operator character, so values containing `=` still work (`url~http://x?a=1`). Invalid clause → `[ZA1004] Operation failed` / `Invalid --filter clause 'x'. Expected field=value, field!=value, or field~value.` (exit 1).

```bash
za-cli org users --filter "role=admin" -o json --envelope
za-cli view list -w 2148712000000012345 --filter "viewType=Table,viewName~sales"
za-cli workspace users 2148712000000012345 --filter "status!=active" --fields emailId,status -o csv
```

`--filter` runs **before** `--fields`, so you may filter on a field you then drop.

---

## 6. Pagination: `--offset`, `--limit`, `--all`

Accepted **only** by: `org users`, `org admins`, `workspace list`, `workspace users`, `workspace datasources`, `workspace folders`, `workspace groups`, `workspace trash`, `view list`.

```text
      --offset=<N>   Number of items to skip before returning results (default: 0)
      --limit=<N>    Maximum number of items to return (default: all)
      --all          Return every item, ignoring --limit
```

- **Display-only**: Zoho has no server-side paging for these lists; the whole list is fetched, then sliced. It does not make the request smaller or faster.
- Clamping, never errors: negative offset → 0; offset beyond length → empty (`(no data)`); negative/zero limit → empty; `--all` beats `--limit`; huge limits do not overflow. Non-numeric values → picocli usage error, exit 2.
- `workspace list` applies the flags **independently** to the Owned and Shared tables (`--limit 5` → up to 5 of each).
- `workspace datasources --detail` slices the data sources first, then expands each.

```bash
za-cli org users --limit 20
za-cli org users --offset 20 --limit 20
za-cli view list -w 2148712000000012345 --all
```

---

## 7. Exit codes and error output

| Exit | Meaning | Examples |
|---|---|---|
| **0** | success (`doctor` with ⚠ warnings still exits 0) | |
| **1** | ran but failed | not logged in, unknown profile, wrong passphrase, Zoho error, validation error, cancelled confirmation, `doctor` with a ✗ |
| **2** | usage error from picocli | unknown command/option, missing required option or argument, non-numeric value |

All za-cli errors go to **stderr** with a stable bracketed code (blank line before and after):

```text
  ⚠  [ZA1001] Not logged in
     Run 'za-cli login' to authenticate.

  ✗  [ZA1002] Profile 'nope' does not exist.
     Run 'za-cli profile list' to see available profiles.

  ✗  [ZA1003] Invalid passphrase
     Unable to decrypt credentials. Check your passphrase and try again.

  ✗  [ZA1004] Operation failed
     <friendly message>
     Run with -v for the full stack trace.

  ✗  [ZA2001] Operation failed
     The specified workspace ID does not exist.  (code 7103)
```

Usage errors (exit 2) print picocli's message plus the command usage, e.g. `Unknown option: '--bogus'` or `Missing required option: '--workspace=<workspaceId>'`.

Full code table, Zoho error-code mapping and remedies: [15 · Troubleshooting](15-troubleshooting-and-error-codes.md).

---

## 8. Versioning and deprecation policy

- **SemVer**: `MAJOR.MINOR.PATCH`. MAJOR may break scripts; MINOR adds commands/options compatibly; PATCH fixes bugs.
- A deprecated behaviour keeps working for at least one more MAJOR version after the notice appears and is removed at the following MAJOR. Warnings go to **stderr** (never stdout) and are silenced by `--quiet`.

| Since | Deprecated | Replacement |
|---|---|---|
| 1.0.0 | `--output json` without `--envelope` (bare payload) | `--output json --envelope` |

(`mcp-server --readonly/--allow-writes` were replaced by `--access-level` before any release shipped, so they were removed outright rather than deprecated.)

---

## 9. Scripting checklist

1. Put the passphrase in `ZA_CLI_PASSPHRASE` (or `keychain:`/`env:` reference), never on the command line.
2. Use `-o json --envelope` (or `ndjson`/`csv`) and parse `stdout`; treat non-zero exit as failure and read `stderr` for `[ZAxxxx]`.
3. Add `--yes` only when you really want destructive calls to proceed unattended; remember that without a console nothing prompts anyway.
4. Preview with `--dry-run` while developing.
5. Pin behaviour with `--no-color -q` for clean logs.
6. Use `--fields`/`--filter` instead of `jq` where sufficient.

```bash
#!/usr/bin/env bash
set -euo pipefail
export ZA_CLI_PASSPHRASE='keychain:za-cli-prod'
ws=$(za-cli -q -o json --envelope workspace list \
      | jq -r '.data.ownedWorkspaces[] | select(.workspaceName=="Sales Analytics") | .workspaceId')
za-cli -q -o json --envelope view list -w "$ws" --filter "viewType=Table" --fields viewId,viewName
```
