# za-cli 1.0.0 — Complete Reference Set

**za-cli** is the Zoho Analytics command-line tool built on the Zoho Analytics Java Client SDK (REST API v2). One Java 17 jar, three surfaces: direct commands for scripts and CI, an interactive terminal menu, and an MCP server for AI agents. This set documents **every feature of the shipped 1.0.0 bundle** (`ZDBStatic/cli/1.0.0/za-cli-1.0.0/`) with syntax, options, examples and sample outputs, and is the base for the future client API docs, release notes and marketing material.

| Sources used | |
|---|---|
| Shipped bundle | `README.txt`, `help.html` (handbook), `settings.example.json`, `setup.sh`, `setup.bat`, launchers, `za-cli.jar` `--help` for all 72 command entries |
| Live runs | outputs captured from the 1.0.0 jar without a login (help, version, doctor, telemetry, alias, completion, error paths) |
| Source tree | `ZA_CLI/src/main/java/com/zoho/analytics/cli/**` (tool catalog schemas, formatters, navigators, error codes, settings, MCP bridge) and its tests |
| Announcement | `References/za-cli-initial-post.txt` |

> Where a sample shows data from a Zoho organization it is **illustrative** (realistic IDs and field names, following the real formatters); outputs labelled *captured* are verbatim from the jar.

---

## Reading order

| # | File | What it covers |
|---|---|---|
| 01 | [Overview and architecture](01-overview-and-architecture.md) | What za-cli is, the three surfaces, tool catalog architecture, key numbers, surface parity matrix, security model, vocabulary |
| 02 | [Installation and setup](02-installation-and-setup.md) | Bundle contents, requirements, install/uninstall on macOS/Linux/Windows, PATH, custom locations, verification, shell completion, upgrade, other distribution channels, build from source |
| 03 | [Authentication and profiles](03-authentication-and-profiles.md) | Zoho credentials, `login` flow with every prompt, `logout`, all `profile` commands, passphrase handling, `env:`/`keychain:` secret references, 12 data centres, encrypted storage |
| 04 | [Global options and output](04-global-options-and-output.md) | Option placement, every common option, `-o table/json/csv/tsv/yaml/ndjson`, `--envelope`, `--fields`, `--filter`, pagination, confirmation policy, `--dry-run`, exit codes, versioning |
| 05 | [Direct commands — org](05-direct-commands-org.md) | `org info`, `users`, `users add/remove`, `admins`, `resources` |
| 06 | [Direct commands — workspace](06-direct-commands-workspace.md) | 17 workspace commands: list/get/create/rename/delete, users, share-info, folders, groups, trash, datasources |
| 07 | [Direct commands — view](07-direct-commands-view.md) | 23 view commands: discovery, URLs/publish, sharing, rename/delete, rows (preview/add/update/delete/import-rows), columns, tables, query tables, charts, summary and pivot reports |
| 08 | [Import and export](08-import-export.md) | `export` (7 formats, criteria, columns, limits) and `import` (append/updateadd/truncateadd, CSV/JSON options), outputs, errors, scheduling |
| 09 | [Search](09-search.md) | Org-wide vs workspace-scoped name search, scoring, result shape |
| 10 | [Helper commands](10-helper-commands.md) | `completion`, `doctor`, `telemetry`, `alias` |
| 11 | [Interactive mode](11-interactive-mode.md) | Startup, keys, filtering, every menu (org, workspace, table, chart, dashboard), guided row editing, sharing presets, `x`/`v` learning aids |
| 12 | [MCP server](12-mcp-server.md) | All `mcp-server` options, access levels, scoping, TOON/auto encoding, multi-org `--profiles`, dry-run, protocol details, prompts, client setup for Claude Desktop/Code, Cursor, ChatGPT |
| 13 | [MCP tool catalog](13-mcp-tool-catalog.md) | All 54 tools with arguments, types, required flags, result shapes, errors; the 2 prompts verbatim; encodings |
| 14 | [Settings, files and logs](14-settings-files-and-logs.md) | Folder layout, `settings.json` keys and precedence, environment variables, logs, correlation IDs, audit log, cache/retry |
| 15 | [Troubleshooting and error codes](15-troubleshooting-and-error-codes.md) | Exit codes, `ZA1001–ZA2001`, all mapped Zoho error codes, validation messages, symptom→fix table |
| 16 | [Recipes and use cases](16-recipes-and-use-cases.md) | CI, cron, bulk load, build-and-share, safe edits, audits, onboarding, multi-org, agents, proxies |
| 17 | [Command quick reference](17-command-quick-reference.md) | One-page cheat-sheet of every command, option, env var and exit code |
| 18 | [Limitations and roadmap](18-limitations-and-roadmap.md) | Unverified areas, disabled features (chat REPL, group sharing), design limits, roadmap |

