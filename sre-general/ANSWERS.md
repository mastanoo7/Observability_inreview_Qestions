# SRE General — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## SLI / SLO / SLA

### 1. Error budget at 80% consumed by day 20

- **Communicate immediately:** Publish a burn-rate dashboard showing projected budget exhaustion date, incidents contributing to burn, and business impact in plain language (not just percentages).
- **Stakeholder briefing:** Hold a short sync with product and leadership explaining that continued current velocity risks missing the monthly SLO; present options (slow releases, reliability sprint, scope deferral).
- **Adjust deployment velocity:** Implement a temporary deployment freeze or require reliability review for all changes until burn rate drops below sustainable levels (e.g., target <50% budget consumed at month midpoint).
- **Short-term reliability improvements:** Prioritize fixes for top error contributors, add circuit breakers on flaky dependencies, increase redundancy for critical paths, and roll back risky recent changes.
- **Freeze non-critical work:** Redirect engineering capacity from features to reliability tasks tied to the highest-impact failure modes.
- **Daily tracking:** Review error budget burn daily until back on track; document decisions in a shared channel for transparency.

### 2. Meaningful SLIs for a data pipeline (batch jobs)

- **Define user journeys:** Map which jobs affect which user-facing outcomes (e.g., billing reconciliation vs. analytics refresh) rather than treating the pipeline as monolithic.
- **Job-level SLIs:** Track per-job success rate, completion within SLA window, and data freshness (time since last successful run).
- **Tier jobs by impact:** Classify jobs as critical, important, or best-effort; weight SLOs accordingly (critical jobs get stricter targets).
- **Freshness SLI:** Measure end-to-end latency from data arrival to downstream availability (e.g., "95% of critical jobs complete within 2 hours of scheduled start").
- **Correctness SLI:** Track validation failures, row counts vs. expected, and downstream consumer errors caused by bad data.
- **Composite SLO:** Roll up tier-weighted job SLIs into a service-level SLO that reflects actual user impact, not binary "pipeline up/down."

### 3. Negotiating 99.999% SLA for a non-critical internal tool

- **Quantify cost:** Explain that five-nines (~5.26 minutes downtime/month) typically requires multi-region redundancy, automated failover, and 24/7 staffing—often 10–50× the cost of three-nines.
- **Anchor on user impact:** Ask who is affected, during what hours, and what happens when the tool is down for 30 minutes vs. 5 minutes.
- **Propose tiered SLOs:** Suggest 99.5–99.9% for internal tools with business-hours support, reserving five-nines for revenue-critical paths.
- **Show trade-offs:** Present a table of availability target vs. estimated infra cost, on-call burden, and delivery velocity impact.
- **Offer compensating controls:** Faster recovery (good runbooks, health checks) and clear communication during outages often deliver more value than marginal uptime for low-impact tools.
- **Document agreed SLO:** Capture the negotiated target in an SLA/SLO doc with explicit exclusions (maintenance windows, third-party deps).

### 4. SLO-based alerting with burn rate alerts

- **Define error budget:** For a 99.9% monthly SLO, error budget = 0.1% of requests (or ~43 minutes downtime equivalent).
- **Burn rate:** Measure how fast you're consuming budget (e.g., 14.4× burn = budget gone in ~2 days if sustained).
- **Multi-window:** Use short windows (e.g., 5m, 1h) to catch fast incidents and long windows (6h, 3d) to catch slow leaks—reduces false positives from brief spikes.
- **Multi-burn-rate:** Alert at 2×, 6×, and 14.4× burn rates with different severities (page vs. ticket vs. email).
- **Why superior to thresholds:** Simple "error rate > 1%" pages on noise; burn-rate alerts are proportional to SLO impact and align alerts with actual budget risk.
- **Implementation:** Use Prometheus recording rules or tools like Sloth/OpenSLO to generate multi-window burn-rate alert rules from SLO definitions.

### 5. Availability within SLO but latency breaching

- **Separate budgets:** Track availability and latency error budgets independently; a latency breach consumes the latency budget even if availability is fine.
- **Combined reporting:** For executive summaries, use a weighted composite or "worst of" policy defined upfront (e.g., either dimension breaching flags the service as unhealthy).
- **Incident communication:** State clearly which SLO is breached, user-visible symptoms (slow pages vs. errors), and that availability metrics alone understate impact.
- **Prioritize response:** Latency breaches often indicate saturation, dependency slowness, or GC issues—triage separately from availability root cause.
- **Post-incident:** Review whether latency SLO thresholds match user expectations and whether alerting should page on latency burn rate, not just availability.

### 6. SLO measurement for sync API + async background processing

- **Split SLIs by path:** API availability/latency for synchronous paths; job success rate and processing lag for async paths.
- **Link async to user impact:** Define SLIs on "time until user-visible outcome" (e.g., email delivered, report ready) not just queue depth.
- **Delayed failure attribution:** Tag async failures with the triggering API request or user session where possible for end-to-end tracing.
- **Synthetic checks:** Run periodic synthetic jobs through the async pipeline to detect silent failures before users report them.
- **Unified dashboard:** Show both real-time API SLO and async pipeline SLO with correlation views for incidents affecting both.

### 7. SLO measurement during planned maintenance

- **Policy decision:** Many orgs exclude pre-announced maintenance from error budget if users are notified; others count all downtime—document explicitly.
- **Technical implementation:** Use SLO exclusion windows (maintenance labels on time ranges) in your SLO tooling so burn-rate calculations skip those intervals.
- **Maintenance SLO:** Optionally track a separate "maintenance success" metric (completed on time, no unexpected extension).
- **Communication:** Publish maintenance windows in status pages; ensure alerts are suppressed or routed differently during planned work.
- **Avoid abuse:** Require approval and max duration for maintenance exclusions so teams don't hide outages as "maintenance."

### 8. Governance for team-defined SLOs

