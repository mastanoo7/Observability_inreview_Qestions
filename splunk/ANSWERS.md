# Splunk — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## Indexers & Indexing Architecture

### 1. Indexing latency spikes every 4 hours correlating with bucket rolling

- **Correlate metrics:** Compare `_internal` metrics (`index_thruput`, `splunkd` CPU/disk) with bucket state transitions using `| rest /services/data/indexes` or the Monitoring Console indexer performance dashboard during roll windows.
- **Disk I/O hypothesis:** Check `iostat`/OS metrics and Splunk `service=server_introspection` for elevated `write`/`fsync` during hot→warm rolls; high `await` on indexer disks strongly implicates I/O.
- **Replication factor hypothesis:** Inspect cluster manager UI or `| rest /services/cluster/master/buckets` for buckets stuck in `Searchable=0` or `Replication=0` states after rolls; replication catch-up spikes CPU and network.
- **Search peer load hypothesis:** If rolls coincide with heavy scheduled searches, check `index=_internal source=*scheduler.log*` for concurrent searches hitting the same indexers; search-induced I/O competes with rolling.
- **Isolate with maintenance window:** Temporarily pause non-critical scheduled searches on affected indexes and observe whether roll-time latency drops—if yes, search load is a contributor.
- **Remediation:** Tune `maxHotBuckets`, `maxWarmBuckets`, and `homePath`/`coldPath` on separate volumes; increase indexer count; ensure search factor is met before aggressive frozen transitions.

```spl
index=_internal source=*metrics.log* group=per_index_thruput series=* 
| timechart span=5m avg(kbps) by series
```

### 2. High number of events in "thawed" state

- **Meaning:** Thawed buckets were restored from frozen (archive) back to searchable state on disk—they are not in the normal hot/warm/cold lifecycle and indicate archived data was brought online for investigation or compliance search.
- **Lifecycle signal:** Frequent thawed buckets suggest over-aggressive freezing, compliance restores, or failed re-freeze operations leaving buckets in limbo.
- **Search performance:** Thawed buckets lack full tsidx optimization of hot/warm tiers; searches spanning thawed data are slower and increase disk I/O; bloom filters may be stale or incomplete.
- **Remediation:** Re-freeze after investigation; use `splunk rebuild` only when necessary; prefer searching frozen data via `restore` jobs with time-bounded scope; audit who initiated thaw operations in `_audit`.

### 3. Internal architecture of an indexer bucket (rawdata, tsidx, metadata)

- **rawdata/journal:** Compressed journal of incoming events; indexing writes here first; searches read `_raw` from this file when retrieving event bodies.
- **tsidx:** Time-series index mapping timestamps and keywords to offsets in rawdata; primary file for accelerating time-range and keyword searches.
- **Metadata files:** `Sources.data`, `Hosts.data`, `SourceTypes.data`, `Strings.data`, `Bloomfilter`—track field lexicons and bloom filters for fast elimination of non-matching buckets.
- **Indexing flow:** Parser writes to memory → journal append → tsidx build → metadata/bloom filter update; bucket roll closes files and starts new hot bucket.
- **Troubleshooting value:** Slow keyword searches often mean tsidx/bloom miss (wrong indexes, rare terms); corruption in tsidx causes incomplete results; `btool`/`splunk fsck` validate bucket integrity.

### 4. Indexing rate dropped from 500GB/day to 50GB/day without log volume change

- **Forwarder pipeline:** Check UF `metrics.log` for `tcpin_connections`, queue full, or `blocked` flags; verify `outputs.conf` still points to correct indexers after network/DNS changes.
- **Indexer health:** `index=_internal` for `group=thruput`, parsing queue stalls, `too_many_buckets` errors, or license warnings throttling ingestion.
- **Parsing failures:** New sourcetype or `props.conf` change causing `MAX_EVENTS` truncation or line-breaking dropping events silently.
- **Cluster state:** Peer not receiving data (`status=Up` but `replication_factor` not met), bucket fixup backlog, or cluster manager preventing writes.
- **License/quotas:** Verify daily license not exceeded; check index-level `maxTotalDataSizeMB` or `frozenTimePeriodInSecs` forcing drops.
- **Compare `_indextime` vs `_time`:** Large gaps suggest timestamp parsing pushing events outside searchable windows, appearing as "missing" volume.

```spl
index=_internal source=*metrics.log* group=per_index_thruput 
| stats sum(kb) as kb_ingested by series 
| sort - kb_ingested
```

### 5. Index-time vs search-time field extraction trade-offs

- **Index-time:** Configured in `transforms.conf` + `props.conf` (`INDEXED_EXTRACTIONS`, `REPORT-*`); fields stored in tsidx/metadata—faster search, immutable after indexing, increases index size and indexing CPU.
- **Search-time:** `EXTRACT-*`, `FIELDALIAS`, `EVAL` in SPL; flexible, no re-index needed, but every search pays parsing cost on `_raw`.
- **Production guidance:** Reserve index-time for high-cardinality filters used in nearly every query (e.g., `src_ip`, `user`); keep verbose regex extractions at search-time or use `INDEXED_EXTRACTIONS` with structured logs (JSON).
- **Risk:** Index-time regex on high-volume unstructured logs can bottleneck parsing queues and cause ingestion lag.

### 6. Indexer disk filling faster than expected with normal license usage

- **Per-index breakdown:** Use `| rest /services/data/indexes` or Monitoring Console storage by index; identify unexpected growth in `summary`, `metric`, or internal indexes.
- **Bucket counts:** Check for stuck warm buckets, failed frozen/archive, or replication factor creating duplicate copies on peers.
- **Sourcetype bloat:** `index=* | stats sum(eval(len(_raw))) as bytes by index, sourcetype` to find noisy sourcetypes (debug logs, packet captures).
- **Remediation:** Lower retention (`frozenTimePeriodInSecs`), enable compression, route verbose logs to cheaper indexes, implement SmartStore for cold/frozen, delete unused summary/indexed extractions.

### 7. Bloom filters in tsidx files

- **Function:** Bloom filters provide probabilistic "definitely not present" checks for keywords/terms per bucket, allowing Splunk to skip buckets that cannot contain search terms.
- **Benefit:** Dramatic speedup for rare terms across many buckets—only candidate buckets are opened and scanned.
- **Failure scenarios:** Very common terms (high false-positive rate) eliminate little; wildcard-leading patterns (`*error`) cannot use blooms efficiently; searches without indexed fields or only time filters scan all buckets; post-filter only fields not in bloom lexicon.

### 8. Hot-warm-cold-frozen lifecycle for 7-year compliance retention

- **Hot/warm on fast SSD:** Recent 30–90 days on local SSD for operational and security queries; tune `maxDataSize` and `maxHotBuckets` for steady roll cadence.
- **Cold on SmartStore/S3:** Move cold buckets to object storage with local cache; keeps peer disk footprint manageable while retaining searchability.
- **Frozen archive:** After `frozenTimePeriodInSecs`, buckets freeze to S3/Glacier with manifest; restore via thaw for legal hold investigations only.
- **Search performance:** Default searches use `index=*` time windows—train users and dashboards to recent indexes; use summary indexing/data models for long-range aggregates.
- **Compliance:** WORM storage, encryption at rest, `_audit` for all searches touching compliance indexes; document retention in `indexes.conf`.

### 9. Index corruption in indexer cluster

- **Identify:** `splunk fsck repair --check-only` or cluster manager "fixup" tasks flagging corrupt buckets; searches returning incomplete results for specific time ranges.
- **Confirm scope:** Note bucket ID (`_bkt`), index, peer; check replication status—healthy copies on other peers mean no data loss.
- **Safe removal:** Use `splunk remove bucket` or cluster manager repair workflow—never manually delete bucket directories on one peer without cluster coordination.
- **Restore:** Cluster replicates from peers meeting replication factor; if all copies corrupt, restore from frozen archive or backup; `splunk rebuild` regenerates tsidx from rawdata when rawdata intact.
- **Prevention:** Graceful rolling restarts, avoid disk-full conditions, monitor fixup backlog alerts.

### 10. SmartStore with S3 for cold/frozen buckets

- **Configuration:** Enable SmartStore on indexers (`indexes.conf` `remotePath` with S3 volume in `server.conf`/`indexes.conf`); cluster manager distributes cache policies.
- **Behavior:** Warm rolls to cold upload to S3; local cache holds frequently accessed buckets; searches fetch from S3 on cache miss.
- **Performance:** Cold searches have higher latency (network); size cache (`volume:max_cache_size`) for working set; avoid wide-time-range ad-hoc searches on cold without index narrowing.
- **Failure modes:** S3 credential expiry, throttling, partial upload leaves bucket fixup stuck; cache exhaustion causes search timeouts; monitor `smartstore` metrics in `_internal`.

---

## Forwarders & Data Collection

### 11. Universal Forwarder dropping events — queue full at peak load

- **Diagnose:** On UF, check `$SPLUNK_HOME/var/log/splunk/metrics.log` for `tcpin_queue`, `aggQueue`, `maxKBps` throttling, or `connection timeout`; correlate with indexer `tcpin_connections` metrics.
- **Root causes:** Indexer saturation, network bandwidth, `outputs.conf` `compressed=false` on high-latency links, excessive parsing on HF upstream, or undersized `maxQueueSize`.
- **Immediate fixes:** Increase `maxQueueSize` in `outputs.conf`, enable compression, add indexers to `targetGroup`, temporarily reduce monitored inputs or sampling.
- **Long-term:** Scale indexers, implement intermediate forwarders for aggregation, use HEC with load-balanced VIP, right-size `sendCookedData` routing, deploy Monitoring Console forwarder alerts.

