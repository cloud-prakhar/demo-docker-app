# Demo App — Kibana & Index Setup Guide

> Elasticsearch stores the logs; Kibana is how you actually read them. Before Kibana can show anything, you must create a **Data View** pointing at the `flask-app-*` indices. This guide covers that plus the queries and dashboards worth building on top.

---

## Where to Run These Commands

All `curl` commands below use relative paths (`elk/certs/http_ca.crt`, `.env`), so run them from the **app dir**:

```bash
cd ~/git-repos/demo-docker-app/demo-app
```

Most examples read the password straight from `.env`, so there is nothing to paste:

```bash
PW=$(grep ELASTIC_PASSWORD .env | cut -d= -f2)
```

---

## Prerequisites

- ELK stack running (`es01` + `kib01`)
- The app stack running and shipping logs (`docker compose ps` shows `app`, `logstash`, `filebeat` all Up)
- At least one `flask-app-YYYY.MM.dd` index exists with documents

Verify there is data to look at:

```bash
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$PW" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
```

If `docs.count` is 0 (or the index does not exist), generate traffic first:

```bash
for i in $(seq 1 10); do
  curl -s http://localhost:5000/ > /dev/null
  curl -s http://localhost:5000/health > /dev/null
done
# wait ~15 seconds, then re-check the index
```

Confirm Kibana is ready to accept API calls — it takes noticeably longer to start than Elasticsearch:

```bash
curl -s http://localhost:5601/api/status | grep -o '"level":"available"'
```

If that returns nothing, Kibana is still initialising. Wait and retry.

---

## Understanding the Index Layout

Logstash writes to a **daily index**, set in `elk/logstash/pipeline/logstash.conf`:

```
index => "flask-app-%{+YYYY.MM.dd}"
```

So you accumulate one index per day:

```
flask-app-2026.08.07
flask-app-2026.08.08
flask-app-2026.08.09   ← today
```

**Why daily indices?** Deleting old data becomes a cheap index drop instead of an expensive delete-by-query, and searches limited to a recent time range only touch the relevant indices.

This is why every query and the Data View use the wildcard **`flask-app-*`** — it spans all days as a single searchable set.

---

## Step 1 — Create the Data View

A Data View (formerly "Index Pattern") tells Kibana which indices to query and which field holds the event time.

### Option A — via the API (fast, scriptable)

```bash
curl -s -u "elastic:$PW" -X POST "http://localhost:5601/api/data_views/data_view" \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
  -d '{"data_view":{"title":"flask-app-*","name":"Flask App Logs","timeFieldName":"@timestamp"}}'
```

A successful response contains the new view's `id`, `name`, and `title`.

> `kbn-xsrf: true` is required on every Kibana `POST`/`PUT`/`DELETE`. Without it Kibana rejects the request with a 400.
>
> Note this call uses **`http://`** on port 5601 — Kibana here is served over plain HTTP, unlike Elasticsearch on 9200 which is HTTPS.

Confirm it exists:

```bash
curl -s -u "elastic:$PW" http://localhost:5601/api/data_views -H 'kbn-xsrf: true'
```

### Option B — via the UI

1. Open Kibana: http://localhost:5601
2. Log in as `elastic` with the password from `.env`
3. Hamburger menu (top-left) → **Stack Management**
4. Under **Kibana** → **Data Views**
5. Click **Create data view**
6. Fill in:
   - **Name:** `Flask App Logs`
   - **Index pattern:** `flask-app-*`
   - **Timestamp field:** `@timestamp`
7. Click **Save data view to Kibana**

> **Pick `@timestamp`, not `timestamp`.** Both exist on these documents and the distinction matters. `@timestamp` is the real event time as a proper date type — this is the one Kibana's time picker and every date histogram rely on. `timestamp` is the app's own `asctime` string promoted by the Logstash pipeline; it is useful to read but is not the time field.

---

## Step 2 — Explore Logs in Discover

1. Hamburger menu → **Discover**
2. Select **Flask App Logs** in the data view dropdown (top-left)
3. Set the time range (top-right) to **Last 15 minutes**

> Seeing "No results match your search criteria" almost always means the **time range**, not a broken pipeline. These are live logs — if the app has been idle, widen the range or generate traffic.

**Recommended columns** — click the `+` next to each field in the left panel:

`level`, `log_message`, `endpoint`, `http_method`, `client_ip`, `logger`

Click the `>` arrow on any row to expand the full document.

---

## Step 3 — Field Reference

These are the fields the Logstash pipeline produces. The renames happen in `elk/logstash/pipeline/logstash.conf`.

