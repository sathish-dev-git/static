# 05 · Direct Commands — `org`

> Part of the **za-cli 1.0.0 Reference Set**. Back to the [index](00-index.md).
> All commands here need a logged-in profile and accept the common options ([04](04-global-options-and-output.md)). IDs and values in samples are illustrative; layouts follow the real formatters.

```text
Usage: za-cli org [-hV] [COMMAND]
Organization operations
Commands:
  info       Show organization info and subscription details
  users      List and manage organization users
  admins     List organization admins
  resources  Show organization resource usage
```

Bare `za-cli org` prints `Usage: za-cli org [info|users|users add|users remove|admins|resources]`.

| Command | Safety | Catalog tool (MCP) | Dry-run applies | Pagination |
|---|---|---|---|---|
| `org info` | READ ONLY | `get_subscription` | no (direct SDK call) | — |
| `org users` | READ ONLY | `list_org_users` | no | ✅ |
| `org users add` | CHANGES DATA | `add_org_users` | no | — |
| `org users remove` | CHANGES DATA | `remove_org_users` | no | — |
| `org admins` | READ ONLY | `list_admins` | no | ✅ |
| `org resources` | READ ONLY | `get_resource_usage` | no | — |

> The `org` family calls the SDK directly (not through the tool catalog), so `--dry-run` has no effect here.

---

## 1. `org info` — organization and subscription details

```text
za-cli org info [common options]
```

Fetches the subscription details of the profile's organization (`GET /restapi/v2/subscription`) and adds `orgId`.

```text
$ za-cli org info

  ╭────────────────────────────────────────────────────────╮
  │ ● Details                                              │
  ├──────────────────┬─────────────────────────────────────┤
  │ Org ID           │ 20087654321                         │
  │ Plan Name        │ Premium                             │
  │ Add-Ons          │ Additional Users x10                │
  │ Trial Status     │ —                                   │
  │ Billing Date     │ 2027-01-31                          │
  │ Plan Type        │ PAID                                │
  │ Users            │ 15                                  │
  ╰──────────────────┴─────────────────────────────────────╯
    7 field(s)

  CLI: $ za-cli org info -P <passphrase>
```

```bash
$ za-cli -o json --envelope org info
{
  "envelopeVersion": 1,
  "data": {
    "planName": "Premium",
    "addOns": "Additional Users x10",
    "trialStatus": "",
    "billingDate": "2027-01-31",
    "planType": "PAID",
    "users": "15",
    "orgId": "20087654321"
  },
  "meta": { "timestamp": "2026-09-22T10:14:02.117Z", "correlationId": "…" },
  "error": null
}
```

Field names beyond `planName`, `addOns`, `trialStatus`, `billingDate`, `orgId` are whatever Zoho returns for your plan. Interactive equivalent: Org menu → **Subscription Details**.

---

## 2. `org users` — list organization users

```text
za-cli org users [--offset <N>] [--limit <N>] [--all] [common options]
```

```text
$ za-cli org users

  ╭──────────────────────────────────────────────────────────────╮
  │ ● Organization Users                                         │
  ├────────────────────────────┬──────────────────┬──────────────┤
  │ EMAIL ID                   │ ROLE             │ STATUS       │
  ├────────────────────────────┼──────────────────┼──────────────┤
  │ admin@example.com          │ Organization Admin│ Active       │
  │ alice@example.com          │ User             │ Active       │
  │ old.account@example.com    │ User             │ Deactivated  │   ← dimmed row, status highlighted
  ╰────────────────────────────┴──────────────────┴──────────────╯
    3 row(s)

  CLI: $ za-cli org users -P <passphrase>
```

Curated columns: `Email ID` (`emailId`/`email`), `Role` (`role`/`userRole`), `Status` (first of `userStatus`, `status`, `statusMessage`, `userState`, `state`). Inactive statuses (`disabled`, `deactivated`, `inactive`, `suspended`, `false`) are dimmed. Full payload with `-o json|yaml|ndjson`:

```bash
$ za-cli org users -o ndjson
{"emailId":"admin@example.com","role":"Organization Admin","status":"Active","firstName":"Ada","lastName":"Admin","zuid":"…"}
{"emailId":"alice@example.com","role":"User","status":"Active","firstName":"Alice","lastName":"Lee","zuid":"…"}
```

Useful variants:

```bash
za-cli org users --limit 50
za-cli org users --filter "role~admin" --fields emailId,role -o csv > admins.csv
za-cli org users --filter "status!=active"
```

