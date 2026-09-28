# 16 · Recipes and Use Cases

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> End-to-end walkthroughs chaining the commands documented elsewhere. Sample IDs: workspace `2148712000000012345`, table **Orders** `2148712000000054321`.

---

## A. First ten minutes

```bash
za-cli --version && za-cli doctor
za-cli login                                     # 6 prompts, verified against Zoho, then saved
za-cli profile info
za-cli workspace list                            # copy a workspace ID
za-cli view list -w 2148712000000012345          # copy a view ID
za-cli view preview -w 2148712000000012345 2148712000000054321
za-cli export -w 2148712000000012345 -v 2148712000000054321 -f csv -O orders.csv
za-cli                                           # explore the same things by menu; press x to learn commands
```

## B. Headless / CI setup

```bash
# 1. once, on a workstation: za-cli login --profile ci-prod
# 2. ship ~/za-cli/config (encrypted) to the runner, or run login there once
# 3. deliver the passphrase as a CI secret
export ZA_CLI_PASSPHRASE="$ZA_PASSPHRASE_SECRET"          # or env:ZA_PASSPHRASE_SECRET
export NO_COLOR=1
za-cli -q -p ci-prod -o json --envelope workspace list > workspaces.json
test "$(jq -r .error workspaces.json)" = "null"
```

GitHub Actions sketch:

```yaml
- uses: actions/setup-java@v4
  with: { distribution: temurin, java-version: '17' }
- run: unzip -q za-cli-1.0.0-dist.zip && za-cli-1.0.0/setup.sh install --install-dir "$HOME/za-cli"
- run: |
    export PATH="$HOME/za-cli/bin:$PATH"
    za-cli -q -o json --envelope view list -w ${{ vars.ZA_WORKSPACE }} --filter "viewType=Table"
  env:
    ZA_CLI_PASSPHRASE: ${{ secrets.ZA_CLI_PASSPHRASE }}
```

## C. Nightly export on a schedule

```bash
za-cli alias add nightly-orders "export -w 2148712000000012345 -v 2148712000000054321 -f csv -O /data/orders-nightly.csv --force"
# crontab
0 2 * * * ZA_CLI_PASSPHRASE=keychain:za-cli-prod /home/you/za-cli/bin/za-cli -q alias run nightly-orders >> /var/log/za-cli-nightly.log 2>&1
```

Add `--criteria "\"OrderDate\" >= '$(date -d yesterday +%F)'"` in a wrapper script for incremental extracts (aliases themselves do not expand `$VAR`).

## D. Bulk load a monthly file safely

```bash
WS=2148712000000012345; T=2148712000000054321
za-cli view columns -w $WS $T --fields columnName,dataTypeName -o csv        # verify headers
za-cli export -w $WS -v $T -f csv -O backup-$(date +%F).csv --force            # safety net
za-cli import -w $WS -v $T -f monthly.csv -t updateadd --matching-columns OrderId --on-error skiprow
za-cli view preview -w $WS $T --limit 10
za-cli org resources                                                          # check row headroom afterwards
```

## E. Create a table, load it, build a chart, share it

```bash
WS=2148712000000012345
T=$(za-cli -q -o json view create-table -w $WS --name Orders \
      --column "OrderId:NUMBER" --column "Region:PLAIN" --column "Revenue:CURRENCY" --column "OrderDate:DATE" | jq -r .tableId)
za-cli import -w $WS -v $T -f orders.csv
za-cli view create-chart -w $WS --base-table Orders --name "Revenue by region" --type bar --x "Region:actual" --y "Revenue:sum"
za-cli view create-summary -w $WS --base-table Orders --name "Revenue Summary" --group "Region:Orders:actual" --agg "Revenue:Orders:sum"
za-cli view create-pivot -w $WS --base-table Orders --name "Region × Quarter" --row "Region:Orders:actual" --col "OrderDate:Orders:quarter" --data "Revenue:Orders:sum"
CHART=$(za-cli -q -o json view list -w $WS --filter "viewName=Revenue by region" | jq -r '.[0].viewId')
za-cli view share -w $WS $CHART --email alice@example.com,bob@example.com --permission read,export
za-cli view url -w $WS $CHART
```

## F. Fix data in place (typed, with a preview)

```bash
WS=2148712000000012345; T=2148712000000054321
CRIT="\"Region\" = 'Eastern'"
za-cli export -w $WS -v $T -f csv -O check.csv --criteria "$CRIT" --force && wc -l check.csv   # rows that will change
za-cli view update-row -w $WS $T --data '{"Region":"East"}' --criteria "$CRIT"
za-cli view delete-rows -w $WS $T --criteria "\"Region\" = 'Test'"                             # no prompt in direct mode!
```

