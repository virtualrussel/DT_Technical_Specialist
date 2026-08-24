# Dynatrace API v2 Quick Reference
 
> **Last verified against docs.dynatrace.com:** 2026-08-24
> **Staleness policy:** Authentication mechanisms and base URLs are actively evolving as Dynatrace migrates capabilities from Classic APIs to the platform (Grail-based) APIs. Confirm which generation an environment is running and which auth model an endpoint expects before handing a customer a curl example. Using the wrong header/token type is the most common integration failure.
 
Fast lookup for common API endpoints, authentication, scopes, request/response formats, and error codes, covering both Classic APIs (Metrics, Logs, Events, Entities, Problems) and Platform APIs (Grail Query API, Business Events).
 
---
 
## Two API Generations: Classic vs. Platform
 
**READ THIS SECTION FIRST.** This is the #1 integration mistake: using the wrong token type, header, or base URL for an endpoint. Wrong auth returns 401/403 with no error message saying "wrong token type," so the entire integration silently fails.
 
Dynatrace environments expose two API surfaces side by side. Endpoints, base URLs, and auth mechanisms differ between them; mixing them up is the most common integration failure.
 
| | **Classic APIs** | **Platform APIs** |
|---|---|---|
| **Base URL** | `https://{environment-id}.live.dynatrace.com` | `https://{environment-id}.apps.dynatrace.com` |
| **Covers** | Metrics v2, Logs v2, Events v2, Entities v2, Problems, Business Events ingest, Audit Logs | Grail Query API (DQL), Workflows/Automation, Documents (dashboards), IAM |
| **Primary auth** | API token (`Api-Token` header) | OAuth client or platform token (`Bearer` header) |
| **Can also be reached via** | Same classic endpoints are also proxied at `https://{environment-id}.apps.dynatrace.com/platform/classic/environment-api/v2/...` using a platform token or OAuth bearer token | N/A |
 
**Rule of thumb:** If you're ingesting/querying classic telemetry (metrics, logs, events, entities, problems) and already have an API token, use the classic base URL. It still works and isn't deprecated. If you need to run DQL programmatically, automate workflows, or manage dashboards/IAM via API, you're in platform territory and need a platform token or OAuth client, not an API token.
 
### Decision Tree: Which Auth Do I Use?
 
```
Am I calling a classic endpoint (Metrics, Logs, Events, Entities, Problems)?
│
├─ YES (e.g., /api/v2/metrics/ingest, /api/v2/logs, /api/v2/entities) →
│  Use: "Authorization: Api-Token {token}" + classic API token
│  Base URL: https://{environment-id}.live.dynatrace.com/api/v2/...
│
├─ NO. I'm calling a platform endpoint (Grail Query, Workflows, Documents) →
│  Use: "Authorization: Bearer {token}" + platform token OR OAuth access_token
│  Base URL: https://{environment-id}.apps.dynatrace.com/platform/...
│
└─ UNSURE. The endpoint doesn't clearly say which generation →
   Check this document for the endpoint name (search for it below)
   If still unclear, assume you need classic auth first
   → If you get 401/403, the wrong token type is the first thing to check
```
 
---
 
## Authentication Basics
 
Dynatrace has three distinct credential types. They are not interchangeable; using the wrong one against an endpoint returns a 401/403, not a helpful error explaining which type was expected.
 
