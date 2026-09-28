# 12 · MCP Server Mode (AI agents)

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md). Every tool's schema and result shape: [13 · MCP tool catalog](13-mcp-tool-catalog.md).

---

## 1. What it is

`za-cli mcp-server` turns za-cli into a **Model Context Protocol** server. An MCP client (Claude Desktop, Claude Code, Cursor, ChatGPT desktop with local connectors, or any MCP-compatible agent) launches it as a subprocess and talks **JSON-RPC over STDIO**. The agent then sees the same 54-tool catalog the CLI uses as callable functions with JSON-Schema-described arguments, so it never has to guess flag syntax.

Key properties:

| Property | Value |
|---|---|
| Transport | STDIO only (no HTTP/SSE; nothing listens on a port) |
| Protocol | JSON-RPC (MCP), Java MCP SDK 2.0.0 |
| Capabilities | `tools` (always) · `prompts` (2 prompts; not in `--profiles` mode) |
| Default exposure | `--access-level=read` → **23 READ_ONLY tools only** |
| Concurrency | Tool calls are executed **one at a time** (serialized) per server process |
| Confirmation prompts | Never printed (the client is responsible for approving calls); `--yes` is implied |
| stdout | Reserved for JSON-RPC. za-cli never prints anything else there while serving |
| Lifetime | Blocks until the client kills the process (Ctrl+C by hand) |

> Running it in your own terminal "does nothing" — it waits silently for a client. That is correct behaviour.

---

## 2. Synopsis

```text
za-cli mcp-server [--access-level=<read|write|full>]
                  [--result-encoding=<json|toon|auto>]
                  [--tools=<name1,name2,…>]…
                  [--exclude-tools=<name1,…>]…
                  [--categories=<cat1,…>]…
                  [--max-result-tokens=<N>]
                  [--profiles=<p1,p2,…>]…
                  [--dry-run] [-P <passphrase>] [--proxy …] [--connect-timeout N] [--read-timeout N]
```

`--help` text from the shipped jar:

```text
Usage: za-cli mcp-server [-hVy] [--dry-run] [--envelope]
                         [--access-level=<accessLevelOption>]
                         [--connect-timeout=<connectTimeoutOption>]
                         [--fields=<fieldsOption>] [--filter=<filterOption>]
                         [--max-result-tokens=<maxResultTokensOption>]
                         [-o=<outputFormatOption>] [-P=<inlinePassphrase>]
                         [--proxy=<proxyOption>]
                         [--read-timeout=<readTimeoutOption>]
                         [--require-confirmation=<requireConfirmationOption>]
                         [--result-encoding=<resultEncodingOption>]
                         [--categories=<categoriesOption>[,<categoriesOption>...]]...
                         [--exclude-tools=<excludeToolsOption>[,<excludeToolsOption>...]]...
                         [--profiles=<profilesOption>[,<profilesOption>...]]...
                         [--tools=<toolsOption>[,<toolsOption>...]]...
Start MCP (Model Context Protocol) server
      --access-level=<accessLevelOption>
                   Coarse-grained tool category profile: read (only read-only
                     tools), write (read-only and mutating tools, but never
                     destructive ones -- e.g. never delete_view/delete_rows,
                     regardless of --tools/--categories), or full (every tool).
                     Default: read.
      --categories=<categoriesOption>[,<categoriesOption>...]
                   Only expose tools in these categories (comma-separated,
                     case-insensitive): METADATA, ROW, DATA, VIEW, MODELLING,
                     WORKSPACE, ADMIN, DIAGNOSTICS.
      --exclude-tools=<excludeToolsOption>[,<excludeToolsOption>...]
                   Never expose these tool names (comma-separated), even if
                     matched by --tools or --categories; unknown names fail the
                     server at startup.
      --max-result-tokens=<maxResultTokensOption>
                   Approximate token budget per tool result (default: unset,
                     matching tools.ToolContext's existing character-based
                     ceiling). Converted to a character budget via a
                     ~4-chars-per-token heuristic, not an exact count.
      --profiles=<profilesOption>[,<profilesOption>...]
                   Serve multiple profiles/orgs from one server session (4.3),
                     comma-separated profile names. Every named profile must
                     already exist and share this invocation's passphrase;
                     unknown names fail the server at startup. Every tool call
                     must then include a "profile" argument naming which one to
                     target. Prompts are not registered in this mode. Omit for
                     the default single-profile behavior (unchanged).
      --result-encoding=<resultEncodingOption>
                   MCP tool result encoding: json, toon, auto (default: json)
      --tools=<toolsOption>[,<toolsOption>...]
                   Only expose these tool names (comma-separated). Combined
                     with --categories as an intersection; unknown names fail
                     the server at startup.
```

