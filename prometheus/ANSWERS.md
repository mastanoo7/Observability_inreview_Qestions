# Prometheus Interview Questions — Answers

Comprehensive answers to all questions in [README.md](README.md).

---

## PromQL & Query Optimization

### 1. `rate()` spikes after pod restart — counter reset vs genuine spike

**Diagnosis:**
- Compare raw counter values with `rate()` output. A counter reset shows as a single-sample drop in the counter followed by a spike in `rate()` equal to `(new_value / scrape_interval)` rather than actual throughput.
- Check `kube_pod_container_status_restarts_total` or pod start time labels; correlate spike timestamp with restart.
- Use `resets(metric[1h])` — a spike coinciding with `resets() > 0` indicates counter reset handling, not real traffic.
- Query raw counter in Grafana table view; genuine spikes show monotonic increase without a downward jump.

**PromQL adjustments:**
```promql
# rate() handles resets; for very noisy short windows, use longer range
rate(http_requests_total[5m])

# increase() over longer window smooths single-reset artifacts
increase(http_requests_total[1h])

# Detect resets explicitly
resets(http_requests_total[1h]) > 0
```

For alerting, prefer `rate(...[5m])` or longer over `[1m]` after restarts. Consider recording rules with longer windows for stability.

---

### 2. 99th percentile latency across microservices with histograms

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket{namespace="prod"}[5m])
  )
)
```

**Trade-offs:**
- `histogram_quantile()` is **estimated** from bucket boundaries, not exact percentiles.
- Coarse buckets (e.g., 0.1, 0.5, 1, +Inf) underestimate/overestimate p99; add buckets near your SLO target (e.g., 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10).
- Finer buckets improve accuracy but increase cardinality (`le` label per bucket per series).
- Summing buckets **before** `histogram_quantile()` (`sum by (le, ...)`) is correct for aggregated percentiles across instances; never average pre-computed quantiles.

---

### 3. `irate()` noise — when to switch to `rate()`

**Switch to `rate()` when:**
- Dashboards show jittery graphs (short scrape intervals + `irate()`'s last-two-sample sensitivity).
- Alerting on sustained throughput/error rates (alerts need smoothed signal).
- Comparing rates across services with different scrape timing.

**`irate()` is appropriate for:** fast anomaly detection on counters where you need immediate reaction to sudden changes (with acceptance of noise).

**Alerting implications:** `irate()` causes more false positives/negatives on brief spikes; `rate()` over `[5m]` aligns with typical alert `for:` durations and reduces flapping.

```promql
# Alerting (stable)
rate(errors_total[5m]) / rate(requests_total[5m]) > 0.05

# Fast spike detection (noisy)
irate(errors_total[5m]) / irate(requests_total[5m]) > 0.05
```

---

### 4. Recording rule slower than evaluation interval

**Diagnosis:**
- Check `prometheus_rule_group_last_duration_seconds` and `prometheus_rule_group_iterations_missed_total`.
- Enable/query `prometheus_rule_evaluation_duration_seconds` per group.
- Profile rule complexity: high-cardinality `sum by`, many joins, subqueries.

**Resolution:**
- Split rule groups; increase `evaluation_interval` for non-critical rules.
- Reduce label cardinality in `sum by` clauses.
- Pre-aggregate at scrape time or use fewer, coarser recording rules.
- Shard rules across multiple Prometheus instances.

**Cascading effects:** Missed evaluations → stale recorded metrics → dependent alerts use outdated data → delayed or missed firings; dashboards show lagging values.

---

### 5. `absent()` vs `absent_over_time()`

| Function | Behavior |
|----------|----------|
| `absent(metric)` | Returns 1 if the metric is **missing at evaluation time** (instant). |
| `absent_over_time(metric[5m])` | Returns 1 if the metric had **no samples in the entire range**. |

**Target down:** Scrape fails → metric may disappear → both can fire depending on staleness and whether the series is removed vs marked stale.

**Metric stops scraped but target up:** Exporter stops emitting a specific metric → `absent()` fires immediately at eval time; `absent_over_time()` requires sustained absence over the window.

**Production strategy:**
```promql
# Target/job completely missing
absent(up{job="payment-api"})

# Metric not reported for sustained period
absent_over_time(custom_heartbeat{service="payment"}[10m])
```

Use `up == 0` for scrape failures; use `absent*()` for "expected metric never arrived" (exporter bugs, misconfiguration).

---

### 6. `label_replace()` at query time vs relabeling at ingestion

**Query-time (`label_replace()`):**
- Re-executed on every query; CPU cost scales with query frequency and series scanned.
- No storage savings; cardinality unchanged in TSDB.
- Good for ad-hoc dashboards, one-off normalization.

**Ingestion-time (`relabel_configs` / `metric_relabel_configs`):**
- Applied once per scrape; normalized labels stored permanently.
- Reduces duplicate series if normalization merges labels; can drop bad labels before storage.
- Preferred for production: recording rules, alerts, and dashboards all benefit.

```promql
# Query-time (expensive at scale)
label_replace(metric, "env", "production", "environment", "prod")
```

---

### 7. `count_over_time()` on gauges — misleading anomaly detection

**Why misleading:** `count_over_time()` counts **samples in range**, not value changes. A gauge scraped every 15s will always return ~20 for `[5m]` regardless of value — it measures scrape density, not anomalies.

**Alternatives:**
```promql
# Deviation from rolling mean (stddev)
abs(metric - avg_over_time(metric[1h])) > 2 * stddev_over_time(metric[1h])

# Rate of change
deriv(metric[30m])  # or delta() for gauges

# Z-score style with recording rules
(metric - avg_over_time(metric[1h])) / stddev_over_time(metric[1h]) > 3

# Compare to same time yesterday
metric / metric offset 1d > 1.5
```

For production anomaly detection, use recording rules, `predict_linear()` for trends, or external systems (ML, seasonal decomposition).

---

### 8. Thundering herd — simultaneous error spikes

```promql
# Count services with elevated error rate in same window
count(
  (
    sum by (service) (rate(http_errors_total[2m]))
    /
    sum by (service) (rate(http_requests_total[2m]))
  ) > 0.1
) > 5

# Stddev across services — many spiking at once
stddev(
  sum by (service) (rate(http_errors_total[2m]))
  /
  sum by (service) (rate(http_requests_total[2m]))
) > 0.05
```

Alert when many services exceed threshold simultaneously within 2 minutes — indicates systemic failure (dependency, network, deployment) not single-service issue.

---

### 9. `topk()` alert flapping

**Stabilization techniques:**
- Use recording rules to compute rank over longer window before `topk()`.
- Alert on aggregate: `count(metric > threshold) > N` instead of membership in top-k.
- Increase `for:` duration.
- Use `topk()` with longer `rate()` window: `topk(5, sum by (pod) (rate(errors[15m])))`.
- Consider `bottomk()` for "worst offenders" with minimum sample threshold.

```promql
# Stable: count offenders above threshold
count(
  sum by (instance) (rate(errors_total[10m])) > 100
) >= 3
```

---

### 10. `sum by()` vs `sum without()` and cardinality

- **`sum by (label1, label2)`** — aggregates preserving only listed labels; output cardinality = cardinality of those labels.
- **`sum without (label1)`** — drops listed labels, keeps all others; output cardinality can explode if high-cardinality labels remain.

**Performance:** `sum without (instance, pod)` on a metric with 10k pods retains pod-level cardinality. `sum by (service)` collapses to ~100 series. Choice matters when remaining label set is large — affects index lookups and memory during query.

Use `sum by` when you know the desired aggregation dimensions; use `sum without` only when explicitly excluding known high-card labels.

---

## Exporters & Instrumentation

### 11. Unbounded label cardinality (50k+ series from user IDs)

**Identify:**
- `prometheus_tsdb_head_series` spike; `topk(10, count by (__name__)({__name__=~".+"}))`.
- `promtool tsdb analyze` or Prometheus UI → Status → TSDB Status.
- Query: `count by (__name__) ({__name__="http_requests_total"})` then `count by (user_id) (...)`.

**Impact:** Each unique label combo = new series ≈ ~1-3 KB head memory + index overhead; 50k series manageable, millions cause OOM.

**Remediation:**
- Drop `user_id` via `metric_relabel_configs` or fix application instrumentation.
- Replace with bounded labels: `user_tier`, `tenant_id` (low cardinality).
- Use logs/traces for per-user debugging, not metrics.
- Add CI linting (prometheus-rules linter, cardinality limits in client libs).

---

### 12. `node_exporter` stale filesystem metrics

**Debug checklist:**
1. Check scrape errors: `up{job="node"}` and target status in UI.
2. Verify mount points: exporter may skip inaccessible mounts; check logs for `collector.filesystem` errors.
3. Stale NFS/CIFS mounts block `statfs` — exclude with `--collector.filesystem.mount-points-exclude`.
4. Compare `node_filesystem_avail_bytes` timestamp vs `time() - node_filesystem_avail_bytes`.
5. Check `scrape_timeout` vs slow filesystem stats on busy hosts.
6. Verify no cgroup/namespace isolation preventing mount visibility.
7. Restart exporter; check for hung syscalls (`strace` in extreme cases).

---

### 13. Go histogram buckets at 10k RPS

**Design:**
- Use default buckets only if latency SLO is unknown; customize around SLO (e.g., 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s).
- Add fine-grained buckets below p99 target (e.g., 50ms, 75ms, 100ms if SLO is 200ms).
- Avoid excessive buckets (>20) — each bucket × label set = cardinality.
- Use native histograms (Prometheus 2.40+) if available for better granularity with lower cardinality.
- Consider `Summary` with quantiles only if you need exact client-side quantiles and accept non-aggregatable metrics.

```go
prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "http_request_duration_seconds",
    Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
})
```

---

### 14. `blackbox_exporter` intermittent HTTPS failures

**Possible causes:**
- TLS: expired cert, SNI mismatch, cipher mismatch, cert chain incomplete.
- DNS: intermittent resolution, slow DNS, wrong resolver in pod.
- Network: packet loss, firewall, connection timeout vs `scrape_timeout`.
- Application: returns 503 only on probe path; rate limiting probe IP.

**Differentiation:**
```bash
# TLS
openssl s_client -connect host:443 -servername host