### 1. API Tokens (Classic): `dt0c01...`
- **Header:** `Authorization: Api-Token {token}`
- **Use for:** Classic endpoints (Metrics, Logs, Events, Entities, Problems, Business Events, Audit Logs) at `.live.dynatrace.com`
- **Create at:** Settings > Access Tokens (in-environment)
- **Scopes:** Fine-grained per endpoint (e.g., `metrics.read`, `logs.ingest`)
### 2. Platform Tokens: `dt0s01...` / `dt0s16...`
- **Header:** `Authorization: Bearer {token}`
- **Use for:** Direct calls to platform services (Grail Query API, Automation, Documents) *and* classic endpoints proxied through `.apps.dynatrace.com/platform/classic/...`
- **Create at:** Account Management > `myaccount.dynatrace.com/platformTokens` (account-level, not environment-level)
- **Key constraint:** A platform token only works within the bounds of the assigned user's own permissions. Granting a scope on the token doesn't grant access the user doesn't already have.
- **Limit:** Up to 10 platform tokens per user per account.
- **Best for:** Scheduled scripts, ETL jobs, ad-hoc API calls tied to a specific person's access.
### 3. OAuth Clients (Client Credentials Flow): machine-to-machine
- **Use for:** Automation/integrations where no human user is involved (CI/CD, Terraform, Monaco, EdgeConnect, scheduled service-to-service calls)
- **Create at:** Account Management > Identity & Access Management > OAuth clients
- **Token request:**
```bash
curl --location --request POST 'https://sso.dynatrace.com/sso/oauth2/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id={your-client-id}' \
  --data-urlencode 'client_secret={your-client-secret}' \
  --data-urlencode 'resource=urn:dtaccount:{your-account-uuid}' \
  --data-urlencode 'scope=storage:events:read storage:metrics:read'
```
- **Response contains** an `access_token`; use it as a Bearer token for the actual API call:
```bash
curl -X GET "https://{environment-id}.apps.dynatrace.com/platform/..." \
  -H "Authorization: Bearer {access_token}"
```
- **Note:** An OAuth client cannot have wider permissions than the user who created it. For client-credentials clients, a "subject user" (service user or account-user-management-permitted user) is required.
### Common Token Scopes
| Scope | Purpose | Endpoints |
|-------|---------|-----------|
| `metrics.ingest` | Ingest custom metrics | `/api/v2/metrics/ingest` |
| `metrics.read` | Query metrics | `/api/v2/metrics/query` (GET) |
| `logs.ingest` | Ingest logs | `/api/v2/logs/ingest` |
| `storage:logs:read` | Query logs via DQL | `fetch logs` in Notebooks / Query API |
| `storage:metrics:read` | Query metrics via DQL | `fetch metrics` in Notebooks / Query API |
| `storage:events:read` | Query Davis AI events via DQL | `fetch dt.davis.events.snapshots` in Notebooks / Query API |
| `storage:bizevents:read` | Query business events via DQL | `fetch bizevents` in Notebooks / Query API |
| `storage:spans:read` | Query spans/traces via DQL | `fetch spans` in Notebooks / Query API |
| `openTelemetryTrace.ingest` | Ingest traces via OTLP | `/api/v2/otlp/v1/traces` |
| `events.ingest` | Ingest Davis/custom events | `/api/v2/events/ingest` |
| `storage:entities:read` | Query monitored entities | `/api/v2/entities`, `fetch dt.entity.*` |
| `problems.read` | Read problems | `/api/v2/problems` |
| `auditlogs.read` | Read audit logs | `/api/v2/auditlogs` |
| `bizevents.ingest` | Ingest business events | `/api/v2/bizevents/ingest` |
| `automation:workflows:read` / `:write` / `:admin` | Manage Workflows | Automation API |
| `openpipeline.events` | Ingest generic events (OpenPipeline built-in) | `/platform/ingest/v1/events` |
| `openpipeline.events.custom` | Ingest generic events (OpenPipeline custom endpoint) | `/platform/ingest/custom/events` |
| `openpipeline.sdlc` | Ingest SDLC events (OpenPipeline built-in) | `/platform/ingest/v1/events.sdlc` |
| `openpipeline.sdlc.custom` | Ingest SDLC events (OpenPipeline custom endpoint) | `/platform/ingest/custom/events.sdlc` |
| `openpipeline.events_security` | Ingest security events (OpenPipeline built-in) | `/platform/ingest/v1/security.events` |
| `openpipeline.events_security.custom` | Ingest security events (OpenPipeline custom endpoint) | `/platform/ingest/custom/security.events` |
| `openpipeline.events_smartscape` | Ingest Smartscape events | `/platform/ingest/v1/smartscape.events` |
 
**Best practice:** Create a token/client with only scopes needed for the specific use case (principle of least privilege).
 
---
 
## Grail Query API (Platform / DQL via API)
 
This is how you run DQL programmatically outside of Notebooks/Dashboards, the primary way integrations pull Grail data (logs, metrics, spans, events, entities) via API in the current platform.
 
