# 10 · Helper Commands — `completion`, `doctor`, `telemetry`, `alias`

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> All four are **local** commands: no login, no passphrase, no common options (`-o`, `-P`, `--dry-run` are rejected as unknown options). Outputs below marked *captured* were produced by the shipped 1.0.0 jar.

---

## 1. `completion <bash|zsh|fish|powershell>`

```text
Usage: za-cli completion [-hV] <shell>
Generate a shell completion script (bash, zsh, fish, or powershell)
      <shell>     Shell: bash, zsh, fish, or powershell
```

Prints a completion script to **stdout**; installs nothing. `pwsh` is accepted as an alias of `powershell`; the shell name is case-insensitive.

| Shell | Generator | Completes |
|---|---|---|
| `bash` | picocli `AutoComplete` (3700+ lines) | subcommands, options, **option argument values** |
| `zsh` | picocli (`#compdef za-cli`, uses `bashcompinit`) | same as bash |
| `fish` | hand-rolled from the live command tree | subcommand names and option flags only |
| `powershell` | hand-rolled (`Register-ArgumentCompleter`) | subcommand names and option flags only |

Install destinations, bash-completion prerequisites and testing steps: [02 § 8](02-installation-and-setup.md#8-shell-completion-optional-but-recommended). Regenerate after every upgrade.

*Captured:*

```text
$ za-cli completion tcsh
Unknown shell 'tcsh'. Expected one of: bash, zsh, fish, powershell.
(exit 1)
```

---

## 2. `doctor`

```text
Usage: za-cli doctor [-hV]
Diagnose Java, config, credentials, proxy, and connectivity in one command
```

Seven independent checks, all run even if one fails. Exit **1 only if a check is ✗ FAIL**; ⚠ warnings exit 0. Never prints passwords.

| # | Check | ✓ OK | ⚠ WARN | ✗ FAIL |
|---|---|---|---|---|
| 1 | **Java** | `17.0.11 (za-cli requires 17+)` | — | `<v> — za-cli requires Java 17 or newer; found on <java.home>` |
| 2 | **PATH** | `'za-cli' resolves to /home/you/za-cli/bin/za-cli` | `'za-cli' not found on PATH — run it via its full install path, or re-run setup.sh/setup.bat` · `could not check PATH: <msg>` | — |
| 3 | **Config directory** | `/home/you/za-cli/config` | `<dir> does not exist yet — it is created on first login` | `<dir> exists but is not writable` |
| 4 | **Profiles** | `2 configured (active: production)` / `… (none active)` | `none configured — run 'za-cli login' first` | `N configured (active: x), M missing their credential file on disk` |
| 5 | **Data centre** | `United States [us]` (+ ` (unverified beyond US — see core.DataCentre)` for others) | `no active profile to report a data centre for` · `active profile 'x' not found in the registry` | — |
| 6 | **Proxy** | `none configured — direct connection` · `proxy.corp.example:3128 (authenticated)` | — | `configured proxy could not be resolved: <msg>` |
| 7 | **Connectivity** | `https://accounts.zoho.com reachable (HTTP 400)` (HEAD request through the resolved proxy, 5 s timeouts) | — | `https://accounts.zoho.com unreachable: <msg>` · proxy/URL resolution errors |

*Captured on a fresh machine (exit 0):*

```text
za-cli doctor  (za-cli 1.0.0)

  ✓  Java — 17.0.11 (za-cli requires 17+)
  ⚠  PATH — 'za-cli' not found on PATH — run it via its full install path, or re-run setup.sh/setup.bat
  ⚠  Config directory — /home/you/za-cli/config does not exist yet — it is created on first login
  ⚠  Profiles — none configured — run 'za-cli login' first
  ⚠  Data centre — no active profile to report a data centre for
  ✓  Proxy — none configured — direct connection
  ✓  Connectivity — https://accounts.zoho.com reachable (HTTP 400)
```

Healthy installation:

```text
za-cli doctor  (za-cli 1.0.0)

  ✓  Java — 17.0.11 (za-cli requires 17+)
  ✓  PATH — 'za-cli' resolves to /home/you/za-cli/bin/za-cli
  ✓  Config directory — /home/you/za-cli/config
  ✓  Profiles — 2 configured (active: production)
  ✓  Data centre — United States [us]
  ✓  Proxy — proxy.corp.example:3128 (authenticated)
  ✓  Connectivity — https://accounts.zoho.com reachable (HTTP 400)
```

`HTTP 400`/`200`/`204` all count as reachable. The proxy line is what za-cli **actually resolved** (flag → env → settings), so it is the quickest way to debug proxy precedence. `za-cli --proxy 127.0.0.1:1 doctor` exits 1 (connectivity FAIL). Include the full output when asking for help — it contains no secrets.

---

## 3. `telemetry [--clear]`

```text
Usage: za-cli telemetry [-hV] [--clear]
Show or clear locally accumulated usage counters (opt-in, off by default)
      --clear     Delete all locally accumulated counts
```

Local-only usage counting: **off by default**, enabled solely by `"telemetry.enabled": true` in `settings.json` (no CLI flag, no per-profile override). Records **only** `command:<path>` and `tool:<name>` keys with counts — no argument values, IDs, names, passphrases or org identifiers. There is **no server**; the file never leaves the machine.

*Captured (disabled, empty):*

```text
$ za-cli telemetry

  Telemetry  (local only — za-cli has no telemetry server to send this to)

  disabled (default)
  Toggle via "telemetry.enabled": true|false in /home/you/za-cli/config/settings.json

  (no usage recorded yet)
```

Enabled with data:

```text
  Telemetry  (local only — za-cli has no telemetry server to send this to)

  enabled
  Toggle via "telemetry.enabled": true|false in /home/you/za-cli/config/settings.json

  command:doctor  1
  command:org users  4
  command:workspace list  12
  tool:list_workspaces  12
  tool:search_metadata  3

  za-cli telemetry --clear   to delete this data
```

*Captured:*

```text
$ za-cli telemetry --clear
✓ Cleared locally accumulated telemetry.
```

`telemetry.json` (flat map, `<config-dir>/telemetry.json`):

```json
{ "command:workspace list": 12, "command:org users": 4, "tool:list_workspaces": 12, "tool:search_metadata": 3 }
```

Failed commands count too. Writes are merged asynchronously so concurrent za-cli processes add up correctly. MCP exposes the same data via `get_telemetry_status`, `get_telemetry_summary`, `clear_telemetry`.

---

## 4. `alias add | list | remove | run`

```text
Usage: za-cli alias [-hV] [COMMAND]
Name a command line so it can be re-run later
Commands:
  add     Save a command line under a name
  list    List saved aliases
  remove  Delete a saved alias
  run     Re-run a saved command line
```

### 4.1 `alias add <name> "<command line>"`

- Name rule (same as profiles): 1–30 chars, letters (either case), digits, hyphens, underscores, starting with a letter or digit.
- Save the command **without** the leading `za-cli`; wrap the whole thing in double quotes and use single quotes inside for values with spaces.
- Saving an existing name silently replaces it.

*Captured:*

```text
$ za-cli alias add my-ws "workspace list -o csv"
✓ Saved alias 'my-ws'.
  za-cli alias run my-ws

$ za-cli alias add 'Bad Name' "x"
Error: Alias name must be 1-30 characters of letters, digits, hyphens, and underscores, starting with a letter or digit.
(exit 1)
```

### 4.2 `alias list`

*Captured:*

```text
$ za-cli alias list

  my-ws  →  workspace list -o csv

  1 alias(es)

$ za-cli alias list            # nothing saved
  No aliases saved. Run 'za-cli alias add <name> "<command line>"' to create one.
```

### 4.3 `alias remove <name>`

*Captured:*

```text
$ za-cli alias remove my-ws
✓ Removed alias 'my-ws'.

$ za-cli alias remove my-ws
Error: no alias named 'my-ws'.
(exit 1)
```

### 4.4 `alias run <name>`

Re-invokes za-cli with the saved words exactly as if you had typed them: same global-option handling, same telemetry, **same exit code** as the wrapped command.

- **No extra arguments** are accepted: `za-cli alias run weekly-sales --force` → `Unknown option: '--force'` (exit 2). Save a second alias for a variant.
- The saved text is split like a shell would (whitespace, single/double quotes, `\"`/`\\` inside double quotes) but with **no** globbing, `$VAR` expansion or pipes.
- Whatever the wrapped command needs (login, passphrase) applies at run time:

*Captured (not logged in):*

```text
$ za-cli alias run my-ws
  ⚠  [ZA1001] Not logged in
     Run 'za-cli login' to authenticate.
(exit 1)
```

Unknown alias → `Error: no alias named 'x'. Run 'za-cli alias list' to see saved aliases.` (exit 1).

### 4.5 Storage

`<config-dir>/aliases.json`, flat map, owner-only permissions, written atomically:

```json
{
  "my-report": "workspace datasources 2148712000000012345 -o csv --fields workspaceId,workspaceName",
  "nightly-export": "export -w 2148712000000012345 -v 2148712000000054321 -f csv -O /data/nightly.csv --force"
}
```

### 4.6 Patterns

```bash
za-cli alias add admins "org users --filter role~admin --fields emailId,role -o csv"
za-cli alias add east-report 'export -w 2148712000000012345 -v 2148712000000054321 -f csv -O "/tmp/east report.csv" --criteria "\"Region\" = '"'"'East'"'"'"'
za-cli alias add staging-ws "-p staging workspace list"        # root options work inside the saved line

# cron (Linux/macOS)
0 7 * * 1 ZA_CLI_PASSPHRASE="keychain:default" /home/you/za-cli/bin/za-cli -q alias run weekly-sales
```

Renaming a profile does not update aliases that reference it (`-p old-name`); edit them by hand.
