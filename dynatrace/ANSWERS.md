# Dynatrace — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## OneAgent Deployment & Management

### 1. OneAgent consuming 15% CPU continuously

- **Compare host vs. fleet:** Check whether CPU is elevated on one host or across the fleet; a single outlier points to local workload or misconfiguration, while fleet-wide elevation may indicate a platform or version issue.
- **Isolate OneAgent overhead:** Use host metrics and OneAgent self-monitoring (process CPU, module load) to separate agent cost from application traffic; correlate with request rate, GC, and new deployments.
- **Review configuration:** Inspect `custom.properties`, monitoring scope (full-stack vs. infra-only), log capture, network monitoring depth, and code-module settings; disable unused sensors and reduce sampling where acceptable.
- **Check for runaway instrumentation:** High-traffic services, deep PurePath capture, or excessive custom metrics can inflate agent work; tune request naming, sensor exclusions, and high-load detection thresholds.
- **Validate platform health:** Confirm ActiveGate/cluster connectivity, update status, and known issues for your OneAgent build; open a support case if overhead persists with minimal scope on an idle host.
- **Tune and verify:** Apply changes in a canary group, measure CPU for 24–48 hours, and document the before/after in a change record.

### 2. Mass deployment across 2,000 hybrid hosts

- **Standardize packaging:** Use Dynatrace Environment API + deployment orchestration (Ansible, Puppet, Chef, SCCM, AWS SSM, Azure Automation) with environment-specific install tokens and host metadata tags.
- **Phased rollout:** Pilot on 50 hosts per environment/region; expand in waves (10% → 50% → 100%) with health checks on agent connected state and host coverage dashboards.
- **Version management:** Pin OneAgent versions in the orchestration repo; use auto-update policies with maintenance windows and staged version targets per environment.
- **ActiveGate topology:** Deploy cluster ActiveGates per site/cloud with load balancing; use Environment ActiveGate for SaaS connectivity from restricted networks.
- **Rollback:** Keep previous installer packages; uninstall/reinstall or downgrade via API; pause auto-update and revert orchestration playbook version if regression detected.
- **Governance:** Tag hosts with `environment`, `team`, `cloud`, and `cost-center`; enforce via CI validation of deployment manifests.

### 3. OneAgent update failed on 50 hosts

- **Collect failure signals:** Check OneAgent updater logs (`/var/log/dynatrace/`), installer exit codes, disk space, permissions, package conflicts, and proxy/firewall blocks to the update endpoint or ActiveGate.
- **Segment failures:** Group by OS version, image, network zone, and agent version; identify common denominator (e.g., read-only filesystem, SELinux, missing dependency).
- **Remediate root cause:** Fix disk, permissions, proxy ACLs, or conflicting packages; for immutable images, bake the new agent version into the AMI/container base and redeploy.
- **Safe catch-up:** Re-run update on failed cohort only after fix validation; use a smaller batch size and maintenance window; monitor host connectivity and data gaps in Dynatrace.
- **Prevent recurrence:** Add pre-update checks (disk, connectivity) to automation; alert on version drift between hosts and desired version.

### 4. Automatic injection in containers

- **How it works:** OneAgent (or the Dynatrace Operator with code modules) injects into supported runtimes via LD_PRELOAD, .NET profiler, or Node hooks when the process starts inside an instrumented pod/node.
- **Failure scenario:** Injection fails on **containerd** with a minimal distroless image because required libc paths or process visibility differ from Docker defaults.
- **Diagnosis:** Check Operator/DaemonSet logs, `dynatrace.com/inject` annotations, code module download status, and process group detection in the pod entity.
- **Resolution:** Enable application monitoring mode appropriate for the runtime; use init-container code module delivery; add required capabilities/volumes; for unsupported images, use OpenTelemetry or SDK instrumentation.
- **Validate:** Confirm PurePaths and service detection after pod restart; document exceptions for sidecar-only or static-binary workloads.

### 5. Communication, buffering, and replay

- **Path:** OneAgent sends data to the Dynatrace cluster directly (SaaS) or via **ActiveGate** (on-premises, restricted egress, or Kubernetes routing).
- **Interruption:** Agent buffers metrics, traces, and events locally within configured limits when the connection drops.
- **Replay:** On reconnect, buffered data is uploaded in order until limits are reached; oldest data may be dropped if the outage exceeds buffer capacity.
- **Operational impact:** Short outages cause minimal gaps; prolonged disconnects create chart gaps and delayed problem detection—monitor `Agent connectivity` and queue depth.
- **Design:** Place redundant ActiveGates behind load balancers for large estates; size network egress for peak traffic plus replay bursts.

### 6. Kubernetes Operator deployment modes

- **Install:** Deploy Dynatrace Operator + `Dynakube` CR with API token, data ingest token, and network policy allowing egress to cluster/ActiveGate.
- **ClassicFullStack:** Node-level OneAgent (DaemonSet) + full injection; closest to traditional VM full-stack monitoring.
- **CloudNativeFullStack:** Operator-managed node agent with cloud-native injection; preferred for modern K8s with non-root and CSI-friendly patterns.
- **ApplicationMonitoring:** Application-only instrumentation without full host monitoring; lighter footprint when infra metrics come from elsewhere.
- **Choose by need:** Full-stack for unified infra+app; CloudNative for standard K8s; ApplicationMonitoring for cost-sensitive or already-instrumented nodes.

### 7. Java application not detected

- **Verify process visibility:** Confirm the JVM process appears under the host/process list; check if it runs in a container without proper injection annotations.
- **Common causes:** Unsupported JDK/vendor, agent already present (double instrumentation), `ProcessGroupDetection` rules merging or splitting incorrectly, insufficient permissions, or monitoring disabled for the process group.
- **Diagnose:** Enable debug/support archives; review `java` sensor status, environment variables, and custom process-group naming rules.
- **Fix:** Adjust detection rules, whitelist the process, set `DT_CUSTOM_PROP` for naming, or use manual service detection; restart the JVM after module deployment.

### 8. High-security / restricted internet

- **Architecture:** Deploy **Environment ActiveGate** (and optional Cluster ActiveGate) on-premises as the only egress point; OneAgents connect to ActiveGate, not the public internet.
- **Firewall:** Allowlist ActiveGate → Dynatrace SaaS endpoints (or Managed cluster) on 443; block direct agent-to-internet if policy requires.
- **Certificates:** Use corporate CA trust stores on agents and gates; support proxy authentication if mandated.
- **Validate:** Test connectivity from a pilot host, confirm data ingestion, and audit that no agent bypasses the proxy path.

### 9. Ephemeral infrastructure

- **Automation-first:** Install OneAgent via user-data/cloud-init at boot with host metadata tags (`aws:autoscaling:groupName`, etc.).
- **Avoid manual drift:** Use golden AMIs or container base images with pre-baked agent version; treat agents as disposable with the instance.
- **Host churn handling:** Dynatrace retires hosts after missing heartbeat; use host-group rules and cloud integration for automatic tagging.
- **Kubernetes:** Operator reconciles DaemonSets on node join; use short-lived workload monitoring via pod injection rather than host naming stability.

### 10. Code module injection by language

- **Java:** JVM sensor via native agent; deep tracing and memory support; limitations with some frameworks, GraalVM native, or strict security managers.
- **.NET:** CLR profiler-based; strong on IIS/Kestrel; limitations on AOT, some single-file publishes, and profiler conflicts.
- **Node.js:** Requires supported Node versions; limitations on bundled/esbuild binaries and some serverless cold-start patterns.
- **Go:** Limited automatic deep tracing compared to Java/.NET; often needs OpenTelemetry or manual spans for rich detail.
- **Manual/uninstrumented:** Use OTel SDK, W3C trace context, and metric API ingestion; define service detection rules to stitch with PurePath where possible.

---

## Davis AI & Anomaly Detection

### 11. False positive during weekly batch job

- **Maintenance windows:** Schedule recurring windows for the batch period to suppress alerting while preserving metric collection.
- **Custom baselines:** Use seasonal baselines or metric events with static thresholds for batch-specific KPIs instead of relying solely on automatic baselines during known spikes.
- **Event suppression rules:** Suppress problems matching known batch signatures (tags, service, timeframe) without disabling global sensitivity.
- **Separate services:** Split batch workloads into distinct services/process groups so Davis learns separate behavior from online traffic.
- **Validate:** Confirm genuine regressions during batch still surface after tuning; review suppressed problems weekly.

### 12. Baseline learning

