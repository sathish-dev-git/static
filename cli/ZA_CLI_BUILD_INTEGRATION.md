# za-cli — how the CLI tool is built from this repository

`za-cli` is the Zoho Analytics command-line tool. It used to live in its own Maven
repository (`ZA_CLI`). It is now a product of **this** repository, built by the same
Ant pipeline as the JDBC driver and the Upload Tool, and shipped inside `ZDBStatic.zip`.

This document lists every change made to bring it in, and how to build and maintain it.

---

## 1. What you get

After a full build, `ZDBStatic.zip` contains a new folder:

```
ZDBStatic.zip
└── cli/
    └── 1.0.0/
        ├── za-cli-1.0.0.zip   <- portable distribution (jar + launchers + docs + licenses)
        └── za-cli.jar         <- the single-file executable jar on its own
```

`za-cli-1.0.0.zip` unpacks to:

```
za-cli-1.0.0/
├── za-cli.jar          all classes + all dependencies, Main-Class set
├── za-cli / za-cli.bat launchers (Linux/macOS, Windows)
├── setup.sh / setup.bat
├── README.txt, help.html, settings.example.json
├── licenses/           third-party licenses
└── logs/
```

Running requires **Java 17 or newer**.

---

## 2. Where the code lives

| Path | Contents |
|---|---|
| `source/cli/source/` | Java sources (`com.zoho.analytics.cli.**`, 147 files) |
| `source/cli/resources/za-cli.properties` | version descriptor read by the tool at runtime (`--version`) |
| `source/cli/dist/` | launchers, setup scripts, README, help, example settings |
| `source/cli/test/` | JUnit 5 tests (not run by the Ant build; kept with the source) |

Why `source/cli/test/` and not `qa/`: the repository commit hook only allows the root
folders from the cm-server directory standard (`build`, `source`, `product_package`, …).
`qa/` is not on that list, so new files under it are rejected.

---

## 3. Build files added or changed (`build/`)

| File | Change |
|---|---|
| `ant.properties` | new **ZA CLI** section + wiring into the build orders (details below) |
| `dist-meta.json` | `"cli/*"` added to the `_purge` list |
| `zacli_manifest.txt` | *new* — manifest for the jar (`Main-Class: com.zoho.analytics.cli.ZaCli`) |
| `zacli_javac.sh` | *new* — javac wrapper that compiles this module with JDK 17 |
| `zacli_check.sh` | *new* — build-time guards (dependency check, smoke test) |

### 3.1 Build orders

Two new orders in `ant.properties`, added to **both** `targetfull_order` (IDC machines)
and `local_order` (developer machines):

```
zacli             -> right after jdbc_v2      (needs AnalyticsClient-<ver>-US.jar from javalib_v2_us)
zacli_staticzip   -> right after jdbc_v2_staticzip
```

A third order builds **only** the CLI, for local use:

```
cli_order=checkout,zrcsv,javalib_v2_us,zacli,zacli_staticzip
```

`zdbstatic_order` gained `deletetask:del_zacli_home`, and `zip_zdbstatic_dir_tozip`
now includes `cli`.

### 3.2 What `zacli_order` does, step by step

```
mkdir:zacli_stage_mk         create the staging dir (so the next step never fails on a fresh checkout)
deletetask:del_zacli_stage   empty it — a jar left over from an earlier run must never be merged
copy:zacli_tpjars_cp         collect third-party jars from thirdparty_packages (matched by NAME, see §4)
copy:zacli_zohojars_cp       collect Zoho jars built earlier: AnalyticsClient-<ver>-US, zr-csv, json, codec, logging
mergejar:zacli_fatjar        fold every dependency into one jar  -> pkg/cli/za-cli-1.0.0/za-cli.jar
exec:zacli_dep_check         GUARD: every required library must be present as classes in that jar
chmod:zacli_javac_exec       make the javac wrapper executable (checkouts may drop the exec bit)
compilesrc:zacli_src         compile the CLI against the merged jar, via zacli_javac.sh (JDK 17, UTF-8)
copy:zacli_resources_cp      put za-cli.properties next to the classes
genjar:zacli                 add the CLI classes + Main-Class manifest into the same jar (in-place update)
exec:zacli_smoke             GUARD: run `java -jar za-cli.jar --version`, must print "za-cli <zacli_version>"
copy:zacli_dist_cp           launchers / setup / docs
copy:zacli_license_cp        third-party licenses (both naming styles, see §4)
chmod:zacli_scripts          za-cli and setup.sh -> 755
ziptask:zacli_zip            output/za-cli-1.0.0.zip
```

`zacli_staticzip_order` then copies the zip and the jar to `output/cli/<zacli_version>/`,
which `zdbstatic` packs into `ZDBStatic.zip`.