### 12. UF vs Heavy Forwarder vs Intermediate Forwarder

- **Universal Forwarder:** Lightweight agent; minimal parsing; ships raw or lightly cooked data; use on every host for log collection.
- **Heavy Forwarder:** Full Splunk processing; index/filter/route, sensitive parsing, anonymization; use when you must transform before indexers or act as syslog aggregator.
- **Intermediate Forwarder:** UF or HF tier that receives from many UFs and forwards upstream—buffers, load-balances, reduces indexer connection fan-out.
- **Example architecture:** UF on 5,000 hosts → intermediate HF tier (routing by `index`) → indexer cluster; dedicated HF for syslog/NFV appliances that can't run UF.

### 13. Duplicate events from Universal Forwarder

- **Forwarder config:** Duplicate `inputs.conf` stanzas, multiple `[tcpout]` to same indexer without load balancing awareness, or UF + another agent shipping same file.
- **Load balancer:** TCP LB without sticky sessions causing same connection data replayed; health-check traffic indexed as data.
- **Indexer side:** Cluster bucket repair duplicating during recovery (rare); HEC clients retrying without acks.
- **Detection:** `index=* sourcetype=X | dedup _raw, _time, host | stats count by host` vs without dedup; use `duplicate` field if configured.

### 14. Heavy Forwarder at 100% CPU — regex routing and extraction

- **Profile:** Enable `splunkd` debug on parsing; use `btool check` for conflicting stanzas; inspect `metrics.log` `group=parsing` queues.
- **Optimize:** Move regex to indexers only if necessary; simplify `transforms.conf`; use structured logs; route early with `accept/reject` in `props.conf` instead of heavy `EVAL` on HF.
- **Scale out:** Add HF instances behind load balancer; shift parsing to indexers or DSP pipeline; reduce `MAX_EVENTS` misconfiguration causing re-parse loops.

### 15. Forwarder management at scale (5,000 UFs, hybrid cloud)

- **Deployment Server / Deployer:** Centralize `serverclass.conf` with staged rollout (canary server classes); use DS for UF, Deployer for SHC/CM config apps.
- **Version upgrades:** Phased server class rollout; monitor `$SPLUNK_HOME/var/log/splunk/splunkd.log` for UF handshake failures; maintain N-1 compatibility matrix.
- **Health monitoring:** Forwarder overview in Monitoring Console; alerts on `lastConnected`, version drift, config checksum mismatch; integrate with CMDB for ownership.

### 16. SSL/TLS and certificate-based auth UF → indexer

- **Configure:** `outputs.conf` `clientCert`, `sslPassword`, `sslVerifyServerCert`; indexer `inputs.conf` `SSL` with `requireClientCert=true` and `sslKeysPassword`.
- **Rotation:** Dual-trust CA period—deploy new certs to indexers first (accept both CAs), roll UFs, remove old CA; use cert management (Vault) with automation.
- **No data loss:** UFs buffer to disk queue during brief handshake failures; extend cert expiry alerts at 30/60 days.

### 17. Forwarder "connection refused" after network change

- **Verify connectivity:** `telnet indexer 9997`, DNS resolution of `server` in `outputs.conf`, firewall rules for `splunkd` management (8089) vs forwarding (9997).
- **Config files:** `$SPLUNK_HOME/etc/system/local/outputs.conf`, `server.conf` `[sslConfig]`; check `deploymentclient.conf` if using DS.
- **Logs:** `$SPLUNK_HOME/var/log/splunk/splunkd.log` (connection errors), `metrics.log` (tcpout); on indexer `splunkd.log` and `metrics.log` `tcpin_connections`.
- **Common fixes:** Wrong IP after migration, SELinux, indexer `inputs.conf` disabled, cluster rolling restart not complete.

### 18. Indexer Discovery vs manual configuration

- **Indexer Discovery:** Cluster manager publishes peer list to forwarders via `master_uri`; forwarders auto-update `outputs.conf` when peers join/leave.
- **Manual:** Static `server` list in `outputs.conf` `targetGroup`—simple but requires config push on topology changes.
- **Failure modes:** CM unreachable → forwarders use stale peer list; new peers not discovered until CM healthy; misconfigured `pass4SymmKey` blocks discovery channel.

### 19. Kubernetes: DaemonSet UF vs Splunk Connect for Kubernetes (SC4K)

- **DaemonSet UF:** UF per node reading container logs from `/var/log/containers`; familiar ops model; you manage parsing/annotations yourself.
- **SC4K:** Operator-driven; HEC ingestion, metadata enrichment (pod/namespace/labels), metrics and events support; better K8s-native integration.
- **Trade-offs:** UF simpler for log-only; SC4K better for multi-signal observability and autoscaling patterns; SC4K adds operator complexity and HEC capacity planning.

### 20. Forwarder load balancing and indexer failure handling

- **Config:** `outputs.conf` `autoLBFrequency`, multiple servers in `targetGroup`, `autoLB = true` for cooked TCP forwarding.
- **Behavior:** UF rotates connections across available indexers; on failure, marks indexer down after timeout and redistributes; queued events flush when indexer returns.
- **Best practice:** Ensure enough indexers for aggregate throughput; monitor `tcpin_connections` distribution for imbalance.

---

## Search Heads & Search Management

### 21. SHC search failures — "search peer not responding"

- **Network:** Verify SH→indexer connectivity on 9887 (management/search channel); check firewall changes, TLS mismatches, certificate expiry.
- **Indexer performance:** Peer CPU/disk saturation, long-running searches blocking search daemon; check `index=_internal` `search_process` metrics on peers.
- **Captain election:** Unstable SHC captain causes job dispatch failures; review `splunkd.log` on SHC members for captaincy churn; verify `conf replication` healthy.
- **Diagnosis SPL:** Run distributed search with `| rest /services/search/distributed/peers` from SH to see peer status.

### 22. SHC captain election and in-flight searches

- **Election:** Raft-based quorum among SHC members; captain coordinates artifact replication, job scheduling, and KV store leader operations.
- **Re-election impact:** Brief window where new jobs may fail; in-flight searches on peers typically continue on indexers; captain-only tasks (scheduled search ownership) migrate.
- **Minimize impact:** Maintain odd member count (3/5/7); ensure stable network; `shc_deployment` health; avoid simultaneous restarts; use `graceful` rolling restarts.

### 23. Search head OOM from concurrent searches

- **Limits:** `limits.conf` `max_searches_per_cpu`, `max_hist_searches`, per-role quotas in `authorize.conf`.
- **Scheduler:** `scheduler.conf` priorities for real-time vs historical; separate scheduled search windows for heavy reports.
- **Resource pools:** `workload_rules.conf` / workload management assigning CPU/memory priority to security vs compliance roles.
- **Tuning:** Reduce `subsearch` depth, `realtime` window size, and concurrent dashboard panel searches.

### 24. Search artifact replication in SHC

- **Mechanism:** Captain replicates search artifacts (results, KV store entries) to SHC members for job continuity and ad-hoc result access.
- **Storage:** Each SH needs disk for `$SPLUNK_HOME/var/run/splunk/dispatch`; high concurrency × large result sets fills disk quickly.
- **Tuning for 50 users:** Shorten `ttl` for artifacts, `limits.conf` `max_mem_usage_mb`, enable `shclustering` artifact sync tuning, store expensive reports as summary indexes instead of large ad-hoc jobs.

### 25. Search head pooling for user groups

- **Separate SH/SHC pools:** Operations SHC for real-time dashboards; compliance SHC with strict concurrency and longer retention searches.
- **Workload management:** Rules routing searches by app/role/index to resource pools with guaranteed CPU for ops.
- **Federated search (optional):** Compliance SHC searches remote indexes without loading ops SHC scheduler.

### 26. Scheduled search every 5 minutes taking 8 minutes

- **Profile:** Search Job Inspector—identify slow command (`join`, `transaction`, `map`), scan count, index selection.
- **Optimize:** Narrow `earliest/latest`, specify `index=`, replace `stats` pipeline with `tstats` from data model, accelerate DM, use summary index.
- **Scheduler:** Enable `allow_skipped` awareness; increase `cron` to 10m if business allows; run heavy search on dedicated SH pool; `auto_summarize` for CIM fields.

### 27. Workload management for incident response

- **Configure:** `workload_pools.conf` and `workload_rules.conf`—pools for `realtime`, `standard`, `low_priority`.
- **Incident mode:** Rule bumping security role searches to high-priority pool with reserved CPU/memory; throttle compliance/historical pools.
- **Monitor:** workload pool usage in Monitoring Console; avoid starvation of critical indexing by capping search pool on indexers (9.x+ unified WM).

### 28. Search head failover (non-clustered)

- **Manual failover:** DNS/load balancer points users to standby SH; restore `$SPLUNK_HOME/etc/users`, KV store, and savedsearches from backup.
- **Data lost:** In-flight ad-hoc jobs, ephemeral dispatch artifacts, uns replicated KV store entries, recent user-generated content not backed up.
- **Minimize:** Regular backup of etc/users and etc/apps; export saved searches to Git; use SHC for production instead of standalone SH.

---

## Clustering & High Availability

### 29. Lost replication factor for 30% of buckets after indexer failure