- **Mechanism:** Davis builds multidimensional baselines from historical metric behavior (time of day, day of week, load) per entity and metric.
- **Time to reliability:** Often **1–2 weeks** for stable services; highly variable or new services may need longer or manual tuning.
- **Inaccuracy factors:** Recent deployments, traffic pattern changes, holidays, incomplete data during onboarding, and insufficient metric history.
- **Mitigation:** Mark deployment events, use maintenance for planned shifts, and adjust detection sensitivity per service tier.

### 13. Incorrect root cause on problem card

- **Investigate:** Open the problem timeline, impacted entities, dependency graph, and evidence chain; compare with your own trace/log analysis.
- **Feedback:** Document findings in the incident record; use problem comments and internal runbooks linking to actual root cause.
- **Improve:** Tune service naming/detection, add custom metrics for missing signals, configure causal priority (e.g., database vs. service), and ensure all layers are instrumented.
- **Avoid blind trust:** Treat Davis as hypothesis generation; validate with PurePath, logs, and change events before remediation.

### 14. Cross-layer correlation

- **How it works:** Davis ingests infra metrics, service traces, RUM sessions, synthetic results, and events; causal AI ranks probable root causes across the stack.
- **Scenario:** User-facing checkout slowdown (RUM) correlates to API latency (service), connection pool saturation (process), and disk latency spike (host)—Davis surfaces DB storage as root cause before individual team alerts fire.
- **Value:** Reduces war-room thrashing by linking symptoms across teams into one problem with ranked causes.

### 15. Highly variable traffic (weekday vs. weekend)

- **Seasonal baselines:** Enable automatic seasonal detection where available; ensure enough history spans multiple weekly cycles.
- **Separate baselines:** Use different management zones or service splits for distinct traffic modes if they behave as different logical services.
- **Adjust sensitivity:** Lower sensitivity for noisy services; use SLO-based alerting for user-visible outcomes instead of every metric anomaly.
- **Metric events:** Define business-hour vs. off-hour thresholds for critical KPIs as a backstop.

### 16. Custom anomaly detection for business metrics

- **Ingest metrics:** Send order rate and payment success via **Metrics API v2** or OpenTelemetry metrics with consistent dimensions (`tenant`, `region`).
- **Metric events:** Create static or adaptive thresholds, seasonal rules, or anomaly detection on ingested business metrics.
- **Dashboards & SLOs:** Build SLOs on business KPIs; tie alerting profiles to metric events for PagerDuty routing.
- **Correlate:** Tag spans/logs with `order_id` where possible to connect technical and business signals in problems.

### 17. Maintenance windows

- **Implementation:** Define one-time or recurring maintenance windows scoped by entities, tags, or management zones.
- **Effect:** Problem detection and notifications are suppressed for matching entities during the window; data collection continues.
- **Caution:** Real outages during maintenance may not page—use scoped windows, communication, and post-maintenance verification.
- **After window:** Davis resumes normal detection; review any suppressed anomalies that indicated real issues.

### 18. Causal AI vs. threshold alerting

- **Difference:** Thresholds fire on isolated metric breaches; Davis correlates multiple degradations and ranks likely causes using dependency topology.
- **Scenario:** Gradual memory leak causes GC pressure (sub-threshold individually) plus rising latency and error rate—thresholds may lag; Davis opens one problem linking JVM, service, and downstream impact.

### 19. PagerDuty integration

- **Setup:** Install PagerDuty app/integration; map Dynatrace problem severity (availability, performance, resource, custom) to PagerDuty urgency and services.
- **Routing:** Use alerting profiles filtered by management zone, tags, and severity; route P1 payment services to critical escalation policies.
- **Resolution:** Enable automatic resolve in PagerDuty when Dynatrace closes the problem; configure deduplication keys to avoid alert storms.
- **Test:** Run synthetic problem or metric event drill monthly.

### 20. Davis AI assistant for incidents

- **Use cases:** Ask natural-language questions about problem context, impacted services, recent changes, and similar past incidents (capabilities evolve by platform version).
- **Workflow:** Start from the problem card → assistant → drill into PurePath, logs, and deployments it references.
- **Limitations:** May hallucinate or miss uninstrumented components; not a substitute for ground-truth traces; data residency and RBAC still apply.
- **Best practice:** Use for acceleration, always verify conclusions against raw telemetry.

---

## Distributed Tracing

### 21. 5-second gap in a distributed trace

- **Open PurePath:** Inspect the waterfall between the two calls for client-side wait, DNS, TLS, queueing, or thread pool blocking.
- **Network vs. app:** Compare client span duration to server span; large delta suggests network/proxy or serialization delay; similar duration on server indicates downstream processing.
- **Thread wait:** Look for lock contention, GC pauses, or async handoff gaps in method-level hotspots.
- **Repeat:** Filter traces by same path and time range to confirm pattern vs. outlier.

### 22. PurePath vs. OpenTelemetry

- **PurePath:** Automatic, code-level, proprietary sensor depth with minimal app changes; strongest in Dynatrace-supported stacks.
- **OpenTelemetry:** Vendor-neutral, SDK/agent explicit instrumentation; ideal for polyglot, serverless, and unsupported languages.
- **Polyglot trade-off:** OTel gives portability; PurePath gives deeper automatic method detail where supported—many enterprises use both with trace correlation.
- **Limitation:** OTel requires consistent propagation; PurePath is tied to Dynatrace ingestion and licensing model.

### 23. Critical transaction missing from traces

- **Check capture rules:** Verify request naming, URL patterns, and whether sampling or high-load detection throttled tracing.
- **Service detection:** Confirm the entry service is named correctly and not merged into another process group.
- **Async entry:** Message consumers may need Kafka/RabbitMQ sensor configuration for end-to-end capture.
- **Load/settings:** Increase capture for the service tier temporarily; validate with a synthetic request you can identify.

### 24. Async tracing (Kafka, RabbitMQ)

- **Sensors:** Enable messaging sensors so producer and consumer spans link via propagated context (Dynatrace trace tags in headers).
- **Propagation:** Ensure libraries are supported versions; for custom clients, inject/extract trace context manually or via OTel.
- **Visualization:** Service flow shows queue nodes; PurePath continues across the broker when headers are preserved.
- **Gap handling:** Broken propagation appears as trace fragmentation—fix header forwarding in wrappers.

### 25. Slowest DB queries across 15 microservices

- **Trace analysis:** Filter slow traces for the business transaction; expand database call nodes to see time per statement and execution count.
- **Aggregate:** Use database service views for top statements by total time and failure rate.
- **Prioritize:** Optimize queries with highest **total time** (frequency × latency), not just single slowest instance.
- **Close loop:** Validate with load tests and deployment comparison after index/query changes.

### 26. Custom span attributes for business context

- **Capture:** Use OpenTelemetry span attributes or Dynatrace SDK/custom request attributes (`customer_tier`, `order_value`) on entry spans.
- **Indexing:** Ensure attributes are allowlisted for trace/search (per platform limits and privacy policy).
- **Analysis:** Filter distributed traces and service requests by attribute; build segment dashboards for VIP tiers.
- **Privacy:** Mask or hash identifiers; avoid raw PII in span tags.

### 27. 2s client vs. 100ms server discrepancy

- **Clock skew:** Rare in managed SaaS but check on-premises collectors; compare NTP across hosts.
- **Network/proxy:** Client time includes DNS, TLS, connection setup, retries, and payload transfer; server may only measure handler time.
- **Async/client-side:** Client may wait for thread pool, callback, or aggregation not visible on server span.
- **Diagnose:** Compare PurePath client vs. server segments; inspect load balancer and service mesh spans if present.

### 28. OpenTelemetry integration

- **Deploy:** OTel Collector or direct OTLP export to Dynatrace with endpoint and API token; configure W3C `tracecontext` propagation.
- **Correlate:** Ensure upstream/downstream use compatible propagation; align `service.name` with Dynatrace service detection rules.
- **Hybrid:** OneAgent-instrumented services continue PurePath; OTel services appear as related services in Smartscape when linking works.
- **Validate:** Run a test transaction crossing both instrumented paths and confirm single stitched trace or linked traces.

---

## Kubernetes Monitoring

### 29. Intermittent gaps across 500 pods

- **DaemonSet health:** Check OneAgent/Operator pod restarts, resource limits, eviction, and node taints blocking scheduling.
- **API server:** Review K8s API latency, audit logs, and rate limiting affecting cluster monitoring collectors.
- **Dynatrace capacity:** Inspect ActiveGate and cluster ingest limits, throttling, and problem storms causing backpressure.
- **Method:** Correlate gap timestamps with node drains, operator upgrades, and API server maintenance; fix the layer with matching errors.

### 30. Namespace-specific monitoring requirements

