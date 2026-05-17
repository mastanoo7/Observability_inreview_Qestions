# Logstash — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## Pipeline Architecture & Bottlenecks

### 1. Throughput dropped from 100K to 10K eps with no config changes

- **Check node stats first:** `GET localhost:9600/_node/stats/pipelines?pretty` — compare `events.in`, `events.filtered`, `events.out`, `queue.events_count`, and per-plugin `duration_in_millis` / `events.out` ratios.
- **Identify the slow stage:** If `events.in` >> `events.out`, the bottleneck is filters or output; if input rate dropped, check upstream (Beats/Kafka/agent).
- **Worker saturation:** In `/_node/stats`, inspect `pipelines.<id>.workers` — if all workers show high `busy_time_in_millis`, CPU-bound filters are likely; if workers idle but queue grows, output is blocked.
- **JVM/GC pressure:** Check `jvm.mem.heap_used_percent` and GC time spikes; stop-the-world GC can collapse throughput without config changes.
- **Downstream dependency:** Elasticsearch bulk rejections, Kafka consumer lag, or disk I/O on persistent queue path often cause sudden drops.
- **Compare baselines:** Pull 24h metrics from Metricbeat/Prometheus (`logstash_*` indices) and correlate with deploys, ES yellow/red, or host CPU steal (noisy neighbor).

```yaml
# logstash.yml — enable API for diagnosis
api.http.host: "0.0.0.0"
api.http.port: 9600
xpack.monitoring.enabled: true
```

### 2. Pipeline workers, batch size, and batch delay interaction

- **Workers** (`pipeline.workers`, default = CPU cores): parallel filter/output execution threads; more workers help CPU-bound pipelines until contention on locks or output limits.
- **Batch size** (`pipeline.batch.size`, default 125): events collected per worker batch before filter chain runs; larger batches improve throughput but increase per-batch latency and memory.
- **Batch delay** (`pipeline.batch.delay`, default 50ms): max wait to fill a batch; lower delay reduces latency for sparse traffic; higher delay improves batching efficiency for bursty small events.
- **Large/complex events:** Fewer workers with moderate batch size (50–125) to limit memory per batch; increase heap; avoid heavy Ruby/aggregate in hot path.
- **Many small events:** Increase workers to core count × 1.5–2, batch size 500–2000, batch delay 10–25ms for higher eps at cost of ~tens of ms added latency.

```yaml
# logstash.yml
pipeline.workers: 16
pipeline.batch.size: 1000
pipeline.batch.delay: 20
```

### 3. Security events delayed by application logs

- **Split pipelines:** Dedicated high-priority pipeline for security (Syslog/Firewall/EDR) with its own workers and persistent queue; separate pipeline for app logs.
- **Pipeline-to-pipeline:** Security input → `pipeline` output to `sec-pipeline`; app logs → `app-pipeline` — isolates filter cost.
- **Separate instances:** Run security Logstash on dedicated nodes with guaranteed CPU/memory; never share workers with bulk app ingestion.
- **Kafka topic isolation:** `security-events` topic with dedicated consumer group and higher partition count; app logs on separate topics/clusters if needed.
- **Rate limiting app path:** Apply `throttle` or sampling on low-priority pipeline only; never throttle security.

```conf
# sec-pipeline.conf
input { pipeline { address => "security" } }
filter { /* lightweight security parsing only */ }
output { elasticsearch { index => "security-%{+YYYY.MM.dd}" } }

# main.conf — fan-in
input { kafka { topics => ["app-logs"] /* ... */ } }
output { pipeline { send_to => "app-pipeline" } }
input { kafka { topics => ["sec-logs"] /* ... */ } }
output { pipeline { send_to => "security" } }
```

### 4. Pipeline-to-pipeline communication

- **`pipeline` input/output** connect pipelines in-process via an internal queue (`address` on output, matching input).
- **Performance win:** Parse once in a lightweight ingress pipeline, fan out to specialized pipelines (enrichment vs. routing) without serializing to disk/network.
- **Use when:** Multi-tenant routing, priority isolation, or reusing a common normalization stage across many outputs.
- **Limitation:** Same Logstash JVM; does not replace horizontal scaling — use Kafka for cross-host fan-out.
- **vs. single pipeline:** Reduces head-of-line blocking when one filter branch is expensive; each sub-pipeline has independent worker pools.

```conf
output { pipeline { send_to => ["enrich", "archive"] } }
# enrich.conf
input { pipeline { address => "enrich" } }
```

### 5. 100% CPU on all worker threads

- **Per-plugin timing:** Enable slow log (`slowlog.threshold.warn` on filters) and check `/_node/stats/pipelines/<id>.plugins.filters[]` for highest `duration_in_millis / events.out`.
- **Common culprits:** Grok with catastrophic backtracking, `ruby` filter, `dns` filter, GeoIP on every event, `aggregate` with large maps.
- **Mitigations:** Replace grok with `dissect`/`kv` where possible; cache DNS/GeoIP; move enrichment to ingest node or ES ingest pipeline; sample non-critical fields.
- **Profile JVM:** `jstack` on Logstash PID during spike; correlate with hot filter.
- **Scale out:** If filters are inherently expensive, add Logstash nodes behind Kafka consumer group.

### 6. In-memory vs persistent queue

- **In-memory (default):** Fast, bounded by `queue.max_events` (default 0 = unlimited in mem — dangerous); events lost on crash or `kill -9` before output ACK.
- **Persistent queue (`queue.type: persisted`):** Writes to disk (`path.queue`); survives restarts; provides backpressure when output slow.
- **Data-loss scenario:** ES outage + in-memory queue → heap exhaustion → OOMKill or `queue.max_events` drop → thousands of events lost.
- **PQ fix:** Enable persistent queue sized for expected outage window; events buffer on disk until ES recovers.

```yaml
queue.type: persisted
path.queue: /var/lib/logstash/queue
queue.max_bytes: 10gb
```

### 7. Backpressure from slow Elasticsearch without drops or OOM

- **Enable persistent queue** sized for peak slowdown duration (`queue.max_bytes`).
- **Tune ES output:** Lower `workers`, increase `bulk_max_size` and `flush_interval` to reduce request count; set `retry_on_conflict` and reasonable `retry_max_interval`.
- **Flow control:** PQ blocks input when full — monitor `queue.events_count`; scale ES or add data nodes before queue fills.
- **Avoid unbounded memory:** Never use in-memory queue for high-volume ES-dependent pipelines.
- **Circuit breaker pattern:** Alert on queue depth; optionally pause non-critical inputs via external control.

```conf
output {
  elasticsearch {
    hosts => ["https://es:9200"]
    bulk_path => "/_bulk"
    manage_template => false
    retry_initial_interval => 2
    retry_max_interval => 64
  }
}
```

### 8. Multi-tenant routing with different retention

- **Identify tenant:** Extract `customer_id` / `tenant` in filter from topic, header, or log field.
- **Conditional outputs:** Route to index per tenant; use ILM policies via index templates per pattern `logs-%{tenant}-%{+YYYY.MM.dd}`.
- **Index templates:** Pre-create templates binding each tenant pattern to hot/warm/delete phases (7d vs 90d retention).
- **Isolation:** Separate Kafka topics per tier; dedicated pipelines for enterprise vs. standard SLAs.
- **Security:** Document-level or cluster isolation for regulated tenants (separate ES cluster).

```conf
filter { mutate { add_field => { "[@metadata][target_index]" => "logs-%{tenant}-%{+YYYY.MM.dd}" } } }
output {
  elasticsearch { index => "%{[@metadata][target_index]}" }
}
```

### 9. Events out of order for time-based aggregations

- **Root causes:** Multiple Logstash instances, clock skew, async buffering, `@timestamp` parsed incorrectly vs. ingestion time.
- **Use `date` filter** to parse event time into `@timestamp`; add `event.original_timestamp` for audit.
- **ES approach:** Prefer `data_stream` + `event.ingested` for ordering diagnostics; use `transform` or TSDB for strict ordering needs.
- **Kafka:** Single partition per ordering key (e.g., `host.name`) preserves order per key at cost of parallelism.
- **Trade-off:** Sorting buffers (external Flink/Spark or ES ingest with `sort` processor) add latency — accept seconds of disorder for logs, not for billing.

### 10. Pipeline monitoring and alerting

- **Built-in:** `xpack.monitoring` → ES, or scrape `/_node/stats` and `/_node/pipelines` via Metricbeat/Prometheus exporter.
- **Key metrics:** `events.out` rate, `queue.events_count`, `queue.capacity.queue_size_in_bytes`, plugin error counts, `reload_failures`.
- **Alerts:** Queue > 70% capacity (warning), > 90% (critical); `events.out` / `events.in` ratio < 0.95 sustained; DLQ growth; JVM heap > 85%.
- **Synthetic:** Inject canary events with known ID; verify arrival in ES within SLO.
- **Dashboards:** Grafana panels for eps in/out, lag, and per-pipeline breakdown.

---

## Queue Tuning & Persistent Queues

### 11. Persistent queue at 500GB not draining

- **Confirm output health:** ES cluster red? Auth failures? Mapping errors sending everything to DLQ while PQ still accepts input?
- **Check drain rate:** `queue.events_count` trend — if flat, workers may be blocked on output; if decreasing slowly, tune workers/bulk size.
- **Disk inode/full:** PQ uses page files under `path.queue`; verify disk not 100% and permissions intact.
- **Emergency:** Stop input (pause Beats/Kafka consumers), scale ES, add data nodes, or temporarily route to S3/Kafka backup output.
- **Resize cap:** Lower `queue.max_bytes` only after drain plan — dropping cap can block ingestion; expand disk first.
- **Root cause:** Often ES indexing rate < ingest rate during maintenance — plan maintenance with pre-scaled consumers and raised `queue.max_bytes`.

### 12. Persistent queue checkpoint mechanism

