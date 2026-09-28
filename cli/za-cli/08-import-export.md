# 08 · Import and Export

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> `import` and `export` talk to the Zoho **Bulk API** directly (not via the tool catalog), so `--dry-run` has no effect on them. Both need `-w` and `-v`. MCP equivalents: `import_data`, `export_data` ([13](13-mcp-tool-catalog.md)).

---

## 1. `export` — save a view to a local file

```text
Usage: za-cli export [-hVy] [--dry-run] [--envelope] [--force] [--show-hidden-cols] [--show-personal-cols]
                     [--columns=<columns>] [--connect-timeout=<n>] [--criteria=<criteria>] [-f=<format>]
                     [--fields=<f>] [--filter=<f>] [--limit=<recordLimit>] [-o=<outputFormat>] -O=<outputFile>
                     [-P=<passphrase>] [--proxy=<proxy>] [--read-timeout=<n>] [--require-confirmation=<p>]
                     -v=<viewId> -w=<workspaceId>
Export data from a view/table
      --columns=<columns>    Columns to export (comma-separated)
      --criteria=<criteria>  Filter criteria (e.g. "Name" = 'John')
  -f, --format=<format>      Export format: csv, json, xml, xls, pdf, html, image (default: csv)
      --force                Overwrite the output file if it already exists
      --limit=<recordLimit>  Maximum number of records to export
  -O, --out=<outputFile>     Output file path
      --show-hidden-cols     Include hidden columns in export
      --show-personal-cols   Include personal columns in export
  -v, --view=<viewId>        View/Table ID
  -w, --workspace=<workspaceId>
                             Workspace ID
```

### 1.1 The two letters that matter

| Flag | Meaning |
|---|---|
| **`-O`, `--out`** (capital) | the **file to write** — required |
| **`-o`, `--output`** (lowercase) | how za-cli prints its own **messages** (table/json/…) — unrelated to the file |

`--limit` here is a real record cap sent to Zoho (unlike the display-only `--limit` on list commands).

### 1.2 Formats

| `-f` | File | Notes |
|---|---|---|
| `csv` (default) | comma-separated values | tables, query tables, reports |
| `json` | JSON | |
| `xml` | XML | |
| `xls` | Excel spreadsheet | |
| `pdf` | PDF document | charts, dashboards (dashboards must be PDF for some layouts) |
| `html` | HTML page | charts, dashboards |
| `image` | PNG/JPG image | charts, dashboards (interactive default extension `.png`) |

Recommended per view type: tables → `csv`; charts → `html`/`pdf`/`image`; dashboards → `pdf`. Unsupported combination → `This view cannot be exported in the requested format. Dashboards must be exported as PDF.` (Zoho 8133).

### 1.3 Examples

```bash
# whole table to CSV
za-cli export -w 2148712000000012345 -v 2148712000000054321 -f csv -O sales.csv

# filtered rows, chosen columns, capped, overwrite
za-cli export -w 2148712000000012345 -v 2148712000000054321 -f csv -O east.csv \
  --criteria "\"Region\" = 'East'" --columns "Region,Revenue,OrderDate" --limit 5000 --force

# dashboard as PDF, chart as image
za-cli export -w 2148712000000012345 -v 2148712000000054340 -f pdf -O dashboard.pdf
za-cli export -w 2148712000000012345 -v 2148712000000054330 -f image -O revenue.png

# include hidden and personal columns
za-cli export -w … -v … -O full.csv --show-hidden-cols --show-personal-cols

# machine-readable confirmation
za-cli -o json --envelope export -w … -v … -O sales.csv --force
```

### 1.4 Output

Table mode:

```text
$ za-cli export -w 2148712000000012345 -v 2148712000000054321 -f csv -O sales.csv
  Exporting data
     Format: csv

  ✓  Export successful!
     File:  /home/you/reports/sales.csv
     Size:  184 KB
```

JSON mode (preamble suppressed):

```json
{"status": "exported", "format": "csv", "outputPath": "/home/you/reports/sales.csv", "sizeBytes": 188416}
```

Relative `-O` paths resolve against the current directory; missing parent directories are created.

### 1.5 Errors

