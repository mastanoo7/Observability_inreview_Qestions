# Elasticsearch — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## Shard Allocation & Cluster Management

### 1. RED status with 15% unassigned shards (disk OK, nodes online)

- **Confirm scope:** `GET /_cluster/health` and `GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason` to list unassigned shards and reasons.
- **Allocation explain:** `GET /_cluster/allocation/explain` (optionally with `{"index":"logs-2024","shard":0,"primary":true}`) for per-shard failure reasons (e.g., `NO_VALID_SHARD_COPY`, `CLUSTER_RECOVERING`, disk watermark).
- **Check allocation settings:** `GET /_cluster/settings?include_defaults=true&filter_path=**.routing.allocation*` for disabled allocation, zone mismatches, or `cluster.routing.allocation.enable: none`.
- **Review recent changes:** Pending tasks `GET /_cluster/pending_tasks`, recent index creation, template changes, or node attribute changes that broke allocation filters.
- **Disk/watermarks anyway:** Even at 15% unassigned, verify `GET /_cat/allocation?v` — `flood_stage` blocks allocation; clear or expand if any node tripped watermarks.
- **Remediate:** Fix root cause (restore from snapshot if no valid copy), then `POST /_cluster/reroute?retry_failed` or remove blocking settings; for missing primaries with replicas, promote replica via reroute API only after confirming data safety.
- **Monitor recovery:** `GET /_cat/recovery?v` until cluster returns GREEN or acceptable YELLOW (replica-only unassigned).

### 2. Primary vs replica allocation and routing algorithm

- **Primary allocation:** Occurs on index creation or when primary is lost; cluster picks a node satisfying routing rules, disk, and awareness; only one primary per shard.
- **Replica allocation:** Placed on a different node than its primary; count governed by `index.number_of_replicas`; replicas can serve search and act as failover.
- **Routing algorithm:** Master applies deciders (disk, awareness, balance, throttling, filters); scores nodes and picks the best; rebalancing moves shards to equalize disk and shard count per tier.
- **Tunable parameters:** `cluster.routing.allocation.balance.*`, `cluster.routing.allocation.awareness.attributes`, `cluster.routing.allocation.node_concurrent_recoveries`, disk watermarks, and per-index `index.routing.allocation.*` filters.
- **API inspection:** `GET /_cluster/allocation/explain` and `GET /_nodes/stats/indices?filter_path=nodes.*.indices.shards`.

### 3. Shard consolidation (1,000 indices × 5 shards, JVM pressure)

- **Assess:** Target ~20–50 GB per shard for logs; `GET /_cat/indices?v&s=store.size:desc` and shard counts per node; identify oversharded small indices.
- **Consolidate via reindex/shrink:** `_reindex` multiple small indices into fewer larger indices, or `POST /source/_shrink/target` after making source read-only and relocating to one node.
- **ILM rollover:** Use larger `number_of_shards` at rollover time in index templates (e.g., 1–3 shards per index instead of 5) for new data; avoid changing shard count on existing indices in place.
- **Reduce shard count:** Shrink (halve shards) or reindex to new index with correct settings; use aliases for zero-downtime cutover: `POST /_aliases` atomically swap read/write alias.
- **Throttle migrations:** Lower `cluster.routing.allocation.node_concurrent_recoveries` during business hours; run bulk consolidations off-peak.
- **Validate:** No data loss via doc count checks `_count` on source vs destination; monitor heap and search latency post-migration.

### 4. Shard allocation awareness (multi-zone)

- **Concept:** Spread primaries and replicas across failure domains (zones) using `cluster.routing.allocation.awareness.attributes` (e.g., `zone`).
- **Node config:** Set `node.attr.zone: us-east-1a` on each node; require at least as many zone values as `index.number_of_replicas + 1`.
- **Forced awareness:** `cluster.routing.allocation.awareness.force.zone.values: zone1,zone2,zone3` prevents allocation if a zone is down (may leave shards unassigned rather than violate zone safety).
- **Verify:** `GET /_cat/shards?v&h=index,shard,prirep,node,zone` (custom column via node attrs) or allocation explain.
- **Never same zone:** Awareness decider ensures replica and primary are on different `zone` attribute values.

### 5. Continuous shard rebalancing (high I/O)

- **Diagnose:** `GET /_cluster/settings` for recent changes; `GET /_nodes/stats` for relocation I/O; check disk watermarks, added/removed nodes, or changed awareness attributes causing endless rebalance.
- **Identify trigger:** Uneven disk usage, new nodes, or `cluster.routing.rebalance.enable` always on with aggressive balance thresholds.
- **Throttle:** `cluster.routing.allocation.node_concurrent_recoveries: 2` (default often 2), `cluster.routing.allocation.node_concurrent_incoming/outgoing_recoveries`, and `indices.recovery.max_bytes_per_sec: 40mb`.
- **Temporary relief:** `PUT /_cluster/settings` with `"cluster.routing.rebalance.enable": "none"` during peak, then re-enable `primaries` or `all` off-peak.
- **Fix root cause:** Add capacity, adjust disk watermarks, or fix mis-tagged zone attributes so cluster reaches equilibrium.

### 6. Shard over-allocation and optimal shard count

- **Impact:** Too many shards increase cluster state size, heap overhead per shard, file descriptors, and merge/search coordination cost; sub-10 GB shards are often inefficient.
- **Rule of thumb:** Target 10–50 GB per shard for time-series; 20–40 shards per GB heap on data nodes (guideline, workload-dependent); cap total shards per node via `cluster.max_shards_per_node`.
- **Calculate:** `(daily_volume_GB / target_shard_size_GB) × retention_days` shards needed; divide by nodes for even distribution.
- **Measure:** `GET /_cluster/stats` and `GET /_cat/shards?v` for average shard size; profile search/index latency vs shard count.

### 7. Index created with 50 shards instead of 5 (production traffic)

- **Cannot shrink in place to arbitrary count:** Shrink only reduces shard count by integer factor when shards are on one node and index is read-only.
- **Shrink path:** `PUT /index/_settings {"index.blocks.write": true}`, relocate all shards to one node, `POST /index/_shrink/target_index` with `"settings": {"index.number_of_shards": 5}` (50→25→5 may need multiple shrink steps or reindex).
- **Reindex (safer for production):** Create new index with 5 shards; `_reindex` with write alias swap for minimal downtime.
- **Prerequisites for shrink:** Index read-only, all shards on single node, primary shard count must shrink by factor; replicas set to 0 during shrink.
- **Zero downtime:** Use dual-write or reindex + alias cutover; `POST /_aliases` to point write alias to new index after catch-up.

### 8. Forced shard allocation routing

- **Manual reroute:** `POST /_cluster/reroute` with `commands` to `allocate`, `move`, or `cancel` shards for hotspot relief or maintenance.
- **`cluster.routing.allocation.exclude`:** Cluster-wide exclude nodes by `_name`, `_ip`, or custom attribute — shards evacuate excluded nodes.
- **`index.routing.allocation.require`:** Per-index require shards on nodes matching attribute (e.g., `"index.routing.allocation.require.box_type": "hot"`).
- **When to use which:** Exclude for decommissioning nodes cluster-wide; require/include for tiered storage and dedicating indices to specific hardware.
- **Caution:** Forced allocation ignoring allocation deciders (`allocate_stale_primary`) risks data loss — only when understanding consequences.

### 9. Shard proliferation and ILM prevention

- **Impact:** Thousands of tiny shards inflate master state, GC pressure, and recovery time; slow cluster coordination and "too many open files."
- **ILM strategy:** Rollover on size/age (`max_size`, `max_age`), reduce shards per index in template, use data streams for time-series.
- **Delete phase:** Auto-delete old indices; warm/cold migration instead of leaving many active small indices.
- **Monitoring:** Alert on shard count per node and average shard size; `GET /_cat/shards` aggregated.

### 10. "Too many open files" on data nodes

- **Relation to shards:** Each shard segment and translog file consumes file descriptors; high shard count × segments drives ulimit exhaustion.
- **OS-level:** Increase `nofile` in `/etc/security/limits.conf` and systemd `LimitNOFILE`; typical production values 65535+.
- **Elasticsearch:** Reduce shard count, forcemerge old indices (careful), increase `MaxBytesLength` only as band-aid; fix `bootstrap.system_call_filter: false` only if needed on some platforms.
- **Verify:** `GET /_nodes/stats/process?filter_path=**.open_file_descriptors` vs max; reduce oversharding long-term.

---

## Cluster Tuning & Performance

### 11. Search latency 50ms → 5s over two weeks (no query change)

- **Cluster health:** `GET /_cluster/health`, pending tasks, unassigned shards causing partial results or retries.
- **Node stats:** `GET /_nodes/stats?filter_path=nodes.*.indices.search,query_cache,request_cache,fielddata,jvm,fs` for GC, heap, cache evictions, segment count.
- **Hot threads:** `GET /_nodes/hot_threads` to find stuck merge, aggregation, or script threads.
- **Segment/merge pressure:** Growing index → more segments; check `GET /_cat/segments?v` and merge throttling.
- **Resource saturation:** CPU, disk IOPS (`iostat`), network; compare with indexing rate increases or new workloads.
- **Query-specific:** Enable slow log, `_profile` on representative queries; check if fielddata or scripted aggs spiked after mapping change.
- **Compare baselines:** Metricbeat/Prometheus trends for thread pool rejections and search queue size over the two-week window.

