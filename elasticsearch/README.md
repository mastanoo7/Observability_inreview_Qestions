# Elasticsearch — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## Shard Allocation & Cluster Management

1. Your Elasticsearch cluster is showing a RED status with 15% of shards unassigned. The cluster has sufficient disk space and all nodes are online. Walk through your systematic approach to diagnosing and resolving unassigned shards.

2. Explain the difference between primary shard allocation and replica shard allocation in Elasticsearch. How does the cluster routing allocation algorithm decide where to place shards, and what parameters can you tune to influence this?

3. A cluster has 1,000 indices with 5 shards each, resulting in 5,000 shards across 10 nodes. You're experiencing high JVM heap pressure and slow search performance. How do you implement a shard consolidation strategy without data loss?

4. Describe how Elasticsearch's shard allocation awareness works for a multi-zone deployment. How do you configure zone awareness to ensure that primary and replica shards are never on the same availability zone?

5. Your Elasticsearch cluster is performing shard rebalancing continuously, causing high I/O and degraded performance. How do you diagnose the cause of continuous rebalancing and implement throttling without stopping rebalancing entirely?

6. Explain the concept of shard over-allocation and its impact on cluster performance. How do you calculate the optimal number of shards for a given data volume and query pattern?

7. A new index was created with 50 shards when it should have had 5. The index is already receiving production traffic. How do you shrink the index without downtime, and what are the prerequisites for the shrink operation?

8. How do you implement forced shard allocation routing to move specific shards to specific nodes for performance optimization, and when would you use `cluster.routing.allocation.exclude` versus `index.routing.allocation.require`?

9. Describe the impact of having too many small shards (shard proliferation) on Elasticsearch cluster stability, and how you would implement an index lifecycle management strategy to prevent it.

10. Your cluster is experiencing "too many open files" errors on data nodes. How does this relate to shard count, and what OS-level and Elasticsearch-level changes do you make to resolve it?

---

## Cluster Tuning & Performance

11. An Elasticsearch cluster's search latency has increased from 50ms to 5 seconds over two weeks without any change in query patterns. How do you systematically identify the root cause using cluster stats, node stats, and hot threads API?

12. Explain how Elasticsearch's circuit breakers work. Describe a production scenario where a circuit breaker prevented a cluster crash, and a scenario where misconfigured circuit breakers caused unnecessary request rejections.

13. Your Elasticsearch cluster is experiencing high GC pressure with frequent stop-the-world GC pauses. How do you diagnose the GC issue, tune JVM settings, and determine whether the issue is heap sizing, GC algorithm selection, or query patterns?

14. Describe how you would tune Elasticsearch's thread pool settings for a cluster that handles a mix of heavy indexing workloads and complex aggregation queries simultaneously.

15. How do you implement Elasticsearch's adaptive replica selection (ARS) to improve search performance, and what metrics do you monitor to verify that ARS is making better routing decisions than round-robin?

16. A bulk indexing operation is causing search latency to spike. How do you implement indexing throttling using `indices.store.throttle` settings and bulk queue management to balance indexing and search performance?

17. Explain how Elasticsearch's query cache, request cache, and field data cache work differently. Describe a scenario where aggressive caching caused memory pressure and how you tuned the cache eviction policies.

18. How do you implement index sorting to improve search performance for specific query patterns, and what are the trade-offs in terms of indexing throughput and storage overhead?

19. Describe how you would use the Elasticsearch Profile API to identify slow query components and optimize a complex nested aggregation query.

20. Your Elasticsearch cluster has high disk I/O during merge operations. How do you tune merge scheduling, segment sizes, and merge policies to reduce I/O impact on search performance?

---

## Heap Issues & JVM Tuning

21. An Elasticsearch node is experiencing repeated OOMKilled events in Kubernetes. The heap is set to 30GB on a 64GB node. How do you diagnose what is consuming memory outside the heap, and what are the correct memory settings for Elasticsearch in Kubernetes?

22. Explain the 50% heap rule for Elasticsearch. Why is setting heap above 32GB potentially counterproductive, and how does the JVM compressed ordinary object pointer (OOP) threshold affect this?

23. Your Elasticsearch nodes are experiencing long GC pauses (>10 seconds) that are causing cluster instability and node timeouts. Walk through your GC tuning approach, including GC algorithm selection, heap sizing, and GC log analysis.

24. How do you diagnose and resolve field data cache eviction storms in Elasticsearch, and what mapping changes would you make to prevent field data from being loaded into heap for high-cardinality fields?

