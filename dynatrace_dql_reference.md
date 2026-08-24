# DQL Reference Guide: Fundamentals, Smartscape Topology, and Performance
 
> **Last verified against docs.dynatrace.com:** July 15, 2026
> **Staleness policy:** Core DQL syntax and the Smartscape topology model are stable architectural concepts; treat those sections as durable. Performance thresholds, exact error message text, default timeframes, and scan limits are UI/behavior-dependent and move on the same ~90-day cadence as the rest of the reference guide and troubleshooting trees, if a query fails with an unexpected error or an unfamiliar relationship/entity type, verify against docs.dynatrace.com/docs/dynatrace-api/grail/dql before treating this file as final.
> **Consolidation note:** This file is the single authoritative source for DQL syntax, topology querying, and DQL performance/troubleshooting. It replaces content formerly duplicated across the Reference Guide's DQL section, Troubleshooting Tree 6, the Common Questions FAQ's DQL entry, and the standalone Smartscape reference file. Those files now contain short pointers here rather than their own copies. If you find DQL guidance anywhere else in this project that isn't just a pointer to this file, flag it, that's drift.
 
---
 
## Part 1: DQL Fundamentals
 
### Core Concepts
- DQL is a **read-only** query language against Grail data, using a pipeline model with the pipe operator (`|`).
- **Core commands**: `fetch`, `filter`, `filterOut`, `search`, `summarize`, aggregation functions, `join`, `parse`, `fields`, `limit`, `sort`, `maketimeseries`.
- **Data types**: boolean, long, double, string, timestamp, duration, ipaddress, uid, array, record.
- **Strongly typed**: functions accept only their declared types; casting is required for conversions between types.
### Data Sources (fetch targets)
| Fetch target | Queries |
|---|---|
| `fetch logs` | Log records from Grail buckets |
| `fetch spans` | Distributed trace span data |
| `fetch events` | Custom business events |
| `fetch dt.entity.*` | Monitored entities (hosts, services, databases, etc.) via Smartscape views, see Part 2 |
| `fetch metrics` | Metric time series (`builtin:*` for out-of-the-box metrics, `custom:*` for ingested metrics) |
 
### Best Practices (why they matter, not just what to do)
- **Use shorter timeframes.** Grail scans more data over longer windows; narrower windows execute faster and cost less.
- **Filter early**, immediately after `fetch`, before any `summarize` or `join`. This is the single highest-leverage change for a slow query, it reduces the data volume every downstream step has to process.
- **Use sampling for large datasets**: `fetch spans, samplingRatio:100` samples 1-in-100 records. Good for exploratory queries where exact counts don't matter yet.
- **Aggregate (`summarize`) before sorting.** Sorting a large unaggregated result set is expensive; sorting a small aggregated one is cheap.
- **Place `sort` at the end of the pipeline.**
- **Avoid expensive transformations inside filters**, e.g. `filter lower(name) == "abc"`. Filter on the raw field instead (`filter name ~ "pattern"`); transformations at read time are evaluated per-record and are expensive at scale.
---
 
## Part 2: Smartscape Topology and Relationship Queries
 
### Nodes and Edges
**Nodes** are monitored entities in Smartscape: hosts, services, service instances, databases, processes, applications, external services, etc. Each node has a unique `entityId`, an `entityType`, attributes (displayName, tags, metrics), and relationships (edges) to other nodes.
 
**Edges** are directional relationships between nodes representing dependencies and causality (a service `runs_on` a host, a service `calls` another service, a service `uses` a database, a host `resides_in` a cloud region). Querying requires knowing which direction you're traversing, `calls` (A→B) is not the same as `is_called_by` (B←A).
 
### Basic Syntax
 
**Fetch entities only (no relationships):**
```dql
fetch dt.entity.service
| fields entityId, displayName, tags
```
 
**Fetch with one relationship hop:**
```dql
fetch dt.entity.service
| expand [runs_on(from: dt.entity.host)]
```
 
**Multi-hop traversal:**
```dql
fetch dt.entity.service
| expand [runs_on(from: dt.entity.host)]
| expand [resides_in(from: dt.entity.cloudPlatform)]
```
 
### Entity Types (Node Types)
| Entity Type | Represents |
|---|---|
| `dt.entity.host` | Physical or virtual machine |
| `dt.entity.service` | Detected application service |
| `dt.entity.service_instance` | Single process instance of a service |
| `dt.entity.database_service` | Database or data store |
| `dt.entity.process_group` | Logical grouping of processes |
| `dt.entity.application` | Web/mobile application (RUM) |
| `dt.entity.external_service` | Third-party API or downstream dependency |
| `dt.entity.cloud_application` | Cloud-native application (Lambda, etc.) |
| `dt.entity.cloud_platform_region` | Cloud region or account |
 