---

## Feature map (where to find each capability)

| Feature | Direct CLI | Interactive | MCP tool(s) | Doc |
|---|---|---|---|---|
| Install / uninstall / PATH / completion | `setup.sh`, `completion` | — | — | 02, 10 |
| Login, profiles, data centres, passphrase references | `login`, `logout`, `profile …` | Switch Profile | `--profiles` | 03 |
| Output formats, envelope, fields/filter, pagination | global options | fixed layout | `--result-encoding` | 04 |
| Confirmation policy, dry run | `--yes`, `--require-confirmation`, `--dry-run` | built-in prompts | `--dry-run` | 04, 12 |
| Organization info, users, admins, resources | `org …` | Org menu | `get_subscription`, `list_org_users`, `list_admins`, `get_resource_usage`, `add/remove_org_users` | 05 |
| Workspaces, folders, groups, trash, datasources, workspace users, share summary | `workspace …` | Workspace menu | 15 WORKSPACE + METADATA tools | 06 |
| Views: details, columns, URLs, publish | `view get/columns/url/embed-url/publish/publish-config` | Table/Chart/Dashboard menus | `get_view_*`, `make_view_public` | 07 |
| Sharing: who has access, grant, revoke | `view share-info/share/remove-share` | Share Info / Share View / Remove Share | `get_view_share_details`, `share_view`, `remove_view_share` | 07 |
| Rows: preview, add, update, delete, inline import | `view preview/add-row/update-row/delete-rows/import-rows` | guided pickers with affected-row preview | `preview_data`, `add_row`, `update_rows`, `delete_rows`, `import_rows` | 07, 11 |
| Modelling: tables, query tables, charts, summary, pivot, add column | `view create-*`, `view add-column` | Create View submenu, Add Column | 5 MODELLING tools + `add_column` | 07 |
| File import/export | `import`, `export` | Import Data / Export Data | `import_data`, `export_data` | 08 |
| Search | `search` | Workspace → Search | `search_metadata` | 09 |
| Diagnostics, telemetry, aliases | `doctor`, `telemetry`, `alias` | — | 3 DIAGNOSTICS tools | 10 |
| AI agents | `mcp-server` | — | 54 tools, 2 prompts | 12, 13 |
| Settings, proxy, logs, audit | `settings.json`, env vars | — | — | 14 |

---

## Key facts at a glance

- **Version** `za-cli 1.0.0` · Java 17+ · single shaded jar · macOS / Linux / Windows.
- **72** command entries · **54** MCP tools (23 read-only, 25 mutating, 6 destructive) · **2** MCP prompts · **6** output formats · **12** data centres (only `us` verified) · **5** stable error codes · exit codes 0/1/2.
- **Security**: AES-256-GCM credentials, PBKDF2-HMAC-SHA256 × 310,000, passphrase never stored; `env:`/`keychain:` references; MCP defaults to read-only; nothing leaves the machine but calls to your Zoho DC.
- **Architecture**: one tool catalog drives CLI, TUI and MCP; every call carries a correlation ID, is audited, cached (60 s TTL) and retried on 429/5xx.

---

## Companion material in this folder

- `../za-cli-initial-post.txt` — the internal release announcement this set was seeded from.
- Bundle: `../../ZDBStatic/cli/1.0.0/za-cli-1.0.0/` (`help.html` is the end-user handbook; `README.txt` its terminal twin).
- Source: `../../ZA_CLI/` (developer `README.md`, `CLAUDE.md`, `SANITY_TEST_PLAN.md`, per-package `README.md`s).