- **Assess risk:** Buckets with RF=0 on failed peer may have SF=0 for some copies—data loss if no other copy exists; cluster manager shows fixup status.
- **Restore:** Replace/restart peer; cluster auto-runs fixup to replicate buckets; monitor `fixup_tasks` backlog; add temporary peer capacity if fixup slow.
- **Prevention:** Maintain N+2 indexer capacity, alert on RF/SF degradation before peer loss, regular cluster health checks, avoid `site replication factor` misconfiguration in multi-site.

### 30. Replication factor vs search factor

- **Replication factor (RF):** Number of full copies of each bucket across peers—protects against data loss.
- **Search factor (SF):** Number of searchable copies (with tsidx)—protects search availability without full duplication of all components on every peer.
- **Production issue example:** RF=3 but SF=1—one peer failure leaves searches needing that peer's tsidx unavailable even though rawdata exists elsewhere, causing incomplete search results and "peer not responding" for specific time ranges.

### 31. Cluster Manager (cluster master) unresponsive

- **Indexing impact:** Peers continue indexing with last-known cluster state; bucket fixup and rolling restart coordination paused; new peers can't join.
- **Search impact:** SHC/indexer cluster searches largely continue; config updates and fixup stall.
- **Recovery:** Restore CM from backup or promote standby; never run two active CMs; `splunk show cluster-bundle-status` after recovery; peers re-register automatically when CM returns.

### 32. Multi-site cluster for DR across two data centers

- **Configure:** `server.conf` `[clustering]` multi-site enabled; `site_replication_factor` and `site_search_factor` (e.g., origin:2, total:3).
- **Topology:** Site1 primary indexing + Site2 DR copies; search heads at both sites with site-aware search.
- **Failover:** Site failure triggers searchable copies from surviving site; rehearse site-down scenario quarterly.

### 33. Adding new indexer peer without disruption

- **Procedure:** Provision peer, enable `clustering` peer mode, point to CM, provide `replication_port`; CM assigns bucket slices gradually.
- **Rebalance:** Use cluster manager rebalance after peer joins to redistribute primaries; expect background fixup traffic—schedule off-peak.
- **Verify:** Peer reaches `Up`, RF/SF met, no fixup errors; forwarders via Indexer Discovery pick up new peer automatically.

### 34. Rolling upgrade of indexer cluster

- **Steps:** Upgrade CM first (or last per Splunk version matrix), then peers one-at-a-time via `splunk rolling-restart cluster-peers`, verify RF/SF between each.
- **Zero data loss:** Never restart more peers than RF allows simultaneously; wait for `bucket status=Up` and fixup complete.
- **Search disruption:** Brief fixup during restart; schedule during low window; communicate SHC users about possible slow searches.

### 35. Excess bucket removal and data loss

- **Mechanism:** Cluster removes excess bucket copies beyond RF/SF to save disk—misconfigured `site_replication_factor` or manual bucket deletion can remove last copy.
- **Incident scenario:** Admin lowered RF during disk crisis without verifying SF; fixup removed copies peers needed; frozen archive was disabled—permanent loss for affected time range.
- **Prevention:** Never reduce RF below 2 for production; use `maintenance_mode` during topology changes; audit cluster changes in `_audit`.

---

## SPL Queries & Search Optimization

### 36. `stats count by host, sourcetype` across 30 days timing out

- **Time filter:** Always bound `earliest=-30d@d latest=now` with `@d` alignment; avoid all-time.
- **Index selection:** `index=web` not `index=*`; verify data model acceleration covers fields.
- **Summary indexing:** Pre-run nightly `stats count by host, sourcetype` into `summary` index; dashboard queries summary.
- **tstats:** If CIM-accelerated DM exists: ``| tstats count where index=web by host, sourcetype``.

### 37. `tstats` vs `stats`

- **tstats:** Queries tsidx/metadata only—no `_raw`; extremely fast for indexed/accelerated fields and data models.
- **stats:** Full search pipeline on events—required for `_raw`, complex `eval`, non-indexed fields.
- **10x+ scenarios:** Large index over long time on common fields in DM; security summaries by `src_ip`, `user`.
- **Limitations:** No arbitrary `_raw` regex; requires accelerated fields; `prestats=true` patterns; complex functions limited.

### 38. Brute force login detection across sourcetypes/indexes

```spl
(index=auth OR index=web) (failed OR failure OR "401" OR "invalid password" OR "authentication failed")
| eval user=coalesce(user, uid, username)
| stats count as failures earliest(_time) as first latest(_time) as last by user, src_ip
| where failures >= 10
| eval duration=last-first
| where duration < 300
| sort - failures
```

- Normalize user/src fields via `FIELDALIAS` or CIM; tune thresholds per application; enrich with `lookup` for known-good IPs; alert with adaptive suppression.

### 39. `join` vs `lookup` vs `append` vs subsearches

- **join:** SQL-like merge on field—expensive, memory-heavy, avoid on large result sets.
- **lookup:** O(lookup table size)—best for enriching with reference data (users, assets).
- **append/appendcols:** Stack results—good for combining pre-aggregated datasets of similar size.
- **Subsearch:** Useful when inner search returns small result set injected into outer `search` filter—dangerous if inner returns millions of rows.

### 40. 95th percentile API response times across 100 microservices (30-day trend)

```spl
index=apm sourcetype=access_* span=1h
| stats p95(response_time) as p95_latency by service, _time
| timechart span=1h p95(p95_latency) by service limit=20
```

- Prefer metrics index with `mstats` if using metric store; use `tstats` from accelerated DM; pre-aggregate daily for executive dashboards.

### 41. Rewriting memory-heavy `transaction`

- **Problem:** `transaction` holds entire sessions in memory.
- **Rewrite:** Use `stats` with `earliest(_time)`, `latest(_time)`, `values()` grouped by `session_id`; or `streamstats` for ordered per-session calculations.
- **Example:** Replace `| transaction session_id` with `| stats min(_time) as start max(_time) as end count by session_id | eval duration=end-start`.

### 42. SPL macros and workflow actions with version control

- **Macros:** Define in Splunk UI or `macros.conf` in apps; export apps to Git; use `macro_name(1)` parameters for reusable filters.
- **Workflow actions:** Link from fields to runbooks/tickets; store in app `default/data/ui/workflow-actions.conf` in Git.
- **Governance:** PR review for app repo; separate dev/prod deployer; naming convention `team_usecase_filter_v1`.

### 43. `eval` null handling and null propagation

- **Behavior:** Most `eval` functions propagate null—`null + 1` is null; comparisons with null are false unless using `isnull()`/`coalesce()`.
- **Production example:** `| eval error_rate=errors/total` yields null when `total=0`, breaking `where error_rate>0.05`—fix with `| eval error_rate=if(total>0, errors/total, 0)`.
- **Defense:** Always `coalesce(field, "unknown")` before stats grouping.

### 44. Anomaly detection without ML Toolkit

```spl
index=metrics metric=cpu_utilization earliest=-7d
| timechart span=5m avg(value) as avg_cpu by host
| streamstats window=288 avg(avg_cpu) as baseline stdev(avg_cpu) as stdev_cpu by host
| eval upper=baseline+3*stdev_cpu, lower=baseline-3*stdev_cpu
| where avg_cpu > upper OR avg_cpu < lower
```

- Tune window to seasonality; use `eventstats` for global baselines; combine with `outlier` command if available.

### 45. `map` command vs subsearches vs lookups

- **map:** Runs subsearch once per row of input—powerful but can spawn thousands of searches—use only on small input sets.
- **Subsearch:** Single injection—better when one list applies to all events.
- **lookup:** Best for static/slowly-changing correlation tables; no search fan-out.

---

## Ingestion Bottlenecks & Parsing Issues

### 46. 2-hour indexing lag during peak hours

- **Forwarder:** Queue depth, `maxKBps`, network latency to indexers.
- **Indexer:** Parsing queue (`metrics.log` `group=parsing`), indexing queue, too many hot buckets, CPU saturation.
- **End-to-end:** Compare event `_time` vs `_indextime` lag metric in Monitoring Console.
- **Fix:** Scale indexers, reduce index-time extractions, fix noisy sourcetypes, increase parsing threads (`parallelIngestionPipelines`), use DSP pre-filtering.

### 47. Incorrect timestamps — wrong time buckets

- **Diagnose:** `index=* | eval lag=_indextime-_time | stats avg(lag) max(lag) by sourcetype`; sample `_raw` for timestamp format.
- **Fix:** `props.conf` `TIME_PREFIX`, `TIME_FORMAT`, `MAX_TIMESTAMP_LOOKAHEAD`, `TZ` for sourcetype; deploy to forwarder tier if using HF for parsing.
- **Existing data:** Cannot re-index without re-ingesting; create `alias` index or use `collect` after re-processing offline; adjust searches to `_indextime` temporarily.

### 48. Multi-line Java stack traces split incorrectly

- **Configure:** `props.conf` `SHOULD_LINEMERGE=true`, `LINE_BREAKER`, `MUST_BREAK_AFTER`, or `TRUNCATE=0` with careful `MAX_EVENTS`.
- **Structured alternative:** Ship JSON stack traces as single events via app logging framework.
- **Validate:** `index=* sourcetype=java | head 5` shows complete stack in `_raw`.

### 49. Transforms not applied — unparsed `_raw`

