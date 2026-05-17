# Grafana — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## Dashboard Design & Troubleshooting

### 1. Dashboard showing "No data" while Prometheus is healthy

- **Reproduce in Explore:** Run the same PromQL in Explore with identical time range, datasource, and variables to isolate dashboard vs. backend issues.
- **Check datasource config:** Verify URL, auth, TLS, `httpMethod`, and that the dashboard references the correct datasource UID (not a broken or renamed instance).
- **Inspect variables:** Empty or mismatched `$namespace`, `$cluster`, or regex variables often produce queries that match zero series.
- **Time range and resolution:** Confirm `$__range`, `$__interval`, and min step aren't causing empty results; try "Last 5 minutes" and disable query caching temporarily.
- **Panel query errors:** Open Query Inspector per panel for HTTP errors, timeouts, or "parse error" from templated queries.
- **Permissions and tenancy:** Ensure the Grafana user/role can access the datasource and that Mimir/Prometheus tenant headers are correct.
- **Recent changes:** Diff dashboard JSON, provisioning, and datasource UIDs from Git; check for renamed metrics or recording rules.
- **Grafana health:** Review Grafana logs for proxy errors, plugin failures, and datasource health-check failures.

### 2. Payment processing dashboard across 50 microservices

- **Hierarchy:** Top row = golden signals (rate, errors, duration) aggregated by tier; middle = service groups; bottom = drill-down per service via variables.
- **Variables:** Use `service`, `environment`, and `region` template variables with chained queries; default to top-N by traffic to limit cardinality.
- **Recording rules:** Pre-aggregate high-cardinality metrics in Prometheus/Mimir so dashboards query `service:http_requests:rate5m` not raw counters.
- **Panel budget:** Cap at ~15–20 panels per view; use rows, collapsed sections, and linked dashboards instead of one mega-board.
- **Refresh strategy:** 30–60s refresh for overview; slower refresh or manual for deep-dive views to reduce backend load.
- **Performance:** Reuse library panels, combine queries where possible, set appropriate `min interval`, and avoid `rate()` over huge ranges without recording rules.
- **Usability:** Consistent units, RED layout, SLO/error-budget row, and clear annotations for deploys.

### 3. 40-panel dashboard taking 45 seconds to load

- **Query Inspector:** Identify slowest panels by request duration, bytes returned, and number of queries per refresh.
- **Parallel vs. serial:** Note whether Grafana is waiting on many sequential datasource round-trips; reduce panel count per row refresh.
- **Optimize queries:** Replace heavy `histogram_quantile` over raw buckets with recording rules; narrow label matchers and time range defaults.
- **Recording rules and aggregation:** Move expensive computations to Prometheus; use `sum by (service)` instead of per-pod where overview suffices.
- **Panel consolidation:** Merge related metrics into one panel with multiple queries; use stat/row repeat sparingly.
- **Caching:** Enable query caching (Enterprise) or reduce concurrent users via playlist/NOC-specific views.
- **Lazy loading:** Split into linked dashboards; load detail only on drill-down.
- **Backend scale:** If many panels are individually "fast" but total is slow, scale query-frontend/Mimir or add read replicas.

### 4. Grafana query caching for Prometheus under 200 concurrent users

- **How it works:** Grafana Enterprise query cache stores datasource responses keyed by query + time range + variables, serving repeated identical requests from cache.
- **Configure:** Enable in `grafana.ini` / Helm values; set TTL aligned to scrape interval (e.g., 30–60s for real-time, longer for historical).
- **Scope:** Cache per datasource; use `cache timeout` overrides on heavy dashboards where slightly stale data is acceptable.
- **Reduce unique queries:** Standardize dashboards, variables, and time ranges so more users hit the same cache keys.
- **Prometheus side:** Increase `query.max-concurrency`, use query-frontend with Mimir/Cortex, and enforce `max_samples` limits.
- **Operational guardrails:** Monitor cache hit rate, backend QPS drop, and alert on cache stampede after TTL expiry.

### 5. Same metrics for dev/staging/prod without duplicating dashboards

- **Environment variable:** Create a custom or query variable `environment` with values `dev|staging|prod`.
- **Datasource variable (optional):** Use one Prometheus with `env` label, or a datasource variable mapping env → different Prometheus UIDs.
- **Templating in queries:** `sum(rate(http_requests_total{env="$environment", service="$service"}[5m]))`.
- **Annotations and links:** Parameterize dashboard links with `?var-environment=prod` for sharing.
- **Provisioning:** Single JSON dashboard in Git; no triplicate copies.
- **RBAC:** Folder permissions per team; all envs visible only to roles that should see them.

### 6. Panel data delayed 10 minutes vs. Prometheus

- **Range variable misuse:** `$__range` in query range selectors can widen windows; verify panel uses `$__rate_interval` for `rate()` not a fixed 10m offset.
- **Query offset:** Check for intentional `offset 10m` in query or recording rule.
- **Recording rule delay:** Stale recording rules or rule evaluation lag can delay aggregated series.
- **Min step / resolution:** Very large `$__interval` can cause apparent smoothing/delay in visualization.
- **Datasource proxy cache:** Enterprise cache TTL or CDN in front of Grafana serving stale responses.
- **Clock skew:** Compare Prometheus `time()` with Grafana server time.
- **Fix:** Remove erroneous offset, align scrape/eval intervals, lower cache TTL, use instant queries for "now" panels.

### 7. Fleet overview for 500 Kubernetes nodes with drill-down

- **Cluster → node variables:** `cluster` query variable; `node` filtered by `label_values(kube_node_info{cluster="$cluster"}, node)`.
- **Overview panels:** Aggregated CPU/memory/disk by cluster; top-N unhealthy nodes; use heatmap or bar gauge for distribution.
- **Repeat rows/panels:** Repeat row per selected node only when user picks one (avoid repeating 500 times by default).
- **Linked dashboards:** Fleet overview links to node detail dashboard with `var-node` passed in URL.
- **Recording rules:** Pre-aggregate node metrics to keep variable queries fast.
- **Performance:** Default to cluster-level view; load node detail on demand.

### 8. Dashboard version control and change management

- **Git as source of truth:** Store dashboards as JSON/Jsonnet in Git; provision via CI/CD or Grafana Operator.
- **PR review:** Require peer review for shared folders; use CODEOWNERS for platform dashboards.
- **Environments:** Promote dev → staging → prod Grafana with automated diff and validation.
- **UID stability:** Pin dashboard UIDs so links and alerts don't break.
- **Audit:** Enable Grafana audit logs (Enterprise) and track who changed what.
- **Library panels:** Shared panels versioned once, referenced by many dashboards.
- **Deprecation:** Tag dashboards `deprecated`; announce migrations in #observability.

### 9. Recording rule renamed — find affected dashboards and migrate

- **Search:** `grep`/ripgrep dashboard JSON in Git for old metric name; use Grafana API `GET /api/search` + export if not in Git.
- **Grafana UI:** Dashboard settings → JSON Model search for old rule name.
- **Alert rules:** Search unified alerting definitions and contact points referencing the metric.
- **Migration:** Bulk replace in Git PR with recording rule alias period (dual-write old+new names).
- **Validation:** Run dashboard linter/CI (Grafonnet tests, `grafanalib` validation) before deploy.
- **Communicate:** Changelog for observability; temporary redirect via recording rule `or` old metric name.

### 10. Real-time incident dashboard highlighting anomalies

- **Alert-driven focus:** Use Grafana OnCall or alert annotations to auto-set time range and variables to firing service.
- **Anomaly panels:** ML plugins or Prometheus `predict_linear` / outlier recording rules for baseline bands.
- **Unified view:** Mix Prometheus (RED), Loki (error logs), Tempo (slow traces) with correlations configured.
- **Dynamic rows:** Repeat panels for `alertname` or `service` labels from active alerts.
- **Thresholds and overrides:** Red highlighting when burn rate or error rate exceeds SLO.
- **Auto-refresh:** 10–30s during incidents; snapshot link for postmortem.
- **Playlists:** NOC playlist of incident + dependency dashboards.

---

## Alerting & Notification

### 11. 500 notifications/hour from flapping

- **Grouping:** Configure notification policies to group by `alertname`, `service`, `cluster` with `group_wait`, `group_interval`, `repeat_interval`.
- **Deduplication:** Use inhibition rules (e.g., node down inhibits pod alerts) and mute timings for known maintenance.
- **For duration:** Increase `for` on noisy rules so brief spikes don't page.
- **Multi-window:** Require sustained breach (e.g., 5m and 1h) for critical pages.
- **Severity routing:** Warning → Slack; critical → PagerDuty only when grouped and confirmed.
- **Alert quality review:** Weekly top-noise alerts; fix thresholds or underlying flapping dependency.
- **Auto-resolve:** Ensure resolved notifications are sent so on-call knows when to stand down.

### 12. Legacy vs. Unified Alerting; migration alert gaps

- **Architecture:** Legacy used backend-specific alerting in dashboards; Unified uses Prometheus-style ruler model with multi-datasource, Grafana-managed state, and notification policies.
- **Gaps on migration:** Dashboard alerts not migrated automatically; different UID/model for notification channels → contact points; evaluation interval and `for` semantics differ.
- **Multi-org:** Unified alerting is org-scoped; missed rules if org context wrong.
- **Datasource permissions:** Unified evaluator needs access; silent failures if SA lacks scope.
- **Mitigation:** Run parallel alerting in staging; export/import alert rules via Terraform; validate firing with test alerts before cutover.

### 13. Multi-dimensional alert firing for wrong series

- **Inspect alert query:** In Grafana, check Reduce/Math expressions and which labels survive `$labels` in notifications.
- **Label matchers:** Add `service=~"$expected"` or explicit matchers in alert condition; avoid overly broad `sum without (pod)`.
- **Debug:** Use "View in Explore" from alert; compare firing instances in Alertmanager UI / Grafana alert history.
- **Fix:** Tighten PromQL with `and on(service) group_left` to required labels; use classic conditions or SQL with explicit dimensions.
- **Test:** Use Grafana's alert testing API to evaluate against historical data.