Why the two-pass jar: `library.xml` has no equivalent of Maven's shade plugin. So the
dependencies are merged first (`mergejar`), the sources are compiled against that single
jar, and `genjar` (which updates in place when a manifest is given) adds the CLI's own
classes and the `Main-Class`.

### 3.3 Version

```
zacli_version=1.0.0
```

This one property names the zip, the jar folder in `ZDBStatic.zip`, and is what the smoke
test expects the jar to report. **Keep it in sync with `version=` in
`source/cli/resources/za-cli.properties`** — the smoke test fails the build if they differ.

### 3.4 Java 17 — `zacli_javac.sh`

The rest of this product compiles with whatever JDK runs Ant (older than 17 on IDC
machines, with a US-ASCII locale). za-cli is Java 17 code with UTF-8 sources, so it is
compiled by a **forked javac** through the wrapper:

```
zacli_src_javac_exe=${basedir}/zacli_javac.sh
zacli_src_javac_compiler=extJavac
```

The wrapper looks for a JDK in this order — `ZACLI_JDK_HOME`, `JAVA17_HOME`, `JDK17_HOME`,
then `javac` on `PATH` — and always runs `javac -encoding UTF-8 --release 17`. Nothing
else in the build is affected. **A build machine needs a JDK 17+ reachable one of those
ways.** If none is found the build stops with `release version 17 not supported`.

---

## 4. Third-party dependencies

The CLI's dependencies come from the thirdparty checkout (`CSV/thirdparty_packages`),
which is produced by the thirdparty Ivy file and published as `tp_components.zip`.
They land flat in `thirdparty_packages/lib/` as `<artifact>-<version>.jar`.

### 4.1 Jars added to the thirdparty Ivy file for za-cli

| Jar | Purpose |
|---|---|
| `picocli-4.7.7` | command-line parsing |
| `jline-3.22.0` | interactive terminal UI |
| `jansi-2.4.0` | Windows terminal support for jline (without it Windows gets a dumb terminal) |
| `mcp-core-2.0.0` | MCP server (`za-cli mcp-server`) — the actual classes |
| `mcp-json-jackson3-2.0.0` | MCP ↔ Jackson 3 bridge |
| `mcp-2.0.0` | POM-only aggregator; contains **no classes**. Harmless, but it satisfies nothing on its own |
| `jackson-core`, `jackson-databind`, `jackson-dataformat-yaml` (3.x), `jackson-annotations-2.20` | Jackson 3 used by MCP |
| `reactor-core-3.7.19`, `reactive-streams-1.0.4` | MCP transport |
| `slf4j-api-2.0.19` | MCP logging (no binding shipped; NOP logger, same as upstream) |
| `json-schema-validator-3.0.5`, `itu-1.14.0`, `snakeyaml-engine-3.0.1` | MCP tool-schema validation |
| `commons-codec-1.21.0` | AnalyticsClient |

Already in this build and reused: `AnalyticsClient-<Client-Version>-US.jar`
(from `javalib_v2_us`), `zr-csv.jar` (from `zrcsv`), `json-20231013.jar`,
`commons-logging-api.jar`.

**Not** used: Apache `commons-csv`. The CLI parses CSV with the product's own `zr-csv`
(see §5). The old `commons_csv/commons-csv.jar` in the tree is explicitly excluded from
the merge because it defines the same class names.

### 4.2 How jars are picked up — by name, not by path

```
zacli_tpjars_cp_copy_includes=**/picocli*.jar **/jline*.jar **/jansi*.jar **/mcp*.jar
    **/jackson-*.jar **/json-schema-validator*.jar **/itu-*.jar **/snakeyaml-engine*.jar
    **/slf4j-api*.jar **/reactor-core*.jar **/reactive-streams*.jar
```

Because matching is by jar name anywhere under `thirdparty_packages`, a version bump in
the Ivy file needs **no change** here. Licenses are matched the same way, in both styles
found in the tree: `LICENSE_<NAME>_<ver>.txt` (hg era) and `<artifact>-<ver>-license.txt`
(Ivy classifier).

### 4.3 Guards — `zacli_check.sh`

Two checks stop the build early with a clear message instead of hundreds of javac errors
or a runtime `NoClassDefFoundError`:

* **`deps`** (after `mergejar`): 18 signature classes must exist inside the merged jar —
  one per library above. If one is missing it prints exactly which Ivy artifact to add,
  e.g. `MISSING io/modelcontextprotocol/server/McpServer.class <- add mcp-core`.
  It also flags POM-only jars and **warns when the three Jackson 3 jars are not the same
  version** (they are released as a set; a mismatch compiles but can fail at runtime).
* **`smoke`** (after `genjar`): runs the shipped jar and requires
  `za-cli <zacli_version>` — proves manifest, class level and the picocli/jline bootstrap
  link at runtime.

---

## 5. Source changes vs the upstream ZA_CLI repository

