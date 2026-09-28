# 01 · Overview and Architecture

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).

---

## 1. What za-cli is

**za-cli** is a Java 17 command-line application for **Zoho Analytics**, built directly on top of the Zoho Analytics Java Client SDK (REST API v2). It brings Analytics administration to the terminal so developers, ops and admin teams can manage organizations, workspaces, tables, rows, views/reports and sharing without a browser.

It ships as **one jar** with **three interfaces** (called *surfaces* in this document set):

| Surface | How you start it | Who it is for |
|---|---|---|
| **Direct command mode** ("headless") | `za-cli <command> …` — one command, then exit | Scripts, CI/CD, cron, ad-hoc admin from a shell |
| **Interactive mode** (TUI) | `za-cli` with no sub-command | Humans exploring or administering by menu; learning the typed commands (`x` key) |
| **MCP server mode** | `za-cli mcp-server …` started by an MCP client | AI agents (Claude Desktop, Claude Code, Cursor, ChatGPT desktop, any MCP-compatible client) |

A fourth, hidden surface — an **LLM chat REPL** (`chat` command with Claude/OpenAI providers) — exists in source but is **not registered in the 1.0.0 jar**; it is on the roadmap (see [18 · Limitations and roadmap](18-limitations-and-roadmap.md)).

### Benefits (from the release announcement)

- **Scalable administration** — automate repetitive setup, bulk row imports and sharing-permission audits across multiple organizations.
- **Pipeline integration** — embed analytics operations into CI/CD, cron or shell scripts with no browser session.
- **Deterministic AI interactions** — agents get a secure, structured, JSON-Schema-described tool surface instead of UI scraping.

> The announcement notes this is **not yet a GA product**: a few modules are intentionally incomplete or disabled, but the architecture, security model and engineering discipline are in place.

---

## 2. Architecture in one picture

```text
                ┌─────────────────────────────────────────────────────────────┐
                │                       za-cli.jar                            │
                │                                                             │
   terminal ───▶│  picocli command tree      ┐                                │
   (direct)     │  (login/profile/org/       │                                │
                │   workspace/view/import/   │        ┌────────────────────┐  │
                │   export/search/alias/…)   ├──────▶ │   TOOL CATALOG     │  │
                │                            │        │  54 ToolDefinitions│  │
   keyboard ───▶│  Interactive TUI           │        │  name · category · │  │
   (menus)      │  (Org → Workspace →        ├──────▶ │  safety · JSON     │  │
                │   Table/Chart/Dashboard    │        │  schema · handler  │  │
                │   navigators)              │        │  · cliCommand      │  │
                │                            │        └─────────┬──────────┘  │
   MCP client ─▶│  MCP server (STDIO,        │                  │             │
   (JSON-RPC)   │  JSON-RPC, tools+prompts)  ┘                  ▼             │
                │                                ┌──────────────────────────┐ │
                │   cross-cutting: ToolContext   │ AnalyticsService         │ │
                │   (dry-run, confirmation,      │  ├ CacheInterceptor (TTL)│ │
                │   correlation IDs, audit log,  │  ├ RetryInterceptor      │ │
                │   result-size cap, telemetry)  │  ├ AuditInterceptor      │ │
                │                                │  └ ZohoAnalyticsService  │ │
                │                                │      (SDK v2 client)     │ │
                └────────────────────────────────┴──────────────┬───────────┘ │
                                                                │  HTTPS       │
                                                     analyticsapi.zoho.<dc>   │
                                                     accounts.zoho.<dc> (OAuth)
```

**Single source of truth.** Every capability is a `ToolDefinition` in a shared **tool catalog** (`com.zoho.analytics.cli.tools`). Each definition carries: a stable tool name (`list_workspaces`, `delete_rows`, …), a **category** (METADATA, ROW, DATA, VIEW, MODELLING, WORKSPACE, ADMIN, DIAGNOSTICS), a **safety level** (READ_ONLY, MUTATING, DESTRUCTIVE), a JSON input schema, a handler, an idempotency flag, and the equivalent **direct CLI command string**. The direct commands, the interactive menus and the MCP server all dispatch through this catalog, so a new SDK operation implemented once appears in all three surfaces with no extra routing or wrapper code.

**Cross-cutting services** applied to every catalog call regardless of surface:

| Concern | Behaviour |
|---|---|
| Correlation ID | A UUID generated per command/tool call; printed with `-vv`, embedded in `--envelope` meta, returned in MCP `_meta.correlationId`, written to the audit log. |
| Audit log | Every catalog call (tool name, safety, profile/org, outcome, correlation ID) is appended to the rotating log. Secrets are never logged. |
| Dry run | `--dry-run` short-circuits mutating/destructive calls with a preview result. |
| Confirmation policy | `destructive` (default) / `mutating` / `none` — prompts only when a real console is attached. |
| In-memory cache | Workspace/view/column listings reused for `cache.ttlSeconds` (default 60 s, max 2000 entries) within one process. |
| Retry | Transient failures are retried with backoff by `RetryInterceptor`. |
| Result cap | MCP tool results capped at 1,000,000 characters, or `--max-result-tokens × 4` chars. |
| Telemetry (opt-in) | Local counters of command/tool names only; never transmitted. |

**Package map (source, for orientation)**

| Package | Role |
|---|---|
| `cli.ZaCli` | Root picocli command; global options; exit-code contract |
| `cli.command.*` | Direct commands (`Login`, `Logout`, `Profile`, `Org`, `Workspace`, `View`, `Import`, `Export`, `Search`, `Completion`, `Doctor`, `Telemetry`, `Alias`) and `BaseCommand` (shared option handling, output pipeline) |
| `cli.tools.*` | Tool catalog, categories, safety, schemas, `CliEquivalents`, handler definitions per category (`defs/*Tools.java`) |
| `cli.mcp.*` | MCP server command, tool/prompt bridges, access levels, tool scope, JSON/TOON/auto result encoders |
| `cli.ui.*` | Interactive shell, rich renderer, navigators (Org, Workspace, Table, Chart, Dashboard), fuzzy filter, theme |
| `cli.service.*` | `AnalyticsService` interface, SDK-backed implementation, decorators (cache, retry, audit) |
| `cli.config.*` | Profiles, encrypted credential store, settings, aliases, telemetry, secret resolver + keychain backends |
| `cli.core.*` | Error codes, output format enums, data centres, proxy config, pagination, filters, table formatters |
| `cli.output.*` | table / json / envelope / csv / tsv / yaml / ndjson formatters |
| `cli.chat.*` | LLM chat REPL (not registered in 1.0.0) |

---

## 3. Numbers that matter