- **Operator scopes:** Use namespace selectors in `Dynakube` or separate Dynakube instances per tier.
- **Full-stack namespaces:** Enable injection annotations on workloads needing PurePath.
- **Infra-only:** Monitor nodes/cluster metrics without code modules for those namespaces.
- **Exclusion:** Exclude namespaces via labels/annotations (`monitoring=off`) and network policies; document exceptions.

### 31. Kubernetes integration and cluster health

- **Objects:** Collects nodes, pods, workloads, quotas, events, and container metrics; links pod resource pressure to service response time.
- **Correlation:** Maps pod restarts and CPU throttling to service problems; ties deployment events to error rate changes.
- **Use:** Cluster overview for scheduling issues; drill to workload → service → trace during incidents.

### 32. Rollout error spike

- **Deployment events:** Enable Kubernetes deployment tracking and CI/CD version tags on services.
- **Correlation:** Dynatrace ties version change events to latency/error anomalies in the problem timeline.
- **Suppress false positives:** Maintenance window for planned rollout or alerting profile requiring sustained degradation beyond canary noise.
- **Automate:** Site Reliability Guardian or workflow to compare error rate pre/post deploy.

### 33. Operators and custom resources

- **CR monitoring:** Enable custom workload monitoring; define metric rules for CRD status fields via Prometheus or Metrics API scrape where supported.
- **Alerting:** Metric events on operator reconciliation failures, work queue depth, and CR phase != Ready.
- **Ownership:** Tag CR-based apps with team tags for routing.

### 34. Workload analysis for resource consumers

- **Workload view:** Sort deployments/StatefulSets by CPU/memory throttling, restarts, and pod count.
- **Impact:** Cross-reference with node saturation and cluster autoscaler events.
- **Action:** Right-size requests/limits, spread noisy neighbors, or add nodes; track in capacity review.

### 35. Container image vulnerability scanning

- **Integration:** Enable Application Security / vulnerability analytics tied to running image digests in K8s.
- **Alerting:** Metric events or security problems on new critical CVEs in running workloads.
- **Process:** Tie to CI image scanning; block deploy on critical CVE; Dynatrace alerts on drift when vulnerable images run in prod.

### 36. Pod restarts and trace continuity

- **Metrics:** Pod entity changes; service-level metrics aggregate across pod instances by service name.
- **Traces:** Individual PurePaths end with pod lifecycle; service view continues across replacements.
- **Visibility:** Use service (not pod) as SLO unit; tag releases with version for deploy correlation.

---

## Service Flow Analysis

### 37. Unexpected dependency between services

- **Verify call:** Drill into PurePath samples confirming real traffic (not mislabeled process group).
- **Configuration error:** Check feature flags, emergency endpoints, shared library callbacks, or incorrect service URL in config.
- **Security:** Unexpected egress may indicate compromise, data exfiltration, or shadow API—correlate with network policies and audit logs.
- **Remediate:** Fix routing/config; add network policy; open security incident if malicious.

### 38. Critical path across 20 microservices

- **Service flow:** Identify the slowest nodes and highest error contribution on the critical path for key transactions.
- **Prioritize:** Optimize services with largest **aggregate** time on the path, not peripheral fast calls.
- **Iterate:** Re-run analysis after each change under representative load.

### 39. Topology monitoring for new dependencies

- **Automatic discovery:** Smartscape updates as new calls appear in traces.
- **Alerting:** Metric events or Davis on new dependency edges (where supported) or anomaly in call count to new services.
- **Governance:** Compare against expected architecture diagrams in CI/CD architecture tests.

### 40. Service detection rules (shared process/container)

- **Rules:** Split by URL, host header, custom request attribute, or process property; avoid over-merging disparate APIs.
- **Testing:** Send traffic to each API and confirm distinct services in Smartscape.
- **Document:** Maintain detection rule repo in Git with review process.

### 41. High response time, fast downstream dependencies

- **Method-level:** Open PurePath method hotspots, CPU, lock, and IO within the service itself.
- **Causes:** Internal logic, cache miss, thread starvation, GC, or waiting on external resource not modeled as a service call.
- **Fix:** Profile hot methods; add caching; tune pool sizes.

### 42. Backtrace for upstream impact

- **Use backtrace:** From failing downstream service, list upstream callers and affected user sessions/RUM geos.
- **Communicate:** Quantify blast radius for status page and prioritization.
- **Mitigate:** Circuit-break upstream features or scale downstream while fixing root cause.

---

## Synthetic Monitoring

### 43. Intermittent failures from specific geos

- **Compare locations:** If one region fails while others pass, suspect CDN PoP, regional DNS, or local private location config.
- **CDN:** Check cache headers, geo routing, and TLS cert chain per region.
- **Script:** Review timeouts, selectors, and authentication token expiry—not all locations may share the same clock/skew.
- **Third-party:** Test from affected location with traceroute and synthetic waterfall capture.

### 44. Complex multi-step user journey

- **Browser monitor:** Record login, form steps, and validations; parameterize credentials via secure vault.
- **Steps:** Assert HTTP status, DOM elements, and download times per step; capture screenshots on failure.
- **Data:** Use test accounts and cleanup scripts; run from private locations for internal apps.

### 45. API monitoring beyond availability

- **HTTP monitor:** Assert status, latency SLA, and JSON body with JSONPath validations.
- **Schema:** Validate required fields and types; check business rules (e.g., `status == confirmed`).
- **Chaining:** Extract tokens from auth response for subsequent calls.

### 46. Synthetic in CI/CD

- **Trigger:** Invoke synthetic execution via API after deploy to staging/production canary.
- **Gate:** Fail pipeline if critical journey success rate < threshold or p95 latency exceeds budget.
- **Isolate:** Use environment-specific URLs and credentials; tag executions with build version.

### 47. Dynamic content false failures

- **Avoid brittle selectors:** Use stable `data-testid` attributes negotiated with dev team.
- **Regex/contains:** Match partial text patterns instead of full dynamic strings.
- **Wait strategies:** Explicit waits for elements; disable strict match on rotating promos.
- **Validate:** Run multiple executions to ensure flake rate is acceptable.

### 48. Global strategy (30 countries)

- **Locations:** Select public and private locations near user concentrations; minimum 3 per region for quorum.
- **Frequency:** Critical journeys every 1–5 minutes; secondary flows less often.
- **Thresholds:** Alert on multi-location failure (e.g., 2 of 3) to reduce single-PoP noise.
- **SLO:** Map synthetic success/latency to availability SLOs with geographic dashboards.

### 49. Synthetic for SLO compliance

- **Define SLI:** Success rate and step duration from synthetic metrics as availability/latency indicators.
- **SLO object:** Configure Dynatrace SLO on synthetic metrics with rolling window aligned to business reporting.
- **Error budget:** Track burn on synthetic SLI alongside RUM for inside-out + outside-in view.

---

## Real User Monitoring (RUM)

### 50. High JavaScript errors for a browser version

- **Segment:** Filter RUM by browser version, OS, and release; quantify session and user impact.
- **Session replay:** Watch failing sessions for stack traces, repro steps, and UI state.
- **Error analysis:** Group by error message and source map to line numbers; tie to recent frontend deploy.
- **Fix:** Patch or polyfill; verify error rate drop in canary release.

### 51. SPA with client-side routing

- **Configure:** Enable SPA framework support (Angular/React/Vue) so route changes generate virtual page views.
- **Naming:** Use consistent route naming rules; avoid hash-only navigation without hooks.
- **Actions:** Instrument key user actions as custom actions linked to routes.

### 52. Conversion failure journeys

- **Funnels:** Build user session query funnels from landing → cart → payment.
- **Drop-off:** Identify steps with highest abandonment; correlate timing with backend trace slowness on same `sessionId`.
- **Prioritize:** Fix journeys with high revenue impact and backend-correlated delays.

### 53. GDPR and RUM privacy

- **Masking:** Configure IP masking, do-not-track respect, and PII masking in session replay (inputs, URLs).
- **Retention:** Set data retention per regulation; EU data collection in EU-enabled environments where required.
- **Consent:** Integrate with CMP; disable recording without consent.
- **Audit:** Document processing in DPA; restrict replay access via RBAC.

### 54. Slow page loads, healthy backend

- **RUM waterfall:** Break down DNS, SSL, TTFB, resource download, and long tasks.
- **Client bottlenecks:** Large JS bundles, render-blocking assets, third-party scripts, and CDN latency.
- **Compare:** RUM geo vs. CDN PoP; device class (mobile vs. desktop).

### 55. Core Web Vitals

- **Enable:** RUM captures LCP, INP (successor to FID), and CLS per page/user segment.
- **Dashboards:** Segment by page, browser, release; set thresholds (e.g., LCP p75 < 2.5s).
- **Alert:** Metric events when p75 degrades over rolling window; tie to deploy markers.

---

## SLOs & Reliability