### 12. Circuit breakers

- **Mechanism:** JVM heap limits for operations (fielddata, request, in-flight requests, accounting); trip before OOM with `CircuitBreakingException`.
- **Prevented crash scenario:** Runaway aggregation on high-cardinality field trips `fielddata` breaker, rejecting queries but keeping node alive for other traffic.
- **Misconfigured rejections:** Breaker limit set too low (e.g., 10% fielddata on analytics cluster) causes false 429s under normal load; tune via `indices.breaker.*` and fix queries to use `keyword` + doc values.
- **API:** `GET /_nodes/stats/breaker` to see tripped counts and estimated sizes.

### 13. High GC pressure and stop-the-world pauses

- **Diagnose:** JVM stats `GET /_nodes/stats/jvm?filter_path=nodes.*.jvm.gc`, GC logs (G1GC for ES 7+), and heap dump if chronic.
- **Heap sizing:** Stay ≤50% RAM for heap (OS cache for Lucene); 31GB cap for compressed OOPs; don't exceed need.
- **GC algorithm:** Elasticsearch bundles G1GC; avoid CMS; tune `InitiatingHeapOccupancyPercent` only with evidence from GC logs.
- **Query patterns:** Reduce fielddata, use `keyword`, avoid deep aggregations; check circuit breaker trips.
- **Remediation:** Scale data nodes, reduce shard count, fix expensive queries before only adding heap.

### 14. Thread pool tuning (indexing + heavy aggregations)

- **Pools:** `write` (indexing/bulk), `search`, `get`, `analyze`, `management`; each has thread count and queue size.
- **Symptoms:** `rejected` in `GET /_cat/thread_pool?v` for search or write queues.
- **Tuning:** Increase threads cautiously (CPU-bound); better to scale nodes and balance shards; `thread_pool.search.queue_size` default 1000 — raising masks overload.
- **Isolation:** Separate coordinating-only nodes, or dedicated master/data tiers; ILM to cold for old data reducing concurrent search load.
- **Bulk management:** Client-side backoff on 429; `indexing_pressure` settings in 7.x+.

### 15. Adaptive replica selection (ARS)

- **Mechanism:** Coordinating node picks replica based on queue depth, response time, and service time using adaptive statistics — not round-robin.
- **Enable:** Default on supported versions; verify `cluster.routing.use_adaptive_replica_selection: true`.
- **Monitor:** Compare p50/p99 search latency per node, replica-level search thread pool stats, and whether hot nodes receive less traffic after ARS.
- **Metrics:** Node-level `search` latency in `GET /_nodes/stats`, imbalance in requests per replica in Elastic Stack monitoring.

### 16. Bulk indexing spiking search latency

- **Cause:** Shared CPU, disk I/O, and thread pools between indexing and search on data nodes.
- **Throttle indexing:** `indices.store.throttle.max_bytes_per_sec` (legacy store throttling), `indices.recovery.max_bytes_per_sec` for recovery; reduce bulk batch size and increase client backoff.
- **Refresh/backpressure:** Increase `refresh_interval` during bulk (`30s` or `-1` temporarily); monitor `indexing_pressure` and write pool rejections.
- **Architecture:** Dedicated ingest nodes/coordinators, hot tier for writes, off-peak bulk windows.
- **Bulk queue:** `GET /_cat/thread_pool/write?v` — if rejected, scale or slow producers.

### 17. Query cache, request cache, field data cache

- **Query cache:** Caches filter bitsets per segment (filter context); invalidated on segment merge/refresh.
- **Request cache:** Caches full aggregation results on shard level (size 0, same request, no `now`); keyed per shard.
- **Field data:** Legacy in-heap ordinals for `text` field sorting/aggs; replaced by `doc_values` on `keyword`.
- **Memory pressure scenario:** Huge request cache on repeated heavy aggs fills heap; tune `indices.requests.cache.size`, disable cache per request `"request_cache": false`, or fix query cardinality.
- **Eviction:** Monitor `GET /_nodes/stats/indices/query_cache,request_cache,fielddata`.

### 18. Index sorting

- **Implementation:** `index.sort.field` and `index.sort.order` at index creation — physically orders documents on disk for early termination.
- **Benefit:** Faster range queries and sorted results on that field (e.g., `@timestamp` desc).
- **Trade-offs:** Lower indexing throughput (must sort on flush), cannot change after creation; slight storage overhead during merge.
- **Use when:** Predictable sort/filter on one or few fields dominates query pattern.

### 19. Profile API for slow nested aggregations

- **Enable:** `GET /index/_search` with `"profile": true` — returns per-shard breakdown of query and aggregation time.
- **Analyze:** Identify expensive nested agg collectors, rewrite filters, reduce `nested` depth, use `reverse_nested` sparingly.
- **Optimize:** `docvalues_fields`, pre-filter with `filter` aggregation, reduce bucket `size`, use `composite` for pagination.
- **Iterate:** Compare `took` vs profile `time_in_nanos` to find coordination vs data node cost.

### 20. High disk I/O during merges

- **Tune merge:** `index.merge.scheduler.max_thread_count`, `index.merge.policy.max_merged_segment` (e.g., 5gb), `index.merge.policy.segments_per_tier`.
- **Throttling:** `indices.store.throttle.max_bytes_per_sec` (if applicable version), schedule `forcemerge` off-peak only when necessary.
- **Reduce segments:** Increase `refresh_interval` during bulk to fewer flushes; avoid excessive small shards.
- **Hardware:** SSD for hot tier, separate index/write from search-heavy nodes if possible.

---

## Heap Issues & JVM Tuning

### 21. OOMKilled in Kubernetes (30GB heap, 64GB node)

- **Off-heap consumption:** Direct buffers, Lucene mmap, thread stacks, OS page cache competition — total process RSS exceeds cgroup limit.
- **Diagnose:** `kubectl top pod`, node stats for `process.mem.total_virtual_in_bytes`, and check if limit is set too close to heap + overhead.
- **K8s settings:** Set container memory limit ~heap + 50% (e.g., 30GB heap → 45–48GB limit) or reduce heap to ~50% of limit minus overhead; requests should match for scheduling.
- **ECK:** Use `nodeSets[].podTemplate` and `spec.nodeSets[].config` with `ES_JAVA_OPTS=-Xms30g -Xmx30g`; avoid limit = heap only.
- **Fix:** Lower heap to 26–28g on 64GB node, or increase pod memory limit; monitor actual RSS via Elastic monitoring.

### 22. 50% heap rule and 32GB threshold

- **Rule:** Allocate ~50% of physical RAM to JVM heap; remainder for OS filesystem cache (critical for Lucene mmap).
- **32GB threshold:** Beyond ~31GB, JVM may disable compressed OOPs, doubling pointer overhead and reducing effective heap.
- **Counterproductive:** 40GB heap on 64GB machine starves OS cache and can increase GC cost without net benefit.
- **Sweet spot:** Often 16–31GB heap per data node depending on RAM.

### 23. Long GC pauses (>10s) causing instability

- **Approach:** Collect GC logs (`-Xlog:gc*` G1), correlate with large heaps, massive fielddata, or full GC from circuit breaker near limit.
- **Tune G1:** Adjust `MaxGCPauseMillis` cautiously; ensure adequate heap headroom; avoid oversizing heap past compressed OOP threshold.
- **Reduce heap pressure:** Fix fielddata usage, shrink shard count, upgrade to versions with better memory accounting.
- **Timeouts:** Increase `discovery.zen.publish_timeout` / fault detection only as temporary measure while fixing root cause.

### 24. Fielddata cache eviction storms

- **Symptom:** Repeated load/evict cycles on `text` fields used for sorting/aggs; high CPU and GC.
- **Diagnose:** `GET /_nodes/stats/indices/fielddata`, breaker trips, slow logs showing fielddata sorts.
- **Fix mapping:** Change to `keyword` with `doc_values: true`; use `text` with `fielddata: false` and multi-fields.
- **Alternative:** `runtime` fields or reindex; for legacy, `eager_global_ordinals` on keyword only when justified.

### 25. Off-heap and filesystem cache balance

- **Lucene:** Uses mmap for indices; hot data served from OS page cache — not JVM heap.
- **Tuning:** More heap ≠ faster search if it steals RAM from OS cache; monitor page cache hit rates and iostat.
- **Guideline:** 50% RAM heap, rest OS; on dedicated ES nodes avoid other memory-hungry processes.

### 26. Heap consistently 95% after GC

- **API:** `GET /_nodes/stats/jvm,indices?filter_path=nodes.*.jvm.mem,nodes.*.indices.fielddata,segments,memory` for heap pools and segment memory.
- **Heap dump:** `jmap` or Elastic diagnostic subagent on node; analyze dominators (fielddata, segments, caches).
- **Common culprits:** Too many shards, fielddata, large aggregations, circuit breaker near limits, memory leak in plugin.
- **Action:** Reduce caches, fix mappings, add nodes, or reduce heap pressure before increasing heap blindly.