- **Lightweight template:** Provide a standard SLO worksheet (user journeys, SLIs, targets, owners) rather than a heavy approval committee.
- **Review at PRR:** Validate SLOs during production readiness review for new services; spot-check existing services quarterly.
- **Benchmark library:** Publish reference SLOs by service tier (critical, standard, internal) so teams don't guess in isolation.
- **Measurability gate:** Reject SLOs that can't be measured with existing instrumentation within one sprint of effort.
- **Alignment checks:** Ensure upstream/downstream SLOs are compatible (a 99.99% consumer can't depend on a 99.9% provider without caching/fallback).
- **Central dashboard:** Aggregate all team SLOs for leadership visibility without owning each team's day-to-day tuning.

---

## Incident Management & Response

### 9. P1 outage — 50% users affected, unknown root cause, 10 engineers

- **Declare incident:** Assign incident commander (IC), establish war room (video bridge + dedicated Slack channel), set severity P1.
- **Role structure:** IC coordinates; Operations Lead drives technical mitigation; Communications Lead handles status page and stakeholder updates; Scribe documents timeline; Subject-matter experts investigate in parallel workstreams.
- **Triage first:** Confirm scope (which regions, features, user segments), recent changes (deploys, config, infra), and external dependencies.
- **Parallel investigation:** Split engineers across hypotheses (network, database, recent deploy, third-party) with 15-minute sync checkpoints—avoid everyone debugging the same path.
- **Communication cadence:** Stakeholder updates every 15–30 minutes; status page updated when customer impact changes; internal channel for technical findings only.
- **Mitigation over root cause:** Prioritize restoring service (rollback, failover, traffic shed) before deep root-cause analysis during active outage.
- **Handoff discipline:** IC owns decisions; debrief within 24 hours for postmortem scheduling.

### 10. Primary monitoring (Prometheus) affected by outage

- **Secondary observability:** Fall back to cloud provider metrics, load balancer logs, `kubectl`/`docker` CLI, and application logs on nodes.
- **Synthetic checks:** Use external uptime monitors (Pingdom, Datadog synthetics) that run outside the affected cluster.
- **Minimal instrumentation:** Deploy ad-hoc exporters or `curl` health checks from bastion hosts if scrape path is broken.
- **Customer signals:** Monitor support tickets, social media, and sales reports for impact when dashboards are down.
- **Paper runbooks:** Ensure critical services have offline runbooks with key commands and escalation contacts.
- **Restore monitoring early:** Treat observability recovery as a parallel workstream—without metrics, mitigation is blind.

### 11. On-call rotation for global team (4 time zones)

- **Follow-the-sun:** Split primary on-call by region with overlapping handoff windows (30–60 min) for live transfer.
- **Handoff checklist:** Outgoing engineer documents open incidents, noisy alerts, in-flight deploys, and known risks in a standard handoff doc.
- **Escalation paths:** Primary → secondary (15 min) → engineering manager → domain expert; define per-severity timeouts.
- **Burnout prevention:** Cap pages per shift, compensate time off after heavy weeks, rotate fairly, and track alert quality (reduce noisy pages).
- **Secondary on-call:** Always staffed for P1; primary handles P2/P3 with next-day follow-up where appropriate.
- **Tooling:** PagerDuty/Opsgenie with timezone-aware schedules, automatic overrides for PTO, and integration with incident tooling.

### 12. Incident recurred 3 hours after incomplete fix

- **Verify fix in production:** Require confirmation that the fix is deployed to all regions/instances and config caches have refreshed before closing the incident.
- **Soak period:** For high-severity incidents, keep enhanced monitoring for 24–48 hours post-resolution.
- **Action item tracking:** Postmortem action items must have owners, due dates, and verification steps; track in a central system (Jira, incident.io).
- **Recurrence review:** If the same incident type repeats within 30 days, escalate to engineering leadership for reliability investment.
- **Definition of done:** Incident isn't closed until root cause is validated, fix is permanent (not a workaround), and monitoring detects recurrence.

### 13. Severity classification system

- **User-impact matrix:** Define P1–P4 by % users affected, data loss risk, revenue impact, and regulatory exposure—not by internal inconvenience.
- **Examples in runbook:** Provide concrete scenarios ("payment processing down" = P1; "internal dashboard slow" = P3).
- **Training:** Run tabletop exercises where engineers classify sample incidents under time pressure; review misclassifications in postmortems.
- **IC override:** Incident commander can upgrade severity; downgrades require explicit acknowledgment of residual risk.
- **Auto-suggest:** Use alert metadata (service tier, error rate) to suggest severity; human confirms during incident.

### 14. Managing a "slow burn" incident

- **Treat as incident early:** Declare when degradation trend threatens SLO even if nothing is "down" yet—slow burns consume error budget silently.
- **Trend dashboards:** Watch derivative metrics (latency p99 trend, queue growth rate, disk fill rate) not just thresholds.
- **Time-boxed investigation:** Assign owner with 2-hour checkpoints; escalate if no hypothesis or if trend accelerates.
- **Proactive mitigation:** Scale up, shed load, or restart unhealthy components before hard failure.
- **Stakeholder comms:** Warn product/leadership that SLO may breach even if current UX is "acceptable."

### 15. Automated incident detection with correlation

- **Alert grouping:** Use alertmanager inhibition, grouping by service/cluster, and deduplication keys to collapse related fires.
- **Event correlation:** Tools like BigPanda, Moogsoft, or custom logic correlate deploy events + error spikes + saturation on one timeline.
- **Composite alerts:** Define SLO burn-rate or "symptom-based" alerts (e.g., elevated 5xx AND elevated latency) as single pages.
- **ML/anomaly optional:** Layer anomaly detection on top of SLO-based rules; tune carefully to avoid opaque false positives.
- **Runbook links:** Every correlated alert includes service owner, runbook URL, and recent deploy info for actionable response.

---

## Error Budgets

### 16. Burning budget in first week due to aggressive deployments

- **Automated policy:** Tie deploy pipeline gates to remaining budget (e.g., >50% remaining = normal; 25–50% = reduced velocity; <25% = freeze except hotfixes).
- **Weekly budget review:** Team reviews burn contributors in standup when budget consumption exceeds linear pace.
- **Deployment batching:** Reduce deploy frequency; require canary + automated rollback on SLO regression.
- **Feature flags:** Decouple deploy from release so code ships safely but risky features stay off until budget recovers.
- **Leadership alignment:** Product agrees that exhausting budget early means feature freeze for remainder of month.

### 17. Error budget with third-party API dependencies

- **Measure separately:** Track "internal" SLI (your code + infra) vs. "end-to-end" SLI (includes dependencies); report both.
- **Dependency SLOs:** Contractually or informally track provider SLOs; exclude provider outages from your budget only if contractually defined (vendor credits).
- **Resilience investment:** Cache, fallback, async retry, and graceful degradation reduce budget impact from external failures you can't control.
- **Blameless attribution:** Postmortems tag third-party vs. internal root cause for capacity planning and vendor escalation.
- **Budget policy:** Document whether third-party failures count against team budget—usually yes for user-facing SLO, with exec escalation for vendor issues.

### 18. Single large incident consumes entire monthly budget

- **Incident exception process:** Allow one-time leadership approval to reset or borrow budget for genuine catastrophic incidents with documented remediation plan.
- **Rolling windows:** Use 28-day rolling SLO windows instead of calendar month so one bad day doesn't block entire calendar month.
- **Separate "freeze" from "budget":** Deployment freeze is a policy decision; technically you can still deploy with explicit risk acceptance after postmortem actions are underway.
- **Investment trigger:** Full budget loss triggers mandatory reliability review and resourcing, not permanent punishment.
- **Insurance metrics:** Track "budget remaining after largest incident" as a resilience KPI over quarters.

### 19. Error budget alerts at 2%, 5%, 10% burn rate per hour

- **2%/hour (slow burn):** Slack/email to team; increase monitoring; no deploy freeze; investigate trending issues within 24 hours.
- **5%/hour (moderate):** Page on-call; halt non-critical deploys; daily leadership visibility; assign reliability owner for the week.
- **10%/hour (fast burn):** Page IC and management; full deploy freeze except hotfixes; war room if customer-facing; executive comms prepared.
- **Implementation:** Map burn rates to multi-window alerts (Google SRE workbook: 14.4×, 6×, 1× for different windows).
- **Runbook actions:** Each threshold links to predefined actions so on-call doesn't debate policy during incident.

### 20. Error budget reporting for quarterly business review

- **Trend charts:** Monthly budget consumption %, deploy count, incident count, and top burn contributors.
- **Incident attribution:** Table of incidents with budget consumed, root cause category, and action item completion status.
- **Deploy correlation:** Scatter or time-series showing deploy frequency vs. budget burn to justify velocity changes.
- **Team comparison:** Normalized view across services (tier-adjusted) for portfolio-level reliability narrative.
- **Forward look:** Projected budget risk based on current burn rate and planned launches.

---

## Production Outages & Kubernetes Failures

### 21. etcd lost quorum — API server unavailable

- **Stop the bleeding:** Prevent further writes; identify which etcd members are healthy vs. corrupted; check disk I/O and latency on all etcd nodes.
- **Do not force recovery blindly:** Quorum loss requires careful procedure—documented restore from snapshot is often safer than split-brain recovery.
- **Restore from backup:** Use latest consistent etcd snapshot (`etcdctl snapshot restore`); verify snapshot age against RPO; restore to new members if disks are corrupted.
- **Rebuild members:** If nodes are unhealthy, provision clean etcd members, join cluster per Kubernetes docs, one member at a time.
- **Validate cluster state:** Once API server is up, verify all nodes, critical pods, and CRDs; check for stale resources or orphaned objects.
- **Post-recovery:** Root-cause disk saturation (IOPS limits, noisy neighbor); add monitoring on etcd latency, db size, and defragmentation schedule.
- **Prevention:** Dedicated etcd hardware/VMs, regular snapshots, automated backup testing, and PDBs for control plane components.

### 22. Cascading failure — evictions causing more load

- **Break the loop:** Temporarily scale up replicas manually or raise resource limits if misconfigured; stop non-critical workloads.
- **Identify trigger:** OOM, CPU throttle, or node pressure—check `kubectl describe node` and pod events for eviction reasons.
- **Cordon problematic nodes:** Prevent new scheduling onto unhealthy nodes; drain gracefully if safe.
- **Adjust HPA/cluster autoscaler:** Prevent aggressive scale-in during instability; widen cooldown periods.
- **Rate-limit evictions:** Review PodDisruptionBudgets and priority classes so critical pods aren't all evicted simultaneously.
- **Load shedding:** Enable circuit breakers, queue backpressure, or 503 responses at gateway to protect remaining capacity.

### 23. Node NotReady — pods not rescheduling

- **Diagnose node:** SSH or cloud console—check kubelet logs, disk pressure, memory pressure, CNI failures, and certificate expiry.
- **Verify controller behavior:** NotReady doesn't always trigger immediate eviction—check `--pod-eviction-timeout` and whether node is cordoned.
- **Safe eviction:** `kubectl cordon <node>` then `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` with PDB respect.
- **Force only if needed:** After timeout, pods should evict; if stuck terminating, investigate finalizers and volume attachments.
- **Replace node:** If hardware/kubelet unrecoverable, terminate node and let autoscaler/managed group replace it.
- **Prevention:** Node problem detector, proactive alerts on NotReady, and automated remediation playbooks.

### 24. Intermittent CoreDNS failures

- **Symptoms:** Random `NXDOMAIN` or timeouts service-to-service; often correlates with CoreDNS pod restarts or high QPS.
- **Check CoreDNS health:** `kubectl logs -n kube-system -l k8s-app=kube-dns`, resource limits, and HPA if DNS QPS is high.
- **Upstream resolvers:** Verify forward plugin config and upstream DNS reliability; test with `dig` from debug pods.
- **Node-local DNS cache:** Deploy node-local-dns to reduce CoreDNS load and improve cache hit rate.
- **Network policies:** Ensure policies aren't blocking DNS (UDP/TCP 53) between namespaces.
- **Load test DNS:** Use `dnsperf` or application-level metrics on DNS latency; scale CoreDNS replicas and CPU.

### 25. Deployment stuck — new pods failing health checks

- **Inspect failing pods:** `kubectl describe pod`, logs, events—distinguish readiness vs. liveness failures.
- **Compare old vs. new:** Image tag, env vars, secrets, config mounts, and resource limits differences.
- **Check probes:** Probe path, port, initialDelaySeconds, and timeout—overly aggressive probes cause false failures.
- **Rollback:** `kubectl rollout undo deployment/<name>` or revert GitOps commit; verify old revision healthy.
- **Pause rollout:** `kubectl rollout pause` while investigating to prevent further bad replicas.
- **Fix forward:** Only after root cause identified; use canary or manual single-replica test before full rollout.

### 26. Pod Disruption Budgets (PDBs)

- **Define minAvailable or maxUnavailable:** For a 10-replica service with 99.9% SLO, `minAvailable: 9` or `maxUnavailable: 1` during voluntary disruptions.
- **Align with SLO:** Calculate minimum replicas needed to handle peak load during disruption.
- **Apply to critical tiers:** All production user-facing deployments and stateful sets should have PDBs.
- **Test during drills:** Node drain exercises must respect PDBs—verify drain completes without breaching capacity.
- **Balance with cluster upgrades:** Coordinate cluster upgrades with PDB windows; may need temporary scale-up before maintenance.

---

## CI/CD Failures & Deployment Safety

### 27. Config change caused outage despite passing tests

- **Separate config pipeline:** Config changes require the same review, canary, and approval as code; store config in Git with PR review.
- **Schema validation:** Validate config against JSON Schema or OpenAPI at CI time; reject invalid values before deploy.
- **Environment parity:** Staging must use production-like config structure and feature flag states.
- **Config diff in deploy:** Show human-readable diff in deploy logs; require explicit approval for production config-only PRs.
- **Automated smoke tests:** Post-deploy synthetics that exercise config-dependent paths (auth, feature toggles, rate limits).
- **Blast radius limits:** Progressive rollout of config via canary namespaces or percentage-based config service.

### 28. Progressive delivery for irreversible schema migration

- **Expand-contract pattern:** Add new columns/tables first (backward compatible); dual-write; migrate reads; remove old schema only after validation.
- **Feature flags:** Gate application code paths that use new schema; roll back behavior without rolling back schema.
- **Canary by traffic:** Route small % of traffic to instances on new schema; monitor errors and latency.
- **Shadow reads:** Read from new schema in background, compare results, don't serve until consistent.
- **Backup and restore drill:** Full backup before migration; tested rollback script even if "can't roll back"—document data repair procedures.
- **Maintenance window:** For truly breaking changes, coordinate read-only mode and communicate downtime budget.

### 29. Automated rollback within 5 minutes of deploy

- **Golden signals:** Monitor error rate, latency p99, and saturation in 1–5 minute windows post-deploy.
- **Comparison to baseline:** Compare canary/new version metrics to previous version or pre-deploy baseline (Mann-Whitney or simple threshold).
- **Pipeline integration:** Argo Rollouts, Flagger, or Spinnaker automated promotion/rollback based on Prometheus queries.
- **Rollback artifact:** Keep previous container image and config revision one-click deployable; test rollback monthly.
- **Alert on rollback:** Page team when auto-rollback fires—rollback is mitigation, not resolution.
- **Guardrails:** Don't auto-rollback on low-traffic noise—require minimum request volume for statistical confidence.

### 30. 45-minute pipeline — optimization without sacrificing safety

- **Profile the pipeline:** Identify longest stages (integration tests, image builds, security scans); parallelize independent stages.
- **Test pyramid:** Move slow E2E tests to post-deploy canary validation; keep fast unit/integration in blocking path.
- **Caching:** Docker layer cache, dependency cache, and incremental test selection based on changed files.
- **Smaller batches:** Faster pipeline enables more frequent small deploys, reducing per-deploy risk.
- **Don't cut safety:** Keep SLO checks, security scan, and canary analysis—optimize duration, not remove gates.
- **Target:** Aim for <15 minute commit-to-production for mature teams; measure deployment frequency and change failure rate.

---

## Scaling Strategies

### 31. Thundering herd / cache stampede prevention

- **Probabilistic early expiration:** Jitter TTL per key so not all keys expire simultaneously.
- **Lock or single-flight:** Only one request regenerates cache; others wait or serve stale (memcached `get` + lock, or `groupcache`).
- **Stale-while-revalidate:** Serve stale value immediately while async refresh happens in background.
- **Pre-warm:** Refresh popular keys before expiry via background job.
- **Request coalescing:** Deduplicate in-flight identical requests at application layer.
- **Rate limit origin:** Cap concurrent cache-miss fetches to downstream DB/API.

### 32. Auto-scaling for highly variable traffic

- **Metric selection:** Prefer custom metrics (queue depth, requests/sec, latency) over CPU alone for IO-bound or bursty workloads.
- **Multiple policies:** Scale on RPS for normal traffic; scale on queue lag for async; use predictive scaling for known patterns (daily peaks).
- **Scale-out fast, scale-in slow:** Aggressive scale-up cooldown (0–60s); conservative scale-down (5–15 min) to avoid flapping.
- **Minimum replicas:** Set floor high enough for baseline + failure of one AZ.
- **Pre-warm for events:** Scheduled scaling before known events (marketing launch, Black Friday).
- **Test scale-in:** Verify PDBs and connection draining during scale-in drills.

### 33. Connection pooling for 500 instances → PostgreSQL

- **PgBouncer (or RDS Proxy):** Central pooler between apps and DB; use transaction pooling mode where compatible.
- **Right-size pool per service:** Each instance needs few connections (e.g., 5–10); 500 × 10 = 5000 is too many—pooler multiplexes to ~100–200 DB connections.
- **Connection limits:** Set PostgreSQL `max_connections` based on instance size; enforce at pooler.
- **Read replicas:** Offload read traffic; separate pools for read vs. write.
- **Monitor pool saturation:** Alert on pool wait time and connection exhaustion.
- **Avoid per-request connections:** Use persistent pools in application lifecycle, not per HTTP request.

### 34. Hot partition in DynamoDB/Cassandra

- **Diagnose:** CloudWatch consumed capacity per key, Cassandra `nodetool tablestats`, trace high-cardinality partition keys.
- **Redesign partition key:** Include high-cardinality suffix (user_id, shard_id) to spread writes; avoid monotonic keys (timestamp-only).
- **Write sharding:** Append random suffix to hot key and scatter writes; aggregate on read.
- **Caching layer:** Absorb read heat on hot keys with Redis/DAX.
- **Adaptive capacity / auto-scaling:** Enable for DynamoDB; still fix key design for sustained hot spots.
- **Load test:** Reproduce spike in staging with realistic key distribution before production events.

### 35. Global load balancing for latency-sensitive app

- **GeoDNS or anycast:** Route users to nearest healthy region via Route53 latency routing, Cloudflare, or GCP Global LB.
- **Health checks:** Per-region health checks remove unhealthy regions from DNS/LB pool within TTL bounds.
- **Active-active with data:** Replicate data with acceptable consistency (CRDTs, eventual sync, or regional shards).
- **Failover:** Automated or manual failover runbook; test quarterly; define RTO for DNS TTL propagation.
- **Data residency:** Route EU users to EU region only if required by regulation—may override pure latency optimization.
- **Observability per region:** Regional SLO dashboards to detect partial outages before global impact.

---

## Disaster Recovery

### 36. Primary data center complete power failure — DR activation

- **Assess and declare:** Confirm scope and expected duration; IC and exec approve DR activation per runbook criteria.
- **Failover DNS/traffic:** Shift global LB or DNS to DR region; reduce TTL beforehand if predictable risk.
- **Activate DR stack:** Start compute, restore latest DB replica or snapshot, apply pending replication lag catch-up.
- **Validate data:** Check replication lag, integrity checks, and critical read/write paths before accepting traffic.
- **Gradual traffic shift:** Canary 5% → 25% → 100% with monitoring at each step.
- **Communicate:** Status page, customer comms, internal war room; document timeline for postmortem.
- **Failback plan:** Don't fail back until primary stable; rehearse failback separately.

### 37. DR plan for 1-hour RPO, 4-hour RTO — with testing

- **Architecture:** Continuous async replication to DR site; backups every 15–30 min for additional safety net.
- **Automation:** Infrastructure-as-code for DR environment; one-click or scripted failover playbook.
- **RPO validation:** Measure replication lag under load; alert if lag approaches 45 minutes.
- **RTO validation:** Quarterly game day: timed failover from declaration to serving traffic; record actual RTO.
- **Runbook:** Step-by-step with owners, dependencies, and rollback; store offline copy.
- **Improvement loop:** Each test produces action items; track RTO trend toward 4-hour target.

### 38. Multi-region active-active for database-backed app

- **Data model choice:** Multi-master (conflict resolution), active-passive with fast failover, or CRDT/event sourcing depending on consistency needs.
- **Conflict resolution:** Last-write-wins, vector clocks, or domain-specific merge rules; avoid silent data loss.
- **Consistency tiers:** Strong consistency within region; eventual across regions with user-facing lag disclosure where needed.
- **Traffic routing:** Geo-sticky sessions or global load balancer with health-aware routing.
- **Failover:** Automated detection of region failure; drain traffic; promote secondary if needed.
- **Testing:** Chaos experiments (region isolation) and regular failover drills.

### 39. Chaos engineering for DR validation

- **Hypothesis-driven:** "If region A fails, we fail over in <4 hours with <1 hour data loss."
- **Experiments:** AZ failure simulation, DNS failover test, database primary kill, network partition.
- **Blast radius:** Start in staging; then production with limited scope (one AZ, internal traffic only).
- **Metrics:** Measure detection time, failover duration, error rate during experiment, data consistency post-recovery.
- **Game days:** Cross-team scheduled exercises with exec observers.
- **Safety:** Abort switches, monitoring dashboards, and rollback procedures mandatory.

### 40. DR test took 8 hours instead of 4 — systematic improvement

- **Timeline reconstruction:** Break recovery into phases (detection, decision, infra start, DB restore, traffic shift, validation) and time each.
- **Identify bottlenecks:** Common culprits—manual approvals, untested scripts, large DB restore, missing credentials, DNS TTL.
- **Automate manual steps:** Script failover, pre-provision DR capacity, use warm standby instead of cold.
- **Parallelize:** Run independent recovery tasks concurrently with clear dependencies.
- **Retest incrementally:** Fix largest bottleneck first; re-run partial tests until full 4-hour target met.
- **Track RTO as SLO:** Report DR RTO trend quarterly to leadership.

---

## Observability Architecture

### 41. Observability for 500-service microservices platform

- **Platform team owns the stack:** Centralize Prometheus/Mimir, Loki/Elasticsearch, Tempo/Jaeger with standardized agents/operators.
- **Golden path instrumentation:** Provide SDKs, sidecars, and Helm charts so teams get metrics/logs/traces with minimal config.
- **Cardinality governance:** Enforce label conventions, recording rules, and drop high-cardinality labels at ingestion.
- **Tiered retention:** Hot storage 7–14 days, warm 30–90 days, cold archive for compliance; sample traces in production.
- **Self-service dashboards:** Template Grafana dashboards per service type; teams customize within guardrails.
- **Cost allocation:** Chargeback or showback by team/namespace; quotas on log volume and trace sampling.

### 42. Unified observability — Prometheus + Elasticsearch + Jaeger

- **Correlation IDs:** Propagate trace_id/request_id across logs, metrics (exemplars), and spans.
- **Exemplars:** Link Prometheus histogram buckets to trace IDs for latency drill-down.
- **Single UI:** Grafana with data source linking, or commercial platforms (Datadog, Dynatrace) if budget allows.
- **Structured logging:** JSON logs with trace_id, service, version fields for Elasticsearch queries.
- **Service graph:** Generate dependency map from trace data for incident blast-radius analysis.
- **On-call integration:** Alert links open pre-filtered views across all three pillars.

### 43. Observability for serverless

- **Managed instrumentation:** Use cloud-native (X-Ray, Cloud Trace, Application Insights) and OpenTelemetry Lambda layers.
- **Structured stdout:** JSON logs to CloudWatch/Stackdriver; avoid relying on host agents.
- **Cold start metrics:** Track init duration, memory, and concurrency as first-class SLIs.
- **Distributed tracing:** Propagate context through API Gateway, queues, and step functions.
- **Sampling:** High-volume functions need aggressive trace sampling with tail-based sampling for errors.
- **Debugging:** Enable ephemeral debug logging levels via environment config without redeploy where possible.

### 44. Observability for multi-stage data pipeline

- **Event tracing:** Assign unique event_id at ingestion; propagate through every stage in metadata.
- **Stage SLIs:** Per-stage processing time, success rate, and backlog depth.
- **Dead letter monitoring:** Alert on DLQ growth with sample payload inspection.
- **End-to-end latency:** Measure time from ingest to final sink; SLO on pipeline freshness.
- **Data quality checks:** Embed validation stages with metrics on pass/fail rates.
- **Lineage visualization:** DAG dashboard showing where events stall or fail.

---

## Reliability Engineering & Postmortems

### 45. Blameless postmortem process

- **Blameless culture:** Focus on systems and processes; use "how did the system allow this?" not "who made the mistake?"
- **Timeline-first:** Detailed incident timeline before root cause discussion.
- **Five whys / contributing factors:** Multiple contributing factors, not single root cause.
- **Action items:** SMART items with owners and due dates; tracked in central system with escalation for overdue items.
- **Review meeting:** Cross-functional attendance; publish internally (minus sensitive details).
- **Effectiveness review:** 30/60/90-day check that action items reduced recurrence risk.

### 46. Reliability review for new features

- **PRR checklist:** SLO defined, monitoring/alerting, runbooks, capacity estimate, failure modes, rollback plan, security review.
- **Tiered requirements:** Stricter checklist for tier-1 services; lighter for internal tools.
- **Gate in CI/CD:** Production deploy blocked until PRR signed off in ticket system.
- **Waiver process:** Explicit risk acceptance by engineering + product leadership with expiry date.
- **Templates:** Reusable architecture patterns (caching, idempotency, timeouts) documented in platform catalog.

### 47. Improving MTTD and MTTR

- **MTTD improvements:** Better alerts (SLO burn rate), synthetic monitoring, anomaly detection, and reducing alert noise so real issues aren't missed.
- **MTTR improvements:** Runbooks, automated rollback, well-practiced incident response, and clear ownership.
- **Measure both:** Track distributions, not just averages; segment by service tier and incident type.
- **Biggest MTTD wins:** Fix paging gaps, add client-side metrics, and deploy observability to blind spots.
- **Biggest MTTR wins:** Safe rollback automation, infrastructure immutability, and game-day practice.

### 48. Five incidents, different root causes — systemic vs. random

- **Affinity analysis:** Group by failure mode category (deploy, dependency, capacity, human process, security).
- **If scattered:** May be immature service needing holistic investment (testing, observability, architecture review).
- **If patterns:** e.g., three deploy-related → invest in deployment pipeline and canary.
- **Error budget trend:** Persistent burn despite "different" causes suggests systemic under-investment in reliability.
- **Prioritization:** Use risk × frequency × impact matrix; address highest expected value fixes first.
- **Stop the bleeding:** Sometimes one architectural fix (bulkhead, timeout, idempotency) prevents multiple failure modes.

---

## Cloud-Native Troubleshooting

### 49. Intermittent production-only failures

- **Production-safe profiling:** Continuous profiling (Parca, Pyroscope), eBPF-based tools, and sampled distributed tracing.
- **Compare environments:** Config diff, traffic patterns, data volume, and dependency versions prod vs. staging.
- **Feature flag experiments:** Enable debug logging for single user/session cohort in prod.
- **Traffic capture:** Service mesh tap, tcpdump on specific pods (brief), or request mirroring to shadow environment.
- **Correlation:** Link failures to deploy times, batch jobs, traffic spikes, or specific regions/AZs.
- **No restart required:** Prefer `/debug/pprof`, OpenTelemetry dynamic sampling, and log level changes via config reload.

### 50. Comprehensive health check strategy

- **Startup probe:** For slow-starting apps; prevents liveness killing during initialization.
- **Readiness probe:** Remove pod from service endpoints when not ready (DB down, warming cache).
- **Liveness probe:** Restart only on deadlock/hung process—not on temporary dependency failure.
- **Tune timing:** `initialDelaySeconds` and `failureThreshold` based on measured startup and recovery times.
- **Check real dependencies:** Readiness should verify critical deps; liveness should be lightweight (HTTP /healthz).
- **Avoid false positives:** Don't include external API calls in liveness; use timeouts > p99 dependency latency.

### 51. Distributed global rate limiting

- **Central store:** Redis, Envoy global rate limit service, or API gateway with shared token bucket (Lua, GCRA algorithm).
- **Consistent hashing:** Partition rate limit keys across Redis cluster for scale.
- **Headers:** Return `X-RateLimit-Remaining` and `Retry-After` for client backoff.
- **Hierarchy:** Global limit + per-user + per-IP limits at different layers.
- **Fail-open vs. closed:** Document policy when limiter is unavailable—usually fail-open for availability with monitoring alert.
- **Testing:** Load test limiter itself; it's now a critical dependency.

### 52. Managed database degraded performance

- **Triangulate:** Check cloud provider status, RDS/Cloud SQL metrics (CPU, IOPS, connections, replication lag), and slow query logs.
- **Application side:** Connection pool exhaustion, N+1 queries, missing indexes, traffic spike correlation.
- **Network:** Latency from app to DB (traceroute, VPC flow logs, security group changes).
- **Compare baselines:** Same query performance from bastion vs. app tier isolates network vs. DB.
- **Scaling:** Temporarily scale instance class, add read replica, or kill long-running queries.
- **Vendor ticket:** Open support case with evidence if provider-side issue suspected.

### 53. Cost optimization for $50k/month observability stack

- **Retention tuning:** Shorten log/trace retention; aggregate metrics with recording rules.
- **Sampling:** Head-based trace sampling 1–10%; tail-based for errors; drop debug logs in production.
- **Cardinality audit:** Remove high-cardinality labels (user_id on metrics); use logs for detail.
- **Tiered services:** Full observability for tier-1; basic metrics-only for internal/low-criticality.
- **Storage class:** Move cold data to S3/GCS with query-on-demand (Grafana Mimir, Thanos).
- **Review licenses:** Right-size commercial tool seats and ingest limits; negotiate or split vendors.

### 54. Service mesh network observability

- **Mesh metrics:** Request rate, errors, latency by source/destination (Istio telemetry, Linkerd viz).
- **Access logs:** Envoy access logs to Loki with mTLS identity and response flags.
- **Distributed traces:** Mesh-generated spans for retries, timeouts, and circuit breaker events.
- **Control plane health:** Monitor istiod/linkerd control plane; config push errors cause subtle failures.
- **Dashboards:** Golden signals per service pair; alert on elevated 5xx between specific dependencies.
- **Debug tools:** `istioctl proxy-config`, tap, and ephemeral debug containers for packet-level issues.

### 55. Production memory leak — OOMKilled every 6 hours

- **Continuous profiling:** Deploy Parca/Pyroscope/Cloud Profiler with low overhead; compare heap allocations over time.
- **Heap dump on OOM:** Configure `GOTRACEBACK` or JVM `-XX:+HeapDumpOnOutOfMemoryError` to object storage before kill.
- **Increase temporarily:** Raise memory limit short-term while investigating to reduce incident frequency.
- **Canary reproduction:** Run production traffic shadow or load test with production data shapes in isolated env.
- **eBPF tools:** bpftrace/bcc for allocation tracking without code changes.
- **Roll out fix:** Patch leak, verify heap stable over 24–48 hours in canary before full promotion.

---

## Basic Questions

### 1. What is Site Reliability Engineering (SRE) and how does it differ from traditional operations?

Site Reliability Engineering applies software engineering practices to operations problems: automation, SLO-driven prioritization, and error budgets. Unlike traditional ops focused on keeping systems running at any cost, SRE balances reliability with feature velocity using measurable targets. SRE teams typically write code, own services end-to-end, and treat operations work as a engineering problem to eliminate (toil).

### 2. What is the difference between an SLI, SLO, and SLA?

An SLI (Service Level Indicator) is a measured metric such as availability or latency. An SLO (Service Level Objective) is an internal target for that SLI (e.g., 99.9% availability). An SLA (Service Level Agreement) is a contractual commitment to customers, often with financial penalties, usually stricter than or equal to the SLO.

### 3. What is an error budget and how is it calculated?

An error budget is the allowable unreliability within an SLO period. For a 99.9% monthly availability SLO, the error budget is 0.1% of time (~43 minutes per month) or equivalently 0.1% of failed requests, depending on how the SLI is defined.

### 4. What is the purpose of a postmortem in SRE?

A postmortem documents what happened during an incident, why it happened, and how to prevent recurrence—without blaming individuals. It produces actionable improvements and shares learning across the organization.

### 5. What is the difference between MTTR and MTTF?

MTTR (Mean Time To Repair/Recover) is the average time to restore service after a failure. MTTF (Mean Time To Failure) is the average time between failures in repairable systems—the operational counterpart to MTBF for non-repairable components.

### 6. What is toil in SRE and why is it important to reduce it?

Toil is manual, repetitive, automatable operational work that scales linearly with service growth and provides no enduring value. Reducing toil frees engineers for engineering projects that improve reliability permanently and prevents ops teams from becoming bottlenecks.

### 7. What is the purpose of a runbook in incident management?

A runbook provides step-by-step procedures for diagnosing and mitigating known failure modes. It enables faster, more consistent incident response, especially for on-call engineers unfamiliar with every subsystem.

### 8. What is the difference between a P1 and P2 incident?

P1 indicates critical user or business impact requiring immediate all-hands response (e.g., major outage). P2 indicates significant degradation with workaround possible, requiring prompt response but typically not waking the entire organization at 3 AM unless policy dictates.

### 9. What is the purpose of on-call rotation in SRE?

On-call ensures qualified engineers are available to respond to production issues 24/7. Rotations distribute burden fairly, maintain accountability, and provide escalation paths when automated systems detect problems.

### 10. What is a change freeze and when would you implement one?

A change freeze pauses non-essential production changes during high-risk periods (holidays, major events) or when error budget is exhausted. Emergency hotfixes are still allowed with extra scrutiny.

### 11. What is the difference between availability and reliability?

Availability measures whether a system is operational and reachable (uptime). Reliability is broader: the probability the system performs its intended function correctly over time, including correctness and durability, not just "up."

### 12. What is a canary deployment and why is it used?

A canary deploys a new version to a small subset of traffic before full rollout. It limits blast radius and allows automated or manual validation of the new version against real production load.

### 13. What is the purpose of a health check endpoint in a service?

A health check endpoint reports whether the service and its critical dependencies are functioning. Load balancers and orchestrators use it to route traffic only to healthy instances.

### 14. What is the difference between a liveness probe and a readiness probe in Kubernetes?

A liveness probe determines if the container is alive; failure restarts the container. A readiness probe determines if the container should receive traffic; failure removes it from Service endpoints without restarting.

### 15. What is a circuit breaker pattern and when would you use it?

A circuit breaker stops calls to a failing dependency after a threshold, failing fast and allowing recovery. Use it when cascading failures from flaky downstream services threaten overall system stability.

### 16. What is the purpose of a load balancer in a distributed system?

A load balancer distributes traffic across multiple instances for capacity and fault tolerance. It performs health checks and can terminate TLS, enabling horizontal scaling and zero-downtime deploys.

### 17. What is the difference between horizontal and vertical scaling?

Horizontal scaling adds more instances (scale out). Vertical scaling increases resources on existing instances (scale up). Horizontal is preferred for cloud-native resilience; vertical has hardware limits.

### 18. What is a blue-green deployment and what are its benefits?

Blue-green maintains two identical environments; traffic switches from old (blue) to new (green) atomically. Benefits include instant rollback, full pre-production validation of green, and reduced deployment risk.

### 19. What is the purpose of a feature flag in software deployment?

Feature flags decouple deployment from release, enabling gradual rollout, A/B testing, and instant disable of problematic features without redeploying code.

### 20. What is the difference between RTO and RPO in disaster recovery?

RPO (Recovery Point Objective) is the maximum acceptable data loss measured in time. RTO (Recovery Time Objective) is the maximum acceptable downtime to restore service.

### 21. What is a chaos experiment and what is its purpose?

A chaos experiment intentionally injects failure to test system resilience and validate assumptions. Its purpose is to find weaknesses before they cause real outages.

### 22. What is the purpose of a service mesh in a microservices architecture?

A service mesh provides uniform traffic management, security (mTLS), and observability via sidecar proxies without embedding that logic in every service.

### 23. What is the difference between a hard dependency and a soft dependency?

A hard dependency is required for core functionality—if it fails, the service cannot operate correctly. A soft dependency enhances functionality but the service can degrade gracefully without it.

### 24. What is the purpose of rate limiting in a distributed system?

Rate limiting protects services from overload, ensures fair resource sharing among tenants, and prevents abuse. It stabilizes systems during traffic spikes or attacks.

### 25. What is the difference between synchronous and asynchronous communication in microservices?

Synchronous communication (HTTP/RPC) blocks the caller until a response is received, creating tight coupling and latency chains. Asynchronous communication (queues, events) decouples services in time, improving resilience but adding complexity for consistency.

### 26. What is a dead letter queue and when would you use it?

A dead letter queue holds messages that failed processing after retries. Use it to prevent poison messages from blocking the main queue and to enable inspection and reprocessing.

### 27. What is the purpose of a retry policy with exponential backoff?

Retries handle transient failures; exponential backoff increases delay between attempts to avoid overwhelming a recovering dependency. Jitter prevents synchronized retry storms.

### 28. What is the difference between a rolling update and a recreate deployment strategy?

Rolling update gradually replaces old pods with new ones, maintaining availability. Recreate terminates all old pods before starting new ones, causing downtime but useful when only one version can run at a time.

### 29. What is the purpose of a pod disruption budget (PDB) in Kubernetes?

A PDB limits concurrent voluntary disruptions (drains, evictions) to ensure a minimum number of pods remain available. It protects availability during maintenance and cluster upgrades.

### 30. What is the difference between a Kubernetes deployment and a statefulset?

Deployments manage stateless pods with interchangeable replicas and random pod names. StatefulSets provide stable network identity, ordered deployment, and persistent storage for stateful applications.

### 31. What is the purpose of resource requests and limits in Kubernetes?

Requests guarantee minimum resources for scheduling. Limits cap maximum CPU/memory usage to prevent a pod from starving others or causing node instability.

### 32. What is a Kubernetes namespace and why is it used?

A namespace provides logical isolation of resources within a cluster for multi-tenancy, RBAC boundaries, and resource quotas. It prevents naming collisions and organizes environments (dev, staging, prod).

### 33. What is the purpose of a Kubernetes ConfigMap?

A ConfigMap stores non-sensitive configuration data as key-value pairs. It decouples configuration from container images and allows config updates without rebuilding images.

### 34. What is the difference between a Kubernetes Secret and a ConfigMap?

Secrets store sensitive data (credentials, tokens) with base64 encoding and optional encryption at rest. ConfigMaps store non-sensitive configuration; neither should be treated as strong encryption without additional controls.

### 35. What is the purpose of a Kubernetes HorizontalPodAutoscaler (HPA)?

HPA automatically scales the number of pod replicas based on observed metrics (CPU, memory, or custom metrics). It matches capacity to demand without manual intervention.

### 36. What is a Kubernetes node affinity and when would you use it?

Node affinity attracts or repels pods to specific nodes based on labels. Use it for workload isolation (GPU nodes), compliance (data residency), or co-locating related services.

### 37. What is the purpose of a Kubernetes PersistentVolume (PV)?

A PV is cluster-level storage provisioned by an administrator or storage class. It provides durable storage that outlives individual pods for stateful workloads.

### 38. What is the difference between a Kubernetes Service of type ClusterIP, NodePort, and LoadBalancer?

ClusterIP exposes the service on an internal cluster IP only. NodePort exposes it on each node's IP at a static port. LoadBalancer provisions an external cloud load balancer for internet/client access.

### 39. What is the purpose of a Kubernetes Ingress resource?

Ingress manages external HTTP/HTTPS routing to cluster services with host/path rules, TLS termination, and load balancing. It provides L7 routing beyond basic Service types.

### 40. What is the difference between a Kubernetes Job and a CronJob?

A Job runs one or more pods to completion for batch work. A CronJob schedules Jobs on a cron-like timetable for recurring batch tasks.

### 41. What is the purpose of a Kubernetes DaemonSet?

A DaemonSet ensures one pod copy runs on every (or selected) node. Use it for node-level agents: log collectors, monitoring exporters, or CNI plugins.

### 42. What is the difference between a Kubernetes rolling update and a blue-green deployment?

Kubernetes rolling update gradually replaces pods in-place within one Deployment. Blue-green uses two full environments and switches traffic atomically—often implemented with two Deployments or external traffic management.

### 43. What is the purpose of a Kubernetes NetworkPolicy?

NetworkPolicy controls pod-to-pod and pod-to-external traffic using label-based firewall rules. It implements zero-trust segmentation within the cluster.

### 44. What is the difference between a Kubernetes StatefulSet and a Deployment?

StatefulSets provide ordered scaling, stable hostnames, and persistent volume binding per pod. Deployments treat pods as fungible with no guaranteed identity or storage stickiness.

### 45. What is the purpose of a Kubernetes ServiceAccount?

A ServiceAccount provides an identity for pods to authenticate to the Kubernetes API and external services via projected tokens. It enables RBAC least-privilege per workload.

---

## Intermediate Questions

### 1. Multi-window, multi-burn-rate SLO alerting

Implement recording rules that compute error budget burn rate over short (5m, 1h) and long (6h, 24h, 3d) windows. Alert when burn exceeds thresholds like 14.4× (page), 6× (ticket), and 1× sustained (email). Short windows catch sudden incidents; long windows catch slow leaks that brief spikes miss.

### 2. On-call escalation policy balancing response and well-being

Define tiered escalation with time-based triggers (primary 15 min → secondary → manager). Cap consecutive on-call weeks, offer compensation time, and measure alert quality to reduce pages. Use follow-the-sun for global teams and require handoff documentation between shifts.

### 3. Blameless postmortem identifying systemic issues

Use structured templates focusing on timeline, contributing factors, and detection/response gaps—not individual blame. Require SMART action items with owners tracked in a central system. Leadership models blameless language and celebrates learning over punishment.

### 4. Reliability review process for new features

Establish a Production Readiness Review checklist covering SLOs, monitoring, runbooks, capacity, rollback, and security. Gate production deploys on completed PRR with tier-appropriate rigor. Provide a formal waiver path with explicit risk acceptance and expiry.

### 5. Error budget policies adjusting deployment velocity

Automate CI/CD gates: high remaining budget allows normal deploys; low budget requires additional approval or freezes non-critical changes. Display budget status in deploy tooling and review burn weekly with product stakeholders.

### 6. Chaos engineering for Kubernetes microservices

Start with pod deletion and network latency experiments in staging, then controlled production blast radius (single AZ). Use tools like Chaos Mesh or Litmus with abort conditions, SLO monitoring, and game-day facilitation. Document hypotheses and measure recovery metrics after each experiment.

### 7. Progressive delivery for high-risk database migration

Apply expand-contract schema changes with backward-compatible application code behind feature flags. Canary traffic to instances using new schema; monitor errors and data consistency. Keep rollback scripts and verified backups even when schema changes are hard to reverse.

### 8. DR plan with 1-hour RPO and 4-hour RTO

Use continuous replication plus frequent snapshots; automate infrastructure provisioning in DR region with IaC. Define runbooks with timed phases and quarterly game days measuring actual RPO/RTO. Alert on replication lag approaching RPO limits.

### 9. Automated rollback triggers in CI/CD

Integrate Prometheus/Datadog queries comparing canary error rate and latency to baseline within 5–15 minutes post-deploy. Tools like Argo Rollouts or Flagger auto-rollback on SLO regression. Require minimum traffic volume to avoid false rollbacks on noise.

### 10. Capacity planning for 10× traffic growth in 12 months

Model growth drivers (users, requests, data size); load test at 2×, 5×, 10× in staging. Identify bottlenecks (DB, cache, network, quotas) quarterly. Plan infrastructure procurement and architecture changes (sharding, CDN) ahead of curves, not at the cliff.

### 11. Kubernetes PDBs for rolling updates without SLO breach

Set `minAvailable` or `maxUnavailable` based on peak traffic capacity math. Test node drains in staging with production-like load. Temporarily scale up before maintenance to absorb disrupted capacity.

### 12. Multi-region active-active for latency-sensitive apps

Deploy in multiple regions with geo-routed traffic via global load balancer. Use data replication with conflict strategy appropriate to consistency needs. Health-check per region and automate failover; measure cross-region latency impact on SLO.

### 13. Service dependency map for incident blast radius

Generate dependency graphs from distributed traces, service catalog, and network flow data. Integrate with incident tooling so alerts show upstream/downstream impact. Update quarterly as part of architecture reviews.

### 14. Toil reduction program

Measure toil as % of on-call/engineering time via quarterly surveys and ticket tagging. Prioritize top toil sources for automation with ROI estimates. Track toil percentage trend as a team KPI; celebrate elimination of manual runbook steps.

### 15. Production readiness review (PRR) for new services

Standard checklist: SLO/SLI, alerting, dashboards, runbooks, capacity plan, security scan, backup/DR, on-call rotation assigned. Block production ingress until PRR approved; tier-1 services require architecture review attendance.

### 16. Kubernetes cluster autoscaling balancing cost and performance

Combine HPA (pod level) with Cluster Autoscaler (node level) and optionally Karpenter for rapid provisioning. Set scale-down delays to prevent flapping; use spot instances for fault-tolerant workloads with on-demand baseline.

### 17. Database connection pooling for 500 service instances

Deploy PgBouncer or RDS Proxy between applications and PostgreSQL. Size pooler connections to DB limits; configure per-service pool sizes in app. Monitor wait time and connection saturation; offload reads to replicas.

### 18. Distributed global rate limiting

Use Redis-backed token bucket or leaky bucket at API gateway with consistent hashing for scale. Return standard rate-limit headers; define fail-open/closed policy. Load test the limiter as a critical dependency.

### 19. Kubernetes node drain with zero downtime

`kubectl cordon` then `kubectl drain` with respect for PDBs; ensure sufficient cluster capacity before drain. Use PodDisruptionBudgets and surge capacity during node maintenance windows.

### 20. CI/CD with automated reliability testing

Add integration tests, contract tests, load tests (subset), and security scans in pipeline. Post-deploy canary analysis on SLO metrics before full promotion. Block deploy on error budget exhaustion policy.

### 21. Service mesh (Istio) for traffic management

Deploy Istio with gradual sidecar injection; configure VirtualService for canary weights and DestinationRule for circuit breakers and outlier detection. Monitor mesh metrics; tune timeouts/retries to prevent retry storms.

### 22. Kubernetes cluster upgrade with zero downtime

Upgrade control plane first, then node pools one at a time with drain and PDB respect. Use surge nodes in managed Kubernetes; validate API compatibility and deprecations before worker upgrade.

### 23. Secrets management for Kubernetes

Use External Secrets Operator or Vault with short-lived dynamic credentials. Enforce RBAC on Secret access; enable encryption at rest; rotate regularly. Never commit secrets to Git; scan repos for leaks.

### 24. Multi-cloud failover within 5 minutes

Use global DNS/LB with health checks across clouds; replicate state asynchronously with known RPO. Automate failover runbook; pre-warm standby capacity. 5-minute RTO requires warm standby, not cold—test frequently.

### 25. Kubernetes resource quotas for 50 teams

Define Namespace per team with ResourceQuota and LimitRange. Monitor usage dashboards; adjust quotas based on demonstrated need. Prevent noisy neighbor with PriorityClasses and optional isolation nodes for large tenants.

### 26. Production debugging for intermittent failures

Enable continuous profiling, dynamic trace sampling, and request correlation IDs. Compare prod vs. staging config and traffic; use feature flags for targeted debug logging. Capture TCP/service mesh taps for short windows during incidents.

### 27. Network policy for zero-trust microservices

Default-deny all ingress/egress; allowlist only required service-to-service paths on specific ports. Automate policy generation from service catalog or network flow observation tools.

### 28. Incident severity classification reflecting user impact

Define matrix by % users affected, data sensitivity, revenue, and regulatory impact. Provide examples and tabletop training. Allow IC to upgrade severity; auto-suggest from alert metadata.

### 29. Deployment pipeline validating infrastructure changes

Run `terraform plan`, OPA/Conftest policy checks, and integration tests in ephemeral environments. Require peer review for IaC PRs; apply canary to infra changes where possible (e.g., one node pool first).

### 30. Kubernetes storage strategy for stateful apps

Use StorageClasses for SSD vs. HDD tiers; StatefulSets with volumeClaimTemplates for stable storage. Test snapshot/restore procedures; monitor volume latency and capacity; implement expansion policies.

### 31. Service catalog for self-service infrastructure

Build a Backstage (or similar) catalog with templates for golden-path service creation including monitoring, CI/CD, and RBAC baked in. Track ownership, tier, dependencies, and SLOs as metadata.

### 32. Log aggregation for 1,000-node cluster

Deploy DaemonSet log shippers (Fluent Bit) with node-level parsing and forwarding to centralized store (Loki/Elasticsearch). Enforce structured JSON logging, retention tiers, and per-namespace quotas to control cost.

### 33. Kubernetes admission controller for policy enforcement

Deploy OPA Gatekeeper or Kyverno to reject non-compliant resources (missing limits, latest tags, privileged containers). Policies as code in Git with CI validation before cluster apply.

### 34. Distributed tracing for 200 services

Standardize OpenTelemetry SDKs with automatic context propagation. Use tail-based sampling for errors; store in Tempo/Jaeger with retention policies. Service graph and SLO dashboards from trace metrics.

### 35. Kubernetes cluster federation for multi-region

Use multi-cluster ingress (GKE Fleet, Admiralty) or GitOps (Argo CD ApplicationSets) for consistent deploys across regions. Federation v2 is limited—often prefer explicit multi-cluster GitOps over legacy federation.

### 36. Backup and restore for Kubernetes stateful apps

Use Velero for cluster resources and volume snapshots; application-consistent backups via hooks. Test restore quarterly to isolated namespace; document RPO/RTO per application tier.

### 37. Pod Security Admission for multi-tenant cluster

Enforce `restricted` profile for tenant namespaces; `baseline` for system components. Audit violations before enforce mode; integrate with admission webhook for custom policies.

### 38. Service mesh observability for mTLS, retries, circuit breakers

Dashboard per-service golden signals from mesh telemetry; alert on elevated 5xx between pairs. Log mTLS handshake failures separately from application errors. Track retry budgets and outlier ejection events.

### 39. Kubernetes CRD for application operational concerns

Define CRDs (e.g., `ApplicationSLO`, `Runbook`) with operators reconciling desired state—auto-create monitors, alerts, and dashboards. GitOps-manage CR instances per service.

### 40. Cost optimization for Kubernetes on cloud

Right-size requests/limits using VPA recommendations; use spot/preemptible for fault-tolerant workloads; cluster autoscale aggressively down off-peak. Remove orphaned volumes and unused load balancers; chargeback by namespace.

---

## Advanced Questions

### 1. SRE organization for 500-engineer company, 200 microservices

- **Team topologies:** Platform SRE (shared tooling, SLO platform), embedded SREs in product verticals, and a central incident/command function.
- **SLO ownership:** Service teams own SLOs; platform provides measurement infrastructure and governance templates.
- **Escalation:** Tiered on-call per service tier → domain SRE → platform → executive for P1 portfolio impact.
- **Conway alignment:** Map reliability ownership to user journeys, not arbitrary infra silos.
- **Capacity:** ~1 SRE per 8–12 engineers for high-criticality domains; lighter ratio for mature automated services.
- **Communities of practice:** Weekly SRE forum for standards; don’t centralize all operational work.

### 2. Reliability platform for 500 services’ SLO compliance

- **SLO-as-code:** OpenSLO/Sloth definitions in Git; CI validates syntax and generates recording rules + dashboards.
- **Auto-discovery:** Register services via catalog metadata; default SLI templates by workload type.
- **Central dashboard:** Executive and team views with burn rates, budget remaining, and deploy freeze status.
- **Policy engine:** Tie CI/CD deploy gates to budget; automated reports to quarterly business reviews.
- **Self-service:** Teams onboard via PR to SLO repo; platform reviews only tier-1 services.

### 3. Global incident management across 10 time zones

- **Follow-the-sun primary IC** with 24/7 coverage; global incident commander for P1 portfolio events.
- **Automated war rooms:** PagerDuty + Slack/Teams auto-create channel, Zoom bridge, and status page incident.
- **Runbook integration:** Incident tooling pulls service tier, owners, dependencies, and recent deploys automatically.
- **Escalation timers:** Auto-escalate if no IC acknowledgment in 5 minutes; executive notification at 30 minutes for P1.
- **Post-incident:** Global postmortem within 72 hours with translation for regional stakeholders.

### 4. Enterprise chaos engineering program

- **Governance board:** Approve experiment catalog, blast radius limits, and production experiment windows.
- **Maturity model:** Start with Game Days → scheduled experiments → continuous chaos in non-prod → limited prod.
- **Safety:** Abort switches, SLO monitoring, blast radius caps (one AZ, 1% traffic), mandatory rollback drills.
- **Metrics:** Track experiments run, weaknesses found, MTTR improvement, and action item closure rate.
- **Buy-in:** Tie to error budget and DR validation; share success stories; never surprise teams without opt-in.

### 5. Multi-region active-active for financial services (strict consistency, sub-100ms)

- **Data strategy:** Synchronous replication within region; careful cross-region design—often strong consistency in-region with geo-partitioned data to avoid cross-region sync on hot path.
- **Latency:** Serve reads/writes from local region partition; use consensus (etcd/Raft) only where latency budget allows.
- **Conflict avoidance:** Partition data by account/tenant to single primary region; cross-region only for DR failover, not active writes to same record.
- **Failover:** Automated health-based traffic shift with documented RPO; regulatory approval for data location.
- **Testing:** Continuous verification of consistency invariants and chaos-based region isolation.

### 6. Zero-downtime PostgreSQL migration (10TB, 10k connections)

- **Logical replication** or `pglogical` to new cluster; dual-write period with application compatibility layer.
- **Connection migration:** PgBouncer/RDS Proxy switch backends atomically; reduce app connections before cutover.
- **Phased cutover:** Read replica promotion, brief read-only window if unavoidable, or blue-green with proxy routing.
- **Validate:** Checksum row counts, replication lag near zero, performance test at 10k connections on new hardware.
- **Rollback plan:** Keep old cluster read-only for 72 hours; DNS/proxy revert path tested.

### 7. Kubernetes multi-cluster platform (100 clusters, 5 clouds)

- **GitOps:** Argo CD/Flux ApplicationSets for consistent deploys; cluster API registered centrally.
- **Policy:** OPA/Gatekeeper policies pushed from central repo; cluster conformance scanning (Kubescape, Polaris).
- **Observability:** Thanos/Mimir federation; centralized Grafana with cluster label.
- **Identity:** Unified OIDC; short-lived tokens; cluster access via privileged access management.
- **Upgrade orchestration:** Wave upgrades by environment; canary clusters per cloud provider.

### 8. Production debugging platform without availability impact

- **Continuous profiling** at <1% overhead; always-on tracing with head sampling + tail sampling for errors.
- **Request mirroring** to shadow environment for safe reproduction.
- **Dynamic instrumentation:** eBPF-based tools (Cilium Hubble, bpftrace) and OTel collector processors.
- **Access control:** Just-in-time production shell access with session recording.
- **Guardrails:** Rate limits on debug endpoints; no heap dumps on all pods simultaneously.

### 9. Reliability-as-code in CI/CD

- **Policy definitions:** Rego/OPA or Sentinel checks for SLO presence, resource limits, PDB, probes in manifests.
- **Contract tests:** Pact or schema validation between services in pipeline.
- **Load tests:** Automated k6/Locust gates on PR for tier-1 services.
- **SLO unit tests:** Validate recording rules and alert thresholds in CI with promtool.
- **Scorecard:** PR comment with reliability score; block merge below threshold for critical services.

### 10. ML-based capacity planning platform

- **Data pipeline:** Ingest historical metrics, deploy events, and business forecasts.
- **Models:** Time-series forecasting (Prophet, seasonal ARIMA) per service with confidence intervals.
- **Automation:** Generate scale recommendations; optional auto-provision with human approval gate.
- **Feedback loop:** Compare predictions vs. actual; retrain monthly.
- **Guardrails:** Never auto-scale production without limits; alert on anomalous predictions.

### 11. DR automation platform

- **Runbook as code:** Ansible/Terraform workflows with idempotent steps and checkpointing.
- **Orchestrator:** Step Functions or Argo Workflows with human approval gates at critical steps.
- **Pre-flight checks:** Validate backups, replication lag, and DR site health before failover trigger.
- **Observability:** Real-time RTO clock and step status dashboard during execution.
- **Testing:** Scheduled dry-runs in isolated DR environment without customer traffic impact.

### 12. Service mesh at scale (10,000 pods)

- **Sidecar resource tuning:** Reduce CPU/memory; consider ambient mesh (Istio) or eBPF dataplane (Cilium) where appropriate.
- **Control plane HA:** Multiple istiod replicas; isolate config propagation failures.
- **Incremental rollout:** Namespace-by-namespace injection; monitor p99 overhead.
- **Observability:** Sample mesh telemetry; aggregate at collection to reduce cardinality.
- **mTLS:** Gradual enablement with permissive mode before strict.

### 13. Progressive delivery platform (canary, A/B, feature flags, SLO rollback)

- **Unified control plane:** Argo Rollouts + feature flag service (LaunchDarkly/Unleash) with SLO integration.
- **Automated analysis:** Prometheus queries for error rate, latency, business KPIs during canary.
- **A/B routing:** Header-based or user cohort routing with statistical significance tracking.
- **Auto-rollback:** On SLO burn rate spike during rollout; notify owners.
- **Audit:** All flag changes and traffic splits logged for compliance.

### 14. Cost optimization platform maintaining reliability

- **Rightsizing recommendations** from actual usage with reliability buffer (p99 + headroom).
- **Spot orchestration** for fault-tolerant workloads with PDB protection.
- **Waste detection:** Orphan resources, over-provisioned volumes, idle load balancers.
- **Policy:** Never recommend cuts that breach SLO or remove critical redundancy without explicit approval.
- **Chargeback dashboards** drive team behavior without central micromanagement.

### 15. Security reliability engineering (SRE + security)

- **Include security SLIs:** Certificate expiry, auth failure rate, vulnerability patch latency in reliability dashboards.
- **Incident integration:** Security incidents use same severity framework and IC process as availability incidents.
- **SLO for security controls:** e.g., "99.9% of API requests pass authz check within 50ms."
- **Chaos:** Include credential rotation failure and KMS unavailability in game days.
- **Joint postmortems** for incidents with both availability and security root causes.

### 16. Internal developer platform (IDP) with reliability guardrails

- **Golden paths:** Backstage templates scaffold service with observability, CI/CD, PDB, and SLO defaults.
- **Policy at create time:** Admission controllers enforce limits, network policy, and scan results.
- **Self-service within guardrails:** Teams choose instance sizes from approved menu; custom infra requires exception.
- **Feedback:** Developer satisfaction surveys; time-to-production metrics.
- **Platform SLO:** Measure platform availability separately from application SLOs.

### 17. Reliability testing from dependency graphs

- **Auto-generate tests:** Traverse service catalog DAG; create fault injection tests per edge (timeout, 500, latency).
- **Contract validation:** Verify consumer expectations against provider behavior under failure.
- **CI integration:** Run subset on every PR; full graph weekly in staging.
- **Prioritize by centrality:** Test highest fan-out dependencies first (highest blast radius).
- **Report:** Reliability scorecard per service based on test pass rate and coverage.

### 18. Kubernetes operator for complex stateful application

- **Reconcile loop:** Manage deploy, scale, backup, restore, and upgrade via CRD spec.
- **Ordered operations:** Respect startup/shutdown order for clustered stateful apps.
- **Backup sidecar:** Scheduled volume snapshots with retention policy in CRD.
- **Upgrade strategy:** Rolling partition-aware updates with pre-upgrade health checks.
- **Failure recovery:** Auto-replace failed members; alert if quorum at risk.

### 19. Global load balancing with 50 countries and data residency

- **Geo-DNS** routing users to in-region endpoints meeting residency (EU data stays in EU).
- **Health-aware routing** removes unhealthy regions; respect data sovereignty boundaries in routing policy.
- **Latency optimization** within residency constraints—may sacrifice pure lowest latency for compliance.
- **Multi-region observability** with per-country SLO views.
- **Failover** only to compliant regions; document legal review per routing change.

### 20. Production readiness framework (automated evaluation)

- **Scorecard API:** Checks Git repo for SLO file, CI config, probes, PDB, dashboards, on-call entry, security scan pass.
- **CI gate:** Deploy to production namespace blocked until score ≥ threshold for service tier.
- **Waivers:** Time-limited exceptions with approver audit trail.
- **Continuous re-evaluation:** Drift detection if standards regress after initial approval.
- **Dashboard for leadership:** % services meeting readiness by tier and team.

### 21. Postmortem-driven reliability improvement program

- **Central action item tracker** with SLA (30/60/90 days) and escalation for overdue items.
- **Metrics:** Repeat incident rate, action item completion %, error budget trend post-investment.
- **Quarterly reliability review:** Prioritize investments from postmortem themes across org.
- **Close the loop:** Verify fixes in production (monitoring, chaos re-test) before marking complete.
- **Celebrate impact:** Share reliability wins tied to completed postmortem actions.

### 22. Cluster autoscaling 100 to 10,000 nodes in 10 minutes

- **Pre-provisioned node pools** or Karpenter with rapid cloud instance allocation.
- **Quota management:** Pre-approve cloud quotas; multi-AZ spread; avoid API rate limits with batching.
- **HPA ahead of demand:** Predictive scaling from queue depth or schedule before traffic arrives.
- **Control plane sizing:** Ensure API server and etcd can handle 10k nodes (sharding, event rate limits).
- **Test at scale:** Regular load tests proving 10× scale-out within target time.

### 23. Distributed systems debugging with causal analysis

- **Causal tracing:** Critical path analysis from traces; highlight dominant latency and error contributors.
- **Event correlation:** Deploy, config change, flag flip, and infra events on unified timeline.
- **Topology-aware UI:** Service graph with live health overlay during incidents.
- **ML-assisted grouping:** Cluster similar failure signatures across services.
- **Runbook integration:** Suggest likely causes based on symptom pattern library.

### 24. Reliability SLA management for B2B SaaS tiers

- **Per-tenant SLO measurement** where contracts require it (synthetic + real traffic).
- **SLA tier definitions:** Gold/Silver/Bronze with different targets and credit policies.
- **Reporting portal:** Customer-facing uptime reports with maintenance exclusions documented.
- **Error budget per tier:** Internal teams prioritize fixes affecting highest-tier customers first.
- **Contract alignment:** Ensure marketing SLAs are looser than internal SLOs (SLO buffer).

### 25. GitOps infrastructure management platform

- **All infra in Git:** Terraform/Kustomize/Helm with PR review, plan in CI, and signed commits.
- **Drift detection:** Regular compare live vs. desired state; alert on manual changes.
- **Progressive rollout:** Apply to canary cluster/namespace before production-wide.
- **Audit trail:** Full history of who changed what; integrate with SOC2/compliance requirements.
- **Rollback:** `git revert` triggers automated rollback via Argo CD; test rollback quarterly.

---

## Rapid-Fire Questions

### 1. What does SRE stand for?

Site Reliability Engineering.

### 2. What is the formula for calculating error budget?

Error budget = 100% − SLO target (as a percentage), applied over the measurement window—e.g., 99.9% SLO → 0.1% budget (failed requests or allowed downtime equivalent).

### 3. What is the difference between MTTR and MTTD?

MTTD (Mean Time To Detect) is how long until you notice a problem; MTTR (Mean Time To Resolve/Recover) is how long until service is restored after detection.

### 4. What does SLI stand for?

Service Level Indicator.

### 5. What is the purpose of a postmortem action item?

To implement a specific, owned change that reduces the likelihood or impact of similar future incidents.

### 6. What is toil in SRE?

Manual, repetitive, automatable operational work that scales with service growth and doesn't provide lasting value.

### 7. What does RTO stand for in disaster recovery?

Recovery Time Objective—the maximum acceptable time to restore service after an outage.

### 8. What is the purpose of a circuit breaker pattern?

To stop calling a failing dependency, fail fast, and allow recovery—preventing cascading failures.

### 9. What does RPO stand for in disaster recovery?

Recovery Point Objective—the maximum acceptable amount of data loss measured in time.

### 10. What is the difference between availability and reliability?

Availability is whether the system is up and reachable; reliability includes correct operation and durability over time, not just uptime.

### 11. What is a canary deployment?

Deploying a new version to a small subset of users/traffic first to validate before full rollout.

### 12. What does MTTF stand for?

Mean Time To Failure.

### 13. What is the purpose of a feature flag?

To enable or disable functionality at runtime without redeploying, supporting gradual rollout and quick rollback.

### 14. What is a blue-green deployment?

Maintaining two identical environments and switching traffic from old to new atomically for safe deploys and fast rollback.

### 15. What does P99 latency mean?

99% of requests complete faster than this value; 1% are slower—captures tail latency experienced by unlucky users.

### 16. What is the purpose of a health check endpoint?

To report service health so load balancers and orchestrators route traffic only to healthy instances.

### 17. What is a rolling deployment?

Gradually replacing old instances with new ones while keeping the service available throughout the rollout.

### 18. What does CRD stand for in Kubernetes?

Custom Resource Definition.

### 19. What is the purpose of a Kubernetes liveness probe?

To detect hung or dead containers and restart them when unhealthy.

### 20. What is a Kubernetes readiness probe?

To determine if a pod is ready to receive traffic; failed readiness removes it from Service endpoints without restart.

### 21. What does HPA stand for in Kubernetes?

Horizontal Pod Autoscaler.

### 22. What is the purpose of a Kubernetes PodDisruptionBudget?

To limit voluntary disruptions (drains, evictions) so a minimum number of pods stay available during maintenance.

### 23. What is a Kubernetes DaemonSet used for?

Running one pod per node for node-level agents like log collectors or monitoring exporters.

### 24. What does PVC stand for in Kubernetes?

PersistentVolumeClaim.

### 25. What is the purpose of a Kubernetes NetworkPolicy?

To control pod network traffic with label-based firewall rules for segmentation and zero-trust.

### 26. What is a Kubernetes StatefulSet used for?

Managing stateful apps needing stable identity, ordered deployment, and persistent storage per pod.

### 27. What does RBAC stand for in Kubernetes?

Role-Based Access Control.

### 28. What is the purpose of a Kubernetes ServiceAccount?

Providing an identity for pods to authenticate to the Kubernetes API and external services.

### 29. What is a Kubernetes Ingress resource?

An API object managing external HTTP/HTTPS access to services with routing rules and TLS.

### 30. What does CNI stand for in Kubernetes?

Container Network Interface.

### 31. What is the purpose of a Kubernetes ConfigMap?

Storing non-sensitive configuration data separately from container images for easier updates.

### 32. What is a Kubernetes Secret used for?

Storing sensitive data like passwords and API keys (with encryption at rest recommended).

### 33. What does OOM stand for in Kubernetes?

Out Of Memory—typically when a container exceeds its memory limit and is killed (OOMKilled).

### 34. What is the purpose of Kubernetes resource requests?

Guaranteed minimum CPU/memory used by the scheduler to place pods on nodes with sufficient capacity.

### 35. What is a Kubernetes resource limit?

The maximum CPU/memory a container can use before being throttled or OOMKilled.

### 36. What does VPA stand for in Kubernetes?

Vertical Pod Autoscaler.

### 37. What is the purpose of a Kubernetes node taint?

To repel pods unless they have a matching toleration—used for dedicated nodes or maintenance.

### 38. What is a Kubernetes node affinity?

Rules attracting or repelling pods to specific nodes based on node labels.

### 39. What does CSI stand for in Kubernetes?

Container Storage Interface.

### 40. What is the purpose of a Kubernetes admission webhook?

To intercept API requests and enforce validation or mutation policies before resources are persisted.

### 41. What is a Kubernetes operator?

A controller extending Kubernetes with custom resources to automate complex application lifecycle management.

### 42. What does CRD stand for?

Custom Resource Definition.

### 43. What is the purpose of a Kubernetes namespace?

Logical cluster subdivision for isolation, RBAC, quotas, and multi-tenancy.

### 44. What is a Kubernetes pod priority class?

A configuration defining pod scheduling priority and preemption behavior under resource pressure.

### 45. What does QoS stand for in Kubernetes pod scheduling?

Quality of Service—classes (Guaranteed, Burstable, BestEffort) based on requests/limits affecting eviction priority.

### 46. What is the purpose of a Kubernetes horizontal pod autoscaler?

Automatically scaling pod replica count based on metrics like CPU, memory, or custom application metrics.

### 47. What is a Kubernetes cluster autoscaler?

Automatically adding or removing nodes based on pending pod resource requirements and utilization.

### 48. What does ETCD store in a Kubernetes cluster?

All cluster state: configuration, object metadata, and desired state for the Kubernetes API.

### 49. What is the purpose of a Kubernetes API server?

The front-end for the Kubernetes control plane validating and persisting all API requests to etcd.

### 50. What is a Kubernetes controller manager responsible for?

Running controllers that reconcile desired state (Deployments, ReplicaSets, endpoints) with actual cluster state.