Eight files differ from upstream. Anyone syncing new code from `ZA_CLI` must re-apply
these — the repository's Java Code Check rejects the commit otherwise.

| Reason | Files |
|---|---|
| **CSV parsing on `zr-csv`** instead of Apache commons-csv (one fewer third-party jar). Behaviour matched against upstream on 24 inputs; only known difference: trailing whitespace of *unquoted* fields is trimmed. | `core/TextUtils.java` (+ comment in `test/.../TextUtilsTest.java`) |
| Code-check rule *ControlStatementBraces* — braces on one-line `if`/`for` | `command/LoginCommand`, `LogoutCommand`, `ProfileCommand`, `core/ShellCompletion`, `ui/RichUI` |
| *FinalMemberRule* — `private static final` fields in `UPPER_SNAKE` | `config/Telemetry` (`COUNTS`, `LAST_FLUSHED`, `FLUSH_EXECUTOR`), `service/ClientFactory` (`CLIENT_CACHE`, `ORG_ID_CACHE`, `DATA_CENTRE_CACHE`, `PROFILE_LOCKS`) |
| *AvoidPlusOperatorInLogger* — `{0}` parameters in `Logger.log` | `service/decorator/RetryInterceptor` |
| *AvoidUsingHardCodedIP* — `InetAddress.getLoopbackAddress()` instead of `"127.0.0.1"` | tests: `OpenAiProviderTest`, `DoctorCommandTest`, `ExitCodeContractTest` |
| `.bat` launchers stored with CRLF (Maven used to convert them at package time) | `dist/za-cli.bat`, `dist/setup.bat` |

---

## 6. Building locally

From `build/`:

```sh
# full CLI-only build: thirdparty checkout, zr-csv, v2 client, za-cli, ZDBStatic staging
ant -Dtarget=cli -DClient-Version=2.9.0 \
    -Dcmtp_hgroot=https://build.zohocorp.com/tp/components/CSV_UPLOAD_TOOL_BRANCH/latest/tp_components.zip
```

* `-DClient-Version` is passed by the IDC framework; locally you must supply it, or the
  client jar is literally named `AnalyticsClient-${Client-Version}-US.jar`.
* The thirdparty download step only recognises URLs containing `/tp/components/`. For a
  zip published elsewhere (e.g. `…/tp_pkg/…/webhost/…`), unpack it into `build/CSV/`
  yourself and skip that step:

```sh
rm -rf CSV && mkdir -p CSV/zips
curl -k -u "<user>:<password>" -o CSV/zips/tp_components.zip  <zip url>
( cd CSV && unzip -qo zips/tp_components.zip )
ant -Dtarget=cli -DClient-Version=2.9.0 -Dcmtp_hgclone_needed=no
```

Result: `CSV/output/za-cli-1.0.0.zip` and `CSV/output/cli/1.0.0/`.

Quick test of the artifact:

```sh
unzip -q CSV/output/za-cli-1.0.0.zip -d /tmp/zatest
/tmp/zatest/za-cli-1.0.0/za-cli --version     # za-cli 1.0.0
/tmp/zatest/za-cli-1.0.0/za-cli doctor
/tmp/zatest/za-cli-1.0.0/za-cli mcp-server    # starts; says "Not logged in" without credentials
```

Things to know:

* Every Ant run overwrites the tracked `build/library.xml` with the copy from `hg_utils`.
  Run `git checkout -- build/library.xml` before committing.
* `ant clean` deletes `CSV/`, `hg_utils/` and `buildlogs/`.
* Build logs for each task are in `build/buildlogs/`.

---

## 7. Maintenance checklist

**Releasing a new CLI version**
1. Set `version=` in `source/cli/resources/za-cli.properties`.
2. Set `zacli_version=` in `build/ant.properties` to the same value.
3. Build; the smoke test confirms the two agree.

**Bumping a dependency** — change the version in the thirdparty Ivy file only. The
name-based globs pick up the new jar; `zacli_check.sh` confirms it actually contains the
classes.

**Adding a dependency** — add it to the Ivy file, add a `**/<name>*.jar` pattern to
`zacli_tpjars_cp_copy_includes`, and add one signature class to the `deps` list in
`zacli_check.sh` so its absence is caught early.

**Syncing code from the ZA_CLI repository** — copy `src/main/java` → `source/cli/source`,
`src/test/java` → `source/cli/test`, `src/dist` → `source/cli/dist`; then re-apply the
items in §5 (CSV on `zr-csv`, code-check rules, CRLF `.bat` files).

**Known open item** — the current Ivy file mixes Jackson 3 versions
(`jackson-core 3.2.2`, `jackson-databind 3.0.0`, `jackson-dataformat-yaml 3.1.1`). It
links and runs today; align them to one version at the next thirdparty publish to remove
the warning the build prints.