# DNS
dig host; kubectl run -it --rm debug --image=busybox -- nslookup host

# Network
blackbox_exporter probe modules: icmp vs tcp vs http
```
Compare `probe_ssl_earliest_cert_expiry`, `probe_dns_lookup_time_seconds`, `probe_http_duration_seconds` phase labels (`connect`, `tls`, `processing`).

---

### 15. `kube-state-metrics` 4GB memory (500 nodes, 5k pods)

**Strategies:**
- Disable unneeded collectors via `--resources` flag (limit to pods, deployments, nodes needed for SLOs).
- Use sharded kube-state-metrics (one per cluster or namespace subset).
- Increase scrape interval for KSM (not pod metrics).
- Upgrade KSM version (significant memory improvements in v2.x).
- Use `metric_allowlist` / `metric_relabel_configs` to drop unused metrics.
- Consider metrics-server + cAdvisor for resource metrics instead of duplicating via KSM.

---

### 16. Pushgateway anti-pattern

**Anti-pattern:** Long-lived jobs push to Pushgateway; metrics persist after job completes → stale, misleading "current" values. Multiple instances overwrite each other without grouping keys.

**Real scenario:** CronJob pushes `batch_last_success` once; job fails but metric remains "success" for days. Alert on stale batch never fires.

**Resolution:** Use Pushgateway only for short-lived batch jobs with `job` + `instance` grouping labels; use `push_delete` on completion; prefer pull-based scraping via Kubernetes pods even for jobs (sidecar or Service per job run). For service-level metrics, instrument the application directly.

---

### 17. JMX exporter inconsistent Kafka metric names across versions

- Maintain version-specific scrape configs or relabel rules mapping old names → canonical names.
- Use `metric_relabel_configs` to rewrite `__name__`.
- Create recording rules producing unified metric names.
- Standardize on one Kafka version per cluster where possible.
- Use OpenTelemetry collector as translation layer between versions.

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'kafka_controller_KafkaController_ActiveControllerCount'
    target_label: __name__
    replacement: 'kafka_controller_active_count'
```

---

### 18. Legacy app monitoring — sidecar vs proxy vs log-based

| Approach | Pros | Cons |
|----------|------|------|
| **Sidecar exporter** | Native K8s pattern; per-pod metrics; no app change | Extra container resources; must expose port |
| **Proxy exporter** | Centralized; one exporter serves many apps | Single point of failure; network path complexity |
| **Log-based (Promtail → Loki/metrics)** | No code change; rich event data | Higher latency; parsing fragility; not true real-time metrics |

**Kubernetes:** Sidecar preferred for per-instance metrics; proxy for shared legacy socket protocols; logs when only audit trail exists.

---

### 19. `mysqld_exporter` performance degradation

**Diagnose:** Enable `performance_schema`; compare MySQL `SHOW PROCESSLIST` during scrape; check `scrape_duration_seconds` for mysqld job.

**Tune:**
- Increase `scrape_interval` (60s–300s for global status).
- Disable expensive collectors: `--no-collect.info_schema.innodb_metrics`, `--collect.info_schema.tables=false`.
- Use custom queries with minimal columns.
- Run exporter on replica, not primary.
- Set shorter `scrape_timeout`.

---

### 20. Custom business metrics in microservices

- Define naming convention: `orders_processed_total`, `payment_failures_total` (counter); use consistent label keys (`service`, `region`, `payment_method` — bounded values only).
- Instrument at domain boundaries (order service, payment gateway).
- Use OpenTelemetry semantic conventions where applicable.
- Recording rules for cross-service KPIs.
- Cardinality review in PR process; never label with `order_id`.
- Document SLIs/SLOs tied to each metric.

---

## Federation & Remote Write

### 21. Hierarchical federation — global missing regional metrics

**Debug:**
1. Verify regional Prometheus is up and scraping: `up` on regional instance.
2. Test `/federate?match[]={__name__=~"up"}` from global to regional endpoint.
3. Check network/firewall between global and regional.
4. Verify `honor_labels` and `external_labels` don't cause collisions/drops.
5. Check federation scrape job `scrape_timeout` — large match sets timeout.
6. Review regional `federation` scrape errors in target status.
7. Confirm `match[]` selectors include missing metrics.
8. Check for clock skew causing out-of-order rejection on global.

---

### 22. Remote_write backpressure — WAL growth

**Monitor:**
- `prometheus_remote_storage_samples_pending`
- `prometheus_remote_storage_samples_failed_total`
- `prometheus_remote_storage_shard_capacity`
- `prometheus_wal_wal_storage_size_bytes`
- `prometheus_tsdb_wal_corruptions_total`

**Remediation (no data loss):**
- Scale remote endpoint (Thanos Receive, Cortex ingesters).
- Tune `queue_config`: increase `capacity`, `max_shards`, `max_samples_per_send`.
- Add network bandwidth; reduce sample rate via `write_relabel_configs` filtering.
- Temporarily increase local retention/disk; never delete WAL manually while running.
- Shard to multiple remote endpoints.

---

### 23. Federation vs remote_write for 20 instances → Thanos/Cortex

| | Federation | remote_write |
|---|------------|--------------|
| **Model** | Pull aggregated metrics up | Push all/raw samples |
| **Failure modes** | Timeout, incomplete match[], single point of pull failure | Backpressure, WAL growth, shard failures |
| **Ops complexity** | Simple for few metrics; painful at scale | More config but standard at scale |
| **Data** | Often downsampled/aggregated | Full fidelity in long-term store |

**Recommendation:** remote_write to Thanos/Cortex for 20 instances; federation only for selective up-sampling of already-aggregated metrics.

---

### 24. Tuning `queue_config` for Thanos Receive write amplification

```yaml
remote_write:
  - url: http://thanos-receive:19291/api/v1/receive
    queue_config:
      capacity: 10000
      max_shards: 50
      min_shards: 5
      max_samples_per_send: 5000
      batch_send_deadline: 5s
```

- Increase `max_shards` when `samples_pending` is chronically high.
- Increase `max_samples_per_send` for fewer, larger batches (reduces HTTP overhead).
- `capacity` buffers bursts; monitor for memory trade-off.
- Write amplification often from duplicate labels or over-sharding; dedupe at receiver.

---

### 25. WAL replay and remote_write after crash

On restart, Prometheus replays WAL into memory and re-sends samples to remote_write. Samples already acknowledged by remote may be **re-sent** (at-least-once delivery). Receivers need deduplication (Thanos `replica` label, Cortex HA tracker).

**Data loss scenarios:**
- WAL corrupted and truncated → samples between last checkpoint and crash lost locally and remotely if not yet sent.
- Remote endpoint down during extended outage + WAL truncated by retention → loss.
- `--storage.tsdb.retention` too short before recovery.

---

### 26. `/federate` endpoint timing out

**Causes:** Too broad `match[]`; high cardinality pull; network latency; regional Prometheus overloaded; `scrape_timeout` too low.

**Fix:** Narrow `match[]` to required metrics; use recording rules on source for pre-aggregation; increase timeout; federate from dedicated read replica; switch to remote_write for large metric sets.

---

### 27. Remote_write to multiple backends