### 27. JVM GC logging for production

- **Java 11+:** `-Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,level,pid,tags:filecount=32,filesize=64m`
- **Balance:** Rotate logs, avoid trace in steady state; use `info` level gc logging normally, enable `debug` briefly during incidents.
- **Centralize:** Ship GC logs to observability stack; correlate timestamps with search latency spikes.

---

## Indexing Latency & Throughput

### 28. Throughput drop 100k → 10k docs/s (no config change)

- **Client-side:** Check bulk batch sizes, thread count, retry storms, DNS/load balancer issues.
- **Network:** Latency/packet loss between ingest and cluster; TLS overhead.
- **Cluster:** `GET /_cluster/health`, thread pool rejections, disk watermarks (`read_only_allow_delete`), merge throttling, hot threads.
- **Compare:** Indexing rate metric per node, CPU/disk saturation, recent ILM relocations or snapshot activity.
- **Isolate:** Single-node bulk test vs full cluster; check if one index/shard is hot-spotted.

### 29. Refresh interval (throughput vs freshness)

- **Effect:** Default `1s` refreshes new segments for search — expensive at high ingest; longer interval improves throughput.
- **Bulk tuning:** `PUT /index/_settings {"index.refresh_interval": "30s"}` or `-1` during load, restore after.
- **Trade-off:** Search visibility delayed; use for log pipelines, not real-time inventory without planning.

### 30. High-throughput log pipeline (1TB/day)

- **Bulk sizing:** 5–15 MB per bulk request, 1–2 concurrent bulks per shard; tune with benchmarks.
- **Client:** Dedicated ingest tier, compression, idempotent retries with exponential backoff on 429.
- **Cluster:** Adequate shards (target shard size), `refresh_interval: 30s`, replicas for ingest parallelism, ILM rollover at 30–50 GB.
- **Hardware:** SSD hot tier, sufficient CPU for parsing, index templates with minimal analyzed fields.

### 31. Translog tuning

- **Purpose:** Durability WAL for not-yet-flushed operations; replayed on recovery.
- **`translog.durability`:** `request` (fsync each op, safest) vs `async` (faster, small loss window on crash).
- **`translog.flush_threshold_size`:** Larger threshold = fewer fsyncs, better throughput, longer recovery replay.
- **Balance:** Use `async` only when acceptable RPO; snapshot SLM for backup safety.

### 32. Bulk 429 errors

- **Diagnose:** Response body distinguishes `es_rejected_execution_exception` (queue) vs `circuit_breaking_exception` vs `cluster_block_exception`.
- **Thread pools:** `GET /_cat/thread_pool/write?v` for rejections.
- **Remediation:** Backoff clients, scale data nodes, reduce bulk size, increase `refresh_interval`, fix heap/breaker limits.
- **Indexing pressure:** Monitor `indexing_pressure` stats in node stats (7.x+).

### 33. Index buffer (`indices.memory.index_buffer_size`)

- **Role:** Shared heap buffer for indexing before flush; default 10% of heap.
- **Mixed workloads:** Lower if search-heavy starves heap; raise slightly for heavy ingest clusters with headroom.
- **Setting:** `indices.memory.index_buffer_size: 20%` cluster or per-node override; watch indexing latency and heap.

### 34. Write aliases and zero-downtime migrations

- **Pattern:** Alias `logs-write` points to current backing index; ILM rollover creates `logs-000002` and atomically moves alias.
- **Rollover:** `POST /logs-write/_rollover` with conditions `max_age`, `max_size`, `max_docs`.
- **Migration:** Reindex to new index, `POST /_aliases` remove old write alias, add to new index in one request.
- **Example:** `{"actions":[{"remove":{"index":"old","alias":"write"}},{"add":{"index":"new","alias":"write","is_write_index":true}}]}`

---

## Split Brain & Cluster Stability

### 35. Split-brain problem

- **Pre-7.x:** `discovery.zen.minimum_master_nodes` = `(master_eligible_nodes / 2) + 1` prevented two masters.
- **7.x+:** Raft-based cluster coordination; `cluster.initial_master_nodes` for bootstrap; voting configuration `GET /_cluster/voting_config_exclusions`.
- **Production scenario:** Network partition with 2 of 3 masters — minority partition cannot elect; majority continues; minority masters step down. Split-brain if misconfigured even number of masters or lost quorum causes write unavailability, not dual writes.
- **Impact:** Duplicate writes (legacy), cluster state divergence, data inconsistency — largely mitigated in 7.x with single elected master.

### 36. Network partition (3 masters → 2 + 1)

- **2-node partition:** Has quorum (if 3 total masters) — continues as cluster; accepts writes.
- **1-node partition:** Loses quorum — cannot elect master; cluster unavailable on that side.
- **Recovery:** When network heals, minority rejoins, resyncs cluster state; verify `GET /_cluster/health` GREEN; check for no stale primaries forced during partition.
- **Avoid:** Always odd number of master-eligible nodes across zones; proper timeout tuning.

### 37. Elasticsearch 7.x coordination vs Zen

- **7.x:** Built-in Raft variant (cluster coordination) — faster, safer master elections, no `minimum_master_nodes`.
- **Operational changes:** Use `cluster.initial_master_nodes` on first boot only; manage voting exclusions for master removal; upgrade all nodes to 7.x+ before removing Zen settings.
- **Simpler recovery:** Clearer master election logs; dedicated masters recommended.

### 38. 3-zone deployment without split-brain

- **Layout:** 3 master-eligible nodes, one per zone (or 3 across zones minimum); 3 data zones with awareness.
- **Config:** `cluster.routing.allocation.awareness.attributes: zone`, `node.attr.zone`, force awareness for 3 zones.
- **Survive zone loss:** Cluster remains if quorum masters survive (2 of 3); may have reduced replica redundancy until zone returns.

### 39. Slow master election

- **Diagnose:** Master logs (`org.elasticsearch.cluster.coordination`), `GET /_cluster/pending_tasks`, JVM GC on master nodes.
- **Improve:** Dedicated master nodes (no data), sufficient heap (2–4GB often enough), low latency network between masters, correct `discovery.seed_hosts`.
- **Avoid:** Heavy master load (ingest/coordinating on master); full cluster state from oversharding slows elections.

### 40. Safely remove master-eligible node

- **Voting exclusion:** `POST /_cluster/voting_config_exclusions/node_name` — safely removes node from voting config before shutdown.
- **Verify:** `GET /_cluster/state?filter_path=metadata.cluster_coordination.last_committed_config` shows new voting set.
- **After removal:** Delete exclusion when node is gone; never remove multiple masters simultaneously without exclusions.

---

## Hot-Warm Architecture

### 41. Hot-warm-cold-frozen for log analytics

- **Hot (7d, SSD):** `node.attr.data: hot`, index template `index.routing.allocation.require.data: hot`, fast ingest, 1–2 replicas.
- **Warm (30d, HDD):** ILM warm phase migrates with `allocate` requiring `data: warm`, forcemerge optional.
- **Cold (1y):** Searchable snapshots to S3 via `searchable_snapshot` repository; `frozen` tier for 7y compliance with partially mounted indices.
- **Frozen (7y):** `frozen` data tier, minimal local storage, high latency acceptable for rare queries.
- **ILM:** Chained hot → warm → cold → frozen → delete per compliance.

### 42. ILM hot-warm-cold with rollover

- **Hot phase:** Rollover `max_size: 50gb`, `max_age: 1d`; set priority, shrink optional.
- **Warm:** `min_age: 7d`, allocate to warm nodes, reduce replicas.
- **Cold:** `min_age: 37d`, searchable snapshot action.
- **Policy:** Attach via index template `"index.lifecycle.name": "logs-policy"`.

### 43. Hot-to-warm migration taking 6 hours

- **Throttle:** `cluster.routing.allocation.node_concurrent_recoveries` (default 2), increase off-peak to 4–8 cautiously.
- **Bandwidth:** `indices.recovery.max_bytes_per_sec: 100mb` during migration window.
- **Reduce data moved:** Shrink/forcemerge before warm transition; fewer shards = faster relocation.
- **Schedule:** ILM `min_age` + maintenance window; temporarily `cluster.routing.rebalance.enable: none` on other nodes if needed.

### 44. Searchable snapshots (cold/frozen)

- **Mechanism:** Snapshots mounted as indices; data fetched from object storage on demand with local cache.
- **Performance:** Cold: seconds latency; frozen: higher latency, shared cache — suitable for compliance, rare queries.
- **Decision:** Keep frequently queried data in warm/hot; archive with searchable snapshot when query SLA allows minutes.

### 45. Data tier routing

- **Node roles:** `data_hot`, `data_warm`, `data_cold`, `data_frozen` in 7.10+.
- **Index routing:** `"index.routing.allocation.include._tier_preference": "data_hot"` in template.
- **ILM:** `set_priority`, `migrate` actions move tiers automatically; verify `GET /_ilm/explain/index`.

---

## Index Lifecycle Management (ILM)

### 46. ILM stuck in "check-rollover-ready"

