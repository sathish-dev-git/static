# 02 · Installation and Setup

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> Source of truth: the shipped bundle `ZDBStatic/cli/1.0.0/za-cli-1.0.0/` (README.txt, help.html, setup.sh, setup.bat, za-cli, za-cli.bat) and the `za-cli.jar` `--help` output.

---

## 1. What you download

| Artifact | Description |
|---|---|
| `za-cli-1.0.0.zip` (a.k.a. `za-cli-1.0.0-dist.zip`) | Portable, cross-platform bundle. Same ZIP for macOS, Linux and Windows. |
| `za-cli.jar` | The standalone shaded Java application (also published beside the ZIP). |

Bundle layout after extraction:

```text
za-cli-1.0.0/
  za-cli.jar               cross-platform Java application (shaded, all dependencies inside)
  za-cli                   macOS/Linux launcher (POSIX sh)
  za-cli.bat               Windows launcher (cmd.exe batch)
  setup.sh                 macOS/Linux install / uninstall script (POSIX sh, no Bash needed)
  setup.bat                Windows install / uninstall script (cmd.exe batch, no PowerShell needed)
  README.txt               complete terminal help (plain text, ~1650 lines)
  help.html                complete browser help ("handbook") – same content, richer layout
  settings.example.json    every settings key at its default value (copy → config/settings.json)
  licenses/                third-party licence texts (picocli, jackson, MCP SDK, reactor, jansi, …)
  logs/                    rotating application logs are written here
```

Third-party components bundled (from `licenses/`): picocli 4.7.7, Jackson core 3.2.2, Jackson YAML 3.1.1, SnakeYAML Engine 3.0.1, json-schema-validator 3.0.5, MCP Java SDK 2.0.0 (+ `mcp-json-jackson3`), Reactor Core 3.7.19, Reactive Streams 1.0.4, SLF4J API 2.0.19, Jansi 2.4.0, org.json 20231013, Commons Codec, Commons Logging.

### What the launchers actually do

`za-cli` (sh) resolves symlinks back to the real install folder and runs:

```sh
exec java "-Dza.cli.home=$APP_HOME" -jar "$APP_HOME/za-cli.jar" "$@"
```

`za-cli.bat` does the equivalent:

```bat
java "-Dza.cli.home=%APP_HOME%" -jar "%APP_HOME%\za-cli.jar" %*
```

`za.cli.home` tells the jar where its home folder is (logs, config, bundled files). If you invoke the jar directly with `java -jar za-cli.jar …` the folder containing the jar is used.

---

## 2. Requirements

| Requirement | Detail |
|---|---|
| **Java 17 or later** on `PATH` | Check with `java -version`. za-cli refuses to start on older Java. (Verified here with OpenJDK 17.0.11; Java 21 also works.) |
| Network access to your Zoho data centre | `accounts.zoho.<dc>` for OAuth and `analyticsapi.zoho.<dc>` for data. Corporate proxies are supported (see [04 · Global options](04-global-options-and-output.md)). |
| A terminal | Any: Terminal.app, iTerm2, GNOME Terminal, cmd.exe, PowerShell, Windows Terminal, WSL, Git Bash. |
| Maven 3.6+ | **Only** if you build from source. Not needed to use the ZIP. |

Zoho prerequisites you will need at login time (see [03 · Authentication](03-authentication-and-profiles.md)):

- A Zoho API Console **Self Client** (or server application) with **Client ID**, **Client Secret** and a **Refresh Token** generated for scopes `ZohoAnalytics.data.all`, `ZohoAnalytics.metadata.read`, `ZohoAnalytics.modeling.all`.
- Your **Organization ID** (ZSOID) from Zoho Analytics → Settings → Organization.

---

## 3. Platform support matrix