- Each `remote_write` block is independent; partial failure affects only that queue.
- Monitor per-queue pending/failed metrics.
- `write_relabel_configs` per endpoint for different metric subsets.
- No cross-backend consistency guarantee — treat as separate pipelines.
- Use external_labels (`cluster`, `replica`) for dedup at each backend.

---

### 28. `out of order samples` in HA setup

Two Prometheus replicas scrape same targets with slightly different timestamps → samples arrive out of order at centralized receiver.

**Fix:** Configure receiver deduplication (Thanos: `replica` external label + `--receive.replication-factor`; Cortex: HA dedupe). Align scrape configs; use consistent `external_labels`. Avoid writing same series from unsynchronized sources.

---

### 29. Metric filtering in remote_write

```yaml
remote_write:
  - url: https://cloud-backend/api/v1/write
    write_relabel_configs:
      - source_labels: [__name__]
        regex: '(critical_slo_.*|up|ALERTS|prometheus_.*)'
        action: keep
```

Maintain allowlist of SLO-critical metrics; use separate full-fidelity remote_write to Thanos for internal use. Document dropped metrics; test alerts still fire.

---

### 30. Migrating federation → remote_write

| Aspect | Federation | remote_write |
|--------|------------|--------------|
| **Freshness** | Limited by federation scrape interval | Near real-time streaming |
| **Query latency** | Query global (aggregated only) | Query Thanos/Cortex (full history) |
| **Failure isolation** | Global depends on regional availability for pull | Regional buffers in WAL; decoupled |

Run both in parallel during migration; validate metric parity; cut federation once remote_write stable.

---

## High Availability & Clustering

### 31. HA replicas — alert fires on one but not the other

Configure Alertmanager with **identical** `external_labels` on both Prometheus except `replica` label. Alertmanager groups by alert labels **excluding** `replica` for deduplication.

```yaml
# prometheus.yml (both replicas)
global:
  external_labels:
    cluster: prod
    replica: $(POD_NAME)  # unique per replica

# alertmanager.yml
route:
  group_by: ['alertname', 'cluster', 'service']
```

Use Thanos/Cortex dedup or Alertmanager clustering so duplicate alerts from HA pair collapse to one notification.

---

### 32. Split-brain in Prometheus HA

**Split-brain:** Network partition separates Alertmanager instances → two quorums think they're primary → duplicate or dropped notifications.

**Gossip protocol:** Alertmanager cluster uses memberlist for state sync (silences, notifications). Helps converge when partition heals.

**Limitations:** During partition, both sides may send alerts; silences may not propagate; no strong consistency guarantee. Mitigate with odd number of AM instances (3+), proper `cluster.peer` configuration, and load balancer health checks.

---

### 33. HA pair 128GB RAM — capacity planning

**Methodology:**
- Measure `prometheus_tsdb_head_series`, bytes per sample, retention, query load.
- Formula approximation: `memory ≈ head_series × 1-3 KB + query overhead + WAL buffer`.
- Options: reduce retention; scale out (federation/sharding); remote_write to Thanos/VictoriaMetrics; drop high-card metrics; increase scrape interval; use downsampling in long-term store.

---

### 34. Prometheus OOMKilled — linear memory growth over 6 hours

**Likely causes:** Cardinality explosion (new deployment with bad labels); WAL replay; large queries caching; memory leak (rare, check version).

**Diagnose:**
- `prometheus_tsdb_head_series` over time.
- `process_resident_memory_bytes`.
- Recent deploys; `topk(20, count by (__name__)({__name__=~".+"}))`.
- Check for cardinality bombs in newest metrics.

**Fix:** Drop/relabel bad metrics; restart with retention limit; increase memory limit temporarily; fix instrumentation.

---

### 35. HA surviving AZ failure (AWS/GCP/Azure)

- Prometheus replicas in **different AZs** (anti-affinity).
- Alertmanager cluster (3 nodes) across AZs.
- Thanos/Cortex with multi-AZ object storage (S3 cross-AZ).
- `external_labels: {replica}` for dedup.
- DNS/load balancer health checks.
- Runbooks for single-AZ degradation; synthetic alerts to detect AZ-wide scrape failures.

---

### 36. Thanos vs Cortex vs VictoriaMetrics (multi-tenant, multi-region)

| | Thanos | Cortex/Mimir | VictoriaMetrics |
|---|--------|--------------|-----------------|
| **Model** | Object storage, sidecar/query | Centralized ingesters | Single/binary cluster |
| **Multi-tenant** | Limited native; label-based | Strong native | Enterprise/limits |
| **Ops** | Moderate; S3-centric | Complex at scale | Simpler deployment |
| **Cost** | S3 cheap at PB scale | Ingester cost | Very efficient compression |
| **Query** | PromQL via Querier | PromQL | MetricsQL/PromQL |

Choose Thanos for cloud-native S3 retention; Mimir for Grafana stack multi-tenancy; VictoriaMetrics for cost-efficient high cardinality.

---

### 37. Zero-downtime Prometheus upgrades

- Run HA pair; upgrade one replica at a time.
- Verify WAL compatibility in release notes (major versions may need migration).
- `/-/reload` for config; rolling restart for binary.
- Pre-apply recording/alert rules to secondary first.
- Monitor `ALERTS` and `prometheus_rule_evaluation_failures_total` during rollout.
- Keep Alertmanager up throughout; notifications continue from surviving replica.

---

### 38. Instance 30 minutes behind after network recovery

Prometheus replays WAL, catches up scrapes, may burst remote_write. During catch-up: alerts may evaluate on stale data; dashboards show gaps then catch-up spikes in `rate()`. Alertmanager may fire late. Mitigate with `for:` clauses, marking replica unhealthy in LB until caught up (`prometheus_tsdb_head_min_time` lag check).

---

### 39. Architecture for 10k-node K8s, 1M active series, sub-second queries

- **Sharding:** Multiple Prometheus per cluster/namespace/team (Prometheus Operator sharding).
- **Thanos Query** for federated queries across shards.
- **Recording rules** on each shard for dashboard queries.
- **SSD/local storage**; adequate CPU for query engine.
- **Avoid** querying raw high-cardinality metrics globally.
- **Cortex/Mimir or VictoriaMetrics** if single-shard limits exceeded.
- Target <100ms via pre-aggregation; sub-second at 1M series requires aggressive recording rules and query limits.

---

### 40. Configuration management at 50+ Prometheus instances

- GitOps (ArgoCD/Flux) with Prometheus Operator or config-reloader.
- Helm charts with values per environment/team.
- Jsonnet/Tanka or Kustomize overlays for scrape/retention differences.
- Centralized rule repo with CI validation (`promtool check rules`).
- Consul/etcd for dynamic fragments; avoid manual UI edits.

---

## Cardinality Issues & Performance

### 41. Memory doubled in 24h — investigation with TSDB metrics

1. `prometheus_tsdb_head_series` — trend over 24h.
2. `prometheus_tsdb_symbol_table_size_bytes`, `prometheus_tsdb_head_chunks`.
3. `topk(20, count by (__name__)({__name__=~".+"}))` — which metrics grew.
4. `count by (label) (metric)` for suspect metrics.
5. Correlate with deployments (`changes(process_start_time_seconds[1h])`).
6. `promtool tsdb analyze` offline if needed.
7. Check new `scrape_configs` or relaxed relabel rules.

---

### 42. `request_id` label — 10M series — immediate remediation

**Immediate:**
- Add `metric_relabel_configs` to drop `request_id` label or entire metric.
- `/-/reload` or rolling restart.
- Scale memory temporarily if still responsive.
- Block deploy pipeline producing bad metrics.

**Long-term:** Client library guardrails; PR review; `prometheus-limit` admission; document "allowed labels"; use exemplars/traces for request-level detail.

---

### 43. TSDB chunks and encoding

Samples grouped into **chunks** (~120 samples or 2h). Encoding types: XOR float, histogram-specific. Affects compression ratio and decode speed during queries.

**`--storage.tsdb.min-block-duration`:** Minimum time range per block (default 2h). Lower = more blocks, more compaction overhead; higher = larger blocks, longer compaction cycles. Tune for write-heavy vs query-heavy workloads; typically leave default unless compaction causes scrape delays.

---

### 44. High `prometheus_engine_query_duration_seconds`