- **Diagnose:** `GET /index/_ilm/explain` for `step`, `phase`, `action`, `failed_step`, `reason`.
- **Common causes:** Rollover alias missing `is_write_index: true`, conditions not met, index not backing alias.
- **Fix:** Correct alias `POST /_aliases`, ensure template matches, fix `max_size`/`max_age`/`max_docs`.
- **Manual advance:** `POST /index/_ilm/move` to next phase/step after fixing, or `POST /index/_ilm/retry` on failed step.

### 47. ILM for security logging (hourly/daily rollover, 90d retention, immutable)

- **Business hours:** ILM rollover with `max_age: 1h` on hot indices via separate index template matching business-hour ingest pattern, or curator-style multiple policies.
- **Off-hours:** Daily rollover via `max_age: 1d` policy on separate data stream or cron-adjusted template.
- **Retention:** Delete phase `min_age: 90d`; frozen/searchable snapshot for immutability on object storage with WORM bucket.
- **Compliance:** Snapshot to immutable S3 with Object Lock; disable delete API via RBAC.

### 48. ILM policy updates for existing indices

- **Behavior:** Indices keep policy version at bind time until re-applied; changing policy affects new steps, not always retroactively.
- **Risks:** Accidental delete phase trigger, unexpected rollover; test in staging.
- **Safe update:** New policy version, apply to new indices via template; `_ilm/start` or move for migration of old indices deliberately.

### 49. Rollover with incorrect settings after template update

- **Cause:** New indices should use updated composable template priority; existing write index has old settings.
- **Fix:** Rollover creates new index from current template — trigger manual `POST /alias/_rollover` after template fix.
- **Wrong active index:** Reindex if settings critical (shard count wrong); shard count cannot change without reindex/shrink.
- **Prevention:** Version templates; validate with index simulate API `POST /_index_template/_simulate_index/index-name`.

### 50. ILM and alias misconfiguration

- **Rollover requires:** Write alias with exactly one `is_write_index: true` backing index.
- **Failure scenario:** Two indices marked write index — data written unpredictably; rollover doesn't fire.
- **Fix:** Atomic alias update in ILM rollover action; audit `GET /_alias/write-alias`.

---

## Snapshot & Recovery

### 51. Catastrophic failure recovery from S3

- **Prerequisites:** Register repository `PUT /_snapshot/s3_repo` (if not exists on new cluster).
- **Restore cluster:** Provision new cluster, same ES major version, register identical S3 repo path.
- **Restore order:** Cluster settings and templates (from Git/backup), then `POST /_snapshot/s3_repo/snap_name/_restore` with `indices`, `include_global_state: true` if needed.
- **Verify:** `GET /_cluster/health`, `_count` per index, spot-check documents, replica allocation.
- **Cutover:** Update application endpoints/aliases; monitor recovery `GET /_cat/recovery`.

### 52. Repository corrupted

- **Diagnose:** `GET /_snapshot/repo/_all` for partial failures; check S3 permissions, manifest files, concurrent writes to repo.
- **Valid snapshots:** List and verify `GET /_snapshot/repo/snap/_status`; test restore single index to scratch cluster.
- **Recovery:** New repository path; restore last known good snapshot; re-snapshot from restored data if repo unrecoverable.

### 53. Incremental snapshots (50TB cluster)

- **SLM:** Frequent incremental snapshots (S3 incremental at block level); snapshot only changed segments.
- **Strategy:** Daily full less frequent; hourly SLM for RPO; exclude cold/searchable snapshot indices if already in object store.
- **Cost:** S3 lifecycle to Glacier; compress where supported; monitor repo size growth.

### 54. Cross-cluster replication (CCR)

- **Config:** Remote cluster settings `cluster.remote.leader.seeds`, follower index `PUT /follower/_ccr/follow` with leader index pattern.
- **Monitoring:** `GET /follower/_ccr/stats` for lag, auto-follow patterns for new indices.
- **Failover:** Pause follower, unfollow, promote to standalone index, redirect aliases, or reverse CCR after recovery.

### 55. Snapshot lifecycle management (SLM)

- **Define:** `PUT /_slm/policy/daily-snap` with schedule, retention (`expire_after`), repository name.
- **Monitor:** `GET /_slm/policy/daily-snap`, `GET /_slm/status`, watch `unhealthy` in cluster health for failed SLM.
- **Alert:** Watcher or external monitor on last successful snapshot age.

### 56. Slow snapshot restore (5TB, 12 hours)

- **Tune:** `indices.recovery.max_bytes_per_sec`, `cluster.routing.allocation.node_concurrent_recoveries`, restore `max_restore_bytes_per_sec` if available.
- **Hardware:** Restore to cluster with high network bandwidth to S3, enough disk IOPS.
- **Parallelism:** Restore multiple indices concurrently if cluster can absorb; fewer shards per index can slow restore — balance shard size.

---

## Performance Troubleshooting

### 57. Query 100ms → 30s (index 10GB → 500GB)

- **Profile:** `"profile": true` and slow log; identify expensive aggs or scripts.
- **Optimize query:** Use filters over queries, avoid `wildcard` leading `*`, reduce `size`, use `search_after` not deep pagination.
- **Mapping:** Ensure filter fields are `keyword` with doc values; avoid scoring on log fields.
- **Architecture:** Roll ILM, add replicas for search, forcemerge old segments off-peak (careful), increase shard count only via reindex if shards too large (>50GB).

### 58. `_explain` and `_profile` together

- **`_explain`:** `GET /index/_explain/doc_id` with query body — shows why document matches or not per clause.
- **`_profile`:** Shows time per shard per component.
- **Workflow:** Explain failing match, profile slow clauses, fix bool structure or field types.

### 59. Fielddata memory issue (high-cardinality text aggs)

- **Fix:** Reindex with `keyword` subfield; aggregate on `.keyword` not analyzed `text`.
- **Immediate:** Clear fielddata `POST /index/_cache/clear?fielddata=true` (disruptive).
- **Prevent:** `fielddata: false` on text; use `doc_values` on keyword.

### 60. "Rejected execution" on search

- **Thread pool:** `GET /_cat/thread_pool/search?v` — queue full vs active threads.
- **Circuit breaker:** Check response for `circuit_breaking_exception`.
- **Resource:** CPU saturation, GC, disk — scale or reduce query load.
- **Fix:** Optimize queries, add coordinating nodes, increase replicas, raise queue cautiously.

### 61. Slow log

- **Enable:** `index.search.slowlog.threshold.query.warn: 10s`, `index.indexing.slowlog.threshold.index.warn: 5s` in index settings or template.
- **Analysis:** Parse logs for query JSON, identify regex, script, or aggregation cost; correlate with index growth.
- **Tooling:** Ship to Kibana Observability or ELK for aggregation of slow query patterns.

### 62. Complex nested aggregation CPU spike

- **Reduce scope:** `filter` aggregation wrapping nested path; limit `size` and `shard_size`.
- **Mapping:** Flatten where possible; `nested` only when required.
- **Parallelism:** Spread across replicas with ARS; add data nodes.
- **Alternative:** Transform jobs to pre-aggregate metrics into summary indices.

---

## Kubernetes Deployments

### 63. ECK data node OOMKilled (memory pressure)

- **Requests/limits:** Set memory limit ≥ heap + 25–40% overhead; requests equal limits for predictable scheduling.
- **JVM:** `ES_JAVA_OPTS=-Xms16g -Xmx16g` with 32Gi limit typical starting point.
- **ECK:** `spec.nodeSets[].podTemplate.spec.containers[].resources` and `config.node.store.allow_mmap: false` only if required on restricted kernels.

### 64. Persistent storage on Kubernetes

- **StorageClass:** SSD for hot (`gp3`, `premium-rwo`); expandable volumes where supported.
- **PVC:** One PVC per data node via `volumeClaimTemplates`; retain policy on StatefulSet delete for safety.
- **Rescheduling:** ECK manages StatefulSet identity; pods reattach same PVC by name.

### 65. Rolling upgrades with ECK (GREEN throughout)

- **ECK orchestration:** `spec.version` bump triggers rolling restart respecting shard allocation.
- **Practice:** Upgrade masters first pattern handled by operator; monitor `GET /_cluster/health` during rollout.
- **Pause:** Use ECK mutation or reduce concurrent recoveries if yellow persists too long.

### 66. Node discovery in Kubernetes

- **ECK:** Configures `discovery.seed_hosts` and initial master nodes via headless services; certificates for transport.
- **Pod reschedule:** New IP — operator updates endpoints; nodes rejoin via published addresses.
- **Manual:** `discovery.seed_hosts: elasticsearch-discovery.namespace.svc` for self-managed.

### 67. K8s vs bare-metal challenges

- **Network:** Overlay CNI adds latency vs bare metal; hostNetwork sometimes used (trade-offs).
- **Storage:** Local SSD vs network-attached latency; mmap and IOPS limits.
- **JVM:** cgroup awareness required; memory limits must include off-heap.
- **Noisy neighbors:** Shared nodes — use dedicated node pools/taints for Elasticsearch.

### 68. Monitoring on Kubernetes

