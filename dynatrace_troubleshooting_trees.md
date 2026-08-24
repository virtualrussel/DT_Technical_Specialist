# Dynatrace Troubleshooting Decision Trees
 
> **Last verified against docs.dynatrace.com:** 2026-08-24
> **Staleness policy:** Decision trees encode current UI paths, CLI commands, and product behavior; these change more frequently than architectural concepts. If a diagnostic step (menu path, kubectl output format, API response shape) doesn't match what the person is seeing, don't assume they're doing it wrong. Verify the step against docs.dynatrace.com or the Dynatrace Community first, since the product likely moved.
 
Use these decision trees to rapidly isolate and diagnose issues in pre-sales and incident response scenarios. Start at the symptom and follow the branches.
 
**Important:** Each tree assumes a particular deployment model or scope. If a tree's assumptions don't match your setup, restart with the tree that covers your deployment model:
 
| If you're using... | Start with... |
|---|---|
| OneAgent monitoring (full-stack, cloud-native, or application-only) | Trees 1, 2, 3, 5 |
| Kubernetes/OpenShift with Dynatrace Operator | Tree 8 |
| Bindplane collectors (BDOT) | Consult docs.bindplane.com; these trees cover Dynatrace, not Bindplane |
| Bluebox AI SRE agent | Consult docs.bluebox.ai; these trees cover Dynatrace observability, not Bluebox investigations |
| DQL queries returning errors or performance questions | `dynatrace_dql_reference.md` directly (Tree 6 here is a pointer stub, not the source of truth) |
| Ingest APIs (metrics, logs, events) | Tree 9 or Tree 10 |
| OpenPipeline: data ingested but missing from Grail | Tree 9 (OpenPipeline branch at the bottom) |
| OpenPipeline: API config or scope errors | Tree 4; also see API Quick Reference (OpenPipeline API section) |
| API returning 401/403 (any endpoint) | Tree 4 |
| RUM / Digital Experience Monitoring data missing | Tree 7 |
| Problem or alert not firing as expected | Tree 11 |
 
---
 
## Tree 1: OneAgent Not Reporting Data
 
**Symptom**: OneAgent installed but no metrics, traces, or logs appearing in Dynatrace.
 
**Assumptions**: You are using OneAgent in some deployment mode (full-stack, cloud-native, or application-only). If you're using Kubernetes with the Dynatrace Operator, start with Tree 8 instead.
 
```
Is OneAgent running?
│
├─ NO → Check installation logs
│       ├─ Check /var/log/dynatrace/ (Linux) or Event Viewer (Windows)
│       ├─ Verify OneAgent has elevated privileges
│       ├─ Verify OS is supported (not Alpine Linux for classic full-stack)
│       └─ Reinstall if corrupted
│
└─ YES → Is network connectivity OK?
         │
         ├─ NO (firewall/proxy blocking) → 
         │   ├─ OneAgent requires OUTBOUND-ONLY to Dynatrace or ActiveGate
         │   ├─ Required ports: 443 (SaaS), 9999 (ActiveGate)
         │   ├─ Check firewall rules; OneAgent never accepts inbound
         │   └─ If using proxy, configure via oneagentctl or config file
         │
         └─ YES → Is OneAgent configured to reach Dynatrace or ActiveGate?
                  │
                  ├─ Pointing to wrong ActiveGate/environment? →
                  │  ├─ Verify hostname resolves
                  │  ├─ Verify ActiveGate is running (systemctl status dynatrace-activegate)
                  │  ├─ Check ActiveGate logs: /opt/dynatrace/gateway/log/
                  │  └─ Verify ActiveGate can reach Dynatrace Cluster
                  │
                  └─ Configuration correct? →
                     ├─ Check OneAgent version (oneagentctl --version)
                     ├─ Check for auto-update issues (consult release notes)
                     ├─ Verify environment ID/tenant ID in Dynatrace settings
                     └─ If issue persists, enable verbose logging and compare against reference guide core architecture section
```
 
**Follow-up questions to ask:**
- Which OS and version?
- Deployment mode: Full-stack, cloud-native, or application-only?
- Is OneAgent behind a proxy?
- Are you using an ActiveGate or direct-to-SaaS?
- What does OneAgent log say? (Check logs directory)
---
 
## Tree 2: High Latency in Traces / Slow Requests
 