The inherited `-o`, `--envelope`, `--fields`, `--filter`, `--require-confirmation` options are accepted but have no effect in server mode (results always go to the client as full JSON/TOON text).

---

## 3. Options in depth

### 3.1 `--access-level read | write | full` (default `read`)

| Level | Registers | Count | Typical use |
|---|---|---|---|
| `read` | READ_ONLY tools only | 23 | Exploration, audits, reporting assistants. Recommended first step. |
| `write` | READ_ONLY + MUTATING, **never** DESTRUCTIVE | 48 | Report builders, data loaders, onboarding bots. Cannot delete anything, whatever it is told. Future destructive tools are excluded automatically. |
| `full` | Every tool including the 6 DESTRUCTIVE (`delete_workspace`, `delete_folder`, `delete_group`, `delete_trash_view`, `delete_rows`, `delete_view`) | 54 | Only with good reason. |

Invalid value → `Unknown --access-level: '<value>'. Valid values: read, write, full.` and the server does not start.

A `read` server **refuses** a mutating call even if the client somehow names it: `{"error":"Read-only mode: mutating operations are disabled"}` (`isError=true`).

### 3.2 `--tools`, `--exclude-tools`, `--categories` (scoping)

- `--tools a,b` — allowlist of tool names.
- `--categories METADATA,VIEW` — allowlist of categories (case-insensitive).
- `--tools` ∩ `--categories` — both must match.
- `--exclude-tools` — applied last, always wins, even over an explicit `--tools` entry.
- `--access-level` is applied **on top**: scoping can only narrow, never widen.
- Each option may be repeated or given as one comma-separated list.
- **Unknown names fail fast** at startup:
  - `Unknown tool: '<name>'. See help.html's MCP tool catalog, or src/main/java/.../tools/README.md, for valid names.`
  - `Unknown category: '<name>'. Valid categories: METADATA, ROW, DATA, VIEW, MODELLING, WORKSPACE, ADMIN, DIAGNOSTICS`

```bash
# a reporting assistant: can look and export, nothing else
za-cli mcp-server --access-level read --categories metadata,row,data

# a report builder: can create, but never delete
za-cli mcp-server --access-level write --categories metadata,modelling,view --exclude-tools delete_view

# exactly two tools
za-cli mcp-server --tools=list_workspaces,create_workspace --access-level=full

# the packaged read-only "za-auditor" agent profile
za-cli mcp-server --access-level=read --categories=metadata,admin,workspace
```

### 3.3 `--result-encoding json | toon | auto` (default `json`)

Changes **only the text of successful tool results**. Schemas, arguments and JSON-RPC framing stay JSON; error results are always JSON.

| Value | Behaviour |
|---|---|
| `json` | Pretty-printed JSON (2-space indent). Safest. |
| `toon` | Compact table text for any result that is a uniform array of flat objects. |
| `auto` | TOON only when the result is > 1000 chars, table-shaped, **and** actually shorter as TOON. Recommended context saver. |

Invalid value → `Invalid MCP result encoding '<v>'. Use one of: json, toon, auto.`

Worked TOON example and exact rules: [13 § 12](13-mcp-tool-catalog.md).

### 3.4 `--max-result-tokens N`

Caps every tool result to roughly N tokens using a 4-chars-per-token estimate (`N × 4` characters). Default unset = 1,000,000 characters. Must be positive: `--max-result-tokens must be positive, got: <N>`. Over-budget JSON is trimmed key-by-key / element-by-element with a `(truncated: showing first N of T fields|items; refine your request to see the rest)` note.

```bash
za-cli mcp-server --max-result-tokens 2000     # ≈ 8,000 chars per result
```

### 3.5 `--profiles p1,p2,…` (multi-organization from one process)