- **Pipeline:** Input (`inputs.conf`) → parsing (`props.conf`/`transforms.conf` on forwarder or indexer) → indexing (index-time extractions) → search (search-time).
- **Debug:** `splunk btool props list <sourcetype> --debug`; verify `TRANSFORMS-*` stanzas, sourcetype assignment, app context (local vs cluster app).
- **Common failures:** Sourcetype mismatch, app not deployed to cluster bundle, `priority` conflicts, missing `REPORT-*` reference in `props.conf`.

### 50. Structured logging for performance and reliability

- **Approach:** JSON logs with consistent keys; `INDEXED_EXTRACTIONS = json` or `KV_MODE = json` in `props.conf`.
- **Benefits:** Fast field search, smaller `_raw` regex cost, reliable `tstats` from data model.
- **configs:** Minimal `transforms.conf`; use `FIELDALIAS` for CIM normalization; avoid heavy index-time regex.

### 51. Inconsistent timestamp formats across app versions

- **Strategy:** Multiple `TIME_FORMAT` not supported in one stanza—use `rename` props by source, or `EVAL` at HF to normalize, or sourcetype per version (`app_v1`, `app_v2`).
- **Robust:** Require ISO-8601 from applications; use `datetime.xml` custom rules sparingly; test with `splunk cmd anonymize` samples.

### 52. Wrong sourcetype assignment

- **Diagnose:** `index=* | stats count by sourcetype, host, source`; inspect mis-parsed events.
- **Fix:** `inputs.conf` explicit `sourcetype`, `transforms.conf` `MetaData:Host` rules, `props.conf` `[source::/path/...]` overrides.
- **Prevention:** Document onboarding checklist; automated tests with `splunk btprobe` sample files.

---

## Scaling & Performance

### 53. Scale from 500GB/day to 5TB/day

- **Indexers:** ~10× peers (rule-of-thumb 100–200GB/day per indexer depending on events/sec); separate hot/warm storage; SmartStore mandatory.
- **Search heads:** Additional SHC members; workload management; summary indexes and accelerated data models.
- **Network:** 10G+ between sites; dedicated forwarding tier; HEC behind load balancers with auto-scaling indexers.
- **Ops:** Monitoring Console capacity planning; license true-up; phased migration with dual-write validation.

### 54. Distributed search optimization

- **Search factor:** Higher SF spreads search I/O across peers—faster parallel bucket scans.
- **Data distribution:** Even bucket primaries per peer; rebalance after adding peers; avoid "hot" peer with disproportionate primaries.
- **Practices:** Use `tstats`, reduce fan-out, `mapred_reduce` optimizations, localize searches to correct sites in multi-site.

### 55. Summary indexing for executive dashboards

- **Pattern:** Scheduled search writes aggregates to `summary` index via `collect` or `outputsummary`.
- **Example:** Nightly `| stats avg(duration), p95(duration) by service | collect index=summary`.
- **Dashboards:** Query summary index with 1-year window in seconds; document summary retention separately from raw.

### 56. High CPU on indexers during search

- **Balance:** `max_searches_per_cpu`, separate search-disabled peers for very large clusters (legacy), workload pools limiting search CPU.
- **Config:** `indexerDiscovery` and `parallelReduce` where supported; disable unnecessary real-time searches; use search head pooling.
- **Latency:** Schedule heavy reports off-peak; use data model acceleration to reduce raw event scans on indexers.

### 57. Data model acceleration

- **Enable:** Accelerate CIM or custom DM; Splunk builds tsidx summaries on schedule.
- **Trade-offs:** Extra disk on peers, background CPU for acceleration rebuilds, staleness until rebuild completes.
- **When worth it:** Pivot, `tstats`, ES correlation searches hitting same fields repeatedly.

### 58. Capacity planning (3-year growth)

- **Storage tiering:** Hot SSD → warm → SmartStore S3 → frozen archive; model compression ratios (~50%).
- **Compute:** Indexer and SHC headroom at 70% sustained; include fixup/rebalance overhead.
- **License:** Enterprise term growth; dev/test environments; sub-license monitoring for violations.

---

## SIEM Use Cases & Security

### 59. Detecting lateral movement

```spl
| tstats summariesonly=true count from datamodel=Network_Traffic.All_Traffic
  where All_Traffic.dest_port IN (445,135,139,3389,5985) by All_Traffic.src, All_Traffic.dest
| `drop_dm_object_name("All_Traffic")`
| lookup internal_hosts asset_ip OUTPUT is_internal
| where is_internal="true"
| stats dc(dest) as unique_targets by src
| where unique_targets > 20
```

- Correlate with auth logs (`LogonType=3`, Pass-the-Hash), new service installs, scheduled tasks; integrate ES Threat Framework analytic stories.

### 60. Enterprise Security (ES) for financial services

- **Notable events:** Tune correlation searches to risk score thresholds; use `suppress` fields to reduce duplicates.
- **Risk scores:** Asset/identity-centric scoring; integrate asset and identity frameworks.
- **Response:** Adaptive Response actions integrated with ticketing and SOAR; audit all automated actions for SOX.
- **Compliance:** Retention on security indexes, role-based access, dual-control for sensitive searches.

### 61. Real-time data exfiltration detection

- **Data sources:** Proxy, firewall, DLP, cloud audit (S3/GCS), DNS, NetFlow.
- **Queries:** Unusual upload volume `| stats sum(bytes_out) by src_ip | streamstats` baseline; rare external destinations; off-hours large transfers.
- **Thresholds:** Dynamic baselines per department; severity by data classification from lookup.

### 62. UBA integration with ES for insider threats

- **Requirements:** Authentication, VPN, HR context, endpoint, cloud app logs—normalized to identities.
- **Integration:** UBA threats push to ES as notable events with mapped risk objects; tune disposition workflows.
- **Training:** Allow 2–4 weeks baseline per user/peer group; review false positives before auto-escalation.

### 63. 1,000 notable events/hour — risk-based alerting

- **RBA:** Aggregate related findings into risk incidents; threshold by cumulative risk score vs per-event alerts.
- **Tuning:** Raise fidelity on noisy searches; throttle by `risk_object`; use ES Analytic Stories with investigation triage.
- **Metrics:** Track MTTR, true positive rate, analyst capacity; continuous content tuning sprints.

### 64. Adaptive Response for compromised host containment

- **Framework:** Define AR action in ES (e.g., isolate host via EDR API, disable AD account, firewall block).
- **Safety:** Human approval step for production servers; test in staging; rollback playbook.
- **Audit:** Log action, actor, correlation search ID in `_audit` and SOAR case.

---

## Production Outage Analysis

### 65. Search heads unavailable — same network segment as affected systems

- **Alternatives:** Access indexers directly via CLI `splunk search` on peer (break-glass), offline frozen bucket search, replicated SH in DR site.
- **Pre-positioned:** Standby SHC in separate AZ/VLAN; offline export of critical saved searches; forward critical logs to cloud SIEM mirror.
- **Prevention:** SHC in dedicated management network; out-of-band console access to indexers.

### 66. Post-incident analysis — 4-hour outage, 10 microservices

- **Timeline strategy:** Start with change events (`index=_internal` deploy logs, CI/CD), then error rate spikes per service, then dependency traces.
- **Queries:** `index=app* earliest=<outage_start-1h> latest=<outage_end+1h> (ERROR OR exception) | timechart span=1m count by service`; correlate `trace_id` across services.
- **Narrow:** Identify first failing service (furthest upstream in DAG); validate with load balancer 5xx, DB connection errors, saturation metrics.

### 67. Real-time search for live incident monitoring

- **Implementation:** `search index=* earliest=-5m` real-time window or RT search heads; dashboards with 30s refresh.
- **Cost:** Each RT search holds open streaming connections and memory on indexers—limit concurrent RT searches; use metrics/alerts for threshold detection instead of many RT dashboards.

### 68. Automated incident detection correlating errors, metrics, user impact

- **Approach:** ITSI or custom correlation search joining application error rate, `mstats` infra metrics, and synthetic RUM checks.
- **Single alert:** Fire when ≥2 of 3 signal classes breach dynamic threshold within 5m window; enrich with service owner lookup.
- **Reduce noise:** Episode policies in ITSI; maintenance windows for known deploys.

### 69. Alert failed due to 30-minute ingestion lag

- **Monitor lag:** `index=_internal | eval lag=_indextime-_time | stats max(lag) by index` alert when lag > 10m.
- **Alert logic:** Use `_indextime` for detection windows or `earliest=-70m@min` with `latest=-30m@min` delayed evaluation; document expected lag in runbook.
- **Escalation:** Separate "data freshness" alert from application alerts.

### 70. ITSI for service-level monitoring

- **KPI base searches:** Define per-service latency, error rate, saturation SPL with normalized units.
- **Service tree:** Model dependencies (web → API → DB); health scores roll up weighted by criticality.
- **Episodes:** Group related KPI breaches; episode review workflow for incident commanders.

---

## Splunk Cloud & Kubernetes Integration

### 71. Splunk Enterprise to Splunk Cloud migration

- **Differences:** No indexer OS access; HEC/DM/data inputs only; apps vetted for cloud compatibility; CM/DS replaced by Splunk Cloud admin.
- **Data sovereignty:** Use Splunk Cloud stacks in required regions (e.g., EU stack for GDPR); federated search to on-prem where data must remain local.
- **Migration:** Dual-feed via HEC, validate parsing, cutover indexes, retire on-prem peers after retention satisfied.