**Identify:** Enable query logging (`--enable-feature=auto-gomaxprocs` won't help — use `query_log_file`). Check slow queries in logs; Grafana query inspector.

**Optimize:**
- Recording rules for dashboard queries.
- Reduce range window; avoid `count_over_time` on high-cardinality.
- `sum by` fewer dimensions.
- Increase `query.timeout` only as last resort.
- Block expensive queries via `query.max_samples`.

---

### 45. High-cardinality labels and inverted index

**Inverted index:** Maps `label=value` → series IDs. High cardinality = large index, slow postings list intersection.

**Ingestion:** Every new series = index entry + head chunk allocation — slows scrapes.
**Query:** More series to scan/decode even with selective matchers — slows queries disproportionately.

---

### 46. 45-second compaction causing scrape delays

**Causes:** Large blocks; slow disk (HDD); high series count; concurrent compactions; insufficient CPU.

**Tune:**
- Use SSD/NVMe.
- Increase `--storage.tsdb.max-block-duration` carefully.
- Scale CPU; reduce cardinality.
- `--storage.tsdb.wal-compression`.
- Consider VictoriaMetrics/Thanos for offload if local compaction can't keep up.

---

### 47. Cardinality governance process

- **Policy:** Approved labels, naming conventions, no IDs in labels.
- **Tooling:** `prometheus-exporter-exporter`, cardinality exporter, `topk` dashboards, CI `promtool` + custom linter.
- **Enforcement:** Admission webhook blocking bad metrics; chargeback for high-card teams; automatic relabel drops with audit log.
- **Education:** Office hours, instrumentation guides.
- **Budgets:** Per-team series limits with alerts on `head_series` by `team` label.

---

### 48. Istio millions of series — selective reduction

- `telemetry.v1alpha1.Prometheus` or mesh config to reduce labels.
- Drop `source_canonical_revision`, `destination_principal`, etc.
- `metric_relabel_configs` on scrape.
- Use **ambient** or reduced telemetry mode.
- Recording rules for golden signals only: request rate, errors, duration by service (not pod × path × response_code × ...).

---

### 49. Active vs head vs total series

| Metric | Meaning |
|--------|---------|
| **Head series** (`prometheus_tsdb_head_series`) | Series in memory-mapped head block — primary OOM predictor. |
| **Active series** | Series receiving samples in last 1-2 scrape intervals (conceptual; use head for ops). |
| **Total series** (compacted blocks + head) | On-disk series count; affects compaction and disk, less RAM than head. |

OOM correlates with **head series** and chunk memory in head, not historical blocks.

---

### 50. `metric_relabel_configs` drop risks

**Risks:** Drop metric still referenced in alert/recording rule → `absent()` false positives or missed alerts; break dashboards.

**Safety:** Run `promtool test rules` in CI; maintain allowlist; dry-run with `action: labeldrop` logging; document all drops; grep rules repo for metric name before dropping; use `honor_labels` carefully.

---

## Alertmanager & Alerting Strategy

### 51. Flapping alert every 5 minutes

**Alertmanager:** Increase `group_wait`, `group_interval`; tune `repeat_interval`.
**PromQL:** Longer `rate()` window `[10m]`; add `for: 10m`; use `avg_over_time` smoothing.
```promql
avg_over_time(
  (rate(errors[5m]) / rate(requests[5m]))[15m:5m]
) > 0.05
```
Avoid threshold too close to normal operating variance.

---

### 52. Debug routing tree (15 routes) — wrong receiver

- Use `amtool config routes test` with sample alert labels.
- `amtool alert add alertname=TestFire ...` to inject test alert.
- Enable `--log.level=debug` temporarily on test instance.
- Trace `match`, `match_re`, `continue`, child routes in order.
- Document label requirements per route (`team`, `severity`, `namespace`).

---

### 53. Multi-tier alerting for payment system

```yaml
route:
  receiver: default
  routes:
    - match: { severity: critical }
      receiver: pagerduty-oncall
      continue: true
    - match: { severity: warning }
      receiver: slack-payments
    - match: { severity: info }
      receiver: email-daily-digest
      active_time_intervals: [business-hours]
```

Escalation: warning → critical via `repeat_interval` + secondary route. Business hours via `time_intervals` in Alertmanager 0.24+. Payment-specific: PCI scope routes to restricted channels.

---

### 54. Alertmanager lost quorum — incident response

**Response:** Failover to backup notification path (manual bridge, secondary AM cluster); scale/restart AM pods; fix network; verify `cluster.status` API; drain problematic node.

**Prevention:** 3+ AM instances across AZs; monitor `alertmanager_cluster_members` and `alertmanager_notifications_failed_total`; regular DR drills.

---

### 55. Alert correlation — suppress downstream

Use `inhibit_rules`:
```yaml
inhibit_rules:
  - source_matchers: [alertname="DatabaseDown"]
    target_matchers: [alertname=~"HighErrorRate|SlowQueries"]
    equal: ['cluster']
```

Root cause alert inhibits dependent alerts with same `cluster` label.

---

### 56. `for: 5m` but issue resolves in 3 minutes

Trade-off: shorter `for` = faster detection but more noise. Options: `for: 2m` with stricter threshold; multi-window burn rate (short + long); `keep_firing_for` to avoid resolve noise; accept that brief incidents may not page (document as known gap for transient blips).

---

### 57. Misconfigured `inhibit_rules` silencing critical alert

**Example:** `source_matchers: [severity="warning"]` too broad inhibits `severity="critical"` child alerts because `equal: ['instance']` matches any warning on same instance — including unrelated warning during outage.

**Fix:** Narrow source matchers to specific root cause alertnames; use `equal: ['alertname', 'cluster']` precisely; test with `amtool`.

---

### 58. Routing by namespace, team, time of day

```yaml
routes:
  - match_re: { namespace: "team-payments-.*" }
    receiver: slack-payments
    routes:
      - match: { severity: critical }
        receiver: pagerduty-payments
  - match_re: { namespace: "team-platform-.*" }
    receiver: slack-platform
  - receiver: slack-default
    active_time_intervals: [business-hours]
```

Use `team` label from Kubernetes annotations via relabeling.

---

### 59. Deduplication across Prometheus, Datadog, synthetic

- Single on-call tool as source of truth (PagerDuty/Opsgenie event orchestration).
- Correlate via common incident ID; dedupe keys: `service + alertname + timestamp window`.
- Route non-Prometheus to same AM via webhook adapters.
- Policy: one system primary per SLI; others supplementary.

---

### 60. Programmatic silences for 200 microservices maintenance

- Generate silences via API/CLI (`amtool silence add`) from maintenance ticket system.
- Use label matchers: `{service=~"svc1|svc2|...", maintenance="true"}`.
- Terraform/Operator for AlertmanagerConfig CRDs.
- Auto-expire silences with `endsAt`; audit in AM UI.
- Silences on `alertname` + `namespace` rather than per-pod.

---

## Recording Rules & Rule Management

### 61. Recording rule differs 15% from instant query

**Causes:**
- Different evaluation times (rule at interval boundary vs ad-hoc query time).
- Rule uses different range window than dashboard.
- Stale rule output from missed evaluation.
- `sum by` label mismatch.
- Counter reset handled differently if windows differ.
- Rule aggregates at scrape interval; instant query uses fresher data.

Align windows and labels; compare at same `@` timestamp.

---

### 62. 500 rules exceeding 15s evaluation interval

- `prometheus_rule_group_last_duration_seconds` per group.
- Split groups by domain; stagger `evaluation_interval`.
- Optimize expensive rules (reduce cardinality in `sum by`).
- Remove duplicate/redundant rules.
- Increase interval for non-critical groups.
- Distribute rules across sharded Prometheus instances.

---

### 63. Recording rule dependency chains

Rules can reference other recorded metrics: `level1` → `level2` → `alert`. **Circular dependency** causes evaluation failure (undefined order).

**Detection:** `promtool check rules`; document dependency graph in CI.

**Failure:** If `level1` fails, dependent rules produce no data → alerts may not fire (fail silent) or fire on `absent()`.

**Fix:** Order groups so dependencies evaluate first; avoid cycles; use separate rule files with explicit group ordering.

---

### 64. Recording rule strategy balancing performance and overhead

- Record only metrics queried >N times/day or used in multiple alerts.
- Co-locate rules with scrape shard owning the data.
- Version control + CI validation.
- Naming convention: `level:metric:operations` (e.g., `service:http_requests:rate5m`).
- Review quarterly; delete unused rules.

---

### 65. Recording rule label change — metric collision during rollout

**Problem:** Old and new rule output same `__name__` with different label sets → duplicate or conflicting series.

**Safe migration:**
1. Create new rule with new name or `record:` prefix version (`:v2`).
2. Update alerts/dashboards to new metric.
3. Remove old rule after rollout complete.
4. Or use separate rule group, deploy atomically in single config version.

---

## Kubernetes Monitoring

### 66. KSM pod phase metrics 5-minute lag

**Diagnose:**
1. Compare KSM metric timestamp vs API: `kubectl get pods -w` vs metric change delay.
2. Check KSM logs for LIST/WATCH errors, `kube-state-metrics_list_total`.
3. API server latency: `apiserver_request_duration_seconds`.
4. Scrape interval on KSM job vs actual `scrape_duration_seconds`.
5. KSM CPU throttling / memory pressure.
6. Large cluster — KSM resync period.

If scrape is 1m but lag is 5m → KSM/API issue. If scrape is 5m → config issue.

---

### 67. RBAC blocking pod scrape in namespace

**Required resources:**
- `ServiceAccount` for Prometheus
- `ClusterRole` or `Role` with `get`, `list`, `watch` on `pods`, `services`, `endpoints`
- `ClusterRoleBinding` / `RoleBinding`

**Debug:**
```bash
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:prometheus -n target-ns
kubectl describe pod prometheus-0 -n monitoring  # check scrape errors
kubectl get endpoints -n target-ns
```

For PodMonitor/ServiceMonitor, Prometheus Operator generates RBAC — verify `prometheus.spec.serviceAccountName` and associated `ClusterRole`.

---

### 68. Stateful app (Cassandra) — pod identity in metrics

- Use **stable labels**: `statefulset.kubernetes.io/pod-name`, custom `cassandra_cluster` label via operator.
- Avoid relying on pod IP in alerts.
- Relabel `instance` to `pod` name.
- Recording rules aggregate by `cluster` + `rack` not `pod` for cluster-level alerts.
- Persistent volume metrics tied to PVC name, not pod.

---

### 69. Prometheus Operator multi-tenant

- One `Prometheus` CR per team/tenant with `serviceMonitorNamespaceSelector`, `podMonitorNamespaceSelector`.
- `AlertmanagerConfig` per tenant with `alertmanagerConfigNamespaceSelector`.
- Resource quotas per namespace; separate storage class/retention in CR spec.
- NetworkPolicy isolating Prometheus pods per tenant.

---

### 70. HPA custom metrics erratic scaling

**Debug pipeline:**
1. Verify metric exists in Prometheus: query `custom_metric`.
2. Check adapter logs: `kubectl logs -n custom-metrics deployment/custom-metrics-apiserver`.
3. Verify adapter Prometheus query matches intended aggregation (rate vs gauge).
4. Check HPA: `kubectl describe hpa` — events, metric value jumps.
5. Align scrape interval with HPA sync period (default 15s); smooth with recording rules.
6. Avoid high-cardinality labels in custom metric (HPA needs single value).

---

### 71. Control plane monitoring on managed K8s (EKS/GKE/AKS)

- Use cloud provider metrics (CloudWatch, Google Cloud Monitoring) for API server, etcd.
- Enable managed cluster monitoring endpoints where exposed.
- **kube-apiserver** via in-cluster scrape of `default/kubernetes` Service (limited on some providers).
- **etcd:** often not accessible on EKS — use provider metrics.
- **Scheduler/controller-manager:** scrape managed endpoints or use `ksm` + provider SLIs.
- Prometheus Operator `kube-prometheus-stack` includes available components.

---

### 72. Network policies and CNI performance

- **CNI metrics:** Cilium (`cilium_*`), Calico (`felix_*`), kube-proxy `iptables` sync metrics.
- **Network policy:** Cilium Hubble metrics; count of dropped packets by policy.
- Exporters: `cilium-agent` metrics ServiceMonitor.
- Alert on policy drops spike, DNS latency, conntrack table exhaustion (`node_nf_conntrack_entries`).

---

### 73. Prometheus evicted from nodes

```yaml
# PodDisruptionBudget
minAvailable: 1
# resources
requests: { memory: "8Gi", cpu: "2" }
limits: { memory: "16Gi" }
# affinity
nodeAffinity: prefer dedicated observability node pool
podAntiAffinity: spread replicas
priorityClassName: system-cluster-critical
```

Avoid scheduling on preemptible nodes; monitor kubelet eviction metrics.

---

### 74. Ephemeral Jobs and CronJobs

**Challenges:** Pod dies before scrape; Pushgateway anti-pattern; inconsistent series.

**Solutions:**
- Short `scrape_interval` + `track_timestamps_staleness`.
- Prometheus annotations on Job template; scrape during run.
- Pushgateway with job completion delete (only if necessary).
- Log-based metrics for batch success/failure.
- Kube-state-metrics `kube_job_status_*` for job-level SLIs without in-job instrumentation.

---

### 75. Cross-cluster monitoring

- One Thanos Query / Mimir frontend with `external_labels: {cluster}` on each Prometheus.
- Consistent metric naming via recording rules and label contract.
- Central Alertmanager with routes by `cluster` label.
- GitOps for identical rule bases with cluster-specific values overlays.

---

## Scaling Challenges & Production Scenarios

### 76. 2M samples/sec, 30s query latency

**Architecture:**
- Shard ingestion (Cortex/Mimir, VictoriaMetrics cluster, or Thanos with many receivers).
- remote_write from lightweight agents.
- Downsampling for historical queries.
- Separate query path from ingest path.

**Migration:** Dual-write remote_write; validate; cut over queries to new backend; keep local Prometheus as short-term buffer during transition.

---

### 77. Black Friday disk exhaustion — post-incident

- Set `--storage.tsdb.retention.size` in addition to time retention.
- Alert on `disk_usage` and `predict_linear` for disk.
- remote_write to durable storage; shorten local retention.
- Pre-scale disk; cardinality limits before events.
- Recording rules to drop raw high-card metrics post-aggregation.
- Runbooks for emergency `metric_relabel` drops.

---

### 78. Serverless (Lambda/Cloud Functions) monitoring

- **OpenTelemetry** export to ADOT/collector → remote_write.
- **CloudWatch exporter** → Prometheus (limited).
- **Lambda extension** for cold start, duration, errors.
- Cannot scrape — must **push** model; avoid Pushgateway for high-volume — use direct remote_write.
- Track `aws_lambda_*` via YACE/cloudwatch exporter for control plane metrics.

---

### 79. 50 new services/week — auto-discovery

- Prometheus Operator `ServiceMonitor` with label selector `monitoring=enabled`.
- Kubernetes SD with annotation-based opt-in (`prometheus.io/scrape`).
- Consul/Consul Connect service discovery.
- GitOps generating ServiceMonitor from service catalog.
- Default alert rules matching `service=~".+"` with templated dashboards.

---

### 80. Cost allocation for Prometheus infrastructure

- Label all metrics with `team`, `cost_center` via relabeling.
- Track per-team series: `count by (team) ({__name__=~".+"})`.
- Chargeback model: storage (samples × retention) + query CPU.
- Dashboards showing top cardinality contributors by team.
- Budget alerts when team exceeds series quota.

---

## Basic Questions

### 1. What is Prometheus?

Open-source monitoring system and time-series database. Pull-based metric collection, PromQL query language, built-in alerting. Solves **dimensional data model** problem: collect metrics with labels, query flexibly, alert on PromQL without proprietary agents.

### 2. Counter, gauge, histogram, summary

| Type | Behavior | Example |
|------|----------|---------|
| **Counter** | Monotonically increasing (resets on restart) | `http_requests_total` |
| **Gauge** | Can go up/down | `memory_usage_bytes` |
| **Histogram** | Observations in configurable buckets + `_sum`, `_count` | Request latency distribution |
| **Summary** | Pre-computed quantiles at client (not aggregatable across instances) | Client-side p99 |

### 3. Scrape interval

Time between metric collections from a target (default 1m). Prometheus **pulls** HTTP GET on `/metrics` endpoint at configured interval.

### 4. Default port and config file

- Port: **9090**
- Config: **`prometheus.yml`** (default path `/etc/prometheus/prometheus.yml` in packages)

### 5. Exporter — three examples

Program translating third-party metrics to Prometheus format: **node_exporter** (host), **blackbox_exporter** (probes), **mysqld_exporter** (MySQL).

### 6. Static scrape target

```yaml
scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['localhost:8080']
```

### 7. PromQL

Prometheus Query Language — used to select, aggregate, and compute time-series data for dashboards and alerts.

### 8. Labels

Key-value pairs identifying dimensions of a metric (`method="GET"`, `status="500"`). Enable filtering, aggregation, and multi-dimensional analysis. Part of unique time series identity.

### 9. Alertmanager purpose

Receives alerts from Prometheus, deduplicates, groups, routes, silences, and sends notifications (PagerDuty, Slack, email). Handles on-call workflow.

### 10. Check scrape success

Prometheus UI → Status → Targets; or query `up{job="myapp"} == 1`; API `GET /api/v1/targets`.

### 11. `up` metric

`1` if last scrape succeeded, `0` if failed. Auto-generated per target.

### 12. `rate()` vs `irate()`

- **`rate()`:** Average per-second rate over range; handles counter resets; smooth.
- **`irate()`:** Instant rate from last two samples; reactive; noisy.

### 13. Total HTTP request count

```promql
sum(http_requests_total{service="myapi"})
# or rate over time:
sum(increase(http_requests_total{service="myapi"}[1h]))
```

### 14. Recording rule

Pre-computes PromQL expression on interval; stores as new time series. Improves dashboard/alert query performance and consistency.

### 15. Pushgateway

Allows short-lived/batch jobs to push metrics. Use only for service-level batch jobs that cannot be scraped. Avoid for per-instance service metrics.

### 16. Alert rule definition

```yaml
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: rate(errors_total[5m]) / rate(requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: High error rate detected
```

### 17. `for` clause

Alert must be **active** (expr true) for this duration before firing. Reduces transient noise.

### 18. `prometheus_tsdb_head_series`

Number of series in TSDB head block — key indicator of memory pressure and cardinality.

### 19. Reload config without restart

`curl -X POST http://localhost:9090/-/reload` (requires `--web.enable-lifecycle`) or `SIGHUP`.

### 20. Service discovery

Automatic target discovery. Examples: **kubernetes_sd_config**, **ec2_sd_config**, consul, DNS, file_sd.

### 21. `relabel_configs` purpose

Modify label set **before** scrape (discovered targets): keep/drop targets, set labels, rewrite addresses.

### 22. Local storage and default retention

TSDB on local disk. Default retention: **15 days** (`--storage.tsdb.retention.time=15d`).

### 23. `sum()` vs `count()`

- **`sum()`:** Adds values across series.
- **`count()`:** Counts number of series (or non-null values in `count()` aggregation).

### 24. Filter by label

```promql
http_requests_total{job="api", status="500"}
```

### 25. Federation

Prometheus scrapes aggregated metrics from another Prometheus `/federate` endpoint. Use for hierarchical views (per-DC → global) with limited metric sets.

### 26. `--web.enable-lifecycle`

Enables `/-/reload` and shutdown endpoints for config reload without full restart.

### 27. Custom application metrics

Import Prometheus client library; register counters/gauges/histograms; expose `/metrics` HTTP endpoint; configure Prometheus to scrape it.

### 28. `node_exporter`

Exports hardware/OS metrics: CPU, memory, disk, filesystem, network, load, etc.

### 29. Alertmanager → Slack

```yaml
receivers:
  - name: slack
    slack_configs:
      - api_url: '<webhook-url>'
        channel: '#alerts'
route:
  receiver: slack
```

### 30. `increase()` vs `rate()`

- **`rate()`:** Per-second rate (float).
- **`increase()`:** Total increase over range (counter reset adjusted). `increase(m[1h])` ≈ `rate(m[1h]) * 3600`.

### 31. `absent()`

Returns 1 if metric absent at eval time; empty if present. Used for "metric should exist" alerts.

### 32. Top 5 services by request rate

```promql
topk(5, sum by (service) (rate(http_requests_total[5m])))
```

### 33. Data model and time series identity

Metric name + label set (sorted labels) = unique time series. Timestamped float64 samples.

### 34. `kube-state-metrics`

Exposes Kubernetes object state metrics: pod phase, deployment replicas, node conditions — from API server, not node agent.

### 35. Pod scrape via annotations

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

Prometheus kubernetes_sd with relabel to honor annotations.

### 36. `metric_relabel_configs` vs `relabel_configs`

- **`relabel_configs`:** Before scrape — target discovery labels.
- **`metric_relabel_configs`:** After scrape — sample/metric labels (can drop metrics before storage).

### 37. Health and readiness

- `GET /-/healthy` — liveness
- `GET /-/ready` — ready to serve traffic (WAL replay done)

### 38. `honor_labels`

If true, prefer scraped metric's `job`/`instance` labels over synthetic ones from service discovery.

### 39. Basic email route

```yaml
receivers:
  - name: email
    email_configs:
      - to: 'oncall@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
route:
  receiver: email
```

### 40. `blackbox_exporter`

Probes endpoints (HTTP, TCP, ICMP, DNS) and exports probe success/latency metrics.

### 41. HTTP API query

```bash
curl 'http://localhost:9090/api/v1/query?query=up'
curl 'http://localhost:9090/api/v1/query_range?query=rate(http_requests_total[5m])&start=...&end=...&step=15s'
```

### 42. `scrape_timeout`

Max time for single scrape (default 10s). Exceeded → scrape fails, `up=0`, may miss samples for that interval.

### 43. Basic auth for scrape

```yaml
scrape_configs:
  - job_name: secure
    basic_auth:
      username: user
      password: secret
    static_configs:
      - targets: ['app:8080']
```

### 44. `external_labels`

Labels added to all exported/time-series data (remote_write, federation). Used for cluster/replica identification: `cluster: prod`, `replica: a`.

### 45. Error rate percentage

```promql
100 *
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

---

## Intermediate Questions

### 1. EC2 service discovery

```yaml
scrape_configs:
  - job_name: ec2
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Monitoring]
        regex: enabled
        action: keep
      - source_labels: [__meta_ec2_private_ip]
        target_label: __address__
        replacement: '${1}:9100'