**Symptom**: Traces show unexpectedly high end-to-end duration or specific service is slow.
 
**Assumptions**: You have OneAgent instrumentation collecting traces and can access trace details in Dynatrace. You are asking about PurePath trace analysis, not Bluebox investigations.
 
```
Is the latency in application code or I/O?
│
├─ Database/Network I/O → 
│  ├─ Run DQL query to find slow spans:
│  │  fetch spans | filter span.kind == "DB" | filter duration > 1s
│  ├─ Check if database is the bottleneck (connection pool exhaustion, slow query, network latency)
│  ├─ Check service instance count (horizontal scaling?)
│  ├─ Check database module is enabled in OneAgent
│  └─ Review profiling data in trace (CPU vs. network vs. waiting time breakdown)
│
└─ Application code → 
   ├─ Is it consistent or intermittent?
   │  ├─ CONSISTENT → Probable code-level inefficiency or resource contention
   │  │  ├─ Drill into PurePath trace and examine code-level detail tabs
   │  │  ├─ Look for lock contention, synchronous I/O in hot path
   │  │  ├─ Check if OneAgent has code-level profiling enabled
   │  │  └─ Compare to baseline (Davis AI should have flagged this as problem)
   │  │
   │  └─ INTERMITTENT → Probable GC, resource saturation, or external dependency
   │     ├─ Check JVM heap/GC metrics (if Java)
   │     ├─ Check host-level CPU/memory (OneAgent module overhead?)
   │     ├─ Check downstream service response times (dependency latency)
   │     ├─ Query for problems: fetch dt.entity.service | filter name == "YourService"
   │     └─ Check service instance distribution (is traffic balanced?)
```
 
**Follow-up questions to ask:**
- Is this affecting all requests or specific endpoints?
- What's the baseline latency (was this slower than normal)?
- Can you share a trace ID?
- Are there any recent deployments or config changes?
- Is the downstream service (database, external API) healthy?
---
 
## Tree 3: Trace Truncation / Node Limit Warnings
 
**Symptom**: Distributed traces show diagnostic message "Trace truncated due to node limit" or "Missing service correlation."
 
```
Is the trace actually incomplete?
│
├─ Check diagnostic message text in trace details
│  └─ "Too many nodes" → OneAgent hit resource protection limit
│     ├─ Reduce number of custom services (each adds spans)
│     ├─ Disable unnecessary OneAgent features
│     ├─ Check if you're tracing extremely long-running transactions
│     └─ Contact Dynatrace if truncation is blocking root cause analysis
│
└─ "Missing service correlation" → OneAgent can't determine service name
   ├─ Check if service detection rule applies (Settings > Service Detection)
   ├─ Verify service has recognizable attributes (e.g., http.route, service name)
   ├─ Check if service is running on supported technology (reference guide platform support)
   └─ Manually configure service detection rule if auto-detection fails
```
 
**Mitigation:**
- **Short-term**: Run narrower DQL queries (shorter time range, filter by service) to analyze untruncated traces
- **Long-term**: Reduce trace node count or enable selective feature flags
- **Reference**: See "Common Feature Boundaries and Limitations" section in reference guide for trace limits
**Follow-up questions to ask:**
- How many spans are in affected traces (rough count)?
- Are custom services configured? How many?
- Is this trace for a batch job or standard request?
- When did truncation start occurring?
---
 
## Tree 4: API Authentication / Token Failures
 
**Symptom**: API call returns 401 Unauthorized, 403 Forbidden, or permission denied.
 
```
Is the API token valid?
│
├─ Token doesn't exist or is expired? →
│  ├─ Go to Settings > Access Tokens
│  ├─ Create new token with required scopes
│  ├─ Copy full token value (not just ID)
│  └─ Verify token hasn't been revoked
│
└─ Token exists → Does it have required scopes?
   ├─ Check which scopes the endpoint needs (API Explorer shows lock icon)
   ├─ Common scopes:
   │  ├─ metrics.read (query metrics)
   │  ├─ metrics.ingest (ingest custom metrics)
   │  ├─ logs.ingest (ingest logs)
   │  ├─ events.ingest (ingest events)
   │  └─ storage:entities:read (query entities)
   ├─ If scopes missing, delete token and create new one with required scopes
   └─ Re-test with corrected token
```
 