- **Metricbeat:** `metricbeat modules enable elasticsearch` with autodiscover for ECK.
- **Prometheus:** elasticsearch-exporter scraping `/_prometheus/metrics` (Elastic 8.x) or exporter sidecar.
- **Metrics:** JVM heap/GC, cluster health, indexing/search rates, thread pools, ILM/SLM status.

### 69. Intermittent network partitions (CNI)

- **Timeouts:** Tune `cluster.fault_detection.leader_check.timeout`, `follower_check.timeout` (carefully, not too high).
- **Publish timeout:** Increase publish timeout for slow networks temporarily.
- **Resilience:** Odd master count, multi-AZ, dedicated masters; fix CNI root cause (MTU, conntrack).

---

## Security & Compliance

### 70. RBAC multi-tenant

- **Roles:** Per-team roles with `indices` privileges scoped to `team-a-*` patterns; `read`, `write`, `create_index` as needed.
- **Users:** Native or SAML realm mapping groups to roles.
- **API:** `POST /_security/role/team_a` with `"indices": [{"names": ["team-a-*"], "privileges": ["read","write"]}]`.
- **Isolation:** Separate index prefixes or data streams per tenant; never shared indices without DLS.

### 71. Field-level and document-level security

- **FLS:** Role excludes fields `"-patient.ssn"` or grants subset of fields for role `nurse`.
- **DLS:** Query template restricting documents `"term": {"department": "{{_user.metadata.department}}"}}`.
- **Healthcare:** Combine with audit logging, encryption at rest, TLS in transit.

### 72. Audit logging (SOC 2, GDPR)

- **Enable:** `xpack.security.audit.enabled: true` with `audit.logs` to file or index.
- **Events:** Authentication, access denied, index operations, security config changes.
- **Retention:** Immutable log store; ship to SIEM; tie to access reviews for GDPR Article 30 records.

### 73. TLS transport and HTTP with rotation

- **Transport:** Node-to-node on port 9300 with CA-signed certs; `xpack.security.transport.ssl.*`.
- **HTTP:** Client-to-node 9200 with same or separate cert.
- **Rotation:** Rolling cert update on nodes one at a time; use same CA; reload via API or rolling restart without full cluster downtime if dual-trust window configured.

---

## Advanced Scenarios

### 74. Real-time product catalog (100M docs, sub-100ms)

- **Index design:** Single product index per region or custom routing by `category_id`; 3–5 primary shards per 30–50GB, 1–2 replicas.
- **Mapping:** `keyword` for filters/SKU, `text` with multi-field for search; `doc_values` on sort fields; avoid nested where flattening works.
- **Query:** Bool filter cache-friendly structure; `dis_max` or `multi_match` with `best_fields`; minimal `_source` filtering.
- **Caching:** Request cache for facet aggs; adequate heap for OS cache; dedicated coordinating nodes.
- **Hardware:** SSD, 16–31GB heap, separate master tier; CDN for static facets if applicable.

### 75. Percolator for real-time alerting

- **Store queries:** Index percolator queries with `percolator` field mapping matching indexed document shape.
- **On ingest:** `POST /_bulk` with `"percolate": {"index": "alerts"}, "document": {...}}` or percolate API on index pipeline.
- **Use case:** Security rules, inventory alerts when new docs match saved queries.

### 76. Multi-cluster architecture (CCS)

- **Config:** `PUT /_cluster/settings` remote clusters `cluster.remote.us.seeds`.
- **Query:** `GET /us:index,local:index/_search` with cross-cluster search.
- **Latency:** Minimize cross-region CCS; replicate hot data locally; async CCR for DR.
- **Failure:** Handle remote cluster unavailable — partial results settings and fallbacks in application.

### 77. Vector search (kNN)

- **Mapping:** `"type": "dense_vector", "dims": 384, "index": true, "similarity": "cosine"`.
- **Search:** kNN query with `"k": 10`, `"num_candidates": 100` for HNSW recall/speed tradeoff.
- **Tuning:** Smaller `dims` if possible, adequate heap, force merge for segment optimization, filter pre-filter before kNN for performance.

### 78. SIEM backend (1B events/day, sub-second search)

- **Design:** Data streams + ILM hot/warm/cold; 1–2 shards per 30–50GB rollover; ingest pipelines for ECS normalization.
- **Ingest:** Dedicated ingest nodes, bulk tuning, `refresh_interval: 5s` on hot.
- **Query:** Filter-first (time range + host.name), avoid full-text on all fields; runtime fields sparingly.
- **Tiering:** Frozen/searchable snapshots for old data; hot cluster sized for 7–14 day working set at sub-second SLA.

---

## Basic Questions

### 1. What is Elasticsearch and what type of data does it store?

Elasticsearch is a distributed search and analytics engine built on Apache Lucene. It stores JSON documents in indices, supporting full-text search, structured filtering, aggregations, and log/metrics analytics at scale.

### 2. What is the difference between an Elasticsearch index, document, and field?

An index is a logical collection of documents (like a database table). A document is a single JSON record with a unique `_id`. A field is a key-value pair within a document, typed via mapping (e.g., `keyword`, `text`, `date`).

### 3. What is a shard in Elasticsearch and why does it exist?

A shard is a Lucene index partition; each index splits into primary shards for horizontal scale. Shards distribute data and query load across nodes in a cluster.

### 4. What is the difference between a primary shard and a replica shard?

A primary shard holds the authoritative copy of a subset of an index's data. Replica shards are copies of primaries that provide redundancy and can serve read requests; there is exactly one primary per shard id.

### 5. What is the Elasticsearch REST API and how do you use it to index a document?

The REST API exposes HTTP endpoints on port 9200 for all operations. Index a document with `PUT /my-index/_doc/1` and JSON body, or `POST /my-index/_doc` for auto-generated id.

### 6. What is a mapping in Elasticsearch and why is it important?

Mapping defines field types, analyzers, and indexing options for an index. It is important because incorrect types (e.g., `text` vs `keyword`) affect search behavior, aggregations, and storage efficiency.

### 7. What is the difference between `keyword` and `text` field types?

`text` fields are analyzed for full-text search (tokenized, lowercased). `keyword` fields are stored as single exact values for filtering, sorting, and aggregations without analysis.

### 8. How do you perform a basic full-text search in Elasticsearch?

`GET /my-index/_search` with a query body such as `{"query": {"match": {"title": "elasticsearch guide"}}}` returns scored matching documents.

### 9. What is the Elasticsearch cluster health status (green, yellow, red)?

GREEN means all primary and replica shards are assigned. YELLOW means all primaries are assigned but some replicas are not. RED means at least one primary shard is unassigned, risking data loss for that shard.

### 10. What is the purpose of the `_id` field in an Elasticsearch document?

`_id` uniquely identifies a document within an index. You supply it on index or let Elasticsearch auto-generate it; it is used for GET, UPDATE, and DELETE operations.

### 11. How do you create an index in Elasticsearch with custom settings?

`PUT /my-index` with a body containing `"settings": {"number_of_shards": 3, "number_of_replicas": 1}` and optional `"mappings": {...}`.

### 12. What is a Kibana index pattern and how does it relate to Elasticsearch indices?

An index pattern (or data view in newer Kibana) is a Kibana configuration matching one or more Elasticsearch indices by name pattern. It tells Kibana which fields exist for visualizations and Discover.

### 13. What is the purpose of the `_source` field in an Elasticsearch document?

`_source` stores the original JSON document as indexed. It is returned in search hits and used for reindexing, updates, and scripts; can be disabled to save space if not needed.

### 14. How do you delete a document from an Elasticsearch index?

`DELETE /my-index/_doc/1` removes the document with id `1`. Returns `result: deleted` on success.

### 15. What is the difference between Elasticsearch's `match` and `term` queries?

`match` analyzes the search term and applies full-text scoring on analyzed fields. `term` looks for exact values without analysis, used on `keyword`, numeric, and date fields.

### 16. What is an Elasticsearch alias and when would you use one?

An alias is a logical name pointing to one or more indices. Use aliases for zero-downtime reindexing, ILM rollover write pointers, and abstracting index names from applications.

### 17. How do you check the status of an Elasticsearch cluster using the API?

`GET /_cluster/health` returns status, node count, and shard counts. `GET /_cat/health?v` provides a compact tabular view.

### 18. What is the purpose of the `refresh_interval` setting?

It controls how often a shard makes new documents visible to search (near-real-time). Longer intervals improve indexing throughput; shorter intervals improve search freshness.

### 19. How do you perform a bulk indexing operation in Elasticsearch?

`POST /_bulk` with newline-delimited JSON action lines: `{"index":{"_index":"logs"}}` followed by document JSON per line, ending with a newline.

### 20. What is the difference between Elasticsearch's `filter` and `query` context?

Query context calculates relevance scores. Filter context answers yes/no without scoring and caches bitsets, making filters faster for exact matches and ranges.

### 21. What is an Elasticsearch aggregation and give two examples?

Aggregations compute analytics over search results. Examples: `terms` aggregation for facet counts by category, and `avg` aggregation for average price.

### 22. How do you configure the number of shards and replicas when creating an index?

Set `"number_of_shards"` and `"number_of_replicas"` in index settings at creation time via `PUT /index`. Shard count cannot be changed without shrink/reindex.