### Relationship Types (Edge Types)
| Edge | Direction | Meaning |
|---|---|---|
| `runs_on` | Service → Host | Service process running on this host |
| `calls` | Service A → Service B | Synchronous or async call |
| `is_called_by` | Service A ← Service B | Inverse of `calls` |
| `uses` | Service → Database | Service queries this database |
| `resides_in` | Host → Cloud Region | Host deployed in this region/account |
| `contains` | Service → Process Group | Service comprises these processes |
| `is_contained_in` | Process Group → Service | Inverse of `contains` |
 
**Note:** this table covers the most common queries, not the full relationship inventory. If a relationship you expect isn't listed, verify it exists in docs.dynatrace.com for your Dynatrace version before assuming it's missing.
 
### Common Topology Query Patterns
 
**Find all services running on a host:**
```dql
fetch dt.entity.host
| filter displayName == "web-server-01"
| expand [runs_on(from: dt.entity.service)]
| fields dt.entity.service.displayName
```
 
**Find all downstream dependencies of a service (one hop):**
```dql
fetch dt.entity.service
| filter displayName == "checkout-api"
| expand [calls(from: dt.entity.service)]
| fields displayName
```
 
**Find all services in a management zone, with hosting hosts:**
```dql
fetch dt.entity.service
| filter in(mzName, ["Production"])
| expand [runs_on(from: dt.entity.host)]
| fields dt.entity.service.displayName, dt.entity.host.displayName
```
 
**Trace a request path (service → service → database):**
```dql
fetch dt.entity.service
| filter displayName == "api-gateway"
| expand [calls(from: dt.entity.service)]
| expand [uses(from: dt.entity.database_service)]
| fields dt.entity.service.displayName, dt.entity.database_service.displayName
```
 
**Identify single-threaded dependencies (high-risk topology):**
```dql
fetch dt.entity.service
| expand [calls(from: dt.entity.service)]
| summarize service_count = count(), by: {called_service_id: dt.entity.service.entityId}
| filter service_count == 1
```
 
**List all services in the environment (count only):**
```dql
fetch dt.entity.service
| summarize count_services = count()
```
 
**Combine topology with metrics** (join a Smartscape traversal with a metric lookup):
```dql
fetch dt.entity.service
| expand [runs_on(from: dt.entity.host)]
| lookup [fetch metrics
  | filter metric.key == "builtin:host.cpu.usage"
  | fields host_id: dt.entity.host.entityId, cpu_usage: avg(value)
] on dt.entity.host.entityId == host_id
| fields dt.entity.service.displayName, cpu_usage
```
 
### Filtering on Topology Attributes
 
**By relationship count:**
```dql
fetch dt.entity.service
| expand [calls(from: dt.entity.service)]
| summarize dependencies = count(), by: {service_id: dt.entity.service.entityId}
| filter dependencies > 5
```
 
**By entity type within a relationship:**
```dql
fetch dt.entity.service
| expand [runs_on(from: dt.entity.host, filter: tag("environment:prod"))]
```
 
### Edge Direction and Traversal Rules
 
Directionality matters, `calls` and `is_called_by` answer different questions:
```dql
# Forward: all services THIS service calls
fetch dt.entity.service
| expand [calls(from: dt.entity.service)]
 
# Backward: all services that call THIS service
fetch dt.entity.service
| expand [is_called_by(from: dt.entity.service)]
```
 
DQL doesn't loop infinitely on circular traversal, but un-scoped repeated expansion of the same relationship type creates bloated result sets. Add a filter (e.g. `filter displayName != previousServiceName`) if you need to revisit a relationship type.
 
### Common Patterns: Do's and Don'ts
| Do | Don't |
|---|---|
| `fetch dt.entity.service \| expand [calls(from: dt.entity.service)]` | `fetch dt.entity.service \| expand [calls]` (ambiguous target type) |
| `filter mzName("Production")` on the entity, before expand | `filter mzName("Production")` on the expanded relationship; may not work as expected |
| `summarize count(), by: {service_id}` to aggregate topology | Expanding all 50k services without filtering first (expensive) |
| `lookup [pre-aggregated metrics]` to join with aggregated data | `lookup [raw span data]` against topology (cardinality explosion) |
 
### Troubleshooting Topology Queries
 
**"Unknown relationship type" error:** confirm the relationship exists for the entity types you're querying (`calls` only exists between services, not between hosts). Check the relationship matrix in docs.dynatrace.com.
 
**Query returns empty results:** verify the source entity exists and has the relationship you're querying. Test in steps, fetch the source entity first, then add one `expand` at a time, use `limit 1` and inspect a result before scaling.
 