**Auth:** OAuth bearer token (client credentials flow) or platform token. Classic API tokens do **not** work here.
 
**Two-step async pattern:** execute, then poll for results:
 
**Step 1: Submit the query:**
```bash
curl -X POST "https://{environment-id}.apps.dynatrace.com/platform/storage/query/v1/query:execute" \
  -H "Authorization: Bearer {access_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "fetch logs | filter loglevel == \"ERROR\" | summarize count()",
    "defaultTimeframeStart": "2026-07-14T00:00:00Z",
    "defaultTimeframeEnd": "2026-07-15T00:00:00Z"
  }'
```
Response includes a `requestToken`.
 
**Step 2: Poll for the result:**
```bash
curl -X GET "https://{environment-id}.apps.dynatrace.com/platform/storage/query/v1/query:poll?request-token={requestToken}" \
  -H "Authorization: Bearer {access_token}"
```
Poll repeatedly until the state is no longer `RUNNING`.
 
**Required scopes:** Match the data type being queried: `storage:logs:read`, `storage:metrics:read`, `storage:events:read`, `storage:spans:read`, `storage:entities:read`, etc.
 
**Notes:**
- Default timeframe if unspecified: 2 hours (matches Notebooks default).
- This is the correct API for any "pull Grail data programmatically" integration request. Don't reach for classic Metrics/Logs GET endpoints when the ask is DQL-shaped.
- This section covers auth and the execute/poll transport only. For DQL syntax, Smartscape topology queries, or why a query itself is slow, see `dynatrace_dql_reference.md`, not this file.
---
 
## Metrics API v2
 
### Ingest Custom Metrics
**POST** `/api/v2/metrics/ingest`
 
**Content-Type:** `text/plain; charset=utf-8`
 
**Scope:** `metrics.ingest`
 
**Format:** Dynatrace Metrics Ingestion Protocol (line-based plaintext)
```
metric_key{dimension_key=dimension_value} metric_value timestamp_ms
```
 
**Examples:**
```bash
# Simple metric
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.orders 150 $(date +%s)000"
 
# With dimensions
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.orders {region=us,tier=gold} 150"
 
# Multiple metrics
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.orders 150
custom:myapp.latency_ms 45.2
custom:myapp.errors {code=503} 2"
```
 
**Response:** `204 No Content` on success
 
**Notes:**
- Metric ID auto-created; no pre-registration needed
- Dimensions must be alphanumeric + underscore
- Reserved dimension: `dt.entity.*` (auto-added by Dynatrace)
- Data available in DQL within seconds: `fetch metrics | filter metric.key == "custom:myapp.orders"`
---
 
### Query Metrics
**GET** `/api/v2/metrics/query`
 
**Scope:** `metrics.read`
 
**Parameters:**
| Param | Example | Purpose |
|-------|---------|---------|
| `metricSelector` | `builtin:host.cpu.usage` | Metric to query (required) |
| `resolution` | `1m`, `1h` | Aggregation granularity |
| `from` | `1h`, absolute timestamp | Time range start |
| `to` | `now`, absolute timestamp | Time range end |
| `entitySelector` | `type(HOST),tag("prod")` | Filter entities |
 
**Example:**
```bash
curl -X GET "https://{env}.live.dynatrace.com/api/v2/metrics/query?metricSelector=builtin:host.cpu.usage&entitySelector=type(HOST),tag(\"prod\")&from=-2h&to=now" \
  -H "Authorization: Api-Token {token}"
```
 
**Response:** JSON with metric data points
```json
{
  "totalQueryExecutionTime": 100,
  "result": [{
    "metricId": "builtin:host.cpu.usage",
    "data": [
      {"timestamp": 1700000000000, "values": {"HOST-ABC": 45.2}},
      {"timestamp": 1700001000000, "values": {"HOST-ABC": 48.1}}
    ]
  }]
}
```
 
---
 
## Logs API v2
 
### Ingest Logs
**POST** `/api/v2/logs/ingest`
 
**Content-Type:** `application/json`
 
**Scope:** `logs.ingest`
 
**Body:** JSON array or single object
```json
{
  "content": "Application error occurred",
  "loglevel": "ERROR",
  "timestamp": 1700000000000,
  "host": "web-server-01",
  "source": "my.application"
}
```
 