| | macOS | Linux | Windows |
|---|---|---|---|
| Launcher | `za-cli` | `za-cli` | `za-cli.bat` |
| Setup script | `./setup.sh install` | `./setup.sh install` | `setup.bat install` |
| App files (default) | `~/za-cli` | `~/za-cli` | `%USERPROFILE%\za-cli` |
| Command on PATH | `~/za-cli/bin/za-cli` (symlink) | `~/za-cli/bin/za-cli` (symlink) | `%USERPROFILE%\za-cli\bin\za-cli.bat` (wrapper) |
| Shell profiles updated | zsh, bash, fish, POSIX sh | bash, zsh, fish, POSIX sh | User `PATH` in registry |
| Keychain secret backend | ✅ verified (`security`) | ⚠ implemented, unverified (`secret-tool`) | ⚠ implemented, unverified (Credential Manager) |

Everything else (direct commands, interactive mode, MCP server, profiles, import/export, encrypted credentials, rotating logs) works identically on all three.

---

## 4. Run without installing

You can use the bundle straight from the extracted folder:

```bash
# macOS / Linux
cd za-cli-1.0.0
./za-cli --help
./za-cli --version

# Windows (Command Prompt)
cd za-cli-1.0.0
za-cli.bat --help
za-cli.bat --version

# Any OS, jar directly
java -jar za-cli.jar --version
```

Sample output (captured from the shipped jar):

```text
$ java -jar za-cli.jar --version
za-cli 1.0.0
```

```text
$ java -jar za-cli.jar --help
Usage: za-cli [-hqVyv] [--dry-run] [--envelope] [--no-color]
              [--connect-timeout=<connectTimeout>] [--fields=<fields>]
              [--filter=<filter>] [-o=<outputFormat>] [-p=<profile>]
              [-P=<passphrase>] [--proxy=<proxy>]
              [--read-timeout=<readTimeout>]
              [--require-confirmation=<requireConfirmation>] [COMMAND]
Zoho Analytics CLI Tool
      --connect-timeout=<connectTimeout>
                            Connection timeout in seconds (default: 5, or
                              config.Settings' "network.connectTimeoutSeconds")
      --dry-run             Preview a mutating/destructive tool call instead of
                              running it; read-only calls are unaffected
      --envelope            With --output json, wrap results in the versioned
                              {data, meta, error} envelope (opt-in; bare JSON
                              is deprecated, see 'za-cli --help')
      --fields=<fields>     Comma-separated field-name allowlist applied to the
                              rendered result, regardless of --output
      --filter=<filter>     Comma-separated field=value/field!
                              =value/field~value conditions (ANDed) narrowing
                              an array result before it's rendered
  -h, --help                Show this help message and exit.
      --no-color            Disable ANSI color output (same effect as the
                              NO_COLOR env var)
  -o, --output=<outputFormat>
                            Output format: table, json, csv, tsv, yaml, or
                              ndjson (default: table, or config.Settings'
                              "output.format" when set)
  -p, --profile=<profile>   Use a specific profile instead of the active one
  -P, --passphrase=<passphrase>
                            Passphrase for credential decryption (avoids prompt)
      --proxy=<proxy>       HTTP proxy: host:port or http://[user:pass@]host:
                              port (falls back to HTTPS_PROXY/https_proxy env
                              var, then config.Settings' "network.proxy")
  -q, --quiet               Suppress deprecation notices and diagnostic
                              preambles
      --read-timeout=<readTimeout>
                            Read timeout in seconds (default: 0/no timeout, or
                              config.Settings' "network.readTimeoutSeconds")
      --require-confirmation=<requireConfirmation>
                            Confirmation threshold: destructive (default),
                              mutating, or none
  -v, --verbose             Print stack traces on error; repeat (-vv) for a
                              per-command diagnostic preamble
  -V, --version             Print version information and exit.
  -y, --yes                 Skip confirmation prompts entirely (equivalent to
                              --require-confirmation=none)
Commands:
  login       Authenticate and store credentials securely
  logout      Remove stored credentials
  workspace   Workspace operations
  view        View/Table operations
  import      Import data from a file into a table
  export      Export data from a view/table
  org         Organization operations
  profile     Manage login profiles (one per organization)
  search      Search workspace/table/column names for a keyword
  completion  Generate a shell completion script (bash, zsh, fish, or
                powershell)
  doctor      Diagnose Java, config, credentials, proxy, and connectivity in
                one command
  telemetry   Show or clear locally accumulated usage counters (opt-in, off by
                default)
  alias       Name a command line so it can be re-run later
  mcp-server  Start MCP (Model Context Protocol) server
```