### 56. Payment service 99.99% / 200ms P99

- **SLIs:** Availability from successful payment requests; latency P99 from service or key transaction.
- **SLO config:** Create SLO with 0.01% error budget monthly and latency target; enable error budget burn visualization.
- **Burn alerts:** Fast burn (1h window) pages on-call; slow burn (24h) tickets.
- **Governance:** Freeze features when budget < 25% remaining.

### 57. SLO dashboard for 50 services

- **Central dashboard:** Tiles per service showing budget remaining, burn rate, and compliance status.
- **Tiering:** Group by criticality; filter by team tag and management zone.
- **Alerts:** Different thresholds for tier-1 (aggressive burn) vs. tier-3 (weekly digest).

### 58. SLO-based 24-hour breach prediction

- **Burn rate:** Use Dynatrace SLO burn-rate alerts comparing current consumption to time left in period.
- **Projection:** Alert when linear extrapolation exhausts budget within 24h.
- **Tune:** Multi-window to avoid single-spike false positives.

### 59. Weekly SLO review automation

- **Reporting:** Scheduled Davis/notebook or API export of SLO status per service.
- **Risk list:** Flag services below 50% budget or negative trend; assign owners via tagging.
- **Meeting:** Auto-post summary to Slack/email for SRE review.

### 60. SLOs with planned maintenance

- **Maintenance windows:** Exclude maintenance intervals from SLO evaluation where platform supports it.
- **Document:** Planned downtime in status communications; align internal error budget policy.
- **Verify:** Post-maintenance SLO calculations exclude only approved windows.

---

## Production Incident Scenarios

### 61. Response time degradation — 15 affected users

- **Problem card:** Review Davis root cause ranking, impacted services, and dependency graph.
- **Blast radius:** Use RUM/session count, synthetic status, and backtrace—not only the “15 users” snapshot which may be sampling.
- **Traces:** Open slowest PurePaths; check recent deployments and infra metrics on ranked cause.
- **Communicate:** Status update with scope, mitigation, and ETA; assign IC if P1.

### 62. Multiple correlated problems across 10 services

- **Trust topology:** Identify upstream problem Davis marks as root; downstream are often symptoms.
- **Timeline:** Align problem start times with deploy, config, or infra event.
- **Prioritize:** Fix root service/infra first; avoid parallel unfocused fixes on symptoms.
- **Close loop:** Confirm downstream problems auto-close when root heals.

### 63. Problem auto-closed but issue persists

- **Why:** Baseline recovered transiently, metric no longer breaches close criteria, or maintenance misconfiguration.
- **Investigate:** Reopen manually; check close reason and metric charts for flapping.
- **Configure:** Adjust problem closing sensitivity, longer rolling windows, or custom metric events on business KPIs.
- **Process:** Do not rely solely on auto-close for incident resolution confirmation.

### 64. Automated incident response workflow

- **Trigger:** Critical problem or SLO burn → Workflow Automation.
- **Actions:** Create Slack/Teams channel, page on-call, attach problem URL, spawn dashboard with key services.
- **Enrich:** Pull ownership tags, recent deployments, and runbook links.
- **Test:** Quarterly game day executing the workflow.

### 65. Deployment tracking and rollback

- **Tag versions:** CI/CD sets `DT_RELEASE_VERSION` / deployment events on each release.
- **Correlate:** Compare error/latency before and after version marker on service.
- **Auto-rollback:** Workflow triggers pipeline rollback when problem severity + deploy event match policy (with human approval for prod).

### 66. Database shown as root cause — DBA disagrees

- **Validate:** Open database service view—wait events, top statements, connection pool, disk IO.
- **False positive causes:** Mis-tagged remote call, connection pool exhaustion in app (looks like DB), or storage on shared host.
- **Collaborate:** Share PurePath wait analysis; compare DB host metrics with DBA tools.

### 67. Log monitoring during incidents

- **Ingest:** Send structured app logs via OneAgent log monitoring or Grail/OpenPipeline.
- **Correlate:** Filter logs by `trace_id` and problem timeframe; use log patterns Davis highlights.
- **Workflow:** Timeline of errors + metric anomalies accelerates root cause vs. logs alone.

---

## Advanced Configuration & Automation

### 68. Management zones and RBAC

- **Zones:** Partition entities by team, environment, and application using tag-based rules.
- **RBAC:** Assign permissions per zone so teams see only their data; platform team retains global zone.
- **Alerting:** Alerting profiles scoped to zone prevent cross-team noise.

### 69. API automation (IaC)

- **APIs:** Use Configuration API, Settings API, and Monaco (Configuration as Code) for alerting profiles, SLOs, and dashboards.
- **Pipeline:** Git PR → validate Monaco → deploy to staging environment → promote to prod.
- **Drift:** Periodic reconcile job detects UI manual changes.

### 70. Tagging strategy at enterprise scale

- **Convention:** Required tags: `team`, `environment`, `application`, `cost-center`, `tier`.
- **Automation:** Propagate from cloud/K8s labels via automatic tagging rules; block deploy without tags in CI.
- **Inheritance:** Use hierarchical rules so hosts inherit from cloud tags and services from processes.

### 71. Custom metrics for business KPIs

- **Ingest:** Metrics API v2 with dimensional labels; rate-limit aware batching.
- **Alert:** Metric events and SLOs on KPIs; dashboard per business unit.
- **Ownership:** Document metric catalog to avoid duplication.

### 72. Workflow automation remediation

- **Trigger:** Problem opened matching type (e.g., pod crash loop).
- **Actions:** Call K8s API to restart pod, scale HPA, or run Ansible playbook via secure webhook.
- **Guardrails:** Approval step for prod; idempotent actions; audit log every execution.

### 73. CMDB integration

- **Enrich:** Import ownership, support group, and business criticality from ServiceNow/BMC into custom attributes/tags.
- **Routing:** Alerting uses CMDB owner when Dynatrace tag missing.
- **Sync:** Scheduled job keeps CMDB and Dynatrace aligned.

---

## Performance Tuning & Capacity Planning

### 74. K8s capacity planning with infrastructure data

- **Trends:** Node CPU/memory allocation vs. usage, pending pods, and throttling over 30–90 days.
- **Forecast:** Project growth from request trends and business forecasts.
- **Bottlenecks:** Identify clusters with consistent pending pods or high throttling before outages.

### 75. Proactive scaling from Davis trends

- **Detection:** Gradual latency or resource anomalies before hard failure.
- **Action:** Trigger cluster autoscaler tuning or HPA changes via workflow when trend persists.
- **Validate:** Compare prediction accuracy monthly.

### 76. Network monitoring in microservices

- **Enable:** Network monitoring on OneAgent for retransmits, latency, and connectivity between hosts/services.
- **Mesh:** Combine with service mesh telemetry if deployed.
- **Diagnose:** Packet loss and TCP issues visible on infrastructure network views.

### 77. Java memory leaks

- **Signals:** Rising heap after GC, increasing GC time, OOM risk alerts.
- **Diagnostics:** Memory snapshots and heap analysis from Dynatrace (where licensed); correlate with growing object types.
- **Fix:** Identify leaking class, patch code, validate heap plateau after deploy.

### 78. Load testing integration

- **Tag tests:** Mark load test traffic via headers or custom attributes.
- **Compare:** Overlay load test window on production baselines cautiously; use scaled environment mirroring prod topology.
- **Representative:** Match data volume, cache warmth, and dependency behavior—not just RPS.

---

## Basic Questions

### 1. What is Dynatrace and what differentiates it from traditional monitoring tools?

Dynatrace is an AI-powered observability platform combining infrastructure, application, digital experience, and security monitoring in one stack. Unlike siloed tools requiring manual correlation, it auto-discovers topology, provides causal root-cause analysis (Davis), and captures deep distributed traces (PurePath) with minimal configuration. The differentiation is automatic entity modeling, full-stack correlation, and deterministic AI-driven problem detection rather than static dashboards alone.

### 2. What is the Dynatrace OneAgent and how does it collect monitoring data?

OneAgent is a single lightweight agent per host (or node) that auto-instruments supported applications and collects host, process, network, log, and trace data. It uses code modules and sensors injected into runtimes to capture PurePaths, metrics, and events, then sends them to Dynatrace directly or via ActiveGate. Deployment is typically install-and-forget with automatic discovery of processes and services.

### 3. What is the Dynatrace SaaS deployment model and how does it differ from Managed?

SaaS is Dynatrace-hosted multi-tenant cloud where customers connect agents to Dynatrace-operated clusters (e.g., `*.live.dynatrace.com`). Managed is a dedicated Dynatrace cluster operated by Dynatrace in your cloud or on-premises with more control over updates, networking, and data residency. Both use the same UI and agents; Managed suits stricter compliance and isolated upgrade schedules.