25. Describe how Elasticsearch uses off-heap memory for file system cache, and how you would tune the balance between JVM heap and OS file system cache for optimal performance.

26. A node's heap usage is consistently at 95% even after GC. How do you use the Elasticsearch nodes stats API and JVM heap dump analysis to identify the source of heap pressure?

27. How do you implement JVM GC logging for Elasticsearch in a way that captures enough detail for performance analysis without impacting production performance?

---

## Indexing Latency & Throughput

28. Your Elasticsearch indexing throughput has dropped from 100,000 docs/second to 10,000 docs/second. No configuration changes were made. How do you diagnose whether this is a client-side, network, or cluster-side issue?

29. Explain how Elasticsearch's refresh interval affects both indexing throughput and search freshness. How do you dynamically tune the refresh interval during bulk indexing operations?

30. Describe how you would implement a high-throughput indexing pipeline for a log analytics use case that needs to index 1TB of data per day, including bulk sizing, client-side batching, and cluster-side tuning.

31. How does Elasticsearch's translog work, and how do you tune `translog.durability` and `translog.flush_threshold_size` to balance data durability with indexing performance?

32. A bulk indexing request is returning `429 Too Many Requests` errors. How do you diagnose whether this is a bulk queue overflow, a circuit breaker trip, or a resource exhaustion issue, and what are your remediation options?

33. Explain how Elasticsearch's index buffer (`indices.memory.index_buffer_size`) affects indexing performance and memory usage. How do you tune this for a cluster with mixed indexing and search workloads?

34. How do you implement index aliases with write aliases to enable zero-downtime index migrations and rolling index strategies for time-series data?

---

## Split Brain & Cluster Stability

35. Explain the split-brain problem in Elasticsearch and how the `minimum_master_nodes` setting (pre-7.x) and the voting configuration (7.x+) prevent it. Describe a production scenario where split-brain occurred and its impact.

36. Your Elasticsearch cluster has experienced a network partition that separated 3 master-eligible nodes into groups of 2 and 1. What happens to each partition, and how do you safely recover the cluster after the partition heals?

37. How does Elasticsearch 7.x's cluster coordination (based on Raft consensus) differ from the previous Zen discovery mechanism, and what operational changes does this require?

38. Describe how you would implement a 3-zone Elasticsearch deployment with master-eligible nodes to ensure that the cluster can survive the loss of an entire availability zone without split-brain.

39. A master node election is taking longer than expected, causing cluster instability. How do you diagnose the election process using cluster logs and what configuration changes improve election stability?

40. How do you safely remove a master-eligible node from an Elasticsearch cluster's voting configuration without causing a cluster disruption?

---

## Hot-Warm Architecture

41. Describe how you would implement a hot-warm-cold architecture in Elasticsearch for a log analytics use case with the following requirements: 7 days hot (SSD), 30 days warm (HDD), 1 year cold (object storage), 7 years frozen (object storage).

42. How do you configure Elasticsearch's ILM (Index Lifecycle Management) to automatically transition indices through hot-warm-cold tiers based on age and size, including the rollover conditions?

43. A hot-to-warm shard migration is taking 6 hours and impacting cluster performance. How do you tune the migration speed using `cluster.routing.allocation.node_concurrent_recoveries` and related settings?

44. Explain how Elasticsearch's searchable snapshots work in the cold and frozen tiers. What are the query performance implications, and how do you decide which data to keep as searchable snapshots versus fully restored?

45. How do you implement data tier routing in Elasticsearch to ensure that new indices are always created on hot nodes and that ILM transitions move data to the correct tier?

---

## Index Lifecycle Management (ILM)

46. An ILM policy is stuck in the "check-rollover-ready" phase for an index that has exceeded the rollover conditions. How do you diagnose why ILM is not progressing and manually advance the index to the next phase?

47. Describe how you would implement an ILM policy for a security logging use case that requires: hourly rollover during business hours, daily rollover off-hours, 90-day retention, and immutable storage for compliance.

48. How do you handle ILM policy updates for existing indices that are already managed by an older policy version, and what are the risks of updating ILM policies in production?

49. An ILM rollover is creating new indices with incorrect settings because the index template was updated after the ILM policy was applied. How do you fix this without losing data or disrupting ingestion?

50. Explain how ILM interacts with index aliases during rollover. Describe a scenario where alias misconfiguration caused data to be written to the wrong index during an ILM rollover.

---

## Snapshot & Recovery