```text
$ za-cli export -w … -v … -O sales.csv                      # file exists
  ✗  [ZA1004] Operation failed
     Output file already exists: /home/you/reports/sales.csv. Pass --force to overwrite, or choose a different --out path.
(exit 1)

$ za-cli export -w … -v … -O x.txt -f xlsx                  # bad format, no console
  ✗  [ZA1004] Operation failed
     Invalid format 'xlsx'. Supported: csv, json, xml, xls, pdf, html, image
(exit 1)
```

With a console attached the bad-format case instead prompts `Format (csv/json/xml/xls/pdf/html/image):` (blank → `Export cancelled: no format provided.`). Missing `-O` → `Missing required option: '--out=<outputFile>'` (exit 2). Bad criteria → `The specified criteria is invalid.` (8002).

### 1.6 Criteria reminders

Column names in double quotes, values in single quotes; escape for your shell:

```bash
--criteria "\"Region\" = 'East'"
--criteria "\"Revenue\" > '1000' AND \"OrderDate\" >= '2026-01-01'"
--criteria '"Region" LIKE '"'"'%east%'"'"''     # bash-safe LIKE with single quotes
```

Use `export --criteria` as the **preview** step before `view update-row` / `view delete-rows` with the same condition.

---

## 2. `import` — load a CSV/JSON file into an existing table

```text
Usage: za-cli import [-hVy] [--auto-identify] [--dry-run] [--envelope] [--connect-timeout=<n>]
                     [--date-format=<dateFormat>] [--delimiter=<delimiter>] -f=<file> [--fields=<f>]
                     [--file-type=<fileType>] [--filter=<f>] [--matching-columns=<matchingColumns>]
                     [-o=<outputFormat>] [--on-error=<onError>] [-P=<passphrase>] [--proxy=<proxy>]
                     [--read-timeout=<n>] [--require-confirmation=<p>] [--skip-rows=<skipRows>]
                     [-t=<importType>] -v=<viewId> -w=<workspaceId>
Import data from a file into a table
      --auto-identify        Auto identify CSV format (default: true)
      --date-format=<dateFormat>
                             Date format (e.g. dd/MM/yyyy, yyyy-MM-dd HH:mm:ss)
      --delimiter=<delimiter>
                             CSV delimiter: comma, tab, semicolon, pipe
  -f, --file=<file>          Path to data file (CSV/JSON)
      --file-type=<fileType> File type (csv, json). Auto-detected if not specified.
      --matching-columns=<matchingColumns>
                             Matching columns for updateadd (comma-separated)
      --on-error=<onError>   On import error: abort, skiprow, setcolumnempty
      --skip-rows=<skipRows> Number of top rows to skip
  -t, --type=<importType>    Import type: append, truncateadd, updateadd (default: append)
  -v, --view=<viewId>        View/Table ID
  -w, --workspace=<workspaceId>
                             Workspace ID
```

The target table **must already exist** (`view create-table` first). File headers should match the table's column names — check with `view columns`.

### 2.1 Import modes

| `-t` | Effect | Needs |
|---|---|---|
| `append` (default) | Adds rows; existing rows untouched | — |
| `updateadd` | Updates rows whose key columns match, adds the rest (upsert) | `--matching-columns col[,col]` |
| `truncateadd` | **Deletes every existing row**, then loads the file | certainty; export first |

### 2.2 File handling options

| Option | Use when |
|---|---|
| `--file-type csv\|json` | extension is not `.csv`/`.json` |
| `--delimiter comma\|tab\|semicolon\|pipe` | file is not comma-separated |
| `--date-format "dd/MM/yyyy"` | dates are not auto-recognised (Java `SimpleDateFormat` patterns, e.g. `yyyy-MM-dd HH:mm:ss`) |
| `--skip-rows N` | title rows precede the header |
| `--on-error abort\|skiprow\|setcolumnempty` | choose whether a bad row stops everything, is skipped, or has the bad cell blanked |
| `--auto-identify` | let Zoho detect the CSV dialect (default true) |

### 2.3 Examples

