# Mining Data from Enterprise Splunk

## 1. The splunkd REST API (Port 8089) — The Foundation

Every Splunk Enterprise instance runs **splunkd**, which exposes a REST API on **port 8089** (HTTPS). This is available on all versions and is the primary programmatic interface. No extra setup is needed — it's always running.

### Authentication Options
- **Username/Password** — returns a session key (expires)
- **Bearer Tokens** (Splunk 7.3+) — long-lived API tokens, no login dance required. Set via `Authorization: Bearer <token>` header. Your Splunk admin enables this in `server.conf` under `[tokens]`.

### Key Endpoints for Pulling Data

**a) Run an ad-hoc search and get results:**
```bash
# One-shot export (streams results, no job created)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://splunk-host:8089/servicesNS/admin/search/search/jobs/export \
  -d search="search index=main earliest=-1h | stats count by host" \
  -d output_mode=json
```

**b) Run a saved search / report by name:**
```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  https://splunk-host:8089/servicesNS/admin/search/search/jobs/export \
  -d search="| savedsearch my_saved_search_name" \
  -d output_mode=json
```

**c) Create a search job (async, for large queries):**
```bash
# Create the job
curl -k -H "Authorization: Bearer $TOKEN" \
  https://splunk-host:8089/servicesNS/admin/search/search/jobs \
  -d search="search index=main | stats count by source" \
  -d output_mode=json

# Poll for results (use the sid returned above)
curl -k -H "Authorization: Bearer $TOKEN" \
  "https://splunk-host:8089/servicesNS/admin/search/search/jobs/<SID>/results?output_mode=json&count=0"
```

Output formats: `json`, `csv`, `xml`, `json_rows`

---

## 2. Splunk SDKs — Wrapper Libraries

Splunk provides official SDKs that wrap the REST API:

| SDK | Status | Notes |
|-----|--------|-------|
| **Python SDK** | Actively maintained | Best supported, most examples |
| **JavaScript SDK** | Maintained | Node.js; browser export has limitations |
| **Java SDK** | Maintained | Enterprise-oriented |