**Request validation:**
- **Classic API** (`/api/v2/...` on `.live.dynatrace.com`): `Authorization: Api-Token {token}` — not Bearer
- **Platform API** (`/platform/...` on `.apps.dynatrace.com`) and Grail Query API: `Authorization: Bearer {token}` with a platform token or OAuth access token — not Api-Token
- Mixing auth types is the most common 401 cause that scope checks don't catch; verify the API generation before checking scopes (see API Quick Reference "Two API Generations" section)
- Content-Type header correct for endpoint (e.g., `text/plain; charset=utf-8` for metrics ingest)
- URL uses correct base domain: `.live.dynatrace.com` for classic; `.apps.dynatrace.com` for platform
- Token is copied exactly as issued (no truncation, no added whitespace, no re-encoding); Dynatrace API tokens are opaque strings sent as-is in the header, not Basic Auth, so they should never be base64-encoded by the client
**Follow-up questions to ask:**
- What's the exact error message (401 vs 403)?
- Which API endpoint are you calling?
- Can you share (sanitized) the Authorization header?
- When was the token created?
---
 
## Tree 5: ActiveGate Connectivity Issues
 
**Symptom**: OneAgents failing to connect to ActiveGate, or ActiveGate failing to forward data.
 
```
Is ActiveGate running?
│
├─ NO → 
│  ├─ Start service: systemctl start dynatrace-activegate (Linux) or Services (Windows)
│  ├─ Check startup logs: /opt/dynatrace/gateway/log/gatewayserver.log
│  ├─ Verify disk space (minimum 10 GB free)
│  ├─ Verify port 9999 is free (or reconfigured)
│  └─ Reinstall if corrupted
│
└─ YES → Can OneAgents reach ActiveGate on port 9999?
         │
         ├─ NO (firewall/network) →
         │  ├─ Verify ActiveGate hostname resolves from OneAgent hosts
         │  ├─ Verify port 9999 is open inbound on ActiveGate host
         │  ├─ Check OneAgent config points to correct ActiveGate FQDN
         │  ├─ Test connectivity: telnet <ag-host> 9999 (from OneAgent host)
         │  └─ Check firewall rules on ActiveGate host
         │
         └─ YES → Can ActiveGate reach Dynatrace Cluster?
                  │
                  ├─ NO (firewall/outbound blocked) →
                  │  ├─ ActiveGate requires OUTBOUND to cluster on port 443
                  │  ├─ Check ActiveGate can resolve Dynatrace domain
                  │  ├─ Test: curl https://{environment-id}.live.dynatrace.com/api/v2/... from ActiveGate
                  │  ├─ If behind proxy, configure proxy in ActiveGate config
                  │  └─ Check outbound firewall rules
                  │
                  └─ YES → Check ActiveGate health
                     ├─ ActiveGate logs: /opt/dynatrace/gateway/log/
                     ├─ Check CPU/memory usage (shouldn't exceed 50% CPU, 80% memory)
                     ├─ Verify enough open file handles: ulimit -n (should be 500,000+)
                     ├─ Check disk space for reliability mechanism (2136 MB for log persistence)
                     └─ Review reference guide ActiveGate sizing section
```
 
**Follow-up questions to ask:**
- How many OneAgents are connected to this ActiveGate?
- What does ActiveGate log say?
- Is ActiveGate seeing data from OneAgents? (Check gateway logs for inbound connections)
- Can ActiveGate reach Dynatrace? (Try curl from ActiveGate host)
- When did connectivity drop?
---
 
## Tree 6: DQL Query Timeouts / Performance
 
**Symptom**: DQL query returns timeout error or takes >30 seconds to execute.
 
**This tree now lives in `dynatrace_dql_reference.md`, Part 3 (Performance and Troubleshooting).** That file has the full decision tree, the optimization checklist, and follow-up questions. It also owns the query-writing and Smartscape topology content that's naturally adjacent to a performance question (e.g., a slow topology `expand`). Use this tree number for routing purposes, the routing table above still points here, but treat the DQL reference file as the source of truth for content.
 
**Quick distinction:** if the DQL query is itself the problem (slow, timing out, badly structured), go to the DQL reference file. If you're using a DQL query as a diagnostic step inside a different problem (e.g., checking whether ingested data landed in Grail per Tree 9), stay in that tree, don't redirect just because `fetch` appears in a diagnostic step.
 