### 14. Routing: PagerDuty critical, Slack warning, email digest; business hours

- **Contact points:** Separate PagerDuty, Slack, and email contact points with templates per severity.
- **Notification policies:** Tree: default → match `severity=critical` → PagerDuty; `severity=warning` → Slack; `severity=info` → email digest route.
- **Time-based muting:** Mute timings for off-hours on non-critical; or reverse—stricter routing after hours for critical only.
- **OnCall integration:** Grafana OnCall schedules for business-hours vs. 24/7 escalation.
- **Group by:** `team`, `service` labels for correct routing.
- **Digest:** Use email contact point with batching or external tool for daily digests.

### 15. Silences and maintenance windows via API + CI/CD

- **Mute timings API:** `POST /api/alertmanager/grafana/config/api/v1/silences` or Grafana mute timing provisioning.
- **CI/CD:** Pipeline step creates silence for `deploy_id`, `cluster`, duration = deploy window + buffer; delete on success.
- **Labels:** Match `service=myapp`, `alertname=~".+"` to scope silences narrowly.
- **GitOps:** Store mute timing YAML in repo; provision via Terraform `grafana_mute_timing`.
- **Audit:** Log silence creation with pipeline URL; auto-expire to prevent stale silences.

### 16. `for: 5m` but alert fires immediately

- **No data / execution:** Unified alerting may treat missing data differently; check "Keep firing for" and recovery settings.
- **Instant vs. range:** Rule type `instant` with wrong reduce function can fire on single evaluation.
- **Multiple conditions:** OR conditions where one sub-condition has no `for`.
- **Recording rule lag:** Not usually instant—check if alert is on different rule than expected.
- **Grafana bug/version:** Verify version; check if `for` is set on the right query ref (A, B, C).
- **Fix:** Set explicit `for` on alert rule; use `last()` over range; validate in alert preview.

### 17. Escalation when on-call doesn't ack in 15 minutes

- **Grafana OnCall:** Escalation chains—notify secondary at +15m, manager at +30m.
- **PagerDuty:** Use PD escalation policies integrated as contact point; Grafana triggers PD incident.
- **Webhook:** Custom webhook to incident tool with escalation logic if OnCall not available.
- **Repeat interval:** Set `repeat_interval: 15m` in notification policy for critical routes.
- **Ack tracking:** Require ack in OnCall/PD to stop escalation; integrate bi-directionally.

### 18. Contact points vs. notification policies; silent drop scenario

- **Contact points:** Define *how* to notify (Slack webhook, email SMTP, PagerDuty key).
- **Notification policies:** Define *which* alerts go *where* via label matchers and nested routes.
- **Silent drop example:** Child policy matches `severity=critical` but has no contact point and `continue: false`, while parent default was overridden—critical alerts match child, go nowhere.
- **Another case:** `continue: false` on a broad matcher steals alerts before they reach PagerDuty route.
- **Prevention:** Diagram policy tree; test with synthetic alerts; default catch-all route to ops Slack.

### 19. Correlated alerts across Prometheus, Loki, Elasticsearch

- **Grafana unified:** Multi-query alert with `Math` expression: `$A > threshold AND $B > threshold`.
- **Limitations:** All datasources must evaluate within same interval; clock alignment matters for logs.
- **Alternative:** Correlate in Prometheus (exporter from logs) or use Grafana OnCall + composite incident rules.
- **Recording from logs:** `logql` metric queries feeding Prometheus for simpler correlation.
- **Trade-off:** Higher evaluation cost and false negatives if one datasource is delayed.

### 20. Testing alert rules in staging

- **Staging Grafana:** Duplicate rules against staging datasources with same PromQL labels.
- **Amtool/CLI:** `grafana alerting test` or API evaluate endpoint with historical time range.
- **Synthetic traffic:** Load tests or chaos in staging to trigger conditions without prod impact.
- **Recording test metrics:** Push `test_alert_fire=1` via Pushgateway in staging only.
- **Promotion:** Terraform plan/apply with review; canary one rule to prod; compare notification delivery.
- **Mute prod:** Ensure staging contact points go to test Slack channel only.

---

## Loki Integration

### 21. Log query timeouts >24 hours — Loki vs. query vs. Grafana

- **Grafana timeouts:** Check `timeout` in datasource JSON (default 60s–300s); increase cautiously for Explore/dashboards.
- **Query complexity:** High-cardinality `| json` + `|=` filters over 24h+ need `query_range` limits; simplify or narrow labels first.
- **Loki limits:** `max_query_length`, `max_query_parallelism`, `split_queries_by_interval`, chunk retention—review querier/frontend logs.
- **Index period:** Very long ranges span many chunks; use log volume metric queries (`count_over_time`) instead of full line fetch.
- **Diagnose path:** Run same LogQL in `logcli` bypassing Grafana; if slow there, fix Loki; if fast, tune Grafana proxy timeout.
- **Mitigation:** Shorter default dashboard range, recording rules for aggregates, downsampled metrics from logs.

### 22. Correlated metrics ↔ logs via exemplars

- **Prometheus:** Enable exemplar storage on histograms; OTel SDK exports trace_id in exemplars.
- **Loki:** Ingest trace IDs in log labels or structured fields (`trace_id`); align label names with Tempo.
- **Grafana datasource linking:** Prometheus datasource → "Exemplars" → Tempo; Loki derived field regex `trace_id=(\w+)` → Tempo.
- **Service map:** Optional metrics-generator in Tempo linking service graph.
- **Validation:** Click exemplar diamond on heatmap → opens trace; click log line → same trace in Tempo.

### 23. Log volume panel shows fewer entries than expected

- **Sampling:** Check if agents (Promtail/Alloy) or Loki ingestion drops debug logs; `limits_config.discard` rules.
- **Retention:** Logs may be deleted past retention while metrics still exist.
- **Query filters:** `|=` pipeline stages excluding lines; verify label selectors match ingested streams.
- **Metric vs. log query:** Volume uses `count_over_time`; content query may hit `max_entries` limit.
- **Cardinality limits:** `max_streams_per_user` causing rejected streams.
- **Fix:** Compare ingest rate (`loki_distributor_bytes_received_total`) with query results; adjust limits and filters.

### 24. Deployment events (Loki) + metric anomalies (Prometheus) on one timeline

- **Loki:** Ingest deploy logs or use `event` label from CI webhook to Loki (`{job="deployments"} |= "deployed"`).
- **Prometheus:** Track `changes()` or version info metrics; anomaly via recording rules.
- **Annotations:** Provision annotation API from CI with deploy tags; display on both panels.
- **Dashboard:** Shared time range; annotation layer on metric panels; log panel filtered by `deploy_id`.
- **Correlations:** Grafana correlations linking deploy annotation time to log/metric drill-down.

### 25. Explore for live incident — Loki + Tempo + Prometheus

- **Split view:** Prometheus metrics for RED; switch to Loki with same `service` label; pivot to trace via derived field.
- **Trace to logs:** Tempo span → "Logs for this span" with trace_id filter in Loki.
- **Time sync:** Narrow time range around spike; use "Logs volume" to find error burst timestamp.
- **Live tail:** Loki live tailing in Explore for streaming errors during active incident.
- **Bookmarks:** Save Explore URLs with query params for war room handoff.

### 26. Loki "context deadline exceeded" — Grafana-side tuning

- **Increase `jsonData.timeout`** on Loki datasource (e.g., 300–600s for heavy queries—use sparingly).
- **Reduce query scope:** Default shorter time range; max lines limit in query options.
- **HTTP settings:** Adjust `max_conns_per_host`, enable HTTP/2 if supported.
- **Query splitting:** Enable query split and parallelization in datasource options where available.
- **Proxy:** If Grafana behind slow proxy, increase proxy timeouts.
- **Right-size:** Prefer fixing Loki querier scale over unbounded Grafana timeouts.

### 27. Log-based alerting with Loki — limitations

- **Implementation:** Grafana unified alerting with LogQL (`count_over_time` or metric queries from logs).
- **Latency:** Log pipeline ingest + evaluation interval (1m+) slower than metric alerts.
- **Reliability:** Log drops, parsing failures, or sampling cause missed alerts; less deterministic than counters.
- **Cost:** High-cardinality LogQL alerts are expensive at scale.
- **Best practice:** Convert critical log patterns to metrics via recording rules or Loki ruler, alert on metrics.

### 28. Derived fields with Loki — production use case

- **Mechanism:** Regex on log line extracts field (e.g., `trace_id`) and links to Tempo datasource URL template.
- **Use case:** Payment failures log `trace_id=abc123` → one click opens full distributed trace for failed transaction.
- **Also:** Link `request_id` to internal admin UI or Kibana for support tooling.
- **Config:** Datasource settings → Derived fields; test with sample log line.
- **Requirement:** Consistent log format (structured JSON) for reliable regex.

---

## Tempo Integration & Distributed Tracing

### 29. 10-second gap between spans — network vs. processing vs. clock skew

- **Trace view:** Compare span timestamps and durations; gap between child start and parent end indicates queue/wait.
- **Network:** Large gap with small span duration on client/server RPC spans → network or LB delay.
- **Processing:** Long span duration on one side → slow handler, DB, or thread pool exhaustion.
- **Clock skew:** Spans on different hosts with negative gaps or impossible ordering → NTP drift; check `clock_skew` in collector.
- **Instrumentation:** Missing intermediate spans (async) can look like gaps—verify context propagation.
- **Tools:** Compare `client.duration` vs `server.duration`; use service graph edge latency.

### 30. Trace-to-metrics and metrics-to-trace navigation