- [Python SDK (GitHub)](https://github.com/splunk/splunk-sdk-python)
- [JavaScript SDK (GitHub)](https://github.com/splunk/splunk-sdk-javascript)

### Python SDK Example
```python
import splunklib.client as client
import splunklib.results as results

service = client.connect(
    host='splunk-host', port=8089,
    token='your-bearer-token'  # or username/password
)

# Stream results (no job created — most efficient)
rr = results.JSONResultsReader(
    service.jobs.export(
        "search index=main earliest=-24h | stats count by host",
        output_mode='json'
    )
)

for result in rr:
    if isinstance(result, dict):
        print(result)
```

---

## 3. Pulling Dashboard Data

### Option A: Extract the SPL from Dashboards, Run via API

Dashboards are stored as XML/JSON definitions. You can **retrieve the dashboard definition** via REST, extract the search queries, and run them independently:

```bash
# Get a dashboard's definition
curl -k -H "Authorization: Bearer $TOKEN" \
  "https://splunk-host:8089/servicesNS/nobody/search/data/ui/views/my_dashboard?output_mode=json"
```

The response contains the `<search>` elements (Classic dashboards) or `ds.search` data source definitions (Dashboard Studio). Parse out the SPL queries and run them through the search jobs API above.

### Option B: Saved Searches / Reports as the Data Layer (Recommended)

The most robust pattern for an internal app:

1. **Create Saved Searches (Reports)** in Splunk that mirror what your dashboards show
2. Set them to run on a schedule (e.g., every 15 minutes) with **acceleration** enabled
3. Pull results via the REST API: `| savedsearch "my_report_name"`
4. The results are pre-computed and return instantly

This is the **recommended approach** — it decouples your app from dashboard internals.

### Option C: Dashboard Studio REST API (Splunk 9.x+)

Dashboard Studio dashboards (version 2 XML with `<dashboard version="2">`) can be managed via REST endpoints. However, this is for managing the dashboard definitions themselves, not for directly streaming the rendered data.

- [Dashboard Studio REST API Usage](https://docs.splunk.com/Documentation/Splunk/latest/DashStudio/RESTusage)

### What About "Automatic API Server per Dashboard"?

What you may have heard about is the **Splunk Dashboard Framework** and its evolution in newer versions. In Dashboard Studio (Splunk 9.x and especially 10.x), dashboards use a **data source model** (`ds.search`, `ds.chain`, etc.) that makes them more API-like internally. However, there is **no feature where Splunk auto-exposes a standalone REST endpoint per dashboard** that external apps can hit directly. The "API-like" behavior is internal to the Splunk web UI rendering engine. External apps still need to go through splunkd's search API.

---

## 4. KV Store — For Structured Data

If your use case involves structured lookup/reference data, the KV Store REST API provides CRUD operations:

```bash
# Read from a KV Store collection
curl -k -H "Authorization: Bearer $TOKEN" \
  "https://splunk-host:8089/servicesNS/nobody/myapp/storage/collections/data/my_collection?output_mode=json"
```

- [KV Store REST Endpoints](https://docs.splunk.com/Documentation/Splunk/9.4.2/RESTREF/RESTkvstore)

---

## 5. Recommended Architecture for an Internal App

```
┌─────────────────┐     ┌──────────────────────┐     ┌──────────────┐
│  Your Internal  │────>│  Splunk splunkd API   │────>│   Splunk     │
│  Application    │     │  (port 8089, HTTPS)   │     │   Indexes    │
└─────────────────┘     └──────────────────────┘     └──────────────┘
        │
        │  Uses:
        │  - Bearer token auth
        │  - /search/jobs/export (ad-hoc)
        │  - | savedsearch (pre-built reports)
        │  - output_mode=json
```

**Steps:**
1. **Get a service account** with a bearer token (ask your Splunk admin to enable token auth and create one scoped to the searches you need)
2. **Build your Splunk queries** and save them as Reports/Saved Searches in Splunk
3. **Use the Python SDK or direct REST calls** from your app to execute `| savedsearch "report_name"` via the export endpoint
4. **Cache results** on your app side to avoid hammering Splunk
5. For dashboards, maintain the Splunk dashboards for Splunk users, but have your app pull the **same underlying saved searches** rather than trying to scrape dashboard output

### Version Considerations

| Feature | Minimum Version |
|---------|----------------|
| REST API (port 8089) | All versions |
| Bearer token auth | 7.3+ |
| Dashboard Studio | 9.0+ |
| Dashboard version history | 9.4+ |
| Python SDK current | Works with 7.x+ |

---

## Sources

- [Splunk REST API Reference](https://dev.splunk.com/enterprise/reference/)
- [Export Data Using the REST API](https://docs.splunk.com/Documentation/Splunk/latest/Search/ExportdatausingRESTAPI)
- [Creating Searches Using the REST API](https://docs.splunk.com/Documentation/Splunk/latest/RESTTUT/RESTsearches)
- [Export Data Using the Splunk SDKs](https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.0/export-search-results/export-data-using-the-splunk-sdks)
- [Dashboard Studio REST API Usage](https://docs.splunk.com/Documentation/Splunk/latest/DashStudio/RESTusage)
- [KV Store REST Endpoints](https://docs.splunk.com/Documentation/Splunk/9.4.2/RESTREF/RESTkvstore)
- [Splunk Python SDK (GitHub)](https://github.com/splunk/splunk-sdk-python)
- [Use Authentication Tokens](https://help.splunk.com/en/splunk-cloud-platform/administer/manage-users-and-security/9.3.2408/authenticate-into-the-splunk-platform-with-tokens/use-authentication-tokens)
- [Splunk as a REST API (Medium)](https://medium.com/@ravelantunes/splunk-as-a-rest-api-98dd1b85a137)
- [Getting Data Out of Splunk](https://discoveredintelligence.com/get-data-out-of-splunk/)