51. Your Elasticsearch cluster needs to be restored from a snapshot after a catastrophic failure. The snapshot repository is on S3. Walk through the complete recovery procedure, including the order of operations and how you verify data integrity.

52. A snapshot operation is failing with "repository is corrupted" errors. How do you diagnose the corruption, determine which snapshots are still valid, and implement a recovery strategy?

53. How do you implement incremental snapshot strategies for a large Elasticsearch cluster (50TB) to minimize snapshot duration and S3 storage costs while maintaining recovery point objectives?

54. Describe how you would implement cross-cluster replication (CCR) as a disaster recovery mechanism, including the configuration, monitoring, and failover procedures.

55. How do you implement snapshot lifecycle management (SLM) to automate snapshot creation, retention, and deletion, and how do you monitor SLM policy execution for failures?

56. A snapshot restore is taking 12 hours for a 5TB index. How do you optimize the restore speed using concurrent shard recovery settings and network bandwidth tuning?

---

## Performance Troubleshooting

57. An Elasticsearch query that previously returned in 100ms is now taking 30 seconds. The query hasn't changed, but the index has grown from 10GB to 500GB. How do you diagnose and optimize the query for the larger dataset?

58. How do you use the Elasticsearch `_explain` API and `_profile` API together to understand why a specific document is not matching a query and where the query execution time is being spent?

59. Describe how you would diagnose and resolve a "fielddata" memory issue caused by a high-cardinality text field being used in aggregations.

60. Your Elasticsearch cluster is experiencing "rejected execution" errors for search requests. How do you diagnose whether this is a thread pool queue overflow, a circuit breaker trip, or a resource exhaustion issue?

61. How do you implement Elasticsearch's slow log feature to capture queries and indexing operations that exceed specific thresholds, and how do you analyze slow logs to identify optimization opportunities?

62. Describe how you would optimize a complex nested aggregation query that is causing high CPU usage across all data nodes simultaneously.

---

## Kubernetes Deployments

63. You're deploying Elasticsearch on Kubernetes using the Elastic Cloud on Kubernetes (ECK) operator. A data node pod is being evicted due to memory pressure. How do you configure resource requests, limits, and JVM heap settings correctly for Kubernetes?

64. How do you implement persistent storage for Elasticsearch on Kubernetes, including the selection of storage class, volume expansion, and handling of pod rescheduling without data loss?

65. Describe how you would implement Elasticsearch rolling upgrades on Kubernetes using ECK, ensuring that the cluster maintains GREEN status throughout the upgrade process.

66. How do you handle Elasticsearch node discovery in Kubernetes when pods are rescheduled and get new IP addresses, and how does the ECK operator manage this automatically?

67. Describe the challenges of running Elasticsearch on Kubernetes in terms of network performance, storage I/O, and JVM tuning compared to bare-metal deployments.

68. How do you implement Elasticsearch cluster monitoring on Kubernetes, including the collection of JVM metrics, cluster health metrics, and index-level metrics using the Elastic Metricbeat or Prometheus exporter?

69. A Kubernetes-deployed Elasticsearch cluster is experiencing intermittent network partitions due to CNI plugin issues. How do you configure Elasticsearch's network timeouts and discovery settings to be more resilient to transient network issues?

---

## Security & Compliance

70. How do you implement Elasticsearch's role-based access control (RBAC) to support a multi-tenant environment where different teams can only access their own indices?

71. Describe how you would implement field-level security and document-level security in Elasticsearch for a healthcare application that needs to restrict access to PII fields based on user roles.

72. How do you implement audit logging in Elasticsearch to track all data access, configuration changes, and authentication events for compliance with SOC 2 and GDPR requirements?

73. Explain how Elasticsearch's TLS encryption works for both transport layer (node-to-node) and HTTP layer (client-to-node) communication, and how you implement certificate rotation without cluster downtime.

---

## Advanced Scenarios

74. You need to implement a real-time search feature for a product catalog with 100 million documents that requires sub-100ms search latency for complex queries. Describe your Elasticsearch architecture, including index design, mapping optimization, and query strategy.

75. How do you implement Elasticsearch's percolator feature for real-time alerting, where stored queries are matched against incoming documents as they are indexed?

76. Describe how you would implement a multi-cluster Elasticsearch architecture for a global application, including cross-cluster search (CCS) configuration, latency considerations, and failure handling.

77. How do you implement Elasticsearch's vector search capabilities for a machine learning use case, including the configuration of dense vector fields, kNN search parameters, and performance tuning?

