# Bindplane Technical Reference Guide (Dynatrace Acquisition)
 
> **Last verified against docs.bindplane.com and docs.dynatrace.com:** 2026-08-24
> **Staleness policy, read this before answering any Bindplane question:** This is the newest and fastest-moving file in the project. The acquisition closed April 15, 2026, roughly four months before this file's verification date. Dynatrace has stated it intends to "accelerate Bindplane's roadmap through increased investment and deeper integration with the Dynatrace platform." That integration is explicitly in progress, not finished. Do not assume unified billing, single sign-on with Dynatrace accounts, in-Dynatrace-UI configuration of pipelines, or any other roadmap item has shipped unless you verify it against current docs.bindplane.com, docs.dynatrace.com, or dynatrace.com/blog first. Bindplane retains its own console, documentation site, and (as of this writing) standalone product identity. Treat every claim about "how integrated" the products are as something to verify fresh, every time, regardless of what this file says.
 
---
 
## Acquisition Context
 
- **Signed:** April 8, 2026. **Closed:** April 15, 2026.
- Dynatrace acquired Bindplane, an OpenTelemetry-native telemetry pipeline company, to extend its platform upstream, into how telemetry is collected, shaped, and routed *before* it reaches an observability backend.
- **Bindplane remains a standalone, vendor-neutral product.** Dynatrace has publicly committed to continued multi-destination routing, existing Bindplane customers can keep routing telemetry to Datadog, Splunk, Elastic, or any other destination alongside or instead of Dynatrace. This is a deliberate strategic choice (Bindplane is also positioned as an on-ramp for customers currently on competing platforms), not an oversight to expect Dynatrace to close off.
- Rationale stated by Dynatrace leadership: AI-driven insights are only as good as the telemetry feeding them, and Bindplane closes a data-collection-layer gap that Dynatrace's core platform (OneAgent, Grail, OpenPipeline) did not previously own.
- **Do not present this as a "Dynatrace product" without qualification**: describe it as "Bindplane, now part of Dynatrace" and be precise about what's actually integrated today (see "How This Fits the Broader Dynatrace Platform" below) versus what's roadmap.
---
 
## What Is Bindplane
 
Bindplane is an OpenTelemetry-native telemetry pipeline, a control plane that sits at or near the point of data collection, letting teams collect, process, and route logs, metrics, and traces in real time before that data is ingested into any downstream system.
 