Or run `za-cli`, open the table, choose **Update Row** / **Delete Rows** for a guided picker with an affected-row count and double confirmation.

## G. Access review / audit

```bash
WS=2148712000000012345
za-cli org admins -o csv > admins.csv
za-cli org users --filter "status!=active" --fields emailId,status -o csv > inactive-users.csv
za-cli workspace users $WS -o csv > ws-users.csv
za-cli workspace share-info $WS -o json > ws-sharing.json
for v in $(za-cli -q -o json view list -w $WS | jq -r '.[].viewId'); do
  za-cli -q view share-info -w $WS $v -o ndjson
done > view-shares.ndjson
za-cli workspace trash $WS
```

MCP alternative: give an agent `--access-level=read --categories=metadata,admin,workspace` and invoke the `audit_workspace` prompt.

## H. Onboard / offboard a person

```bash
WS=2148712000000012345
za-cli org users add newhire@example.com
za-cli workspace users add $WS newhire@example.com -r viewer
za-cli view share -w $WS 2148712000000054340 --email newhire@example.com --permission read

# offboarding
za-cli view remove-share -w $WS 2148712000000054340 --email leaver@example.com
za-cli workspace users remove $WS leaver@example.com
za-cli org users remove leaver@example.com
```

## I. Multi-organization operations

```bash
za-cli profile add acme-staging && za-cli profile add acme-eu --dc eu
for p in acme-prod acme-staging acme-eu; do
  echo "== $p"; za-cli -q -p "$p" -o csv workspace list --fields workspaceId,workspaceName
done
```

Per-profile settings (`config/profiles/acme-eu.settings.json`) can carry a different proxy or output format.

## J. Find things

```bash
za-cli search churn                              # org-wide, names only
za-cli search revenue -w 2148712000000012345 -o json | jq '.matches[] | select(.matchedOn=="column name")'
za-cli view list -w 2148712000000012345 --filter "viewType=Dashboard"
za-cli workspace list --filter "workspaceName~finance"
```

## K. Diagnose a stale table

```bash
za-cli workspace datasources 2148712000000012345 -d          # last sync status/time per schedule and view
za-cli view get -w 2148712000000012345 2148712000000054321   # lastModifiedTime
```

## L. AI agent setups

```jsonc
// read-only explorer (Claude Desktop / Claude Code / Cursor)
{"mcpServers":{"zoho-analytics-read":{"command":"/home/you/za-cli/bin/za-cli",
  "args":["mcp-server","--access-level=read","--result-encoding","auto","--max-result-tokens","2000"],
  "env":{"ZA_CLI_PASSPHRASE":"keychain:default"}}}}
```

```bash
# report builder that can never delete
za-cli mcp-server --access-level write --categories metadata,row,modelling,view --exclude-tools delete_view
# rehearsal against production with zero risk
za-cli mcp-server --access-level full --dry-run
# two orgs in one process
za-cli mcp-server --profiles acme-prod,acme-staging --access-level read
```

Then ask: *"List my Zoho Analytics workspaces"*, *"Which table has churn data?"*, *"Build a bar chart of revenue by region in Sales Analytics"* (uses the `build_report` prompt on write/full servers).

## M. Behind a corporate proxy

```bash
za-cli --proxy proxy.corp.example:8080 doctor
za-cli --proxy http://svc:pa55@proxy.corp.example:3128 login
# permanent
cat > ~/za-cli/config/settings.json <<'JSON'
{ "network.proxy.host": "proxy.corp.example", "network.proxy.port": "8080",
  "network.proxy.username": "svc-account", "network.proxy.password": "p@ss:w/rd",
  "network.connectTimeoutSeconds": 15, "network.readTimeoutSeconds": 300 }
JSON
```

## N. Machine-readable everything

```bash
za-cli -q -o json --envelope org info | jq .data.planName
za-cli -q -o ndjson org users | jq -c 'select(.role|test("admin";"i"))'
za-cli -q -o csv workspace folders 2148712000000012345 | column -s, -t
za-cli -q -o yaml view get -w 2148712000000012345 2148712000000054321
```

## O. Clean uninstall / reinstall without losing logins

```bash
~/za-cli/setup.sh uninstall          # keeps config/ and logs/
# … later …
unzip za-cli-1.0.0-dist.zip && cd za-cli-1.0.0 && ./setup.sh install
za-cli profile list                  # profiles still there
```