**Expanding returns null/empty nested fields:** the relationship may not exist for all entities matching your fetch. Use `filter` on the expand to scope it, or accept nulls and filter downstream (not every service calls a database).
 
---
 
## Part 3: DQL Performance and Troubleshooting
 
**Symptom**: DQL query returns a timeout error, or takes more than roughly 30 seconds to execute.
 
**Assumptions**: you're running DQL in Notebooks, the Query API, or dashboards, and you have sufficient token scopes to query Grail data (if you're getting a permission-denied error instead of a timeout, that's an auth issue, not a performance issue, see the API Quick Reference file's classic-vs-platform auth decision tree instead).
 
```
How large is the time range?
│
├─ >7 days? →
│  ├─ Reduce to last 24-48 hours and re-run
│  ├─ Use the bucket parameter if querying specific buckets
│  └─ Large time ranges are expensive in Grail; default query timeframe is 2 hours
│
└─ ≤7 days → Is the query filtering early?
             │
             ├─ NO (filter after summarize or join) →
             │  ├─ Move filter right after fetch
             │  ├─ Correct order: fetch → filter → summarize → sort
             │  └─ This reduces data scanned dramatically
             │
             └─ YES → Is the query using expensive operations?
                      │
                      ├─ YES (lower(), regex transformations in filter) →
                      │  ├─ Filter on the raw field instead of a transformed value
                      │  ├─ Example: filter name ~ "pattern" instead of filter lower(name) == "abc"
                      │  └─ Transformations at read time are expensive
                      │
                      └─ NO → Check data volume
                             ├─ Use samplingRatio parameter:
                             │  fetch spans, samplingRatio:100 (samples 1-in-100 records)
                             ├─ Increase scanLimitGBytes if needed (default 500 GB):
                             │  fetch logs, scanLimitGBytes: -1 (no limit, not recommended)
                             ├─ Check if the query is hitting a rate limit (see API docs)
                             └─ If still slow, build the query incrementally in a notebook
```
 
### Optimization Checklist
1. Shorter timeframe first (start with `-2h` or `-24h`)
2. Filter immediately after fetch
3. Summarize before sort
4. Avoid transformations in filters
5. Use sampling for exploratory queries
6. Place sort at the end of the pipeline
### Validating Query Structure
```
# Discover available fields on a data type
fetch logs | fieldsSnapshot logs
 
# Fast sanity check (no aggregation)
fetch spans | limit 1
 
# Test with sampling
fetch spans, samplingRatio:100 | summarize count_spans = count()
```
 
**Follow-up questions to ask:**
- What's the exact timeout error, and how many records were you expecting to scan?
- Can you share the query?
- Is sampling already in use?
- What's the time range?
**When to escalate:** if a query is still timing out after the full checklist above (short timeframe, early filter, no transformations in filter, sampling applied), that's a case for Dynatrace Support, not further self-optimization, per the Quick Decision Matrix in the Troubleshooting Trees file.
 
---
 
## Glossary
 
| Term | Definition |
|---|---|
| **Node** | A monitored entity in Smartscape (host, service, database, etc.) |
| **Edge** | A directional relationship between two nodes |
| **Expand** | DQL operation that traverses an edge, adding related entities to the result set |
| **Directionality** | The direction of an edge; `calls` (A→B) is not the same as `is_called_by` (B←A) |
| **Multi-hop** | Traversing multiple edges in sequence (entity A → B → C) |
| **Relationship type** | The semantic meaning of an edge (e.g., `runs_on`, `calls`, `uses`) |
| **Entity type** | The classification of a node (SERVICE, HOST, DATABASE_SERVICE, etc.) |
| **samplingRatio** | DQL fetch parameter controlling data sampling (1, 10, 100, 1000, 10000); higher values sample less data |
| **scanLimitGBytes** | DQL fetch parameter controlling the maximum data volume a query is allowed to scan (default 500 GB) |
 
---
 
## When to Escalate
 
- New relationship types or entity types not documented here → check docs.dynatrace.com or escalate to Dynatrace support.
- Performance issues with large Smartscape queries (timeouts, memory) after the full optimization checklist above → escalate; may require query tuning or index support from Dynatrace.
- Unexpected missing relationships (an edge you expect doesn't exist) → verify with support whether the dependency was actually captured by OneAgent, or whether relationship discovery is disabled, before assuming it's a query problem.
- Questions about running DQL programmatically via the Grail Query API (execute/poll pattern, OAuth vs. platform token requirements) → that's an authentication topic, not a query-language topic. See the API Quick Reference file's "Grail Query API" and "Two API Generations" sections; don't try to resolve auth failures from this file.