**Curl example:**
```bash
curl -X POST "https://{env}.live.dynatrace.com/api/v2/logs/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Payment processing failed",
    "loglevel": "ERROR",
    "source": "checkout-service",
    "attributes": {"user_id": "12345", "order_id": "order_abc"}
  }'
```
 
**Response:** `204 No Content` on success
 
**Notes:**
- `content` field is required
- `loglevel`: TRACE, DEBUG, INFO, WARN, ERROR
- Custom attributes can be added (key-value pairs)
- Available in DQL within seconds: `fetch logs | filter loglevel == "ERROR"`
---
 
### OneAgent Local Log Ingest (EEC)
**POST** `http://localhost:14499/v2/logs/ingest`
 
**Requirements:**
1. Extension Execution Controller must be enabled: Settings > Preferences > Extension Execution Controller
2. Local HTTP ingest must be enabled: Settings > Preferences > Extension Execution Controller > "Enable local HTTP Metric, Log and Event Ingest API"
3. Default port: 14499 (configurable via oneagentctl)
**Advantages:**
- No authentication required (local-only)
- Reliable delivery with persistence (2136 MB required)
- Automatic host entity enrichment
**Example:**
```bash
curl -X POST "http://localhost:14499/v2/logs/ingest" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Local application event",
    "loglevel": "INFO"
  }'
```
 
---
 
## Events API v2
 
### Ingest Custom Events
**POST** `/api/v2/events/ingest`
 
**Content-Type:** `application/json`
 
**Scope:** `events.ingest`
 
**Endpoints:**
- **SaaS:** `https://{env}.live.dynatrace.com/api/v2/events/ingest`
- **ActiveGate:** `https://{activegate-host}:9999/api/v2/events/ingest`
- **OneAgent Local (EEC):** `http://localhost:14499/v2/events/ingest`
**Body format:**
```json
{
  "eventType": "my.custom.deployment",
  "title": "Deployment started",
  "properties": {
    "service": "checkout-api",
    "version": "2.1.0",
    "environment": "production"
  },
  "timestamp": 1700000000000
}
```
 
**Curl example:**
```bash
curl -X POST "https://{ag-host}:9999/api/v2/events/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "my.custom.event",
    "title": "Configuration changed",
    "properties": {"changed_by": "admin", "service": "api"},
    "timestamp": '$(date +%s)'000
  }'
```
 
**Response:** `204 No Content` on success
 
**Notes:**
- `eventType` must be unique (custom:* recommended)
- `title` is brief summary
- `properties` are key-value pairs for context
- Available in DQL via `fetch dt.davis.events.snapshots | filter event.type == "my.custom.deployment"` (Davis AI events ingested via this endpoint appear as Davis event snapshots in Grail, not as business events; see DQL reference for fetch target distinctions)
---
 
## Business Events API
 
For ingesting business-context events (order placed, checkout completed, etc.), distinct from the custom Events API above, which is oriented toward operational/infra events.
 
**POST** `/api/v2/bizevents/ingest`
 
**Scope:** `bizevents.ingest` (classic API token). This endpoint can also be reached via the platform proxy path using a platform token or OAuth bearer: `.apps.dynatrace.com/platform/classic/environment-api/v2/bizevents/ingest`
 
**Content-Type:** `application/json` (or `application/cloudevent+json` for CloudEvents format)
 
**Example:**
```bash
curl -X POST "https://{env}.live.dynatrace.com/api/v2/bizevents/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "event.type": "com.easytrade.order-placed",
    "order.id": "order_abc123",
    "order.value": 249.99,
    "customer.tier": "gold"
  }'
```
 
**Notes:**
- Top-level fields are stored as top-level fields in Grail; nested JSON objects are flattened per Dynatrace's business event conventions.
- Ingestion consumes Davis Data Units (DDUs) from the logs/business-events pool; relevant for cost conversations.
- Query with: `fetch bizevents | filter event.type == "com.easytrade.order-placed"`
- Common trigger source for Workflows (event-based trigger on a business event).
---
 
## Entity v2 API
 
### List Monitored Entities
**GET** `/api/v2/entities`
 