- **PQ structure:** Memory-mapped page files; `queue.page_capacity` (default 64mb) per page; head/tail acked pages freed.
- **Checkpoints:** `queue.checkpoint.writes` (default 1024) — after N written events, checkpoint records durable position.
- **Crash behavior:** Uncheckpointed events may be replayed → **at-least-once** delivery to outputs; downstream dedup via `fingerprint` + document `_id` if needed.
- **In-flight:** Events read from PQ but not yet ACKed by output plugin are reprocessed after crash.
- **Tuning:** Lower checkpoint interval = more durability overhead; higher = more replay on crash.

### 13. Tune PQ for 1-hour ES maintenance buffer

- **Size calculation:** `queue.max_bytes` ≥ peak_eps × avg_event_bytes × 3600 × 1.2 safety factor.
- **Example:** 50K eps × 2KB × 3600 ≈ 360GB — provision disk accordingly or reduce retention window with spill to S3.
- **`queue.page_capacity`:** Larger pages (128–256mb) reduce file count on high-volume systems.
- **`queue.checkpoint.writes`:** 4096–8192 for maintenance windows if replay cost is acceptable vs. checkpoint I/O.
- **Pre-maintenance:** Verify PQ empty-ish before maintenance; throttle inputs if approaching cap.

```yaml
queue.type: persisted
queue.max_bytes: 400gb
queue.page_capacity: 128mb
queue.checkpoint.writes: 4096
```

### 14. Persistent queue vs Kafka buffer

- **Logstash PQ:** Simpler ops, local disk bound, same JVM lifecycle, limited replay tooling, good for short ES outages.
- **Kafka:** Durable, replicated, replayable, cross-team consumers, handles multi-hour/day buffers, independent scaling of producers/consumers.
- **Choose PQ:** Single-team ELK, outages < few hours, want fewer moving parts.
- **Choose Kafka:** High volume, multiple consumers, long buffer, compliance replay, decouple ingest from processing.
- **Hybrid:** Kafka → Logstash → ES with small PQ as second-line defense.

### 15. Corrupted PQ after shutdown

- **Symptoms:** Logstash fails to start, PQ corruption errors in logs.
- **Recovery:** Stop Logstash; backup `path.queue` directory; delete corrupted page files only if support/runbook confirms — often safest is archive and **start fresh queue** after exporting what you can.
- **Data loss:** Uncheckpointed + corrupted pages are lost; replay from upstream (Kafka/Beats with stored offsets) if retention allows.
- **Prevention:** Graceful shutdown (`queue.drain: true`), UPS, avoid `kill -9`, use `queue.checkpoint.acks: true` (default).

### 16. Queue depth monitoring and alerts

- **Metrics:** `queue.events_count`, `queue.capacity.max_queue_size_in_bytes`, `queue.capacity.queue_size_in_bytes` from `/_node/stats`.
- **Alert thresholds:** Warning at 60% bytes, critical at 85%, page on sustained growth > 30 min.
- **Metricbeat module:** `logstash.queue.events_count` shipped to ES/Prometheus.
- **Runbook:** Link alerts to scale ES, add Logstash workers, or pause non-critical inputs.

### 17. `queue.drain` shutdown behavior

- **`queue.drain: true` (default):** On shutdown, Logstash finishes processing PQ events before exit — graceful, slower shutdown.
- **`queue.drain: false`:** Faster shutdown; in-flight PQ events remain for next start — use in K8s preStop with short termination grace only if next pod picks up same PVC.
- **K8s:** Set `terminationGracePeriodSeconds` high enough for drain; use same `path.queue` on PVC for StatefulSet.
- **Trade-off:** Long drain during deploys vs. risk of duplicate processing at output (mitigate with idempotent `_id`).

```yaml
queue.drain: true
```

---

## Parsing Failures & Grok Debugging

### 18. High `_grokparsefailure` rate

- **Sample failures:** `stdout { codec => rubydebug }` on tagged events or query ES for `tags:grokparsefailure` and pull `message` samples.
- **Iterate in Grok Debugger:** Test against 50+ real samples including edge cases (truncated lines, multiline).
- **Break patterns:** Use multiple grok with `break_on_match => true` or `if [field]` conditionals instead of one mega-pattern.
- **Fallback:** On failure, keep `message`, tag `grok_failed`, route to `failed-parsing` index for triage; don't drop.
- **Onboard:** `grok { tag_on_failure => ["_grokparsefailure"] match => { "message" => ["%{PATTERN}", "%{GREEDYDATA:unparsed}"] } }`

### 19. Custom grok for variable/optional/nested JSON in syslog

- **Stage parsing:** Syslog grok first → extract payload → `json` filter on payload field if JSON-in-syslog.
- **Optional sections:** Use `(...)?` non-capturing groups sparingly; prefer multiple specific patterns over one optional-heavy regex.
- **Variable length:** Use `%{GREEDYDATA}` or `NOTSPACE` terminators; avoid `.*` across full line.
- **Test harness:** CI job with `bin/logstash -t -f pipeline.conf` + fixture files; Grok Debugger for dev loop.
- **Version patterns:** `APP_LOG_v1`, `APP_LOG_v2` in pattern library.

### 20. Grok taking 500ms/event (backtracking)

- **Enable slow log:**

```conf
filter {
  grok {
    match => { "message" => "..." }
    slowlog_threshold => 10
  }
}
```

- **Find pattern:** Logs show which pattern exceeded threshold; test regex at regex101.com with catastrophic backtracking detection.
- **Fix:** Replace nested quantifiers `(.*)*`, use possessive/atoms, anchor with `^`, switch to `dissect` for fixed-delimiter logs.
- **Measure:** Compare `duration_in_millis` before/after in node stats.

### 21. Grok pattern library for 50 formats

- **Repo structure:** `patterns/<app>/` custom pattern files + `pipelines/<app>.conf` referenced via `patterns_dir`.
- **Version control:** Git PR review for regex changes; CODEOWNERS per application team.
- **Testing:** RSpec/logstash-filter-verifier with golden sample logs; CI fails on regression.
- **Deploy:** Config management (Ansible/K8s ConfigMap) with `config.reload.automatic`; canary instance first.
- **Catalog:** Internal docs mapping `service → pattern version → index`.

### 22. Log format changes without downtime

- **Dual patterns:** Run old + new grok with `break_on_match`; route unmatched to review index.
- **Feature flag:** `translate` or env var to enable new pattern path on subset of hosts.
- **Reload:** `config.reload.automatic: true` — push new conf, validate with `-t`, monitor `grokparsefailure` rate.
- **Coordination:** App team announces format change; SRE keeps old pattern 2 weeks for stragglers.

### 23. Grok works in Debugger, fails in production

- **Encoding:** UTF-8 BOM, Windows CRLF, binary junk — check `charset` on input codec.
- **Field mismatch:** Debugger tests raw string; production may use different field (`event.original` vs `message`).
- **Multiline:** Production merges stack traces; Debugger tests single lines — fix `codec => multiline`.
- **Pattern path:** Custom patterns not in `patterns_dir` on server.
- **ECS:** Field renamed by upstream agent — grok runs on wrong field order in pipeline.

### 24. Conditional grok for multiple formats

```conf
filter {
  if [service] == "api" {
    grok { match => { "message" => "%{API_LOG}" } }
  } else if [service] == "batch" {
    grok { match => { "message" => "%{BATCH_LOG}" } }
  } else {
    grok { match => { "message" => "%{GENERIC_LOG}" } }
  }
}
```

- **Performance:** Sequential `if` avoids running all patterns on every event — prefer `if` on cheap discriminator (topic, tag, first token) before grok.
- **Anti-pattern:** Chaining 10 grok filters without conditionals — every event runs every regex.

### 25. `dissect` vs grok

- **Dissect:** Delimiter-based, no regex backtracking, very fast for `timestamp|level|msg` fixed-order logs.
- **Limitations:** Fixed delimiter order, limited optional fields (`%{?skip}`), no regex power.
- **Use dissect when:** Apache/nginx access logs with stable separators, CSV-like, key-value with known keys.
- **Use grok when:** Irregular text, regex extraction, syslog PRI, embedded IPs in variable positions.

```conf
dissect {
  mapping => { "message" => "%{timestamp} %{+timestamp} %{level} %{msg}" }
}
```

---

## Throughput Optimization

### 26. Scale from 50K to 500K events per second

- **Architecture:** Kafka (or similar) as buffer; 10–50+ Logstash instances; ES tier sized for bulk indexing (30+ data nodes, dedicated ingest).
- **Pipeline:** Minimize filters; JSON `filter { json { source => message } }` beats grok; push parsing to Beats/Fluent Bit where possible.
- **Workers/batch:** `pipeline.workers: 32+`, `batch.size: 2000`, `batch.delay: 5` per instance after CPU profiling.
- **JVM:** Heap 8–16GB (`-Xms` = `-Xmx`), G1GC, `UseG1GC`, avoid CMS; leave 50% RAM for OS page cache (PQ).
- **Output:** Multiple ES output workers, `bulk_max_size => 5000`, dedicated coordinating nodes.
- **Hardware:** NVMe for PQ, 10G+ network, avoid oversubscribed VMs.

### 27. `aggregate` filter without memory leaks

- **Define `task_id`** narrowly — avoid high-cardinality keys (request_id OK per short window; never raw `message`).
- **Set `timeout` and `timeout_tags`** to expire stale state; use `push_previous_map_as_event => false` unless needed.
- **Map size limits:** Application-level cap; alert on aggregate map growth via custom `metrics` filter.
- **Alternative:** Stream correlation in Flink/Kafka Streams; ES transforms for batch correlation.
- **Isolate:** Run aggregate pipelines on dedicated instances with smaller heap and fewer workers.

### 28. DNS lookup bottleneck

