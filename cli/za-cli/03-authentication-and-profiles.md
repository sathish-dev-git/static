# 03 · Authentication, Profiles and Data Centres

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Source: `command/BaseCommand`, `command/LoginCommand`, `command/LogoutCommand`, `command/ProfileCommand`, `config/ProfileManager`, `config/ConfigManager`, `config/CredentialEncryptor`, `config/SecretResolver`, `core/DataCentre`, plus README.txt / help.html.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Profile** | One saved connection to one Zoho Analytics organization: Client ID, Client Secret, Refresh Token, Organization ID (ZSOID) and data centre, encrypted with a passphrase you choose. |
| **Active profile** | The one used when `-p` is not given. Recorded in `config/profiles.json` (`activeProfile`). The first profile you create becomes active automatically. |
| **Passphrase** | Min 8 characters, chosen at login. Encrypts the credential file locally. Never stored, logged or transmitted. Lose it → run `za-cli login` again. |
| **Data centre (dc)** | Zoho region the profile talks to. Fixed at creation; to change it, create another profile. |

Profile names: **1–30 chars**, letters (either case), digits, hyphens, underscores, starting with a letter or digit (regex `^[A-Za-z0-9][A-Za-z0-9_-]{0,29}$`). The name is stored exactly as typed. Valid: `default`, `prod-2026`, `1abc`, `acme_eu`, `Prod`, `IDC_Sathish`. Invalid: `-lead`, `_lead`, `has space`, `../escape`, 31+ chars — rejected with `Name not allowed.` followed by the rule.

> Two profiles cannot differ only by capitalization. A profile name is also a file name (`config/profiles/<name>.enc`), and on a case-insensitive file system (macOS, Windows) `Prod` and `prod` would share one credential file. `za-cli login` reuses the profile you already have (`Profile 'prod' already exists - using it.`); `za-cli profile add` refuses with `Profile names cannot differ only by capitalization.`

---

## 2. Before you start: Zoho credentials