78. You're implementing Elasticsearch as the backend for a security information and event management (SIEM) system. Describe the index design, ILM policies, and query optimizations needed to handle 1 billion events per day with sub-second search latency.

---

## Basic Questions

1. What is Elasticsearch and what type of data does it store?
2. What is the difference between an Elasticsearch index, document, and field?
3. What is a shard in Elasticsearch and why does it exist?
4. What is the difference between a primary shard and a replica shard?
5. What is the Elasticsearch REST API and how do you use it to index a document?
6. What is a mapping in Elasticsearch and why is it important?
7. What is the difference between `keyword` and `text` field types in Elasticsearch?
8. How do you perform a basic full-text search in Elasticsearch?
9. What is the Elasticsearch cluster health status (green, yellow, red) and what does each mean?
10. What is the purpose of the `_id` field in an Elasticsearch document?
11. How do you create an index in Elasticsearch with custom settings?
12. What is a Kibana index pattern and how does it relate to Elasticsearch indices?
13. What is the purpose of the `_source` field in an Elasticsearch document?
14. How do you delete a document from an Elasticsearch index?
15. What is the difference between Elasticsearch's `match` and `term` queries?
16. What is an Elasticsearch alias and when would you use one?
17. How do you check the status of an Elasticsearch cluster using the API?
18. What is the purpose of the `refresh_interval` setting in an Elasticsearch index?
19. How do you perform a bulk indexing operation in Elasticsearch?
20. What is the difference between Elasticsearch's `filter` and `query` context?
21. What is an Elasticsearch aggregation and give two examples?
22. How do you configure the number of shards and replicas when creating an index?
23. What is the purpose of the `_cat` API in Elasticsearch?
24. How do you update a document in Elasticsearch using the Update API?
25. What is the difference between Elasticsearch's `index` and `create` operations?
26. What is an Elasticsearch template and when would you use one?
27. How do you search across multiple indices in Elasticsearch?
28. What is the purpose of the `took` field in an Elasticsearch search response?
29. How do you configure Elasticsearch's JVM heap size?
30. What is the difference between Elasticsearch's `must`, `should`, and `must_not` clauses?
31. What is the purpose of the `_version` field in an Elasticsearch document?
32. How do you use Elasticsearch's `range` query for date-based filtering?
33. What is an Elasticsearch node role and name four common node roles?
34. How do you check which node is the master in an Elasticsearch cluster?
35. What is the purpose of the `index.number_of_replicas` setting?
36. How do you use Elasticsearch's `terms` aggregation for faceted search?
37. What is the difference between Elasticsearch's `match_phrase` and `match` queries?
38. How do you configure Elasticsearch's network settings for cluster communication?
39. What is the purpose of the `_shards` field in an Elasticsearch search response?
40. How do you use Elasticsearch's `exists` query to find documents with a specific field?
41. What is the Elasticsearch `_analyze` API used for?
42. How do you configure Elasticsearch's index refresh to improve indexing performance?
43. What is the purpose of the `doc_values` setting in Elasticsearch field mappings?
44. How do you use Elasticsearch's `scroll` API for large result sets?
45. What is the difference between Elasticsearch's `index` and `search` operations in terms of consistency?

---

## Intermediate Questions