### 4. What is a Dynatrace entity and give four examples of entity types?

An entity is a monitored object with a unique ID in Smartscape. Examples: **HOST**, **SERVICE**, **PROCESS_GROUP**, and **APPLICATION** (RUM). Others include **KUBERNETES_CLUSTER**, **DATABASE_SERVICE**, and **SYNTHETIC_TEST**.

### 5. What is the purpose of Dynatrace's Smartscape topology map?

Smartscape visualizes real-time dependencies between hosts, processes, services, data centers, and applications based on actual traffic observed in traces. It helps incident investigation by showing blast radius and unexpected connections without manual CMDB drawing.

### 6. What is Davis AI and what is its primary function in Dynatrace?

Davis is Dynatrace's deterministic AI engine for automatic multivariate anomaly detection and causal root-cause analysis. Its primary function is to open **problems** that correlate related degradations and rank likely root causes instead of flooding operators with unrelated metric alerts.

### 7. What is a Dynatrace problem card and what information does it contain?

A problem card is Davis-generated incident record summarizing a detected issue. It typically includes severity, impacted entities, root-cause ranking, timeline of events, affected users/sessions, and links to traces, logs, and related problems.

### 8. What is the purpose of Dynatrace's service flow analysis?

Service flow shows how services call each other for a given workload, with response times and failure rates on each edge. It reveals architecture, critical paths, and bottlenecks for performance optimization and dependency governance.

### 9. What is a Dynatrace management zone and when would you use one?

A management zone is a logical scope filtering entities by tags or rules for views, RBAC, and alerting. Use it to separate teams, environments (prod vs. non-prod), or business units so each group sees only relevant data.

### 10. What is the difference between Dynatrace's full-stack monitoring and infrastructure monitoring?

Full-stack monitoring includes infrastructure plus deep application traces, method-level detail, and real user monitoring. Infrastructure monitoring focuses on hosts, networks, and processes without deep code-level instrumentation—lighter footprint but less application insight.

### 11. What is a Dynatrace SLO and how do you configure one?

An SLO defines a target for an SLI (e.g., 99.9% availability or P99 latency < 200ms) over a rolling period with error budget tracking. Configure it under SLO settings by selecting a metric or built-in service indicator, setting target and time window, and attaching burn-rate alerts.

### 12. What is the purpose of Dynatrace's synthetic monitoring?

Synthetic monitoring runs automated browser or HTTP tests from global or private locations to verify availability and performance before users are affected. It provides outside-in SLIs and early warning for CDN, DNS, and third-party failures.

### 13. What is Dynatrace RUM (Real User Monitoring) and what does it measure?

RUM captures telemetry from real user browsers and mobile apps: page load times, Core Web Vitals, JavaScript errors, user actions, and geographies. It measures actual experience, not just backend health.

### 14. What is the purpose of Dynatrace's distributed tracing (PurePath)?

PurePath provides end-to-end distributed traces with automatic code-level detail across services. It enables latency breakdown, error analysis, and dependency discovery without manual span wiring in supported stacks.

### 15. What is a Dynatrace tag and how do you apply one to an entity?

Tags are key-value metadata (e.g., `team:payments`) for filtering, management zones, and alerting. Apply via automatic tagging rules, OneAgent environment variables, Kubernetes labels, or the Tags API on entities.

### 16. What is the purpose of Dynatrace's anomaly detection?

Anomaly detection establishes dynamic baselines for metrics and flags deviations indicating performance, availability, or resource issues. It reduces static threshold maintenance and feeds Davis problem detection.

### 17. What is a Dynatrace dashboard and how do you create one?

A dashboard is a customizable visualization of metrics, problems, and logs. Create one in Dashboards classic or Dashboards 2.0 by adding tiles, selecting metrics/entities, and sharing with teams or pinning to apps.

### 18. What is the purpose of Dynatrace's alerting profiles?

Alerting profiles define **who** gets notified and **how** (email, Slack, PagerDuty) for problems and events matching filters. They connect Davis problems to on-call workflows without alerting on every raw metric.

### 19. What is the Dynatrace ActiveGate and when is it required?

ActiveGate is a gateway/proxy for OneAgent traffic, cloud API polling, and extensions. It is required when agents cannot reach SaaS directly (firewall), for Kubernetes routing at scale, and for many cloud integrations and private synthetic locations.

### 20. What is the purpose of Dynatrace's host monitoring?

Host monitoring tracks CPU, memory, disk, network, and OS-level health of physical and virtual machines. It provides infrastructure context for application problems and capacity planning.

### 21. What is a Dynatrace metric event and how does it differ from a problem notification?

A metric event is a user-defined alert on a specific metric threshold or anomaly rule. A problem notification is sent when Davis opens/closes a **problem** that may correlate many metrics—metric events are narrower and more deterministic.

### 22. What is the purpose of Dynatrace's process group monitoring?

Process groups cluster similar processes (same command line, metadata) for aggregated metrics and service detection. They bridge hosts and services when multiple instances run the same workload.

### 23. What is the Dynatrace API and what can you do with it?

The Dynatrace API (REST) automates configuration, queries metrics/problems, ingests data, and integrates CI/CD. Use it for Monaco deploys, custom dashboards, SLO setup, and workflow integrations.

### 24. What is the purpose of Dynatrace's network monitoring?

Network monitoring analyzes TCP/UDP performance between hosts: retransmissions, latency, and connection failures. It helps diagnose microservice communication issues not visible in application logs alone.

### 25. What is a Dynatrace maintenance window and when would you use one?

A maintenance window suppresses problem alerting for planned work on scoped entities. Use it for patches, releases, or known batch jobs to avoid false pages while still collecting data.

### 26. What is the purpose of Dynatrace's log monitoring?

Log monitoring ingests and analyzes log files and streams, correlating them with traces and problems. It supports pattern detection, Grail queries, and faster incident search.

### 27. What is the Dynatrace Operator for Kubernetes and what does it deploy?

The Dynatrace Operator manages OneAgent, ActiveGate, and code modules on Kubernetes via `Dynakube` custom resources. It automates lifecycle, upgrades, and injection configuration per cluster.

### 28. What is the purpose of Dynatrace's database monitoring?

Database monitoring shows statements, wait times, connection health, and resource usage for supported databases. It links slow queries to calling services in PurePath for end-to-end analysis.

### 29. What is a Dynatrace calculated service metric and when would you use one?

A calculated service metric derives new metrics from existing service data (ratios, aggregations across calls). Use it for custom KPIs like error ratio per endpoint or business-weighted latency without external pipelines.

### 30. What is the purpose of Dynatrace's cloud infrastructure monitoring?

Cloud monitoring ingests metrics and metadata from AWS, Azure, GCP via cloud integrations and ActiveGate. It correlates cloud services (Lambda, RDS, etc.) with OneAgent and Kubernetes telemetry.

### 31. What is the Dynatrace OneAgent Operator and how does it manage agent deployments?

In Kubernetes context, the Dynatrace Operator (often called OneAgent Operator colloquially) reconciles desired agent state in `Dynakube`—DaemonSets, codemodules, and version pins. It rolls out updates cluster-wide and handles namespace-scoped injection settings.

### 32. What is the purpose of Dynatrace's application security monitoring?

Application security detects runtime vulnerabilities, attacks, and risky libraries in running applications. It connects exploit attempts and CVE exposure to services for DevSecOps prioritization.

### 33. What is a Dynatrace custom device and when would you create one?

A custom device represents non-standard endpoints (IoT gateway, appliance, legacy system) sending metrics via API. Create one when no OneAgent can be installed but you need unified dashboards and alerting.

### 34. What is the purpose of Dynatrace's business analytics feature?

Business analytics correlates technical metrics with business outcomes (conversion, revenue, abandonment). It helps prioritize fixes by dollar impact, not only milliseconds.

### 35. What is the Dynatrace Grail data lakehouse and what does it store?

Grail is Dynatrace's unified observability data lakehouse storing logs, metrics, traces, events, and security data at scale. It powers DQL analytics, long retention, and cross-signal correlation without separate silos.

### 36. What is the purpose of Dynatrace's session replay feature?

Session replay records user interactions (with privacy controls) to reproduce UX issues seen in RUM. It accelerates frontend debugging when metrics alone are insufficient.

### 37. What is a Dynatrace extension and how do you install one?

Extensions package custom metrics, dashboards, and integrations (often via ActiveGate) for technologies Dynatrace does not natively deep-monitor. Install from Dynatrace Hub or upload extension packages and assign to ActiveGate groups.

### 38. What is the purpose of Dynatrace's infrastructure observability?