| Field | Source | Description | Example |
|---|---|---|---|
| `@timestamp` | Filebeat/Logstash | **The event time field.** Use this for all time-based charts | `2026-08-09T05:11:56.000Z` |
| `service` | Added by Logstash | Constant tag identifying this app | `flask-demo-app` |
| `level` | `levelname` | Log level | `INFO`, `WARNING`, `ERROR` |
| `log_message` | `message` | The human-readable message | `Health check` |
| `logger` | `name` | Which Python logger emitted it | `app`, `werkzeug` |
| `timestamp` | `asctime` | App's own formatted time (string, not a date type) | `2026-08-09T05:11:56` |
| `endpoint` | `endpoint` | URL path, on route-handler logs | `/health` |
| `http_method` | `method` | HTTP verb, on `before_request` logs | `GET` |
| `client_ip` | `remote_addr` | Caller IP, on `before_request` logs | `172.20.0.1` |
| `response_status` | `status` | Status value from the route handler | `ok` |
| `container.name` | `add_docker_metadata` | Source container — should always be `demo-app-app-1` | `demo-app-app-1` |

**Two loggers, two shapes of event.** The app emits two events per request:

- `logger: app` with the `before_request` hook → carries `http_method` and `client_ip`
- `logger: app` from the route handler → carries `endpoint` and `response_status`
- `logger: werkzeug` → Flask's built-in access log line; has `level` and `log_message` but none of the custom fields

So **not every document has every field**. A query on `client_ip` legitimately matches only a subset. This is expected, not a parsing failure.

> **`service` is present but `level` and `endpoint` are missing on every document?** That is a real misconfiguration — Filebeat decoded the JSON before Logstash could. See the troubleshooting section in [elk-configuration.md](./elk-configuration.md).

---

## Step 4 — KQL Searches

Type these into the Discover search bar.

**By log level:**
```kql
level: "ERROR"
level: ("ERROR" or "WARNING")
not level: "INFO"
```

**By endpoint:**
```kql
endpoint: "/health"
endpoint: "/"
```

**Only this app's own logs, excluding Flask's access log noise:**
```kql
logger: "app"
```

**Only the request-entry events (those carrying client IP and method):**
```kql
http_method: "GET" and logger: "app"
```

**Requests from a specific client:**
```kql
client_ip: "172.20.0.1"
```

**Sanity check that collection is scoped correctly** — this should return *everything*; any other container appearing means Filebeat is over-collecting:
```kql
container.name: "demo-app-app-1"
```

**Free-text search across the message:**
```kql
log_message: *health*
```

---

## Step 5 — Build Visualizations

Hamburger menu → **Visualize Library** → **Create visualization** → **Lens**.

### 1 — Log Level Distribution (Donut)
- **Slice by:** `level.keyword` (Terms)
- **Size by:** `Count`
- **Title:** `Log Levels`

### 2 — Requests Over Time (Bar vertical stacked)
- **Horizontal axis:** `@timestamp` (Date histogram, interval Auto)
- **Vertical axis:** `Count`
- **Break down by:** `level.keyword`
- **Title:** `Requests Over Time`

### 3 — Top Endpoints (Bar horizontal)
- **Vertical axis:** `endpoint.keyword` (Terms, top 10)
- **Horizontal axis:** `Count`
- **Title:** `Top Endpoints`

> Only route-handler events carry `endpoint`, so this chart intentionally covers a subset of documents.

### 4 — Top Clients (Bar horizontal)
- **Vertical axis:** `client_ip.keyword` (Terms, top 10)
- **Horizontal axis:** `Count`
- **Title:** `Top Clients`

### 5 — Error Rate (Line)
- **Horizontal axis:** `@timestamp` (Date histogram, 1 minute)
- **Vertical axis:** `Count`
- Add a visualization-level filter: `level: "ERROR"`
- **Title:** `Error Rate`

> `.keyword` matters. Text fields are analysed and cannot be aggregated; the `.keyword` sub-field is the exact-value version Terms aggregations need. Use `level.keyword`, not `level`.

---

## Step 6 — Create a Dashboard

1. Hamburger menu → **Dashboards** → **Create dashboard**
2. **Add from library** → add the visualizations above
3. Suggested layout:
   ```
   [ Log Levels (donut) ]   [ Error Rate (line)  ]
   [ Requests Over Time (bar) — full width       ]
   [ Top Endpoints (bar) ]  [ Top Clients (bar)  ]
   ```
4. Set the time range to **Last 15 minutes**
5. Save as `Flask Demo App Overview`

---

## Step 7 — Dev Tools Queries

Hamburger menu → **Dev Tools**. These run against Elasticsearch directly.

**Count all documents:**
```json
GET /flask-app-*/_count
```

**10 most recent events:**
```json
GET /flask-app-*/_search
{
  "size": 10,
  "sort": [{"@timestamp": {"order": "desc"}}]
}
```