### 72. Splunk monitoring for Kubernetes

- **Collect:** SC4K for logs/metrics/events; HEC for app logs; OpenTelemetry collector sidecar optional.
- **Sources:** Container logs, `kube-apiserver`, audit logs, node metrics, ingress controller access logs.
- **Enrichment:** Pod labels, namespace, cluster name as indexed fields for filtering.

### 73. HEC at scale for 500 microservices

- **Architecture:** Multiple HEC tokens per team/service class; load-balanced HEC endpoints (Splunk Cloud or indexer tier).
- **Client:** Retry with exponential backoff; batch events; use ack-enabled HEC for guaranteed delivery.
- **Governance:** Token rotation, rate limits per token, index routing via `index` field in payload.

### 74. Multi-cloud consistent logging

- **Standardize:** Common sourcetype/CIM models; Infrastructure app for AWS/GCP/Azure; unified `index=cloud` naming.
- **Collection:** SC4K/UF/OTel per cloud with same `deployment_server` or GitOps app repo.
- **Search:** Federated search across stacks or single Splunk Cloud with data routed by `cloud_provider` field.

### 75. Observability Cloud + Splunk Enterprise unified view

- **Integration:** Correlate APM traces/metrics (Observability Cloud) with Splunk logs via `trace_id`, `service.name`.
- **Pattern:** Dual-send from apps; use Splunk App for Observability Cloud; linked dashboards from log spike to trace waterfall.

### 76. HEC overwhelmed during traffic spike

- **Load balance:** Multiple indexers with HEC behind LB; auto-scale indexers (Victoria/autoscaling groups).
- **Backpressure:** Clients honor HTTP 503; queue locally; increase `max_content_length` carefully; tune `maxThreads` on HEC input.
- **Tokens:** Separate tokens per service to identify noisy neighbors; rate limit abusive tokens.

### 77. Data Stream Processor (DSP) for real-time routing

- **Use cases:** Filter DEBUG logs before indexers, enrich with lookups, route PII-free copy to analytics pipeline.
- **Deploy:** DSP pipelines as SPL2; connect sources (Kafka, HEC) to Splunk indexers or external sinks.
- **Benefits:** Reduces indexer load and license consumption; sub-second routing decisions.

### 78. HIPAA compliance logging

- **Controls:** PHI in dedicated indexes; field masking at ingest (`transforms.conf` SEDCMD or HF anonymization); encryption in transit (TLS) and at rest.
- **Access:** RBAC least privilege; MFA; no shared accounts; `_audit` and Splunk Cloud audit logs for all access.
- **BAA:** Splunk Cloud BAA with AWS; document subprocessors; breach detection searches on abnormal PHI access patterns.

---

## Basic Questions

### 1. What is Splunk and what is its primary use case in enterprise IT?

Splunk is a platform for searching, monitoring, and analyzing machine-generated data (logs, metrics, events) at scale. Its primary enterprise use case is centralized observability, security analytics (SIEM), and IT operations troubleshooting—turning unstructured logs into actionable insights via SPL.

### 2. What is the difference between a Splunk indexer, search head, and forwarder?

An **indexer** stores data and executes search jobs on local buckets. A **search head** provides the UI/API and coordinates distributed searches across indexers but typically stores no production data. A **forwarder** collects data from sources and sends it to indexers without serving interactive search (UF) or with optional intermediate processing (HF).

### 3. What is a Splunk index and how does it organize data?

An index is a logical container for events, defined in `indexes.conf` with retention, storage paths, and access controls. Physically, data is split into time-bucketed directories (hot/warm/cold/frozen) containing rawdata and tsidx files.

### 4. What is SPL (Search Processing Language) and what is it used for?

SPL is Splunk's query language for filtering, transforming, aggregating, and visualizing machine data. It is used to investigate incidents, build dashboards, define alerts, and power security correlation searches.

### 5. What is a Universal Forwarder and how does it differ from a Heavy Forwarder?

A **Universal Forwarder (UF)** is a lightweight agent that forwards data with minimal processing. A **Heavy Forwarder (HF)** runs the full Splunk processing engine and can parse, filter, route, and index data before forwarding.

### 6. How do you perform a basic keyword search in Splunk?

Enter terms in the search bar, e.g., `error OR failed`, optionally scoped with `index=main sourcetype=access_combined`. Splunk matches keywords against the full index and returns matching events in the Events tab.

### 7. What is a Splunk sourcetype and why is it important?

A sourcetype identifies the format/structure of incoming data (e.g., `access_combined`, `syslog`). It determines which `props.conf`/`transforms.conf` parsing rules apply, directly affecting field extraction, timestamps, and search accuracy.

### 8. What is the purpose of the `index` field in a Splunk search?

The `index` field scopes a search to a specific logical dataset, reducing scan volume and enforcing data boundaries. Using `index=` is a best practice for performance and RBAC alignment.

### 9. How do you use the `stats` command in SPL?

`stats` aggregates events, e.g., `| stats count, avg(response_time) by host`. It discards raw events and outputs a results table—ideal for summaries and dashboards.

### 10. What is a Splunk saved search and when would you use it?

A saved search stores an SPL query and parameters for reuse. Use it for scheduled reports, alerting, dashboard panels, and operational runbooks that teams run repeatedly.

### 11. What is the purpose of the `host` field in Splunk events?

`host` identifies the originating machine or device (hostname or IP). It is set at input time and is commonly used to filter and aggregate data by system.

### 12. How do you use the `table` command in SPL to format search results?

`| table _time, host, user, status` selects and displays only the specified fields in columnar format. It is used to simplify output for analysts and reports.

### 13. What is a Splunk dashboard and how do you create one?

A dashboard is a collection of visual panels (charts, tables, single values) driven by SPL searches. Create one in Dashboard Studio or Classic by adding searches and choosing visualization types, then publish to an app.

### 14. What is the purpose of the `eval` command in SPL?

`eval` creates or modifies fields using expressions, e.g., `| eval mb=bytes/1024/1024`. It is the primary tool for calculated fields, conditional logic, and normalizing values during search.

### 15. How do you use the `rex` command to extract fields from log data?

`| rex field=_raw "(?<status>\d{3})"` applies a regex to `_raw` and captures named groups as fields. Use it for ad-hoc or search-time extraction without changing `props.conf`.

### 16. What is a Splunk alert and how do you configure one?

An alert runs a saved search on a schedule or in real time and triggers an action (email, webhook, script) when conditions are met. Configure via **Settings → Searches, reports, and alerts** with trigger conditions and throttle/suppression.

### 17. What is the purpose of the `timechart` command in SPL?

`timechart` aggregates metrics over time buckets, e.g., `| timechart span=1h count by sourcetype`. It is used for trend graphs and temporal pattern analysis.

### 18. How do you use the `dedup` command to remove duplicate events?

`| dedup host, _raw` keeps the first event per unique combination of fields. Use it to collapse duplicate log lines while preserving one representative copy.

### 19. What is a Splunk lookup and when would you use it?

A lookup enriches events with external data (CSV, KV store) via `| lookup asset_table ip OUTPUT hostname, owner`. Use it for asset context, geolocation, threat intel, and normalization without hardcoding in SPL.

### 20. What is the purpose of the `where` command in SPL?

`| where status>=400` filters results after initial processing, similar to SQL WHERE. It keeps only rows matching the expression—useful after `stats` or `eval`.

### 21. How do you configure a Splunk forwarder to send data to an indexer?

In `outputs.conf`, define `[tcpout]` with `server=indexer1:9997,indexer2:9997` and enable `defaultGroup`. Restart the forwarder; verify connection in `metrics.log`.

### 22. What is the purpose of the `props.conf` file in Splunk?

`props.conf` controls parsing at search/index time: line breaking, timestamp extraction, sourcetype rules, and field extraction references. It is the primary file for making data searchable correctly.

### 23. How do you use the `top` command to find the most common values?

`| top limit=10 user` returns the most frequent values of `user` with counts and percentages. Use it for quick frequency analysis during triage.

### 24. What is a Splunk field extraction and how do you create one?

Field extraction pulls values from `_raw` into fields via Field Extractor UI, `props.conf` (`EXTRACT-*`), or `transforms.conf`. Search-time extractions are preferred for flexibility; index-time for high-performance filters.

### 25. What is the purpose of the `transforms.conf` file in Splunk?

`transforms.conf` defines reusable transformation stanzas referenced from `props.conf`—regex extractions, field aliases, host/sourcetype routing, and SEDCMD anonymization.

### 26. How do you use the `transaction` command in SPL?

`| transaction session_id maxspan=30m` groups related events into transactions. Use it for session analysis when events share a common ID but are not a single log line.

### 27. What is a Splunk data model and when would you use it?

A data model defines hierarchical fields and objects (often CIM-aligned) for Pivot and `tstats`. Use it to accelerate security/ops searches and standardize field names across sourcetypes.

### 28. How do you configure Splunk to monitor a log file on a local system?

In `inputs.conf`, add `[monitor:///var/log/app.log]` with `sourcetype`, `index`, and optional `host`. Restart Splunk or deploy via forwarder.

### 29. What is the purpose of the `inputs.conf` file in Splunk?

`inputs.conf` defines data inputs—files (`monitor`), network (TCP/UDP), scripts, and modular inputs. It controls what data enters Splunk and initial metadata (host, sourcetype, index).

### 30. How do you use the `sort` command in SPL?

