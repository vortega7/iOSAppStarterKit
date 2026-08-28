# New Relic CLI automation (NerdGraph)

Querying New Relic (NRQL) and building dashboards directly from the
command line via New Relic's GraphQL API (NerdGraph) — no need to hand
every question to the user to run in the New Relic UI. Every command
below was actually run successfully against a real production app's
account; every gotcha noted was hit for real.

## Prerequisites

1. **A New Relic User API key**, not an Ingest key — generate one at
   **Account settings → API keys** in the New Relic UI. Starts with
   `NRAK-`.
2. **The account ID** — a plain integer, shown in the same Account
   settings area.

## Step 1 — store credentials outside the repo, never in it

Same reasoning as `docs/app-store-connect-cli-automation.md`'s private
key: a dotfile under the user's home directory, mode `600`, never
committed:

```bash
mkdir -p ~/.newrelic && chmod 700 ~/.newrelic
cat > ~/.newrelic/credentials.json <<'EOF'
{
  "user_api_key": "[FILL IN: your NRAK-... User API key]",
  "account_id": "[FILL IN: your account ID, a plain integer]"
}
EOF
chmod 600 ~/.newrelic/credentials.json
```

## Step 2 — a reusable query script

Reads credentials from the file above rather than hardcoding them, so
the script itself is safe to read, share, or commit:

```python
#!/usr/bin/env python3
"""Run an NRQL query via NerdGraph. Usage: nrql.py "<NRQL query>" """
import json, subprocess, sys, os

CREDENTIALS_PATH = os.path.expanduser("~/.newrelic/credentials.json")

def main():
    nrql = sys.argv[1]
    with open(CREDENTIALS_PATH) as f:
        creds = json.load(f)
    graphql_query = (
        "{ actor { account(id: %s) { nrql(query: %s) { results } } } }"
        % (creds["account_id"], json.dumps(nrql))
    )
    body = json.dumps({"query": graphql_query})
    proc = subprocess.run(
        ["curl", "-s", "-H", f"API-Key: {creds['user_api_key']}",
         "-H", "Content-Type: application/json", "-d", body,
         "https://api.newrelic.com/graphql"],
        capture_output=True,
    )
    response = json.loads(proc.stdout)
    print(json.dumps(response["data"]["actor"]["account"]["nrql"]["results"], indent=2))

if __name__ == "__main__":
    main()
```

```bash
chmod 700 ~/.newrelic/nrql.py
python3 ~/.newrelic/nrql.py "FROM MobileRequest SELECT count(*) SINCE 1 day ago"
```

**Gotcha hit for real**: `SELECT` with raw event fields (not just
aggregate functions) cannot be combined with `FACET` in one NRQL query —
```
FROM MobileRequest SELECT uuid, city FACET uuid ...
```
fails with `FACET cannot be used when selecting on event fields`
(`NRDB:1102004`). Either drop `FACET` and get raw rows (dedupe/group
client-side, e.g. in Python), or drop the raw field selection and use
only aggregate functions (`count(*)`, `earliest(timestamp)`,
`latest(city)`, etc.) alongside `FACET`.

## Useful investigation patterns

- **Discover what attributes an event type actually carries**, before
  guessing field names: `FROM SomeEventType SELECT keyset() SINCE 90
  days ago`. Mobile agent events (`MobileRequest`, custom events sent
  via `NewRelic.recordCustomEvent`) both carry rich auto-attached
  attributes worth knowing about: `uuid` (device ID), `city`/
  `countryCode`/`regionCode`/`asnOwner` (GeoIP), `appBuild`/
  `appVersion`, `sessionId`, `deviceModel`, `install` (true on a
  session's very first event after a fresh install).
- **Cross-reference custom events with `MobileRequest`** (New Relic's
  own auto-collected network-call event) for the same `uuid` to
  reconstruct what actually happened in a session, beyond what your own
  custom events capture — e.g. correlating a "recording failed" custom
  event against the actual HTTP status/timing of the underlying API
  call, or confirming a usage-metering flow's pipeline stages (a
  speech-to-text call followed by translate/synthesize calls) actually
  completed rather than silently stopping partway.
