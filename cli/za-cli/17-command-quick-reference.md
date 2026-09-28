# 17 · Command Quick Reference (cheat-sheet)

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Every line below is taken from the shipped jar's `--help`. Safety tags follow the handbook: READ ONLY · CHANGES DATA · DELETES DATA · SAVES A FILE · LONG RUNNING · RUNS ANYTHING.

## Legend

- `<workspace-id>` as a **bare positional** on `workspace …` commands; as **`-w`** on `view …`, `import`, `export`, `search`.
- `[…]` optional · `<…>` required value · `…` repeatable.
- Common options (`-o`, `--envelope`, `--fields`, `--filter`, `-P`, `-y`, `--require-confirmation`, `--dry-run`, `--proxy`, `--connect-timeout`, `--read-timeout`) are accepted by every command marked **★**. They are **not** accepted by `profile …`, `alias …`, `completion`, `doctor`, `telemetry`, nor by bare group names.
- Root-only options (must come **before** the command): `-p/--profile`, `-v/--verbose`, `-q/--quiet`, `--no-color`. (`login`/`logout` also accept their own `-p`.)

---

## Getting connected

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli login [-p <profile>] [--dc <dc>]` | CHANGES DATA | ★ | Authenticate, verify against Zoho, store encrypted credentials |
| `za-cli logout [-p <profile>]` | CHANGES DATA | ★ | Deactivate profile, keep encrypted credentials |
| `za-cli profile list` | READ ONLY | | List profiles, mark active |
| `za-cli profile add <name> [--dc <dc>]` | CHANGES DATA | | Create a profile (same prompts as login); does **not** activate it |
| `za-cli profile switch <name>` | CHANGES DATA | | Make a profile active |
| `za-cli profile remove <name>` | DELETES DATA | | Delete profile entry + `.enc` file (confirm prompt) |
| `za-cli profile info [<name>]` | READ ONLY | | Org ID, data centre, created time (no secrets) |
| `za-cli profile rename <old> <new>` | CHANGES DATA | | Rename profile and its credential file |

## Finding your data

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli workspace list [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Owned + shared workspaces |
| `za-cli workspace get <workspace-id>` | READ ONLY | ★ | One workspace's details |
| `za-cli view list -w <ws> [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Views in a workspace |
| `za-cli view get -w <ws> <view-id>` | READ ONLY | ★ | One view's details |
| `za-cli view columns -w <ws> <view-id>` | READ ONLY | ★ | Column names, types |
| `za-cli view preview -w <ws> <view-id> [--limit <1-100>]` | READ ONLY | ★ | First rows (default 20, max 100) |
| `za-cli search <query> [-w <ws>]` | READ ONLY | ★ | Name search; `-w` also searches column names |

## Getting data out

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli export -w <ws> -v <view> -O <file> [-f csv\|json\|xml\|xls\|pdf\|html\|image] [--criteria "…"] [--columns a,b] [--limit N] [--show-hidden-cols] [--show-personal-cols] [--force]` | SAVES A FILE | ★ | Export to a local file (`-O` capital!) |
| `za-cli view url -w <ws> <view-id>` | READ ONLY | ★ | Direct Zoho Analytics URL |
| `za-cli view embed-url -w <ws> <view-id>` | READ ONLY | ★ | Embeddable URL |

## Putting data in

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli import -w <ws> -v <view> -f <file> [-t append\|truncateadd\|updateadd] [--file-type csv\|json] [--auto-identify] [--matching-columns a,b] [--date-format fmt] [--delimiter comma\|tab\|semicolon\|pipe] [--on-error abort\|skiprow\|setcolumnempty] [--skip-rows N]` | CHANGES DATA | ★ | Import CSV/JSON file |
| `za-cli view import-rows -w <ws> <view-id> --rows '[{…}]' [--type append\|truncateadd\|updateadd]` | CHANGES DATA | ★ | Inline JSON rows |
| `za-cli view add-row -w <ws> <view-id> -d '{"Col":"v"}'` | CHANGES DATA | ★ | One row |
| `za-cli view update-row -w <ws> <view-id> -d '{…}' -c "<criteria>"` | CHANGES DATA | ★ | Update matching rows |
| `za-cli view delete-rows -w <ws> <view-id> (-c "<criteria>" \| --all)` | DELETES DATA | ★ | Delete matching or all rows |

## Building tables and reports

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli view create-table -w <ws> --name <n> --column "Name:TYPE"… \| --json '<args>'` | CHANGES DATA | ★ | New table (`PLAIN NUMBER DATE EMAIL CURRENCY URL POSITIVE_NUMBER DECIMAL_NUMBER`) |
| `za-cli view create-query-table -w <ws> --name <n> --query "SELECT …" \| --json` | CHANGES DATA | ★ | SQL-backed table |
| `za-cli view create-chart -w <ws> --base-table <t> --name <n> --type bar\|line\|pie\|scatter\|bubble --x "col:op" --y "col:op" \| --json` | CHANGES DATA | ★ | Chart |
| `za-cli view create-summary -w <ws> --base-table <t> --name <n> --group "col:table:op"… --agg "col:table:op"… \| --json` | CHANGES DATA | ★ | Summary report |
| `za-cli view create-pivot -w <ws> --base-table <t> --name <n> --row "col:table:op"… --col "…"… --data "…"… \| --json` | CHANGES DATA | ★ | Pivot report |
| `za-cli view add-column -w <ws> <table-id> --name <col> --type <TYPE>` | CHANGES DATA | ★ | Add a column |
| `za-cli view rename -w <ws> <view-id> -n "<new>"` | CHANGES DATA | ★ | Rename view/table |
| `za-cli view delete -w <ws> <view-id>` | DELETES DATA | ★ | Delete view/table (confirm) |

## Sharing and people

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli org users [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Org users |
| `za-cli org users add <email>…` | CHANGES DATA | ★ | Add org users |
| `za-cli org users remove <email>…` | CHANGES DATA | ★ | Remove org users |
| `za-cli org admins [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Org admins |
| `za-cli workspace users <ws> [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Workspace users + roles |
| `za-cli workspace users add <ws> [-r <role>] <email>…` | CHANGES DATA | ★ | Add (default role viewer) |
| `za-cli workspace users remove <ws> <email>…` | CHANGES DATA | ★ | Remove |
| `za-cli workspace groups <ws> [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Groups |
| `za-cli workspace create-group <ws> --name <n> --emails a,b` | CHANGES DATA | ★ | Create group |
| `za-cli workspace delete-group <ws> <group-id>` | DELETES DATA | ★ | Delete group |
| `za-cli workspace share-info <ws>` | READ ONLY | ★ | Workspace-wide sharing summary |
| `za-cli view share-info -w <ws> <view-id>` | READ ONLY | ★ | Who a view is shared with |
| `za-cli view share -w <ws> <view-id> --email <e>… --permission <p>…` | CHANGES DATA | ★ | Grant (`read export vud drillDown addRow updateRow deleteRow deleteAllRows importAppend importAddOrUpdate importDeleteAllAdd share discussion`) |
| `za-cli view remove-share -w <ws> <view-id> --email <e>…` | CHANGES DATA | ★ | Revoke |
| `za-cli view publish -w <ws> <view-id>` | CHANGES DATA | ★ | Make public, return link |
| `za-cli view publish-config -w <ws> <view-id>` | READ ONLY | ★ | Publish/share config |

## Managing workspaces

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli org info` | READ ONLY | ★ | Org + subscription details |
| `za-cli org resources` | READ ONLY | ★ | Resource usage vs plan |
| `za-cli workspace create "<name>"` | CHANGES DATA | ★ | New workspace |
| `za-cli workspace rename <ws> --name "<new>"` | CHANGES DATA | ★ | Rename |
| `za-cli workspace delete <ws>` | DELETES DATA | ★ | Delete workspace (no trash!) |
| `za-cli workspace folders <ws> [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Folders |
| `za-cli workspace create-folder <ws> --name "<n>"` | CHANGES DATA | ★ | New folder |
| `za-cli workspace delete-folder <ws> <folder-id>` | DELETES DATA | ★ | Delete folder |
| `za-cli workspace trash <ws> [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Trashed views |
| `za-cli workspace restore-trash <ws> <view-id>` | CHANGES DATA | ★ | Restore |
| `za-cli workspace delete-trash <ws> <view-id>` | DELETES DATA | ★ | Purge for good |
| `za-cli workspace datasources <ws> [-d] [--offset N] [--limit N] [--all]` | READ ONLY | ★ | Data sources; `-d` adds schedules + synced views |

## Tools and housekeeping

| Command | Safety | ★ | Purpose |
|---|---|---|---|
| `za-cli completion bash\|zsh\|fish\|powershell` | READ ONLY | | Print completion script |
| `za-cli doctor` | READ ONLY | | 7 environment checks |
| `za-cli telemetry [--clear]` | READ ONLY | | Local usage counters (opt-in) |
| `za-cli alias add <name> "<command line>"` | CHANGES DATA | | Save alias |
| `za-cli alias list` | READ ONLY | | List aliases |
| `za-cli alias remove <name>` | CHANGES DATA | | Delete alias |
| `za-cli alias run <name>` | RUNS ANYTHING | | Run saved command (no extra args allowed) |
| `za-cli mcp-server [--access-level read\|write\|full] [--result-encoding json\|toon\|auto] [--tools …] [--exclude-tools …] [--categories …] [--max-result-tokens N] [--profiles a,b]` | LONG RUNNING | ★ | MCP server over STDIO |
| `za-cli --version` / `za-cli <cmd> --help` | | | Version / help |

---

## Global option cheat-sheet

```text
-o, --output <table|json|csv|tsv|yaml|ndjson>   output format (default table or settings output.format)
    --envelope                                   with -o json: {envelopeVersion,data,meta,error}
    --fields a,b,c                               keep only these fields (any format)
    --filter "f=v,f!=v,f~v"                      AND-ed conditions on list results (=,!= exact ci; ~ substring ci)
-p, --profile <name>        (root only)          use a profile for this run
-P, --passphrase <value|env:NAME|keychain:acct>  passphrase without prompt (prefer ZA_CLI_PASSPHRASE env var)
-y, --yes                                        never prompt (= --require-confirmation=none)
    --require-confirmation <destructive|mutating|none>
    --dry-run                                    preview mutating/destructive catalog calls
    --proxy <host:port|http://[u:p@]host:port>   else HTTPS_PROXY/https_proxy, else settings network.proxy
    --connect-timeout <s>   (default 5)          --read-timeout <s> (default 0 = none)
-v / -vv (root only)                             stack traces / + diagnostic preamble with correlationId
-q, --quiet (root only)                          hide deprecation notices and -vv preamble
    --no-color (root only)                       same as NO_COLOR env var
-h, --help   -V, --version
```

## Environment variables

| Variable | Purpose |
|---|---|
| `ZA_CLI_PASSPHRASE` | Passphrase or `env:NAME` / `keychain:account` reference |
| `ZA_CLI_SETTING_<KEY>` | Override any settings key, e.g. `ZA_CLI_SETTING_OUTPUT_FORMAT=json` |
| `HTTPS_PROXY` / `https_proxy` | Proxy fallback |
| `NO_COLOR` | Disable colour |
| `ZA_CLI_NO_ANIM` | Disable the interactive spinner |
| `ZA_CLI_HOME` / `ZA_CLI_INSTALL_DIR` | Install folder for `setup.sh` |
| JVM `-Dza.cli.home`, `-Dza.cli.config` | Home folder (set by launcher) / separate config folder |

## Exit codes and error codes

| Exit | Meaning |
|---|---|
| 0 | Success (warnings in `doctor` still exit 0) |
| 1 | Ran but failed: `ZA1001` not logged in · `ZA1002` unknown profile · `ZA1003` wrong passphrase · `ZA1004` unexpected error · `ZA2001` tool/Zoho error |
| 2 | Usage error (unknown option, missing required option/argument) – picocli prints usage |