Running one server per org costs ~33,000 characters of tool descriptions sent to the model per server plus ~0.5 s startup each. With `--profiles` one process serves several orgs:

- Every named profile must already exist and **share the passphrase** used to start the server. Unknown name → `Unknown profile in --profiles: <name>. Registered profiles: [a, b]` (checked before any credential file is opened).
- Every tool's schema gains **one required argument** `profile` ("Which configured profile/org this call should run against."). The client still sees exactly 54 tools, not 54 × profiles.
- Each profile gets its own SDK client, cache, retry and audit chain, so the audit log records which org each call touched; `_meta.profile` is added to successful results.
- Routing errors (`isError=true`, plain text): `Missing "profile" argument. Configured profiles: [acme-prod, acme-staging]` · `Unknown "profile" argument: "gamma". Configured profiles: […]`
- **Prompts are not registered** in this mode (v1 scope: tools only).

```bash
za-cli mcp-server --profiles=acme-prod,acme-staging --access-level=full
```

### 3.6 `--dry-run`

Applies to the **whole server session**: every MUTATING/DESTRUCTIVE call from any connected client returns a preview instead of running:

```json
{"dryRun": true, "tool": "create_workspace", "safety": "MUTATING", "args": {"name": "Test"}, "message": "Dry run: no changes were made"}
```

Delivered as a *successful* result (no `isError`). READ_ONLY tools run normally. Useful for testing an agent's plan against production credentials without risk.

### 3.7 Passphrase for a headless process

An MCP client cannot answer prompts, so supply the passphrase non-interactively:

```bash
ZA_CLI_PASSPHRASE='your-passphrase'      za-cli mcp-server        # literal (least secure)
ZA_CLI_PASSPHRASE='env:MY_ORG_ZA_PASS'   za-cli mcp-server        # indirection to another env var
ZA_CLI_PASSPHRASE='keychain:default'     za-cli mcp-server        # OS credential store (recommended)
za-cli mcp-server -P 'keychain:default'
```

See [03 · Authentication § secret references](03-authentication-and-profiles.md).

### 3.8 Network options

`--proxy`, `--connect-timeout`, `--read-timeout` (and the corresponding settings keys) apply to every request the server makes. Behind a corporate proxy put `--proxy host:port` in the client's `args` or rely on `settings.json`.

---

## 4. Startup sequence and failure modes

1. Parse options → resolve result encoding → access level → tool scope → token budget → profiles. Any invalid value fails **before** the transport starts.
2. Resolve passphrase (`-P`, `ZA_CLI_PASSPHRASE`, secret references). Missing/blank with no console → exit 1.
3. Unlock the profile's credentials (wrong passphrase → `[ZA1003] Invalid passphrase`), build the decorated service (cache → retry → audit).
4. Build tool specs (filtered by access level + scope), prompt specs (gated), start the STDIO transport, block.

Observed startup errors (captured from the jar with no profile configured):

```text
$ za-cli mcp-server --tools bogus_tool < /dev/null

  ⚠  [ZA1001] Not logged in
     Run 'za-cli login' to authenticate.
(exit 1)
```

The login check runs first; with a valid login the same command would fail with `Unknown tool: 'bogus_tool'. …`.

---

## 5. Protocol details agents and integrators should know

| Aspect | Detail |
|---|---|
| Tool input | JSON object matching the tool schema (`additionalProperties: false`). IDs as strings. |
| Tool output | One `text` content block. JSON pretty-printed (or TOON). Plain text for `list_workspaces` / `list_views` summaries. |
| `_meta` on every result | `correlationId` (UUID, matches the log line) and `cliCommand` (e.g. `view share`). In `--profiles` mode also `profile`. Present on error results too. |
| Errors | `isError: true`. Body is the executor's JSON `{"error": "...", "errorCode": <zoho code>}`, kept verbatim (never TOON). If an exception escaped the executor entirely the body is `<ExceptionClass>: <message>`. |
| Annotations | `title`, `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint=false`. |
| Size cap | 1,000,000 chars or `--max-result-tokens × 4`. |
| Cancelled | `{"error": "User cancelled the operation"}` only on interactive surfaces; never in MCP. |
| Working directory | `export_data` / `import_data` paths must be inside the server's cwd (set by the client). |
| Logging | Same rotating log as the CLI (`~/za-cli/logs/za-cli.N.log`); the audit logger `com.zoho.analytics.cli.audit` records every mutating call with profile, org, arguments (truncated at 200 chars), outcome and duration. |
| Telemetry | If `telemetry.enabled` is true, `tool:<name>` counters are incremented locally. |

