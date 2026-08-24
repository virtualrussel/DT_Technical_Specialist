# Dynatrace Common Questions FAQ
 
> **Last verified against docs.dynatrace.com:** 2026-08-24
> **Staleness policy:** Pricing, licensing units, token scope names, and feature availability are the fastest-moving content in this file. Treat these as directional, not final. Confirm current scope names, retention numbers, and licensing terms against docs.dynatrace.com or the account team before quoting them to a customer in a pre-sales conversation.
 
Pre-structured answers for the most frequently asked questions in pre-sales and incident response. Use these to provide consistent, accurate responses without re-deriving answers.
 
---
 
## Deployment & Installation
 
### Q: What's the difference between Full-Stack, Cloud-Native, and Application-Only monitoring?
 
**A:** Three deployment modes for different environments:
 
| Mode | Use Case | Coverage | Requirements |
|------|----------|----------|--------------|
| **Full-Stack** | Traditional VMs, bare metal | Infrastructure + Apps + RUM | OneAgent with elevated privileges |
| **Cloud-Native** | Kubernetes, OpenShift | Infrastructure + Apps in containers | Dynatrace Operator; Kubernetes 1.25+ |
| **Application-Only** | Serverless, PaaS, restricted envs | Apps only (no infra) | Minimal footprint; Cloud Foundry, Lambda, Heroku, App Engine |
 
**Decision rule:** Use Full-Stack by default. Switch to Cloud-Native if containerized. Use Application-Only if infrastructure access is restricted or for serverless.
 
**Reference:** OneAgent Deployment Modes section in reference guide.
 
---
 
### Q: Which operating systems and platforms does OneAgent support?
 
**A:** Full support matrix in reference guide, but key points:
 
**Linux/Windows:**
- Linux: RHEL, CentOS, Debian, Ubuntu (x86, ARM64)
- Windows: Server 2012+ (x86, ARM Graviton2)
- **NOT Alpine Linux** for classic full-stack (use cloud-native alternative)
- Solaris, AIX: Application-only injection only
**Containers:**
- Docker: Standalone and swarm (Full-Stack supported)
- Kubernetes: 1.25+ with Dynatrace Operator
- OpenShift: 4.16+ (4.19+ for s390x)
- AWS Lambda: Full tracing support
- AWS Fargate: Application-only
- Cloud Foundry: Buildpack integration
**Rule of thumb:** If it's Linux/Windows on x86-64, OneAgent works. Otherwise, check platform support matrix.
 
**Reference:** OneAgent Platform Support section in reference guide.
 
---
 
### Q: Can OneAgent monitor [technology]?
 
**A:** Check the Technology Support matrix at docs.dynatrace.com/docs/ingest-from/technology-support
 
Common supported technologies:
- **Languages:** Java (all), .NET Framework/.NET Core 3+, Node.js 12+, Python 3.8+, Go 1.9+, PHP 5.6+, C/C++ (SDK)
- **Databases:** Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, DynamoDB (auto-detected via JDBC, .NET, Node.js, Python drivers)
- **Messaging:** Kafka, RabbitMQ, AWS SQS/SNS, Google Pub/Sub
- **Clouds:** AWS, Azure, Google Cloud (services auto-discovered via OneAgent)
**If not in matrix:** Use OpenTelemetry SDK to manually instrument.
 
---
 
### Q: How do I deploy and troubleshoot Dynatrace on Kubernetes/OpenShift?
 
**A:** Dynatrace Operator manages the lifecycle of OneAgent, ActiveGate, and related components in-cluster via a Custom Resource called **DynaKube**.
 
**Deployment modes:**
| Mode | Coverage |
|------|----------|
| `cloudNativeFullStack` | Infrastructure + application observability in containers |
| `applicationMonitoring` | Application-only injection, no host-level monitoring |
| `hostMonitoring` | Infrastructure-only, no code injection |
| Kubernetes Platform Monitoring | Cluster topology, node/container metrics (no OneAgent required) |
 