| Metric | Value (1.0.0) |
|---|---|
| Direct CLI commands (leaf + group entries, as counted by the handbook) | **72** |
| MCP tools in the catalog | **54** |
| … of which READ_ONLY (the default MCP exposure) | **23** |
| … MUTATING | **25** |
| … DESTRUCTIVE | **6** (`delete_workspace`, `delete_folder`, `delete_group`, `delete_trash_view`, `delete_rows`, `delete_view`) |
| MCP prompts | **2** (`audit_workspace`, `build_report`) |
| MCP tool categories | 8 (METADATA 13, ROW 4, DATA 3, VIEW 4, MODELLING 5, WORKSPACE 15, ADMIN 7, DIAGNOSTICS 3) |
| Output formats | 6 (`table`, `json`, `csv`, `tsv`, `yaml`, `ndjson`) + `--envelope` |
| Data centres | 12 (`us` verified; others follow Zoho's URL convention, unverified) |
| Distinct options across all commands | 67 |
| Stable error codes | 5 (`ZA1001`–`ZA1004`, `ZA2001`) |
| Exit codes | 0 success · 1 runtime failure · 2 usage error |

---

## 4. Surface parity — what works where

Every capability reads from the same catalog, but the surfaces deliberately differ in a few places:

| Capability | Direct CLI | Interactive TUI | MCP |
|---|---|---|---|
| Browse workspaces / views / columns, preview rows | ✅ | ✅ | ✅ |
| Export to file / import a file | ✅ | ✅ | ✅ |
| Row edits (add / update / delete rows, inline import) | ✅ typed `--criteria` | ✅ **guided** column/operator/value picker + affected-row count preview | ✅ |
| Create tables, query tables, charts, summary & pivot reports | ✅ | ✅ | ✅ |
| Sharing: see who has access (`share-info`) | ✅ | ✅ tables, charts, dashboards (read-only *Share Info*) | ✅ |
| Sharing: grant / revoke | ✅ | ✅ **table menu only**, via permission presets | ✅ |
| Search | ✅ org-wide **or** `--workspace` scoped | ✅ **workspace-scoped only** (by design, rate-limit safety) | ✅ both |
| Output shaping (`-o`, `--fields`, `--filter`) | ✅ | ❌ fixed layout | ❌ full (or size-capped) result |
| Pagination flags on 9 list commands | ✅ | n/a (paged screens) | ✅ (`offset`/`limit` args on list tools) |
| Dry run | ✅ | ❌ (own confirmation prompts) | ✅ (whole session) |
| Confirmation policy | ✅ | separate built-in prompts | n/a (client approves calls) |
| Guided multi-step prompts (`audit_workspace`, `build_report`) | ❌ | ❌ | ✅ |
| Multi-org in one process (`--profiles`) | per-command `-p` | *Switch Profile* menu | ✅ |
| Shell completion, doctor, telemetry, aliases | ✅ | ❌ | telemetry only (3 diagnostics tools) |
| Equivalent-command reflection | — | `x` key shows the direct command | `_meta.cliCommand` on every result |

---

## 5. Security model (summary)

- **Credentials at rest**: Client ID, Client Secret, Refresh Token, Org ID and data centre are stored per profile in `config/profiles/<name>.enc`, encrypted with **AES-256-GCM**; the key is derived from your passphrase via **PBKDF2-HMAC-SHA256, 310,000 rounds**, fresh random salt. GCM detects tampering.
- **Passphrase** is never stored, never logged, never transmitted. Minimum 8 characters. Lost passphrase ⇒ `za-cli login` again.
- **Secret references**: anywhere a passphrase is accepted you may pass `env:<NAME>` or `keychain:<account>` instead of a literal.
- **Never logged**: passphrases, client secrets, refresh tokens, proxy passwords (enforced by tests).
- **Nothing leaves your machine** except calls to your own Zoho Analytics data centre. No za-cli server, no update check, no telemetry upload.
- **Least privilege for agents**: MCP defaults to `--access-level=read`; `write` can never expose a destructive tool; `--tools/--categories/--exclude-tools` narrow further; unknown names fail fast at startup.
- **File permissions**: credential files, profile registry and aliases are written user-only and atomically.

Details: [03 · Authentication](03-authentication-and-profiles.md), [12 · MCP server](12-mcp-server.md), [14 · Files and logs](14-settings-files-and-logs.md).

---

## 6. Vocabulary used throughout this set

| Term | Meaning |
|---|---|
| **Profile** | One saved connection to one Zoho Analytics organization (credentials + org ID + data centre), named `1-30` chars `[A-Za-z0-9][A-Za-z0-9_-]*`. |
| **Active profile** | The profile used when `-p` is not given. Tracked in `config/profiles.json`. |
| **Workspace** | Zoho Analytics container of tables and reports. Identified by a long numeric `workspaceId`. |
| **View** | Anything inside a workspace: Table, Query Table, Chart, Report (summary/pivot/tabular), Dashboard. Identified by `viewId`. |
| **Tool** | A catalog entry callable from all surfaces. MCP clients see tools as functions. |
| **Safety level** | `READ_ONLY` / `MUTATING` / `DESTRUCTIVE`, drives confirmation prompts and MCP access levels. |
| **Envelope** | The versioned `{envelopeVersion, data, meta, error}` JSON wrapper for scripts (`-o json --envelope`). |
| **Correlation ID** | Per-call UUID tying output, MCP `_meta` and log lines together. |
| **TOON** | Compact table-like text encoding for large uniform MCP list results (`--result-encoding toon|auto`). |
| **Data centre (dc)** | Zoho region code (`us`, `eu`, `in`, `au`, `jp`, `ca`, `cn`, `sa`, `uae`, `sg`, `inec`, `uk`) fixed per profile. |
| **Criteria** | Zoho Analytics SQL-like row filter, e.g. `"Region" = 'East'` — column in double quotes, value in single quotes. |

---

## 7. Where to go next

- Install → [02](02-installation-and-setup.md) · Log in → [03](03-authentication-and-profiles.md) · Understand options and output → [04](04-global-options-and-output.md)
- Commands: org [05](05-direct-commands-org.md) · workspace [06](06-direct-commands-workspace.md) · view [07](07-direct-commands-view.md) · import/export [08](08-import-export.md) · search [09](09-search.md) · helpers [10](10-helper-commands.md)
- Interactive mode → [11](11-interactive-mode.md) · MCP → [12](12-mcp-server.md) + tool catalog [13](13-mcp-tool-catalog.md)
- Settings/files/logs → [14](14-settings-files-and-logs.md) · Troubleshooting → [15](15-troubleshooting-and-error-codes.md) · Recipes → [16](16-recipes-and-use-cases.md) · Cheat-sheet → [17](17-command-quick-reference.md) · Limits & roadmap → [18](18-limitations-and-roadmap.md)
