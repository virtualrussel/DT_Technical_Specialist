# Dynatrace Technical Reference Guide
 
> **Last verified against docs.dynatrace.com:** July 15, 2026
> **Staleness policy:** Dynatrace ships new features roughly every 2 weeks. Treat anything in this file older than ~90 days, or anything version-specific (Operator versions, Kubernetes/OpenShift support windows, API scopes, UI navigation paths), as a starting hypothesis rather than fact. Before relying on it for a customer-facing answer, verify against docs.dynatrace.com, dynatrace.com/blog, or the API Explorer. If the live source contradicts this file, the live source wins; flag the discrepancy rather than silently defaulting to whichever came first in context.
 
## Core Architecture
 
### OneAgent
- Single OneAgent process per host collects all monitoring data (metrics, traces, logs, topology).
- Comprises multiple specialized code modules: Java, .NET, Python, Node.js, Go, PHP, C/C++, CICS, etc.
- **Automatic discovery and instrumentation**: OneAgent discovers running processes and activates instrumentation based on what it finds.
- **Outbound communication only**: OneAgent always initiates communication to Dynatrace Cluster or ActiveGate. Dynatrace never initiates inbound communication.
- **Code injection**: OneAgent injects itself into processes to enable code-level tracing and diagnostics.
- **Real User Monitoring (RUM)**: OneAgent injects JavaScript tags into HTML pages for web RUM.
### ActiveGate
- Optional proxy between OneAgent and Dynatrace Cluster.
- Provides: data forwarding, environment connectivity, synthetic monitoring execution, local ingest endpoints.
- **Default port**: 9999 for environment ActiveGate.
- **Architecture**: x86-64 supports full functionality; other architectures (ARM64, s390x) have partial support.
- **Scaling**: Size ActiveGates based on number of connected OneAgents (roughly 100 agents per 2 CPU / 4GB RAM baseline).
### Dynatrace Environment
- SaaS multi-tenant architecture: each customer gets a dedicated environment on AWS with individual domain.
- **Data storage**: Grail (unified data lakehouse) stores all observability data at rest.
- **Data in transit**: All communication encrypted with TLS 1.2+ (SSL Labs Grade A+).
- **High availability**: 99.5% SLA with clustered architecture, multiple availability zones, automatic failover.
- **Update cadence**: New features delivered every 2 weeks; OneAgent/ActiveGate auto-updates supported.
---
 
## Data Collection and Storage
 
### Grail (Unified Data Lakehouse)
- Centralized repository for all platform data: traces, metrics, logs, events, user experience data, business events.
- **On-read schema**: Data doesn't require pre-defined schema; schema is applied during query execution.
- **Data partitioning**: Grail samples data on write; fetch queries can specify sampling ratio (1, 10, 100, 1000, 10000).
- **Query language**: Dynatrace Query Language (DQL) for ad-hoc analysis.
- **Data retention**: Policy-based; default varies by data type (e.g., traces 10 days, metrics per retention tier).
### DQL (Dynatrace Query Language)
- Read-only, pipe-based query language (`|`) against Grail data.
- Covers core syntax and commands, data sources (logs, spans, events, entities, metrics), Smartscape topology/relationship queries, and performance/troubleshooting.
- **See `dynatrace_dql_reference.md` for the full, authoritative treatment** of all of the above; that file consolidates what used to be split across this section, Troubleshooting Tree 6, and the FAQ's DQL entry.
---
 
## Distributed Tracing and PurePath
 
### PurePath Technology
- Patented distributed tracing with automatic end-to-end visibility without instrumentation changes.
- Captures traces across devices, operating systems, page actions, code methods.
- **Core capabilities**:
  - Code-level visibility down to method execution.
  - Topology context via Smartscape.
  - Metadata enrichment (user experience, business context).
  - Profiling data (CPU, network, waiting time per span).
### Spans and Traces
- **Span**: Record of interaction between a request and a service; contains name, timestamps, attributes, parent span ID.
- **Trace**: Sequence of spans identified by trace ID; traverses at least one service (typically multiple in microservices).
- **Trace retention**: 10 days default for detailed trace data; 90-minute timeout for running traces; 90-minute timeout triggers trace discontinuation.
- **Trace truncation**: Applied when resource limits exceeded; reduced if too many nodes, too many dependencies, or extended duration.
### PurePath Trace Analysis
- **Waterfall chart**: Visualize request sequence and service dependencies; colors/positions show synchronous vs. asynchronous execution.
- **Execution breakdown**: Time distribution across trace segments.
- **Code-level diagnostics**: Drill into affected code at method/line level.
- **Error analysis**: Identify exact location of error in call tree.
- **Log correlation**: Logs in trace context show which service generated each log entry.
### Limitations
- **Node limits**: OneAgent limits each trace to ~3000 span nodes to protect resources; excessive spans trigger truncation and a diagnostic message. This is a per-trace limit, distinct from the separate 3,000-traces-per-view limit (see "Distributed Traces View" under Common Feature Boundaries and Limitations).
- **Data availability**: Older data beyond 10 days is not available in the detailed trace view.
- **Diagnostic messages**: Indicate data loss, correlation failures, or capture issues (e.g., "Trace truncated due to node limit").
---
 