- **Prometheus:** Exemplars enabled on histograms; `exemplarTraceIdDestinations` in Grafana Prometheus datasource.
- **Tempo:** Metrics generator (optional) produces service graph and span metrics; `traces_to_metrics` config.
- **Grafana:** Link Prometheus → Tempo and Tempo → Prometheus in datasource settings; enable "Trace to metrics" feature toggle.
- **Requirements:** Consistent `service.name`, `resource` attributes; trace IDs in exemplars and metric labels.
- **RED from traces:** Use TraceQL metrics or Tempo metrics-generator recording rules.

### 31. RED dashboard from Tempo TraceQL metrics

- **TraceQL metrics:** `rate()` over `{ } | rate()`  or dedicated metrics from metrics-generator (`traces_spanmetrics_*`).
- **Dashboard:** Per-service variables; panels for request rate, error rate (`status_code=ERROR`), duration histograms.
- **Datasource:** Tempo datasource with metrics query type or Prometheus fed by generator.
- **Recording:** Promote to Prometheus/Mimir for long retention and alerting.
- **Alignment:** Same `service` labels as k8s/Prometheus for correlation.

### 32. Incomplete traces in Tempo — sampling vs. ingestion vs. query

- **Sampling:** Head/tail sampling drops spans; check collector `probabilistic_sampler` and tail sampling policies.
- **Ingestion:** Dropped spans from `rate_limit`, message too large, or attribute limits—check distributor logs.
- **Query:** `max_bytes` trace limit truncates large traces in UI; increase query frontend limits.
- **Propagation:** Broken context loses child spans—appears as incomplete, not sampled.
- **Diagnose:** Compare ingest span count vs. query; test with `debug` ID full sampling; verify block storage integrity.

### 33. Service Graph panel for cascading failures

- **Setup:** Enable Tempo metrics-generator service graph; Grafana Service Graph panel with Tempo datasource.
- **Incident use:** Identify upstream node turning red; follow edges with rising latency/error rate.
- **Correlate:** Click node → node graph / traces / RED metrics for that service.
- **Filters:** Narrow time range to incident window; filter by `namespace` or `cluster`.
- **Limitation:** Needs sufficient trace volume; sampling may hide rare paths.

---

## Mimir Integration

### 34. Data gaps at Prometheus local storage ↔ Mimir boundary

- **Cause:** Remote write lag, different retention, or query merging two sources with gap/overlap misalignment.
- **Diagnose:** Query each backend separately for same metric/time; check `prometheus_remote_storage_*` metrics.
- **HA:** Verify remote write queue not backlogged; no dropped samples.
- **Query:** Mixed datasource or `prometheus` + `mimir` with different `min time`—unify via single query path to Mimir with `-querier.store-gateway` for all blocks.
- **Fix:** Backfill missing range; align `external_labels`; use single long-term store query after migration complete.

### 35. Multi-tenant Mimir — teams see only their metrics

- **Mimir:** Enable multi-tenancy; per-team `X-Scope-OrgID` tenant ID.
- **Grafana:** Separate datasource per tenant with custom HTTP header `X-Scope-OrgID: team-a` in JSON secure fields.
- **RBAC:** Datasource permissions limit which roles see which tenant datasource.
- **Automation:** Provision datasources via Terraform per team; never share admin datasource across tenants.

### 36. Mimir slower than Prometheus direct — why and optimize

- **Reasons:** Object storage block fetch, query-frontend queueing, replication, larger historical scans, compactor not caught up.
- **Optimize:** Query-frontend cache, memcached for chunks, `parallelize_shardable_queries`, right-size ingesters/store-gateways.
- **Dashboard:** Narrow time range, recording rules, lower resolution (`min step`), avoid high-cardinality aggregations.
- **Compare:** Same query in `mimirtool` vs. Prometheus; profile `cortex_query_frontend` latency.

### 37. Short-range → Prometheus, long-range → Mimir automatically

- **Mixed queries:** Not native split-by-duration; common patterns: (1) two datasources with duplicate panels hidden by variable threshold, (2) single Mimir with recent data from ingesters (near-real-time), (3) custom Grafana plugin.
- **Practical approach:** Migrate to Mimir only with ~2h querier cache; use `query_store_after` so recent data served from ingesters.
- **Recording rules:** Run in Mimir rulers; Prometheus only for scrape if dual-writing during migration.
- **Variable:** `range` selector switches datasource UID when `$__range` > 24h.

### 38. Query splitting with Mimir query-frontend

- **Behavior:** Grafana splits long range queries into sub-queries by interval; frontend merges results.
- **Helps:** Very long dashboards over weeks/months; parallelizes across queriers.
- **Hurts:** Too many sub-queries → thundering herd, duplicate work, inconsistent step alignment.
- **Tune:** `split_queries_by_interval` in datasource; match Prometheus `max_query_parallelism`.
- **When to disable:** Real-time short windows or when frontend already splits and doubling causes overload.

---

## RBAC & Multi-Tenancy

### 39. Team modified shared infrastructure dashboard — RBAC prevention

- **Folder permissions:** Shared infra folder = `Viewer` for all teams; only Platform/SRE `Editor`.
- **Dashboard permissions:** Per-dashboard override if needed; disable "Edit" for general users on golden dashboards.
- **GitOps provisioning:** Provision shared dashboards from Git with `allowUiUpdates: false` so UI edits are ephemeral/reverted.
- **Branch protection:** Changes only via PR to observability repo.
- **Audit:** Alert on dashboard save API for protected folders.
- **Fork pattern:** Teams copy to their folder for experiments; link to canonical golden dashboard.

### 40. RBAC: SREs edit all, devs view team folder, execs one dashboard

- **Roles:** Custom roles (Enterprise) or built-in: SRE = `Admin`/`Editor` global; devs = `Viewer` + folder Editor only in dev folders (avoid—use Viewer).
- **Folders:** `Executive Summary` folder with Viewer for exec group; `Team-*` folders with team Editor; `Platform` for SRE Editor.
- **Teams:** Map SSO groups to Grafana teams; assign folder permissions per team.
- **Datasource:** Devs Viewer on dashboards but no ad-hoc query to prod without SRE role.
- **Least privilege:** Executives get direct link + Viewer on single dashboard, not org-wide discovery.

### 41. Datasource permissions — prod for SRE, staging for devs

- **Enterprise datasource permissions:** `Query` + `Edit` for SRE on prod Prometheus; dev team `Query` only on staging datasource.
- **Separate UIDs:** `prometheus-prod` vs `prometheus-staging` with different URLs and credentials.
- **RBAC:** `datasources:query` scoped via `enable_permissions` and team assignments.
- **Prevent bypass:** Disable anonymous; audit API `/api/ds/query` usage.
- **Network:** Staging Prometheus not routable from dev laptops if querying outside Grafana.

### 42. Organizations vs. folders — trade-offs

- **Organizations:** Hard isolation (users, dashboards, datasources separate); high operational overhead; duplicate config.
- **Folders:** Single org, logical separation; shared SSO and plugins; easier cross-team visibility with RBAC.
- **Choose orgs:** Regulatory isolation, MSP separate customers, completely separate branding/billing.
- **Choose folders:** Most enterprises with 50 teams—folders + teams + RBAC sufficient.
- **Hybrid:** One org per environment (prod/nonprod) with folders per team.

### 43. Enterprise SSO (Okta/Azure AD) + group → role mapping

- **OAuth/SAML:** Configure Azure AD/Okta app; map `groups` claim to Grafana role sync (Enterprise).
- **Team sync:** `team_sync_enabled` maps IdP groups to Grafana teams automatically.
- **Role mapping:** e.g., `Grafana-Admins` → Admin, `Grafana-Editors` → Editor, default Viewer.
- **Lifecycle:** SCIM or periodic sync removes access when user leaves group; short session TTL.
- **Test:** Login as user in each group; verify folder and datasource access.

### 44. User queries forbidden datasource via direct API

- **Audit:** Grafana audit logs, reverse proxy access logs for `/api/ds/query`; correlate user ID and datasource UID.
- **Fix:** Enable datasource permissions; remove Admin from regular users; use service accounts with scoped tokens.
- **Network policy:** Grafana is only path to Prometheus; mTLS and network ACLs on Prometheus.
- **Prometheus:** Enforce label-based auth via proxy (e.g., oauth2-proxy, thanos tenant) if defense in depth required.
- **Review:** Pen test; rotate API keys; disable legacy API keys with broad scope.

### 45. Team-based permissions — private vs. shared dashboards

- **Private:** Dashboard in team folder with team Editor, org Viewer denied on folder.
- **Shared:** `Platform` or `SRE Golden` folder with team Viewer, platform Editor; library panels for reuse.
- **Contribution flow:** PR adds library panel or row JSON; platform merges to shared dashboard.
- **Search/tags:** Tag `team:payments` vs `shared:true` for discoverability without edit rights.

---

## Datasource Optimization & Performance

### 46. 300 users overloading Prometheus

- **Query cache:** Enterprise cache with 30–60s TTL on popular dashboards.
- **Rate limiting:** Grafana concurrency limits; Prometheus `max_concurrent_queries`; queue at query-frontend.
- **Dashboard hygiene:** Reduce auto-refresh; standardize time ranges; eliminate duplicate panels.
- **Recording rules:** Pre-compute heavy queries; federate or remote_write to Mimir for reads.
- **Read path:** Dedicated query replicas; Thanos/Mimir query-frontend separate from scrapers.
- **Governance:** Dashboard review for query count; chargeback metrics per team/dashboard.

### 47. Multiple Prometheus (per cluster) — unified view without federation

- **Grafana variables:** Multi-value `cluster` variable; query each datasource with `-- Mixed --` or repeat queries.
- **Mimir/Cortex:** Remote write all clusters to single tenant with `cluster` label—preferred at scale.
- **Thanos Querier:** Global query layer over sidecars; single Grafana datasource.
- **Recording:** Aggregate in central Mimir with `sum by (cluster)` for fleet panels.
- **Avoid:** 50 separate queries per panel—use one backend with label dimension.

