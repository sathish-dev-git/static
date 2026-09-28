# 14 · Settings Files, Environment Variables, Config Folder and Logs

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Source: `config/Settings`, `core/AppPaths`, `core/LoggingConfig`, `core/AuditLog`, `service/decorator/*`, `settings.example.json`, README.txt.

---

## 1. Folder layout (unified install)

```text
~/za-cli/                          (%USERPROFILE%\za-cli on Windows; --install-dir / ZA_CLI_HOME changes the root)
├── za-cli.jar, za-cli, za-cli.bat, setup.sh, setup.bat, README.txt, help.html, settings.example.json
├── bin/za-cli                     symlink / wrapper on PATH
├── logs/
│   └── za-cli.0.log … za-cli.9.log     rotating, 10 MB × 10, oldest replaced
└── config/                        <config-dir>
    ├── profiles.json              profile registry + active profile (no secrets)
    ├── profiles/<name>.enc        AES-256-GCM encrypted credentials, one per profile
    ├── profiles/<name>.settings.json   optional per-profile settings overrides
    ├── settings.json              optional global settings (you create it; za-cli only reads)
    ├── aliases.json               saved command lines
    └── telemetry.json             local usage counters (only when enabled)
```

Path resolution:

| Location | Resolution order |
|---|---|
| Home (`homeDir`) | `-Dza.cli.home` (set by the launcher) → `ZA_CLI_HOME` env → `~/za-cli` |
| Config (`configDir`) | `-Dza.cli.config` (independent override) → `<homeDir>/config` |
| Logs (`logsDir`) | `-Dza.cli.logs` → `<homeDir>/logs` → if not writable, `<java.io.tmpdir>/za-cli/logs` |
| Legacy config | `~/.config/za-cli` — migrated automatically into the unified folder on first run |

`za-cli doctor` and `za-cli telemetry` print the exact config folder. The folder is created on first login; create it by hand if you want a `settings.json` before logging in.

Every file za-cli creates:

| File | Contents | Secrets? |
|---|---|---|
| `profiles.json` | `{"version":"2.0","activeProfile":"…","profiles":{"<name>":{"name","orgId","dataCentre","createdAt"}}}` | no |
| `profiles/<name>.enc` | `{"version":"1.0","data":"<base64 salt‖iv‖ciphertext>"}` | encrypted |
| `settings.json` / `profiles/<name>.settings.json` | flat JSON of settings keys | may contain a proxy password |
| `aliases.json` | `{"<alias>":"<command line>"}` | no (unless you put one in a command) |
| `telemetry.json` | `{"command:…":n,"tool:…":n}` | no |
| `logs/za-cli.N.log` | text or JSON log lines incl. audit events | never passphrases/tokens/secrets |

---

## 2. Settings files

Two optional, **read-only** (never written by za-cli), flat JSON files:

| File | Applies to |
|---|---|
| `<config-dir>/settings.json` | every profile |
| `<config-dir>/profiles/<profile>.settings.json` | that profile only; wins over the global file |

Bootstrap from the template shipped in the bundle (every key at its default, so copying as-is changes nothing):

```bash
# macOS / Linux
cp ~/za-cli/settings.example.json ~/za-cli/config/settings.json
# Windows
copy "%USERPROFILE%\za-cli\settings.example.json" "%USERPROFILE%\za-cli\config\settings.json"
```

`settings.example.json`:

```json
{
  "_comment": "Every key below is set to za-cli's built-in default, so copying this file as-is changes nothing. …",

  "output.format": "table",
  "confirmation.policy": "destructive",

  "network.proxy": "",
  "network.proxy.host": "",
  "network.proxy.port": "",
  "network.proxy.username": "",
  "network.proxy.password": "",
  "network.connectTimeoutSeconds": 5,
  "network.readTimeoutSeconds": 0,

  "cache.ttlSeconds": 60,
  "cache.maxEntries": 2000,

  "telemetry.enabled": false
}
```

### 2.1 Keys