- **Remove per-event DNS:** Use `mutate` to keep IP; enrich at query time in Kibana or ES `enrich` processor.
- **Conditional:** `if [need_hostname]` only for security tier events.
- **Cache:** External DNS cache (nscd) or pre-built IP→hostname dictionary via `translate` filter refreshed hourly.
- **Async/offline:** Batch DNS resolution job updating CMDB lookup table for `jdbc_streaming`/`translate`.
- **Disable reverse DNS** on inputs (`manage_template` / Beats `reverse_lookup: false`).

### 29. Event batching for Elasticsearch

- **Logstash side:** Larger `pipeline.batch.size` + ES output `bulk_max_size` (2000–10000) and `flush_interval` (1–5s).
- **Trade-off:** Higher bulk = better throughput, worse near-real-time visibility (acceptable for logs).
- **ES side:** Increase `thread_pool.write.queue_size`, proper shard count (20–50GB/shard), disable unnecessary replicas during bulk reindex.
- **Monitor:** Bulk rejection metrics, `indexing_pressure`, and Logstash output `duration_in_millis`.

### 30. GeoIP optimization

- **Conditional:** `if [client_ip]` and `[need_geo]` — skip internal RFC1918 via `cidr` filter first.
- **Cache:** GeoIP filter caches automatically; ensure MMDB on local SSD; refresh DB on schedule.
- **Sampling:** `if [@metadata][sample] == "true"` for 1% geo enrichment on high-volume CDN logs.
- **Pre-enrich:** Geo at edge (CloudFront/Fastly logs include country); drop GeoIP filter entirely.
- **Alternative:** ES ingest pipeline `geoip` on indexing path for unified enrichment.

### 31. JVM heap and GC tuning

- **Heap:** Typically 4–8GB for filter-heavy; up to 50% of container limit; never exceed 32GB (compressed oops).
- **Always set `-Xms` = `-Xmx`** to avoid resize pauses; use `LS_JAVA_OPTS` in `jvm.options`.
- **G1GC:** Default in modern Logstash; monitor `jvm.gc.collectors.old.collection_time_in_millis`.
- **Symptoms of under-heap:** Frequent GC, PQ spill thrashing; over-heap: long GC pauses, OOM on off-heap/direct buffers.
- **Off-heap:** PQ and Netty use direct memory — leave headroom beyond heap in container limits.

```bash
LS_JAVA_OPTS="-Xms8g -Xmx8g -XX:+UseG1GC"
```

### 32. Pipeline profiling beyond monitoring API

- **Slow log** per filter plugin with thresholds.
- **`_node/stats` plugin breakdown** for relative CPU share.
- **JFR/async-profiler** on JVM during load test.
- **logstash-filter-verifier** + benchmark harness with production sample files.
- **Distributed tracing:** Not native — inject timing fields with `ruby` filter for critical sections in POC.

---

## Scaling Logstash

### 33. Horizontal scale for 10x log volume

- **Stateless ingest:** Kafka with N consumers in one group → N Logstash instances (partitions ≥ instances).
- **Load balancing:** Beats → L4 LB → multiple Logstash :5044; or Kafka as sole buffer (preferred at scale).
- **No duplicate processing:** Kafka consumer group coordinates partitions; Beats use separate paths or Kafka in between.
- **Avoid:** File input on shared NFS with multiple readers (duplicates/gaps) — use Beats/Agent per host.
- **ES scaling:** Increase shards/coordinating nodes proportionally; use data streams + ILM.

### 34. Logstash on Kubernetes with HPA

- **StatefulSet + PVC** for `path.queue` when using PQ; one PVC per pod.
- **Scale-up:** Increase replicas; Kafka rebalances partitions to new consumers.
- **Scale-down:** `queue.drain: true`, `preStop` hook sleep, ensure consumer completes partition before kill.
- **HPA metrics:** Custom metric `kafka_consumer_lag` or `logstash_queue_fill_ratio` — not CPU alone.
- **Anti-pattern:** Deployment with emptyDir PQ — data loss on reschedule.

### 35. Stateful pipelines at scale

- **`aggregate` state** is JVM-local — does not survive scale-out; use external store (Redis, Kafka state store) or single-replica pipeline for correlation only.
- **Persistent queues** are per-node — cannot share PVC across replicas; each pod has own queue.
- **Mitigation:** Move stateful logic out of Logstash; keep Logstash stateless transform + route.

### 36. Blue-green pipeline deploys

- **Dual releases:** Blue/green Logstash Deployments consuming same Kafka group (only one active) OR separate consumer groups with offset management.
- **Safer pattern:** New consumer group on new version, validate output index, switch traffic, decommission old.
- **Config reload:** `SIGHUP` / automatic reload for minor changes; major changes need new image + coordinated cutover.
- **Validate:** `-config.test_and_exit` in init container before marking pod ready.

### 37. Uneven load across instances

- **Kafka:** Uneven partition sizes or hot keys — increase partitions, salt keys, monitor per-partition lag.
- **Beats:** Sticky connections to one node — use LB with consistent hash or funnel through Kafka.
- **Check:** Per-node `events.in` in monitoring; rebalance partitions with `kafka-consumer-groups --describe`.
- **Fix:** Add instances, reassign partitions, ensure `pipeline.workers` consistent across nodes.

### 38. Autoscaling on queue depth

- **Metric:** `queue.events_count / queue.capacity.max_queue_size_in_bytes` or Kafka `sum(consumer_lag)`.
- **Scale-up:** Lag > threshold 5 min → +1 replica (max cap).
- **Scale-down:** Lag near zero for 30+ min, cooldown 15 min, drain PQ first.
- **Safety:** `minReplicas` ≥ 2 for HA; max bounded by Kafka partition count.

---

## Kafka Integration

### 39. Kafka consumer lag growing, Logstash "healthy"

- **Triangulate:** Compare Kafka `records-lag-max` per partition vs Logstash `events.in` rate vs ES indexing rate.
- **If lag grows, LS in low:** Consumer paused, poll timeout, or network to Kafka — check consumer logs.
- **If LS in high, out low:** Logstash filter/output bottleneck — tune workers or ES.
- **If both low:** Producer slowdown or topic retention expiring — verify upstream volume.
- **Tune:** `consumer_threads`, `max_poll_records`, `fetch_max_bytes`, `session_timeout_ms`.

### 40. Consumer group rebalancing

- **On rebalance:** Partitions revoked and reassigned; in-flight batches may be reprocessed after rebalance (at-least-once).
- **Minimize:** Keep `max.poll.interval.ms` > worst-case batch processing time; stable consumer count; avoid frequent scaling.
- **Static membership:** `group.instance.id` (plugin-dependent) reduces unnecessary rebalance on restart.
- **Processing guarantee:** Enable ES `document_id` from `%{[@metadata][kafka][offset]}-%{[@metadata][kafka][partition]}` for idempotency during rebalance replay.

### 41. Exactly-once Logstash → ES via Kafka

- **Reality:** Logstash does not support Kafka transactions end-to-end; achieve **effective exactly-once** via idempotent writes.
- **Pattern:** `fingerprint` → ES `_id`; Kafka EOS on producer side + Logstash idempotent consumer offset commit after bulk success (manual/advanced).
- **Limitations:** Plugin versions, rebalance replay, aggregate filter side effects break strict EOS.
- **Better:** Kafka Streams/Flink for EOS; Logstash for near-EOS with dedup at ES.

### 42. Multiple Kafka topics, different schemas

```conf
input {
  kafka { topics => ["app-a", "app-b"] decorate_events => true }
}
filter {
  if [@metadata][kafka][topic] == "app-a" { json { source => message } }
  else if [@metadata][kafka][topic] == "app-b" { grok { /* ... */ } }
}
output {
  elasticsearch { index => "%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}" }
}
```

### 43. `max.poll.interval.ms` timeouts / kicked from group

- **Cause:** Filter processing exceeds `max.poll.interval.ms` (default 5m) — worker blocked on grok, JDBC, or ES.
- **Fix:** Increase `max_poll_interval_ms` in kafka input; reduce per-poll batch (`max_poll_records`); optimize filters; add consumers.
- **Also check:** GC pauses, thread deadlock, ES hang without timeout.
- **Diagnose:** Logstash logs `CommitFailedException` / rebalance messages correlated with slow plugin stats.

### 44. Partition strategy and parallelism

- **Partitions ≥ total pipeline workers across all consumers in group** for full parallelism (ideal: partitions = N × workers per instance).
- **Keying:** Partition by `host.name` or `trace_id` for ordering; random keys for even spread.
- **More partitions:** Higher parallelism, more ES bulk threads, more file handles — don't over-partition (target 10–50MB/s per partition).

### 45. Kafka input and output (stream processing)

- **Pattern:** Read topic A → transform → write topic B for downstream; dead letter to `topic-dlq`.
- **Failures:** On filter failure, `pipeline { send_to => dlq-pipeline }` or kafka output to DLQ topic with original metadata.
- **Reprocess:** Separate consumer on DLQ with fixed logic version; max retry counter in headers.
- **Avoid loops:** Tag `reprocessed=true`, skip after N attempts.

---

## Dead Letter Queues

### 46. DLQ filling rapidly

- **Enable DLQ:**

```yaml
dead_letter_queue.enable: true
path.dead_letter_queue: /var/lib/logstash/dlq
```

- **Monitor:** `dead_letter_queue.get` API or file size metrics; alert on growth rate.
- **Analyze:** `bin/logstash -f dlq_reader.conf` with `dead_letter_queue` input plugin; inspect `reason`, `plugin_type`, `plugin_id`.
- **Reprocess pipeline:** Read DLQ → fix mapping/transform → ES; track success/fail counts.
- **Prioritize:** Fix top failure reason (usually mapping or date parse) before bulk replay.

### 47. Logstash DLQ vs Kafka dead letter topic

- **Logstash DLQ:** Captures events that failed inside Logstash pipeline (output reject, exception); local files on disk.
- **Kafka DLT:** Broker-level sink for poison messages from consumers; better for multi-subscriber architectures.
- **Unified strategy:** All paths → normalized DLQ format in Kafka `dlq-v1` with `failure_reason`, `source_pipeline`, `original_topic` for single reprocessor.