`| sort - count` orders results ascending/descending by field. Use a minus prefix (`- field`) for descending numeric or lexicographic order.

### 31. What is a Splunk knowledge object and give three examples?

Knowledge objects are reusable user-created entities: **saved searches**, **field extractions**, **tags**, **macros**, **workflow actions**, **lookups**, and **data models**. They accelerate analysis and standardize team practices.

### 32. How do you use the `rename` command in SPL?

`| rename OLD_FIELD as new_field` renames fields in results. Use it to normalize output column names for dashboards and exports.

### 33. What is the purpose of Splunk's HTTP Event Collector (HEC)?

HEC is an HTTP(S) API for sending events directly to Splunk without a forwarder. It is used for applications, containers, and cloud services that push JSON logs at scale.

### 34. How do you use the `head` and `tail` commands in SPL?

`| head 100` returns the first 100 events; `| tail 100` the last 100 after prior processing. Use them to sample large result sets quickly.

### 35. What is a Splunk index-time field extraction versus a search-time field extraction?

Index-time extractions are applied during ingestion and stored in tsidx—fast but permanent. Search-time extractions apply during queries—flexible and changeable without re-indexing.

### 36. How do you configure Splunk's `outputs.conf` for a Universal Forwarder?

Define `[tcpout:default-autolb-group]` with `server=host:9997`, set `defaultGroup`, enable `compressed=true`, and optionally `clientCert` for TLS. Deploy to `system/local` or via deployment server.

### 37. What is the purpose of the `_internal` index in Splunk?

`_internal` stores Splunk's own diagnostic logs and metrics (splunkd, metrics.log, scheduler). Use it to troubleshoot Splunk performance, ingestion, and search issues.

### 38. How do you use the `makeresults` command in SPL for testing?

`| makeresults count=5 | eval x=random()` generates synthetic events without searching an index. Use it to prototype SPL logic and test expressions.

### 39. What is a Splunk macro and how do you create one?

A macro is a parameterized SPL snippet, e.g., `` `security_filter(1)` ``. Create under **Settings → Advanced search → Search macros** or in `macros.conf` with `definition` and `arguments`.

### 40. How do you use the `iplocation` command in SPL?

`| iplocation src_ip` adds geographic fields (City, Country, lat/lon) from MaxMind DB. Use it to map client IPs and detect anomalous geographies.

### 41. What is the purpose of Splunk's `_audit` index?

`_audit` records user activity—logins, searches run, configuration changes. It is essential for security compliance and investigating who accessed or changed what.

### 42. How do you configure Splunk's time zone settings for correct timestamp parsing?

Set `TZ` in `props.conf` per sourcetype or host; configure server timezone in `server.conf`; use `timezone` in user preferences for UI display. Ensure application logs include timezone offsets when possible.

### 43. What is the `splunkd.log` file and what information does it contain?

`splunkd.log` is the main Splunk daemon log—startup, errors, clustering, search execution, and configuration issues. It is the first place to look when Splunk components fail or crash.

### 44. How do you use the `fieldsummary` command to understand data quality?

`| fieldsummary maxvals=20` shows field names, types, counts, and sample values in the current result set. Use it when onboarding new sourcetypes to discover available fields.

### 45. What is the purpose of Splunk's `_metrics` index?

`_metrics` stores internal Splunk performance metrics in metric store format. Use it for platform health monitoring and capacity troubleshooting.

---

## Intermediate Questions

### 1. How do you implement Splunk's `tstats` command for accelerated searches over data models?

Enable acceleration on the data model, then query with ``| tstats summariesonly=true count from datamodel=Network_Traffic.All_Traffic by All_Traffic.src``. Use `summariesonly=true` to force accelerated data for speed.

### 2. Configure index clustering with RF=3 and SF=2

In `server.conf` on Cluster Manager: `[clustering] replication_factor=3, search_factor=2`. Deploy cluster bundle; verify all buckets reach RF=3 and SF=2 in cluster dashboard.

### 3. Summary indexing for dashboard performance

Create a scheduled search that aggregates key metrics and writes to a summary index via `collect`. Point dashboards at the summary index for long time ranges.

### 4. Bucket lifecycle (hot, warm, cold, frozen)

Hot buckets accept writes; warm are closed local buckets; cold are older local or SmartStore buckets; frozen are archived offline. Configure via `indexes.conf` `maxHotBuckets`, `maxWarmBuckets`, `coldPath`, `frozenTimePeriodInSecs`, and SmartStore `remotePath`.

### 5. Search Head Clustering (SHC) for high availability

Deploy 3+ search heads with `shclustering` enabled, connect to deployer, elect captain. Users connect via SHC VIP; artifacts and KV store replicate for failover.

### 6. Forwarder load balancing across indexers

In `outputs.conf`, list multiple indexers in `server=` and enable `autoLBFrequency` with `autoLB=true`. Forwarders rotate TCP connections to distribute load.

### 7. Role-based access control by index and sourcetype

In `authorize.conf`, define roles with `srchIndexesAllowed`, `srchFilter` (e.g., `sourcetype!=privileged`), and index exclusions. Assign users via LDAP/SSO groups mapped to roles.

### 8. `join` vs `lookup` for performance

`join` runs a subsearch merge—expensive at scale. `lookup` reads a static table—O(1) per event enrichment. Prefer `lookup` for asset/threat intel; use `join` only on small, pre-filtered datasets.

### 9. Data model acceleration for pivot performance

Accelerate the DM on indexers; Splunk builds summaries on schedule. Pivot and `tstats` queries use summaries instead of scanning raw events—trade disk and rebuild CPU for query speed.

### 10. SmartStore cold buckets on S3

Configure S3 volume in `indexes.conf` `remotePath = volume:s3volume/...` and enable SmartStore on the index. Cold buckets upload to S3; local cache serves frequent queries.

### 11. `streamstats` for running calculations

`| streamstats avg(response_time) as rolling_avg by host` computes running statistics in event order. Use for per-session or time-ordered sequences unlike `stats` which aggregates all events.

### 12. `eventstats` vs `stats`

`stats` collapses events into groups; `eventstats` adds aggregate values back to each original event (e.g., `| eventstats avg(duration) as avg_by_host by host`). Use `eventstats` when you need per-event context with group statistics.

### 13. Alert suppression for duplicate notifications

Configure alert **Throttle** (suppress for X period per field) or use `suppress` in correlation searches (ES). Prevents alert storms during ongoing incidents.

### 14. Multi-site clustering for disaster recovery

Enable multi-site on CM with `available_sites`, `site_replication_factor`, `site_search_factor`. Place peers in `site` attributes; searches respect site affinity for performance and DR.

### 15. `predict` for time series forecasting

`| timechart span=1h count | predict count as prediction` applies forecasting to numeric time series. Use for capacity trending with awareness of seasonality limitations.

### 16. `cluster` command for grouping similar events

`| cluster t=0.8 showcount=true` groups similar `_raw` patterns. Useful for log pattern discovery and deduplicating noisy repeated messages.

### 17. Workflow actions for interactive links

Define in `workflow-actions.conf` to link field values to external URLs (ticketing, CMDB). Example: click `src_ip` opens firewall lookup page with token replacement.

### 18. Indexer Discovery configuration

On CM enable indexer discovery; on forwarders set `master_uri` in `server.conf` `[indexer_discovery]`. Forwarders receive dynamic `outputs.conf` updates when cluster topology changes.

### 19. `anomalydetection` command

`| anomalydetection response_time` flags outliers in numeric fields using statistical methods. Useful for metric and performance anomaly triage without MLTK.

### 20. `map` command usage

`| map search="| search index=threat intel_ip=$src_ip$"` runs a subsearch per input row. Appropriate only for small input sets; prefer lookups for large-scale enrichment.

### 21. Data integrity control

Enable `enableDataIntegrityControl` in `server.conf` for signed buckets (tamper detection). Critical for compliance environments requiring proof logs were not altered.

### 22. Search scheduler concurrency management

Tune `scheduler.conf` `max_searches_perc`, per-user limits in `authorize.conf`, and use search quotas. Separate scheduled search windows for heavy reporting vs operational searches.

### 23. `geostats` for geographic visualization

`| iplocation src_ip | geostats latfield=lat longfield=lon count by Country` produces map-ready aggregated data. Requires valid lat/lon fields.

### 24. `accum` for running totals

`| timechart count | accum` adds running sum column. Use for cumulative event counts over time.

### 25. Field aliases for normalization

In `props.conf`, `FIELDALIAS-src_ip = client_ip AS src_ip` maps varying field names to one canonical name. Enables unified searches across sourcetypes without per-search `eval`.

### 26. Heavy Forwarder for routing and filtering

Configure `outputs.conf` `tcpout` with multiple groups; use `transforms.conf` and `props.conf` to route by host/sourcetype to different indexes. Filter with `regex` transforms to drop noise before indexers.

### 27. `bucket` command for time aggregations

`| bucket _time span=5m | stats count by _time` groups events into fixed time bins. Alternative to `timechart` when you need custom stats per bin.

### 28. `appendcols` for combining results

`| stats count by host | appendcols [search index=other | stats avg(latency) by host]` adds columns from a subsearch. Requires matching keys; watch subsearch size limits.

### 29. `inputlookup` and `outputlookup`

`| inputlookup asset_inventory` reads CSV/KV lookup; `| outputlookup append=t asset_inventory` writes results. Manage files in `lookups/` or KV store collections.

### 30. Deployment Server at scale