> **Tip:** if your JVM prints `Picked up _JAVA_OPTIONS: …` before every command, that is your environment's `_JAVA_OPTIONS` variable, not za-cli. It goes to stderr and does not affect `-o json` output.

---

## 5. Install from the ZIP

### 5.1 macOS / Linux

```bash
unzip za-cli-1.0.0-dist.zip
cd za-cli-1.0.0
./setup.sh install
```

If the execute bit was lost during copy/extract:

```bash
sh setup.sh install
```

### 5.2 Windows

1. Extract `za-cli-1.0.0-dist.zip`.
2. Open **Command Prompt** inside the `za-cli-1.0.0` folder.
3. Run:

```bat
setup.bat install
```

### 5.3 What `install` does

`setup.sh --help`:

```text
Install or uninstall za-cli from this portable bundle.

Usage:
  ./setup.sh install [options]
  ./setup.sh uninstall [options]

Options:
  --install-dir <path>      Single unified folder for everything -- jar, launcher, config,
                              credentials, logs (default: $ZA_CLI_HOME, or $ZA_CLI_INSTALL_DIR,
                              or ~/za-cli). Same folder every OS uses by default; override it
                              once here and every subsystem follows.
  --purge-logs              Uninstall only: remove logs also; by default logs are preserved
  --purge-config            Uninstall only: also remove stored profiles and encrypted credentials.
                              This deletes the OAuth refresh token; reinstalling requires 'za-cli login'
                              again. Neither uninstall nor --purge-logs removes this by default.
  -h, --help                Show this help

Installed files (all under one root -- no separate bin/config folders elsewhere):
  <install-dir>/za-cli.jar
  <install-dir>/za-cli
  <install-dir>/za-cli.bat
  <install-dir>/setup.sh
  <install-dir>/setup.bat
  <install-dir>/README.txt
  <install-dir>/help.html
  <install-dir>/settings.example.json
  <install-dir>/logs/
  <install-dir>/config/         (settings, profiles, encrypted credentials)
  <install-dir>/bin/za-cli -> <install-dir>/za-cli   (added to PATH)
```

Steps performed:

1. Copies the bundle into the unified install folder (`~/za-cli` or `%USERPROFILE%\za-cli` by default).
2. Creates `bin/za-cli` (symlink) or `bin\za-cli.bat` (wrapper).
   - **macOS/Linux also**: links `za-cli` into the first directory that is *already* on your `PATH` —
     `~/.local/bin`, then `~/bin`. This is what makes the command work in the terminal you ran the
     installer from; a shell reads `PATH` once at startup, so a profile edit alone can only affect
     *new* shells. A directory not already on `PATH` is never created or used, and an existing
     `za-cli` that this installer did not create is never overwritten.
3. Adds that `bin` folder to `PATH`:
   - macOS/Linux: detects `$SHELL` and appends a guarded block to that shell's own files (zsh: `~/.zprofile`, `~/.zshrc`, `~/.profile`; bash: see below; fish: `~/.config/fish/config.fish` or `$XDG_CONFIG_HOME/fish/config.fish`, plus `~/.profile`; other POSIX shells: `~/.profile`).
   - Under **bash** the installer never creates a login file. Bash reads only the *first* of `~/.bash_profile`, `~/.bash_login`, `~/.profile` that exists, so creating one would permanently hide the others — on Debian/Ubuntu that would also stop `~/.bashrc` from loading, since the stock `~/.profile` is what sources it. The installer writes whichever of those three already exists (`~/.profile` if none does), plus `~/.bashrc` and `~/.profile`. If a `~/.bash_profile` created by an older build is already hiding your `~/.profile`, the installer says so and prints the one line that restores it.
   - Windows: writes the user `PATH` in the registry and broadcasts `WM_SETTINGCHANGE` so new cmd/PowerShell windows see it immediately.