- **Estimate a request's payload size/duration from `bytesSent` alone**
  when `HttpResponseBodyCapture`/full body capture is intentionally
  disabled (see `new-relic-analytics-setup.md`'s data-volume notes):
  for a JSON body wrapping base64-encoded raw audio, subtract the fixed
  JSON-structure overhead (measure it directly — build the same
  skeleton object with an empty payload field and check
  `len(json.dumps(...))`), treat the remainder as the base64 string,
  multiply by `3/4` for raw bytes, then divide by the raw format's
  bytes/second (e.g. 16kHz × 2 bytes/sample mono PCM16 = 32,000
  bytes/sec) to get an approximate duration. Useful for reconstructing
  "how long did this recording actually run" without ever capturing the
  audio itself.
- **A device's identifier changes on full uninstall + reinstall** if
  the app is the only one installed from that vendor —
  `identifierForVendor`-derived IDs (which New Relic's mobile agent
  `uuid` is typically based on) reset in that case. If you're
  investigating "the same physical device" across a reinstall, filter
  by network/device characteristics instead (`asnOwner`, `city`,
  `deviceModel`) to find the new `uuid`, rather than assuming it stayed
  the same.

## Creating a dashboard via `dashboardCreate`

```bash
python3 - <<'PYEOF'
import json, subprocess, os

with open(os.path.expanduser("~/.newrelic/credentials.json")) as f:
    creds = json.load(f)
account_id = int(creds["account_id"])

dashboard = {
    "name": "My App – Technical Health",
    "permissions": "PUBLIC_READ_WRITE",
    "pages": [{
        "name": "Overview",
        "widgets": [{
            "title": "Total HTTP Requests (7d)",
            "layout": {"column": 1, "row": 1, "width": 3, "height": 3},
            "visualization": {"id": "viz.billboard"},
            "rawConfiguration": {
                "nrqlQueries": [{"accountId": account_id,
                    "query": "FROM MobileRequest SELECT count(*) SINCE 7 days ago"}]
            }
        }]
        # more widgets: visualization ids include viz.billboard, viz.line,
        # viz.area, viz.bar, viz.table, viz.pie. layout is a 12-column
        # grid; row/column/width/height are all integer grid units.
    }]
}

payload = {
    "query": "mutation($accountId: Int!, $dashboard: DashboardInput!) { dashboardCreate(accountId: $accountId, dashboard: $dashboard) { entityResult { guid name } errors { description type } } }",
    "variables": {"accountId": account_id, "dashboard": dashboard}
}
result = subprocess.run(
    ["curl", "-s", "-H", f"API-Key: {creds['user_api_key']}",
     "-H", "Content-Type: application/json", "-d", json.dumps(payload),
     "https://api.newrelic.com/graphql"],
    capture_output=True,
)
print(result.stdout.decode())
PYEOF
```

A successful response has `"errors": null` and an `entityResult.guid` —
the dashboard URL is `https://one.newrelic.com/redirect/entity/<guid>`.

## Updating specific widgets on an existing dashboard

Fetch the page's `guid` and each widget's `id` first (needed to update
in place rather than creating a duplicate dashboard):

```graphql
{ actor { entity(guid: "<DASHBOARD_GUID>") { ... on DashboardEntity {
  pages { guid name widgets { id title layout { column row width height } visualization { id } rawConfiguration } }
} } } }
```

Then call `dashboardUpdateWidgetsInPage(guid: <PAGE_guid>, widgets:
[...])`, passing the **complete** widget list for that page (it
replaces all widgets on the page, not just the ones you name) — include
every existing widget's `id`/title/layout/visualization/rawConfiguration
unchanged, and only modify the one(s) you actually want to change.

**Gotcha hit for real**: the mutation's input type for *updating* is
`DashboardUpdateWidgetInput`, not `DashboardWidgetInput` (the type used
by `dashboardCreate`) — using the create-mutation's type name in the
update mutation's signature fails with a GraphQL type-mismatch error
(`Variable of type [DashboardWidgetInput!]! found as input to argument
of type [DashboardUpdateWidgetInput!]!`). The two mutations otherwise
take an identically-shaped widget object.