1. **Zoho API Console** (`https://api-console.zoho.com`, or your DC's console): create a **Self Client** (or a server-based application).
2. Note the **Client ID** and **Client Secret**.
3. Generate a **Refresh Token** for the scopes `ZohoAnalytics.data.all`, `ZohoAnalytics.metadata.read`, `ZohoAnalytics.modeling.all` (generate a grant code for those scopes, exchange it for a refresh token per Zoho's OAuth flow).
4. Find your **Organization ID** in Zoho Analytics → Settings → Organization (a long number, sometimes labelled ZSOID).
5. Know your **data centre** (where your Zoho account lives).

> Copy the refresh token carefully — a stray space at either end is the most common cause of `Could not authenticate`. Non-US accounts **must** pass `--dc`.

---

## 3. `za-cli login`

```text
Usage: za-cli login [-hVy] [--dry-run] [--envelope] [--connect-timeout=<n>] [--dc=<dc>]
                    [--fields=<f>] [--filter=<f>] [-o=<fmt>] [-p=<profileName>] [-P=<passphrase>]
                    [--proxy=<proxy>] [--read-timeout=<n>] [--require-confirmation=<policy>]
Authenticate and store credentials securely
      --dc, --data-centre, --data-center=<dc>
                   Data centre for a new profile: us, eu, in, au, jp, ca, cn,
                     sa, uae, sg, inec, or uk (default: us, or an interactive
                     prompt if omitted). Unverified beyond us -- see 'za-cli
                     login --help'.
  -p, --profile=<profileName>
                   Profile to login to (default: active profile)
```

`login` is one of only two commands (with `logout`) that accept `-p` **after** the command name. It also honours `--proxy`, `--connect-timeout`, `--read-timeout`, so a proxied credential check behaves like every later command.

### 3.1 Interactive flow — new profile

```text
$ za-cli login
No profiles found.
Profile name: production

Profile: production
Enter credentials from https://api-console.zoho.com

Client ID: 1000.ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
Client Secret: ********
Refresh Token: ********
  Organization ID (ZSOID): 20087654321

  Data centre (unverified beyond US -- see docs):
     1) United States (us) (default)
     2) Europe (eu)
     3) India (in)
     4) Australia (au)
     5) Japan (jp)
     6) Canada (ca)
     7) China (cn)
     8) Saudi Arabia (sa)
     9) United Arab Emirates (uae)
    10) Singapore (sg)
    11) India Enterprise (inec)
    12) United Kingdom (uk)
  Select [1-12, default 1]: 1

Set a passphrase to encrypt credentials (min 8 chars):
  Passphrase (min 8 chars): ********
  Confirm Passphrase: ********
✓ Profile 'production' saved.
```

Order of events:

1. **Profile selection.** No profiles → `No profiles found - creating your first profile.` / `Choose a name to identify it, for example 'default'.` then `Profile name:` (blank re-prompts; a name outside the rule → `Name not allowed.` followed by the rule). Profiles exist → numbered menu with the active one marked `*` and a final `N) + New profile`, then `Select [1-N]:`. `--profile <name>` skips the menu.
2. **Credential prompts** (Client ID plain, Client Secret and Refresh Token masked, Organization ID plain). Blank → `Required.`; non-numeric org → `⚠  Organization ID must be numeric. Example: 6158XXXXX`.
3. **Data centre** menu (skipped when `--dc` is given). Blank → `us`; bad input → `⚠  Enter a number 1-12.`
4. **Verification against Zoho before anything is saved** — a real `getOrgs()` call through your proxy/timeouts. Failure:
   ```text
   Could not authenticate with these credentials: <Zoho message>
   Check the Client ID, Client Secret, Refresh Token, and Organization ID.
   ```
   → exit 1, nothing written.
5. **Passphrase** twice. `⚠  Passphrase is required.` · `⚠  Too short. Must be at least 8 characters.` · `⚠  Passphrases do not match.` The passphrase is **not trimmed**.
6. **Save** (up to 3 attempts): registry entry + encrypted file, profile becomes active → `✓ Profile '<name>' saved.` Failure → `Save failed: <msg>` … `Could not save after 3 attempts.`

### 3.2 Interactive flow — existing profile (re-authenticate / re-activate)

```text
$ za-cli login --profile production
Passphrase [production]: ********
✓ Authenticated as production
```

Up to 3 attempts: `Incorrect passphrase (1/3)` … `Authentication failed.` (exit 1). Empty → `Aborted.` The profile becomes active only after a successful unlock.

### 3.3 Non-interactive constraints

`login` **needs a console** for the prompts: piping stdin or running in CI prints `No console available. Run in a terminal.` (exit 1). Automate everything *after* login with `ZA_CLI_PASSPHRASE`. If you must provision headlessly, run `login` once on a workstation and ship the `config/` folder (it is encrypted) with the passphrase delivered separately, or seed `keychain:` entries.

### 3.4 Legacy migration

If a pre-profile `config.json` exists it is migrated to a profile named `default` (`✓ Migrated existing config to profile 'default'`); old scattered locations (`~/.config/za-cli`, …) are relocated into the unified folder on first run.

---

## 4. `za-cli logout`

```text
$ za-cli logout
✓ Logged out [production]
  Credentials are retained. Reactivate with 'za-cli profile switch production' or 'za-cli login'.
```

- Clears the active-profile marker and the in-memory client cache. **The encrypted credential file stays**, which is why signing back in only asks for the passphrase.
- `za-cli logout --profile <name>` targets a specific profile.
- No active profile → `No active profile.` (exit 0, captured).
- To delete credentials use `za-cli profile remove <name>`.

---

## 5. `za-cli profile …`

```text
Usage: za-cli profile [-hV] [COMMAND]
Manage login profiles (one per organization)
Commands:
  list    List all profiles
  add     Create a new profile
  switch  Switch active profile
  remove  Remove a profile
  info    Show profile details
  rename  Rename a profile
```

None of the `profile` commands accept the common options (`-o`, `-P`, `--dry-run`, …) — they are local, need no login, and work offline.

### 5.1 `profile list`

```text
$ za-cli profile list

  📋 Profiles
  ──────────────────────────────────────────────────
   ●  production
      Org: 20087654321
      staging
      Org: 20087654399  •  DC: EU

  2 profile(s)  •  Active: production
```

`●` marks the active profile; `DC:` is shown only for non-US profiles. Empty state (captured):

```text
  No profiles configured. Run 'za-cli profile add <name>' to create one.
```

### 5.2 `profile add <name> [--dc <dc>]`

Same credential prompts and server verification as login (prompts read `  Client ID: `, `  Client Secret: `, `  Refresh Token: `, `  Organization ID: `; errors `Required field.`, `Must be numeric (e.g. 6158XXXXX).`, `Minimum 8 characters required.`, `Passphrases do not match. Try again.`).

```text
$ za-cli profile add staging --dc eu

  🔐 New Profile: staging
  ──────────────────────────────────────────────────
  Client ID: …
  …
  ✓  Profile 'staging' created!
     Switch with: za-cli profile switch staging
```

**Adding does not activate** the new profile. Without a console: `No console available.` (exit 1).

### 5.3 `profile switch <name>`

```text
$ za-cli profile switch staging
✓ Active: staging
```

Unknown → `✗  Profile 'nope' does not exist.` (exit 1). A profile whose credential file is missing cannot be activated: `Profile 'x' has no stored credentials. Run 'za-cli login --profile x' first.`

For a single command, prefer `za-cli -p staging workspace list` (leaves the active profile alone).

### 5.4 `profile info [<name>]`

```text
$ za-cli profile info

  👤 Profile: production
  ────────────────────────────────────────
  Name:        production
  Org ID:      20087654321
  Data centre: United States (us)
  Active:      Yes
  Credentials: Stored
  Created:     04 Mar 2026 11:22:31 IST
```

Secrets are never displayed. Errors (captured): `✗  Profile 'x' not found.` (exit 1); no active and no name → `No active profile. Specify a name.` (exit 1).

### 5.5 `profile remove <name>`

```text
$ za-cli profile remove staging
Remove profile 'staging'? (y/N): y
✓ Removed 'staging'.
```

Deletes `profiles/<name>.enc` and the registry entry — **irreversible**. If it was the active profile you are asked to pick another (`Select [1-N] (blank to skip):`) or told `No profiles remain. Run 'za-cli login' to add one.` Without a console the confirmation is skipped. Unknown → `Profile 'nope' does not exist.` (exit 1).

### 5.6 `profile rename <old> <new>`

```text
$ za-cli profile rename test staging
  ✓  Renamed 'test' → 'staging'
```

Moves the credential file, then updates the registry (rolled back if the registry write fails). Aliases that mention the old name must be edited by hand.

---

## 6. Supplying the passphrase without a prompt

Every command that needs credentials unlocks them with the passphrase. `command/BaseCommand.getPassphrase()` checks **four** sources in order and returns the first non-blank one:

| # | Source | Notes |
|---|---|---|
| 1 | `-P` / `--passphrase <value>` on the **subcommand** (`za-cli workspace list -P <value>`) | Highest. Overrides a root-level `-P`. |
| 2 | `-P` / `--passphrase <value>` at the **root** (`za-cli -P <value> workspace list`) | Both forms are visible in process lists — least preferred. |
| 3 | `ZA_CLI_PASSPHRASE` environment variable | The recommended non-interactive route. |
| 4 | Interactive prompt `Passphrase:` | Only when a console is attached. No console and no value → `No console available for passphrase input.` + `Set ZA_CLI_PASSPHRASE env variable for non-interactive use.`, exit 1. |

Tiers 1–3 all pass through `SecretResolver.resolve()`, so any of them may carry an `env:` / `keychain:` reference instead of a literal (below).

### 6.1 Only one environment variable is ever read

za-cli performs exactly **one** `System.getenv("ZA_CLI_PASSPHRASE")`. It does **not** read `~/.bashrc`, `~/.profile`, `~/.bash_profile` or any `env.sh` — it has no knowledge of shell configuration files at all.

This matters when the passphrase is exported from a sourced file such as `~/.config/za-cli/env.sh`:

- That file is **not a second source** that competes with an "already global" value. Sourcing it simply *sets* the one variable, before za-cli starts.
- Precedence between shell sources is therefore settled **by the shell**, not by za-cli: the last `export` to run wins, and a `VAR=value command` prefix beats whatever the rc files exported.
- za-cli sees only the surviving value. There is no merging, no fallback to a "previous" value, and no second lookup.

```bash
# env.sh exported ZA_CLI_PASSPHRASE=correct
za-cli workspace list                              # uses 'correct'         -> works
ZA_CLI_PASSPHRASE=wrong za-cli workspace list      # prefix REPLACES it     -> [ZA1003]
ZA_CLI_PASSPHRASE=wrong za-cli -P correct workspace list   # tier 2 beats tier 3 -> works
za-cli -P wrong workspace list -P correct          # tier 1 beats tier 2    -> works
```

The single exception is the `env:` form: there the variable's *value* names a second variable to read (`ZA_CLI_PASSPHRASE='env:MY_PASS'` → za-cli reads `MY_PASS`). That is indirection, not precedence.

### Secret references instead of a literal passphrase

Anywhere a passphrase is accepted you may pass a **reference**; za-cli resolves it at startup:

| Form | Behaviour |
|---|---|
| `env:<NAME>` | Read environment variable `NAME`. Unset/empty → `Environment variable 'NAME' referenced by 'env:NAME' is not set or is empty.` (never falls back to a prompt). |
| `keychain:<account>` | Read credential `<account>` under service name `za-cli` from the OS store: macOS Keychain (`security`, **verified**), Linux Secret Service (`secret-tool`, implemented, unverified), Windows Credential Manager (implemented, unverified). za-cli only ever **reads**. |
| anything else | Used literally as the passphrase. |

Malformed references: `env: reference is missing a variable name, e.g. env:MY_SECRET_VAR` · `keychain: reference is missing an account name, e.g. keychain:default` · `Failed to read keychain:<account> — <msg>`. `vault:`, `aws-sm:`, `azure-kv:`, `gcp-sm:` are deliberately **not** recognised.

A reference that cannot be resolved fails under **`ZA1004`**, not `ZA1003` — resolution happens before any decryption is attempted, so the passphrase was never wrong, it was never obtained:

```text
  ✗  [ZA1004] Operation failed
     Environment variable 'MY_PASS' referenced by 'env:MY_PASS' is not set or is empty.
     Run with -v for the full stack trace.
(exit 1)
```

Seeding the OS store (once):

```bash
# macOS
security add-generic-password -s za-cli -a default -w 'your-passphrase'
security find-generic-password -s za-cli -a default -w        # verify
security delete-generic-password -s za-cli -a default          # before re-adding a new value

# Linux (libsecret-tools + running keyring daemon; headless servers: use env: instead)
secret-tool store --label="za-cli default" service za-cli account default
secret-tool lookup service za-cli account default

# Windows (target must be za-cli:<account>)
cmdkey /generic:za-cli:default /user:za-cli /pass:your-passphrase
cmdkey /list:za-cli:default
```

Missing entry → za-cli prints the exact seed command for your platform, e.g.

```text
No macOS Keychain entry found for service 'za-cli', account 'default'.
Seed it first: security add-generic-password -s za-cli -a default -w '<secret-value>'
```

Usage examples:

```bash
ZA_CLI_PASSPHRASE='your-passphrase'        za-cli workspace list
ZA_CLI_PASSPHRASE='env:MY_ORG_ZA_PASS'     za-cli workspace list
ZA_CLI_PASSPHRASE='keychain:default'       za-cli mcp-server
za-cli -P 'keychain:acme-prod' -p acme-prod org info
```

Wrong passphrase (any surface):

```text
  ✗  [ZA1003] Invalid passphrase
     Unable to decrypt credentials. Check your passphrase and try again.
(exit 1)
```

---

## 7. Data centres

| Code | Region | Accounts (OAuth) host | Analytics API host | Verified |
|---|---|---|---|---|
| `us` | United States | `accounts.zoho.com` | `analyticsapi.zoho.com` | ✅ end-to-end |
| `eu` | Europe | `accounts.zoho.eu` | `analyticsapi.zoho.eu` | ⚠ URL convention only |
| `in` | India | `accounts.zoho.in` | `analyticsapi.zoho.in` | ⚠ |
| `au` | Australia | `accounts.zoho.com.au` | `analyticsapi.zoho.com.au` | ⚠ |
| `jp` | Japan | `accounts.zoho.jp` | `analyticsapi.zoho.jp` | ⚠ |
| `ca` | Canada | `accounts.zohocloud.ca` | `analyticsapi.zohocloud.ca` | ⚠ |
| `cn` | China | `accounts.zoho.com.cn` | `analyticsapi.zoho.com.cn` | ⚠ |
| `sa` | Saudi Arabia | `accounts.zoho.sa` | `analyticsapi.zoho.sa` | ⚠ |
| `uae` | United Arab Emirates | `accounts.zoho.ae` | `analyticsapi.zoho.ae` | ⚠ |
| `sg` | Singapore | `accounts.zoho.sg` | `analyticsapi.zoho.sg` | ⚠ |
| `inec` | India Enterprise | `accounts.zohohq.in` | `analyticsapi.zohohq.in` | ⚠ |
| `uk` | United Kingdom | `accounts.zoho.uk` | `analyticsapi.zoho.uk` | ⚠ |

- `--dc` is case-insensitive; an unrecognised/blank code falls back to `us`.
- `za-cli doctor` shows the active profile's DC and appends `(unverified beyond US — see core.DataCentre)` for non-US.
- Wrong DC at login manifests as an authentication failure (za-cli checks the wrong regional server).

---

## 8. Storage and security

```text
~/za-cli/config/                          (%USERPROFILE%\za-cli\config on Windows; override with -Dza.cli.config)
├── profiles.json                         registry — no secrets
├── profiles/
│   ├── production.enc                    AES-256-GCM encrypted credentials
│   ├── staging.enc
│   └── production.settings.json          optional per-profile settings
├── settings.json                         optional global settings
├── aliases.json
└── telemetry.json                        only when telemetry is enabled
```

`profiles.json`:

```json
{
  "version": "2.0",
  "activeProfile": "production",
  "profiles": {
    "production": { "name": "production", "orgId": "20087654321", "dataCentre": "us", "createdAt": "2026-03-04T05:52:31.442Z" },
    "staging":    { "name": "staging",    "orgId": "20087654399", "dataCentre": "eu", "createdAt": "2026-05-11T08:10:02.007Z" }
  }
}
```

`profiles/<name>.enc`:

```json
{ "version": "1.0", "data": "<base64: 16-byte salt ‖ 12-byte IV ‖ AES-GCM ciphertext>" }
```

Decrypted payload: `{clientId, clientSecret, refreshToken, orgId, dataCentre}`.

| Control | Detail |
|---|---|
| Cipher | `AES/GCM/NoPadding`, 256-bit key, 128-bit tag (tamper-evident) |
| Key derivation | `PBKDF2WithHmacSHA256`, **310,000** iterations, random 16-byte salt per write |
| Passphrase | ≥ 8 chars, never persisted; wrong passphrase → `AEADBadTagException` → `[ZA1003]` |
| File permissions | directories `rwx------`, files `rw-------` (best effort on non-POSIX), atomic temp-file + move writes |
| Logging | passphrases, client secrets, refresh tokens, proxy passwords are never logged (tested) |
| Corrupt registry | `The profiles registry at <path> is corrupt. Fix or remove the file, then run 'za-cli login'.` |

Uninstall keeps `config/` unless `--purge-config` is passed ([02 § 11](02-installation-and-setup.md#11-uninstall)).

---

## 9. Multi-organization patterns

```bash
za-cli login --profile acme-prod                 # first org (becomes active)
za-cli profile add acme-staging --dc us          # second org (not active)
za-cli profile add acme-eu --dc eu               # third org, EU DC

za-cli profile list
za-cli -p acme-staging workspace list            # one-off
za-cli profile switch acme-eu                    # change default
za-cli mcp-server --profiles=acme-prod,acme-staging --access-level=read   # both orgs from one MCP server (same passphrase)
```

Per-profile settings (e.g. a different proxy or default output) go in `config/profiles/<name>.settings.json` — see [14 · Settings](14-settings-files-and-logs.md).

---

## 10. Troubleshooting login

| Symptom | Fix |
|---|---|
| `Could not authenticate with these credentials` | Re-copy the refresh token (no spaces); confirm Client ID/Secret pair; confirm Org ID; pass `--dc` for non-US. |
| `[ZA1001] Not logged in` | No active profile: `za-cli login` or `za-cli profile switch <name>`. |
| `[ZA1002] Profile 'x' does not exist.` | `za-cli profile list` for real names. |
| `[ZA1003] Invalid passphrase` | Wrong passphrase or wrong keychain entry; lost passphrase → `za-cli login` and re-enter credentials. |
| `No console available. Run in a terminal.` | Login is interactive by design; run it in a real terminal. |
| Behind a proxy | `za-cli --proxy http://user:pass@proxy:3128 login` or `settings.json` network keys. |