4. **Prints plainly whether the command is usable now** — read that line. Either
   `Ready to use now: 'za-cli' is linked into <dir>, which is already on your PATH`, or the PATH-was-updated
   case with a one-line `export PATH="<bin>:$PATH"` to use it in the current shell, or the not-updated case
   with manual instructions.
5. Prints the resolved absolute launcher path and a ready-to-paste MCP JSON snippet, plus the exact `cp`/`copy` command to seed `settings.json` from `settings.example.json`.

The PATH block it writes (macOS/Linux):

```sh
# >>> za-cli PATH >>>
ZA_CLI_BIN_DIR="$HOME/za-cli/bin"
if [ -d "$ZA_CLI_BIN_DIR" ]; then
    case ":$PATH:" in
        *":$ZA_CLI_BIN_DIR:"*) ;;
        *) export PATH="$ZA_CLI_BIN_DIR:$PATH" ;;
    esac
fi
# <<< za-cli PATH <<<
```

> On macOS/Linux the command itself normally works straight away, because setup also links it into a
> directory already on your `PATH` (step 2). A **new terminal** is still needed for shell **completion**
> scripts, for Windows, and whenever setup reports that no such directory was available.

### 5.4 Custom install location

```bash
# macOS
./setup.sh install --install-dir /Applications/za-cli
# Linux
./setup.sh install --install-dir /opt/za-cli
# Windows
setup.bat install --install-dir C:\Tools\za-cli
```

Or set `ZA_CLI_HOME` (or the older `ZA_CLI_INSTALL_DIR`) before running setup. One folder holds **everything**: jar, launcher, `config/` (profiles, encrypted credentials, settings), `logs/`.

To split only the **config** folder out to another location, set the JVM system property `za.cli.config` (see [14 · Settings, files and logs](14-settings-files-and-logs.md)).

### 5.5 Default locations after install

| | macOS / Linux | Windows |
|---|---|---|
| App files | `~/za-cli` | `%USERPROFILE%\za-cli` |
| Command | `~/za-cli/bin/za-cli` | `%USERPROFILE%\za-cli\bin\za-cli.bat` |
| Config folder | `~/za-cli/config` | `%USERPROFILE%\za-cli\config` |
| Logs | `~/za-cli/logs` | `%USERPROFILE%\za-cli\logs` |
| Help files | `~/za-cli/help.html`, `~/za-cli/README.txt` | `%USERPROFILE%\za-cli\help.html`, `…\README.txt` |

---

## 6. Verify the installation

```bash
za-cli --version
za-cli --help
za-cli doctor
```

`doctor` output on a fresh, not-yet-logged-in machine (captured):

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

Exit code was **0** — warnings (⚠) never fail `doctor`; only ✗ does. `HTTP 400` from the accounts host is normal: it proves the server is reachable. See [10 · Helper commands](10-helper-commands.md) for every check.

---

## 7. PATH troubleshooting

**macOS/Linux** — first check the line setup printed. If it says `Ready to use now`, `za-cli` already resolves
in the current shell via `~/.local/bin` (or `~/bin`) and nothing further is needed. If PATH was updated but no
such directory existed, either open a new terminal or run the `export PATH="…"` line setup printed. If setup
reported PATH was *not* updated (e.g. every candidate profile file is read-only), paste the block from §5.3
into your shell's startup file yourself:

| Shell | File(s) |
|---|---|
| zsh (macOS default) | `~/.zshrc`, **and** `~/.zprofile` (Terminal.app opens a login shell) |
| bash | `~/.bashrc`, **and** your login file — whichever of `~/.bash_profile`, `~/.bash_login`, `~/.profile` already exists (bash reads only the first one, so do not create a new one) |
| fish | `~/.config/fish/config.fish` → `fish_add_path $HOME/za-cli/bin` |
| other POSIX sh | `~/.profile` |

**Windows** — if the registry write was blocked (Group Policy), either:

- Settings GUI: *Edit environment variables for your account → User variables → Path → New* → `%USERPROFILE%\za-cli\bin`, or
- `setx PATH "%PATH%;%USERPROFILE%\za-cli\bin"` (note: `setx` truncates a PATH longer than 1024 chars).