| Key | Type | Default | Equivalent flag | What it does |
|---|---|---|---|---|
| `output.format` | `table\|json\|csv\|tsv\|yaml\|ndjson` | `table` | `-o` | default output format |
| `confirmation.policy` | `destructive\|mutating\|none` | `destructive` | `--require-confirmation` (`--yes` overrides all) | when to prompt |
| `network.proxy` | `host:port` or `http://[user:pass@]host:port` | none | `--proxy` | combined proxy string; wins over the 4 split keys |
| `network.proxy.host` | string | none | — | split proxy (use when the password contains `@`, `:` or `/`) |
| `network.proxy.port` | number/string | none | — | |
| `network.proxy.username` | string | none | — | optional |
| `network.proxy.password` | string | none | — | optional; never logged or printed by `doctor` |
| `network.connectTimeoutSeconds` | int | `5` | `--connect-timeout` | |
| `network.readTimeoutSeconds` | int | `0` (none) | `--read-timeout` | |
| `cache.ttlSeconds` | int | `60` | — | how long workspace/view/column listings are reused within one process; `0` disables caching |
| `cache.maxEntries` | int | `2000` | — | LRU bound of the in-memory cache |
| `telemetry.enabled` | boolean | `false` | — (global file only) | local usage counting |

Unknown keys are ignored. A corrupt file logs `Settings file could not be read: <path>` and falls through to the next layer; an unparsable integer falls back to the default with a warning.

### 2.2 Precedence

```text
1. option typed on the command line        -o json
2. environment variable                     ZA_CLI_SETTING_OUTPUT_FORMAT=json
3. profile settings file                    config/profiles/live.settings.json
4. global settings file                     config/settings.json
5. built-in default                         table
```

Proxy specifically: `--proxy` → `ZA_CLI_SETTING_NETWORK_PROXY` → settings `network.proxy` → settings `network.proxy.*` → `HTTPS_PROXY`/`https_proxy` → none. Settings deliberately outrank the ambient env var; combined and split forms are never merged.

### 2.3 Environment-variable form of any setting

`ZA_CLI_SETTING_` + key upper-cased with `.` and `-` → `_` (inner capitals are just upper-cased, not split):

| Setting | Variable |
|---|---|
| `output.format` | `ZA_CLI_SETTING_OUTPUT_FORMAT` |
| `confirmation.policy` | `ZA_CLI_SETTING_CONFIRMATION_POLICY` |
| `network.proxy` | `ZA_CLI_SETTING_NETWORK_PROXY` |
| `network.connectTimeoutSeconds` | `ZA_CLI_SETTING_NETWORK_CONNECTTIMEOUTSECONDS` |
| `cache.ttlSeconds` | `ZA_CLI_SETTING_CACHE_TTLSECONDS` |
| `telemetry.enabled` | `ZA_CLI_SETTING_TELEMETRY_ENABLED` |

### 2.4 Examples

Default to JSON, prompt before any change, cache for two minutes:

```json
{ "output.format": "json", "confirmation.policy": "mutating", "cache.ttlSeconds": 120 }
```

Corporate proxy with an awkward password (split keys, no escaping needed):

```json
{
  "network.proxy.host": "proxy.corp.example",
  "network.proxy.port": "8080",
  "network.proxy.username": "svc-account",
  "network.proxy.password": "p@ss:w/rd"
}
```

Per-profile override — `config/profiles/acme-eu.settings.json`:

```json
{ "network.proxy": "eu-proxy.corp.example:3128", "output.format": "ndjson" }
```

Errors: `Invalid proxy '<v>': expected host:port or http://[user:pass@]host:port` · `Invalid proxy: host must not be blank (check network.proxy.host)` · `Invalid proxy port '<v>': expected a number (check network.proxy.port)`.

---

## 3. All environment variables

| Variable | Purpose |
|---|---|
| `ZA_CLI_PASSPHRASE` | passphrase or `env:NAME` / `keychain:account` reference |
| `ZA_CLI_SETTING_<KEY>` | override any settings key |
| `HTTPS_PROXY`, `https_proxy` | proxy fallback below settings |
| `NO_COLOR`, `ZA_CLI_NO_COLOR` | disable colour (`--no-color` equivalent) |
| `ZA_CLI_NO_ANIM` | disable the interactive intro/spinner |
| `ZA_CLI_ASCII` | force ASCII box drawing |
| `ZA_CLI_NO_EMOJI` / `ZA_CLI_EMOJI` | force emoji off / on in the TUI |
| `ZA_CLI_HOME`, `ZA_CLI_INSTALL_DIR` | install/home folder (setup scripts and `AppPaths`) |
| `TERM`, `COLORTERM`, `WT_SESSION`, `TERM_PROGRAM`, `ANSICON`, `ConEmuANSI` | terminal capability detection |
| `_JAVA_OPTIONS`, `JAVA_TOOL_OPTIONS` | standard JVM hooks (za-cli does not set them; note `Picked up _JAVA_OPTIONS` on stderr comes from the JVM) |