**Key rules:**
- Only one DynaKube can inject into a given namespace at a time.
- Classic full-stack OneAgent and webhook injection cannot coexist in the same Pod (both mount `/var/lib/dynatrace`).
- The CSI driver reduces redundant code-module downloads (once per node vs. once per Pod).
**First troubleshooting steps:**
```bash
kubectl get dynakubes -n dynatrace
kubectl -n dynatrace get pods
kubectl exec deploy/dynatrace-operator -n dynatrace -- dynatrace-operator troubleshoot
```
 
**Reference:** Dynatrace Operator Deep Dive section in reference guide; Troubleshooting Tree 8.
 
---
 
## Data Retention & Limits
 
### Q: How long is trace and metric data retained?
 
**A:** Retention policy varies:
 
| Data Type | Default Retention | Notes |
|-----------|------------------|-------|
| **Traces (detailed)** | 10 days | Full span data with code-level info |
| **Trace metadata** | 90 minutes | Running traces timeout after 90 min |
| **Metrics (1-min granularity)** | 7 days | High-resolution data |
| **Metrics (1-hour granularity)** | 400 days (~13 months) | Aggregated view |
| **Logs** | Contract-dependent | Configured per subscription |
| **Custom metrics** | Per subscription tier | Varies |
 
**Key:** Traces older than 10 days won't appear in Distributed Traces view. Metrics can extend beyond 10 days. Use DQL to query older metric data if available.
 
**Reference:** Common Feature Boundaries and Limitations section in reference guide.
 
---
 
### Q: What are the limits on traces, queries, and API calls?
 
**A:** Hard limits to know:
 
| Limit | Value | Impact |
|-------|-------|--------|
| **Nodes per trace** | ~3000 span nodes | Excessive spans truncate; diagnostic message appears |
| **Traces per view** | 3,000 most recent | Older traces don't appear in Distributed Traces view |
| **Entity selector string length** | 2,000 characters | Long selectors are rejected |
| **1:n relationship records** | 100 entity IDs max | Relationship lookups cap at 100 |
| **DQL data scan** | 500 GB default | Queries abort if they'd scan more (configurable) |
| **API rate limiting** | Per environment | Consult Environment API docs for current limits |
 
**Rule:** If you hit a limit, you'll see a diagnostic message or error. Reference the guide and adjust (e.g., use sampling for large DQL queries, reduce custom services to cut trace nodes).
 
**Reference:** Common Feature Boundaries and Limitations section in reference guide.
 
---
 
## Configuration & Scoping
 
### Q: How do I set up custom services or name endpoints?
 
**A:** Two approaches depending on service detection version:
 
**Service Detection v2 (recommended):**
- Automatic if service exposes `http.route` or gRPC method names
- Manual config rare; check Settings > Service Detection if needed
**Service Detection v1 (legacy):**
- Use "Enhanced endpoints" feature for automatic endpoint detection (no key request config needed)
- Or manually configure key requests for specific endpoints
- Rule format: `{HTTP-METHOD} /path` with URL conditions
**For custom services (trace segmentation):**
- Use OneAgent SDK or manually configure in Settings > Service Detection
- Each custom service adds trace nodes; limit to avoid node truncation
**Reference:** Application Observability section in reference guide.
 
---
 
### Q: How do I capture custom request attributes (user ID, region, etc.)?
 
**A:** Use request attributes:
 
**For OneAgent-instrumented services:**
1. Settings > Application Observability > Request Attributes
2. Define extraction rule: source (HTTP header, query param, cookie, body)
3. Name the attribute: `my.request.user_id`
4. Attributes appear in trace details and can be used in DQL/dashboards
**For OpenTelemetry:**
- Include attributes in spans (standard semantic conventions)
- Dynatrace auto-enriches with topology context
**Common pattern:**
```
Source: HTTP Header
Header name: X-User-ID
Attribute name: user.id
```
 
**Reference:** Application Observability section in reference guide.
 
---
 
### Q: What are management zones, and when should I use them?
 
**A:** Logical grouping of entities for access control and alerting scope:
 