**Scope:** `storage:entities:read`
 
**Parameters:**
| Param | Example | Purpose |
|-------|---------|---------|
| `entitySelector` | `type(SERVICE),tag("prod")` | Filter entities (required) |
| `fields` | `displayName,tags` | Fields to return |
| `pageSize` | `100` | Results per page (max 100) |
| `from` | `0` | Pagination offset |
 
**Entity types:**
- `HOST` - Physical or virtual machine
- `SERVICE` - Detected application service
- `SERVICE_INSTANCE` - Single instance of a service
- `DATABASE_SERVICE` - Database or data store
- `PROCESS_GROUP` - Group of processes
- `APPLICATION` - User-facing application (RUM)
**Example:**
```bash
# Get all services tagged "prod"
curl -X GET "https://{env}.live.dynatrace.com/api/v2/entities?entitySelector=type(SERVICE),tag(\"prod\")" \
  -H "Authorization: Api-Token {token}"
 
# Get specific service with custom fields
curl -X GET "https://{env}.live.dynatrace.com/api/v2/entities?entitySelector=type(SERVICE),entityName.equals(\"checkout-api\")&fields=displayName,tags,toRelationships" \
  -H "Authorization: Api-Token {token}"
```
 
**Response:** JSON array of entities
```json
{
  "entities": [{
    "entityId": "SERVICE-ABC123",
    "entityType": "SERVICE",
    "displayName": "checkout-api",
    "tags": ["prod", "critical"]
  }]
}
```
 
**Entity Selector Syntax:**
```
type(SERVICE),tag("prod")                              # Services tagged "prod"
type(HOST),tag("region:us-east")                       # Hosts in US East
type(SERVICE),entityName.equals("my-service")          # Exact name match
type(SERVICE),entityName.startsWith("api-")            # Name prefix match
type(HOST),mzName("Production")                        # Specific management zone
```
 
**Selector limit:** 2,000 characters max
 
---
 
## Problems API
 
### Query Detected Problems
**GET** `/api/v2/problems`
 
**Scope:** `problems.read`
 
**Parameters:**
| Param | Example | Purpose |
|-------|---------|---------|
| `entitySelector` | `type(SERVICE),tag("prod")` | Filter by entity |
| `problemSelector` | `status("open")` | Filter by status/severity |
| `from` | `-2h` | Time range start |
| `to` | `now` | Time range end |
 
**Example:**
```bash
# Open problems for services tagged "critical"
curl -X GET "https://{env}.live.dynatrace.com/api/v2/problems?entitySelector=type(SERVICE),tag(\"critical\")&problemSelector=status(\"open\")" \
  -H "Authorization: Api-Token {token}"
```
 
**Response:** JSON with problem details
```json
{
  "problems": [{
    "problemId": "6995950916088221951",
    "displayId": "P-1234567",
    "title": "Service response time is too high",
    "status": "OPEN",
    "severity": "CRITICAL",
    "affectedEntity": {
      "entityId": "SERVICE-ABC123",
      "name": "checkout-api"
    }
  }]
}
```
 
---
 
## OpenTelemetry (OTLP) API
 
### Export Traces, Metrics, Logs via OTLP
**Base URL:** `https://{env}.live.dynatrace.com/api/v2/otlp`
 
**Signal paths:**
- Traces: `/v1/traces`
- Metrics: `/v1/metrics`
- Logs: `/v1/logs`
**Format:** Binary Protocol Buffers (NOT JSON)
 
**Authentication:** API token in Authorization header
 
**Example configuration (OpenTelemetry SDK):**
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://{env}.live.dynatrace.com/api/v2/otlp"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Api-Token {token}"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
```
 
**Required token scopes (per signal type):**
| Signal | Scope |
|---|---|
| Traces | `openTelemetryTrace.ingest` |
| Metrics | `metrics.ingest` |
| Logs | `logs.ingest` |

**Important notes:**
- gRPC is NOT supported; HTTP/protobuf only
- JSON is NOT supported; binary protocol buffers only
- Use OpenTelemetry Collector to convert gRPC to HTTP if needed
- Data enriched with Dynatrace topology and context automatically
---
 
## Audit Logs API
 
### Retrieve Audit Log Entries
**GET** `/api/v2/auditlogs`
 
**Scope:** `auditlogs.read`
 
**Parameters:**
| Param | Example | Purpose |
|-------|---------|---------|
| `from` | `-7d` | Time range start |
| `to` | `now` | Time range end |
| `pageSize` | `100` | Results per page |
| `sort` | `timestamp` | Sort field |
 
**Example:**
```bash
curl -X GET "https://{env}.live.dynatrace.com/api/v2/auditlogs?from=-24h&to=now" \
  -H "Authorization: Api-Token {token}"