### 48. Mixed datasource dashboard — Prometheus + Elasticsearch + Jaeger

- **Mixed mode:** Panel or dashboard set datasource to `-- Mixed --`; each query selects its datasource.
- **Time alignment:** Ensure all sources use UTC; sync clock; same global time picker.
- **Correlations:** Link trace_id between Jaeger and ES logs; Prometheus exemplars to Jaeger.
- **Layout:** Rows by signal type (metrics/logs/traces); shared variables for `service` where labels align.
- **Performance:** Limit ES query size; use metricized log counts for overview.

### 49. 150 queries per page load — refactor

- **Audit:** Query Inspector total; list duplicate PromQL across panels.
- **Combine:** One panel, multiple series; use `or` label sets carefully or recording rules per service group.
- **Recording rules:** `service:slo:burnrate5m` shared by many panels.
- **Repeats:** Replace 50 repeated panels with one panel and `by (service)` in query.
- **Rows:** Collapse hidden rows; lazy load detail dashboard.
- **Target:** <20 queries per dashboard view.

### 50. Query Inspector for slow queries

- **Use:** Panel → Inspect → Query; see request time, response size, raw request/response.
- **Info helps:** Step resolution, number of series returned, PromQL parse, remote read stats.
- **Optimize:** Reduce time range, increase step, drop labels, fix cardinality, add recording rule.
- **Share:** Export query JSON for reproducing in `promtool query analyze`.

---

## Incident Visualization & Real-Time Troubleshooting

### 51. Dashboards slow during incident when Prometheus stressed

- **Architecture:** Separate query path (Mimir read replicas, Thanos query) from scrape/ingest path.
- **Pre-aggregated views:** Incident dashboards query recording rules only—cheap queries under load.
- **Caching:** Short TTL cache on NOC dashboards; stale-while-revalidate acceptable during firefight.
- **Offline fallbacks:** Runbooks with `promtool`/`kubectl` commands; don't depend solely on Grafana.
- **Shed load:** Disable auto-refresh company-wide except war room; reduce ad-hoc Explore.
- **Design:** SLO burn and top-10 error panels only—defer deep dives to post-incident.

### 52. War room dashboard auto-populated from PagerDuty

- **Webhook:** PagerDuty → Grafana OnCall or custom webhook sets incident variables/annotations.
- **OnCall integration:** Auto-create incident channel with Explore links filtered by `service` from alert labels.
- **Dynamic dashboard:** URL template `?var-service={{alert.service}}&from=now-1h`; playlist of service + deps.
- **Annotations:** PD incident start/end as annotation layers.
- **Automation:** Terraform/Operator provisions per-service sub-dashboard links in runbook panel.

### 53. Blast radius dashboard for failing service

- **Service graph:** Tempo service graph or dependency map from metrics/traces.
- **Downstream metrics:** Error rate/latency of callers and callees via `grpc_client`/`http` labels with `depends_on` topology.
- **Synthetic/user impact:** Checkout success rate, queue depth, support ticket rate (business metrics).
- **Variables:** Center on `failed_service`; show hop-by-hop RED for neighbors.
- **Logs/traces:** Top errors from Loki; slowest traces from Tempo for that service.

### 54. Annotations for deploys, incidents, config changes

- **Manual:** Dashboard → Annotations → define query or built-in list.
- **Automated:** CI/CD `POST /api/annotations` with tags `deploy`, `commit`, `service`.
- **Sources:** GitHub webhooks, Argo CD sync events, PagerDuty incidents, feature flag systems.
- **Display:** Enable annotation layers on panels; color-code by tag.
- **GitOps:** Provision annotation sources via YAML.

### 55. SLO dashboard — burn rate, remaining budget, projected breach

- **Recording rules:** Multi-window burn rates from SLI metrics (Sloth/OpenSLO).
- **Panels:** Gauge for remaining budget %; time series for burn rate; stat for projected exhaustion from linear extrapolation.
- **Thresholds:** Green/yellow/red bands at 50%/25%/0% budget remaining.
- **Alerts:** Link to unified alerting on same metrics.
- **Variables:** Per-service SLO selection.

### 56. Metric spike at 2 AM every night

- **Annotations:** Overlay cron/deploy annotations; check CI schedules.
- **Compare:** Day-over-day overlay in Grafana; use `offset 1d` in PromQL.
- **Logs:** Loki query for batch jobs at 02:00; `kube_cronjob` metrics.
- **Breakdown:** Split by label `job`, `instance`, `region` to find batch vs. attack (unusual geo/IP).
- **Table:** Top contributors panel with `topk` by label.

### 57. State timeline for 100 services over 30 days

- **Data:** Prometheus `up` or custom health metric; Loki optional for state changes.
- **Panel:** State timeline with value mappings (0=down, 1=degraded, 2=up).
- **Query:** `max_over_time(health_state[$__interval])` or discrete status from recording rule.
- **Performance:** 30d at high resolution is heavy—use recording rule with 5m granularity.
- **UX:** Filter top 100 by variable; sort by worst uptime; export PDF for review.

### 58. Geomap for global errors and latency

- **Requirements:** Metrics with `geo` or `region`/`country` label; or join IP → geo via recording rule.
- **Panel:** Geomap layer with color by error rate, size by request volume.
- **Datasource:** Prometheus/Mimir with `latitude`/`longitude` labels or Grafana JSON API.
- **Use:** Identify regional outages vs. global; correlate with CDN/cloud region status.

---

## Grafana as Code & Automation

### 59. GitOps dashboards — Grafonnet, Jsonnet, Terraform

- **Repo structure:** `dashboards/`, `alerts/`, `datasources/` as Jsonnet or Terraform `grafana_dashboard` resources.
- **Grafonnet:** Reusable libraries for standard rows (RED, resources, SLO).
- **CI:** PR runs `jsonnetfmt`, renders JSON, validates with `grafanactl` or API dry-run.
- **Deploy:** Merge to main → pipeline applies to Grafana via Terraform or `grafanactl sync`.
- **Environments:** Separate branches/workspaces for staging/prod Grafana.
- **UID/version:** Pin UIDs; prevent manual drift with `disableDeletion` provisioning.

### 60. Automated dashboard testing

- **Render tests:** `grafanalib` or custom script queries each panel PromQL against Prometheus in CI.
- **API validation:** `POST /api/dashboards/db` to staging; run health check job.
- **Screenshot/regression:** Optional Percy/Chromatic on rendered PNGs via image renderer.
- **Alert tests:** Promtool rules unit tests for alert queries extracted from dashboards.
- **Block merge:** Fail PR if any query returns error or zero series in fixture environment.

### 61. HTTP API for environment provisioning

- **Workflow:** Terraform or script: create org/folders → datasources → dashboards → alert rules → teams.
- **Endpoints:** `/api/datasources`, `/api/dashboards/db`, `/api/v1/provisioning/alert-rules`, `/api/teams`, `/api/org/users`.
- **Auth:** Service account token with scoped permissions; store in vault.
- **Idempotency:** Use UIDs; PUT updates; track state in Terraform.
- **Bootstrap:** New region = apply module; smoke test with synthetic query.

### 62. Migrate 200 dashboards with different datasource UIDs

- **Export:** `GET /api/search?type=dash-db` + export each; or bulk from Git.
- **Transform:** `jq`/`sed` replace `datasource` UID references in JSON (`"uid": "old"` → `"uid": "new"`).
- **Mapping table:** CSV of old→new UID per environment; script validates all references resolved.
- **Import:** `POST /api/dashboards/db` with `overwrite`; preserve dashboard UID if URLs must stay stable.
- **Verify:** Automated panel query test post-migration.

### 63. Dashboard snapshots for postmortems

- **Snapshots:** Dashboard → Share → Snapshot; stores panel data at point in time.
- **Automation:** API `POST /api/snapshots` during incident webhook; store URL in incident ticket.
- **External:** Export PDF via reporting/image renderer for exec summary.
- **Retention:** Snapshots on Grafana DB—archive to S3 via plugin or copy JSON to postmortem doc.
- **Note:** Snapshots may contain sensitive data—restrict access.

---

## Kubernetes & Cloud Integration

### 64. Grafana on Kubernetes — HA, persistence, provisioning

- **HA:** 2+ replicas, PDB, anti-affinity; shared PostgreSQL backend (not SQLite).
- **Storage:** PVC for plugins if needed; config via ConfigMap/Secret; dashboards via Git not PVC.
- **Provisioning:** Sidecar or Operator watches ConfigMaps for datasources/dashboards in labeled folder.
- **Ingress:** TLS termination at ingress; OAuth at Grafana.
- **Secrets:** External Secrets Operator for DB password and API keys.

### 65. Kubernetes app plugin for cluster health

- **Install:** Grafana Kubernetes Monitoring app (integrated stack) or legacy k8s plugin.
- **Views:** Cluster/nodes/pods/namespaces; prebuilt dashboards for CPU, memory, restarts, PV usage.
- **Datasources:** Auto-linked Prometheus (kube-prometheus-stack) and Loki.
- **Variables:** Cluster, namespace, workload filters.
- **Ops:** Keep app updated with GKE/EKS version compatibility.

### 66. Grafana Cloud multi-cloud unified view

- **Label normalization:** Enforce `cloud`, `region`, `service` labels via Alloy/OTel collectors and recording rules.
- **Remote write:** All clouds → Grafana Cloud Mimir single tenant or logical tenants merged in queries.
- **Dashboards:** Avoid cloud-specific metric names in panels—use recording rules `service:requests:rate5m`.
- **Variables:** `cloud=aws|gcp|azure` filter; comparable SLO panels per cloud.
- **Cost:** Single Grafana Cloud stack vs. per-cloud Grafana instances.