Use server classes in `serverclass.conf` with whitelist/blacklist, staged rollout, and phone-home monitoring. DS pushes `apps/` to UFs; never use DS for indexer cluster config (use Deployer).

### 31. `sendalert` for programmatic alerting

`| sendalert myaction param.value="$result$"` triggers a custom alert action from SPL. Used in advanced correlation and SOAR integration.

### 32. `sistats` vs `stats` in distributed search

`sistats` performs partial aggregation on indexers before merging on SH—more efficient for high-cardinality `stats` at scale. Standard `stats` may pull more raw data to search head.

### 33. `typelearner` for automatic field typing

`| typelearner` infers field types from sample values. Useful during data onboarding to understand field quality before building data models.

### 34. Index-time anonymization for PII

Use `SEDCMD` or `transforms.conf` `REGEX` with `DEST_KEY` masking in `props.conf` at index time. Irreversible—test carefully; prefer tokenization for reversible pseudonymization.

### 35. `contingency` for cross-tabulation

`| contingency user action` builds co-occurrence tables between fields. Used for statistical correlation analysis in security and UX analytics.

### 36. `multikv` for multi-value key-value parsing

`| multikv fields src, dst` parses key-value segments in logs. Useful for firewall, WMI, and structured text logs.

### 37. `crawl` for filesystem discovery

`splunk crawl` (legacy) discovers local log files for potential monitoring. Largely replaced by explicit `monitor` inputs and deployment automation.

### 38. Alert actions for automated remediation

Configure custom alert actions (webhook, script, Adaptive Response) in `alert_actions.conf`. Pass search results as parameters to ticketing or orchestration tools.

### 39. `xyseries` for pivot tables

`| xyseries _time host count` reshapes data for charting with hosts as series. Common after `chart` command for visualization-friendly format.

### 40. `trendline` for moving averages

`| timechart count | trendline count sma5` adds simple moving average. Use for smoothing noisy time series in operational dashboards.

---

## Advanced Questions

### 1. Global enterprise architecture — 50TB/day, 5 DCs, multi-site, SmartStore, 7-year retention

- **Multi-site clustering:** 5 sites with `site_replication_factor` ensuring copies across geographies; origin site holds searchable primaries for local queries.
- **Indexer fleet:** Hundreds of peers sized ~100–200GB/day each; hot on NVMe, SmartStore S3 for cold; frozen to Glacier with compliance holds.
- **Search tier:** Multiple SHC pools per region + federated search for global analytics; workload management for tier-1 security vs reporting.
- **Ingestion:** Tiered UF → regional intermediate forwarders → HEC for cloud-native; DSP for filtering to control license and storage costs.
- **Compliance:** WORM S3, encryption, `_audit`, legal hold processes; documented restore/thaw for investigations only.
- **Operations:** Monitoring Console + Observability Cloud for platform metrics; automated capacity alerts at 70% sustained utilization.

### 2. Splunk SIEM for financial services (SOX, PCI-DSS)

- **ES deployment:** CIM-compliant data models, correlation searches mapped to MITRE/use cases, risk-based alerting to reduce noise.
- **Real-time detection:** Low-latency indexes for auth/payment/DLP; `tstats` acceleration for cardholder data environment boundaries.
- **Automated response:** Adaptive Response integrated with SOAR; dual approval for containment actions in production.
- **Compliance reporting:** Scheduled PDF/CSV reports from saved searches; prove log retention, access reviews, and change control via `_audit`.
- **Segmentation:** Separate indexes and RBAC for PCI scope; tokenization of PAN where possible at ingest.

### 3. MLTK for predictive anomaly detection on infrastructure metrics

- **Training:** Collect 30–90 days baseline per metric/host; use `fit` commands (e.g., `LinearRegression`, `PCA`) in MLTK.
- **Validation:** Hold-out test set; tune false positive rate; compare against rule-based detectors.
- **Production:** Scheduled `apply` producing anomaly scores to summary index; alert on score threshold; retrain monthly.
- **Governance:** Document model versions; monitor drift; fallback rules if model service unavailable.

### 4. 1M events/sec ingestion with sub-second security search

- **Ingestion:** Massive indexer farm, HEC with LB, Kafka ingest + DSP pre-processing, aggressive noise filtering.
- **Parsing:** Structured logging (JSON), minimal index-time extractions, parallel ingestion pipelines.
- **Search:** High SF, data model acceleration, summary indexes for dashboards; dedicated RT search pool; metrics store for KPIs.
- **Architecture:** Multi-site, rack-aware placement, network 25G+, separate management network; continuous load testing.

### 5. ITSI for 500-service dependency tree

- **Modular:** Build service templates; auto-import from CMDB/discovery tools.
- **KPIs:** Error rate, latency p95, saturation per service; threshold policies with dynamic baselines.
- **Health scores:** Weight child services; propagate degradation up the tree.
- **Episodes:** Correlate KPI breaches; episode policies route to correct team; integrate with PagerDuty/ServiceNow.

### 6. Observability platform — logs, metrics, traces correlation

- **Instrumentation:** OpenTelemetry across services; common `trace_id`, `service.name`, `deployment.environment`.
- **Splunk:** Logs to indexers via HEC; traces/metrics to Observability Cloud or metrics index.
- **Correlation:** SPL joins `trace_id` from logs with trace analytics; unified service dashboards.
- **Incident workflow:** Start from alert → logs → trace waterfall → infrastructure metrics in one timeline.

### 7. Federated search across security zones

- **Federated providers:** Each zone runs Splunk with local data sovereignty; central SHC federates providers with role-filtered queries.
- **Access:** SSO with zone-specific credentials; `srchFilter` enforced on provider; no data movement across zones by default.
- **Use case:** Central SOC searches EU and US stacks without consolidating PHI/PII into one index.

### 8. Real-time fraud detection for payment processing

- **Data:** Transaction logs, device fingerprint, geo velocity, chargeback history in lookups.
- **SPL:** Sliding window stats on amount, merchant, card; `streamstats` for velocity; ML anomaly on behavioral baselines.
- **Alerts:** Risk score > threshold triggers step-up auth via AR webhook; suppress during known marketing events.
- **Latency:** Sub-second detection requires HEC + accelerated summaries, not batch reports.

### 9. Data fabric search (Splunk, S3, external DBs)

- **Federated analytics / DFS:** Query datasets in place using federated providers and catalog integrations.
- **Pattern:** Hot data in Splunk; historical in S3 via SmartStore or Amazon S3 Federated Provider; JDBC for relational enrichment.
- **Governance:** Unified RBAC layer; audit all cross-source queries.

### 10. GDPR compliance monitoring platform

- **Data mapping:** Indexes tagged by data category; processing activity lookups.
- **DSR handling:** Saved searches locate subject data across indexes; pseudonymization at ingest.
- **Breach detection:** Alerts on abnormal bulk export, privilege escalation, cross-border transfer anomalies.
- **Retention:** Automated deletion workflows after retention expiry with audit proof.

### 11. UBA for insider threats with ES integration

- **Data:** Auth, HR offboarding, DLP, cloud, endpoint—identity normalized.
- **UBA models:** Peer group baselines; exfiltration and privilege anomaly models.
- **ES integration:** UBA threats create notables with risk objects; analysts triage in Incident Review.
- **Tuning:** 4–6 week learning; adjust thresholds by department risk profile.

### 12. Auto-scaling indexer capacity

- **Cloud:** Auto Scaling Groups / MIGs triggered by `_internal` ingestion lag, CPU, queue depth metrics.
- **Splunk:** Indexer Discovery adds peers; cluster rebalance after scale-out; CM coordinates.
- **Dynamic storage:** SmartStore cache scales with instance store; S3 for bulk.
- **Guardrails:** Cooldown periods; max scale limits; cost alerts.

### 13. Adaptive Response with SOAR integration

- **Playbooks:** Map ES AR actions to SOAR playbooks (isolate host, disable user, block IP).
- **Human in the loop:** Approval gates for destructive actions; automatic enrichment only.
- **Metrics:** Track MTTR, false positive rate, action success/failure in SOAR.

### 14. Network security monitoring — lateral movement, exfil, C2

- **Data:** NetFlow, DNS, proxy, IDS, firewall, EDR.
- **Detection:** Beaconing (periodic low-volume DNS), rare domains, internal scanning, large outbound transfers.
- **Real-time:** ES correlation + Risk Based Alerting; integrate threat intel lookups.
- **Response:** Firewall/EDR blocks via AR with documented rollback.

### 15. Mission Control for unified SecOps

- **Integrate:** ES, UBA, SOAR, threat intel in Mission Control workspace.
- **Workflow:** Queue-based triage, analyst assignment, SLA tracking, metrics on detection efficacy.
- **Reporting:** Executive dashboards on mean time to detect/respond.

### 16. Healthcare Splunk — HIPAA

- **PHI controls:** Dedicated indexes, encryption, access logging, minimum necessary RBAC.
- **Masking:** Tokenize MRN at HF; restrict raw access roles.
- **Breach workflow:** Alerts on bulk PHI access; incident runbooks; BAA with Splunk Cloud.
- **Audit:** `_audit` + Cloud audit logs retained per policy.

### 17. Observability Cloud + Enterprise unified view

- **Dual telemetry:** OTel → O11y Cloud metrics/traces; logs → Enterprise via HEC.
- **Correlation:** `trace_id` linking; unified service entity model.
- **Dashboards:** App for Infrastructure showing logs-to-traces drilldown.