**WSL / Git Bash on Windows** track PATH separately from cmd/PowerShell — add the folder to *that* shell's profile using the macOS/Linux instructions.

---

## 8. Shell completion (optional but recommended)

```bash
za-cli completion <bash|zsh|fish|powershell>
```

Prints the script to stdout; **nothing is installed for you**. Bash and zsh get full, argument-aware completion generated by picocli (3700+ lines); fish and PowerShell get subcommand-name and option-flag completion only.

| Shell / system | Save to |
|---|---|
| bash, macOS + Homebrew (Intel or Apple Silicon) | `za-cli completion bash > "$(brew --prefix)/etc/bash_completion.d/za-cli"` |
| bash, Linux system-wide | `sudo sh -c 'za-cli completion bash > /etc/bash_completion.d/za-cli'` |
| bash, user-level, any OS | `mkdir -p ~/.local/share/bash-completion/completions && za-cli completion bash > ~/.local/share/bash-completion/completions/za-cli` |
| zsh | `za-cli completion zsh > "${fpath[1]}/_za-cli"` (or `source <(za-cli completion zsh)` from `.zshrc`) |
| fish | `za-cli completion fish > ~/.config/fish/completions/za-cli.fish` |
| PowerShell | `za-cli completion powershell >> $PROFILE` |

**Bash prerequisite:** the `bash-completion` package must be installed **and sourced**, or nothing will ever read the file.

- macOS: `brew install bash-completion` (not `@2` unless you run a Homebrew bash 4+ as login shell), then ensure `~/.bash_profile` contains
  `[[ -r "$(brew --prefix)/etc/profile.d/bash_completion.sh" ]] && . "$(brew --prefix)/etc/profile.d/bash_completion.sh"`
- Linux: usually pre-installed; else `sudo apt install bash-completion` / `sudo dnf install bash-completion`.
- Windows: not applicable outside WSL; use the PowerShell script.

Open a new terminal, then test: `za-cli wor<TAB>` → `za-cli workspace`. **Regenerate after every upgrade**; the script is a snapshot of the command tree.

Head of the generated scripts (captured):

```bash
$ za-cli completion bash | head -8
#!/usr/bin/env bash
#
# za-cli Bash Completion
# =======================
#
# Bash completion support for the `za-cli` command,
# generated by [picocli](https://picocli.info/) version 4.7.7.
```

```fish
$ za-cli completion fish | head -3
# za-cli fish completion — generated from the live command tree, not hand maintained.
# Regenerate with: za-cli completion fish
complete -c za-cli -f
```

```powershell
$ za-cli completion powershell | head -4
# za-cli PowerShell completion — generated from the live command tree, not hand maintained.
# Regenerate with: za-cli completion powershell
$zaCliCompletions = @{
    "" = @{
```

Unsupported shell:

```text
$ za-cli completion tcsh
Unknown shell 'tcsh'. Expected one of: bash, zsh, fish, powershell.
(exit code 1)
```

---

## 9. First login (summary)

```bash
za-cli login                      # 6 prompts: Client ID, Client Secret, Refresh Token, Org ID, data centre, passphrase (×2)
za-cli login --profile eu-prod --dc eu
za-cli profile info
za-cli workspace list
```

Full detail, all prompts, data centres and secret handling: [03 · Authentication and profiles](03-authentication-and-profiles.md).

---

## 10. Upgrade

There is no in-place updater and no update check. To upgrade:

1. Download the new ZIP, extract, `./setup.sh install` (or `setup.bat install`) into the same `--install-dir`. Config and logs are preserved (they live in `config/` and `logs/` and are never overwritten by install).
2. Regenerate shell completion scripts.
3. Read the deprecation notices printed on stderr (see [04 · Versioning](04-global-options-and-output.md)).

Version numbers follow **SemVer** (`MAJOR.MINOR.PATCH`). The single source of the version is `za-cli.properties` inside the jar; `za-cli --version` prints it.

---

## 11. Uninstall