- **Use case 1:** Restrict user access (e.g., team sees only their services)
- **Use case 2:** Separate alerting thresholds per environment (prod vs. staging)
- **Use case 3:** Organize large environments by geography or business unit
**Configuration:**
1. Settings > Management Zones
2. Define entity filter (e.g., `tag("env:prod")`)
3. Assign users to zone
4. Alerts scoped to zone only
**Default:** No zone = all users see all entities.
 
**Reference:** Configuration and Settings section in reference guide.
 
---
 
### Q: We have multiple Dynatrace environments, how do we manage them together?
 
**A:** Multi-environment setups are managed at the **account level**, separate from any single environment's settings:
 
- **Account Management**: Customers can manage multiple Dynatrace environments (tenants) under one account; each environment gets its own dedicated domain and (for SaaS) its own AWS S3 storage.
- **Cross-environment access**: User permissions, SSO/SAML configuration, and billing/subscription details are managed centrally in Account Management, then applied per environment.
- **Common reasons to split environments**: Separating production from non-production, regional data residency requirements, or organizational/business-unit boundaries.
- **Cross-environment tracing**: If configured, trace data fetched from remote environments is aggregated (not truncated) when displayed.
- **Management zones vs. multiple environments**: Use management zones to segment access *within* one environment; use separate environments when you need full data isolation, separate billing, or separate upgrade/version control.
**Rule:** Default to management zones within a single environment unless there's a hard requirement for data isolation, billing separation, or regional residency. Multiple environments add operational overhead (separate dashboards, separate SLOs, separate configuration).
 
**Reference:** Core Architecture (Dynatrace Environment) section and Data Security and Compliance section in reference guide.
 
---
 
### Q: How do I configure a Service-Level Objective (SLO)?
 
**A:** Three SLO types, chosen based on how you want the SLI calculated:
 
| Type | How it works |
|------|--------------|
| **Metric-based** | Pre-aggregated metrics; fastest evaluation |
| **Request-based (event-based)** | Ratio of "good" events to total events; adjusts to load automatically |
| **Time-slice-based (window-based)** | Classifies each interval as good/bad against a threshold |
 
**Setup path:**
1. Use the SLO wizard with a built-in template (service availability, service performance, synthetic availability, host CPU, etc.), **or**
2. Define a **custom DQL query** (must return an `sli` field as an array of doubles)
**Key configuration elements:**
- **Target**: the goal (e.g., 99.9% success rate)
- **Evaluation period**: the rolling window the status is calculated over
- **Error budget**: how much "badness" remains before breaching target; burn rate = how fast that budget is being consumed
**Common next steps:**
- Scope the SLO to a management zone if it shouldn't be visible to all users
- Create a burn-rate alert directly from the SLO (appears as a metric event)
- Pin the SLO to a dashboard as a tile
**Reference:** Service-Level Objectives (SLO) section in reference guide.
 
---
 
## Data Ingestion
 
### Q: How do I ingest custom metrics, logs, or events?
 
**A:** Three methods:
 
**1. API Endpoints (External)**
```bash
# Custom metrics
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.orders {region=us} 150"
 
# Custom logs
curl -X POST "https://{env}.live.dynatrace.com/api/v2/logs/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: application/json" \
  -d '{"content":"error message","loglevel":"ERROR"}'
 
# Custom events
curl -X POST "https://{ag-host}:9999/api/v2/events/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: application/json" \
  -d '{"eventType":"my.custom.event","properties":{"key":"value"}}'
```
 
**2. OneAgent Local Endpoints (EEC)**
- Enable Extension Execution Controller: Settings > Preferences > Extension Execution Controller
- OneAgent exposes `http://localhost:14499/v2/{metrics,logs,events}/ingest`
- No authentication required; local-only
- Requires 2136 MB disk space for reliability persistence
**3. OpenTelemetry (OTLP)**
- Export traces/metrics/logs via standard OTLP protocol
- Dynatrace enriches with topology and context
- Endpoint: `https://{env}.live.dynatrace.com/api/v2/otlp/v1/{traces,metrics,logs}`
- Binary protobuf over HTTP only; gRPC and JSON are not supported
**Rule:** For applications running on OneAgent host, use EEC. For external systems, use API. For OpenTelemetry libraries, use OTLP.
 