### 48. Automatic DLQ reprocessing

- **Pipeline:** DLQ input → validate fix version field → conditional output; increment `[@metadata][retry_count]`.
- **Circuit:** Drop to manual queue when `retry_count > 3`; alert — never infinite loop.
- **Gate reprocessing:** Only after root cause deploy confirmed (mapping updated, grok fixed).
- **Idempotent `_id`:** Same fingerprint on replay prevents ES duplicates.

### 49. DLQ from Elasticsearch mapping conflicts

- **Diagnose:** DLQ reason shows `mapper_parsing_exception` or `illegal_argument_exception`; compare event field types vs index mapping.
- **Fix:** Dynamic mapping off — use composable index templates; `rename` conflicting fields (`data.count` → `data_count` via `de_dot`).
- **Reindex:** Replay DLQ to new index with corrected template; use `_reindex` for already-indexed bad docs if any.
- **Prevention:** ECS compatibility mode; schema registry for structured logs.

### 50. DLQ alerting and reporting

- **Metrics:** DLQ byte size, events/hour by `plugin_id` and `reason` (parsed from DLQ reader batch job).
- **Dashboard:** Top 10 failure reasons, trend, affected `service.name`.
- **Weekly report:** Auto-email from Elastic Watcher or Grafana on DLQ growth.
- **No manual inspection:** Alert only on new reason types (anomaly) or threshold breach.

---

## Filter Plugins & Data Enrichment

### 51. External database enrichment without bottleneck

- **Prefer `jdbc_static`** for periodic full-table load into memory vs per-event `jdbc_streaming` for high cardinality.
- **Connection pool:** Limit pool size; set query timeouts (5s); use read replicas.
- **Failure handling:** `jdbc_streaming` leaves field null on failure — tag `enrich_failed` and route to retry topic; never block pipeline indefinitely.
- **Cache:** Local `translate` YAML refreshed from DB hourly for reference data.
- **Scale:** Move enrichment to ES ingest `enrich` processor or pre-compute at source.

### 52. `jdbc_streaming` filter

```conf
filter {
  jdbc_streaming {
    jdbc_driver_library => "/path/postgresql.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://db:5432/app"
    jdbc_user => "${DB_USER}"
    jdbc_password => "${DB_PASS}"
    statement => "SELECT name, dept FROM users WHERE id = :id"
    parameters => { "id" => "user_id" }
    target => "user_info"
    cache_expiration => 300
    cache_size => 10000
  }
}
```

- **Pool:** `jdbc_pool_timeout`, `jdbc_validate_connection`; monitor DB connection count × Logstash instances.
- **Slow queries:** Index DB columns used in lookup; avoid `%` wildcard leading queries.

### 53. Real-time IP reputation lookups

- **Feed:** Download threat intel MMDB/CSV; refresh via cron to shared volume.
- **Lookup:** `translate` or `jdbc_static` with CIDR ranges; or dedicated threat API with strict timeout.
- **Cache:** In-memory with TTL; negative cache misses to avoid hammering API.
- **Conditional:** Only external IPs; sample internal DNS logs.
- **Updates:** Blue-green feed files (`threat_intel_v2.yml`) swap with reload signal.

### 54. `ruby` filter in production

- **Use for:** Complex branching, array manipulation, custom crypto — when no native filter exists.
- **Performance:** Interpreted; 10–100× slower than native — avoid per-event on 100K+ eps paths.
- **Security:** Disable arbitrary user scripts; code review; no `eval` of event fields; restrict file permissions on pipeline configs.
- **Alternative:** Elasticsearch ingest Painless for index-time logic.

### 55. Event deduplication with `fingerprint`

```conf
filter {
  fingerprint {
    source => ["message", "host.name", "@timestamp"]
    method => "SHA256"
    target => "[@metadata][doc_id]"
  }
}
output {
  elasticsearch {
    document_id => "%{[@metadata][doc_id]}"
    action => "index"
  }
}
```

- **Trade-offs:** CPU for hash; collisions theoretically possible with SHA256 — negligible; near-duplicate messages still dedupe incorrectly if timestamp differs.
- **Accuracy vs perf:** Narrow `source` fields for true business-key dedup; wider fields for aggressive dedup.

---

## Input/Output Plugins & Integration

### 56. Elasticsearch output under cluster pressure

- **Tune:** Reduce `workers` to avoid overwhelming ES; increase `retry_max_interval`; enable `healthcheck_path`.
- **Bulk:** Lower `bulk_max_size` temporarily if rejections occur; fix ES capacity first.
- **Buffer:** Persistent queue + alert on queue depth.
- **Circuit break:** Custom `if [@metadata][es_down]` route to S3/kafka backup output when health check fails.
- **ILM:** Ensure index lifecycle not forcing mass merges during peak.

### 57. S3 historical + Kafka real-time

- **Separate pipelines:** `s3-historical` pipeline with low `pipeline.workers` and dedicated nodes; `kafka-realtime` on production pool.
- **Resource isolation:** Different K8s node pools or cgroup limits so S3 backfill cannot starve realtime.
- **S3 input:** `sincedb_path` per worker for checkpoint; schedule backfill off-peak.
- **Priority:** Never share consumer threads — S3 is batch-read, Kafka is streaming.

### 58. Multiple destinations (ES, S3, Kafka)

```conf
output {
  if [store_es] {
    elasticsearch { index => "logs-%{+YYYY.MM.dd}" }
  }
  if [store_archive] {
    s3 { bucket => "logs-archive" prefix => "%{tenant}/" codec => json_lines }
  }
  if [downstream_siem] {
    kafka { topic_id => "normalized-logs" codec => json }
  }
}
```

- **Use `clone` filter** to duplicate event for different filter chains if needed.
- **Format per dest:** `mutate` before each output or separate pipelines via `pipeline` output.

### 59. HTTP output to webhooks

```conf
output {
  http {
    url => "https://hooks.example.com/events"
    http_method => "post"
    format => "json"
    retryable_codes => [429, 500, 502, 503]
    automatic_retries => 5
    connect_timeout => 10
    socket_timeout => 30
    headers => { "Authorization" => "Bearer ${WEBHOOK_TOKEN}" }
  }
}
```

- **Rate limits:** Honor `Retry-After` header via retry; throttle input if 429 sustained.
- **Batch:** Consider aggregating events or using message bus instead of per-event HTTP at scale.

### 60. Windows Event Log + Active Directory enrichment

- **Input:** `winlogbeat` → Logstash `beats` input (preferred) or `snmp`/agentless WEF subscription.
- **Normalize:** Map `winlog.event_id`, `winlog.channel` to ECS; `date` filter on `winlog.time_created`.
- **AD enrich:** `ldap` filter or `jdbc_streaming` against AD snapshot (`samAccountName` → department); cache heavily.
- **Volume:** Dedicated pipeline; filter security vs operational channels early.

---

## Kubernetes & Cloud Deployment

### 61. Logstash on Kubernetes with persistent queues

- **StatefulSet** with `volumeClaimTemplates` mounting `path.queue` and `path.dead_letter_queue`.
- **Pod disruption:** PDB `minAvailable: 1`; `terminationGracePeriodSeconds: 300` for PQ drain.
- **Reschedule:** Same PVC reattaches — queue survives; verify `podManagementPolicy: OrderedReady` for rolling updates one-at-a-time.
- **Maintenance:** Cordon node, drain pod gracefully, verify consumer lag before next pod.

### 62. ConfigMaps and hot reload

- **Mount** `/usr/share/logstash/pipeline` from ConfigMap; `config.reload.automatic: true`.
- **Validation:** Init container runs `logstash -t` before main container starts.
- **Limitation:** JVM/plugin settings in `logstash.yml` still need restart; only pipeline conf hot-reloads.
- **GitOps:** Flux/Argo sync ConfigMap; rolling restart for non-reloadable changes.

### 63. Resource management on Kubernetes

- **Requests/limits:** CPU request = expected workers; memory limit = heap + 2× heap for PQ/direct (e.g., 8G heap → 20G limit).
- **JVM:** Set `LS_JAVA_OPTS` from downward API `resources.limits.memory` (e.g., 50% for heap).
- **OOMKilled:** Increase limit or reduce heap/PQ size; check for native memory leak in plugin.
- **Probes:** HTTP GET `9600/_node/stats` for liveness; check `status` green and queue not full for readiness.

### 64. HPA on Kafka consumer lag

- **Prometheus adapter:** Expose `kafka_consumergroup_lag` per group; HPA `type: External` target average lag per pod.
- **Formula:** `desiredReplicas = ceil(currentLag / targetLagPerPod)`.
- **Cooldown:** Scale-down stabilization window 300s; scale-up 60s.
- **Cap:** `maxReplicas` ≤ Kafka partition count.

### 65. Monitoring on Kubernetes

- **Metricbeat** `logstash` module or Prometheus exporter scraping port 9600.
- **Metrics:** JVM heap/GC, pipeline events in/out, queue depth, plugin durations.
- **Grafana:** Dashboard per pipeline ID; alert to PagerDuty on lag and DLQ.
- **Logs:** Logstash own logs to stdout → cluster logging stack; separate from processed data path.

---

## Security & Compliance

### 66. TLS for inputs and outputs

- **Inputs/outputs:** `ssl => true`, `ssl_certificate_verification => true`, mount certs from secrets.
- **Beats/Kafka:** Mutual TLS with client cert rotation via cert-manager; 90-day renewal automation.
- **Rotation:** Dual-trust window — accept old + new CA during rollover; monitor expiry alerts at T-30/T-7 days.
- **Cipher suites:** Modern TLS 1.2+ only; disable weak protocols in `ssl_supported_protocols`.

### 67. PII masking and redaction

```conf
filter {
  mutate {
    gsub => [
      "message", "\b\d{3}-\d{2}-\d{4}\b", "[SSN_REDACTED]",
      "email", ".+", "[EMAIL_REDACTED]"
    ]
  }
  prune { fields => ["password", "credit_card"] }
}
```