---
 
## Tree 7: RUM / Digital Experience Data Not Appearing
 
**Symptom**: No RUM data, partial data, or specific user actions missing for a web/mobile application.
 
```
Is RUM JavaScript being injected at all?
│
├─ Check Application Health Check page (first stop for any RUM issue)
│  └─ Run injection diagnostics: shows injection attempt count, failure ratio, failure reasons, top failing URLs
│
├─ NOT injected / high failure ratio →
│  ├─ Is OneAgent installed on the web/app tier?
│  │  ├─ NO → Auto-injection requires OneAgent on at least one instrumentable tier
│  │  │        → Use agentless RUM (manual JS insertion) if OneAgent can't be deployed
│  │  └─ YES → Is automatic RUM injection supported for this technology?
│  │           → Check Technology Support - RUM - Web servers/applications matrix
│  │
│  ├─ Is the HTML valid?
│  │  └─ Malformed HTML (unclosed/missing tags) is the most common injection failure cause
│  │     → Validate HTML structure; injection requires a valid <head> element
│  │
│  └─ Is content being served from a CDN or cache?
│     ├─ Long Cache-Control/Expires headers delay injection until cache eviction
│     └─ Verify cache control header optimization is active (OneAgent modifies ETag/Last-Modified)
│
└─ Injection succeeds but data still missing →
   ├─ Check browser dev tools: RUM JS downloads with 200 status?
   ├─ Beacon endpoint responding? (Response should start with "OK(BF)")
   ├─ Custom injection rule accidentally set to "Do not inject" for the affected URL pattern?
   └─ Check for ad blockers / content security policy (CSP) blocking the beacon endpoint
```
 
**Follow-up questions to ask:**
- Is this affecting all pages/users or specific ones?
- Auto-injected (OneAgent) or agentless (manual JS insertion)?
- Single-page application (SPA) or traditional multi-page site?
- Behind a CDN or reverse proxy?
- What does the Application Health Check page show for injection diagnostics?
---
 
## Tree 8: Kubernetes Operator / Injection Failures
 
**Symptom**: OneAgent, ActiveGate, or code-module injection failing in a Kubernetes/OpenShift cluster.
 
**Assumptions**: You are using the Dynatrace Operator to manage OneAgent/ActiveGate in a Kubernetes or OpenShift cluster. You have kubectl access to the dynatrace namespace. If you're using Bindplane collectors on Kubernetes instead, consult the Bindplane reference guide.
 
```
What is the current cluster state?
│
├─ Check DynaKube status: kubectl get dynakubes -n dynatrace
├─ Check pod status: kubectl -n dynatrace get pods
└─ Run built-in diagnostics: kubectl exec deploy/dynatrace-operator -n dynatrace -- dynatrace-operator troubleshoot
   (validates environment reachability, image pull access, proxy/cert config)
│
├─ Operator/OneAgent/CSI pods not Running →
│  ├─ kubectl describe pod <pod-name> -n dynatrace for events (image pull errors, resource limits, node scheduling)
│  ├─ kubectl logs <pod-name> -n dynatrace for startup errors
│  └─ Verify registry/image accessible from cluster (pull secrets configured correctly)
│
├─ Pods running but injection not happening in application namespaces →
│  ├─ Is the namespace labeled per the DynaKube's namespaceSelector?
│  ├─ Is the namespace assigned to more than one DynaKube? (Only one permitted per namespace)
│  ├─ Classic full-stack OneAgent and webhook injection both targeting the same pod?
│  │  (These are incompatible, both attempt to mount /var/lib/dynatrace)
│  └─ Check webhook is registered: kubectl get mutatingwebhookconfigurations
│
└─ Injection succeeds but no data in Dynatrace →
   ├─ Verify DynaKube apiUrl is correct and immutable (cannot be changed post-creation without recreating)
   ├─ Check ActiveGate connectivity from in-cluster ActiveGate to Dynatrace Cluster
   ├─ For OTLP-based apps: verify OTLP auto-configuration or manual exporter config points to correct endpoint
   └─ Generate a full support archive if escalating: 
      kubectl exec -n dynatrace deployment/dynatrace-operator -- dynatrace-operator support-archive --stdout > operator-support-archive.zip
```
 