### 67. Auto-provision dashboards when namespace created

- **Grafana Operator:** `GrafanaFolder` + `GrafanaDashboard` CRs in namespace; watch Namespace creation via controller or Helm hook.
- **Sidecar:** Prometheus Operator pattern—label namespace `grafana-dashboard: enabled`; sidecar imports ConfigMaps.
- **Templating:** Helm chart per namespace with parameterized dashboard for team name.
- **GitOps:** Argo CD ApplicationSet generates per-namespace dashboard CR from cluster generator.

### 68. Alerting for pod disruption — evictions, pressure, errors

- **Metrics:** `kube_pod_status_reason`, node pressure, `container_oom`, app error rate recording rules.
- **Unified alert:** Multi-query: eviction rate ↑ AND node memory pressure AND 5xx rate ↑ for affected workloads.
- **Correlation panel:** State timeline of evictions overlaid with latency.
- **Routing:** Namespace label → team contact point.
- **Runbook:** Link to drain/cordon procedures in notification template.

---

## Advanced Features & Production Scenarios

### 69. Critical plugin vulnerability across 50 dashboards

- **Inventory:** List plugin version on all instances; identify dashboards using plugin panel type via API search in JSON.
- **Staging:** Upgrade plugin in staging Grafana; open each dashboard visually and via CI render test.
- **Rollout:** Blue-green Grafana instances or rolling upgrade during low traffic; pin plugin version in config.
- **Mitigation:** Disable plugin via config if zero-day; replace panels with core panel types temporarily.
- **Comms:** Notify dashboard owners; deadline for migration off vulnerable panel.

### 70. Weekly SLO reports — PDF and email

- **Grafana Reporting (Enterprise/Cloud):** Schedule dashboard PDF; email distribution list.
- **Image renderer:** Headless Chrome service for PNG/PDF; scale renderer replicas.
- **Content:** SLO overview dashboard with burn rate, compliance %, top violators.
- **Automation:** Cron in Grafana or external Jenkins calling report API.
- **Archive:** Store PDFs in S3 with week stamp for compliance.

### 71. Transformations — client-side manipulation

- **Use:** Join, filter, organize fields, calculate columns when datasource can't (or for presentation-only).
- **Prefer server-side when:** Large datasets, repeated logic across users, or need single source of truth in PromQL/SQL.
- **Client preferable for:** Formatting, hiding series, merge queries from mixed datasources for one table, ad-hoc incident views.
- **Caution:** Heavy transforms on 10k series slow browser—aggregate in query first.

### 72. Cost attribution dashboard — infra metrics + AWS Cost Explorer

- **Data:** AWS CUR or Cost Explorer API → Athena/BigQuery → Grafana Infinity/Athena datasource; or prometheus `cloud_cost` exporter.
- **Join:** Shared labels `service`, `team`, `account`, `cluster` between `container_cpu` and cost metrics.
- **Panels:** Cost per request, cost per team, idle resource cost; trend vs. traffic.
- **Refresh:** Daily cost data; metrics real-time—note latency mismatch in subtitles.

### 73. NOC playlist

- **Create playlist:** Add critical dashboards (overview, network, payments, SLO).
- **Settings:** 1–5 min interval per dashboard; TV mode (`/playlists/play/1?kiosk`).
- **Refresh:** Each dashboard auto-refresh 30s; playlist advances regardless.
- **Ops:** Dedicated NOC user account; read-only; large display optimized layouts.

### 74. Public dashboards securely

- **Feature:** Enable public dashboards (Grafana 10+); share link with unauthenticated read-only view.
- **Security:** Sanitize panels—no variables exposing all namespaces; dedicated "public" dashboard clone with aggregated metrics only.
- **Network:** Optional IP allowlist at ingress; short-lived tokens.
- **Review:** No PII in labels; no drill-down to pod-level; disable Explore on public link.

### 75. Database storage full from dashboard versions

- **Cause:** Each save creates version row in `dashboard_version` table.
- **Policy:** Periodic job `DELETE FROM dashboard_version WHERE id NOT IN (latest per dashboard) AND created < NOW() - INTERVAL '90 days'`.
- **Config:** Limit versions in Enterprise or use provisioning-only (no UI saves).
- **Backup:** Export JSON to Git before purge; test restore.
- **Prevent:** Discourage UI edits on provisioned dashboards.

### 76. Chaos engineering dashboard

- **Inputs:** Experiment ID label on metrics during chaos window; annotate start/end from chaos tool webhook.
- **Panels:** SLO burn, error budget, latency before/during/after; comparison with `offset` or time regions.
- **Logs/traces:** Spike in errors linked to experiment label `chaos_experiment=network-delay`.
- **Safety:** Auto-stop links and abort threshold panels visible during game day.

### 77. Custom incident management via webhooks

- **Contact point:** Webhook type with custom payload template including alert labels and URLs.
- **Retry:** Grafana sends retries on 5xx; implement idempotent receiver with dedup key `fingerprint`.
- **Failure handling:** Dead-letter queue on receiver; alert on webhook delivery failures via meta-monitoring.
- **Auth:** HMAC signature validation on receiver side.

### 78. Financial services — SOC 2 and PCI-DSS

- **Audit logging:** Enterprise audit logs shipped to SIEM; log all auth, datasource query, dashboard changes.
- **Access:** SSO + MFA; no shared accounts; quarterly access review.
- **Data:** No cardholder data in metrics/logs; segment prod PCI workloads separate org/datasource.
- **Retention:** Policy-aligned metrics/logs retention; encrypted at rest and in transit (TLS 1.2+).
- **Secrets:** Vault integration; no credentials in dashboard JSON.
- **Compliance scans:** Regular penetration tests; plugin allowlist only.

---

## Basic Questions

### 1. What is Grafana and its primary use case?

Grafana is an open-source observability platform for visualizing time-series metrics, logs, and traces. Its primary use case is building dashboards and alerts on top of datasources like Prometheus, Loki, and Tempo so teams can monitor system health and investigate incidents in one UI.

### 2. What is a Grafana datasource and how do you add one?

A datasource is a configured connection to a metrics, log, or trace backend. Add one via **Connections → Data sources → Add data source**, choose the type (e.g., Prometheus), enter URL and auth, then **Save & test**.

### 3. Difference between a panel and a dashboard?

A **dashboard** is a collection of visualizations arranged in rows. A **panel** is a single visualization (graph, stat, table, etc.) on that dashboard, each with its own queries and display options.

### 4. How do you create a new dashboard?

Click **Dashboards → New → New dashboard**, add panels with **Add visualization**, configure queries and layout, then **Save dashboard** with a name and folder.

### 5. What is a Grafana variable?

A variable is a dynamic placeholder (dropdown, query, custom) used in panel queries and titles. It makes one dashboard reusable across environments, clusters, or services without duplicating dashboards.

### 6. Purpose of the time range picker?

It sets the global start/end time for all panels on the dashboard, controlling which historical data is queried and displayed.

### 7. How do you share a dashboard with a team member?

Save the dashboard to a shared folder with appropriate folder permissions, or share a direct URL (and ensure the teammate has Grafana access and datasource permissions).

### 8. What is Grafana Loki and how does it differ from Prometheus?

Loki is a log aggregation system optimized for label-based indexing (like Prometheus, but for logs). Prometheus stores numeric time-series metrics; Loki stores log lines and is queried with LogQL, not PromQL.

### 9. Default port for Grafana?

**3000** (HTTP).

### 10. How do you add a Prometheus datasource?

**Connections → Data sources → Add → Prometheus**, set the Prometheus URL (e.g., `http://prometheus:9090`), configure auth if needed, **Save & test**.

### 11. Grafana alert vs. Prometheus alert?

Prometheus alerts are evaluated by Prometheus/Alertmanager from PromQL rules in Prometheus config. Grafana alerts (unified alerting) are evaluated by Grafana from dashboard or Grafana-managed rules and route through Grafana notification policies—supporting multiple datasources and centralized routing.

### 12. Purpose of Explore view?

Explore is an ad-hoc query workspace for investigating metrics (Prometheus), logs (Loki), or traces (Tempo) without building a permanent dashboard—ideal for incident debugging.

### 13. How do you create a simple time series panel?

Add a panel → choose **Time series** visualization → select Prometheus datasource → enter a PromQL query (e.g., `rate(http_requests_total[5m])`) → apply unit and legend → save.

### 14. What is a Grafana annotation?

An annotation is a vertical marker on graphs indicating events (deploys, incidents). Use them to correlate metric changes with operational events.

### 15. Bar chart vs. histogram panel?

A **bar chart** displays categorical or time-bucketed values as bars. A **histogram** visualizes distribution of values in buckets over time (common for latency distributions from Prometheus histogram metrics).

### 16. Auto-refresh every 30 seconds?

Dashboard settings (gear icon) → **General → Auto refresh** → select `30s` (or add custom interval in dashboard JSON).

### 17. What is the stat panel used for?

Displaying a single prominent numeric value—current CPU, error rate, or "last non-null" summary—with optional sparkline and thresholds.

### 18. Import a pre-built dashboard from Grafana.com?

**Dashboards → New → Import**, enter dashboard ID from grafana.com/grafana/dashboards, select your Prometheus datasource, **Import**.

### 19. Purpose of the table panel?

Display query results in tabular form—multiple labels as columns, useful for top-N lists, inventory views, and multi-series comparisons.

### 20. Configure thresholds for color changes?

Panel → **Edit → Field → Thresholds** → add steps (e.g., green < 80, yellow 80–95, red > 95) → choose threshold display mode (absolute, percentage).

### 21. Alert state history?

Unified alerting stores alert instance history (pending, firing, resolved). Access via **Alerting → Alert rules → View history** or Alerting → Silences/Active notifications UI.

### 22. Grafana OSS vs. Enterprise?