- **Nested JSON:** `json` filter then `mutate` on nested fields or `ruby` for recursive redaction.
- **Tokenize:** Replace with HMAC token for correlation without storing raw PII.
- **Validate:** Unit tests proving redaction on sample events before deploy.

### 68. Pipeline access controls and audit

- **Git-only changes:** Pipeline configs in Git with PR approval; no manual edits on prod hosts.
- **RBAC:** K8s Role limiting who can edit Logstash ConfigMaps/Secrets.
- **Audit:** Git commit history + OPA/Conftest policy checks in CI; ship audit logs to SIEM.
- **Secrets:** Vault/ES keystore for `${VAR}` credentials — never plain text in ConfigMap.

### 69. GDPR data subject deletion

- **Identification:** Index fields `user.id`, `email.keyword` for search; maintain data map of which indices hold PII.
- **Deletion:** ES `delete_by_query` on `user.id:"xyz"` across data streams — Logstash does not delete historical data by itself.
- **Pipeline role:** Tag all events with `user.id` at ingest for future deletes; optional `fingerprint` for traceability.
- **Retention:** ILM delete phase enforces maximum retention; legal hold index alias excluded from deletion jobs.

---

## Advanced Troubleshooting Scenarios

### 70. Incorrect timestamps / wrong daily indices

- **Check `date` filter** `match` patterns against sample `message` — app may have changed format → `_dateparsefailure`.
- **Timezone:** Set `timezone => "UTC"` or correct zone; DST bugs cause day-boundary shifts.
- **Clock skew:** Compare `@timestamp` vs `event.created`; use `date` on correct field not ingestion time.
- **Fix forward:** Deploy corrected `date` filter; new indices correct; reindex only affected days if compliance requires — not full history.
- **Detect:** Alert on sudden index name date mismatch vs `now`.

### 71. OOMKill every 6 hours (memory leak)

- **Heap dump:** `-XX:+HeapDumpOnOutOfMemoryError` on OOM; analyze with Eclipse MAT for retained `aggregate` maps or plugin buffers.
- **Metrics:** Rising `jvm.mem.heap_used_percent` trend with flat throughput = leak; sawtooth = normal.
- **Suspects:** `aggregate`, `ruby`, `jdbc_streaming` cache unbounded, PQ misconfigured unlimited.
- **Mitigate:** Upgrade plugin version; restart cron as temp fix; isolate pipeline.

### 72. Intermittent silent event loss

- **Enable tracking:** `tags` + `uuid` filter at input; compare counts input vs ES (`_count` API) per hour.
- **Check:** `drop {}` filter accidental `if` logic; `queue.max_events` dropping; output `action => "delete"`.
- **PQ:** Events expired if `queue.max_bytes` too small with silent backpressure.
- **Pipeline ordering:** Multiple outputs — one path drops. Use `stdout { codec => dots }` in staging to trace.

### 73. Microservices correlation via trace ID

```conf
filter {
  json { source => message }
  if ![trace.id] { mutate { add_field => { "trace.id" => "%{[headers][x-trace-id]}" } } }
  mutate { add_field => { "[@metadata][index_suffix]" => "logs-%{+YYYY.MM.dd}" } }
}
```

- **ECS:** Use `trace.id`, `span.id`, `service.name` fields aligned with OpenTelemetry.
- **Downstream:** Kibana APM/Traces correlation or ES ES|QL joins on `trace.id`.

### 74. CI/CD pipeline testing

- **Syntax:** `bin/logstash -f pipeline.conf -t` in CI on every PR.
- **Unit:** `logstash-filter-verifier` with YAML test cases (input → expected output).
- **Integration:** Docker Compose with ES + sample Kafka topic; assert document count and fields.
- **Perf:** JMeter/gohai load test in staging; gate on p99 filter latency and eps target.

### 75. Breaking filter plugin upgrade (20 pipelines)

- **Rollback:** Pin previous Logstash Docker image tag in Helm values; revert Git pipeline commit.
- **Migration:** Test matrix in staging per pipeline; changelog review for plugin breaking changes.
- **Phased:** Upgrade canary instance per pipeline family; monitor DLQ and `grokparsefailure`.
- **Pin plugins:** `logstash-plugin install --version X` in custom image for reproducibility.

### 76. Log rotation without loss or duplicates

- **Use Filebeat/Agent** — not raw `file` input in prod; handles inode rotation via `close_inactive`, `clean_removed`.
- **If `file` input:** `sincedb_path` persisted; `discover_interval`; `file_completed_action => delete` only when safe.
- **Duplicates:** Rotation race causes re-read — Beats `file_identity` or `inode` tracking prevents; `fingerprint` dedup safety net.
- **Loss:** Missing sincedb on restart re-reads or skips — persist sincedb on PVC.

### 77. High-volume metrics to ES and TSDB

- **Split:** `clone` or metric-specific pipeline; `if [data_stream.type] == "metrics"` route to Prometheus remote_write via `http` output or dedicated exporter.
- **Format:** Convert logs-derived metrics to Prometheus exposition format with `ruby` or `metricbeat` at source (preferred).
- **Cardinality:** Drop high-cardinality labels before TSDB; keep full log in ES.

### 78. Tamper-evident / signed logs

- **Pattern:** `fingerprint` or canonical JSON serialization → sign with `ruby` filter using OpenSSL private key (HSM-backed).
- **Fields:** Add `log.signature`, `log.signature_algorithm`, `log.chain_hash` (hash previous + current for chain).
- **Verify:** Downstream verifier service or ES ingest pipeline validates on write.
- **Limits:** Signing 100K eps is CPU-heavy — sign at aggregate boundaries (batch files) or use WORM storage + audit trail instead.

---

## Basic Questions

### 1. What is Logstash and its role in the ELK stack?

Logstash is a server-side data processing pipeline that ingests data from multiple sources, transforms it, and sends it to a destination (usually Elasticsearch). In the ELK stack it sits between data sources (Beats, apps, Kafka) and Elasticsearch, handling parsing, enrichment, and routing. Kibana visualizes the data Elasticsearch stores.

### 2. Three main components of a Logstash pipeline

A pipeline has **inputs** (receive data), **filters** (transform/enrich), and **outputs** (send to destinations). Events flow in one direction: input → filter chain → output. Multiple inputs and outputs can exist in one pipeline configuration.

### 3. Input plugin — three examples

Input plugins read data from external sources. Examples: `beats` (Elastic Agent/Beats), `kafka` (Kafka topics), and `file` (log files on disk).

### 4. Filter plugin — three examples

Filter plugins modify events in the pipeline. Examples: `grok` (pattern extraction), `mutate` (field changes), and `date` (timestamp parsing).

### 5. Output plugin — three examples

Output plugins send processed events to destinations. Examples: `elasticsearch`, `kafka`, and `stdout` (console debugging).

### 6. Purpose of the `grok` filter

Grok parses unstructured text into structured fields using named regex patterns (built-in and custom). It is the primary tool for turning raw log lines into queryable fields in Elasticsearch.

### 7. Start Logstash with a specific configuration file

Run `bin/logstash -f /path/to/pipeline.conf` or set `path.config` in `logstash.yml`. For a directory of configs: `bin/logstash -f /etc/logstash/conf.d/`.

### 8. Purpose of the `mutate` filter

Mutate performs general field transformations: rename, remove, add, replace, convert types, and `gsub` string substitution. It is one of the most commonly used filters for light-touch cleanup.

### 9. Logstash codec — two examples

Codecs encode/decode data on input or output (serialization format). Examples: `json` (parse/emit JSON) and `plain` (plain text with optional line delimiter).

### 10. Test configuration without starting full pipeline

Use `bin/logstash -f pipeline.conf -t` or `--config.test_and_exit` to validate configuration syntax and plugin registration, then exit.

### 11. Purpose of the `date` filter

The `date` filter parses a timestamp string from a field and sets it as the event's `@timestamp`. Correct `@timestamp` is essential for time-based index routing and Kibana time picker accuracy.

### 12. Configure file input

```conf
input {
  file {
    path => "/var/log/app/*.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb"
    codec => "plain"
  }
}
```

### 13. Purpose of `if` conditional in pipeline

`if` / `else` blocks conditionally execute filters or outputs based on field values, tags, or metadata. They enable routing, selective parsing, and dropping events without separate pipelines.

### 14. Logstash `beats` input plugin

Receives events from Elastic Beats (Filebeat, Metricbeat, etc.) over the Lumberjack protocol, typically on port 5044. It is the standard production path for shipping logs from hosts to Logstash or Elasticsearch.

### 15. Configure output to Elasticsearch