1. How do you implement Elasticsearch's ILM (Index Lifecycle Management) policy for a log analytics use case with hot-warm-cold tiers?
2. Describe how you would configure Elasticsearch's shard allocation awareness for a multi-zone deployment.
3. How do you implement Elasticsearch's cross-cluster search (CCS) to query data across multiple clusters?
4. Explain how Elasticsearch's circuit breakers work and how you tune them to prevent OOM errors.
5. How do you implement Elasticsearch's field-level security to restrict access to sensitive fields based on user roles?
6. Describe how you would configure Elasticsearch's snapshot lifecycle management (SLM) for automated backups to S3.
7. How do you implement Elasticsearch's custom analyzer for a multilingual search application?
8. Explain how Elasticsearch's `_reindex` API works and when you would use it.
9. How do you implement Elasticsearch's percolator for real-time document matching against stored queries?
10. Describe how you would configure Elasticsearch's hot-warm architecture for a time-series log analytics platform.
11. How do you implement Elasticsearch's rollup jobs for long-term metric aggregation with reduced storage?
12. Explain how Elasticsearch's `search_after` pagination differs from `from/size` and when each is appropriate.
13. How do you implement Elasticsearch's runtime fields for dynamic field computation at query time?
14. Describe how you would configure Elasticsearch's cross-cluster replication (CCR) for disaster recovery.
15. How do you implement Elasticsearch's data streams for time-series data management?
16. Explain how Elasticsearch's `_bulk` API handles partial failures and how you implement retry logic.
17. How do you implement Elasticsearch's function score query for custom relevance scoring?
18. Describe how you would configure Elasticsearch's index templates with component templates for modular configuration.
19. How do you implement Elasticsearch's geo-spatial queries for location-based search?
20. Explain how Elasticsearch's `significant_terms` aggregation works for anomaly detection.
21. How do you implement Elasticsearch's nested objects versus parent-child relationships for complex data models?
22. Describe how you would configure Elasticsearch's ingest pipeline for data enrichment and transformation.
23. How do you implement Elasticsearch's `collapse` feature for deduplicating search results?
24. Explain how Elasticsearch's adaptive replica selection (ARS) improves search performance.
25. How do you implement Elasticsearch's transform API for creating summarized indices?
26. Describe how you would configure Elasticsearch's TLS encryption for both transport and HTTP layers.
27. How do you implement Elasticsearch's `more_like_this` query for content-based recommendation?
28. Explain how Elasticsearch's `_profile` API helps diagnose slow queries.
29. How do you implement Elasticsearch's watcher for automated alerting based on query results?
30. Describe how you would configure Elasticsearch's index sorting for improved search performance.
31. How do you implement Elasticsearch's `terms_enum` API for autocomplete functionality?
32. Explain how Elasticsearch's `_field_caps` API works for schema discovery.
33. How do you implement Elasticsearch's `knn` search for vector similarity queries?
34. Describe how you would configure Elasticsearch's slow log for performance troubleshooting.
35. How do you implement Elasticsearch's `enrich` processor for IP-to-geolocation enrichment in ingest pipelines?
36. Explain how Elasticsearch's `_cluster/reroute` API works for manual shard allocation.
37. How do you implement Elasticsearch's `alias` with filters for multi-tenant data isolation?
38. Describe how you would configure Elasticsearch's `index.routing.allocation.require` for dedicated node assignment.
39. How do you implement Elasticsearch's `_pit` (point-in-time) API for consistent pagination?
40. Explain how Elasticsearch's `_cat/thread_pool` API helps diagnose performance issues.

---

## Advanced Questions

1. Design an Elasticsearch architecture for a global e-commerce platform handling 10 billion product searches per day with sub-100ms latency, including index design, shard strategy, and caching.
2. How do you implement a multi-tenant Elasticsearch platform for a SaaS application with 1,000 customers, ensuring complete data isolation, per-tenant resource limits, and automated provisioning?
3. Describe how you would implement an Elasticsearch deployment that handles petabyte-scale log analytics with 2-year retention, using searchable snapshots and tiered storage to minimize costs.
4. How do you implement a zero-downtime Elasticsearch major version upgrade for a production cluster with 500TB of data and 24/7 availability requirements?
5. Describe how you would implement an Elasticsearch-based real-time fraud detection system that processes 1 million transactions per minute with sub-second detection latency.
6. How do you implement an Elasticsearch deployment that automatically rebalances shards across nodes as data volume grows, without impacting search performance?
7. Describe how you would implement Elasticsearch's vector search at scale for a machine learning recommendation system with 100 million items and real-time updates.
8. How do you implement an Elasticsearch deployment that survives a complete availability zone failure with zero data loss and automatic failover within 60 seconds?
9. Describe how you would implement an Elasticsearch-based security analytics platform that correlates events across 50 different log sources to detect advanced persistent threats (APTs).
10. How do you implement an Elasticsearch deployment for a healthcare organization with HIPAA compliance requirements, including field-level encryption, access auditing, and data retention policies?
11. Describe how you would implement Elasticsearch's cross-cluster search for a global organization with data sovereignty requirements, ensuring that queries respect geographic data boundaries.
12. How do you implement an Elasticsearch deployment that handles 100,000 concurrent search requests with consistent sub-200ms latency using query caching, replica routing, and request coalescing?
13. Describe how you would implement an Elasticsearch-based observability platform that ingests metrics, logs, and traces from 10,000 microservices with automatic correlation and anomaly detection.
14. How do you implement an Elasticsearch deployment that automatically detects and remediates shard imbalances, hot spots, and performance degradation without manual intervention?
15. Describe how you would implement Elasticsearch's machine learning features for automated anomaly detection in time-series data, including model training, validation, and alert integration.
16. How do you implement an Elasticsearch deployment that provides real-time analytics for a financial trading platform, handling 10 million events per second with microsecond indexing latency?
17. Describe how you would implement an Elasticsearch-based content management system with multilingual search, faceted navigation, and personalized ranking for 1 billion documents.
18. How do you implement an Elasticsearch deployment that automatically scales horizontally based on query load, with automatic shard rebalancing and zero-downtime scaling?
19. Describe how you would implement Elasticsearch's security features for a zero-trust architecture, including mTLS, RBAC, field-level security, and comprehensive audit logging.
20. How do you implement an Elasticsearch deployment that handles time-series data with high write throughput (1 million docs/second) while maintaining fast aggregation query performance?
21. Describe how you would implement an Elasticsearch-based log analytics platform that automatically classifies log events using machine learning and routes them to appropriate response workflows.
22. How do you implement an Elasticsearch deployment for a multi-cloud environment, ensuring consistent performance, data replication, and failover across AWS, GCP, and Azure?
23. Describe how you would implement Elasticsearch's snapshot and restore strategy for a 100TB cluster with 1-hour RPO and 4-hour RTO requirements.
24. How do you implement an Elasticsearch deployment that provides both real-time search (sub-second freshness) and historical analytics (years of data) cost-effectively using tiered storage?
25. Describe how you would implement an Elasticsearch-based customer 360 platform that aggregates data from 20 different source systems, providing real-time unified customer profiles for 500 million customers.