Infrastructure observability provides unified visibility into hosts, containers, networks, and cloud resources supporting applications. It ensures resource-layer issues are detected and tied to service impact.

### 39. What is the Dynatrace DQL (Dynatrace Query Language) used for?

DQL queries Grail-stored logs, metrics, spans, and events for analytics, troubleshooting, and reporting in Notebooks and dashboards. It replaces ad-hoc silo queries with one language across signals.

### 40. What is the purpose of Dynatrace's workflow automation feature?

Workflow automation orchestrates actions when problems, events, or schedules trigger—notifications, webhooks, remediations, and integrations. It operationalizes runbooks without separate scripting glue for every alert type.

### 41. What is a Dynatrace site reliability guardian and what does it do?

Site Reliability Guardian (SRG) validates release quality against SLOs and performance gates in CI/CD. It blocks or warns on deployments that would burn error budget or violate latency targets.

### 42. What is the purpose of Dynatrace's Davis anomaly detection baseline?

The baseline is the learned normal behavior envelope for each metric considering seasonality and load. Deviations from baseline trigger anomalies feeding problem detection with fewer false static thresholds.

### 43. What is the Dynatrace Hub and what resources does it provide?

Dynatrace Hub is the marketplace for extensions, apps, and integrations (cloud, alerting, security). It provides certified packages to extend monitoring without building from scratch.

### 44. What is the purpose of Dynatrace's ownership feature?

Ownership assigns teams and contacts to services and entities for alert routing and accountability. It ensures problems reach the right on-call group based on metadata, not guesswork.

### 45. What is the Dynatrace Notebooks feature used for?

Notebooks are collaborative analytics documents combining DQL queries, charts, and narrative for investigations and postmortems. They replace one-off exports with shareable, repeatable analysis.

---

## Intermediate Questions

### 1. OneAgent deployment on Kubernetes with CloudNativeFullStack

Install the Dynatrace Operator and create a `Dynakube` CR with API and data-ingest tokens, specifying `cloudNativeFullStack` mode. The Operator deploys node-level components and injects code modules into labeled namespaces/workloads. Configure namespace selectors, resource limits, and ActiveGate if egress requires a proxy.

### 2. Management zones for 50-team access control

Define tag standards (`team`, `environment`) applied automatically from CI/CD and cloud labels. Create one management zone per team with rules matching their tags plus a global platform zone for SRE. Map IAM groups to zone permissions and scope alerting profiles so teams only receive their problems.

### 3. SLO monitoring with error budget and burn rate alerting

Create per-service SLOs on availability and latency indicators with monthly rolling windows. Enable error budget visualization and configure fast/slow burn alerts (e.g., 14.4× and 6× multipliers). Integrate burn status into deployment pipelines and team dashboards.

### 4. Davis causal analysis and root cause accuracy

Davis ranks causes using topology and correlated metric degradations across layers. Improve accuracy by correct service detection, full instrumentation, deployment events, and disabling noisy entities. Tune sensitivity per service tier and validate top causes against PurePath during incidents.

### 5. Distributed tracing for Kafka

Enable Kafka sensors/producers/consumers so trace context propagates via message headers. Verify consumer process groups appear in Smartscape linked to producers. For unsupported clients, use OpenTelemetry instrumentation with header injection at publish/subscribe boundaries.

### 6. Synthetic monitoring for authenticated multi-step journeys

Build a browser synthetic with secure credential vault, login step, and subsequent navigation/assertions. Parameterize environment URLs; use private locations for internal apps. Set success criteria on each step and alert on multi-location failure.

### 7. Custom metrics via Metrics API

POST business KPIs to Metrics API v2 with dimension labels (`region`, `product`). Create metric events and SLOs on ingested series; dashboard per KPI owner. Enforce naming conventions and cardinality limits in CI.

### 8. Service detection rules for shared processes

Create splitting rules on URL patterns, host headers, or custom request attributes so each API becomes a distinct service. Test with representative traffic and avoid over-splitting that fragments metrics. Store rules in Monaco/Git.

### 9. Alerting profiles for different problem types

Create profiles filtered by severity (availability vs. performance), management zone, and tags. Route availability to PagerDuty critical, performance to Slack, and resource warnings to email digests. Test with staged metric events.

### 10. RUM for SPA with client-side routing

Enable SPA support in application settings and ensure the RUM JavaScript tag loads on initial HTML shell. Configure virtual page detection for route changes and name routes consistently. Track key actions as custom user actions.

### 11. Log monitoring with Davis log anomalies

Ingest structured JSON logs via OneAgent or OpenPipeline into Grail. Define log processing rules for fields; Davis detects anomalous log patterns and correlates with problems. Use `trace_id` in logs to jump to PurePath.

### 12. PurePath automatic end-to-end capture

OneAgent sensors inject at runtime entry points (HTTP, messaging, DB) and follow context propagation automatically. Each PurePath includes timing, failures, and method hotspots without manual span creation in supported languages.

### 13. Kubernetes workload monitoring with CRDs

Enable monitoring for operator-managed workloads; scrape Prometheus metrics from operators where needed. Alert on CR status conditions and tie workloads to services via labels and detection rules.

### 14. Application security for runtime vulnerability detection

Enable Application Security on eligible services to detect vulnerable libraries and runtime exploits. Integrate with CI image scanning; route critical CVE problems to security on-call. Tune noise by baselining known accepted risks with documented exceptions.

### 15. Business analytics for conversion and revenue impact

Ingest business events (purchase, cart abandon) as metrics or via Business Events API. Correlate drops with latency problems on checkout services. Dashboard revenue-at-risk during incidents for prioritization.

### 16. Calculated service metrics for custom aggregations

Define formulas combining existing service metrics (e.g., failed requests / total). Use for SLIs not available as defaults. Keep cardinality low and document definitions for SLO consumers.

### 17. Maintenance windows via API for CI/CD

Call Settings API to create maintenance windows matching deploy pipeline duration and scoped tags. Automate create/delete around terraform/helm applies. Verify suppression in test environment before prod.

### 18. Network monitoring for service mesh issues

Enable deep network analysis between meshed pods; combine with mesh telemetry if available. Look for elevated retransmits and connection failures between expected service pairs. Correlate with mTLS or proxy config changes.

### 19. Tagging strategy with automatic inheritance

Define automatic tagging rules from host → process → service hierarchy using cloud/K8s labels. Require mandatory tags at deploy time via admission policy. Audit missing tags weekly with DQL.

### 20. Anomaly detection sensitivity and false positives/negatives

Higher sensitivity detects smaller deviations (more false positives); lower sensitivity misses subtle issues (false negatives). Tune per service criticality; use SLO-based alerts for user-visible symptoms alongside Davis.

### 21. Workflow automation for infrastructure remediation

Trigger workflows on problem types like pod crash loops or disk saturation. Execute verified runbooks (restart, scale) with approval for production. Log all actions and measure success rate.

### 22. Davis for highly variable traffic patterns

Allow longer baseline learning; use seasonal baselines and separate services for batch vs. online paths. Combine Davis with SLO and synthetic monitoring for external validation.

### 23. OpenTelemetry for non-OneAgent services

Deploy OTel SDK/collector exporting OTLP to Dynatrace with proper `service.name` and W3C propagation. Align detection rules to stitch traces with PurePath services. Monitor ingest volume and attribute cardinality.

### 24. Smartscape topology for incident investigation

Smartscape is built from observed trace traffic between entities, updated continuously. During incidents, start at the impacted service and walk upstream/downstream to find first failing node and unexpected dependencies.

### 25. Session replay with GDPR masking

Enable replay with default input masking, URL parameter masking, and role-restricted access. Respect consent banners and exclude sensitive pages from recording. Document retention and EU data handling in privacy policy.

### 26. Cloud infrastructure monitoring for AWS multi-account

Deploy ActiveGate with IAM role per account or cross-account role assumption; enable AWS integration for all accounts. Tag resources consistently; use management zones per account/environment.

### 27. DQL in Notebooks for custom analytics

Write DQL to query logs, metrics, and spans in Grail—e.g., error rate by `k8s.namespace` over 7 days. Visualize in Notebooks and share links in postmortems. Parameterize queries for reuse.

### 28. Site Reliability Guardian for deployment gates

Configure SRG validation against SLOs and key requests after deploy in pipeline. Fail build if error rate or latency regresses beyond threshold compared to baseline window. Integrate with GitHub/GitLab checks.

### 29. Extension framework for custom technologies

Build or install Hub extensions that collect metrics via ActiveGate for mainframes, queues, or proprietary systems. Map to custom devices/services and add dashboards/alerts.

### 30. PagerDuty problem notification integration