**Core value proposition (per Bindplane's own positioning):**
- Reduce telemetry volume/cost before it hits a backend (commonly cited: up to ~40% log volume reduction)
- Centralize fleet management for potentially thousands to millions of OpenTelemetry Collectors
- Vendor-neutral routing: one pipeline, multiple possible destinations
- Ease migration between observability/SIEM platforms without re-instrumenting
This is conceptually adjacent to, but distinct from, Dynatrace OpenPipeline. OpenPipeline processes data *after* it's ingested into Grail; Bindplane processes/routes data *before* it reaches any backend, at the collection/edge layer. See "How This Fits" below.
 
---
 
## Architecture
 
### Bindplane Server (management/control plane)
- **GraphQL Server:** serves configuration and collector detail queries
- **REST Server:** backs the Bindplane CLI and UI
- **WebSocket Server:** pushes configuration updates to collectors via **OpAMP** (Open Agent Management Protocol)
- **Store:** pluggable storage layer for configuration and collector state
- **Deployment options:**
  - **Bindplane Cloud**: hosted and managed by Bindplane; recommended as the fastest path to start
  - **Self-hosted**: available on Enterprise and Google editions; you run the server in your own infrastructure
### BDOT Collector (data plane)
- **BDOT** = Bindplane Distro for OpenTelemetry. Bindplane's own distribution of the upstream OpenTelemetry Collector, built using the OpenTelemetry Collector Builder (OCB).
- First collector distribution to implement OpAMP for remote management.
- **Two operating modes** (not manually configured, implicit based on what sources are attached):
  - **Agent mode:** collector runs on the host it's monitoring (e.g., a database host, an API server) and collects locally.
  - **Gateway mode:** collector receives telemetry from *other* collectors over the network (e.g., via an OTLP or "Bindplane Gateway" source), optionally does additional processing, and forwards onward. Any source type that receives from multiple remote collectors puts a collector into gateway mode.
- **V1 vs V2:** V1 uses a custom OpAMP manager built into the BDOT Collector. V2 uses the official OpenTelemetry OpAMP Supervisor instead. V2 was in beta as of this file's original verification date; **verify current GA/beta status at docs.bindplane.com before advising a customer to adopt V2 in production**. Confirm which version a customer is running before troubleshooting config-push issues; the mechanism differs.
- You are not limited to the BDOT Collector. Bindplane can manage **custom OpenTelemetry Collector distributions** (built via OCB with your own component manifest) as long as the OpAMP Supervisor is packaged in.
---
 
## Core Configuration Model
 
A Bindplane pipeline is built from these resource types, defined as YAML (`apiVersion: bindplane.observiq.com/v1`):
 
| Resource | Purpose |
|---|---|
| **Sources** | Define where telemetry originates. 100+ supported integrations (hosts, cloud services, applications, other collectors). |
| **Processors** | Transform data in-flight: filter, redact, parse, enrich, convert between formats. Inserted either *after a source* (affects only that source's data) or *before a destination* (affects all data flowing into it). |
| **Processor Bundles / Blueprints** | Pre-built groupings of processors for common transformation use cases; reduces hand-assembling the same processor chains repeatedly. |
| **Connectors** | Bridge pipelines: convert between signal types, derive metrics from traces, or fan telemetry out to multiple downstream pipelines. |
| **Destinations** | Final export targets (observability platforms, cloud storage, SIEMs, etc.). |
| **Extensions** | Collector-level capabilities outside the telemetry pipeline itself, health/profiling endpoints, persistent storage for queue durability across restarts. |
| **Configurations** | Tie sources, processors, and destinations together; applied to collectors via label selectors (`matchLabels`). |
 
**Example minimal custom processor + configuration:**
```yaml
apiVersion: bindplane.observiq.com/v1
kind: Processor
metadata:
  id: custom
  name: custom
spec:
  type: custom
  parameters:
    - name: configuration
      value: |
        resource:
          attributes:
            - action: upsert
              key: custom
              value: true
    - name: telemetry_types
      value: [Metrics, Traces, Logs]
```
 
---
 
## Deployment Patterns
 
- **Agent-only:** collector runs on the host, ships directly to the destination. Simplest, fine for small/simple environments.
- **Agent-Gateway (recommended for HA/scale):** agent collectors run on every host/node; gateway collectors (centralized, horizontally scalable) receive from agents, optionally process, and forward on. This is the standard high-availability pattern.
**Why use gateways:**
- Credential centralization: destination credentials live only on the gateway tier, not every agent, reducing attack surface
- Offload heavy processing from resource-constrained edge agents to a scalable gateway fleet
- Network segmentation: gateways can sit in a DMZ, allowing agents to reach the gateway without granting agents direct internet/backend access
**Sizing guidance:**
- Bindplane nodes should manage no more than ~30,000 collectors each for maximum fault tolerance (each collector opens two connections: one for OpAMP config, one for throughput metrics, so connection count scales 2x collector count).
- Gateway fleets should sit behind a load balancer supporting the network protocols in use (gRPC, HTTP, etc.); this enables horizontal scaling and graceful handling of node failures.
---
 
## Key Platform Features
 
- **Fleets**: manage collectors at scale via grouped configurations
- **Library**: reusable sources, processors, connectors, destinations saved for reuse across configs
- **Progressive Rollouts / Rollouts**: stage, compare, and gradually deploy configuration changes to subsets of collectors before a full rollout
- **Live Preview**: preview pipeline changes in real time before rolling out
- **Snapshots**: view recently collected logs, metrics, and traces samples for a collector (useful for validating a pipeline change or debugging without waiting on the full destination round-trip)
- **Pipeline Intelligence**: AI-assisted pipeline configuration and analysis (e.g., auto log-type identification, parser suggestions)
- **Secrets Management**: manage sensitive values (tokens, credentials) referenced in configurations without embedding them in plaintext
- **SSO**: Okta, Microsoft Entra, and other identity providers
- **RBAC**: role-based user permissions
- **Audit Trail**: tracks configuration changes for compliance/accountability
- **OpAMP Gateway (extension)**: relay extension for downstream OpAMP collectors in multi-tier deployments
---
 
## Editions and Licensing
 
Bindplane's documented plan tiers as of last verification (verify current pricing at bindplane.com before quoting):

| Edition | Hosting | Price | Notes |
|---|---|---|---|
| **Free** | Bindplane Cloud (hosted) | $0/month | Entry tier |
| **Growth** | Bindplane Cloud (hosted) | Starts at $499/month | Mid-tier |
| **Enterprise** | Cloud or self-hosted | Custom, starts at ~$50k/year | Unlocks additional processors/capabilities; self-hosted available |
| **Google Edition** | Cloud | Free for Google Cloud customers | Bundled with Google SecOps Enterprise Plus |

- **Self-hosted** is available for Enterprise and Google editions only.
- **Enterprise Edition**: unlocks additional processors/capabilities not in lower tiers. Bindplane's own docs are non-specific about exactly which processors are Enterprise-gated at any given time and direct customers to `sales@bindplane.com` for current specifics. **Do not guess at which features are Enterprise-only; route pricing/tier questions to sales, same as the parent Dynatrace-licensing rule.**
- **Google Edition**: adds capabilities like PII masking, log deduplication, and a 12-month allowance to route data to non-Google destinations during a SIEM migration. Google SecOps permissions expanded for project-scoped users as of v1.102.0 (August 2026).
- **Consult Bindplane's current Plans & Pricing page before quoting tier boundaries**: this is exactly the kind of fast-moving, customer-remembered detail the parent project instructions already flag as high-risk for staleness.
---
 
## Dynatrace Destination (Bindplane → Dynatrace)
 
**Quick summary:** Bindplane is a collection-time transformation layer. It exports to Dynatrace the same way any OpenTelemetry exporter does, via OTLP. No special Dynatrace connectors needed. From Dynatrace's perspective, Bindplane is just another OTLP data source.
 
Bindplane exports to Dynatrace using **OTLP over HTTP** (gRPC is not supported, same constraint as Dynatrace's native OTLP endpoints documented elsewhere in this project).
 
**Critical endpoint note:** Do not use `.apps.dynatrace.com` URLs in the Dynatrace destination. Only `.live.dynatrace.com` paths are valid for OTLP ingest; the platform-API `.apps` domain returns HTTP 404 with no useful error message.

**Metrics limitation:** Monotonic cumulative sums are not currently supported for metrics via `dynatrace_otlp`.

### Destination type
- **Current:** `dynatrace_otlp`
- **Deprecated:** a destination simply called `dynatrace` still exists and still functions, but receives no further enhancements. **If you see a customer's config using the plain `dynatrace` type, flag it for migration** to `dynatrace_otlp`.
### Deployment Type options (determines the endpoint URL)
| Deployment Type | What you provide | Resulting endpoint |
|---|---|---|
| **SaaS** | Environment ID + API token | `https://{environment-id}.live.dynatrace.com/api/v2/otlp` (derived automatically) |
| **ActiveGate** | ActiveGate hostname, port (default `9999`), Environment ID | Routed through the ActiveGate |
| **Custom** | Full OTLP endpoint URL directly | For Managed or non-standard routing scenarios |
 
### Authentication
- Classic Dynatrace API token, sent as `Authorization: Api-Token {token}`. This is a classic-API-token flow, not the platform token/OAuth model used elsewhere in the current Dynatrace platform.
- **Token scopes required, and this is important, don't conflate it with the classic ingest scopes documented elsewhere in this project:**
| Signal | OTLP ingest scope (used by Bindplane) | Note |
|---|---|---|
| Metrics | `metrics.ingest` | Same scope name as classic metric ingest (`/api/v2/metrics/ingest`); the OTLP and classic ingest endpoints are distinct but share this scope. |
| Logs | `logs.ingest` | Same scope name as classic log ingest; this one does overlap. |
| Traces | `openTelemetryTrace.ingest` | OTLP-specific; no classic-API equivalent scope name. |
 
Combine whichever scopes match the signals you're exporting onto a single token.

### Additional destination parameters
Beyond the core deployment/auth fields, the `dynatrace_otlp` destination supports:

| Parameter | Default | Purpose |
|---|---|---|
| **Additional Headers** | (empty) | Map of custom HTTP headers appended to every OTLP request. |
| **Drop Raw Copy** | `true` | Removes `log.record.original` from log records before export. Reduces log payload size; disable only if raw log preservation is required downstream. |
| **Enable Retry on Failure** | `true` | Retry failed exports. Initial interval: 5s, max interval: 30s, max elapsed time: 300s. |
| **Number of Consumers** | `10` | Parallel goroutines consuming the sending queue. |

- **Partial signal loss is the single most common Bindplane→Dynatrace support pattern:** a token missing one of the three scopes causes *only that signal* to be silently rejected, metrics and logs succeed while traces vanish (or any other combination), which reads to the customer like a Bindplane bug when it's actually a scope gap. Always check scopes first when only one signal type is missing.
### Resilience defaults
- **Sending queue:** enabled by default, default size 5,000 batches. Increase if the customer sees bursty traffic exceeding that.
- **Persistent queuing:** enabled by default; buffers telemetry to disk so a collector restart or a temporary Dynatrace outage doesn't drop data. Keep both on in production; don't disable either as a "simplification."
### TLS
- For ActiveGate or Custom deployment types against a private/internal CA, set the **TLS Certificate Authority File** parameter rather than disabling verification.
- "Skip TLS Certificate Verification" exists but is explicitly intended for short-lived testing only; flag it if you see it enabled in what looks like a production config.
- **ActiveGate topology mapping:** when routing through an ActiveGate, Dynatrace requires explicit host topology mapping via resource attributes for entities to appear correctly. Verify resource attribute configuration when a customer reports missing topology context after switching from SaaS to ActiveGate deployment type.
### Example configs
 
**SaaS:**
```yaml
apiVersion: bindplane.observiq.com/v1
kind: Destination
metadata:
  name: dynatrace
spec:
  type: dynatrace_otlp
  parameters:
    - name: telemetry_types
      value: [Logs, Metrics, Traces]
    - name: deployment_type
      value: SaaS
    - name: your_environment_id
      value: abc12345
    - name: dynatrace_api_token
      value: dt0c01.REPLACE_WITH_TOKEN
```
 
**ActiveGate:**
```yaml
apiVersion: bindplane.observiq.com/v1
kind: Destination
metadata:
  name: dynatrace-activegate
spec:
  type: dynatrace_otlp
  parameters:
    - name: telemetry_types
      value: [Metrics, Traces]
    - name: deployment_type
      value: ActiveGate
    - name: activegate_hostname
      value: activegate.internal.example.com
    - name: port
      value: 9999
    - name: your_environment_id
      value: abc12345
    - name: dynatrace_api_token
      value: dt0c01.REPLACE_WITH_TOKEN
```
 
---
 
## Troubleshooting: Bindplane → Dynatrace
 
### Data not appearing in Dynatrace at all
```
Is the destination type dynatrace_otlp (not the deprecated "dynatrace" type)?
│
├─ NO (using deprecated type) → Still functions, but migrate to dynatrace_otlp; no further fixes will ship for the old type
│
└─ YES → Is deployment_type correct for the actual environment?
         │
         ├─ Wrong type (e.g., SaaS selected but customer is on Managed/ActiveGate) →
         │  Wrong endpoint URL gets derived; fix deployment_type and re-verify hostname/port/environment ID
         │
         └─ Correct → Check the API token
                      ├─ Valid and unexpired?
                      ├─ Environment ID correct (request reaching intended environment)?
                      └─ See "Partial signal loss" below if some but not all signals are missing
```
 
### Partial signal loss (e.g., traces missing, logs/metrics fine)
- Almost always a token scope gap, not a connectivity issue.
- Confirm the token carries the scope matching **each** signal being exported: `metrics.ingest`, `logs.ingest`, `openTelemetryTrace.ingest`.
- Regenerate the token with all required scopes combined; don't create three separate single-scope tokens unless there's a specific reason to isolate them.
### TLS / connectivity errors (ActiveGate or Custom deployment types)
- Confirm the ActiveGate hostname and port (default `9999`) are reachable from the Bindplane collector host.
- For a private CA: set the TLS Certificate Authority File rather than skipping verification.
- Confirm the SaaS OTLP endpoint pattern if applicable: `https://{environment-id}.live.dynatrace.com/api/v2/otlp`.
- **If using `.apps.dynatrace.com`:** this is the wrong domain for OTLP ingest and returns HTTP 404. Switch to the `.live.dynatrace.com` endpoint.
### Gaps correlating with restarts or network instability
- Verify the sending queue and persistent queuing are both enabled (they're on by default; check nothing disabled them).
- If bursts exceed the default queue size of 5,000 batches, increase it.
---
 
## How This Fits the Broader Dynatrace Platform
 
**Key Architectural Fact:** Bindplane has its own control plane (Bindplane Server), its own data plane (BDOT Collectors), its own console, and its own tokens. It is separate from the Dynatrace data plane (OneAgent, Grail, OpenPipeline). Bindplane exports to Dynatrace via OTLP the same way any external source does. Do not assume unified login, shared tokens, or in-Dynatrace UI configuration until you verify it's actually shipped.
 
- Bindplane operates **upstream** of OneAgent/OpenTelemetry ingestion, a pre-processing and routing layer at or near the collection point. It is not a replacement for OneAgent, and it does not replace Grail as the storage/query layer.
- It is conceptually adjacent to **OpenPipeline** (Dynatrace's existing ingest-time processing layer), but operates at a different point in the data lifecycle: OpenPipeline transforms data *after* it reaches Dynatrace and before it's written to Grail; Bindplane transforms/routes data *before* it reaches any backend at all, including non-Dynatrace ones.
- **As of this writing, Bindplane remains a standalone product**: its own console (Bindplane Cloud or self-hosted), its own documentation (docs.bindplane.com), its own licensing/editions, separate from Dynatrace Account Management, SSO, and Dynatrace Platform Subscription billing.
- Dynatrace has stated an intent to deepen integration and accelerate Bindplane's roadmap post-acquisition. **Do not describe any of the following as already true unless freshly verified:** unified login between Dynatrace and Bindplane accounts, Bindplane pipeline configuration inside the Dynatrace UI, combined billing/subscription, or Bindplane telemetry appearing natively in Grail without going through the OTLP destination like any other external source. These are plausible future directions, not confirmed current behavior.
---
 
## Glossary Additions
 
| Term | Definition |
|------|------------|
| **Bindplane** | OpenTelemetry-native telemetry pipeline company acquired by Dynatrace (closed April 15, 2026); operates as a control plane for telemetry collection, processing, and routing. |
| **Bindplane Server** | The management/control plane half of Bindplane's architecture (GraphQL Server, REST Server, WebSocket Server, Store); pushes configuration to BDOT Collectors via OpAMP. Deployed as Bindplane Cloud (hosted) or self-hosted. |
| **BDOT Collector** | Bindplane's own distribution of the OpenTelemetry Collector, built via the OpenTelemetry Collector Builder; first collector distro to implement OpAMP. |
| **OpAMP** | Open Agent Management Protocol: how Bindplane Server pushes configuration to and manages fleets of collectors. |
| **Agent mode** | A BDOT Collector configured to collect telemetry from the individual host it runs on. |
| **Gateway mode** | A BDOT Collector configured to receive telemetry from other collectors over the network and forward it onward, optionally with additional processing. |
| **Blueprint** | A pre-built processor bundle addressing a common transformation use case. |
| **Fleet** | A grouped set of collectors managed together under one configuration umbrella. |
| **Snapshot (Bindplane)** | A recent sample of logs/metrics/traces captured from a collector for inspection, distinct from Dynatrace's own "Session Replay" or trace snapshot concepts. |
 
---
 
## When to Escalate (Bindplane-specific)
 
- Questions about current Enterprise vs. Cloud vs. Google edition feature boundaries, or pricing → Bindplane/Dynatrace sales, not this file.
- Questions about roadmap integration depth ("will Bindplane configs live inside Dynatrace?") → acknowledge it's a stated direction, but don't speculate on timing; point to dynatrace.com/blog for announcements.
- Custom OpenTelemetry Collector distribution build issues (OCB manifests, GitHub Actions workflows) → this is a deeper build-engineering topic; confirm the customer's manifest includes the required OpAMP Supervisor packaging before troubleshooting further, and consider routing to Bindplane support for anything beyond basic connectivity.
 