### 23. What is the purpose of the `_cat` API?

The `_cat` API returns human-readable tabular cluster information (health, indices, shards, nodes) for quick operational inspection in terminals.

### 24. How do you update a document using the Update API?

`POST /my-index/_update/1` with `"doc": {"field": "new_value"}` merges partial fields, or use a script in the `script` block for computed updates.

### 25. What is the difference between Elasticsearch's `index` and `create` operations?

`index` indexes or overwrites a document with a given id. `create` fails with version conflict if a document with that id already exists, ensuring insert-only semantics.

### 26. What is an Elasticsearch template and when would you use one?

Index templates (or composable index templates) automatically apply settings and mappings to new indices matching a pattern. Use them for consistent log/metrics index configuration and ILM policies.

### 27. How do you search across multiple indices?

`GET /index1,index2/_search` or `GET /logs-*/_search` with wildcard patterns in the URL path.

### 28. What is the purpose of the `took` field in a search response?

`took` reports milliseconds spent executing the search on the coordinating node (not including network serialization or client-side time).

### 29. How do you configure Elasticsearch's JVM heap size?

Set `ES_JAVA_OPTS="-Xms16g -Xmx16g"` environment variable or `jvm.options` file, keeping heap ≤50% of RAM and typically ≤31GB for compressed OOPs.

### 30. What is the difference between `must`, `should`, and `must_not` clauses?

In a `bool` query, `must` clauses are required and contribute to score, `should` clauses are optional (boost score when matched), and `must_not` clauses must not match (filter context, no score).

### 31. What is the purpose of the `_version` field?

`_version` tracks document changes for optimistic concurrency control. Updates can specify `if_seq_no` and `if_primary_term` (preferred in 7.x+) or legacy `version` to prevent lost updates.

### 32. How do you use the `range` query for date-based filtering?

`{"query": {"range": {"@timestamp": {"gte": "now-1d", "lte": "now"}}}}` filters documents within a date range; ensure the field is mapped as `date`.

### 33. What is an Elasticsearch node role? Name four common roles.

Node roles define node responsibilities. Common roles: `master` (cluster state), `data` (stores shards), `ingest` (pipelines), `coordinating_only` (routes requests, no data).

### 34. How do you check which node is the master?

`GET /_cat/nodes?v&h=name,node.role,master` shows `*` in the master column, or `GET /_cluster/state?filter_path=master_node,nodes.*.name`.

### 35. What is the purpose of `index.number_of_replicas`?

It sets how many replica copies exist per primary shard. More replicas improve read capacity and availability but increase storage and indexing overhead.

### 36. How do you use the `terms` aggregation for faceted search?

`"aggs": {"categories": {"terms": {"field": "category.keyword", "size": 10}}}` returns bucket counts per unique category value for faceted navigation.

### 37. What is the difference between `match_phrase` and `match` queries?

`match` finds documents containing analyzed terms in any order. `match_phrase` requires terms to appear in the same order with optional `slop` for word distance.

### 38. How do you configure network settings for cluster communication?

Set `network.host`, `transport.port` (9300), and `discovery.seed_hosts` / `cluster.initial_master_nodes` in `elasticsearch.yml` for node binding and discovery.

### 39. What is the purpose of the `_shards` field in a search response?

`_shards` reports how many shards were queried, successful, skipped, and failed, helping diagnose partial search results during cluster issues.

### 40. How do you use the `exists` query?

`{"query": {"exists": {"field": "email"}}}` returns documents where the field is present and not null (with index-time existence).

### 41. What is the `_analyze` API used for?

`POST /_analyze` tests how analyzers tokenize text without indexing, useful for debugging custom analyzers and search behavior.

### 42. How do you configure index refresh to improve indexing performance?

Increase `index.refresh_interval` (e.g., `30s` or `-1` to disable temporarily) via `PUT /index/_settings` during bulk loads, then restore to `1s` after.

### 43. What is the purpose of the `doc_values` setting?

`doc_values` stores columnar on-disk structures for sorting, aggregations, and scripting. Enabled by default on `keyword`, numeric, and date fields; more efficient than fielddata.

### 44. How do you use the `scroll` API for large result sets?

Initial search with `"size": 1000, "scroll": "2m"` returns `_scroll_id`; subsequent `POST /_search/scroll` with that id retrieves batches until exhausted. Not recommended for user-facing pagination.

### 45. What is the difference between `index` and `search` in terms of consistency?

After indexing, documents are searchable after the next refresh (default ~1s). Search is near-real-time, not immediately consistent; use `refresh=wait_for` on write for stricter visibility.

---

## Intermediate Questions

### 1. ILM policy for log analytics (hot-warm-cold)

Define a policy with hot phase (rollover on `max_size`/`max_age`, high priority), warm phase (`min_age: 7d`, allocate to warm nodes, optional forcemerge), and cold phase (`min_age: 30d`, searchable snapshot or migrate to cold tier). Attach via composable index template to `logs-*` data streams with `index.lifecycle.name`.

### 2. Shard allocation awareness (multi-zone)

Set `node.attr.zone` on each node and `cluster.routing.allocation.awareness.attributes: zone`. Use `cluster.routing.allocation.awareness.force.zone.values` to list all zones. Replicas automatically place in different zones than primaries.

### 3. Cross-cluster search (CCS)

Register remote cluster: `PUT /_cluster/settings` with `"cluster.remote.cluster_b.seeds": ["node:9300"]`. Query with `GET /cluster_b:index-*/_search` prefixing remote cluster name to index expression.

### 4. Circuit breakers and tuning

Breakers limit fielddata, request, and in-flight memory on the JVM heap. Tune `indices.breaker.fielddata.limit`, `indices.breaker.request.limit`, and parent breaker; fix queries to use `keyword`/doc values rather than only raising limits.

### 5. Field-level security

Create roles with `field_security` granting or excluding fields: `"grant": {"field1": ["read"]}, "except": ["ssn"]`. Assign roles to users via SAML/LDAP realm for role-based field visibility.

### 6. SLM for automated S3 backups

Register S3 repository, then `PUT /_slm/policy/nightly` with `"schedule": "0 1 * * *"`, `"repository": "s3_backup"`, `"retention": {"expire_after": "30d"}`. Monitor with `GET /_slm/status`.

### 7. Custom analyzer (multilingual)

Define in index settings: `"analysis": {"analyzer": {"multilingual": {"tokenizer": "standard", "filter": ["lowercase", "german_normalization", "french_stem"]}}}`. Apply via `analyzer` on `text` fields in mapping.

### 8. `_reindex` API

`POST /_reindex` copies documents from source to destination index with optional script transforms. Use for mapping changes, shard count changes, or cluster migrations; supports `slices` for parallelism and throttling.

### 9. Percolator for real-time matching

Store queries in a percolator index, then on ingest call percolate API or use ingest processor to match incoming documents against stored queries for alerting workflows.

### 10. Hot-warm architecture for time-series logs

Tag nodes `node.attr.data: hot|warm`, configure ILM to allocate hot indices to hot nodes and migrate to warm after age threshold. Use index templates with tier preferences and separate hardware profiles.

### 11. Rollup jobs

Rollup aggregates historical metrics into summary indices on schedule, reducing storage for dashboards. Define via `PUT /_rollup/job/metrics` with date histogram and metric aggregations on source index pattern.

### 12. `search_after` vs `from/size`

`from/size` paginates with offset but is limited by `index.max_result_window` (default 10000) and is inefficient for deep pages. `search_after` uses sort values from previous page for stable, efficient deep pagination.

### 13. Runtime fields

Define in search request or index mapping: `"runtime": {"day_of_week": {"type": "keyword", "script": "emit(doc['@timestamp'].value.dayOfWeekEnum.getDisplayName(...))"}}`. Computed at query time without reindexing.

### 14. Cross-cluster replication (CCR)

Configure remote cluster, create follower index with auto-follow pattern or manual follow API. Monitor lag via `_ccr/stats`; failover by unfollowing and promoting follower index.

### 15. Data streams

Data streams abstract backing indices for time-series; write to `logs-datastream` with matching template and ILM. Elasticsearch manages rollovers automatically; simplifies ingest and lifecycle for logs/metrics.

### 16. `_bulk` partial failures

Bulk response includes `errors: true` with per-item `status` and `error` objects. Implement client retry only for retryable failures (429, 503), with exponential backoff; log and dead-letter permanent failures (400 mapping errors).

### 17. Function score query

`function_score` wraps a query and applies functions (weight, field_value_factor, decay) to modify relevance. Use for boosting recent documents, popular items, or business rules in search ranking.

### 18. Composable index templates

Split shared settings/mappings into component templates (`PUT /_component_template/logs-mappings`), compose via index template `composed_of` array for modular, reusable configuration across teams.

### 19. Geo-spatial queries

Map `geo_point` fields, then use `geo_distance`, `geo_bounding_box`, or `geo_shape` queries. Example: `{"geo_distance": {"distance": "10km", "location": {"lat": 40, "lon": -74}}}`.

### 20. `significant_terms` aggregation