```

### 2. Scrape pods with specific annotation in namespace

```yaml
kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [my-namespace]
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    regex: "true"
    action: keep
```

### 3. Multi-tier alerting with routing tree

Nested `routes` with `match` on `severity`: `critical` → PagerDuty, `warning` → Slack, `info` → email. Use `continue: true` for multi-receiver escalation. See Alertmanager Q53.

### 4. WAL and crash recovery

WAF append-only log of incoming samples. On crash, replay WAL into head. Ensures minimal data loss between checkpoints. Increases restart time proportional to WAL size.

### 5. remote_write to Thanos Receive

```yaml
remote_write:
  - url: http://thanos-receive:19291/api/v1/receive
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
```

### 6. Prometheus HA + Alertmanager dedup

Two Prometheus with same config, different `replica` external_label. Both send to shared Alertmanager cluster. AM deduplicates by grouping labels excluding replica.

### 7. `label_replace()` normalization

```promql
label_replace(
  metric{environment="prod"},
  "env", "production", "environment", ".*"
)
# Or normalize mixed values:
label_replace(metric, "env", "production", "env", "prod|production|prd")
```

### 8. `histogram_quantile()` accuracy

Interpolates within bucket containing quantile. Accuracy limited by bucket boundaries — p99 between 1s and 5s buckets is approximate. Use fine buckets near SLO; sum buckets before quantile across instances.

### 9. Recording rules for dashboards

```yaml
groups:
  - name: aggregations
    interval: 30s
    rules:
      - record: service:http_requests:rate5m
        expr: sum by (service) (rate(http_requests_total[5m]))