Install integration, map problem severity to PagerDuty urgency, and scope alerting profiles by team zone. Enable auto-resolve on problem close; test with synthetic metric event monthly.

### 31. RUM for Core Web Vitals tracking and alerting

Enable CWV in RUM application settings; segment dashboards by page and device. Create metric events when p75 LCP/INP/CLS exceed Google thresholds. Correlate regressions with frontend deploy markers.

### 32. Grail vs. traditional time-series storage

Traditional TSDBs optimize metric cardinality; Grail unifies logs, metrics, traces, and events for DQL at scale with longer retention. It enables cross-signal queries without federating multiple backends.

### 33. Ownership for automatic alert routing

Assign owning team emails/Slack channels on services via tags or ownership records. Alerting profiles route using ownership when problems lack explicit tags. Sync from CMDB where possible.

### 34. Global synthetic monitoring from 30 locations

Deploy tests from geographically distributed public locations plus private locations for internal APIs. Alert on quorum failures; tune frequency by journey criticality. Map results to regional SLO dashboards.

### 35. Infrastructure monitoring for on-premises VMware

Use OneAgent on VMs and integrate vCenter for host/cluster metrics. Tag by cluster and environment; monitor datastore latency and resource contention affecting apps.

### 36. Automatic baseline learning for seasonal patterns

Davis incorporates daily/weekly seasonality when enough history exists. Baselines adapt after sustained pattern shifts; mark deployments to explain step-changes. Supplement with maintenance windows for annual events.

### 37. API monitoring for response correctness

Use HTTP synthetic with JSON body validations, schema checks, and business rule assertions—not only HTTP 200. Chain auth and dependent calls; alert on functional failures with zero availability impact.

### 38. Log monitoring for structured JSON

Configure log content rules to parse JSON fields into indexed attributes. Query in DQL with `| filter status == "ERROR"`. Correlate `trace_id` and `service.name` fields with traces.

### 39. Multi-cluster Kubernetes unified visibility

Deploy Operator/ActiveGate per cluster with consistent tagging; use management zones or dashboards filtered by `k8s.cluster.name`. Optional centralized ActiveGate tier for egress control.

### 40. Problem correlation into a single problem card

Davis groups related anomalies on connected entities occurring in a causal window into one problem. Symptoms on downstream services attach to the same card as upstream root cause when topology supports it.

---

## Advanced Questions

### 1. Enterprise deployment for 10,000 nodes (hybrid multi-cloud)

- **OneAgent:** Phased auto-deploy via cloud-init/SSM/Ansible with version pinning; host groups by `environment` and `region`.
- **ActiveGate:** Hierarchical Environment + Cluster ActiveGates per site/cloud with load balancing and HA pairs.
- **Management zones:** By business unit × environment; platform SRE global view.
- **Network:** Private egress only via ActiveGate; document allowlists and bandwidth sizing for replay bursts.
- **Governance:** Monaco GitOps for config; environment separation (prod/non-prod) at tenant or environment level.

### 2. SRE platform for 500 services (SLOs, error budgets, reports)

- **Standard templates:** Monaco modules for tier-1/2/3 SLOs, burn alerts, and dashboards.
- **Service catalog:** Required tags + ownership; auto-create SLO from catalog metadata in CI.
- **Reporting:** Weekly DQL/Notebook jobs exporting compliance and at-risk services.
- **Policy:** Deployment gates tied to remaining error budget via API checks.

### 3. Davis in microservices with cascading failures

- **Topology quality:** Accurate service detection and mesh/header propagation to avoid fragmented problems.
- **Noise control:** Dependency-aware problem merging; tune sensitivity on edge services vs. core.
- **Runbooks:** Validate top-3 Davis causes with PurePath before remediation; feed postmortems back into detection tuning.
- **Game days:** Practice large-scale failures to calibrate correlation thresholds.

### 4. Real-time business impact analysis during incidents

- **Business Events/metrics:** Stream orders/revenue with technical traces sharing session/business keys.
- **Dashboards:** Side-by-side technical SLI and revenue impact; Davis problems annotated with business KPI drop.
- **Executive view:** Single incident dashboard with affected users, regions, and estimated revenue at risk.

### 5. Kubernetes monitoring at 100-cluster scale

- **Operator per cluster** with GitOps ApplicationSet; standardized `Dynakube` tiers (full-stack vs. infra-only).
- **Federation:** Cross-cluster dashboards via consistent `k8s.cluster.name` tagging and Grail queries.
- **ActiveGate pools** per region for egress and API polling efficiency.
- **Capacity:** Monitor ingest and ActiveGate health as first-class platform metrics.

### 6. Automated incident response platform

- **Workflows:** Problem severity + tag filters trigger Slack war room, PagerDuty, and pre-built dashboard.
- **Context:** Attach ownership, recent deployments, similar problems, and top PurePaths automatically.
- **Governance:** Human approval for destructive remediations; audit trail for compliance.

### 7. Application security in zero-trust architecture

- **Runtime:** Enable attack detection, vulnerability analytics, and image digest tracking in K8s.
- **Attack paths:** Use security dashboards linking exposed services to CVEs and reachability.
- **Remediation:** Workflow tickets to patch or scale; block deploy on critical CVE via CI + SRG.

### 8. Financial services compliance deployment

- **Managed/SaaS region:** Choose data residency (e.g., EU tenant); document sub-processors.
- **RBAC/audit:** SSO, management zones, audit logs export, least-privilege tokens for automation.
- **Retention:** Grail retention policies; PII masking in RUM/logs; legal hold processes.
- **Reporting:** Scheduled compliance reports via DQL Notebooks.

### 9. Global e-commerce synthetic strategy (200 journeys)

- **Tier journeys:** P0 every 1–5 min from quorum locations; P2 hourly.
- **Performance budgets:** Per-step thresholds; regression detection vs. 7-day baseline.
- **CI integration:** Run subset on every deploy; full suite nightly.
- **Ownership:** Team-tagged synthetics with alerting profiles routed by tag.

### 10. Chaos engineering observability platform

- **Tag experiments:** Label chaos runs with experiment ID in metrics/traces.
- **Blast radius:** Smartscape + problems during experiment; compare SLO burn to hypothesis.
- **Gate:** Abort chaos if error budget burn exceeds policy in real time via workflow webhook.

### 11. OpenTelemetry at scale (500 services, 10 languages)

- **OTel Collector tier** per cluster/region with tail sampling and attribute enforcement.
- **Standards:** W3C propagation, semantic conventions, max attribute allowlist.
- **Correlation:** Service detection rules aligning OTel `service.name` with Dynatrace services.
- **Cost control:** Head-based sampling default; tail sample errors and high latency.

### 12. Real-time capacity planning recommendations

- **Trend analytics:** DQL over 90-day infra and service metrics; forecast CPU/memory/request growth.
- **Davis:** Flag gradual resource contention before saturation.
- **Outputs:** Weekly recommendations dashboard + optional workflow ticket to capacity board.

### 13. Self-healing workflow automation

- **Catalog safe actions:** Pod restart, HPA bump, cache flush with idempotency checks.
- **Triggers:** Typed problems with guardrails (max executions/hour, environment scope).
- **Learning:** Measure MTTR delta and rollback rate; require approval in prod initially.

### 14. DevOps platform with CI/CD validation

- **SRG gates** on latency, errors, and security vulnerabilities per release.
- **Deploy events:** Automatic version tagging; compare canary vs. baseline in pipeline.
- **Feedback:** PR comments with Dynatrace deep links to regressions.

### 15. Retail business analytics (conversion, cart, revenue)

- **Ingest:** Business events for funnel steps with `session` linkage to RUM.
- **Dashboards:** Conversion vs. latency heatmaps by region/device.
- **Incidents:** Quantify cart abandonment spike correlated with checkout service degradation.

### 16. Serverless (Lambda, Azure Functions) monitoring

- **Cloud integrations** for platform metrics and invocations; OneAgent not on function runtime.
- **Tracing:** OpenTelemetry Lambda layer exporting OTLP to Dynatrace.
- **Service model:** Function-as-service entities with cold-start and duration SLIs.

### 17. Security analytics for runtime attacks

- **Enable Application Security** for injection, RCE, and data exfiltration patterns.
- **Alerting:** Route attack problems to SOC; integrate SIEM export via API.
- **Runbooks:** Isolate service, capture forensics, patch vulnerability.

### 18. Cost optimization platform

- **Correlate:** Cloud cost metrics (via integration) with utilization and SLO compliance.
- **Identify:** Over-provisioned services with low CPU but high cost; rightsizing recommendations.
- **Guardrail:** Never cut resources for services near SLO breach without explicit approval.

### 19. Event-driven architecture trace continuity