Empty organization list: `  (no data)`.

---

## 3. `org users add <email>…` — add users to the organization

```text
za-cli org users add <emails>... [common options]
```

```text
$ za-cli org users add alice@example.com bob@example.com

  ╭────────────────────────────────────────╮
  │ ● Message                              │
  ├────────────────────────────────────────┤
  │ Added 2 user(s) to organization        │
  ╰────────────────────────────────────────╯

  CLI: $ za-cli org users add alice@example.com bob@example.com -P <passphrase>
```

`-o json` → `{"message": "Added 2 user(s) to organization"}`. Emails are space-separated positionals (one or more). Organization membership does **not** grant workspace access — follow with `workspace users add`. Zoho-side failures (invalid email, plan limit) surface as `[ZA1004] Operation failed` with the mapped message, e.g. `One or more email addresses are not in a valid format.` (8509) or `Cannot activate users: the user count would exceed your plan's allowed limit.` (6021).

---

## 4. `org users remove <email>…` — remove users from the organization

```text
za-cli org users remove <emails>... [common options]
```

```text
$ za-cli org users remove old.account@example.com
  ╭──────────────────────────────────────────────╮
  │ ● Message                                    │
  ├──────────────────────────────────────────────┤
  │ Removed 1 user(s) from organization          │
  ╰──────────────────────────────────────────────╯
```

Removes the person from **every** workspace at once — broader than `workspace users remove`. Not a destructive catalog tool, so no confirmation prompt is shown by default.

---

## 5. `org admins` — list organization administrators

```text
za-cli org admins [--offset <N>] [--limit <N>] [--all] [common options]
```

```text
$ za-cli org admins

  ╭───────────────────────────────╮
  │ ● Organization Admins         │
  ├───────────────────────────────┤
  │ EMAIL ID                      │
  ├───────────────────────────────┤
  │ admin@example.com             │
  │ cto@example.com               │
  ╰───────────────────────────────╯
    2 row(s)
```

Zoho may return admins as bare email strings or as objects with `emailId`; both render. `-o json` returns the raw array.

---

## 6. `org resources` — resource usage against plan limits

```text
za-cli org resources [common options]
```

```text
$ za-cli org resources

  ╭───────────────────────────────────────────────────────────────────────────────────╮
  │ ● Resource Usage                                                                  │
  ├──────────────────┬─────────────────────┬─────────────────────┬─────────────────────┬────────────────────┤
  │ RESOURCE         │ USED                │ ALLOCATED           │ REMAINING           │ REMARKS            │
  ├──────────────────┼─────────────────────┼─────────────────────┼─────────────────────┼────────────────────┤
  │ Users            │ 11                  │ 50                  │ 39                  │ —                  │
  │ Workspaces       │ 56                  │ Unlimited           │ Unlimited           │ —                  │
  │ Rows             │ 20361390            │ 310000000           │ 289638610           │ —                  │
  │ Rows Processed   │ 304270              │ 10000000            │ 9695730             │ daily limit        │
  │ Code Studio      │ 0                   │ 0                   │ 0                   │ Free Credits : 500 │
  ╰──────────────────┴─────────────────────┴─────────────────────┴─────────────────────┴────────────────────╯
    5 row(s)
```

Raw shape (`-o json`):

```json
[
  {"resourceName": "users", "resourceUsage": {"used": "11", "allocated": "50", "remaining": "39", "remarks": ""}},
  {"resourceName": "workspaces", "resourceUsage": {"used": "56", "allocated": "Unlimited", "remaining": "Unlimited", "remarks": ""}},
  {"resourceName": "rowsProcessed", "resourceUsage": {"used": "304270", "allocated": "10000000", "remaining": "9695730", "remarks": "daily limit"}}
]
```

Resource labels are humanised (`rowsProcessed` → `Rows Processed`, `roUsers` → `Read-only Users`, `queryTables` → `Query Tables`, `archivedrows` → `Archived Rows`). Run this before a large import to check headroom.

---

## 7. Failure samples common to the family

```text
$ za-cli org info                       # no login
  ⚠  [ZA1001] Not logged in
     Run 'za-cli login' to authenticate.
(exit 1)

$ za-cli org users add                  # missing positional
Missing required parameter: '<emails>'
Usage: za-cli org users add [-hVy] … <emails>...
(exit 2)

$ za-cli org users add not-an-email     # Zoho validation
  ✗  [ZA1004] Operation failed
     One or more email addresses are not in a valid format.
     Run with -v for the full stack trace.
(exit 1)
```