---

## Rapid-Fire Questions

1. What is the default number of primary shards in Elasticsearch 7.x and later?
2. What does a RED cluster health status indicate?
3. What is the purpose of the `_cat/health` API?
4. What is the difference between `keyword` and `text` field types?
5. What does `GET /_cluster/stats` return?
6. What is the purpose of the `index.refresh_interval` setting?
7. What does a YELLOW cluster health status indicate?
8. What is the Elasticsearch `_bulk` API used for?
9. What is the purpose of the `doc_values` field mapping parameter?
10. What does `GET /_cat/indices?v` display?
11. What is the difference between Elasticsearch's `query` and `filter` context?
12. What is the purpose of the `_source` field?
13. What does `GET /_nodes/stats` return?
14. What is the Elasticsearch `fielddata` cache used for?
15. What is the purpose of the `index.number_of_replicas` setting?
16. What does `POST /index/_forcemerge` do?
17. What is the difference between Elasticsearch's `match` and `term` queries?
18. What is the purpose of the `_id` field in a document?
19. What does `GET /_cat/shards?v` display?
20. What is the Elasticsearch `translog` used for?
21. What is the purpose of the `index.blocks.write` setting?
22. What does `GET /_cluster/allocation/explain` return?
23. What is the difference between Elasticsearch's `index` and `create` operations?
24. What is the purpose of the `_version` field?
25. What does `POST /index/_refresh` do?
26. What is the Elasticsearch `circuit_breaker` used for?
27. What is the purpose of the `index.routing.allocation.require` setting?
28. What does `GET /_cat/nodes?v` display?
29. What is the difference between Elasticsearch's `scroll` and `search_after` pagination?
30. What is the purpose of the `index.merge.policy.max_merge_at_once` setting?
31. What does `GET /_cluster/pending_tasks` return?
32. What is the Elasticsearch `_reindex` API used for?
33. What is the purpose of the `index.store.type` setting?
34. What does `GET /_nodes/hot_threads` return?
35. What is the difference between Elasticsearch's `nested` and `object` field types?
36. What is the purpose of the `index.max_result_window` setting?
37. What does `POST /index/_shrink/new_index` do?
38. What is the Elasticsearch `_pit` API used for?
39. What is the purpose of the `cluster.routing.allocation.enable` setting?
40. What does `GET /_cat/recovery?v` display?
41. What is the difference between Elasticsearch's `must` and `filter` clauses?
42. What is the purpose of the `index.lifecycle.name` setting?
43. What does `GET /_ilm/policy` return?
44. What is the Elasticsearch `_enrich` processor used for?
45. What is the purpose of the `cluster.max_shards_per_node` setting?
46. What does `POST /index/_split/new_index` do?
47. What is the difference between Elasticsearch's `terms` and `term` queries?
48. What is the purpose of the `index.hidden` setting?
49. What does `GET /_data_stream` return?
50. What is the Elasticsearch `_transform` API used for?