```bash
# check headers first
za-cli view columns -w 2148712000000012345 2148712000000054321 --fields columnName,dataTypeName

# plain append
za-cli import -w 2148712000000012345 -v 2148712000000054321 -f monthly-figures.csv

# upsert on OrderId
za-cli import -w 2148712000000012345 -v 2148712000000054321 -f monthly-figures.csv \
  -t updateadd --matching-columns OrderId

# replace all rows, tab-delimited, custom dates, skip 2 title rows, skip bad rows
za-cli import -w … -v … -f dump.txt --file-type csv --delimiter tab \
  --date-format "dd/MM/yyyy" --skip-rows 2 --on-error skiprow -t truncateadd

# JSON array file
za-cli import -w … -v … -f rows.json -t append

# then verify
za-cli view preview -w 2148712000000012345 2148712000000054321
```

### 2.4 Output

```text
$ za-cli import -w 2148712000000012345 -v 2148712000000054321 -f monthly-figures.csv -t append
  Importing monthly-figures.csv
     Type: append  │  Format: csv

  ✓  Import successful!
     Total Rows:  1250
     Success:     1248
     Warnings:    2

  CLI: $ za-cli import -w 2148712000000012345 -v 2148712000000054321 -f "monthly-figures.csv" -t append -P <passphrase>
```

The `Importing …` preamble is printed in every mode; in `-o json` the raw Zoho result follows. The fields za-cli itself reads are `importSummary.totalRowCount`, `importSummary.successRowCount` and `importSummary.warnings`; any further keys are whatever the API returns:

```json
{
  "importSummary": {
    "totalRowCount": 1250,
    "successRowCount": 1248,
    "warnings": 2
  }
}
```

If `importSummary` is absent the whole response is printed as-is.

### 2.5 Validation and errors

Each check prompts for a correction when a console is attached, and fails hard in scripts:

| Problem | Script (no console) | Interactive prompt |
|---|---|---|
| file missing | `File not found: /abs/path` (exit 1) | `Enter valid file path:` → blank: `Import cancelled: no file provided.` |
| unknown extension | `Cannot detect file type. Use --file-type to specify.` | `File type (csv/json):` |
| bad `-t` | `Invalid import type 'x'. Supported: append, truncateadd, updateadd` | `Import type (append/truncateadd/updateadd):` |
| `updateadd` without keys | `Matching columns are required for updateadd imports.` | `Matching columns (comma-separated):` |

Zoho-side messages you may see: `None of the column names in the source data match the table's columns.` (7235) · `The file could not be imported.` (7249) · `The import was aborted.` (7232) · `A value does not match the data type of its column.` (7507) · `A mandatory column was left without a value.` (7511) · `The file size exceeds the supported limit.` (8513).

---

## 3. Inline rows vs file import

| Need | Use |
|---|---|
| a handful of rows typed inline | `za-cli view import-rows … --rows '[{…}]'` ([07 § 4.5](07-direct-commands-view.md)) |
| a single row | `za-cli view add-row … -d '{…}'` |
| hundreds+ rows, scheduled loads, CSV from another system | `za-cli import` |
| guided, with prompts | interactive Table menu → **Import Data** |

---

## 4. Interactive equivalents

Table menu → **Export Data** (format menu, column multi-select, criteria, hidden/personal toggles, path with Tab completion, review screen) and **Import Data** (file path with Tab completion, mode menu, matching-column picker for `updateadd`, advanced options, review). Chart menus export `CSV/JSON/PDF/Image`; dashboards `CSV/JSON/PDF/HTML/Image`. Files are written to the folder you started za-cli from unless you give a path.

---

## 5. MCP equivalents

`export_data` (`format` required; default output name `za-export-<ws>-<view>.<format>`; path must be inside the server's working directory) and `import_data` (`filePath` must be inside the cwd, `.csv`/`.json` only). Both MUTATING → need `--access-level write` or `full`. Details in [13 § 5](13-mcp-tool-catalog.md).

---

## 6. Scheduling an export (cron)

```bash
za-cli alias add nightly-export "export -w 2148712000000012345 -v 2148712000000054321 -f csv -O /data/nightly.csv --force"
# crontab -e
0 2 * * * ZA_CLI_PASSPHRASE=keychain:za-cli-prod /home/you/za-cli/bin/za-cli -q alias run nightly-export >> /var/log/za-export.log 2>&1
```

Windows: Task Scheduler running `za-cli alias run nightly-export` with `ZA_CLI_PASSPHRASE` set as a task environment variable.