**Reference:** API Quick Reference file for full endpoint and scope details; Extension Execution Controller section in reference guide for EEC specifics.
 
---
 
### Q: What is OpenPipeline, and how does it fit into the data flow?

**A:** OpenPipeline is Dynatrace's server-side, ingest-time data processing layer. Data arriving at Dynatrace (via OTLP, the classic ingest APIs, or the OpenPipeline ingest endpoints) passes through OpenPipeline before being written to Grail. This is where you configure parsing, enrichment, masking, routing, and field extraction rules that apply to every record of a given signal type.

**Data flow:**
```
External source → [OneAgent or Bindplane or direct API] → OpenPipeline (ingest-time processing) → Grail (storage) → DQL (query)
```

**Key points:**
- OpenPipeline is always present — it's the ingest layer for logs, events, bizevents, spans, and metrics, not an optional add-on.
- Configuration used to be managed via the OpenPipeline Configurations API. That API was **deprecated June 29, 2026** — configuration now goes through the Settings API with `builtin:openpipeline.<scope>.*` schemas (e.g., `builtin:openpipeline.logs.routing`).
- The OpenPipeline ingest endpoints (`/platform/ingest/v1/events`, `/platform/ingest/v1/events.sdlc`, etc.) use `.live.dynatrace.com` base URL and classic API tokens with `openpipeline.*` scopes — see the API Quick Reference file for the full endpoint and scope table.
- "OpenPipeline processing" is not the same as "Bindplane pipeline." OpenPipeline runs inside Dynatrace after data arrives; Bindplane runs at or near the collection point before data leaves your network.

**Reference:** API Quick Reference file (OpenPipeline API section); reference guide (OpenPipeline section).

---

### Q: OpenPipeline vs. Bindplane — which should I use?

**A:** They operate at different points in the data path and are not mutually exclusive:

| | OpenPipeline | Bindplane |
|---|---|---|
| **Where it runs** | Inside Dynatrace (server-side) | At the source, before data leaves your network |
| **When it processes** | After data arrives at Dynatrace | Before data is sent to any backend |
| **Primary use case** | Parsing, enrichment, masking, routing within Dynatrace | Filtering, reducing, and routing telemetry to multiple destinations |
| **Multi-destination** | No (Dynatrace only) | Yes (Dynatrace, Datadog, Splunk, etc.) |
| **Network egress reduction** | No (data already sent) | Yes (filter before sending) |
| **Manages** | Ingest-time transforms on data already in Dynatrace | Which data gets sent to Dynatrace (and others) at all |

**Decision rule:** If the goal is reducing how much data leaves your network or routing to multiple backends, use Bindplane (or use Bindplane to pre-filter before sending to Dynatrace, where OpenPipeline then handles further processing). If the goal is transforming or routing data that's already arriving at Dynatrace, use OpenPipeline. These are complementary, not competing.

**Reference:** Bindplane reference file; API Quick Reference file (OpenPipeline API section).

---

### Q: What token scopes do I need for different use cases?
 
**A:** Common scopes per use case:
 
| Use Case | Required Scopes |
|----------|-----------------|
| **Ingest custom metrics** | `metrics.ingest` |
| **Query metrics via DQL** | `storage:metrics:read` |
| **Ingest logs** | `logs.ingest` |
| **Query logs via DQL** | `storage:logs:read` |
| **Ingest events** | `events.ingest` |
| **Query events via DQL** | `storage:events:read` |
| **Query entities** | `storage:entities:read` |
| **Read problems** | `problems.read` |
| **Read audit logs** | `auditlogs.read` |
 
**Rule:** Create a token with only scopes needed for the specific use case (principle of least privilege).
 
**Reference:** Authentication section (under Dynatrace API v2) in reference guide; Authentication Basics section in API Quick Reference file.
 
---
 
## Distributed Tracing & Analysis
 
### Q: Why are some of my traces truncated or missing spans?
 
**A:** Dynatrace truncates traces under two conditions:
 