Example successful call (conceptual JSON-RPC):

```json
// → request
{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"rename_view","arguments":{"workspaceId":"2148712000000012345","viewId":"2148712000000054321","newName":"Q1 Sales"}}}
// ← result
{"jsonrpc":"2.0","id":7,"result":{
  "content":[{"type":"text","text":"{\n  \"status\": \"renamed\",\n  \"viewId\": 2148712000000054321,\n  \"newName\": \"Q1 Sales\"\n}"}],
  "isError":false,
  "_meta":{"correlationId":"3f9c1d2e-8a41-4b77-9c1e-0d5a6f7b8c9d","cliCommand":"view rename"}}}
```

Example error:

```json
{"jsonrpc":"2.0","id":8,"result":{
  "content":[{"type":"text","text":"{\"error\":\"The specified view/table ID does not exist.\",\"errorCode\":7104}"}],
  "isError":true,
  "_meta":{"correlationId":"…","cliCommand":"view get"}}}
```

---

## 6. Prompts

Two curated multi-step prompts ship with the server (full text in [13 § 11](13-mcp-tool-catalog.md#11-prompts-2)):

| Prompt | Arguments | Needs tools | Available at |
|---|---|---|---|
| `audit_workspace` | `workspaceName` | 7 READ_ONLY tools | read / write / full |
| `build_report` | `workspaceName`, `goal` | list/inspect tools + `create_summary_report`, `create_pivot_report`, `create_chart` | write / full only |

A prompt is withheld if **any** tool it references is out of scope, and prompts are absent entirely with `--profiles`. Prompts never touch data; they return step-by-step instructions that resolve a workspace *name* to an ID first and stop to ask if zero or several match.

---

## 7. Client setup

Use an **absolute path** to the launcher (desktop apps do not inherit your shell PATH) — after a default install: `/Users/you/za-cli/bin/za-cli`, `/home/you/za-cli/bin/za-cli`, or `C:\Users\you\za-cli\bin\za-cli.bat`. `setup.sh`/`setup.bat` print the exact resolved path and a ready JSON snippet at the end of install. Never use `$HOME` or `%USERPROFILE%` inside the config; the client spawns za-cli without a shell.

Recommended pattern: register **three servers** (read / write / full) and enable the one you need per session.

### 7.1 Claude Desktop

Config file: macOS `~/Library/Application Support/Claude/claude_desktop_config.json` · Windows `%APPDATA%\Claude\claude_desktop_config.json` · Linux `~/.config/Claude/claude_desktop_config.json`. Fully quit and restart the app after editing.

```json
{
  "mcpServers": {
    "zoho-analytics-read": {
      "command": "/Users/YOUR_USER/za-cli/bin/za-cli",
      "args": ["mcp-server", "--access-level=read", "--result-encoding", "auto"],
      "env": { "ZA_CLI_PASSPHRASE": "keychain:default" }
    },
    "zoho-analytics-write": {
      "command": "/Users/YOUR_USER/za-cli/bin/za-cli",
      "args": ["mcp-server", "--access-level=write", "--result-encoding", "auto"],
      "env": { "ZA_CLI_PASSPHRASE": "keychain:default" }
    },
    "zoho-analytics-full": {
      "command": "/Users/YOUR_USER/za-cli/bin/za-cli",
      "args": ["mcp-server", "--access-level=full", "--result-encoding", "auto"],
      "env": { "ZA_CLI_PASSPHRASE": "keychain:default" }
    }
  }
}
```

Windows variant: `"command": "C:\\Users\\YOUR_USER\\za-cli\\bin\\za-cli.bat"`.

Seed the keychain once: macOS `security add-generic-password -s za-cli -a default -w 'your-passphrase'` · Linux `secret-tool store --label="za-cli default" service za-cli account default` · Windows `cmdkey /generic:za-cli:default /user:za-cli /pass:your-passphrase`.

Alternative `env` blocks:

```json
"env": { "ZA_CLI_PASSPHRASE": "your-actual-passphrase" }
```
```json
"env": { "ZA_CLI_PASSPHRASE": "env:MY_ORG_ZA_PASSPHRASE", "MY_ORG_ZA_PASSPHRASE": "your-actual-passphrase" }
```

### 7.2 Claude Code

- Project-scoped: `.mcp.json` at the repo root (shareable via git).
- User-scoped: `~/.claude.json` → `mcpServers`, or:

```bash
claude mcp add --scope user zoho-analytics-read /home/you/za-cli/bin/za-cli mcp-server --access-level=read
```

Same JSON shape as Claude Desktop; takes effect on the next `claude` session. Each server can be toggled on/off individually.

### 7.3 Cursor

- Project: `.cursor/mcp.json` · Global: `~/.cursor/mcp.json` · or Settings → MCP.
- Same JSON shape. Reload the window (*Developer: Reload Window*) after editing. Settings → MCP shows a per-server on/off toggle.

### 7.4 ChatGPT desktop

Settings → Connectors / Apps / Developer Mode, when your plan exposes custom MCP. It asks for the parts separately:

| Field | Value |
|---|---|
| Command | `/Users/YOUR_USER/za-cli/bin/za-cli` |
| Arguments | `mcp-server --access-level=read --result-encoding auto` |
| Environment | `ZA_CLI_PASSPHRASE=keychain:default` |

Register only the connector(s) you need (no per-server toggle there). If ChatGPT asks for a **server URL**, za-cli's STDIO server is not enough on its own; you would need a separate STDIO-to-HTTP MCP bridge, which za-cli does not ship.

### 7.5 First test

Ask the client: *"List my Zoho Analytics workspaces."* If nothing happens, check the client's MCP panel and run `za-cli doctor`; confirm `za-cli workspace list` works on its own first.

---

## 8. Recipes

**Dry-run everything an agent wants to do**

```json
"args": ["mcp-server", "--access-level=full", "--dry-run"]
```

**Keep context small**

```json
"args": ["mcp-server", "--access-level=read", "--result-encoding", "auto", "--max-result-tokens", "2000"]
```

**Two orgs, one server**

```json
"args": ["mcp-server", "--profiles=acme-prod,acme-staging", "--access-level=write", "--result-encoding", "auto"]
```

**Behind a proxy**

```json
"args": ["mcp-server", "--access-level=read", "--proxy", "proxy.corp.example:8080"]
```

**Audit trail** — every mutating tool call is one JSON audit line in `~/za-cli/logs/za-cli.0.log`:

```text
2026-09-22T10:14:02.117Z INFO [com.zoho.analytics.cli.audit] [corr=3f9c…] {"event":"analytics.renameWorkspace","timestamp":"2026-09-22T10:14:02.117Z","correlationId":"3f9c…","principal":"you","profile":"acme-prod","orgId":20087654321,"operation":"renameWorkspace","safetyClass":"MUTATING","arguments":["2148712000000012345","Sales Analytics 2026"],"outcome":"success","durationMs":412}
```

---

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Client says server failed to start | Use an absolute launcher path; make sure `ZA_CLI_PASSPHRASE` (or a `keychain:`/`env:` reference) is set in `env`; run `za-cli doctor`. |
| `[ZA1001] Not logged in` in the client log | The profile has no active login. Run `za-cli login` (or `za-cli profile switch`). |
| `[ZA1003] Invalid passphrase` | Wrong passphrase / wrong keychain entry. Verify with `security find-generic-password -s za-cli -a default -w`. |
| `Unknown tool: '…'` / `Unknown category: '…'` | Typo in `--tools/--exclude-tools/--categories`. Names in [13](13-mcp-tool-catalog.md). |
| Agent says it "deleted" but nothing changed | Server runs with `--dry-run`; results carry `"dryRun": true`. |
| Agent cannot delete anything | Intended: `--access-level=write` never exposes destructive tools. Use `full`. |
| Prompts missing | `build_report` needs `write`/`full`; no prompts at all under `--profiles`; a scoped-out referenced tool withholds the prompt. |
| Export file "outside the current directory" | `export_data.outputPath` must be within the server process cwd. |
| Looks frozen when run by hand | Normal. Ctrl+C to stop. |