OSS is free/open-source with core features. Enterprise adds advanced RBAC, reporting, query caching, audit logs, SAML enhancements, and commercial support.

### 23. Create a folder for dashboards?

**Dashboards → New → New folder**, name it, then save dashboards into that folder via save dialog or drag-and-drop in the browse view.

### 24. Purpose of dashboard links?

Dashboard links connect related dashboards (drill-downs) or external runbooks—static URLs or templated with variables like `${service}`.

### 25. Configure SMTP for email alerts?

Set `[smtp]` in `grafana.ini` (host, user, password, from_address) or Helm values; then create an **Email** contact point in Alerting referencing that SMTP config.

### 26. What is a playlist?

A playlist rotates through multiple dashboards automatically—used for NOC wall displays and executive overview screens.

### 27. Duplicate a panel?

Open panel menu (title dropdown) → **More → Duplicate**, or copy/paste panel JSON in dashboard settings.

### 28. Purpose of `$__interval`?

Grafana-calculated minimum step between data points based on panel width and time range—used in PromQL to align `rate()` windows and avoid over-fetching.

### 29. Configure datasource health check?

In datasource settings, **Save & test** runs a health check; optionally enable **Alerting** on datasource health via Grafana's built-in health notification or synthetic checks.

### 30. What is Grafana Tempo?

Tempo is Grafana's distributed tracing backend storing trace spans. It integrates with Grafana for trace search, service graphs, and correlation with metrics and logs.

### 31. Create an alert rule from a Prometheus query?

**Alerting → Alert rules → New alert rule**, select Prometheus datasource, enter query, set condition/threshold and `for` duration, assign contact point via notification policy.

### 32. Purpose of `$__timeFilter`?

Macro expanding to Prometheus time filter for the selected range (legacy/influx contexts); in PromQL prefer explicit range selectors with `$__rate_interval`.

### 33. Configure organization settings?

**Administration → General → Default preferences** (theme, timezone, home dashboard); org admins manage users, teams, and preferences under **Administration**.

### 34. Contact point and relation to alerting?

A **contact point** defines a notification destination (Slack, PagerDuty, email). Alert rules fire **alert instances** routed by **notification policies** to the appropriate contact point.

### 35. Query Inspector for slow panels?

Panel → **Inspect → Query** shows request duration, response size, and raw query—use it to see if slowness is Grafana, network, or the backend.

### 36. Purpose of `$__range`?

The selected dashboard time range duration (e.g., `6h`)—usable in queries and alert conditions; distinct from `$__interval` which is step size.

### 37. Multiple datasources in one panel?

Set panel datasource to **-- Mixed --**, then assign each query (A, B, C) its own datasource in the query editor.

### 38. What is Grafana Mimir?

Mimir is Grafana's horizontally scalable long-term Prometheus-compatible metrics storage, providing multi-tenancy, global query, and durable object-store backend.

### 39. Local account authentication?

Enable `[auth]` in `grafana.ini`; create users under **Administration → Users**; assign passwords on invite or via `grafana cli admin reset-admin-password`.

### 40. Dashboard version history?

Grafana stores previous saves of a dashboard. Access via dashboard settings → **Versions** to view, compare, or restore an earlier revision.

### 41. Show last value when data is missing?

Panel → **Standard options → No value** → choose **Last non-null value** (or similar) under value mapping / connect null values settings depending on panel version.

### 42. `rate` vs. `irate` in Prometheus datasource?

`rate()` averages per-second rate over the range window—smoother, better for alerts and graphs. `irate()` uses only last two points—instantaneous, spikier, more sensitive to scrape timing.

### 43. Configure notification policies?

**Alerting → Notification policies** → edit default policy tree, add child routes with label matchers, contact points, grouping, and mute timings.

### 44. Purpose of transformations?

Transformations reshape query results in Grafana (filter, join, rename, calculate) without changing the backend query—useful for presentation and combining queries.

### 45. Export dashboard as JSON?

Dashboard settings → **JSON Model → Copy** or **Save to file**; also available via API `GET /api/dashboards/uid/:uid`.

---

## Intermediate Questions

### 1. Chained variables — cluster filters namespace

Create a `cluster` query variable from `label_values(cluster)`. Create `namespace` with query `label_values(kube_namespace_labels{cluster="$cluster"}, namespace)` and set **Refresh** to trigger when `cluster` changes. Chain ensures namespace dropdown only shows namespaces in the selected cluster.

### 2. Unified alerting routing by label matchers

Build a notification policy tree: child routes match `team=payments` → Payments Slack, `team=platform` → Platform PagerDuty. Use `continue: true` only when multiple destinations needed. Group by `alertname` and `cluster` to reduce noise.

### 3. RBAC — developers view dashboards, not edit datasources/alerts

Assign developers **Viewer** org role. Grant **Editor** only on team-specific folders if they may edit team dashboards. Use datasource permissions (Enterprise) to deny `datasources:query` on prod for dev role; restrict **Alerting** admin to SRE custom role.

### 4. Query caching to reduce Prometheus load

Enable Enterprise query cache with TTL 30–60s on the Prometheus datasource. Standardize popular dashboards and time ranges to improve hit rate. Combine with recording rules so cached queries are cheaper.

### 5. SLO dashboard with error budget burn rate

Create Prometheus recording rules for SLI ratio and multi-window burn rates (or use Sloth). Dashboard panels: compliance %, remaining budget gauge, burn rate time series with thresholds. Alert rules in Grafana mirror the same burn-rate metrics.

### 6. Correlate Loki logs with Prometheus metrics

Configure Loki derived fields to extract `trace_id` or `request_id`. Add Prometheus **Exemplars** linking to Tempo. Use **Data links** on metric panels to open Loki with matching `service` and time range. Shared template variables align labels.

### 7. Mixed datasource — Prometheus + Elasticsearch in one panel

Set panel to **-- Mixed --**; query A from Prometheus for request rate, query B from Elasticsearch for log-derived error count; use transformations to join on `service` field if timestamps align.

### 8. Alert evaluation interval and detection latency

Grafana evaluates alert rules on a fixed interval (e.g., 1m). An alert with `for: 5m` must breach for five consecutive evaluations—worst-case detection latency ≈ interval + `for` + query duration. Shorter interval improves speed but increases load.

### 9. State timeline for service health transitions

Query a numeric state metric (0=down, 1=degraded, 2=up) or map from `up`. Use **State timeline** visualization with value mappings and discrete thresholds. Recording rules can compress raw flapping into state changes.

### 10. Tempo trace-to-metrics and metrics-to-trace

In Prometheus datasource, configure exemplar destination to Tempo. In Tempo datasource, enable **Traces to logs/metrics** links with consistent service labels. Requires exemplars on histograms and metrics-generator or RED metrics in Tempo/Prometheus.

### 11. Dashboard provisioning via ConfigMaps

Mount dashboard JSON in a ConfigMap labeled for Grafana sidecar (`grafana_dashboard: "1"`). Sidecar or Grafana Operator imports into Grafana. Datasources provisioned similarly via `grafana_datasource` ConfigMap or Operator CRD.

### 12. `$__rate_interval` vs. `$__interval`

`$__interval` is chart resolution step. `$__rate_interval` adds scrape interval + buffer for PromQL `rate()`/`increase()` to cover full windows—always use `$__rate_interval` inside `rate()` to avoid graph artifacts.

### 13. Silences via Grafana API

`POST` to Alertmanager-compatible silence API or use Terraform `grafana_mute_timing` / silence resources with matchers like `service=myapp`. Integrate CI/CD to create timed silences during deploys.

### 14. LDAP integration for enterprise SSO

Configure `[auth.ldap]` in `grafana.ini` with bind DN, search filters, and `group_dn` mappings to Grafana roles. Enable sync on login; test with ldapsearch before production. Prefer SAML/OAuth for MFA integration.

### 15. Multi-value variable across services

Create query variable `service` with **Multi-value** and **Include All** enabled. Use `service=~"$service"` in PromQL regex matcher to filter multiple services simultaneously.

### 16. Loki log volume vs. log content queries

**Log volume** uses metric queries like `sum(count_over_time({app="foo"}[1m]))`—fast aggregates. **Log content** returns actual log lines—slower, limited by `max_entries`. Use volume for trends; content for investigation.

### 17. Service graph panel from Tempo

Enable Tempo metrics-generator service graph. Add **Service graph** panel, select Tempo datasource, filter by time range and namespace. Shows service nodes and edges with RED indicators.

### 18. PagerDuty contact point with severity mapping

Create PagerDuty contact point with integration key. In notification policy, route `severity=critical` to PD with template setting `severity` field. Map Grafana labels to PD custom details via message templates.

### 19. Dashboard as code with Jsonnet and Grafonnet

Write Jsonnet using Grafonnet libraries to generate dashboard JSON parametrically (service name, SLO targets). CI renders JSON and provisions via Terraform or `grafanactl`. Eliminates manual JSON drift.

### 20. Query splitting for long time ranges

Grafana splits wide queries into sub-queries by interval for parallel execution. Improves long-range dashboard load times on Mimir/Thanos. Disable if it causes query fan-out overload on small backends.

### 21. Datasource permissions for production metrics

Enterprise: enable permissions on prod Prometheus; grant Query to SRE team only. Staging datasource available to developers. Combine with separate network paths for defense in depth.

### 22. Mimir datasource for multi-tenant queries

Configure Mimir URL with `X-Scope-OrgID` header per tenant in datasource JSON secure fields. One datasource per team or dynamic header via proxy—not native in OSS without Enterprise tricks.

### 23. Exemplar support — metrics to traces

Enable exemplars in Prometheus scrape/config. In Grafana Prometheus datasource, set exemplar trace ID destination to Tempo. Click exemplar markers on heatmap/time series to open the trace.

### 24. Alert state machine — pending, firing, resolved