**Follow-up questions to ask:**
- Which Dynatrace Operator version? Which Kubernetes/OpenShift version?
- Which deployment mode: cloudNativeFullStack, applicationMonitoring, hostMonitoring, or Kubernetes Platform Monitoring only?
- Was this working before and broke after an upgrade (Operator, Kubernetes, or OneAgent)?
- Are you using Helm or manifest-based installation?
- Any recent changes to the DynaKube CR?
---
 
## Tree 9: Log/Metric/Event Ingestion "Succeeds" but Data Missing in Grail
 
**Symptom**: API call returns 204 (success) or EEC accepts the payload, but the data never appears in DQL queries or the UI.
 
```
Did the ingest call actually succeed?
│
├─ Re-verify response code was 204 (not silently swallowed 4xx/5xx by a wrapper script)
│
├─ YES, confirmed 204 → Check timing
│  ├─ Data ingestion to query availability can take seconds, not always instant
│  ├─ Retry the DQL query after 30-60 seconds
│  └─ fetch logs | filter <known unique field> == "<test value>" (narrow, specific filter)
│
├─ Still missing after time buffer → Check field/bucket routing
│  ├─ Run fieldsSnapshot to confirm the field exists in the data object:
│  │  fetch logs | fieldsSnapshot logs
│  ├─ Was a non-default bucket specified at ingest that your query isn't targeting?
│  └─ Check permission: storage:logs:read / storage:metrics:read / storage:events:read required to query
│     (ingest scope success doesn't guarantee your query token has read scope)
│
├─ Using OneAgent local EEC endpoint? →
│  ├─ Is EEC actually enabled? (Settings > Preferences > Extension Execution Controller)
│  ├─ Is local HTTP ingest specifically enabled (separate toggle from EEC itself)?
│  ├─ Check EEC reliability/persistence disk space (2136 MB required; if unmet, persistence silently turns off)
│  └─ EEC endpoint only available for Full-Stack/Infrastructure Monitoring, not containerized setups
│
└─ Using OpenPipeline/processing rules? →
   ├─ Check if a processor is filtering, transforming, or dropping the record before storage
   └─ Review OpenPipeline configuration for unintended match conditions
```
 
**Follow-up questions to ask:**
- What was the exact HTTP response code and body from the ingest call?
- Which ingestion method: direct API, OneAgent EEC, or OpenTelemetry/OTLP?
- How long ago was the data sent?
- Can you share the exact DQL query being used to look for it?
- Is any OpenPipeline processing configured on this data type?
---
 
## Tree 10: Custom Metric Ingestion Rejected or Limited
 
**Symptom**: Custom metric ingestion returns an error, or expected data points don't appear.
 
```
What does the ingest response say?
│
├─ 400 Bad Request →
│  ├─ Check Content-Type header: must be "text/plain; charset=utf-8" for Metrics API v2 ingest
│  ├─ Check line format: metric_key{dimension=value} value timestamp
│  ├─ Verify metric key doesn't contain invalid characters
│  └─ Verify dimension keys are alphanumeric (plus underscore)
│
├─ 401/403 →
│  └─ See Tree 4 (API Authentication), requires metrics.ingest scope
│
├─ 204 (accepted) but metric doesn't appear →
│  ├─ Custom metric naming should use custom: prefix (recommended, not strictly enforced)
│  ├─ Check for reserved dimension collision (dt.entity.* is Dynatrace-reserved)
│  ├─ Query directly: fetch metrics | filter metric.key == "custom:yourmetric.name"
│  └─ Check subscription-tier custom metric limits/quotas (consult Environment API docs or account team)
│
└─ Metric appears but retention seems short →
   └─ Custom metric retention is tied to subscription tier, confirm with account team, not a fixed platform default
```
 
**Follow-up questions to ask:**
- What's the exact error response (status code + body)?
- Can you share a sample of the exact ingest payload (redact sensitive values)?
- Is this a new metric key or one that's worked before?
- How many distinct dimension combinations (cardinality) is this metric generating?
---
 
## Tree 11: Problem/Alert Not Firing as Expected
 
**Symptom**: A known degradation occurred but no problem was raised, or an alert/notification didn't fire.
 