```

### 10. K8s control plane monitoring

`kube-prometheus-stack`: scrape `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet`, `coredns`. Use in-cluster endpoints and appropriate TLS config.

### 11. Inhibition rules

```yaml
inhibit_rules:
  - source_matchers: [alertname="NodeDown"]
    target_matchers: [alertname="PodNotReady"]
    equal: ['node']
```

### 12. Retention size vs time

- `--storage.tsdb.retention.time=30d` — delete blocks older than 30 days.
- `--storage.tsdb.retention.size=50GB` — delete oldest blocks when size exceeded.
Both can apply — whichever limit hit first wins.

### 13. Counter resets and `rate()`

`rate()` detects counter decrease (reset) and excludes that interval's artifact from calculation, computing only over valid samples within range.

### 14. TLS client cert scrape

```yaml
scrape_configs:
  - job_name: mtls
    scheme: https
    tls_config:
      cert_file: /etc/prometheus/certs/client.crt
      key_file: /etc/prometheus/certs/client.key
      ca_file: /etc/prometheus/certs/ca.crt
```

### 15. `predict_linear()` disk alert

```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*3600) < 0
```
Predicts disk full in 4 hours based on 6h trend.

### 16. Alertmanager grouping

```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
```
Groups related alerts into single notification during widespread failure.

### 17. Stateful app on K8s with PVs

Monitor via StatefulSet pod labels; scrape per-pod exporter; alert on PVC usage `kubelet_volume_stats_*`; use stable network identity for dashboards.

### 18. Subquery syntax

```promql
max_over_time(
  rate(http_requests_total[5m])[1h:5m]
)
```
Inner range selector evaluated at subquery step over outer range. Useful for rolling max of rates.

### 19. Federation for specific metrics

```yaml
scrape_configs:
  - job_name: federate-apps
    honor_labels: true
    metrics_path: /federate
    params:
      match[]:
        - '{job=~"app-.*"}'
        - 'up'
    static_configs:
      - targets: ['app-prometheus:9090']
```

### 20. Drop high-cardinality metrics

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'expensive_metric_.*'
    action: drop
  - regex: 'user_id'
    action: labeldrop
```