**1. Node limit exceeded (~3000 spans/trace):**
- Symptom: Diagnostic message "Trace truncated due to node limit"
- Cause: Too many custom services, long chains of calls, or unusual topology
- Fix: Reduce custom services, disable unnecessary OneAgent features, or contact Dynatrace if this is blocking root cause analysis
**2. Running for >90 minutes:**
- Symptom: Trace stops receiving new data after 90 min
- Cause: Resource protection limit
- Fix: Investigate why transaction is running so long; may indicate hung process
**Adaptive traffic management** also reduces sampling under high load (some traces won't be captured).
 
**Rule:** Check trace diagnostic messages. If consistent truncation, investigate custom service count.
 
**Reference:** Distributed Tracing and PurePath section in reference guide.
 
---
 
### Q: How do I correlate logs to traces?
 
**A:** Automatic if OneAgent is installed:
 
1. OneAgent injects trace ID into application logs
2. Both appear in Grail with matching `trace_id` field
3. In trace view, select "Logs" tab to see correlated logs
4. In log view, click `trace_id` value to jump to trace
**Requirements:**
- OneAgent must be monitoring the service
- Logging framework must support MDC/context propagation (Java, .NET, Node.js, Python, Go, PHP auto-supported)
**If manual instrumentation:**
```
Inject trace_id from span context into every log record
Example (pseudocode):
  span = tracer.current_span()
  logger.info(f"User action completed", trace_id=span.context.trace_id)
```
 
**Reference:** Distributed Tracing and PurePath section in reference guide.
 
---
 
### Q: What's the difference between PurePath and OpenTelemetry traces in Dynatrace?
 
**A:** Both appear in Dynatrace, but PurePath has richer context:
 
| Aspect | PurePath (OneAgent) | OpenTelemetry |
|--------|-------------------|----------------|
| **Auto-instrumentation** | Yes (code injection) | No (manual or library) |
| **Code-level visibility** | Yes (methods, lines) | Limited (spans only) |
| **Topology enrichment** | Full (Smartscape context) | Manual (if provided) |
| **Profiling data** | Yes (CPU, network, wait time) | No |
| **Sampling** | Adaptive (Dynatrace decides) | User-configured |
 
**Best practice:** Use OneAgent for auto-instrumentation + code visibility. Use OpenTelemetry for languages/platforms OneAgent doesn't support or for seamless multi-vendor traces. Dynatrace enriches both.
 
**Reference:** Distributed Tracing and PurePath section; OpenTelemetry Integration section in reference guide.
 
---
 
## Performance & Overhead
 
### Q: How much CPU/memory does OneAgent use?
 
**A:** Typical overhead:
 
- **CPU:** 1-5% depending on workload, technology stack, instrumentation depth
- **Memory:** Minimal; configurable per module
- **Network:** Outbound only; data aggregated before sending
**Factors affecting overhead:**
- More services = more instrumentation = higher CPU
- More custom spans = higher overhead
- Database monitoring enabled = adds tracing overhead
- Real User Monitoring enabled = JavaScript injection and analysis
**Rule:** If OneAgent overhead >5%, check if excessive custom services or feature flags are enabled.
 
**Reference:** Performance and Sizing section in reference guide.
 
---
 
### Q: How should I size ActiveGate?
 
**A:** Rough sizing:
 
- **Baseline:** ~100 OneAgents per 2 CPU / 4 GB RAM
- **Scale linearly:** 200 agents = 4 CPU / 8 GB RAM
- **Headroom:** Keep ActiveGate <50% CPU, <80% memory during normal load
- **High availability:** Deploy 2+ ActiveGates in load-balanced config for large environments
**Example:** 500 OneAgents = 10 CPU / 20 GB RAM (5 × baseline), plus ~10% headroom.
 
**Reference:** ActiveGate Sizing section in reference guide.
 
---
 
## Troubleshooting & Diagnostics
 
### Q: OneAgent is running but I see no data. Where do I look?
 
**A:** Follow the decision tree in Troubleshooting: Tree 1 (OneAgent Not Reporting Data).
 
**Quick checklist:**
1. Is OneAgent process running? (ps aux, tasklist)
2. Can it reach Dynatrace/ActiveGate on network? (telnet, curl)
3. Does it have elevated privileges? (Required for full-stack)
4. Are there errors in OneAgent logs? (/var/log/dynatrace/, Event Viewer, docker logs)
**Reference:** Troubleshooting Indicators section in reference guide.
 
---
 
### Q: DQL query is timing out. How do I optimize it?
 
**A:** See `dynatrace_dql_reference.md`, Part 3 (Performance and Troubleshooting) — it has the full optimization checklist, decision tree, and follow-up questions. That file is also authoritative for DQL syntax and Smartscape topology queries.
 
---
 
## Billing & Licensing
 
### Q: What's the cost model for Dynatrace?
 
**A:** Consumption-based pricing with multiple units:
 
- **Full-Stack monitoring:** By host/container monitored
- **Infrastructure monitoring:** By host
- **Digital Experience monitoring:** By user session
- **Application Security:** By container/process group
- **Log ingest:** By GB ingested
- **Custom events/metrics:** Per data point
**Pricing varies by region and contract terms. Contact Dynatrace sales for specific quote.**
 
**Key takeaway:** Monitor intentionally; avoid monitoring unnecessary hosts or enabling features you don't need.
 
---
 
### Q: How do I reduce my Dynatrace bill?
 
**A:** Common optimization strategies:
 
1. **Scope OneAgent** to specific host groups (don't over-monitor)
2. **Disable unnecessary features** (e.g., Real User Monitoring, Application Security if not needed)
3. **Adjust log retention** (short retention = lower cost)
4. **Use sampling** for non-critical data (DQL queries, custom events)
5. **Monitor only production** if dev/test monitoring isn't justified
**Reference:** Route detailed billing questions to Dynatrace sales/account team.
 
---
 
## Bindplane & Bluebox (Adjacent Products)
 
**Staleness caution:** Both products move faster than the rest of this FAQ. Bindplane was acquired in April 2026 and Bluebox launched only weeks before this file's verification date. Treat the answers below as orientation only; verify anything integration-specific against the dedicated reference files or live docs before a customer-facing answer.
 
### Q: What is Bindplane, and how does it relate to Dynatrace?
 
**A:** Bindplane is an OpenTelemetry-native telemetry pipeline company Dynatrace acquired in April 2026. It collects, processes, and routes logs, metrics, and traces at or near the point of collection, before that data reaches any backend, and can route to Dynatrace, Datadog, Splunk, Elastic, or other destinations at once.
 
**Key point:** It's a separate product with its own console, tokens, and documentation site (docs.bindplane.com). It exports to Dynatrace via OTLP the same way any external OpenTelemetry source does; no special connector required.
 
**Don't assume:** Unified login, shared billing, or in-Dynatrace-UI pipeline configuration. These are stated roadmap directions, not confirmed current behavior.
 
**Reference:** Bindplane reference guide for architecture, setup, and troubleshooting.
 
---
 
### Q: What is Bluebox, and how does it relate to Dynatrace?
 
**A:** Bluebox is a Dynatrace-built (not acquired) AI SRE agent. It detects production issues from live OpenTelemetry telemetry, investigates root cause through your service topology, and hands the result to a coding agent or human as an evidence-backed GitHub Issue.
 
**Key point:** Despite being Dynatrace-owned, Bluebox has its own separate workspace, OTLP ingest endpoint, and tokens. Bluebox findings do not appear in the Dynatrace UI, and Bluebox investigations do not query Grail directly.
 
**Don't assume:** Unified investigation experience between Dynatrace and Bluebox without fresh verification.
 
**Reference:** Bluebox reference guide for setup, CLI usage, and architecture.
 
---
 
### Q: Is "the pipeline" or "the agent" Bindplane/Bluebox, or core Dynatrace?
 
**A:** Ambiguous by default; ask which product before answering.
 
- "The pipeline" or "the collector" could mean Bindplane's BDOT collector, Dynatrace OneAgent, or Dynatrace OpenPipeline. These are different products at different layers. See the OpenPipeline vs. Bindplane FAQ entry above for the architectural distinction.
- "The agent," "investigations," or "findings" could mean Bluebox or Dynatrace's built-in Davis AI. These are related but have separate logins and data models.
**Rule:** Confirm the product before troubleshooting; the diagnostic paths don't overlap.
 
---
 
## Miscellaneous
 
### Q: How often does Dynatrace release new features?
 
**A:** Every 2 weeks. OneAgent/ActiveGate auto-update by default (can be deferred manually).
 
---
 
### Q: Can I use Dynatrace for compliance (PCI, HIPAA, SOC 2)?
 
**A:** Yes. Dynatrace SaaS includes:
- Audit logging (all config changes, access logged)
- Data encryption in transit (TLS 1.2+) and at rest
- Sensitive data masking (PII detection + masking rules; built-in rules for card numbers and email addresses are **off by default** in paid environments and must be explicitly activated)
- Role-based access control (SAML, user groups)
**Get formal compliance documentation from Dynatrace sales/trust team.**
 
**Reference:** Data Security and Compliance section in reference guide.
 
---
 
### Q: How do I configure sensitive data masking (PII, credit cards, etc.)?
 
**A:** Dynatrace applies masking at three possible levels, pick based on where the data needs to stop:
 
| Level | When it applies | Where to configure |
|-------|-----------------|---------------------|
| **At capture** | Before data ever leaves the monitored environment | OneAgent: Settings > Log Monitoring > Sensitive data masking (for logs); Settings > Preferences > Data privacy > OneAgent-side masking (URLs, exceptions) |
| **At ingest/storage** | After data reaches Dynatrace, before it's written to Grail | OpenPipeline processors (DQL-based transform rules); required for non-OneAgent sources like external log shippers or OTLP |
| **At display** | Data stored in original form, access-restricted | Requires "View sensitive request data" permission to unmask; everyone else sees `*****` |
 
**For logs specifically:**
- Rules use a regex search expression + masking type (replace with string, or SHA-256 hash)
- Can be scoped at host, host group, or environment level
- Rules execute top-to-bottom; max 256 masking objects per scope; 10-second combined execution time limit per log source
**For RUM/end-user data:**
- Settings > Preferences > Data privacy > IP masking (controls end-user IP and GPS masking specifically, separate from OneAgent-side masking)
**For OpenTelemetry pipelines:**
- Use the OTel Collector's `transform` processor for partial redaction (e.g., mask last octet of an IP) or `redaction` processor for full-value redaction by pattern
**Best practice:** Combine at-capture and at-ingest masking for defense in depth. At-capture ensures data never leaves your environment; at-ingest catches anything from channels that bypass OneAgent.
 
**Reference:** Data Privacy section in reference guide.
 
---
 
### Q: How do I export or share Dynatrace data?
 
**A:** Options:
 
1. **Dashboards:** Export as PDF
2. **Notebooks:** Export DQL queries, share links
3. **Data:** Query via DQL → export to CSV/JSON
4. **Reports:** Use Dynatrace reporting or query Grail directly
5. **API:** Fetch data programmatically via Metrics/Problems/Entities APIs
**Reference:** API Quick Reference file for endpoint details.
 
---
 
### Q: Can I integrate Dynatrace with [tool]?
 
**A:** Dynatrace integrates with:
 
- **Incident management:** PagerDuty, Opsgenie, VictorOps
- **Chat:** Slack, MS Teams, Discord
- **ITSM:** Jira, ServiceNow
- **Cloud platforms:** AWS, Azure, Google Cloud (native integrations)
- **Observability:** Prometheus, Grafana, ELK (data export via API)
**Check Dynatrace Hub for full list of integrations and extensions.**
 
---
 
## When to Escalate
 
If the answer isn't in this FAQ or reference guide, escalate to:
 
1. **Pre-sales:** Licensing, feature availability, architecture consulting
2. **Support:** Bugs, configuration issues, non-standard setups
3. **Professional Services:** Custom integration, large-scale deployment
**Indicators to escalate:**
- Repeated diagnostic messages despite troubleshooting
- Data loss (missing traces/logs across many requests)
- Performance degradation of OneAgent/ActiveGate
- Unresolved API or authentication issues
 