```conf
output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    user => "elastic"
    password => "${ES_PASSWORD}"
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

### 16. Purpose of `add_field` option

`add_field` adds new fields to an event during filter or plugin execution. It is commonly used in `mutate` and input plugins to attach metadata like environment, pipeline name, or parsing version.

### 17. Pipeline worker

A pipeline worker is a parallel execution thread that processes batches of events through the filter and output stages. Worker count is controlled by `pipeline.workers` and defaults to the number of CPU cores.

### 18. Configure logging level for debugging

Set `log.level: debug` in `logstash.yml` or use `--log.level debug` on the command line. Increase temporarily during troubleshooting; revert to `info` in production to avoid log volume.

### 19. Purpose of `remove_field`

`remove_field` deletes specified fields from an event, often used to drop noisy or sensitive fields before indexing. It appears in `mutate` and several other filters.

### 20. Logstash `stdin` input plugin

Reads events from standard input for interactive testing and quick experiments. Typically paired with `stdout` output and `rubydebug` codec for development.

### 21. Parse JSON log events

```conf
filter {
  json { source => "message" }
}
```

Or on input: `codec => json`. Failed parses tag `_jsonparsefailure`.

### 22. Purpose of the `tags` field

Tags are labels added to events (e.g., `_grokparsefailure`, `nginx`) to mark processing outcomes. They drive conditional logic, alerting, and troubleshooting in downstream systems.

### 23. Logstash `stdout` output plugin

Writes events to the console, used for debugging pipelines. Often combined with `codec => rubydebug` to print all fields in human-readable form.

### 24. Add a timestamp to events

Use the `date` filter to parse a field into `@timestamp`, or `mutate { add_field => { "ingested_at" => "%{@timestamp}" } }` for ingestion metadata. `@timestamp` is set automatically on creation but should be corrected if log time differs.

### 25. Purpose of `@metadata` field

`@metadata` holds transient fields used only within the pipeline (routing, index names) and is **not** indexed into Elasticsearch by default. It is the safe place for secrets and internal control flags.

### 26. Drop specific events using a filter

```conf
filter {
  if [level] == "DEBUG" {
    drop { }
  }
}
```

### 27. Logstash `tcp` input plugin

Listens on a TCP socket for incoming log streams, common for syslog-forwarding and app direct-send. Configure `port`, `host`, and optional TLS.

### 28. Parse syslog format events

```conf
filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:syslog_message}" }
  }
}
```

Or use the `syslog` input plugin which handles parsing at ingest.

### 29. Purpose of `rename` in `mutate`

Renames an existing field to a new name (`rename => { "old_name" => "new_name" }`). Used for ECS field alignment and fixing naming conflicts before Elasticsearch indexing.

### 30. Logstash `http` input plugin

Receives log events via HTTP requests (POST/GET), useful for webhooks and lightweight agents. Supports codecs, authentication, and TLS for secure ingestion.

### 31. Route events to different outputs by field values

```conf
output {
  if [type] == "apache" {
    elasticsearch { index => "apache-%{+YYYY.MM.dd}" }
  } else {
    elasticsearch { index => "other-%{+YYYY.MM.dd}" }
  }
}
```

### 32. Purpose of `gsub` in `mutate`

`gsub` performs regex-based search-and-replace on field values. Used for sanitizing messages, stripping ANSI codes, or normalizing delimiters before grok.

### 33. Logstash `kafka` input plugin

Consumes messages from Apache Kafka topics as a Kafka consumer group member. It is the standard approach for high-volume, buffered log ingestion in production.

### 34. Configure pipeline batch size

Set `pipeline.batch.size: 125` in `logstash.yml` or `pipeline.batch.size => 125` in `pipelines.yml` per pipeline. Higher values increase throughput at the cost of latency.

### 35. Purpose of `convert` in `mutate`

`convert` changes a field's data type (e.g., string to integer, string to boolean). Important for correct Elasticsearch mappings and numeric aggregations.

### 36. Handle multiline log events

Use the `multiline` codec on file/beats input to merge stack traces into one event:

```conf
codec => multiline {
  pattern => "^\["
  negate => true
  what => "previous"
}
```

### 37. Logstash `elasticsearch` input plugin

Reads documents from Elasticsearch indices (scroll/search), used for reprocessing, migration, or enriching from existing indexed data. Not used for normal log ingestion.

### 38. Add GeoIP information

```conf
filter {
  geoip {
    source => "client.ip"
    target => "client.geo"
  }
}
```

Requires GeoLite2 database files updated periodically.

### 39. Purpose of the `split` filter

Splits an event with multiple values in one field (e.g., array or delimited string) into separate events. Used when one log line contains multiple records to index independently.

### 40. Environment variables in configuration

Reference `${VAR}` or `${VAR:default}` in pipeline configs; store secrets in the Logstash keystore (`bin/logstash-keystore add ES_PASSWORD`). Kubernetes injects env from Secrets into the pod spec.

### 41. Logstash `s3` input plugin

Reads log files from AWS S3 buckets for batch reprocessing or cloud-native log archives. Uses sincedb for checkpointing which objects/bytes have been read.

### 42. Parse CSV format events

```conf
filter {
  csv {
    columns => ["timestamp", "user", "action", "result"]
    separator => ","
  }
}
```

### 43. Purpose of the `clone` filter

Duplicates an event into multiple identical copies (with optional tags) so different filter/output chains can process the same event differently — e.g., security copy + archive copy.

### 44. Configure dead letter queue

```yaml
# logstash.yml
dead_letter_queue.enable: true
path.dead_letter_queue: /var/lib/logstash/dlq
```

Failed events that cannot be processed are written to DLQ files for later inspection and replay.

### 45. Logstash `http` output plugin

Sends events to an HTTP endpoint via POST (or other methods), useful for webhooks, SaaS integrations, and custom APIs. Supports retries, auth headers, and format codecs.

---

## Intermediate Questions

### 1. Persistent queue to prevent data loss on restart

Set `queue.type: persisted` and `path.queue` on durable disk with adequate `queue.max_bytes`. On restart, Logstash replays unacknowledged events from the queue before accepting new input, protecting against process crashes when outputs were temporarily unavailable.

### 2. Pipeline-to-pipeline communication for routing

Define a `pipeline` output in the source pipeline with `send_to => "name"` and a matching `pipeline { address => "name" }` input in the target pipeline. This in-process routing isolates expensive processing and enables fan-out without external messaging.

### 3. `aggregate` filter for multi-event correlation

Use `aggregate` with a shared `task_id` (e.g., session ID) and `code` blocks triggered on `map_action` start/end to accumulate fields across events, then push a merged event on timeout or completion. Always set `timeout` to prevent unbounded state growth.

### 4. `dissect` vs `grok`

Dissect uses delimiter-based extraction without regex backtracking, making it faster and safer for fixed-format logs. Grok uses regex patterns and handles irregular text better. Prefer dissect for stable structured formats; grok for syslog and variable unstructured lines.

### 5. Kafka input with consumer group parallelism

Configure `topics`, `group_id`, and `consumer_threads` (≤ partition count). Multiple Logstash instances sharing the same `group_id` divide partitions for parallel consumption; scale instances up to partition count for maximum throughput.

### 6. Elasticsearch output with ILM integration

Set `ilm_enabled => true`, `ilm_rollover_alias`, and `ilm_pattern` (or use data streams with `data_stream => auto`). Logstash creates backing indices that ILM policies roll over, warm, and delete per retention requirements.

### 7. `ruby` filter for complex transformations

Embed Ruby code in the `ruby` filter `code =>` block with access to `event` object for arbitrary logic. Use sparingly in production due to performance cost; pin and review code like application source.

### 8. Queue checkpoint and at-least-once delivery

Persistent queue checkpoints record durable write positions on disk at intervals (`queue.checkpoint.writes`). After crash, events after the last checkpoint may be replayed, ensuring outputs receive events at least once (duplicates possible without idempotent `_id`).

### 9. `jdbc_streaming` for real-time enrichment

Per-event JDBC lookup joins database rows into the event via parameterized `statement` and `parameters`. Use `cache_expiration` and `cache_size` to reduce DB load; size connection pools for `instances × workers`.

### 10. Pipeline workers and batch settings for throughput

Increase `pipeline.workers` to match CPU-bound capacity, `pipeline.batch.size` for higher ES bulk efficiency, and lower `pipeline.batch.delay` when latency-sensitive. Profile with node stats — if workers idle and queue grows, bottleneck is output not workers.

### 11. `fingerprint` filter for deduplication

Hash selected fields (`source`, `method`, `target`) to produce a deterministic ID used as Elasticsearch `document_id`. Identical events collapse to updates instead of duplicates, useful for at-least-once delivery paths.

### 12. `translate` filter for field value mapping

Loads a YAML/hash dictionary mapping input values to output values (e.g., HTTP code → description). Supports `fallback` for unknown keys and periodic `refresh_interval` when dictionary file changes on disk.

### 13. `throttle` filter for rate limiting

Limits how many events matching a key pass through per time period (`key`, `period`, `max_events`). Used to cap noisy sources, protect downstream systems, or sample DEBUG floods without full `drop`.

### 14. S3 output for archiving

```conf
output {
  s3 {
    bucket => "log-archive"
    prefix => "logs/%{+YYYY}/%{+MM}/%{+dd}/"
    region => "us-east-1"
    codec => "json_lines"
    time_file => 5
  }
}
```

Buffers events locally and uploads time-rolled objects for cheap long-term storage.

### 15. `xml` filter for XML logs

Parses XML content from a field into structured fields using XPath targets. Configure `store_xml` and `remove_field` to avoid keeping redundant raw XML in the indexed document.

### 16. `useragent` filter

Parses User-Agent strings into `device`, `os`, and `browser` fields using built-in regex databases. Apply to `user_agent.original` or web server access log fields after grok extraction.

### 17. `cidr` filter for IP range matching

Tests whether an IP field falls within configured CIDR blocks and adds tags or routes based on match. Used to classify internal vs external traffic or geo-policy routing without full GeoIP.

### 18. Kafka output with custom partitioning

Set `partition_key_format` or use `message_key` from event field so related events land on the same partition for ordering. Match partition count to downstream consumer parallelism.

### 19. `csv` filter for structured CSV data

Parses delimited strings into columns defined by `columns` array; handles quoted fields and custom `separator`. Faster than grok for pure CSV log formats.

### 20. `elapsed` filter for processing time

Tracks time between two events sharing a `unique_id_field` — typically start and end markers of an operation. Reports `elapsed_time` on the end event for latency analysis in logs.

### 21. `prune` filter for removing unwanted fields

Whitelist (`whitelist_names`) or blacklist (`blacklist_names`) fields to keep indexed documents lean. Essential for reducing mapping explosion and storage costs after verbose parsing.

### 22. HTTP output with authentication and retry

Configure `url`, `http_method`, `headers` for Bearer/Basic auth, `automatic_retries`, and `retryable_codes` for 5xx/429. Set timeouts to avoid blocking pipeline workers indefinitely.

### 23. `de_dot` filter for dotted field names

Replaces dots in field names with a substitute (underscore) because Elasticsearch 2.x+ treats dots as nested path separators. Required when parsing JSON with dotted keys before indexing.

### 24. `sleep` filter

Pauses processing for a specified duration per event — rarely used in production except testing backpressure or throttling reproduction. Avoid in high-volume paths.

### 25. `truncate` filter for field length limits

Cuts field values to `length_bytes` to prevent Elasticsearch `ignore_above` issues and oversized documents from stack traces or blob fields.

### 26. Monitoring API for pipeline health

Enable `api.http.host/port` and query `GET /_node/stats/pipelines` for event rates, queue depth, and plugin timings. Integrate with Metricbeat or Prometheus for dashboards and alerting.

### 27. `urldecode` filter

Decodes URL-encoded field values (`%20` → space) common in web access logs and query string parameters extracted by grok.

### 28. `kv` filter for key-value pair parsing

Extracts `key=value` pairs from a field using `field_split` and `value_split` delimiters. Lighter than grok for `level=INFO msg=hello` style logs.

### 29. `json_encode` filter

Serializes a subset of fields back into a JSON string field for outputs expecting a single JSON column (S3, certain APIs).

### 30. Pipeline configuration reloading

Set `config.reload.automatic: true` and `config.reload.interval: 3s` in `logstash.yml`. Logstash detects pipeline file changes, validates, and hot-applies without full JVM restart.

### 31. `metrics` filter for counting and timing

Increments counters and records execution time for tagged code paths — useful for custom observability of filter branches. Metrics exposed via monitoring API when configured.

### 32. `collate` filter for merging multiple inputs

Correlates events from different inputs by key and merges fields when all parts arrive within `timeout`. Similar to aggregate but designed for multi-input join patterns.

### 33. `extractnumbers` filter

Pulls numeric values from a string field into an array field. Useful for quick metric extraction from unstructured text without full grok.

### 34. Elasticsearch output with custom index templates

Set `template => "logs-template"`, `template_overwrite => true`, and `template_pattern` in the ES output, or manage templates separately via API/Terraform with correct mappings before first index.

### 35. `tld` filter for top-level domain extraction

Extracts registered domain and TLD from URL/host fields for web analytics and security domain reputation without full DNS lookup.

### 36. `environment` filter for environment variables

Adds OS environment variables to events as fields at pipeline start. Useful for tagging `deployment`, `region`, or `cluster` from pod env without hardcoding.

### 37. `range` filter for field value validation

Drops or tags events where numeric fields fall outside configured ranges — data quality guardrail for metrics-like log fields.

### 38. DLQ reprocessing pipeline

Use `dead_letter_queue` input plugin reading from `path.dead_letter_queue`, apply fixed transforms, output to ES with monitoring. Track retry counts and route perpetual failures to manual review index.

### 39. `uuid` filter for unique identifiers

Generates a UUID and stores it in a target field for every event — useful for trace tokens when source does not provide IDs.

### 40. `zeromq` input plugin

Receives high-throughput messages over ZeroMQ pub/sub or push/pull patterns for low-latency internal systems. Less common today than Kafka but useful for specialized embedded pipelines.

---

## Advanced Questions

### 1. 10M eps from 5,000 microservices

- **Ingest tier:** Elastic Agent/Fluent Bit on hosts → Kafka (1000+ partitions, tiered storage) — not direct Logstash per service.
- **Processing tier:** 50–200+ Logstash/ingest nodes in consumer groups; lightweight JSON parsing only.
- **Elasticsearch:** Dedicated coordinating/ingest nodes, 100+ data nodes, data streams, ILM, frozen/searchable snapshots for cold.
- **Failure handling:** Kafka retention 72h+, DLQ topics, idempotent `_id`, multi-AZ everywhere.
- **Governance:** Schema registry, ECS, per-team topics, SLO on end-to-end lag not Logstash CPU alone.
- **Cost control:** Hot/warm/cold architecture; sampling for DEBUG namespaces at edge.

### 2. Real-time enrichment platform (threat intel, CMDB, user directory)

- **Tier enrichment:** Edge (Agent) → stream (Logstash) → index-time (ES enrich processor) — put static lookups in `jdbc_static`/`translate` caches.
- **Async APIs:** Threat intel via batched refresh to local MMDB, not per-event API calls.
- **Isolation:** Separate enrichment pipelines per source latency SLA with circuit breakers.
- **Cache metrics:** Hit rate, staleness, DB connection pool saturation per enrichment type.
- **Fallback:** Tag `enrich_degraded` and index without enrichment rather than blocking pipeline.

### 3. Exactly-once with Kafka transactions and ES document IDs

- **Practical EOS:** Kafka idempotent producer + transactional consume-process-produce in stream processor; Logstash uses fingerprint `_id` + careful offset commit after bulk ACK.
- **Limitations:** Logstash lacks native Kafka EOS; rebalance and filter side effects break strict semantics.
- **Recommendation:** Flink/Kafka Streams for true EOS; Logstash as best-effort with dedup.
- **ES:** `create` vs `index` action, version conflicts handled with `retry_on_conflict`.

### 4. Schema-adaptive pipeline without restarts

- **Schema registry:** Confluent/JSON Schema version in Kafka headers; `if [schema_version]` conditional filters.
- **Dynamic Scripting:** Avoid; prefer `json` filter + ES `dynamic: runtime` fields for unknown fields.
- **Reload rules:** Parsing rules in Git-backed ConfigMap DB polled every N minutes + `config.reload.automatic`.
- **Unknown schema:** Route to `schema-unknown` index for analyst review and pattern addition.

### 5. Compliance platform with cryptographic signing

- **Sign at boundary:** HSM-backed key signs canonical event bytes; attach `signature` + `key_id` fields.
- **Chain of custody:** Immutable S3 Object Lock / WORM storage; ES stores hash references only.
- **Verification service:** Independent verifier on ingest before marking `verified=true`.
- **Performance:** Batch signing for archive windows; not per-event at 1M+ eps without hardware acceleration.

### 6. Survive 24h Elasticsearch outage

- **Kafka buffer:** 24h+ retention (size retention bytes calculated from peak eps × event size).
- **Logstash PQ:** Secondary buffer on each node sized for shorter local outages only.
- **Spill to S3:** Output plugin writes to S3 when ES health check fails; replay via S3 input when ES returns.
- **Runbook:** Gradual replay throttle to avoid overwhelming recovering cluster.
- **Monitoring:** Alert at 50% buffer capacity with estimated time-to-full.

### 7. Financial services — fraud routing to SIEM

- **Parse transactions:** Strict schema validation; `fingerprint` for transaction ID dedup.
- **Rules:** CEP in Flink or ES transform for velocity/geo anomalies; Logstash for normalization/enrichment only.
- **Routing:** `if [risk_score] > threshold` → Kafka SIEM topic + full copy to compliance index.
- **Latency:** Sub-second path with dedicated small pipeline; audit all routing decisions in `audit` field.
- **PCI:** Tokenize PAN in Logstash before any storage; never log raw card data.

### 8. Autoscale on Kafka lag with stateful filters

- **Stateless Logstash tier:** Autoscale HPA on lag; move `aggregate` state to Redis/Flink.
- **Scale-up:** Fast (60s); **scale-down:** Slow with 30m stabilization + partition rebalance planning.
- **Zero downtime:** Rolling StatefulSet one pod at a time; `queue.drain` + consumer static membership.
- **Max replicas = partition count** for Kafka-bound workloads.

### 9. ML-based PII detection and redaction

- **Integration:** Call ML API (sampled) or run local ONNX model via `http` filter / sidecar — not inline ruby at scale.
- **Human review:** Low-confidence detections → quarantine index.
- **Feedback loop:** Analyst corrections improve model; version `redaction_model_version` on events.
- **Compliance log:** Record what was redacted (field names only, not values) for audit.

### 10. 100 application types — dynamic parsing rules from database

- **Rules service:** DB/API returns grok/dissect pattern by `service.name` + `log.format_version`.
- **Cache:** In-memory rules cache refreshed every 60s; `config.reload` for file-based fallback.
- **Pipeline:** Generic `ruby` or plugin calling rules engine — minimize per-app pipeline files.
- **Testing:** Each rule version has fixture tests in CI before DB promotion.

### 11. Healthcare HL7 with HIPAA

- **HL7:** Dedicated Mirth/HL7 integration or `hl7` codec/filter; map to FHIR/ECS subset.
- **Encryption:** Field-level encrypt PHI (`ssn`, `mrn`) with KMS keys before ES; store keys separately.
- **Access audit:** Log every enrichment DB lookup with user/service identity.
- **BAA:** Ensure ES/S3/Kafka vendors HIPAA-eligible; no PHI in Logstash logs or DLQ without encryption.

### 12. Multi-stage attack correlation with aggregate + external state

- **Don't use JVM aggregate at scale:** Redis/Elasticsearch for session state keyed by `host.id` + `user.name`.
- **Rules:** Windowed joins (Flink) for lateral movement patterns across firewall, auth, and endpoint logs.
- **Logstash role:** Normalize all sources to ECS security fields; publish to detection topic.
- **Alerting:** SIEM rules on correlated index, not Logstash stdout.

### 13. Real-time data quality monitoring

- **Quality metrics:** Rate of `_grokparsefailure`, missing `service.name`, timestamp drift > 5min, schema version mismatch.
- **Implementation:** `metrics` filter + Prometheus exporter or Logstash monitoring → ES data stream `logstash-quality`.
- **Alert:** Anomaly detection on failure rate % per source.
- **Dashboard:** Per-team data quality scorecards tied to onboarding gates.

### 14. Event streaming to multiple downstream consumers

- **Kafka as bus:** Logstash normalizes once → Kafka topic → independent consumers (ES, S3, SIEM).
- **Or clone pipeline:** `pipeline` fan-out with format-specific branches.
- **Contracts:** Schema registry + compatibility checks per consumer.
- **Backpressure:** Each consumer group manages own lag; don't block ES on slow SIEM.

### 15. Historical backfill + real-time

- **Separate consumer groups:** `backfill-group` on S3 input pipeline with capped workers; `realtime-group` on live Kafka.
- **Index naming:** `logs-{service}-backfill-*` vs live to avoid ILM confusion.
- **Throttle:** `sleep` or `throttle` on backfill; schedule off-peak.
- **Completion:** Alert when sincedb reaches end of S3 prefix list.

### 16. ML log classification routing

- **Inference:** HTTP to model service with log text; receive `class` + `confidence`.
- **Routing:** `if [confidence] > 0.9` route; else `unclassified` index for review.
- **Feedback:** Relabel events in UI → retrain model; version field on pipeline.
- **Fallback:** Rule-based classifier when model unavailable.

### 17. Multi-cloud log collection (AWS, GCP, Azure)

- **Cloud-native → central Kafka:** Kinesis Firehose / Pub/Sub / Event Hub → forwarder → Kafka (neutral buffer).
- **Logstash:** Single normalization pipeline to ECS; cloud metadata in `cloud.*` fields.
- **Avoid:** 3 different Logstash deployment patterns — one code path, parameterize by `cloud.provider`.
- **Egress cost:** Compress on wire; aggregate in-region before cross-cloud transfer.

### 18. Extract metrics from unstructured logs to Prometheus

- **Prefer:** OpenTelemetry/metrics at source; Logstash as last resort.
- **Pattern:** `grok` extract numbers → `http` output to Prometheus Pushgateway or expose via metricbeat parser.
- **Cardinality control:** Limit label set; drop high-cardinality IDs from metric labels.
- **Alternative:** ES ingest pipeline + transform to metrics index.

### 19. End-to-end event tracing with Jaeger/Zipkin

- **Inject:** Propagate `trace.id`/`span.id` from OTel SDK in apps; Logstash preserves on ingest.
- **Normalize:** Map various header names (`X-Trace-Id`, `traceId`) to ECS fields in one filter.
- **Correlation:** Store logs in ES with same `trace.id` as APM data; Kibana unified view.
- **Don't generate traces in Logstash** — only preserve and validate presence.

### 20. High-cardinality cost control

- **Detect:** Track unique field cardinality via ES field stats API scheduled job.
- **Sample/drop:** `throttle` + hash-modulo sampling for noisy DEBUG; drop fields with >100k unique values.
- **Routing:** High-cardinality streams to separate cluster with stricter ILM (1d retention).
- **Edge filtering:** Drop useless fields at Beats before Logstash.

### 21. IoT — 1M devices

- **Protocol gateways:** MQTT → Kafka (not 1M TCP connections to Logstash).
- **Normalization:** Device registry lookup (`device.id` → model/firmware); anomaly rules in stream processor.
- **Partitioning:** Kafka keyed by `device.id` for ordered device telemetry.
- **Scale:** Logstash/Kafka consumers in hundreds; not one Logstash for all devices.

### 22. Immutable audit logging platform

- **Append-only:** WORM S3 + ES index with `index.blocks.write` after rollover.
- **Integrity:** Hash chain per batch; periodic anchor to external timestamp authority.
- **Logstash:** Minimal transform — timestamp, sign, route; no `mutate` delete on audit path.
- **Access:** Break-glass procedures logged to separate meta-audit stream.

### 23. Auto-recovery from failures without duplicates

- **Kafka retention + consumer offset commit after ES bulk success.**
- **Idempotent `_id`:** `%{topic}-%{partition}-%{offset}` or content fingerprint.
- **Ordering:** Partition by entity key; accept per-key ordering only.
- **Chaos test:** Kill Logstash mid-batch, verify ES doc count matches Kafka offset progress.

### 24. Geospatial enrichment and data sovereignty

- **GeoIP/MMDB:** Add `geo.country_iso_code`; route `if [geo.country]` to region-specific ES cluster output.
- **Sovereignty:** EU tenants → EU cluster only; enforce in output conditional, not just index name.
- **Accuracy:** Use MaxMind GeoIP2; refresh weekly; don't rely on DNS geolocation.

### 25. Self-service pipeline configuration platform

- **UI/API:** Teams submit grok samples + field catalog → generates reviewed pipeline snippet.
- **Guardrails:** Schema lint, cardinality check, PII scanner in CI before merge to Git.
- **Runtime:** Shared multi-tenant pipeline reads team's rules version from config service — SRE owns platform not each grok.
- **Observability:** Per-team lag, parse failure rate, and cost attribution dashboards.

---

## Rapid-Fire Questions

### 1. Default port for Beats input?

**5044**

### 2. What does the `grok` filter do?

Parses unstructured text into structured fields using named regex patterns.

### 3. Purpose of `@timestamp`?

Canonical event time used for Elasticsearch index sorting, Kibana time picker, and ILM; distinct from ingestion time.

### 4. What does `%{COMBINEDAPACHELOG}` match?

A standard Apache combined access log line (client IP, ident, user, timestamp, request, status, bytes, referrer, user-agent).

### 5. Purpose of `--config.test_and_exit`?

Validates pipeline configuration and plugin setup, then exits without running the pipeline.

### 6. What does `mutate` `convert` do?

Changes a field's data type (e.g., string to integer or boolean).

### 7. Purpose of persistent queue?

Buffers events on disk during output failures or restarts to prevent data loss and provide backpressure.

### 8. What does `_grokparsefailure` indicate?

The grok filter failed to match the event message against any provided pattern.

### 9. Purpose of `--pipeline.workers`?

Sets the number of parallel worker threads processing filter/output batches.

### 10. What does the `date` filter do?

Parses a timestamp from a field and sets the event's `@timestamp`.

### 11. Purpose of `--pipeline.batch.size`?

Maximum number of events collected per worker batch before filter execution.

### 12. What does `_jsonparsefailure` indicate?

The `json` filter (or JSON codec) failed to parse the field as valid JSON.

### 13. Purpose of `@metadata`?

Internal transient storage for pipeline routing/control fields not sent to outputs by default.

### 14. What does the `fingerprint` filter do?

Computes a hash digest of specified fields for deduplication or deterministic IDs.

### 15. Purpose of `--config.reload.automatic`?

Automatically reloads pipeline configuration files when they change on disk.

### 16. What does the `dissect` filter do?

Splits a string into fields using delimiter-based mapping (no regex).

### 17. Purpose of dead letter queue?

Stores events that fail processing for later inspection and reprocessing.

### 18. What does `_dateparsefailure` indicate?

The `date` filter could not parse the timestamp field with given patterns.

### 19. Purpose of `--path.queue`?

Filesystem directory where persistent queue data is stored.

### 20. What does the `aggregate` filter do?

Correlates multiple events sharing a task ID into combined state or a synthesized event.

### 21. Purpose of `--pipeline.batch.delay`?

Maximum time (ms) a worker waits to fill a batch before processing a partial batch.

### 22. What does the `clone` filter do?

Creates duplicate copies of an event for parallel processing paths.

### 23. Purpose of `--log.level`?

Controls Logstash's own log verbosity (fatal, error, warn, info, debug, trace).

### 24. What does the `split` filter do?

Splits one event with multiple values into separate events.

### 25. Purpose of `--queue.type`?

Selects in-memory (`memory`) or disk-backed (`persisted`) queue implementation.

### 26. What does the `translate` filter do?

Maps field values using a dictionary file (lookup table).

### 27. Purpose of `--queue.max_bytes`?

Maximum disk space the persistent queue may consume before backpressuring inputs.

### 28. What does the `throttle` filter do?

Rate-limits events matching a key to a maximum count per time period.

### 29. Purpose of `--pipeline.id`?

Unique identifier for a pipeline when running multiple pipelines in one instance.

### 30. What does the `prune` filter do?

Removes fields not in a whitelist (or removes blacklisted fields).

### 31. Purpose of `--path.config`?

Directory or file path containing pipeline configuration(s).

### 32. What does the `ruby` filter do?

Executes custom Ruby code to transform events.

### 33. Purpose of `--node.name`?

Human-readable name for the Logstash node in monitoring and cluster views.

### 34. What does the `kv` filter do?

Parses `key=value` pairs from a field into separate fields.

### 35. Purpose of `--http.port`?

Port for Logstash monitoring/management HTTP API (default 9600).

### 36. What does the `useragent` filter do?

Parses User-Agent strings into browser, OS, and device fields.

### 37. Purpose of `--queue.checkpoint.writes`?

Number of queue write events between durable checkpoints on disk.

### 38. What does the `geoip` filter do?

Adds geographic location fields based on an IP address using a GeoIP database.

### 39. Purpose of `--pipeline.unsafe_shutdown`?

Allows fast shutdown without draining queues (risk of data loss/replay).

### 40. What does the `cidr` filter do?

Matches IP addresses against CIDR ranges and tags or routes accordingly.

### 41. Purpose of `--config.reload.interval`?

How often Logstash checks for pipeline configuration file changes (seconds).

### 42. What does the `elapsed` filter do?

Measures elapsed time between related start and end events.

### 43. Purpose of `--pipeline.ecs_compatibility`?

Sets Elastic Common Schema compatibility mode for plugin field naming defaults.

### 44. What does the `truncate` filter do?

Shortens field values exceeding a specified byte length.

### 45. Purpose of `--queue.page_capacity`?

Size of each persistent queue page file on disk (default 64mb).

### 46. What does the `de_dot` filter do?

Replaces dots in field names with another character for Elasticsearch compatibility.

### 47. Purpose of `--modules`?

Runs bundled Logstash modules (pre-built integrations) by name, e.g., `nginx`.

### 48. What does the `metrics` filter do?

Records custom counters and timing metrics for pipeline observability.

### 49. Purpose of `--cloud.id`?

Connects to Elastic Cloud deployment using a cloud ID string instead of manual host list.

### 50. What does the `uuid` filter do?

Generates and adds a UUID value to a specified field.

---