- **Messaging sensors + OTel** across Kafka/SQS/EventBridge with strict header propagation.
- **Async boundaries:** Document broken links when brokers strip headers; use business correlation IDs.
- **SLOs:** End-to-end latency from publish to consumer handling completion.

### 20. Hybrid cloud migration unified observability

- **Dual instrumentation:** OneAgent on legacy VMs; Operator + OTel on new K8s services.
- **Consistent tags:** `environment`, `migration-wave`, `platform` on all entities.
- **Dashboards:** Single Smartscape view where traces connect; Grail for cross-platform queries.

### 21. Predictive incident detection (30-minute lead time)

- **Leading indicators:** Gradual GC, queue depth, error rate slope, synthetic degradation.
- **Davis tuning:** Enable early sensitivity on precursor metrics; validate false positive rate.
- **Action:** Auto-scale or preemptive traffic shift via workflow before user impact.

### 22. Platform engineering golden path observability

- **Templates:** Starter `Dynakube`, tags, SLOs, dashboards, and SRG in internal developer portal.
- **Policy:** Services without required telemetry fail CI; scorecard in catalog.
- **Versioning:** Pin supported OneAgent/OTel versions centrally.

### 23. RUM + synthetic comprehensive UX strategy

- **Synthetic** for proactive SLIs and regression; **RUM** for real-user validation and segmentation.
- **Align:** Same key journeys in both; alert when synthetic passes but RUM degrades (real-world-only issues).
- **SLOs:** Combine both SLIs with explicit weighting in executive dashboards.

### 24. Multi-tenant SaaS per-tenant observability

- **Tag/zone per tenant** with strict RBAC; optional dedicated environments for largest tenants.
- **Platform view:** Aggregated anonymized metrics cross-tenant for operator SRE.
- **Cardinality control:** Limit custom metric labels per tenant; rate limits on ingest API.

### 25. Grail for long-term retention and ML

- **Retention tiers:** Hot Grail for ops; longer retention for compliance and trend ML.
- **DQL pipelines:** Scheduled exports to data science for capacity and anomaly models.
- **Governance:** PII policies, access auditing, and cost monitoring on ingest volume.

---

## Rapid-Fire Questions

### 1. What does OneAgent automatically discover without manual configuration?

Hosts, processes, services, dependencies, and many technologies (supported runtimes, databases, messaging) without manual instrumentation configuration.

### 2. What is the purpose of Dynatrace's `problem.open` event in webhook notifications?

To signal that Davis opened a new problem—used by integrations to trigger paging, tickets, or workflows on incident start.

### 3. What does Davis AI stand for in Dynatrace?

Davis is a product name for Dynatrace's deterministic AI engine (not a public acronym); it powers automated anomaly detection and causal analysis.

### 4. What is the purpose of Dynatrace's `management zone` feature?

To scope data visibility, RBAC, and alerting to logical subsets of the environment (team, env, app).

### 5. What does Dynatrace's `Smartscape` visualize?

Real-time topology of entities and dependencies discovered from actual traffic.

### 6. What is the purpose of Dynatrace's `ActiveGate`?

Gateway for agent traffic, cloud monitoring, extensions, and private synthetic execution in restricted networks.

### 7. What does Dynatrace's `PurePath` technology capture?

End-to-end distributed traces with code-level timing and error detail across services.

### 8. What is the purpose of Dynatrace's `synthetic monitor`?

Proactive availability and performance testing via automated HTTP/browser scripts from global/private locations.

### 9. What does Dynatrace's `RUM` measure?

Real user experience: page loads, Core Web Vitals, JS errors, user actions, and geography.

### 10. What is the purpose of Dynatrace's `SLO` feature?

Track SLI targets, error budgets, and burn-rate alerting for reliability governance.

### 11. What does Dynatrace's `anomaly detection` baseline on?

Historical metric behavior including daily/weekly seasonality and load patterns per entity.

### 12. What is the purpose of Dynatrace's `tagging` feature?

Metadata for filtering, management zones, alerting routing, and ownership.

### 13. What does Dynatrace's `service flow` diagram show?

Service-to-service call paths with latency and failure rates for a workload.

### 14. What is the purpose of Dynatrace's `maintenance window`?

Suppress problem notifications during planned work while continuing data collection.

### 15. What does Dynatrace's `problem card` contain?

Severity, impacted entities, root-cause ranking, timeline, and links to related telemetry.

### 16. What is the purpose of Dynatrace's `alerting profile`?

Define notification channels and filters for who gets alerted on problems/events.

### 17. What does Dynatrace's `calculated service metric` create?

A derived metric from existing service data (ratios, aggregations) for custom SLIs.

### 18. What is the purpose of Dynatrace's `ownership` feature?

Map services to owning teams for accountability and alert routing.

### 19. What does Dynatrace's `Davis AI` root cause analysis identify?

The most likely originating entity/degradation causing correlated symptoms across the stack.

### 20. What is the purpose of Dynatrace's `workflow automation`?

Automate actions (notify, remediate, integrate) triggered by problems, schedules, or events.

### 21. What does Dynatrace's `session replay` capture?

Recorded user sessions (with masking) for reproducing frontend issues seen in RUM.

### 22. What is the purpose of Dynatrace's `site reliability guardian`?

CI/CD quality gate validating releases against SLOs and performance benchmarks.

### 23. What does Dynatrace's `DQL` query language access?

Logs, metrics, spans, events, and security data stored in Grail.

### 24. What is the purpose of Dynatrace's `Grail` data lakehouse?

Unified, scalable storage and analytics for all observability signals.

### 25. What does Dynatrace's `Notebooks` feature provide?

Collaborative DQL-based analysis documents with charts and narrative.

### 26. What is the purpose of Dynatrace's `extension framework`?

Extend monitoring to custom technologies via ActiveGate-based extensions.

### 27. What does Dynatrace's `application security` feature detect?

Runtime vulnerabilities, attacks, and risky dependencies in running applications.

### 28. What is the purpose of Dynatrace's `business analytics`?

Correlate technical performance with business KPIs like conversion and revenue.

### 29. What does Dynatrace's `infrastructure monitoring` cover?

Host, network, process, container, and cloud resource health metrics.

### 30. What is the purpose of Dynatrace's `log monitoring`?

Ingest, parse, and analyze logs correlated with traces and problems.

### 31. What does Dynatrace's `network monitoring` detect?

TCP issues, latency, retransmissions, and connectivity problems between hosts.

### 32. What is the purpose of Dynatrace's `cloud infrastructure monitoring`?

Monitor AWS/Azure/GCP services and link cloud metrics to applications.

### 33. What does Dynatrace's `Kubernetes monitoring` track?

Cluster, node, pod, workload health, events, and resource usage.

### 34. What is the purpose of Dynatrace's `database monitoring`?

Database performance, top statements, waits, and links to calling services.

### 35. What does Dynatrace's `process group` represent?

A group of identical processes aggregated for monitoring and service detection.

### 36. What is the purpose of Dynatrace's `host group`?

Logical grouping of hosts (e.g., by environment or role) for aggregated analysis and settings.

### 37. What does Dynatrace's `service detection rule` configure?

How processes/requests are split or merged into named services.

### 38. What is the purpose of Dynatrace's `custom device`?

Represent non-standard systems sending custom metrics into Dynatrace.

### 39. What does Dynatrace's `metric event` trigger?

Alerts on user-defined metric threshold or anomaly conditions.

### 40. What is the purpose of Dynatrace's `problem notification`?

Inform on-call/tools when Davis opens, updates, or closes a problem.

### 41. What does Dynatrace's `deployment event` track?

Application version releases and changes correlated with performance shifts.

### 42. What is the purpose of Dynatrace's `configuration event`?

Record configuration changes that may explain behavioral shifts.

### 43. What does Dynatrace's `availability event` indicate?

Service or endpoint availability degradation detected by Davis.

### 44. What is the purpose of Dynatrace's `custom event`?

User- or API-ingested events for context on dashboards and problem correlation.

### 45. What does Dynatrace's `error event` capture?

Increased error rates or error spikes on services or requests.

### 46. What is the purpose of Dynatrace's `performance event`?

Signal latency or response-time degradation on services or infrastructure.

### 47. What does Dynatrace's `resource contention event` indicate?

CPU, memory, disk, or network saturation affecting performance.

### 48. What is the purpose of Dynatrace's `slowdown event`?

Highlight significant response-time slowdowns as part of problem evidence.

### 49. What does Dynatrace's `unexpected high traffic event` indicate?

Abnormal traffic increase vs. baseline—possible load spike, attack, or misrouting.

### 50. What is the purpose of Dynatrace's `low traffic event`?

Abnormal traffic drop vs. baseline—possible outage, routing failure, or client issue.

---
