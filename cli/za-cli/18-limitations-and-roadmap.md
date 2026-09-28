# 18 · Known Limitations, Unverified Areas and Roadmap

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Compiled from README.txt, help.html, the developer CLAUDE.md/README.md, code comments in the 1.0.0 source tree, and the release announcement. The announcement states plainly that 1.0.0 is **not yet a GA product**: some modules are intentionally incomplete or disabled.

---

## 1. Unverified (implemented, follows documentation, not exercised on real infrastructure)

| Area | Status |
|---|---|
| Data centres other than `us` (`eu in au jp ca cn sa uae sg inec uk`) | URLs follow Zoho's regional convention; not tested against a live non-US tenant |
| `keychain:` on Linux (`secret-tool` / Secret Service) | implemented and reviewed, unverified on real hardware; use `env:` if in doubt |
| `keychain:` on Windows (Credential Manager via PowerShell helper) | implemented, unverified |
| winget manifests | structurally valid, unverified (no Windows machine available) |
| `updatedRows` response shape from update-row | reads both top-level and nested `result.updatedRows`; exact SDK shape unconfirmed |

## 2. Disabled or hidden in 1.0.0

| Feature | State |
|---|---|
| **LLM chat REPL** (`za-cli chat`, `za-cli config set-llm`) with Claude / OpenAI / Ollama providers | Fully compiled in source, **not registered** in the command tree; interactive **AI Chat** / **Configure AI** menu items commented out. `mcp-server` is the supported AI path. A `gemini` provider name is accepted by the config command but no provider class exists. |
| Group-based sharing in interactive **Share View / Remove Share** and in `view share` (`groupIds`) | Temporarily disabled pending a fix; email-based sharing only |
| Failure envelope on stdout | `Envelope.failure` exists but errors are always reported on stderr + exit code in 1.0.0 |
| `--readonly` / `--allow-writes` on `mcp-server` | Replaced by `--access-level` before release; not accepted |

## 3. Behavioural limitations (by design or accepted)

| Limitation | Notes |
|---|---|
| Pagination flags are display-only | Zoho's list endpoints have no server-side paging; the full list is always fetched |
| `--dry-run` covers catalog calls only | `workspace list/get/create/users*/datasources`, `view list/url/embed-url/publish/rename/delete/add-row/update-row/delete-rows`, `import`, `export`, `org *` call the SDK directly |
| `view delete`, `view delete-rows` do not prompt in direct mode | they bypass the catalog's confirmation policy; export first, or use the guided TUI which previews affected rows |
| No affected-row preview for `view update-row`/`delete-rows` in direct mode | preview with `export --criteria` |
| Interactive **Contains** operator does not escape `%`/`_` | Zoho's escape mechanism unconfirmed; a literal `%` acts as a wildcard |
| Interactive affected-row preview is a separate call | concurrent edits can make the count differ from what actually changes; exact only up to 1000 rows (`more than 1000 rows`) |
| Interactive criteria picker offers 7 operators only | `IN`, `NOT IN`, `BETWEEN` deliberately omitted |
| Interactive search is workspace-scoped only | org-wide search available via `za-cli search` / MCP |
| Interactive profile switch does not re-authenticate | restart za-cli |
| `--fields`/`--filter` are top-level, exact-key only | no dot paths; `--filter` ignores single-object results |
| MCP transport is STDIO only | no HTTP/SSE; ChatGPT "server URL" connectors need an external bridge |
| MCP prompts unavailable with `--profiles` | v1 scope: tools only |
| MCP tool calls are serialized | SDK token-refresh state is mutable |
| `export_data`/`import_data` (MCP) restricted to the server's cwd | security boundary |
| `Retry-After` header not honoured | SDK does not expose headers; jittered backoff instead |
| Columns can be added but not removed/re-typed; folders cannot be renamed; group members cannot be edited | not exposed by za-cli |
| `login` requires a console | no headless first-time provisioning |
| Releases unsigned | verify `SHA256SUMS.txt` |
| Homebrew / apt / yum / winget not yet published | packaging prepared, no release artifact to point at |
| `help.html` and the tool catalog page are hand-maintained | can drift from code between releases |
| Very little automated coverage of the TUI navigators | behaviour is exercised manually via `SANITY_TEST_PLAN.md` (which still lists the disabled chat items) |

## 4. Numbers that differ between documents

The developer README mentions "67 commands / 51 MCP tools" and the announcement "72 commands / 54 tools / 23 read-only". The shipped 1.0.0 jar exposes **54 tools** (23 READ_ONLY, 25 MUTATING, 6 DESTRUCTIVE) and the handbook counts **72** command entries (leaf commands plus group entries). The `--help` output in this reference set is authoritative for what the binary accepts.

## 5. Roadmap (from the announcement)

- **LLM-based Chat REPL** — native conversational terminal interface (code already present, to be enabled).
- **Automated authentication setup** — browser-based OAuth to replace manual Client ID / Secret / Refresh Token copy-paste.
- **Full API parity** — expand the operations catalog toward 100 % of public Zoho Analytics APIs (each new SDK operation implemented once appears in CLI, TUI and MCP simultaneously).
- **More pre-defined MCP prompts** — beyond `audit_workspace` and `build_report` (e.g. `build_chart`).
- **Native package managers** — Homebrew, apt-get and friends via published release binaries.
- **Advanced telemetry / observability** — latencies, invocation patterns and bottlenecks (still local/opt-in in spirit).
- Code signing for releases once a certificate / Apple Developer account is available.

## 6. Feedback

The announcement invites feedback on the bundle (`help.html`, README.txt, install ZIP) before rollout to end users. Include `za-cli doctor` output and the failing command with `-v` in any report.