## Dynatrace API v2
 
### Authentication
- **OAuth 2.0 tokens** required for all endpoints.
- **Scopes**: Tokens grant specific permissions (e.g., `storage:events:read`, `metrics.read`, `documents:read`).
- **Token creation**: Go to Access Tokens; select specific scopes per use case.
- **Authorization header**: `Authorization: Api-Token <your-token>`.
- **API Explorer**: Access via `https://{environment-id}.live.dynatrace.com/rest-api-doc/`.
### Key Endpoint Categories
 
#### Metrics API v2
- **GET /api/v2/metrics**: List metric descriptors (metadata about available metric keys).
- **GET /api/v2/metrics/query**: Query metric time series data points.
- **POST /api/v2/metrics/ingest**: Push custom metric data points.
- **Metric selector syntax**: `builtin:host.cpu.usage`, `custom:myapp.requests`.
- **Entity selectors**: Target specific hosts, services, databases: `type(HOST),tag("prod")`.
- **Transformations**: Aggregation (avg, sum, min, max), filters, arithmetic operations.
#### Logs API v2
- **POST /api/v2/logs/ingest**: Ingest log records via ActiveGate or direct endpoint.
- **Query via DQL**: `fetch logs | filter loglevel == "ERROR"`.
- **OneAgent local endpoint**: `http://localhost:14499/v2/logs/ingest` (requires EEC enabled; default port 14499).
#### Events API v2
- **POST /api/v2/events/ingest**: Ingest custom operational/infrastructure events (e.g., deployments, configuration changes). Not the same as business events, see Business Events API below.
- **Endpoints**: ActiveGate (https://{ag-host}:9999/api/v2/events/ingest) or OneAgent EEC endpoint.
- **Content-Type**: application/json.
#### Business Events API
- **POST /api/v2/bizevents/ingest**: Ingest business-context events (order placed, checkout completed, etc.); distinct from the operational Events API v2 above.
- **Scope**: `bizevents:ingest`.
- **Billing note**: Ingestion consumes Davis Data Units (DDUs) from the logs/business-events pool.
#### OpenTelemetry (OTLP) Endpoints
- **Base URL**: `https://{environment-id}.live.dynatrace.com/api/v2/otlp`.
- **Signal paths**: `/v1/traces`, `/v1/metrics`, `/v1/logs`.
- **Format**: Binary Protocol Buffers (not JSON).
- **Authentication**: API token in Authorization header.
- **ActiveGate endpoint**: `http://{activegate-host}:9999/api/v2/otlp/v1/...`.
#### Entity v2 API
- **GET /api/v2/entities**: List monitored entities (hosts, services, databases, applications).
- **Entity types**: `HOST`, `SERVICE`, `SERVICE_INSTANCE`, `DATABASE_SERVICE`, `PROCESS_GROUP`, `APPLICATION`.
- **Entity selector syntax**: `type(SERVICE),tag("environment:prod")`.
- **2000 character limit** on entity selector string length.
#### Problems API
- **GET /api/v2/problems**: Query detected problems (anomalies, errors, performance degradation).
- **Filtering**: By entity type, severity, status, time range.
#### Audit Logs API
- **GET /api/v2/auditlogs**: Retrieve configuration changes and access logs for compliance.
### Rate Limits
- API calls subject to environment-level rate limiting; consult Environment API documentation for current limits.
---
 
## OneAgent Deployment Modes
 
### Full-Stack Monitoring
- Collects infrastructure (host, OS, process), application, and user experience data.
- **Requirements**: Full OneAgent installation with elevated privileges.
- **Supported platforms**: Windows, Linux, AIX, Solaris, macOS, containerized environments.
- **Not supported**: Alpine Linux (musl libc) for classic full-stack; cloud-native alternative available.
### Cloud-Native Full-Stack
- OneAgent in containers via Dynatrace Operator (Kubernetes, OpenShift).
- Supports infrastructure and application observability in containerized environments.
- **Limitations**: Auto-injection disabled for non-containerized processes; auto-update of modules disabled.
### Application-Only Monitoring
- Minimal agent footprint; application tracing and RUM only.
- **Used for**: Cloud Foundry, Heroku, AWS Lambda, Google App Engine, serverless platforms.
- **Universal injection**: Code-module injection used when auto-injection unavailable.
### Kubernetes / OpenShift
- **Deployment**: Dynatrace Operator manages OneAgent rollout.
- **Architecture support**: Kubernetes 1.25+; OpenShift 4.16+.
- **Limitations**: s390x support requires RHEL-based nodes (CoreOS nodes support only cloud-native mode on OpenShift 4.19+).
---
 
## Dynatrace Operator Deep Dive (Kubernetes/OpenShift)
 
### Core Components
- **Dynatrace Operator**: Manages lifecycle of Dynatrace components in-cluster; single replica typical (leader election handles failover).
- **Webhook**: Mutates Pod definitions to inject code modules; mutates namespaces to enable monitoring (label-based).
- **CSI Driver**: Provides code modules to application Pods via shared node-level storage, minimizing redundant downloads. Without CSI, each Pod downloads its own code module copy (100 Pods = 100 downloads); with CSI, code modules download once per node.
- **DynaKube**: Custom Resource (CR) that defines and configures the Operator's monitoring behavior (deployment mode, namespaces, ActiveGate settings, etc.).
### Deployment Modes via DynaKube
- **cloudNativeFullStack**: Infrastructure + application observability in containers.
- **applicationMonitoring**: Application-only injection (no host-level monitoring).
- **hostMonitoring**: Infrastructure-only monitoring (no code injection).
- **Kubernetes Platform Monitoring**: Cluster topology, node/container metrics, Kubernetes events (does not require OneAgent; can be combined with other modes).
### Common Configuration Notes
- Metadata enrichment is automatically enabled when OneAgent injection is configured for a namespace (as of recent Operator versions); no separate flag needed.
- Each namespace should be assigned to only one DynaKube for injection; assigning multiple causes conflicts.
- OneAgent (classic full-stack) and webhook injection cannot coexist in the same Pod (both attempt to mount `/var/lib/dynatrace`).
### Troubleshooting Commands
```bash
# Check DynaKube status
kubectl get dynakubes -n dynatrace
 
# Check pod status (Operator, OneAgent, CSI driver)
kubectl -n dynatrace get pods
 
# Inspect specific pod logs
kubectl logs <pod-name> -n dynatrace
 
# Describe a resource for detailed status/events
kubectl describe <resource-type> <resource-name> -n dynatrace
 
# Run built-in troubleshoot subcommand (checks environment reachability, image pull access, etc.)
kubectl exec deploy/dynatrace-operator -n dynatrace -- dynatrace-operator troubleshoot
 
# Generate a full support archive for escalation to Dynatrace support
kubectl exec -n dynatrace deployment/dynatrace-operator -- dynatrace-operator support-archive --stdout > operator-support-archive.zip
```
 
### Known Constraints
- Dynatrace Operator follows major.minor.patch versioning; minor releases roughly every 2-3 months. Latest 3 versions are tested against latest Kubernetes/OpenShift releases.
- Full support lasts until a Kubernetes/OpenShift version reaches end of life, followed by ~1 year of maintenance support (reduced testing cadence, bug fixes backported by severity).
- Recommend running the latest patch version of the Operator; new features increment the minor version.
---
 
## Monitoring Scopes and Capabilities
 
### Application Observability
- **Service detection**: Automatic identification of services; v1 (classic) and v2 (enhanced) modes.
- **Endpoint metrics**: Response time, throughput, failure count per API endpoint.
- **Enhanced endpoints (SDv1)**: Auto-detection of endpoints without key request configuration.
- **Request attributes**: Custom metadata captured from requests (e.g., user ID, region, transaction type).
### Infrastructure Observability
- **Host metrics**: CPU, memory, disk, network.
- **Process group monitoring**: Process-level resource usage and dependencies.
- **OS module**: Required for out-of-the-box infrastructure alerting.
- **Network monitoring**: Process-level network topology and health.
### Database Monitoring
- **Automatic detection**: JDBC, .NET, Node.js, Python database clients.
- **Supported databases**: Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, DynamoDB, etc.
- **Metrics**: Connection pooling, query performance, transaction time.
### Real User Monitoring (RUM)
- **JavaScript injection**: OneAgent auto-injects RUM tags into the `<head>` of HTML pages; only modifies HTML content (not images, CSS, JSON, XML, or plain text).
- **Injection requirements**: Requires valid, well-formed HTML (unclosed/missing tags are the most common cause of injection failure).
- **Agentless RUM**: Manual JavaScript insertion when OneAgent can't be installed on the web tier (e.g., static CDN-hosted pages); no automatic updates unless using the JS tag snippet format.
- **Cache control header optimization**: OneAgent modifies cache headers (ETag, Last-Modified) so that RUM configuration changes take effect without waiting for full cache expiration.
- **Health check tooling**: Application health check page shows injection attempt counts, failure ratio, and top failing URLs (first troubleshooting stop for missing/partial RUM data).
- **Session Replay**: Video-like replay of user sessions (opt-in feature, off by default).
- **User journey analytics**: Trace user actions across pages and services.
### Synthetic Monitoring
- **Browser monitors**: Simulate multi-step user journeys (clickpaths) in a real browser; capture waterfall, screenshots, and Web Vitals.
- **HTTP monitors**: Lightweight API/endpoint availability and response validation checks (no browser rendering).
- **Locations**: Run from Dynatrace-managed public locations or customer-hosted private synthetic locations (private location requires its own ActiveGate).
- **Availability SLIs**: `builtin:synthetic.browser.availability.location.total` (with a maintenance-window-excluded variant) used as the standard synthetic availability metric for SLOs.
- **Scheduling**: Configurable execution frequency per monitor; supports maintenance windows to suppress alerting during planned downtime.
### Application Security
- **Runtime vulnerability detection**: Identifies exploitable vulnerabilities in runtime.
- **Code-level risk analysis**: OWASP Top 10, CWE violations.
- **Behavioral threat detection**: Unusual runtime behavior (data exfiltration, privilege escalation).
- **Integrated into DevOps**: Alerts in CI/CD pipelines; prioritized by real-world attack surface.
---
 
## OneAgent Platform Support
 
### Supported Runtimes
- **Java**: Any version supported for code instrumentation.
- **.NET**: .NET Framework, .NET Core 3+.
- **Node.js**: v12+.
- **Python**: 3.8+.
- **Go**: 1.9+.
- **PHP**: 5.6+.
- **C/C++**: Via OneAgent SDK for custom tracing.
### Supported Operating Systems
- **Windows**: Server 2012+; both 64-bit x86 and ARM (AWS Graviton2).
- **Linux**: RHEL, CentOS, Debian, Ubuntu; both x86 and ARM64.
- **Alpine Linux**: Container-only support (musl libc); not supported for native installations or classic full-stack.
- **IBM i, AIX, Solaris**: Supported via universal injection for application-only monitoring.
- **macOS**: Supported.
### Supported Container Platforms
- **Docker**: Standalone and swarm.
- **Kubernetes**: 1.25+ with Dynatrace Operator.
- **OpenShift**: 4.16+ (4.19+ for s390x).
- **AWS Fargate**: Application-only via CloudWatch integration.
- **Google Cloud Run**: With restrictions on execution environments.
- **AWS Lambda**: Integrated tracing for Lambda functions.
- **Cloud Foundry**: Buildpack integration.
### Limitations by Architecture
- **x86-64**: All OneAgent features supported.
- **ARM64 (AArch64)**: Supported as of OneAgent 1.189.
- **s390x**: Full OpenShift support; limited architecture support for other platforms.
- **PowerPC (PPCLE)**: Supported as of OneAgent 1.169; partial feature support.
---
 
## Extension and Custom Instrumentation
 
### OpenTelemetry Integration
- **Native support**: Ingest OpenTelemetry spans, metrics, logs alongside OneAgent data.
- **Enrichment**: OpenTelemetry data enriched with Dynatrace topology and context.
- **Endpoint**: Standard OTLP endpoints, HTTP/protobuf only. Dynatrace does not accept gRPC directly; a source emitting gRPC must be routed through an OpenTelemetry Collector to convert to HTTP/protobuf first.
### OneAgent SDK
- **C/C++**: Custom tracing for unsupported languages via SDK.
- **Node.js**: Custom transaction/span creation.
- **Python**: Custom instrumentation for libraries not auto-detected.
- **Java**: JMX extensions for monitoring JMX metrics from applications.
### Extension Execution Controller (EEC)
- **Local ingest endpoints**: OneAgent exposes ports for metrics, logs, events (default port 14499).
- **No authentication**: Local endpoints do not require API tokens.
- **Reliability mechanism**: Persistent queue for log ingest; 2136 MB disk space required.
- **Persistence configuration**: Configurable via extensionsuser.conf.
### Custom Metrics
- **Ingestion**: Push via Metrics API v2 or local EEC endpoint.
- **Naming**: `custom:*` prefix; no pre-registration required.
- **Dimensions**: Key-value pairs (must be alphanumeric); reserved dims (dt.entity.*).
- **Usage**: Available in dashboards, alerts, service-level objectives (SLOs).
---
 
## Alerting and Problem Management
 
### Problem Detection
- **Automatic baseline detection**: Dynatrace Davis AI establishes baselines and detects anomalies.
- **Multi-dimensional analysis**: Problems correlated across metrics, traces, events, topology.
- **Severity levels**: Critical, High, Medium, Low.
- **Diagnostic messages**: Traces include diagnostic info (e.g., truncation, correlation issues).
### Metric Events
- **Threshold-based alerts**: Static thresholds on builtin or custom metrics.
- **Dynamic baselines**: Anomaly detection based on learned patterns.
- **Comparison operators**: Equals, less than, greater than, in range.
### Event Subscription (Automation)
- **Problem notifications**: Post to Slack, PagerDuty, Jira, webhooks.
- **Custom workflows**: Trigger remediation actions via API.
- **Throttling**: Deduplicate notifications by entity and problem type.
---
 
## Automation and Workflows (AutomationEngine)
 
### AutomationEngine
- **Purpose**: Process orchestrator combining observability, security, and business data with causal AI to automate BizDevSecOps use cases (notifications, remediation, quality/security gating, vulnerability response).
- **Not a data pipeline**: AutomationEngine/Workflows is not intended for mass data ingestion or export; use OpenPipeline or Extensions for large-scale data processing.
- **Core concepts**: A **workflow** assembles **tasks** in series, parallel, or conditional paths. Each **execution** is a single run of a workflow, triggered manually, via API, on a schedule, or by an event (e.g., a Davis problem).
### Triggers
- **On-demand**: Manual trigger via UI or API.
- **Schedule**: Fixed time, interval, or cron-based (time zone aware).
- **Event-based**: Fires on Davis problems/events, generic Grail events, business events, or security events. Event triggers are capped at 1,000 firings/hour per workflow; exceeding this 3 times in 7 days auto-disables the trigger.
### Permissions
- Requires AppEngine + AutomationEngine-specific permissions (e.g., `automation:workflows:write`, `automation:workflows:admin`).
- A **simple workflow** (single task, no JavaScript) doesn't consume workflow hours and can be granted to a broader user base without cost implications.
- First run of a workflow prompts the user to authorize AutomationEngine to act on their behalf; a 403 error usually indicates missing authorization or permissions.
### Billing
- Unit of measure: **workflow hours** (one per hour a configured workflow exists, since creation). Simple and draft-only workflows don't consume workflow hours.
- Each task execution consumes an AppEngine function invocation, billed separately.
---
 
## Service-Level Objectives (SLO)
 
### Core Concepts
- **SLI (Service-Level Indicator)**: The measured value (e.g., success rate, response time, availability).
- **Target**: The goal for the SLI (e.g., 99.9% of requests succeed).
- **Evaluation period**: The time window the SLO status is calculated over (e.g., rolling 7 days).
- **Error budget**: Remaining allowable "badness" before breaching the target; burn rate measures how fast that budget is being consumed relative to the evaluation period.
### SLO Types
- **Metric-based**: Pre-aggregated metrics; fastest evaluation, lowest overhead.
- **Request-based (event-based)**: Ratio of "good" events to total events (e.g., successful requests / total requests); automatically adjusts to load variation.
- **Time-slice-based (window-based)**: Classifies each time interval as "good" or "bad" against a threshold; SLI = percentage of good time slices.
### Configuration
- Built via SLO wizard using out-of-the-box templates (service availability, service performance, synthetic availability, host CPU, etc.) or a **custom DQL query** (must return an `sli` field as an array of doubles).
- SLOs can be scoped to a management zone; unscoped SLOs are visible to all users.
- Burn-rate alerts can be created directly from an SLO and appear as metric events.
- SLOs can be pinned to dashboards as tiles showing status, error budget, and target.
### API Management
- SLOs can be created, edited, listed, deleted, and evaluated via the SLO Service Public API, useful for CI/CD gating (e.g., blocking a release if error budget is exhausted).
---
 
## Smartscape and Topology
 
### Smartscape
- **Real-time topology model**: Automatically maps applications, infrastructure, cloud services, dependencies.
- **Entity types**: Hosts, processes, services, service instances, databases, external services, applications.
- **Relationship mapping**: 1:n and n:1 relationships (e.g., service runs_on host, host contains processes).
- **On Grail transition**: Version 1.334+ recommends migration from classic entity model to Smartscape on Grail (dt.entity.* views).
### Service Dependency Mapping
- **Automatic discovery**: No manual configuration; OneAgent discovers dependencies.
- **Call paths**: Synchronous and asynchronous communication, queuing.
- **External services**: Outbound calls to third-party APIs, databases, message queues.
- **Visualization**: Service flow diagram shows request paths and bottlenecks.
---
 
## Data Security and Compliance
 
### Encryption
- **In transit**: TLS 1.2+ (all OneAgent ↔ Cluster/ActiveGate communication).
- **At rest**: Grail data stored on dedicated AWS S3 bucket per customer.
- **Protocol**: Google Protocol Buffers for serialization.
### Access Control
- **Role-based access**: User groups and permissions; SAML integration.
- **Token scopes**: Fine-grained permissions per API use case.
- **Audit logging**: All configuration changes, user access, and API calls logged; downloadable via audit log API.
### Data Privacy
- **Three levels of protection**: Masking at **capture** (OneAgent, before data leaves the monitored environment), at **storage/ingest** (OpenPipeline processors, transform before writing to Grail), and at **display** (data stored in original form but restricted to users with the "View sensitive request data" permission).
- **Log masking configuration**: Settings > Log Monitoring > Sensitive data masking. Rules can be scoped at host, host group, or environment level; each rule uses a regex search expression and a masking type (replace with string, or SHA-256 hash). Rules execute top-to-bottom in the order listed; max 256 masking objects per scope; a 10-second combined execution time limit per log source (source is disabled if exceeded).
- **OneAgent-side masking (non-log)**: Settings > Preferences > Data privacy > OneAgent-side masking (covers URLs and exception messages captured outside of Log Monitoring).
- **RUM/end-user data masking**: Settings > Preferences > Data privacy > IP masking (controls masking of end-user IP addresses and GPS coordinates; separate from OneAgent-side masking for URLs).
- **OpenPipeline/ingest-side masking**: For log/event/data sources outside OneAgent (e.g., external log shippers, OTLP), configure a DQL-based processor in OpenPipeline to mask or redact fields after ingest but before storage.
- **OpenTelemetry Collector masking**: Use the Collector's `transform` processor (partial redaction, e.g., masking part of an IP) or `redaction` processor (full-value redaction by pattern match) before exporting to Dynatrace.
- **PII detection**: Automatic or rule-based detection and masking; built-in rules exist for common patterns like payment card numbers and email addresses but are deactivated by default in paid environments.
- **Data retention policies**: Configurable per data type; deletion enforced after retention period.
---
 
## Common Feature Boundaries and Limitations
 
### Trace Data
- **Retention**: Detailed trace data retained 10 days; metric retention (see Metric Retention below) can extend well beyond that.
- **Trace timeout**: 90 minutes; traces older than 90 minutes not added to.
- **Node limits**: ~3000 span nodes per trace; excessive spans trigger truncation warnings. Distinct from the 3,000-traces-per-view limit (see Distributed Traces View below).
- **Sampling**: Adaptive traffic management reduces data sent under high load; some traces not available.
### Entity Query
- **Selector limit**: 2000 characters max for entity selector string.
- **Max returned**: 100 entities per query (paginate for larger results).
- **Relationship limit**: 1:n relationships return max 100 entity IDs per record in DQL.
### Metric Retention
- **1-minute resolution**: 7 days.
- **1-hour resolution**: 400 days (approx. 13 months).
- **Custom metrics**: Retention based on subscription tier.
### Log Storage
- **Default bucket**: Logs stored in Grail; retention based on contract.
- **EEC persistence**: 2136 MB required; logs queued locally if ActiveGate unavailable.
### Distributed Traces View
- **Limit**: 3,000 most recent traces per view per timeframe/management zone.
- **Search**: By trace ID, service name, attributes.
- **Data availability**: Toggle "Show data retention" for visibility into retention boundaries.
---
 
## Configuration and Settings
 
### Settings Hierarchy
- **Environment-level**: Applies to all monitored entities unless overridden.
- **Host group level**: Configuration for specific host groups.
- **Host-level**: Host-specific configuration overrides.
### Common Configuration Areas
- **OneAgent features**: Enable/disable specific capabilities (e.g., Database monitoring, .NET tracing).
- **Service detection**: SDv1 vs. SDv2; endpoint naming rules.
- **Request attributes**: Capture custom fields from requests.
- **Sensitive data masking**: Rules for PII.
- **Management zones**: Logical grouping of entities for access control and alerting scope.
- **Alerting policies**: Thresholds, notification rules, integrations.
---
 
## Version and Update Strategy
 
### OneAgent Auto-Update
- **Default**: Enabled; OneAgent updates automatically from Dynatrace Cluster.
- **Frequency**: Multiple times per week as bug fixes and features available.
- **Manual update**: Can be deferred; `oneagentctl` command for version control.
### ActiveGate Auto-Update
- **Enabled by default**: Similar to OneAgent; manual control available via Settings.
- **Zero-downtime upgrades**: Cluster upgrades support zero downtime.
### Managed vs. SaaS
- **SaaS**: Fully managed by Dynatrace; always on latest version.
- **Managed**: Customer-managed cluster; upgrade strategy configurable.
- **Deprecated versions**: Dynatrace supports current and previous major versions typically.
---
 
## Performance and Sizing
 
### OneAgent Overhead
- **Typical impact**: 1-5% CPU; varies by workload and instrumentation depth.
- **Memory**: Minimal; configurable heap size for specific modules.
- **Network**: Outbound only; metrics, traces, logs aggregated before transmission.
### ActiveGate Sizing
- **Rule of thumb**: ~100 OneAgents per 2 CPU / 4 GB RAM.
- **CPU/Memory limits**: Keep ActiveGate below 50% CPU, 80% memory during normal operation.
- **Scaling**: Deploy multiple ActiveGates in load-balanced configuration for high-volume environments.
### DQL Query Performance
- See `dynatrace_dql_reference.md`, Part 3 (Performance and Troubleshooting), for the full decision tree, optimization checklist, and default limits (timeframe, scanLimitGBytes, samplingRatio).
---
 
## Troubleshooting Indicators
 
### Diagnostic Messages in Traces
- **"Trace truncated due to node limit"**: Too many spans in single trace; reduce custom services or enable feature flags selectively.
- **"Missing service correlation"**: Service not properly named; check service detection rules.
- **"Data not received"**: OneAgent communication issue; verify network, ActiveGate connectivity.
### DQL Query Failures
- See `dynatrace_dql_reference.md`, Part 3 (Performance and Troubleshooting), for query timeouts and structure validation. A "permission denied" error is a token-scope issue, not a DQL issue, see the API Quick Reference file's auth decision tree instead.
### OneAgent Issues
- **"Alpine Linux not supported"**: Use cloud-native full-stack on Alpine; classic full-stack requires glibc-based image.
- **"Universal injection failed"**: Ensure process is not running as root with restricted permissions; verify agent installation.
---
 
## Quick Reference: Common Tasks
 
**DQL query examples (service errors, slow database queries, listing services, and all Smartscape/topology queries) now live in `dynatrace_dql_reference.md`.** The two tasks below stay here because they're API-specific, not DQL pipelines.
 
### Ingest custom metric via API
```
curl -X POST "https://{env}.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token {token}" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "custom:myapp.orders {region=us} 150"
```
 
### Get entity selector syntax for services tagged "prod"
```
type(SERVICE), tag("env:production")
```
 
---
 
## Bindplane vs. OpenPipeline: Key Differences
 
**Staleness note:** This comparison is architectural/conceptual (stable). But any claim about Bindplane's *current* capabilities, editions, or integration depth should defer to `dynatrace_bindplane_reference.md`, which carries a shorter staleness window than this file (Bindplane was only acquired in April 2026).
 
Both Bindplane and OpenPipeline transform observability data, but they operate at different layers of the data pipeline. Understanding the difference is critical for choosing the right tool for your use case.
 
| Aspect | **Bindplane** | **OpenPipeline** |
|--------|---------------|-----------------|
| **What it is** | OpenTelemetry-native collection/routing platform (acquired by Dynatrace, April 2026) | Dynatrace's ingest-time data processing layer |
| **Where it operates** | At or near data collection (edge/source) | After data reaches Dynatrace, before storage in Grail |
| **Deployment** | Standalone, can be self-hosted or cloud-managed | Runs inside Dynatrace environment (no separate deployment) |
| **Supported destinations** | Multi-destination (Dynatrace, Datadog, Splunk, Elastic, etc.) | Dynatrace-only |
| **Use case #1** | Cost reduction at source (drop/sample data before it leaves your infrastructure) | Cost reduction at ingest (transform non-critical data after Dynatrace receives it) |
| **Use case #2** | Multi-backend routing (route same telemetry to multiple observability platforms) | Dynatrace-specific shaping (PII masking, field extraction, event normalization) |
| **Use case #3** | Migration between platforms (collect once, route to both old and new backend during transition) | Sensitive data handling (mask credit cards, PII before storage in Grail) |
| **Typical question** | "We're paying for too much log volume. Can we drop low-value logs before sending to the backend?" | "We're sending sensitive data; can we mask it in Dynatrace before someone with limited permissions sees it?" |
 
**Rule of thumb:**
- **Use Bindplane** when your optimization goal is preventing data from leaving your infrastructure, or when you need to route data to multiple backends.
- **Use OpenPipeline** when your optimization is about shaping data once it's in Dynatrace, or when you need Dynatrace-specific transformations like masking, enrichment, or field extraction.
- **Use both** for comprehensive cost and security control: Bindplane drops/samples at source, OpenPipeline masks sensitive fields at ingest.
See the Bindplane reference guide for Bindplane-specific architecture and setup details.
 
---
 
## Glossary
 
| Term | Definition |
|------|------------|
| **OneAgent** | Single per-host agent that auto-discovers and instruments processes, hosts, and applications. |
| **ActiveGate** | Optional proxy/router between OneAgent and the Dynatrace Cluster; also runs synthetic monitors and local ingest endpoints. |
| **Grail** | Dynatrace's unified data lakehouse; stores all observability, security, and business data with an on-read schema. |
| **DQL (Dynatrace Query Language)** | Pipe-based query language used to analyze data stored in Grail. |
| **PurePath** | Dynatrace's patented distributed tracing technology; auto-instrumented, code-level, topology-aware traces captured by OneAgent. |
| **Smartscape** | Real-time topology model mapping hosts, processes, services, and their relationships. |
| **Davis AI** | Dynatrace's causal AI engine; performs automatic baselining, anomaly detection, and root cause analysis. |
| **Dynatrace Intelligence** | The umbrella term for Dynatrace's predictive, causal, and generative AI capabilities across the platform. |
| **AutomationEngine** | Orchestration engine powering Workflows; automates actions in response to observability/security data. |
| **Workflow** | A configured sequence of tasks (with triggers and conditional logic) run by AutomationEngine. |
| **AppEngine** | Platform layer that runs custom Dynatrace apps and powers Workflow task execution (function invocations). |
| **Dynatrace Operator** | Kubernetes/OpenShift controller that manages the lifecycle of OneAgent, ActiveGate, and related components in-cluster. |
| **DynaKube** | The Kubernetes Custom Resource (CR) used to configure Dynatrace Operator's behavior in a cluster. |
| **SLI / SLO** | Service-Level Indicator (the measured value) and Service-Level Objective (the target for that value) used for reliability tracking. |
| **Error budget** | The allowable margin between current SLO status and target before a breach occurs. |
| **EEC (Extension Execution Controller)** | OneAgent-hosted local endpoint for metrics/logs/events ingestion without requiring an API token. |
| **Management Zone** | Logical grouping of entities used to scope user access and alerting. |
| **Entity Selector** | Query syntax used to filter monitored entities (hosts, services, etc.) in APIs and DQL. |
| **RUM (Real User Monitoring)** | Captures real end-user interactions with a web or mobile application via injected JavaScript or native SDK. |
| **Synthetic Monitoring** | Scripted, scheduled checks (browser or HTTP) that proactively test availability and performance, independent of real traffic. |
| **OTLP (OpenTelemetry Protocol)** | Standard wire protocol for exporting traces, metrics, and logs; Dynatrace supports it over HTTP/protobuf (not gRPC, not JSON). |
| **OpenPipeline** | Dynatrace's ingest-time processing layer; used for large-scale transformation, masking, and routing of incoming data before it lands in Grail. |
| **Bindplane** | OpenTelemetry-native telemetry pipeline company acquired by Dynatrace (closed April 15, 2026); operates as a standalone collection-time control plane, separate from OneAgent and OpenPipeline. See the Bindplane reference guide for architecture and setup details. |
| **Bluebox** | Dynatrace-built (not acquired) AI SRE agent; detects production issues from OpenTelemetry data and proposes evidence-backed fixes as GitHub Issues. Operates on its own workspace and data plane, separate from core Dynatrace. See the Bluebox reference guide for architecture and setup details. |
| **DDU (Davis Data Unit)** | Dynatrace's consumption unit for billing certain ingested data (e.g., logs, business events); consumed per ingestion, relevant to cost conversations. |
 