Finds terms that appear more frequently in the search result set than in the background corpus — useful for anomaly detection and "unusual terms" in a subset of logs.

### 21. Nested vs parent-child

`nested` types index arrays of objects as separate hidden Lucene documents, keeping object integrity for queries. Parent-child (`join` field) relates separate documents across shards — use when children update independently at scale; nested is simpler for contained arrays.

### 22. Ingest pipelines

`PUT /_ingest/pipeline/enrich-geo` with processors (`grok`, `set`, `geoip`, `rename`) applied at index time via `pipeline` query param or default pipeline on index template.

### 23. Collapse feature

`"collapse": {"field": "user.id.keyword"}` returns top hit per unique field value for deduplicating results (e.g., one product per brand in search results).

### 24. Adaptive replica selection (ARS)

Coordinating nodes route search requests to replicas with lower queue depth and better response times using adaptive statistics, improving load distribution vs round-robin.

### 25. Transform API

Scheduled jobs pivot or aggregate source indices into destination summary indices. `PUT /_transform/logs-summary` with `source`, `dest`, and `pivot` aggregation for KPI dashboards.

### 26. TLS for transport and HTTP

Configure `xpack.security.transport.ssl.enabled` and `http.ssl` with certificate paths in `elasticsearch.yml`. Use same CA for all nodes; enable verification mode `certificate` for mutual TLS.

### 27. `more_like_this` query

`more_like_this` finds documents similar to given text or document ids based on term frequency — used for recommendations and "related content" features.

### 28. `_profile` API for slow queries

Add `"profile": true` to search requests to get hierarchical timing of queries and aggregations per shard, identifying bottlenecks without guessing.

### 29. Watcher for alerting

Watcher (or Elastic Rules in Kibana) runs scheduled queries and triggers actions (email, Slack, PagerDuty) when conditions match — e.g., error rate threshold exceeded.

### 30. Index sorting configuration

Set at index creation: `"settings": {"index.sort.field": ["@timestamp"], "index.sort.order": ["desc"]}` to optimize range queries and sorted retrieval on those fields.

### 31. `terms_enum` API for autocomplete

`POST /index/_terms_enum` efficiently iterates unique terms for a field with prefix filter — designed for autocomplete UIs without expensive aggregation scans.

### 32. `_field_caps` API

`GET /index/_field_caps?fields=*` returns field types and capabilities (searchable, aggregatable) across indices for schema discovery and UI field pickers.

### 33. kNN vector search

Index `dense_vector` fields with HNSW index enabled; query with `knn` clause specifying `field`, `query_vector`, `k`, and `num_candidates` for approximate nearest neighbor search.

### 34. Slow log configuration

Set `index.search.slowlog.threshold.query.warn` and `index.indexing.slowlog.threshold.index.warn` in templates. Ship logs to analysis platform; tune thresholds to capture p99 outliers without noise.

### 35. `enrich` processor

Create enrich policy matching reference data (e.g., IP → geo), execute policy, then use `enrich` processor in ingest pipeline to add fields at index time based on lookup keys.

### 36. `_cluster/reroute` API

`POST /_cluster/reroute` with commands to `move`, `allocate`, or `cancel` shard allocation — for manual recovery, maintenance, or forcing shard relocation after allocation failures.

### 37. Filtered aliases for multi-tenant isolation

`POST /_aliases` with `"filter": {"term": {"tenant_id": "acme"}}` on alias `acme-logs` pointing to shared index — applications query alias and see only tenant documents (combine with RBAC for security).

### 38. `index.routing.allocation.require`

Per-index setting like `"index.routing.allocation.require.data": "hot"` forces shards onto nodes with matching attribute — dedicates indices to specific hardware tiers.

### 39. Point-in-time (PIT) API

`POST /index/_pit?keep_alive=5m` opens consistent snapshot for pagination; use with `search_after` and `pit.id` in subsequent searches for consistent exports without scroll limitations.

### 40. `_cat/thread_pool` API

`GET /_cat/thread_pool/search,write?v` shows active, queue, and rejected counts per node — first check for thread pool saturation during performance incidents.

---

## Advanced Questions

### 1. Global e-commerce (10B searches/day, sub-100ms)

- **Sharding:** Category-based custom routing or multiple regional clusters with CCS; 20–50 GB shards, adequate replicas per AZ.
- **Mapping:** `keyword` filters, minimal analyzed fields, `copy_to` search-as-you-type where needed.
- **Caching:** Request cache for facet queries, CDN for static content, coordinating-only tier.
- **Query:** Bool filter-first, `dis_max` multi_match, avoid deep pagination; precomputed suggest indices.
- **Ops:** Hot tier SSD, ARS, autoscaling ingest nodes, SLO monitoring on p99 search latency.

### 2. Multi-tenant SaaS (1,000 customers)

- **Isolation:** Separate index prefix or data stream per tenant, or shared index with DLS + filtered aliases.
- **RBAC:** Per-tenant roles; API keys scoped to tenant indices.
- **Limits:** `xpack.security` application privileges, ingest pipelines tagging tenant, rate limiting at proxy.
- **Provisioning:** Terraform/API automation creating index template + ILM per tenant on signup.

### 3. Petabyte log analytics (2-year retention, tiered storage)

- **Hot:** Recent 7–14 days on SSD with ILM rollover at 50 GB.
- **Warm/cold:** Migrate to HDD then searchable snapshots on S3.
- **Frozen:** Compliance tier for infrequent access; delete phase after 2 years.
- **Cost:** SLM to object storage, compression, rollup for long-term metrics dashboards.

### 4. Zero-downtime major version upgrade (500TB, 24/7)

- **Rolling upgrade:** Upgrade non-master nodes one at a time; verify compatibility matrix.
- **Reindex:** Cross-cluster replication or reindex for breaking mapping changes across major versions.
- **Validation:** Dual-cluster with reindex + alias cutover for highest safety; feature flags on clients.
- **Rollback:** Snapshot before upgrade; maintain previous cluster until soak period ends.

### 5. Real-time fraud detection (1M tx/min, sub-second)

- **Ingest:** Dedicated ingest/coordinating tier, bulk with `refresh_interval: 1s` or faster on hot indices.
- **Detection:** Percolator or streaming aggregations (Transforms), ML jobs for anomaly scores.
- **Query:** Filter on `user_id`, `amount`, time window; pre-indexed risk features.
- **Latency:** Hot data on NVMe, minimal shard count per index for routing efficiency.

### 6. Auto-rebalancing without search impact

- **ILM tiering:** Proactive migration before nodes saturate.
- **Throttling:** `indices.recovery.max_bytes_per_sec`, concurrent recovery limits during business hours.
- **Shard sizing:** Right-size at creation to reduce constant rebalancing.
- **Autoscaling:** Add nodes before rebalance storms; use index allocation filters for new shards.

### 7. Vector search at scale (100M items, real-time updates)

- **HNSW:** `dense_vector` with `index: true`, tune `ef_construction` and `m` at index time.
- **Sharding:** Split by category; filter before kNN to reduce candidate set.
- **Updates:** Frequent updates resegment — balance refresh interval; consider dedicated vector tier.
- **Hybrid:** Combine BM25 bool filter with kNN for relevance + performance.

### 8. Survive AZ failure (zero data loss, 60s failover)

- **Topology:** 3 AZs, awareness attributes, `number_of_replicas: 2` minimum for zone loss tolerance.
- **Masters:** 3 master-eligible across AZs; quorum survives one AZ loss.
- **RTO:** Automated DNS/load balancer shift to healthy region; CCR for cross-region DR if required.
- **Testing:** Regular AZ isolation game days.

### 9. Security analytics / APT correlation (50 log sources)

- **Schema:** ECS normalization via ingest pipelines; data streams per source type with unified destination.
- **Correlation:** Enrich processor, entity-centric indices (user, host), graph-like joins via multiple queries or EQL.
- **Performance:** Hot tier for 30-day window, cold for historical hunt; saved searches and detection rules.

### 10. Healthcare HIPAA deployment

- **Encryption:** TLS in transit, encryption at rest (KMS), field-level security for PHI.
- **Access:** RBAC, DLS, audit logs to immutable store.
- **Retention:** ILM with legal hold indices; SLM to compliant object storage.
- **BAA:** Elastic Cloud or self-managed with documented controls and access reviews.

### 11. Cross-cluster search with data sovereignty

- **Regional clusters:** Data stays in-region; CCS queries remote cluster without copying data.
- **Query routing:** Application routes EU users to EU cluster alias only.
- **Governance:** Block CCS at network policy level for prohibited cross-border indices.

### 12. 100k concurrent searches, sub-200ms

- **Scale:** Many data nodes, sufficient replicas, coordinating tier autoscaling.
- **Caching:** Request cache for repeated aggs, shard request cache warming.
- **Coalescing:** Search thread pool tuning, ARS, connection pooling on clients.
- **Query design:** Filter-first, minimal `_source`, reject expensive ad-hoc queries at API gateway.

### 13. Observability platform (10k microservices)