JVM system properties: `za.cli.home`, `za.cli.config`, `za.cli.logs`, `za.cli.log.format` (`json` for JSON lines), `za.cli.no.color`. Pass with `java -Dza.cli.config=/etc/za-cli -jar za-cli.jar …` or via `JAVA_TOOL_OPTIONS`.

---

## 4. Logs

- Location: `<homeDir>/logs/za-cli.N.log` (fallback `<tmp>/za-cli/logs`). Rotation **10 MB × 10 files**, append mode.
- Level: everything from the `com.zoho.analytics.cli` logger tree. Failures to initialise logging are swallowed — logging never breaks a command.
- Text format (default):

```text
2026-09-22T10:14:02.117Z INFO [com.zoho.analytics.cli.command.BaseCommand] [corr=3f9c1d2e-8a41-4b77-9c1e-0d5a6f7b8c9d] Executing tool rename_workspace
2026-09-22T10:14:02.530Z WARNING [com.zoho.analytics.cli.service.decorator.RetryInterceptor] [corr=3f9c…] Retrying getViews after HTTP 429 (attempt 1 of 3 retries), waiting 312ms
```

- JSON format (`-Dza.cli.log.format=json`): one object per line `{timestamp, level, logger, correlationId, message[, stackTrace]}`.
- **Never logged**: passphrases, client secrets, refresh tokens, proxy passwords, API keys. Tool failure logs redact any argument key containing `pass`, `secret`, `token` or `key` as `***`.

### 4.1 Correlation IDs

Every run has a process UUID; every catalog tool call opens a call-scoped UUID. It appears in `-vv` output, in `--envelope` `meta.correlationId`, in MCP `_meta.correlationId`, and in every log line — so a script result or an agent action can be tied to its log entry.

### 4.2 Audit log

Every **mutating or destructive** service call (never read-only `get*` calls) is recorded by the `com.zoho.analytics.cli.audit` logger into the same rotating file, one JSON object per event:

```text
2026-09-22T10:14:02.117Z INFO [com.zoho.analytics.cli.audit] [corr=3f9c…] {"event":"analytics.renameWorkspace","timestamp":"2026-09-22T10:14:02.117Z","correlationId":"3f9c…","principal":"sathish","profile":"production","orgId":20087654321,"operation":"renameWorkspace","safetyClass":"MUTATING","arguments":["2148712000000012345","Sales Analytics 2026"],"outcome":"success","durationMs":412}
```

Fields: `event`, `timestamp`, `correlationId`, `principal` (OS user), `profile`, `orgId`, `operation`, `safetyClass` (`MUTATING`/`DESTRUCTIVE`), `arguments` (positional, truncated at 200 chars), `outcome` (`success`/`failure`), `durationMs`, `error` (failure only). In `--profiles` MCP mode each org has its own audit chain.

```bash
grep '"safetyClass":"DESTRUCTIVE"' ~/za-cli/logs/za-cli.*.log | jq -R 'split("] ")[-1] | fromjson'
```

---

## 5. Cache, retry and resilience

| Concern | Behaviour |
|---|---|
| Cache | Any service method starting with `get` is cached for `cache.ttlSeconds` (default 60 s) up to `cache.maxEntries` (LRU, default 2000). Any non-`get` call clears the whole cache. `ttl 0` disables it. Scope: one process (a one-shot command effectively has no cache; MCP servers and interactive sessions benefit). Cached objects are copied on return. |
| Retry | Up to **4 attempts** (1 + 3 retries) on HTTP `429, 500, 502, 503, 504` with full-jitter exponential backoff (caps 400 ms, 800 ms, 1600 ms; max 6 s). Other errors fail immediately. `Retry-After` is not honoured (not exposed by the SDK). Logged at WARNING. |
| Rate limits | Reactive only (429 retries) plus adaptive pacing inside `search`. Lower call volume with `cache.ttlSeconds` and scoped searches. |

---

## 6. Housekeeping commands

```bash
za-cli doctor                              # prints resolved config dir, proxy, DC
za-cli telemetry                           # prints settings.json path
cat ~/za-cli/config/profiles.json          # safe to read: no secrets
tail -f ~/za-cli/logs/za-cli.0.log
~/za-cli/setup.sh uninstall --purge-logs   # remove logs only
~/za-cli/setup.sh uninstall --purge-config # remove profiles, credentials, settings, aliases (asks first)
```