**Breakdown by log level:**
```json
GET /flask-app-*/_search
{
  "size": 0,
  "aggs": { "by_level": { "terms": {"field": "level.keyword"} } }
}
```

**Breakdown by endpoint:**
```json
GET /flask-app-*/_search
{
  "size": 0,
  "aggs": { "by_endpoint": { "terms": {"field": "endpoint.keyword", "size": 10} } }
}
```

**Which containers are in the index** — health check for Filebeat scoping. Should return only `demo-app-app-1`:
```json
GET /flask-app-*/_search
{
  "size": 0,
  "query": { "range": { "@timestamp": { "gte": "now-10m" } } },
  "aggs": { "by_container": { "terms": {"field": "container.name.keyword", "size": 20} } }
}
```

> The `range` filter is deliberate. Without it you aggregate over all history, so any noise collected before a past config fix still appears and makes a healthy pipeline look broken. Old documents are never rewritten.

**All ERROR events, newest first:**
```json
GET /flask-app-*/_search
{
  "query": { "term": {"level.keyword": "ERROR"} },
  "sort": [{"@timestamp": {"order": "desc"}}],
  "size": 20
}
```

**Inspect the field mapping:**
```json
GET /flask-app-*/_mapping
```

---

## Generating Demo Traffic

The app has two endpoints, so vary the mix:

```bash
# Normal traffic
for i in $(seq 1 10); do
  curl -s http://localhost:5000/ > /dev/null
  curl -s http://localhost:5000/health > /dev/null
done

# 404s — produces werkzeug warnings
curl -s http://localhost:5000/does-not-exist > /dev/null
```

Wait ~15 seconds, then refresh Discover.

> The demo app only emits `INFO` level logs in normal operation, so `level: "ERROR"` charts will be empty on a healthy run. To demo error-level logging, add a route that calls `app.logger.error(...)` — see "How to Add a New Route" in [app-configuration.md](./app-configuration.md) — or use the `notes-app`, which ships purpose-built `/demo/error` and `/demo/warning` endpoints.

---

## Managing Data Views

**List all data views:**
```bash
curl -s -u "elastic:$PW" http://localhost:5601/api/data_views -H 'kbn-xsrf: true'
```

**Refresh the field list** after the pipeline starts producing a new field — Kibana caches the field list, so new fields may not appear in Discover until the view is refreshed. In the UI: **Stack Management → Data Views → Flask App Logs → the refresh icon**.

**Delete a data view** (does not touch the underlying indices):
```bash
curl -s -u "elastic:$PW" -X DELETE \
  "http://localhost:5601/api/data_views/data_view/<data-view-id>" \
  -H 'kbn-xsrf: true'
```

> Deleting the data view only removes Kibana's *pointer* to the indices. To delete the log data itself, see [teardown.md](./teardown.md).

---

## Troubleshooting

### Kibana API returns `Unauthorized`
Use the `elastic` user and the password from `.env`. Note Kibana's own internal connection to Elasticsearch uses the separate `kibana_system` account — that is unrelated to how you log in.

### `Kibana server is not ready yet`
Kibana starts considerably slower than Elasticsearch, especially right after `es01` starts. Poll until ready:
```bash
curl -s http://localhost:5601/api/status | grep -o '"level":"[a-z]*"'
```

### Data view created, but Discover shows no fields
The index has no documents yet — Kibana derives the field list from the mapping. Generate traffic, then refresh the data view's field list.

### Charts show data from other applications

Documents from `grafana`, `prometheus`, `es01`, `kib01`, or `demo-app-logstash-1` are in the index. Determine whether this is **live** or **historical** before changing anything — run the container breakdown from Step 7 with the `now-10m` range filter:

- **Extra containers appear in the last 10 minutes** → Filebeat is currently over-collecting. See the "unrelated containers" entry in [elk-configuration.md](./elk-configuration.md).
- **Only `demo-app-app-1` appears recently, but wide time ranges still show others** → leftover data from before the config was fixed. The pipeline is healthy; the old documents simply remain.

For the historical case, either keep your dashboards on a recent time range, add `container.name: "demo-app-app-1"` as a filter, or delete the old documents using the cleanup snippet in [elk-configuration.md](./elk-configuration.md).

> A blunt alternative, if the log history has no value to you, is to drop the affected indices entirely and let them rebuild — see [teardown.md](./teardown.md). This deletes **all** logs for those days, not just the noise.

---

*For the log pipeline itself, see [elk-configuration.md](./elk-configuration.md)*
*For the Flask app and its logging, see [app-configuration.md](./app-configuration.md)*
*For cleanup, see [teardown.md](./teardown.md)*