```bash
# macOS / Linux
~/za-cli/setup.sh uninstall
~/za-cli/setup.sh uninstall --purge-logs        # also delete logs/
~/za-cli/setup.sh uninstall --purge-config      # also delete config/ (profiles, encrypted creds, settings, aliases)
~/za-cli/setup.sh uninstall --purge-logs --purge-config

# Windows
%USERPROFILE%\za-cli\setup.bat uninstall
%USERPROFILE%\za-cli\setup.bat uninstall --purge-logs
%USERPROFILE%\za-cli\setup.bat uninstall --purge-config
```

Behaviour:

- Default uninstall removes the program, launcher and PATH entry. **Logs and config are preserved** so reinstalling resumes where you left off.
- `--purge-config` deletes the OAuth refresh token with everything else; you are shown the folder and asked to confirm (`y`/`yes`) on Windows always, on macOS/Linux only when a terminal is attached. Reinstalling then needs `za-cli login` again.
- If your shell's current directory is inside the folder being deleted, the uninstaller `cd`s somewhere safe first and tells you so.
- Completion scripts you saved elsewhere are **not** removed; delete them by hand.
- Leftover config folder by hand: `~/za-cli/config` or `%USERPROFILE%\za-cli\config`.

### Pre-unified-layout installs

Very early builds scattered files across `%APPDATA%\za-cli`, `~/Library/Application Support/za-cli`, `~/.config/za-cli`, `~/.local/share/za-cli`, `%USERPROFILE%\bin\za-cli.bat`. za-cli never auto-migrates and no cleanup script ships; remove them by hand once the unified install works.

> Do **not** delete `~/.local/bin/za-cli`: current builds create that symlink deliberately (§5, step 2) and `setup.sh uninstall` removes it for you.

---

## 12. Other distribution channels (prepared, not yet published)

These exist in the source tree under `ZA_CLI/packaging/` and are ready for a real release URL/sha256. None is published yet.

| Channel | Artifact | Status |
|---|---|---|
| Homebrew | `packaging/homebrew/za-cli.rb` (depends on `openjdk@17`, writes its own `bin/za-cli` wrapper) | Formula written; needs release URL + sha256, then homebrew-core PR or a tap `brew tap your-org/za-cli && brew install za-cli` |
| Docker | `packaging/docker/Dockerfile` (multi-stage: temurin 17 JDK build → 17 JRE runtime, non-root user `za-cli`) | Builds and runs today: `docker build -f packaging/docker/Dockerfile -t za-cli:1.0.0 .` then `docker run --rm -it -v za-cli-config:/home/za-cli/.config/za-cli -e ZA_CLI_PASSPHRASE za-cli:1.0.0 workspace list` |
| winget | `packaging/winget/*.yaml` (3 manifests) | Structurally valid; unverified on a Windows machine |
| apt / yum | `packaging/linux/build-packages.sh` (via fpm) | Builds `.deb`/`.rpm` locally; no hosted repository yet |
| curl-pipe-sh | `install.sh` at repo root | Builds from source then hands off to `setup.sh` |

Releases are **unsigned**; verify downloads against `SHA256SUMS.txt`. A CycloneDX SBOM (`bom.json`) is generated per build.

---

## 13. Build from source (developers)

```bash
cd ZA_CLI
mvn clean package            # → target/za-cli-1.0.0-dist.zip, target/bom.json
./scripts/install-local.sh   # rebuild + install into ~/za-cli   (Windows: scripts\install-local.bat)
./scripts/install-local.sh --skip-build --install-dir /opt/za-cli
```

Stack: Java 17, Maven, picocli 4.7.7, Zoho Analytics Java Client SDK v2 (vendored under `lib/`, see `lib/PROVENANCE.md`), MCP Java SDK 2.0.0, Jackson 3, Jansi.

---

## 14. Quick smoke test script

```bash
#!/usr/bin/env sh
set -e
za-cli --version
za-cli doctor
za-cli profile list
za-cli alias list
za-cli telemetry
za-cli completion bash > /dev/null && echo "completion ok"
```

All of the above work **without** logging in and are safe to run in CI images to validate an install.