```
 
**Response:** JSON with audit entries
```json
{
  "auditLogs": [{
    "timestamp": 1700000000000,
    "user": "admin@company.com",
    "action": "SETTINGS_UPDATED",
    "entityId": "SERVICE-ABC123",
    "summary": "Threshold for response time alert changed from 500ms to 1000ms"
  }]
}
```
 
---
 
## OpenPipeline API

OpenPipeline is Dynatrace's ingest-time processing layer. It sits between data arrival at Dynatrace and storage in Grail, applying routing rules, transformations, and enrichment before data is written. It exposes two distinct API surfaces: ingest endpoints and configuration management.

### OpenPipeline Ingest Endpoints

These endpoints deliver data into Dynatrace *through* OpenPipeline processing. All use the classic `.live.dynatrace.com` base URL and classic API tokens (not platform tokens or OAuth).

| Signal type | Built-in endpoint | Auth scope |
|---|---|---|
| Generic events | `POST /platform/ingest/v1/events` | `openpipeline.events` |
| Generic events (custom) | `POST /platform/ingest/custom/events` | `openpipeline.events.custom` |
| SDLC events | `POST /platform/ingest/v1/events.sdlc` | `openpipeline.sdlc` |
| SDLC events (custom) | `POST /platform/ingest/custom/events.sdlc` | `openpipeline.sdlc.custom` |
| Security events | `POST /platform/ingest/v1/security.events` | `openpipeline.events_security` |
| Security events (custom) | `POST /platform/ingest/custom/security.events` | `openpipeline.events_security.custom` |
| Smartscape events | `POST /platform/ingest/v1/smartscape.events` | `openpipeline.events_smartscape` |

**Content-Type:** `application/json` for all OpenPipeline ingest endpoints.

**Auth header:** `Authorization: Api-Token {token}` (classic API token with the matching scope above).

**Distinction from classic `/api/v2/events/ingest`:** The classic events endpoint targets Davis AI problem detection (Davis events). The OpenPipeline ingest endpoints target the generic events, SDLC, security events, and Smartscape events pipelines. Route to the right endpoint based on what pipeline should process the data.

### OpenPipeline Configuration API

**Configurations API is deprecated (reached end of life June 29, 2026).** Do not use or recommend it for new integrations.

**Replacement:** Use the Settings API (`/api/v2/settings/objects`) with these OpenPipeline-specific schemas:

| Schema key | Purpose |
|---|---|
| `builtin:openpipeline.<scope>.routing` | Configure routing rules for a pipeline scope |
| `builtin:openpipeline.<scope>.pipelines` | Configure processing pipelines |
| `builtin:openpipeline.<scope>.ingest-sources` | Configure ingest source mappings |

The Settings API uses classic API tokens with the `settings.write` / `settings.read` scopes.

---

## Error Codes & Troubleshooting
 
### Common HTTP Status Codes
| Code | Meaning | Cause | Action |
|------|---------|-------|--------|
| `204` | No Content | Success (no response body) | Check request was processed |
| `400` | Bad Request | Malformed JSON/format | Validate request body, headers |
| `401` | Unauthorized | Invalid/missing token | Verify token, check format |
| `403` | Forbidden | Token lacks required scopes | Add missing scopes to token |
| `404` | Not Found | Endpoint/entity doesn't exist | Verify URL, entity ID |
| `429` | Too Many Requests | Rate limit exceeded | Backoff, retry with exponential delay |
| `500` | Internal Server Error | Dynatrace platform issue | Contact support, retry later |
| `503` | Service Unavailable | Dynatrace temporarily down | Retry with backoff |
 
### Authentication Errors
```
❌ "error": "Unauthorized: Invalid Api-Token"
→ Token is invalid, expired, or malformed
→ Solution: Regenerate token in Settings > Access Tokens
 