### 18. DevSecOps platform with CI/CD integration

- **Pipelines:** HEC ingest build/deploy events; vulnerability scan results indexed.
- **Detection:** Correlate new CVE deployments with error spikes; IaC change detection.
- **Gates:** Fail pipeline on critical findings; Splunk dashboards for deployment health.

### 19. DSP for real-time enrichment and routing

- **Pipelines:** SPL2 filters PII, enriches with lookups, routes security vs ops indexes.
- **Sources:** Kafka, HEC; sinks to Splunk, S3, security tools.
- **Benefit:** Reduces indexer load; sub-second routing decisions at edge.

### 20. Retail real-time business intelligence

- **Data:** POS transactions, inventory, web clickstream, loyalty IDs in unified model.
- **SPL:** Real-time sales dashboards; `streamstats` for basket analysis; anomaly on returns fraud.
- **Scale:** Summary indexes for exec KPIs; raw retained shorter for stores.

### 21. Multi-cloud monitoring (AWS, GCP, Azure)

- **Apps:** Splunk Add-ons for each cloud pull CloudTrail, Activity Logs, Activity Log.
- **Normalization:** CIM-compliant fields; common `cloud_provider` tag.
- **Central SHC:** Federated or single stack with provider-specific indexes and unified security dashboards.

### 22. IoT manufacturing — 100k sensors

- **Ingestion:** MQTT/Kafka → HEC/DSP; metric protocol for time-series at scale.
- **Detection:** `mstats`/`tstats` anomaly on vibration/temperature; MLTK for predictive maintenance.
- **Edge:** Optional edge preprocessing to reduce central volume.

### 23. Risk-based alerting (RBA) for SOC alert fatigue

- **Aggregate:** Multiple findings → risk incident by `risk_object`.
- **Threshold:** Page when cumulative risk > critical; lower severities create tickets.
- **Measure:** Notables/hour, true positive rate, analyst capacity.

### 24. Encrypted log ingestion and analysis

- **Ingest:** Apps send encrypted payloads; HEC TLS in transit; store encrypted fields tokenized.
- **Keys:** KMS/HSM integration; decryption only on dedicated investigation SH with break-glass audit.
- **Selective:** `eval` decrypt only for authorized roles during active investigation.

### 25. Content pack deployment for MSSP (100 customers)

- **Tenant isolation:** Separate indexes/HEC tokens per customer; RBAC per tenant.
- **Content:** ES analytic stories deployed via CI/CD with customer-specific tuning parameters.
- **Versioning:** Git-managed content packs; staged rollout; per-tenant enablement matrix.
- **Reporting:** Per-customer SLA dashboards; centralized SOC with strict data boundaries.

---

## Rapid-Fire Questions

### 1. What is the default Splunk web interface port?

**8000** (HTTPS often **8089** for management API; forwarding typically **9997**).

### 2. What SPL command do you use to count events by field value?

**`stats`** — e.g., `| stats count by host`.

### 3. What is the purpose of the `_time` field in Splunk?

The parsed event timestamp used for time-range filtering and chronological ordering.

### 4. What does `index=main sourcetype=syslog` search for?

Events in the **main** index with sourcetype **syslog**.

### 5. What is the difference between `stats` and `eventstats` in SPL?

**`stats`** collapses events into groups; **`eventstats`** adds aggregate values back to each original event row.

### 6. What SPL command extracts fields using regex patterns?

**`rex`**.

### 7. What is the purpose of Splunk's `_raw` field?

Contains the original log line exactly as received, used for display and search-time extraction.

### 8. What does the `head 10` command do in SPL?

Returns the first **10** events of the result set.

### 9. What is a Splunk bucket and what are its states?

A time-partitioned directory of indexed data; states: **hot**, **warm**, **cold**, **frozen**, **thawed**.

### 10. What is the purpose of Splunk's `props.conf` `TIME_FORMAT` setting?

Defines the `strptime` format for parsing timestamps from event text.

### 11. What SPL command do you use to calculate a moving average?

**`trendline`** (e.g., `sma5`) or **`streamstats`** for custom rolling windows.

### 12. What is the difference between Splunk's `search` and `where` commands?

**`search`** filters events (often at start of query); **`where`** filters tabular results after commands like `stats`.

### 13. What is the purpose of Splunk's `_indextime` field?

Unix timestamp when the event was **indexed** (written to disk), used to measure ingestion lag.

### 14. What does `| stats count by host` return?

A table of **host** values with **count** of events per host.

### 15. What is the Splunk `kvstore` used for?

JSON document store for lookups, collections, and app backend storage (e.g., modular inputs).

### 16. What is the purpose of Splunk's `transforms.conf` `REGEX` setting?

Pattern used in transform stanzas for extraction, routing, or masking when referenced from `props.conf`.

### 17. What SPL command do you use to join two searches?

**`join`**.

### 18. What is the difference between Splunk's `rex` and `extract` commands?

**`rex`** uses inline regex in SPL; **`extract`** uses field extraction rules from `props.conf`/KV settings.

### 19. What is the purpose of Splunk's `_sourcetype` field?

Identifies the data format/rule set applied to the event (metadata copy of sourcetype).

### 20. What does `| timechart span=1h count` display?

Hourly event counts as a time series chart.

### 21. What is the Splunk `fishbucket` used for?

Tracks file read positions (CRC/hash) so forwarders don't re-send already-read log data.

### 22. What is the purpose of Splunk's `inputs.conf` `monitor` stanza?

Continuously watches a file or directory for new data to ingest.

### 23. What SPL command do you use to remove duplicate events?

**`dedup`**.

### 24. What is the difference between Splunk's `append` and `appendcols` commands?

**`append`** stacks rows (union); **`appendcols`** adds columns from a subsearch matched by row order/keys.

### 25. What is the purpose of Splunk's `_meta` field?

Contains host, source, sourcetype metadata (comma-separated key-value pairs) for the event.

### 26. What does `| top limit=10 sourcetype` return?

Top **10** most common **sourcetype** values with counts and percentages.

### 27. What is the Splunk `dispatch` directory used for?

Stores search job artifacts (results, metadata, temp files) under `$SPLUNK_HOME/var/run/splunk/dispatch`.

### 28. What is the purpose of Splunk's `outputs.conf` `tcpout` stanza?

Configures TCP forwarding of data to remote indexers (host:9997).

### 29. What SPL command do you use to calculate percentiles?

**`stats`** or **`eventstats`** with `p95()`, `perc90()`, etc.

### 30. What is the difference between Splunk's `eval` and `fieldformat` commands?

**`eval`** creates/modifies field values; **`fieldformat`** changes display formatting only (underlying value unchanged).

### 31. What is the purpose of Splunk's `_subsecond` field?

Stores fractional seconds for high-precision timestamps.

### 32. What does `| rare limit=5 host` return?

The **5** least common **host** values.

### 33. What is the Splunk `lookups` directory used for?

Stores CSV and other lookup table files used by `lookup` commands.

### 34. What is the purpose of Splunk's `props.conf` `BREAK_ONLY_BEFORE` setting?

Only break events before lines matching the regex—used for multi-line event merging.

### 35. What SPL command do you use to create a lookup table from search results?

**`outputlookup`**.

### 36. What is the difference between Splunk's `transaction` and `stats` commands for session analysis?

**`transaction`** groups related events into multi-event transactions; **`stats`** aggregates fields into single rows per group.

### 37. What is the purpose of Splunk's `_cd` field?

Chunk/directory identifier pointing to the bucket location on disk (internal storage reference).

### 38. What does `| makeresults count=10` generate?

**10** synthetic events with no indexed data.

### 39. What is the Splunk `modinput` used for?

**Modular input** framework for custom scripted/API data inputs (e.g., AWS, DB connectors).

### 40. What is the purpose of Splunk's `transforms.conf` `DEST_KEY` setting?

Specifies which field or metadata key the transform writes to (e.g., `_meta`, a field name).

### 41. What SPL command do you use to format numbers and dates?

**`fieldformat`** (display) or **`eval`** with `strftime()` / `tostring()`.

### 42. What is the difference between Splunk's `iplocation` and `geoip` commands?

**`iplocation`** uses Splunk's built-in MaxMind DB; **`geoip`** (legacy/ES) may use different DBs—functionally both geolocate IPs.

### 43. What is the purpose of Splunk's `_bkt` field?

Bucket ID identifying which index bucket stores the event.

### 44. What does `| sistats count by host` do differently from `stats`?

**`sistats`** performs partial aggregation on indexers before merging—more efficient for distributed high-cardinality stats.

### 45. What is the Splunk `splunk_search_history` index used for?

Stores users' search history for audit and UI recent-search features (if enabled).

### 46. What is the purpose of Splunk's `props.conf` `MAX_EVENTS` setting?

Maximum lines merged into a single event during line merging—prevents unbounded multi-line events.

### 47. What SPL command do you use to pivot data into a matrix format?

**`xyseries`** (often after `chart`).

### 48. What is the difference between Splunk's `search` mode and `fast` mode?

**Fast mode** skips expensive field extraction for quicker results; **verbose/smart** modes extract all fields.

### 49. What is the purpose of Splunk's `_serial` field?

Internal event serial number within a bucket used for ordering/tie-breaking.

### 50. What does `| gentimes start=-7d increment=1d` generate?

Synthetic events with `_time` at **daily** intervals starting **7 days** ago (one per day).