```
Did Davis AI detect the underlying anomaly at all?
│
├─ Check Problems feed / fetch dt.entity.service | filter name == "YourService" for related timeframe
│
├─ NO problem raised →
│  ├─ Is this a metric with a static threshold configured, or relying on Davis AI dynamic baselining?
│  │  ├─ Dynamic baselining needs sufficient historical data to establish a baseline
│  │  │  (a new service/metric may not have enough history yet)
│  │  └─ Static threshold: verify the threshold value and comparison operator are correct
│  ├─ Is the affected entity in scope for alerting? (Check management zone / entity tag filters)
│  └─ Was the anomaly transient/brief enough to fall under Davis AI's evaluation window?
│
├─ Problem WAS raised, but notification didn't fire →
│  ├─ Check Event Subscription / notification integration configuration (Slack, PagerDuty, webhook, etc.)
│  ├─ Check throttling/deduplication settings, is this entity+problem-type combination being suppressed?
│  ├─ If using a Workflow-based notification: check Executions page for the workflow, did it run, and what was its final state?
│  │  (403 errors on workflow tasks usually indicate missing permissions for the workflow's actor)
│  └─ If event-triggered workflow: check if the 1,000 triggers/hour limit was hit (auto-disables trigger after repeated breaches)
│
└─ SLO-based alerting (burn rate) not firing →
   ├─ Confirm the SLO's error budget burn rate alert is configured and enabled
   └─ Check the SLO's own evaluation period/timeframe, it evaluates independently of dashboard timeframe
```
 
**Follow-up questions to ask:**
- Is this a built-in problem detection (Davis AI) or a custom metric event/threshold?
- Is the notification path direct (native integration) or via a Workflow?
- Is this a new service/entity, or one with established monitoring history?
- Can you confirm whether a Problem was created at all vs. the Problem existing but not notifying?
---
 
## Quick Decision Matrix: When to Escalate to Dynatrace Support
 
| Symptom | Severity | Escalate? |
|---------|----------|-----------|
| OneAgent won't start after OS update | High | Yes, immediately |
| Trace node truncation on every trace | High | Yes, may need config change |
| ActiveGate memory leak (grows unbounded) | High | Yes, likely bug |
| 10%+ of requests missing from traces | High | Yes, data loss issue |
| DQL query times out on simple query | Medium | Yes if after optimization |
| Single service latency high | Medium | No, investigate logs/dependencies first |
| OneAgent 1-5% CPU overhead | Medium | No, within normal range |
| API returns 404 on valid endpoint | Low | Maybe (check API docs first) |
| Dynatrace Operator pods CrashLoopBackOff after upgrade | High | Yes, likely version incompatibility |
| RUM injection failure ratio consistently high | Medium | No, check HTML validity/health check first |
| Custom metric ingestion silently dropped at scale | Medium | Yes if quota/cardinality limit suspected |
| Workflow event trigger auto-disabled (rate limit) | Medium | No, adjust trigger filter first |
| SLO burn rate alert never fires despite breaches | Medium | No, verify alert configuration first |
 
---
 
## Diagnostic Command Reference
 
### OneAgent Health (All Platforms)
```bash
# Check OneAgent version and running processes
oneagentctl --version
ps aux | grep dynaAgent  # Linux/macOS
tasklist | findstr dynaAgent  # Windows
```
 
### OneAgent Logs
```bash
# Linux
tail -f /var/log/dynatrace/agent/agent.log
 
# Windows
Get-Content "C:\ProgramData\dynatrace\oneagent\agent\log\*.log" -Tail 50 -Wait
 
# Docker container
docker logs <container-id>
```
 
### ActiveGate Health
```bash
# Check if running
systemctl status dynatrace-activegate  # Linux
Get-Service "Dynatrace ActiveGate"  # Windows
 
# Check connectivity to Dynatrace
curl -i https://{environment-id}.live.dynatrace.com/api/v2/environment/active \
  -H "Authorization: Api-Token {token}"
 
# Check listening ports
netstat -tlnp | grep 9999  # Linux
netstat -ano | findstr :9999  # Windows
```
 
### DQL Validation
```
# Test DQL schema discovery
fetch logs | fieldsSnapshot logs
 
# Test simple count (no aggregation, fast)
fetch spans | limit 1
 
# Test with sampling
fetch spans, samplingRatio:100 | summarize count_spans = count()
```
 