- **Data streams:** Logs, metrics, traces with ECS; ILM hot/warm/cold.
- **Correlation:** `trace.id`, `service.name` fields; runtime fields sparingly.
- **Anomaly:** ML jobs on metric indices, unified Kibana Observability or Grafana.
- **Scale:** Ingest nodes, fleet-managed agents, cross-cluster search for global view.

### 14. Auto-detect and remediate shard imbalances

- **Monitoring:** Alert on disk skew, shard count per node, hotspot thread pools.
- **Automation:** Curator/ILM, Elastic autoscaling API, custom operator calling allocation explain + reroute APIs.
- **Self-heal:** Expand disk, add node, or `_cluster/reroute` move from overloaded node with throttling.

### 15. ML anomaly detection (time-series)

- **Jobs:** `PUT /_ml/anomaly_detectors/job` on metric indices with detectors on key fields.
- **Validation:** Model plot in Kibana, adjust `bucket_span`, influencers, and training window.
- **Alerts:** Watcher/rules on anomaly scores integrated with on-call.

### 16. Financial trading (10M events/s, microsecond indexing)

- **Architecture:** Multiple dedicated indexing clusters, minimal analysis, optimized mappings.
- **Hardware:** Bare metal, local NVMe, kernel tuning, huge pages where appropriate.
- **Elasticsearch limits:** May require tiered approach — hot ES for recent, other systems for ultra-low latency tick data.

### 17. CMS multilingual search (1B documents)

- **Analyzers:** Per-language analyzers with `language` field and `match` on appropriate subfield.
- **Facets:** `keyword` + `terms` aggs; `collapse` for duplicate content variants.
- **Ranking:** `function_score` for personalization signals; separate suggest index.

### 18. Horizontal autoscaling with zero-downtime

- **Orchestration:** ECK/K8s HPA on CPU/heap; add data nodes via StatefulSet scale.
- **Shards:** New nodes receive rebalanced shards throttled off-peak.
- **Coordination:** Scale coordinating tier independently from data tier.

### 19. Zero-trust security architecture

- **mTLS:** Transport and HTTP mutual TLS with short-lived certs via SPIFFE or internal CA rotation.
- **RBAC:** Least privilege roles, API key per service.
- **FLS/DLS:** Sensitive field and row restrictions.
- **Audit:** Full audit trail to SIEM; no anonymous access.

### 20. Time-series (1M docs/s write + fast aggs)

- **Write path:** Data streams, bulk ingest, `refresh_interval: 30s`, minimal replicas during ingest.
- **Read path:** Rollup/transform summary indices for dashboards; hot tier sized for query window.
- **Shards:** Target 30–50 GB rollovers; avoid oversharding.

### 21. Log classification with ML routing

- **Ingest:** Ingest pipeline with `inference` processor for classification.
- **Routing:** `routing` or separate indices per classification via pipeline `set` + conditional processors.
- **Workflows:** Watcher triggers ticketing based on classified severity.

### 22. Multi-cloud Elasticsearch (AWS, GCP, Azure)

- **Strategy:** Cluster per cloud for data locality; CCR between clouds for DR.
- **Networking:** PrivateLink/VPN peering; CCS for federated search.
- **Consistency:** Same ES version, templates from GitOps across clouds.

### 23. Snapshot strategy (100TB, 1h RPO, 4h RTO)

- **SLM:** Hourly snapshots with incremental S3 storage; retain 24h minimum.
- **RTO:** Pre-provisioned restore cluster IaC; documented restore runbook tested quarterly.
- **Validation:** Automated restore test of canary index monthly.

### 24. Real-time search + historical analytics (tiered storage)

- **Hot:** Recent data sub-second freshness with normal indices.
- **Cold/frozen:** Searchable snapshots for years of history at lower cost.
- **Query:** Application routes time range — hot cluster for last 7d, CCS to cold for historical.

### 25. Customer 360 (500M customers, 20 sources)

- **Ingest:** Enrich processor and transforms to unify profiles in `customer-360` index.
- **ID strategy:** Global `customer_id` keyword as join key; periodic full sync + real-time updates.
- **Search:** Single alias `customer` with nested relationships where needed; strict RBAC per business unit.

---

## Rapid-Fire Questions

### 1. Default primary shards in Elasticsearch 7.x+

**1** primary shard per new index (changed from 5 in 6.x).

### 2. RED cluster health

At least one **primary** shard is unassigned — data for that shard may be unavailable.

### 3. Purpose of `_cat/health`

Human-readable cluster health summary (status, nodes, shards) for quick CLI inspection.

### 4. `keyword` vs `text`

`keyword` = exact, not analyzed; `text` = analyzed for full-text search.

### 5. `GET /_cluster/stats`

Aggregated cluster statistics: indices count, docs, store size, JVM, OS, and shard summary.

### 6. `index.refresh_interval`

Controls how often refreshes make new docs visible to search (default `1s`).

### 7. YELLOW cluster health

All primaries assigned; **some replicas** are unassigned (often single-node cluster).

### 8. `_bulk` API

High-throughput batch indexing/updating/deleting of many documents in one HTTP request.

### 9. `doc_values`

On-disk columnar storage for sorting, aggregations, and scripting (default on most structured types).

### 10. `GET /_cat/indices?v`

Table of indices with health, docs count, store size, and shard counts.

### 11. `query` vs `filter` context

Query scores relevance; filter is yes/no, cacheable, no scoring.

### 12. `_source`

Stores the original JSON document returned in search hits.

### 13. `GET /_nodes/stats`

Per-node statistics: JVM, OS, indices, thread pools, filesystem, and circuit breakers.

### 14. `fielddata` cache

Legacy in-heap cache for analyzed field ordinals; largely replaced by `doc_values` on `keyword`.

### 15. `index.number_of_replicas`

Number of replica copies per primary shard.

### 16. `POST /index/_forcemerge`

Merges segments into fewer segments; reduces segments but is I/O heavy and blocks writes on that index.

### 17. `match` vs `term`

`match` = analyzed full-text; `term` = exact value match.

### 18. `_id`

Unique document identifier within an index.

### 19. `GET /_cat/shards?v`

Lists every shard with state, node, and index.

### 20. Translog

Write-ahead log ensuring durability of operations not yet committed to Lucene segments.

### 21. `index.blocks.write`

Blocks all write operations on the index (required before shrink/snapshot consistency).

### 22. `GET /_cluster/allocation/explain`

Explains why a shard is unassigned or which node was chosen for allocation.

### 23. `index` vs `create`

`index` upserts; `create` fails if document id already exists.

### 24. `_version`

Document version for optimistic concurrency control.

### 25. `POST /index/_refresh`

Makes recent writes visible to search without waiting for scheduled refresh.

### 26. Circuit breaker

Prevents operations from exhausting JVM heap by rejecting requests before OOM.

### 27. `index.routing.allocation.require`

Requires shards to allocate only on nodes matching specified attributes.

### 28. `GET /_cat/nodes?v`

Lists cluster nodes with roles, load, and disk usage.

### 29. `scroll` vs `search_after`

`scroll` keeps search context for bulk export (deprecated for app pagination); `search_after` is stateless efficient deep pagination.

### 30. `index.merge.policy.max_merge_at_once`

Maximum segments merged in one merge operation.

### 31. `GET /_cluster/pending_tasks`

Lists cluster state update tasks queued on the master.

### 32. `_reindex` API

Copies documents from one index to another with optional transforms.

### 33. `index.store.type`

Storage implementation (e.g., `fs`, `niofs`; hybrid storage in newer versions for tiered).

### 34. `GET /_nodes/hot_threads`

Shows busiest JVM threads with stack traces for performance debugging.

### 35. `nested` vs `object`

`object` flattens arrays internally; `nested` indexes array elements as separate docs preserving array element boundaries.

### 36. `index.max_result_window`

Maximum `from + size` for pagination (default 10000).

### 37. `POST /index/_shrink/new_index`

Reduces shard count by merging shards on a single node into a new smaller-shard index.

### 38. `_pit` API

Point-in-time reader for consistent pagination across index changes.

### 39. `cluster.routing.allocation.enable`

Controls whether any shard allocation/rebalancing occurs (`all`, `primaries`, `new_primaries`, `none`).

### 40. `GET /_cat/recovery?v`

Shows shard recovery/relocation progress and bytes transferred.

### 41. `must` vs `filter` (bool query)

`must` affects score and is required; `filter` is required but non-scoring and cacheable.

### 42. `index.lifecycle.name`

Names the ILM policy managing the index lifecycle.

### 43. `GET /_ilm/policy`

Returns all defined Index Lifecycle Management policies.

### 44. `_enrich` processor

Looks up and adds fields from a pre-built enrich index at ingest time.

### 45. `cluster.max_shards_per_node`

Hard limit on total shards per node to prevent oversharding (default 1000).

### 46. `POST /index/_split/new_index`

Increases shard count by splitting each shard into multiple shards.

### 47. `terms` vs `term` query

`term` matches one exact value; `terms` matches any of a list of exact values.

### 48. `index.hidden`

Marks index hidden from wildcard queries (`.*` patterns exclude unless explicitly requested).

### 49. `GET /_data_stream`

Lists data streams with generation, backing indices, and store size.

### 50. `_transform` API

Creates and manages continuous transform jobs producing aggregated summary indices.

---