### 21. ServiceMonitor and PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
```

PodMonitor selects pods directly (no Service required).

### 22. `repeat_interval` and `group_interval`

- **`group_interval`:** Wait before sending updates for existing group.
- **`repeat_interval`:** Re-notify for ongoing alert (default 4h). Increase to reduce noise; decrease for escalation reminders.

### 23. Staleness handling

If target down >5 scrape intervals (~5 min default), samples marked **stale** (special NaN marker). Disappears from `rate()` calculations. Dashboards show gaps — correct behavior.

### 24. Consul service discovery

```yaml
consul_sd_configs:
  - server: consul:8500
    services: [myapp]
relabel_configs:
  - source_labels: [__meta_consul_tags]
    regex: '.*,metrics,.*'
    action: keep
```

### 25. Business KPI metrics

Instrument at service with bounded labels (`product_line`, `region`); use counters for events; recording rules for rates; SLO dashboards; document definitions.

### 26. Query active alerts via API

```bash
curl http://localhost:9090/api/v1/alerts
curl http://alertmanager:9093/api/v2/alerts
```

### 27. Ingress controller monitoring

Scrape nginx-ingress-controller or traefik metrics: `nginx_ingress_controller_requests`, `request_duration_seconds`, error rates. Recording rules for per-ingress SLIs.

### 28. `offset` modifier

```promql
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 1w)
```
Compare current to same period last week for anomaly detection.

### 29. Istio sidecar scrape

```yaml
kubernetes_sd_configs:
  - role: pod
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_container_name]
    regex: istio-proxy
    action: keep
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:15020
    target_label: __address__
```

Or use Istio telemetry v2 Prometheus merge.

### 30. SLO monitoring — multi-window burn rate

```yaml
# Recording rules for SLI
- record: sli:request_success_rate5m
  expr: sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Multi-burn-rate alert (Google SRE)
- alert: SLOBurnRateHigh
  expr: |
    sli:request_success_rate5m < (1 - 14.4 * (1 - 0.999))
    and sli:request_success_rate1h < (1 - 14.4 * (1 - 0.999))
  for: 2m
```

### 31. Query concurrency and timeout

```bash
--query.max-concurrency=20
--query.timeout=2m
```
Prevent query overload; reject long-running queries on heavily loaded instances.

### 32. HPA custom metrics from Prometheus

Deploy prometheus-adapter with config mapping Prometheus query to custom.metrics.k8s.io API. HPA references metric name; adapter queries Prometheus.

### 33. Alertmanager HA gossip

Alertmanager instances cluster via gossip (memberlist) syncing notification state and silences. Requires UDP/TCP between peers (`--cluster.peer`).

### 34. Self-signed TLS scrape

```yaml
tls_config:
  insecure_skip_verify: true  # dev only
  # or
  ca_file: /etc/prometheus/ca.pem
```

### 35. `group_left` and `group_right`

Many-to-one join — attach labels from one side:

```promql
sum by (pod) (rate(container_cpu[5m]))
* on (pod) group_left (version)
kube_pod_info
```

`group_left` adds labels from right (one) to left (many).

### 36. Auto-discover new namespaces

```yaml
# Prometheus Operator
namespaceSelector:
  matchLabels:
    monitoring: enabled
```
Or cluster-wide watch with namespace label relabeling; teams opt-in via namespace label.

### 37. Node resource pressure alerts

```promql
# Memory pressure
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) < 0.1

# Disk pressure
(node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes) < 0.1