Alerts start **Normal**; breach → **Pending** (during `for` duration); sustained breach → **Firing**; recovery → **Resolved** (notification optional). Pending prevents flapping pages; resolved confirms fix to on-call.

### 25. Reporting — scheduled PDF reports

Grafana Enterprise/Cloud Reporting: select dashboard, schedule weekly, add email recipients. Requires image renderer. Reports snapshot current panel state at generation time.

### 26. Kubernetes monitoring with Kubernetes app

Install Grafana Kubernetes Monitoring solution; connects Prometheus, Loki, and prebuilt dashboards for clusters, nodes, workloads. Configure fleet management for multiple clusters from one Grafana.

### 27. Public dashboards for external stakeholders

Enable feature; publish specific dashboard; share read-only URL. Clone dashboard with aggregated, non-sensitive metrics only; review labels for PII before publishing.

### 28. Transformation pipeline — three examples

(1) **Filter by name** — hide unused series. (2) **Merge** — combine queries A+B into one table. (3) **Add field from calculation** — compute error % from success/fail queries client-side.

### 29. Alert notification templates

Edit contact point message template using Go templates: `{{ .Labels.service }}`, `{{ .Annotations.summary }}`, links to dashboard and runbook. Customize Slack blocks and PagerDuty payloads per team.

### 30. High availability with shared database

Run 2+ Grafana replicas behind load balancer; use PostgreSQL (not SQLite) for sessions, dashboards, users. Store secrets in vault; use sticky sessions or shared session store if needed.

### 31. Dashboard search and tagging

Tag dashboards `team:payments`, `slo:tier1` in JSON metadata. Use Grafana search filters by tag. Enforce tagging standards in CI when provisioning dashboards.

### 32. Loki label browser

In Explore with Loki, **Label browser** shows available label names and values for the time range—use to discover streams before writing LogQL. Helps explore unknown log structures.

### 33. Canvas panel for status boards

**Canvas** panel allows custom layouts, text, shapes, and metric-driven colors—build NOC status boards with service tiles turning red/green from query thresholds without external tools.

### 34. Azure Monitor multi-subscription

Add Azure Monitor datasource with service principal; use template variables for `subscription`, `resourceGroup`, and `resource`. Repeat panels per subscription or use multi-value variable with Resource Graph queries.

### 35. Alert grouping to reduce noise

Notification policy: `group_by: ['alertname','cluster']`, `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`. One notification per group instead of per pod.

### 36. Flux vs. InfluxQL for InfluxDB

**InfluxQL** is SQL-like for InfluxDB 1.x. **Flux** is InfluxData's newer functional language for InfluxDB 2.x/Cloud with richer data shaping. Grafana supports both depending on Influx version and datasource plugin.

### 37. Heatmap for latency distribution

Query Prometheus histogram buckets (`le` label); visualize as **Heatmap** panel showing request count per latency bucket over time—reveals tail latency shifts.

### 38. Google Cloud Monitoring for GKE

Add **Google Cloud Monitoring** datasource; use filters for GKE cluster, namespace, pod metrics. Template variables from resource labels; combine with logging datasource for correlated views.

### 39. Variable refresh when time range changes

Variable settings → **Refresh: On time range change** re-runs variable query when user zooms—ensures label values match visible data (useful for Loki/Prometheus cardinality).

### 40. Prometheus recording rules vs. raw queries in Grafana

Recording rules pre-aggregate expensive PromQL; Grafana queries the recorded metric name—faster and consistent. Raw queries offer flexibility but higher cost; use recording rules for shared dashboard patterns.

---

## Advanced Questions

### 1. Grafana architecture for 500-team enterprise

- **Topology:** Shared Grafana HA cluster per environment; folders per team/domain; centralized datasource provisioning with tenant headers to Mimir.
- **SSO:** Okta/Azure AD with group sync to teams; custom roles for SRE/platform vs. viewer.
- **GitOps:** All dashboards/alerts in Git; `grafanactl`/Terraform apply; no manual golden dashboard edits.
- **Isolation:** Mimir multi-tenancy + datasource permissions; optional separate orgs for regulated units.
- **Scale:** Enterprise query cache, read replicas, rate limits per org.
- **Governance:** Dashboard linter CI, catalog linking services to owners, audit logs to SIEM.

### 2. Unified observability — Mimir, Loki, Tempo with correlation

- **LGTM stack:** Single Grafana with datasources linked via exemplars, derived fields, and correlations feature.
- **Consistent labels:** `service`, `namespace`, `cluster` on metrics, logs, traces via OTel collector.
- **Explore workflows:** Metric spike → exemplar/trace → logs for same trace_id.
- **Incident view:** Prebuilt correlated dashboard per tier-1 service.
- **OnCall:** Grafana OnCall routes from unified alerts across signal types.

### 3. Multi-tenancy for MSP with 200 customers

- **Orgs per customer** (strong isolation) or Mimir/Loki tenants with branded Grafana subdomains (Enterprise).
- **Provisioning:** Terraform module per customer onboarding (org, datasources, default dashboards).
- **Branding:** Custom themes/logos per org; white-label URLs.
- **Billing:** Per-tenant usage metrics from Mimir/Loki billing exporters.
- **Ops cost:** Automate everything—manual org creation doesn't scale at 200 customers.

### 4. Dashboard availability during Prometheus failure

- **Read path:** Query Mimir/Thanos with cached blocks; Grafana query cache for last good data.
- **Fallback datasource:** Variable switches to secondary region Prometheus.
- **Snapshots:** NOC uses pre-rendered snapshots if live queries fail.
- **Design:** Incident dashboards use only pre-aggregated recording rules on HA backend.
- **Status:** Display "data stale since" stat from `timestamp()` comparison.

### 5. SRE platform — auto SLO dashboards for 500 services

- **Sloth/OpenSLO:** Generate recording rules + Grafana dashboards + alert rules from SLO YAML per service.
- **Service catalog:** Sync ownership labels; auto PR when new service registered.
- **Templates:** Jsonnet library parameterized by service name, SLO target, SLI queries.
- **Incident correlation:** Auto-link burn-rate alerts to dependency and deploy annotations.

### 6. Alerting at scale — 10,000 rules, sub-minute evaluation

- **Sharding:** Grafana alerting horizontal scalability (Enterprise) or split across multiple Grafana instances by domain.
- **Efficient rules:** Recording rules in Mimir; alerts on cheap metrics not raw high-cardinality.
- **Evaluation groups:** Tune group interval; avoid 1s eval on all rules.
- **Notification:** Dedicated Alertmanager receivers; rate limit webhooks; OnCall for delivery guarantees.
- **Testing:** Continuous validation of rule health and notification latency SLO.

### 7. Financial services — data residency

- **Regional Grafana stacks:** EU metrics never query US Mimir; separate datasources per region locked by network policy.
- **SSO geo-fencing:** Route users to regional Grafana via DNS geo-routing.
- **Mimir:** `X-Scope-OrgID` per region; blocks stored in region-local object storage.
- **Audit:** Prove in audits that cross-border query paths are disabled.
- **Dashboards:** Git repo per region or templated with region-locked datasource UIDs.

### 8. Incident management platform with war rooms

- **PD/webhook → OnCall:** Auto-create incident, Slack channel, variable-scoped dashboard URL.
- **Timeline:** Annotations from deploys, alerts, human notes via API.
- **Correlation:** Pre-indexed service dependency graph panels.
- **Post-incident:** Auto snapshot dashboards at incident close; attach to postmortem template.

### 9. RBAC for nested team hierarchies

- **Folder tree:** `Platform/`, `Platform/Team-A/`, shared `Platform/Golden` with inherited Viewer.
- **Custom roles:** Fine-grained `dashboards:write` without `datasources:write`.
- **Team sync:** IdP groups map to nested Grafana teams; parent team Viewer on child folders.
- **Shared infra:** Break-glass Editor for platform only; developers propose changes via PR.

### 10. 10,000 concurrent dashboard users

- **Grafana scale:** 10+ replicas, PostgreSQL tuned, connection pooling.
- **Query path:** Mimir query-frontend + aggressive caching; ban expensive ad-hoc queries.
- **CDN/edge:** Cache static assets; rate limit API.
- **Dashboard standards:** Low query count, long min step on popular boards.
- **Load test:** k6 against Grafana API with realistic dashboard refresh patterns.

### 11. Cost observability platform

- **Ingest CUR into BigQuery/Athena;** Infinity/SQL datasource in Grafana.
- **Join keys:** `service`, `team`, `k8s_namespace` aligned with kube-state-metrics.
- **Panels:** Unit cost per request, idle cost, trend vs. traffic.
- **Alerts:** Weekly cost anomaly to FinOps Slack.

### 12. Multi-cluster Kubernetes alerting — 50 clusters

- **Label:** `cluster` on all metrics via external labels from Prometheus Agent.
- **Rules:** One Grafana rule set with `cluster` in group_by; notification policy routes `cluster=prod-eu` to EU on-call.
- **Terraform:** `for_each` cluster applies same rule module with cluster parameter.
- **Testing:** Synthetic alert per cluster monthly.

### 13. Auto-provision on microservice deploy via CI/CD

- **Pipeline:** New service repo → registers in catalog → triggers Terraform/Operator to create folder, dashboard from template, datasource labels, alert rules.
- **GitOps:** Merge observability PR alongside service deploy.
- **UID:** Service name as stable dashboard UID.

### 14. Loki at 10TB/day scale

- **Ingest:** Distributed Loki with S3, appropriate chunk/target sizes; avoid high-cardinality labels.
- **Query:** Query-frontend, caches, `max_query_parallelism`, log volume metrics for dashboards not raw logs.
- **Grafana:** Short default ranges; recording rules from LogQL for alerts.
- **Retention:** Tiered storage; different retention per log stream.

### 15. Chaos engineering observability platform

