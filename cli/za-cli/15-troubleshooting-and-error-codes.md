# 15 · Troubleshooting, Exit Codes and Error Codes

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Source: `core/ErrorCode`, `core/ErrorHandler`, `tools/ToolErrors`, README.txt, help.html; captured runs of the shipped jar.

---

## 1. First step: `za-cli doctor`

Seven checks (Java, PATH, config folder, profiles, data centre, proxy, connectivity). ⚠ usually means "not set up yet"; only ✗ needs action and makes the exit code 1. Works without login. Paste its output (no secrets in it) together with the failing command run with `-v` when asking for help.

---

## 2. Exit codes

| Exit | Meaning | Typical causes |
|---|---|---|
| **0** | success | also `doctor` with only warnings, `logout` with no active profile |
| **1** | ran but failed | not logged in · unknown profile · wrong passphrase · Zoho API error · validation error (bad format, missing criteria/--all, unknown permission) · declined confirmation · `doctor` ✗ · `completion` unknown shell · alias errors |
| **2** | usage error (picocli) | unknown command/option · missing required option/argument · non-numeric where a number is expected · extra args to `alias run` |

`alias run` propagates the wrapped command's exit code.

---

## 3. za-cli error codes (stderr)

Every error za-cli itself prints (not picocli usage errors, not Zoho's own numeric codes) carries a stable bracketed code:

| Code | Message | Meaning | Fix |
|---|---|---|---|
| `ZA1001` | `Not logged in` / `Run 'za-cli login' to authenticate.` | no active, valid login for the profile in use | `za-cli login` or `za-cli profile switch <name>` |
| `ZA1002` | `Profile '<name>' does not exist.` / `Run 'za-cli profile list' to see available profiles.` | `-p` named an unknown profile | check `profile list` |
| `ZA1003` | `Invalid passphrase` / `Unable to decrypt credentials. Check your passphrase and try again.` | passphrase was obtained but does not unlock `profiles/<name>.enc` — i.e. an `env:`/`keychain:` reference *resolved* but to the wrong value | retry; check the value behind the keychain/env reference; lost → `za-cli login` again |
| `ZA1004` | `Operation failed` + friendly message + `Run with -v for the full stack trace.` | any other error (SDK exception, validation, network) — **including an `env:`/`keychain:` passphrase reference that cannot be resolved** | read the message; `-v` for the trace |
| `ZA2001` | `Operation failed` + message `(code NNNN)` | a catalog tool returned an error result; Zoho's numeric code is appended | see § 4 |

Captured examples:

```text
  ⚠  [ZA1001] Not logged in
     Run 'za-cli login' to authenticate.

  ✗  [ZA1002] Profile 'nope' does not exist.
     Run 'za-cli profile list' to see available profiles.
```

Other stderr forms: `⚠ DEPRECATED  …` (silence with `-q`); `✗  <message>` without a code for errors raised before a command runs (e.g. an invalid `-o` value); picocli `Unknown option: '--bogus'` + usage (exit 2).

---

## 4. Zoho Analytics error codes mapped by za-cli

When the Zoho API returns an error, za-cli replaces the raw message with a friendlier one and appends the numeric code. In MCP the same appears as `{"error": "...", "errorCode": NNNN}`.

### Request / validation
| Code | Message |
|---|---|
| 7003 | Required parameters are missing from the request. |
| 8078 | A required attribute was sent with an empty value. |
| 8079 | A required attribute is missing from the JSON configuration. |
| 8083 | The organization ID is missing from the request. |
| 8119 | An invalid value was provided for one of the attributes. |
| 8504 | A required parameter is missing or improperly formatted. |
| 8506 | A parameter was sent more times than allowed. |
| 8516 | An invalid value was passed in one of the parameters. |
| 8534 | The request contains invalid JSON. |

### Authentication / authorization
| Code | Message |
|---|---|
| 7301 | You do not have permission to perform this operation. |
| 7925 | You are not part of this Analytics organization. |
| 8023 | You do not have the required permission for this operation. |
| 8518 | Authentication failed. Your credentials may be invalid or expired. |
| 8535 | Invalid OAuth token. Please re-authenticate (run 'za-cli login'). |

### Workspaces, views, columns, folders
| Code | Message |
|---|---|
| 7101 | A workspace with that name already exists. Please choose a different name. |
| 7103 | The specified workspace ID does not exist. |
| 7104 | The specified view/table ID does not exist. |
| 7105 / 7138 | The specified view ID does not exist. |
| 7106 | The view is not present in this workspace. |
| 7107 | The column is not present in the specified table. |
| 7111 | A table with that name already exists in the workspace. |
| 7128 | A column with that name already exists in the table. |
| 7144 | The specified folder is not present in the workspace. |
| 7159 | This column cannot be deleted because it is used by reports, formula columns, or query tables. |
| 7161 | The specified table is a system table and cannot be modified. |
| 7165 | The specified workspace is a system workspace and cannot be deleted. |
| 7277 | This view has dependent views and cannot be deleted until they are removed. |
| 7387 | The specified workspace does not belong to the current organization. |
| 7395 | The column is not present in the workspace. |
| 8027 | The specified workspace or view was not found. |

### Rows and criteria
| Code | Message |
|---|---|
| 8002 | The specified criteria is invalid. |
| 8004 | A column referenced in the criteria is not present in the table. |
| 8016 | At least one column value is required for an insert or update. |
| 7507 | A value does not match the data type of its column. |
| 7511 | A mandatory column was left without a value. |

### Import / export / query
| Code | Message |
|---|---|
| 7232 | The import was aborted. |
| 7235 | None of the column names in the source data match the table's columns. |
| 7248 | The uploaded file content is not in the expected multipart/form-data format. |
| 7249 | The file could not be imported. |
| 7403 | The SQL query could not be parsed. Please check its syntax. |
| 8046 | None of the selected columns are valid. |
| 8120 / 8121 / 8122 / 8124 | Export job not found / was not initiated / has not completed yet / access denied. |
| 8133 | This view cannot be exported in the requested format. Dashboards must be exported as PDF. |
| 8137 / 8138 | Import job not found / access denied. |
| 8513 | The file size exceeds the supported limit. |
| 8166 | Invalid operation for the column's data type. Use sum/average/min/max/count for numbers, actual/count/distinctCount for text, and year/month/week/etc. for dates. |

### Users, groups, sharing
| Code | Message |
|---|---|
| 6021 | Cannot activate users: the user count would exceed your plan's allowed limit. |
| 6028 | The user is in an inactive state. |
| 7282 | A group with that name already exists. |
| 7338 | The group does not belong to the current workspace. |
| 8026 | At least one permission is required (e.g. read, export, addRow, updateRow, deleteRow, share). |
| 8060 | The specified domain does not exist. |
| 8509 | One or more email addresses are not in a valid format. |
| 8025 | Invalid custom domain. |

### HTTP-status fallbacks (when no code matched)
| HTTP | Message |
|---|---|
| 401 | Authentication failed or your session expired. Please re-authenticate (run 'za-cli login'). |
| 403 | You do not have permission to perform this operation. |
| 404 | The requested resource was not found. |
| 429 | Too many requests — the API rate limit was exceeded. Please wait a moment and try again. |
| 500 / 502 / 503 / 504 | Zoho Analytics reported a temporary server error. Please retry shortly. |

Anything else → Zoho's own message, or `Zoho Analytics API error`. Transient statuses (429, 5xx) are retried automatically up to 3 times before surfacing.

---

## 5. za-cli validation messages (exact text)

| Where | Message |
|---|---|
| `view delete-rows` without `-c`/`--all` | `Must specify --criteria or --all to delete rows. Use --criteria "<expression>" to delete matching rows, or --all to delete all rows in the view.` |
| `view share` bad permission | `Unknown permission '<p>'. Valid permissions: read, export, vud, drillDown, addRow, updateRow, deleteRow, deleteAllRows, importAppend, importAddOrUpdate, importDeleteAllAdd, share, discussion` |
| `view share` none | `At least one --permission is required.` |
| `export` bad `-f` (no console) | `Invalid format '<f>'. Supported: csv, json, xml, xls, pdf, html, image` |
| `export` existing file | `Output file already exists: <path>. Pass --force to overwrite, or choose a different --out path.` |
| `import` | `File not found: <path>` · `Cannot detect file type. Use --file-type to specify.` · `Invalid import type '<t>'. Supported: append, truncateadd, updateadd` · `Matching columns are required for updateadd imports.` |
| `--filter` | `Invalid --filter clause '<c>'. Expected field=value, field!=value, or field~value.` |
| `--proxy` | `Invalid proxy '<v>': expected host:port or http://[user:pass@]host:port` |
| `workspace users` without ID | `Workspace ID is required: za-cli workspace users <workspaceId>` |
| profile / alias names | `Name not allowed. Use 1-30 characters: letters, digits, hyphens, and underscores, starting with a letter or digit.` · `Alias name must be 1-30 characters of letters, digits, hyphens, and underscores, starting with a letter or digit.` · `Profile '<n>' already exists.` + `Profile names cannot differ only by capitalization.` |
| `profile switch` w/o credentials | `Profile '<n>' has no stored credentials. Run 'za-cli login --profile <n>' first.` |
| corrupt registry | `The profiles registry at <path> is corrupt. Fix or remove the file, then run 'za-cli login'.` |
| catalog tools | `Missing required field: <name>` · `Field must not be empty: columnValues` · `Refusing to update with a match-all criteria. Provide a criteria that selects specific rows.` · `criteria is required. To delete every row, set deleteAll to true.` · `Refusing to delete with a match-all criteria. Set deleteAll to true to intentionally delete every row.` · `Provide at least one of 'row', 'column', or 'data' for the pivot report.` · `Read-only mode: mutating operations are disabled` · `User cancelled the operation` |
| MCP startup | `Unknown --access-level: '<v>'. Valid values: read, write, full.` · `Invalid MCP result encoding '<v>'. Use one of: json, toon, auto.` · `Unknown tool: '<n>'. …` · `Unknown category: '<n>'. Valid categories: METADATA, ROW, DATA, VIEW, MODELLING, WORKSPACE, ADMIN, DIAGNOSTICS` · `--max-result-tokens must be positive, got: <n>` · `Unknown profile in --profiles: <n>. Registered profiles: […]` |
| secret references | `Environment variable 'X' referenced by 'env:X' is not set or is empty.` · `No macOS Keychain entry found for service 'za-cli', account '<a>'. Seed it first: security add-generic-password -s za-cli -a <a> -w '<secret-value>'` |

---

## 6. Symptom → cause → fix

| Symptom | Almost always because | Fix |
|---|---|---|
| `za-cli: command not found` | `bin` not on PATH, and no already-on-PATH directory was available for the installer to link into | read the PATH line printed by `setup.sh install` — it gives an `export PATH=…` line for the current shell; otherwise open a new terminal; [02 § 7](02-installation-and-setup.md#7-path-troubleshooting) |
| `Picked up _JAVA_OPTIONS: …` before every command | `_JAVA_OPTIONS` or `JAVA_TOOL_OPTIONS` is set in your environment | printed by the **Java runtime**, not za-cli, before any application code runs — no JVM flag or CLI option disables it. It goes to **stderr**, so piped and redirected output is unaffected. Unset the variable only if you do not need what it sets. |
| Other tools vanish from `PATH` after installing (`~/.local/bin` gone, `~/.bashrc` no longer loading at login) | a `~/.bash_profile` created by a **pre-1.0.0 build** of `setup.sh` — bash reads only the first of `~/.bash_profile`, `~/.bash_login`, `~/.profile`, so it hides the stock `~/.profile` that loads `~/.bashrc` and adds `~/.local/bin` | `sed -i '1i [ -f "$HOME/.profile" ] && . "$HOME/.profile"' ~/.bash_profile`, then open a new terminal. Current builds never create the file and print this same line when they detect it |
| `java: command not found` / wrong version | Java 17+ missing from PATH | install JDK/JRE 17+; `java -version` in the same terminal |
| `Missing required option: '--workspace=<workspaceId>'` | a `view`/`import`/`export`/`search` command needs `-w` | add `-w <id>` |
| `Unknown option: '-w'` | `workspace …` commands take the ID as a bare positional | drop `-w` |
| `Unknown option: '-p'` after the command | `-p`, `-v`, `-q`, `--no-color` must precede the command | `za-cli -p staging workspace list` |
| `Could not authenticate with these credentials` at login | refresh token copied with a space; wrong DC; wrong org ID | re-paste; pass `--dc`; verify org ID |
| `[ZA1003] Invalid passphrase` | wrong passphrase, wrong keychain account | check `security find-generic-password -s za-cli -a <acct> -w` |
| Export wrote nothing / wrong place | used `-o` instead of `-O` | `-O file.csv` |
| Export refuses to run | target file exists | `--force` |
| Import rejected every row | headers don't match table columns | `za-cli view columns …`; fix the header row |
| `updateadd` fails | no `--matching-columns` | add key columns |
| A table looks stale | data source sync stopped | `za-cli workspace datasources <ws> -d` |
| Slow / blocked behind firewall | proxy needed | `--proxy`, `HTTPS_PROXY`, or settings keys; raise `--connect-timeout`/`--read-timeout` |
| `Too many requests` (429) | rate limit; org-wide search or tight loops | scope searches with `-w`; raise `cache.ttlSeconds`; retries already happen automatically |
| Huge list output | list commands return everything | `--limit/--offset/--all`, `--filter`, `--fields` |
| `Picked up _JAVA_OPTIONS: …` on every run | JVM env var on your machine | harmless; goes to stderr only |
| MCP client cannot start server | relative path, missing passphrase, stdout pollution | absolute launcher path; `ZA_CLI_PASSPHRASE`; never print to stdout |
| Agent says it deleted but nothing changed | `--dry-run` server | check for `"dryRun": true` |
| Agent cannot delete | `--access-level write` never exposes destructive tools | use `full` if intended |
| Tab completion silent (bash) | `bash-completion` package not installed/sourced | [02 § 8](02-installation-and-setup.md#8-shell-completion-optional-but-recommended) |
| Interactive mode: `No console available. Run in a terminal.` | stdin is not a TTY (pipe, CI, some IDE consoles) | run in a real terminal |
| Colours/box characters garbled | terminal lacks Unicode/ANSI | `--no-color`, `ZA_CLI_ASCII=1` |
| Non-US account behaves oddly | only `us` DC verified end-to-end | report it; double-check `--dc` |

---

## 7. Getting more detail

| Add | Get |
|---|---|
| `-v` | full stack trace on error |
| `-vv` | plus preamble `[verbose] command=… profile=… outputFormat=… correlationId=…` |
| `--dry-run` | preview of catalog calls without doing them |
| `~/za-cli/logs/za-cli.0.log` | every request, retry, and audit event with the correlation ID |
| `za.cli.log.format=json` | machine-readable log lines |

Interactive mode: `v` shows the raw Zoho response for the current screen; `x` shows the direct command to reproduce it.