# CPU
(1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 0.9
```

### 38. `@` modifier

```promql
rate(http_requests_total[5m] @ 1609746000)
```
Evaluate query at specific Unix timestamp — useful for incident replay comparing to normal.

### 39. Multi-cluster central Prometheus

Each cluster Prometheus remote_writes to Thanos with `cluster` external_label. Central Thanos Query for cross-cluster PromQL. Avoid single Prometheus scraping all clusters directly.

### 40. Chunk encoding and compaction tuning

```bash
--storage.tsdb.wal-compression
--storage.tsdb.min-block-duration=2h
--storage.tsdb.max-block-duration=36h
```
For write-heavy: ensure adequate disk IOPS; consider lower compaction frequency; monitor `prometheus_tsdb_compactions_failed_total`.

---

## Advanced Questions

### 1. 50k nodes, 500M active series architecture

- **Sharding:** 100+ Prometheus/Cortex tenants by region/team; ~5M series per shard max practical.
- **Ingest:** remote_write to Cortex/Mimir or VictoriaMetrics cluster.
- **Storage:** S3/GCS object storage (Thanos/Mimir blocks); 2-5 year retention with downsampling.
- **Query:** Thanos Query Frontend with query splitting; cache (Memcached/Redis).
- **Federation:** Only for global aggregates, not raw series.
- **Governance:** Hard cardinality limits; automatic enforcement.

### 2. Multi-tenant monitoring platform

- **Cortex/Mimir** native multi-tenancy (`X-Scope-OrgID`) or **Thanos** with label-based isolation.
- Separate Alertmanager routes per tenant.
- Grafana org per tenant; RBAC on data source.
- Resource quotas per tenant; billing by samples ingested.

### 3. Global Thanos — petabyte, 2-year retention

- Prometheus → Thanos Sidecar → S3 blocks.
- Thanos Compactor: downsample 5m, 1h resolutions.
- Thanos Store Gateway serves historical blocks.
- Query layer with dedup for HA `replica` labels.
- Index cache; store gateway sharding by time range.

### 4. Cardinality governance automation

- Cardinality exporter tracking per-metric/team series counts.
- Alerts on budget breach; CI blocks new high-card labels.
- Auto-generate relabel drops with approval workflow.
- Dashboard per team showing burn-down against quota.

### 5. Prometheus HA failure modes in network partition

| Component | Partition effect |
|-----------|------------------|
| Prometheus replicas | Both scrape independently; duplicate samples to remote storage without dedup |
| Alertmanager | Split-brain notifications; silences diverge |
| Queries | Each replica may return different results |
| remote_write | Both buffer; catch-up on heal may cause OOO samples |

Mitigation: odd AM nodes, replica dedup, quorum-based routing.

### 6. Statistical anomaly detection

- `stddev_over_time`, `deriv`, `predict_linear` for baselines.
- Seasonal: compare `offset 1w` / `offset 1d`.
- Recording rules for z-scores.
- External: Prometheus → streaming → ML (Prophet, ADTK) → alert webhook.
- Avoid pure static thresholds for cyclical metrics.

### 7. Zero-trust Prometheus

- mTLS on all scrapes (`tls_config` both sides).
- NetworkPolicy restricting scrape sources.
- OAuth/OIDC on Prometheus/Thanos Query UI.
- RBAC in Grafana; audit logs for config changes.
- Encrypt S3 with KMS; TLS for remote_write.

### 8. Survive complete region failure — no data loss, <5min RTO

- Multi-region remote_write with replicated object storage (S3 CRR).
- Prometheus/Thanos in 3 regions; global Query routes to healthy region.
- Alertmanager in each region with global routing fallback.
- DNS failover for query/alert endpoints.
- RTO: pre-provisioned standby region; automated failover runbooks.

### 9. 80% storage cost reduction, 2-year retention, sub-second queries

- VictoriaMetrics or Mimir with aggressive compression.
- Downsampling: raw 15d, 5m resolution 90d, 1h resolution 2y.
- Drop high-cardinality raw labels after aggregation via recording rules + relabel.
- Query recent data from local/SSD; historical from object storage with cache.

### 10. Istio 10k pods, 50M series

- Disable detailed telemetry; use **ambient mesh** metrics.
- Allowlist metric names in scrape relabel.
- Aggregate at sidecar to service level via recording rules.
- Separate Prometheus shard per namespace.
- Use eBPF (Cilium) for L4 metrics instead of full L7 per-request.

### 11. Capacity planning — 30-day prediction

```promql
predict_linear(
  sum(node_filesystem_avail_bytes{mountpoint="/"})[7d], 30*24*3600
) < 0
```
Fleet-wide recording rules; trend CPU/memory/request growth; integrate with capacity ticketing system.

### 12. 10M samples/sec, sub-100ms queries

- VictoriaMetrics cluster or Cortex with 50+ ingesters.
- Separate read path; query only recording rules / downsampled data for dashboards.
- Ingest path horizontally scaled; no single Prometheus.
- Query cache; pre-computed aggregates; limit ad-hoc PromQL.

### 13. SLO platform for 500 services

- Standardized SLI instrumentation library.
- Recording rules: `sli:ratio_rate5m`, `sli:ratio_rate1h`.
- Alert templates with multi-window burn rates.
- Grafana SLO dashboards; error budget policies automated from metrics.

### 14. Federation — global views + local high-resolution

- Local Prometheus: full resolution, short retention.
- Recording rules export aggregates only via federation or remote_write filtered metrics.
- Global Thanos queries local sidecars for recent; object storage for history.
- No duplicate raw storage globally.

### 15. Regulated financial services deployment

- Encryption at rest (TSDB disk, S3 KMS) and in transit (TLS everywhere).
- RBAC, MFA, audit logs for all config access.
- Data residency per region; retention policies per regulation.
- Immutable audit trail for alert changes; SOC2-compliant logging.

### 16. ML-dynamic alert thresholds

- Export metrics to ML pipeline (seasonal decomposition).
- Push dynamic thresholds via recording rules updated by batch job (careful with staleness).
- Or use Grafana ML / third-party AIOps integrating with Alertmanager webhooks.
- Fallback static thresholds for safety.

### 17. Auto-scaling Prometheus horizontally

- **Cortex/Mimir/VictoriaMetrics** native sharding with auto-scaling ingesters on ingestion rate.
- Hash ring rebalancing (Cortex) with token migration.
- Kubernetes HPA on ingester CPU / samples rate.
- Zero data loss: replication factor 3, WAL on ingesters, graceful shutdown.

### 18. Multi-cloud unified observability

- OpenTelemetry collectors in each cloud → central remote_write.
- `cloud` external_label; unified naming convention document.
- Thanos global Query across cloud regions.
- Correlate via `trace_id` in exemplars/logs, not metric labels.

### 19. Real-time streaming + local storage

- remote_write to Kafka (via adapter/custom receiver).
- Local Prometheus for ad-hoc queries.
- Consumers: Flink, analytics, ML.
- Thanos for historical; Kafka for real-time fan-out.

### 20. Incident correlation system

- Alertmanager grouping by `cluster`, `region` during incident.
- Recording rules for cross-service error rate correlation.
- Exemplars + traces (Tempo/Jaeger) linked from Prometheus alerts.
- Event correlation: deploy webhooks + metric change detection (`changes()` around deploy time).

### 21. Metric backfill from legacy systems

- Import via `promtool tsdb create-blocks-from` or remote_write adapter.
- Backfill into separate Thanos blocks; compact separately.
- Don't mix with live ingesters — use dedicated backfill job.
- Validate label schema before import.

### 22. Change detection correlated with deploys

- Annotate deploys via Alertmanager or Grafana annotations API.
- Exporter for CD pipeline events as metrics (`deploy_timestamp`).
- Alert on metric deviation within 15m of `changes(deploy_info[15m]) > 0`.
- Integrate ArgoCD/Flux metrics.

### 23. 100-cluster K8s observability

- One Prometheus (or agent) per cluster → Thanos with `cluster` label.
- Hierarchy: cluster → region → global Query.
- Consistent rules via GitOps monorepo with Kustomize per cluster.
- Central Alertmanager; routes by `cluster` and `team`.

### 24. Reliability engineering platform (MTTR/MTTF/availability)

- Incident start/end from Alertmanager webhooks → time series or database.
- Recording rules: `availability = 1 - (downtime_seconds / period_seconds)`.
- MTTR = `alert_resolved_time - alert_fired_time` tracked in incident tool; export as metrics.
- Grafana dashboards per service from standardized labels.

### 25. Sub-second freshness + 5-year retention for trading

- Local Prometheus/VM with sub-second scrape for real-time path.
- remote_write to low-latency backend (VictoriaMetrics) for dashboards.
- Archive to S3 with Thanos; 5y downsampled for compliance.
- Separate "hot" (seconds) and "cold" (years) query paths; never query 5y data for trading dashboards.

---

## Rapid-Fire Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | HTTP method for scrape? | **GET** |
| 2 | Default scrape interval? | **1 minute** (`1m`) |
| 3 | Metric type for HTTP request count? | **Counter** |
| 4 | What `rate(http_requests_total[5m])` calculates? | Per-second average rate of increase over 5 minutes |
| 5 | Default `node_exporter` port? | **9100** |
| 6 | Purpose of `__address__` label? | Target address (host:port) to scrape from service discovery |
| 7 | How to silence an alert? | Alertmanager UI/API/`amtool silence add` |
| 8 | `sum()` vs `avg()`? | `sum` adds values; `avg` computes mean across series |
| 9 | Target "UNKNOWN" state? | Scrape not yet attempted or SD issue; target state indeterminate |
| 10 | Stale marker in TSDB? | Special value marking series absent ~5 min; excluded from functions |
| 11 | Check cardinality of a metric? | `count({__name__="metric_name"})` or TSDB status API |
| 12 | `honor_timestamps` purpose? | Use scraped sample timestamps instead of Prometheus receive time |
| 13 | `topk(5, metric)` returns? | 5 series with highest sample value at eval time |
| 14 | `/-/healthy` endpoint? | Liveness check — returns 200 if Prometheus running |
| 15 | Recording vs alerting rules? | Recording: precompute metrics; Alerting: fire alerts |
| 16 | `increase(counter[1h])`? | Total increase over 1 hour (reset-adjusted) |
| 17 | `__metrics_path__` label purpose? | HTTP path for metrics (default `/metrics`) |
| 18 | Query all series with label value? | `{label_name="value"}` |
| 19 | Default rules evaluation interval? | **1 minute** (same as default `evaluation_interval`) |
| 20 | `vector(0)` returns? | Instant vector with single element 0 (no labels) |
| 21 | `keep_firing_for` purpose? | Keep alert firing briefly after expr goes false (reduce resolve flapping) |
| 22 | Check Prometheus version? | `prometheus --version` or `GET /api/v1/status/buildinfo` |
| 23 | `ALERTS` metric? | Time series indicating firing alerts (1=firing, 0=resolved) |
| 24 | `min_over_time(metric[1h])`? | Minimum sample value within past hour |
| 25 | `cluster` label in federation? | Identifies source Prometheus cluster for deduplication/aggregation |
| 26 | `absent(up{job="myapp"})` when target down? | Returns empty if `up` exists (even as 0); `absent` needs metric **missing entirely** — use `up == 0` for down target |
| 27 | List active targets via API? | `GET /api/v1/targets` |
| 28 | `label_replace()` vs `label_join()`? | `label_replace` renames/replaces one label; `label_join` concatenates multiple labels into one |
| 29 | `resets(counter[1h])`? | Number of counter resets in 1 hour |
| 30 | `--storage.tsdb.allow-overlapping-blocks`? | Allow compaction to produce overlapping blocks (migration/debug; not normal ops) |
| 31 | `quantile_over_time(0.99, metric[1h])`? | 99th percentile of gauge values over 1 hour |
| 32 | WAL replay status after restart? | Logs show "replay completed"; `prometheus_tsdb_data_replay_duration_seconds`; `/-/ready` returns 200 when done |
| 33 | `prometheus_rule_evaluation_failures_total`? | Count of failed rule evaluations — alert on increase |
| 34 | `clamp_min(metric, 0)`? | Sets samples below 0 to 0 |
| 35 | `send_resolved` in receivers? | Send notification when alert resolves (default true) |
| 36 | Active time series count? | `prometheus_tsdb_head_series` or `count({__name__=~".+"})` (expensive) |
| 37 | `delta(gauge[1h])`? | Difference between first and last value in range (gauge) |
| 38 | `--rules.alert.for-outage-tolerance`? | Restores `for` state after restart to avoid re-firing alerts post-outage |
| 39 | `sort_desc(topk(10, metric))`? | Top 10 series sorted descending by value |
| 40 | `prometheus_tsdb_compactions_total`? | Total TSDB compactions completed |
| 41 | `bool` modifier in comparison? | Returns 0/1 instead of filtering; keeps all series |
| 42 | `group_wait` purpose? | Initial wait to batch alerts in group before first notification |
| 43 | Check recording rule evaluation? | Query recorded metric exists; check `prometheus_rule_group_last_duration_seconds` |
| 44 | `time()` in PromQL? | Returns current evaluation timestamp (Unix seconds) |
| 45 | `--web.console.templates`? | Path to console template files for Prometheus UI consoles |
| 46 | `changes(gauge[1h])`? | Number of times value changed in 1 hour |
| 47 | `prometheus_notifications_dropped_total`? | Alerts dropped due to throttling/misconfiguration |
| 48 | Force TSDB compaction? | POST `http://localhost:9090/api/v1/admin/tsdb/delete_series` (destructive) or restart with tombstones; normal: automatic. Use `promtool tsdb compact` offline |
| 49 | `scalar(single_element_vector)`? | Converts single-element vector to scalar for scalar-vector math |
| 50 | `match[]` in federation endpoint? | Label selectors filtering which metrics to return in federation scrape |

---

*Generated as comprehensive reference answers for interview preparation and SRE study.*