- **Experiment webhook →** annotations and `chaos_run_id` label.
- **Dashboard:** Before/during/after SLO comparison; blast radius service graph.
- **Automated stop:** Alert if error budget burn exceeds threshold during experiment.

### 16. Security hardening

- **TLS:** Terminate at ingress; TLS to datasources.
- **Secrets:** Vault, no secrets in dashboards; rotate API keys.
- **Auth:** SSO+MFA; disable anonymous; short session lifetime.
- **Audit:** Enterprise audit logs; plugin signature verification.
- **Scanning:** Container image CVE scans; regular penetration tests.

### 17. Real-time BI + operational metrics

- **Mixed datasources:** Prometheus for ops KPIs; SQL/BigQuery for revenue/conversion.
- **Sync:** ETL business metrics to Prometheus as gauges for unified timeline or use transformations in Grafana.
- **Governance:** Separate folders; exec Viewer only on BI+ops summary.

### 18. Enterprise plugin management — 50 instances

- **Allowlist:** Only approved plugins; version pinned in Helm values.
- **CI:** Staging upgrade test across representative dashboards.
- **Rollout:** Canary instances → wave update; rollback procedure documented.

### 19. Capacity planning platform

- **Historical queries:** Mimir long-range `predict_linear` or ML plugin for forecast.
- **Dashboard:** CPU/memory trend, days-until-full, recommended replica count.
- **Export:** Weekly PDF to infra planning team.

### 20. Disaster recovery — restore in 30 minutes

- **Git:** All dashboards/alerts/datasources as code—source of truth.
- **DB backup:** PostgreSQL snapshots for users/preferences if not fully in Git.
- **Runbook:** New region Grafana from Helm + restore DB + `terraform apply` dashboards.
- **RTO drill:** Quarterly restore test to blank cluster.

### 21. A/B testing dashboard layouts

- **Hypothesis:** Layout A (RED top) vs. B (topology first) reduces MTTR.
- **Method:** Randomize incident commander assignment; measure time-to-mitigation from incident system.
- **Implementation:** Two dashboard UIDs; OnCall link randomly or by team cohort.
- **Analyze:** Statistical comparison over quarter; adopt winning layout.

### 22. Service catalog integration

- **Links:** Dashboard panels link to Backstage/service catalog for owner, tier, runbook.
- **Variables:** Populate from catalog API via Grafana Infinity/JSON API datasource.
- **On-call:** Embed PagerDuty schedule in dashboard header via plugin or link.

### 23. HIPAA compliance monitoring

- **No PHI in metrics/logs** — validate instrumentation guidelines.
- **Access:** Audit logs, BAA with Grafana Cloud if used, encryption everywhere.
- **RBAC:** Minimum necessary; break-glass audited.
- **Retention:** Policy-aligned; automatic purge jobs.

### 24. Tempo at 1 billion spans/day

- **Ingest:** Kafka + distributors; appropriate sampling (head + tail for errors).
- **Storage:** S3 backend; compactors scaled; partition by trace ID.
- **Query:** Query-frontend caches; limit trace size in UI.
- **Grafana:** Search by trace ID and structured tags; metrics-generator for RED without scanning all spans.

### 25. Executive dashboards with freshness and SLA

- **Aggregated metrics only;** no pod-level detail.
- **Freshness indicator:** Stat panel `time() - max(timestamp(metric))`.
- **SLA summary:** Tier-1 services compliance table from SLO recording rules.
- **Reporting:** Weekly PDF email; public-safe clone for board meetings.

---

## Rapid-Fire Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | Default port for Grafana? | **3000** |
| 2 | Dashboard export format? | **JSON** |
| 3 | Purpose of `$__from` and `$__to`? | Unix epoch milliseconds for the selected time range start and end, usable in queries and links. |
| 4 | What does the `stat` panel display? | A single aggregated numeric value (optionally with sparkline/color thresholds). |
| 5 | Difference between Grafana `alert` and `recording rule`? | Alerts notify on conditions; recording rules precompute and store new time series without notifications. |
| 6 | Reset admin password from CLI? | `grafana cli admin reset-admin-password <newpassword>` (or `grafana-cli` depending on install). |
| 7 | Purpose of `mixed` datasource? | Allows multiple datasources in one panel, each query pointing to a different backend. |
| 8 | What does the `gauge` panel display? | A single value on a radial/minimal gauge against min/max thresholds. |
| 9 | Purpose of `provisioning` directory? | Auto-load datasources, dashboards, and plugins from disk/ConfigMaps at startup without UI setup. |
| 10 | Grafana OSS vs. Grafana Cloud? | OSS is self-hosted open source; Cloud is Grafana Labs' hosted managed LGTM/observability stack. |
| 11 | What does `$__interval` represent? | Auto-calculated query step/resolution based on time range and panel width. |
| 12 | Enable anonymous access? | Set `[auth.anonymous] enabled = true` in `grafana.ini` (use only with strict read-only/public dashboards). |
| 13 | Purpose of panel `overrides`? | Per-field or per-series customization of colors, units, thresholds without separate panels. |
| 14 | What does the `news` panel display? | RSS/Atom feed items from a configured feed URL. |
| 15 | Purpose of panel `repeat`? | Duplicates a row/panel for each value of a variable (e.g., one panel per server). |
| 16 | Configure session timeout? | `[session]` `session_life_time` in `grafana.ini` or `login_maximum_lifetime_duration` settings. |
| 17 | `bar gauge` vs. `gauge`? | Bar gauge shows horizontal/vertical bars per series; gauge shows radial/arc single-value display. |
| 18 | What does Explore mode allow? | Ad-hoc querying and investigation of metrics, logs, and traces without saving a dashboard. |
| 19 | Purpose of `link` panel type? | Deprecated/utility panel for dashboard navigation links between boards or external URLs. |
| 20 | PostgreSQL instead of SQLite? | Set `[database] type=postgres` with host, user, password in `grafana.ini` or Helm `grafana.database` values. |
| 21 | Purpose of `fieldConfig` in panel JSON? | Defines defaults and per-field display settings (unit, thresholds, color) for panel data. |
| 22 | What does the `logs` panel display? | Log lines from log datasources (Loki, Elasticsearch, etc.) with optional labels and wrapping. |
| 23 | Purpose of `threshold` configuration? | Changes colors/styles when values cross defined limits for visual alerting on panels. |
| 24 | Configure `smtp` for email? | `[smtp]` section in `grafana.ini` (enabled, host, user, password, from_address). |
| 25 | `contact point` vs. `notification policy`? | Contact point is the destination (Slack/email); notification policy routes which alerts go to which contact points. |
| 26 | What does `flame graph` panel visualize? | Hierarchical profile/stack data (e.g., CPU profiles) as a flame graph. |
| 27 | Purpose of dashboard `uid`? | Stable unique identifier for API, provisioning, and URLs—survives renames unlike numeric ID. |
| 28 | Configure rendering service for PDF? | Deploy Grafana Image Renderer; set `rendering` server URL in Grafana config; enable reporting. |
| 29 | Purpose of `alerting` folder in provisioning? | YAML files for provisioning alert rules, contact points, and notification policies at startup. |
| 30 | What does the `traces` panel display? | Distributed trace spans from Tempo/Jaeger/Zipkin in trace view format. |
| 31 | Purpose of `correlations` feature? | Predefined links between datasources (e.g., Loki → Tempo) for one-click pivot during investigation. |
| 32 | Configure `cookie_samesite`? | Set `[security] cookie_samesite = lax|strict|none` in `grafana.ini` for CSRF/cross-site control. |
| 33 | `alert rule` vs. `alert instance`? | Rule is the definition; instance is a specific label set evaluation currently pending/firing/resolved. |
| 34 | What does `geomap` panel display? | Geospatial data on a map with layers (coordinates, regions, heatmaps). |
| 35 | Purpose of `data source uid` in dashboard JSON? | References which datasource to use; must match provisioned UID when moving between environments. |
| 36 | Configure `max_open_files`? | System-level `ulimit` for Grafana process; tune in systemd unit or container limits—not a primary Grafana.ini knob. |
| 37 | Purpose of dashboard `tags`? | Searchable metadata for organizing and filtering dashboards in the UI. |
| 38 | What does `xy chart` panel display? | X vs. Y relationship (scatter/line) between two metrics or fields. |
| 39 | Purpose of `alert group` in unified alerting? | Collection of alert rules evaluated together on the same schedule (`evaluation group`). |
| 40 | Configure `allow_embedding`? | `[security] allow_embedding = true` in `grafana.ini` to permit iframe embedding (consider CSP risks). |
| 41 | `panel` vs. `row`? | Row is a collapsible container/group; panel is an actual visualization inside a row. |
| 42 | What does `histogram` panel display? | Distribution of values in buckets over time (typically from Prometheus histogram metrics). |
| 43 | Purpose of `snapshot`? | Point-in-time frozen copy of dashboard data shareable via URL without live datasource access. |
| 44 | Configure `auto_assign_org_role`? | `[users] auto_assign_org_role = Viewer` assigns default role when users join an org. |
| 45 | Purpose of `alert evaluation group`? | Defines how often a set of alert rules is evaluated together (interval grouping). |
| 46 | What does `candlestick` panel display? | Open/high/low/close financial-style time series bars. |
| 47 | Purpose of `library panel`? | Reusable panel stored centrally; referenced by multiple dashboards for consistent visualization. |
| 48 | Configure `login_maximum_inactive_lifetime_duration`? | `[auth]` or `[session]` settings in `grafana.ini` / Grafana 9+ `login_maximum_inactive_lifetime_duration`. |
| 49 | `folder` vs. `general` dashboard location? | General is the default root folder; named folders organize dashboards with separate permissions. |
| 50 | What does `trend` panel display? | Compact sparkline-style trend of a time series for dense dashboards (Grafana 10+ panel type). |

---