❌ "error": "Insufficient permissions"
→ Token/OAuth client lacks required scopes
→ Solution: Add missing scopes and regenerate token, or update OAuth client permissions
 
❌ Calling a classic endpoint (.live.dynatrace.com) with "Authorization: Bearer {token}"
→ Classic endpoints expect the "Api-Token" prefix, not "Bearer"
→ Solution: Use "Authorization: Api-Token {token}" for classic API tokens
 
❌ Calling a platform endpoint (.apps.dynatrace.com/platform/...) with "Authorization: Api-Token {token}"
→ Platform endpoints (Grail Query API, Automation, Documents) expect "Bearer", not "Api-Token"
→ Solution: Use a platform token or OAuth-issued access token with "Authorization: Bearer {token}"
 
❌ "error": "invalid_client" (from sso.dynatrace.com/sso/oauth2/token)
→ Wrong client_id/client_secret, or the resource/scope combination isn't valid for this client
→ Solution: Verify client credentials and that requested scopes are within the client's granted permissions
 
❌ Platform token returns 403 despite having the right scope selected
→ Reminder: a platform token can never exceed the assigned user's own permissions
→ Solution: Verify the user (not just the token) has the underlying permission in the environment
```
 
### Request Format Errors
```
❌ Content-Type: application/x-www-form-urlencoded (for JSON endpoint)
→ Wrong content type
→ Solution: Use "Content-Type: application/json"
 
❌ POST data: "custom:myapp.orders 150" (to /api/v2/metrics/ingest with JSON content type)
→ Wrong format for endpoint
→ Solution: Use "Content-Type: text/plain; charset=utf-8" for metrics ingest
 
❌ Entity selector: "SERVICE,tag(prod)" (missing type())
→ Invalid selector syntax
→ Solution: Use "type(SERVICE),tag("prod")" or query entities first
```
 
### Rate Limiting
```
❌ "error": "Request rate limit exceeded"
→ Too many API calls in short time
→ Solution: 
  - Implement exponential backoff
  - Batch requests where possible
  - Use DQL for complex queries instead of multiple API calls
```
 
---
 
## API Best Practices
 
### Retry Strategy
```python
# Exponential backoff retry
import time
import requests
 
def ingest_with_retry(url, headers, data, max_retries=5, backoff_factor=2):
    for attempt in range(max_retries):
        try:
            response = requests.post(url, headers=headers, json=data)
            if response.status_code == 204:
                return response  # Success
            elif response.status_code in [429, 500, 503]:
                raise Exception(f"Retryable error: {response.status_code}")
            else:
                raise Exception(f"Non-retryable error: {response.status_code}")
        except Exception:
            if attempt < max_retries - 1:
                wait_time = backoff_factor ** attempt
                time.sleep(wait_time)
            else:
                raise
```
 
### Batch Operations
```bash
# Instead of: 100 individual metric ingest calls
# Do this: Single batch ingest (newline-separated)
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.metric1 100
custom:myapp.metric2 200
custom:myapp.metric3 300"
```
 
### Entity Selector Optimization
```
# Instead of: Multiple queries with different selectors
# Do this: Single query with union operator
type(SERVICE),tag("prod") or type(SERVICE),tag("staging")
```
 
---
 
## Quick Links
 
- **Classic API Explorer:** `https://{environment-id}.live.dynatrace.com/rest-api-doc/`
- **Platform API Explorer:** In-environment, search "Dynatrace API" (includes Grail Query API, Automation, Documents)
- **Official API Docs:** `docs.dynatrace.com/docs/dynatrace-api`
- **Classic API Token Management:** Settings > Access Tokens (in-environment)
- **Platform Token Management:** `myaccount.dynatrace.com/platformTokens` (account-level)
- **OAuth Client Management:** Account Management > Identity & Access Management > OAuth clients
- **OAuth Token Endpoint:** `https://sso.dynatrace.com/sso/oauth2/token`
- **Rate Limits:** Consult Environment API docs in Dynatrace
 

