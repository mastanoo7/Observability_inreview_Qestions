# Prometheus — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## PromQL & Query Optimization

1. Your `rate()` calculation for a counter metric is returning unexpected spikes after a pod restart. How do you diagnose whether this is a counter reset issue or a genuine traffic spike, and what PromQL adjustments would you make?

2. You need to calculate the 99th percentile latency across all microservices in a Kubernetes cluster using histogram metrics. Write the PromQL query and explain the trade-offs between `histogram_quantile()` accuracy and bucket granularity.

3. A PromQL query using `irate()` is producing noisy, inconsistent results in a Grafana dashboard. Under what production conditions would you switch from `irate()` to `rate()`, and what are the implications for alerting accuracy?

4. You have a recording rule that computes a complex aggregation across 500 time series. The rule evaluation is taking longer than the evaluation interval. How do you diagnose and resolve this, and what are the cascading effects on dependent alerts?

5. Explain how `absent()` and `absent_over_time()` differ in behavior when a target goes down versus when a metric simply stops being scraped. How would you use each in a production alerting strategy?

6. You're using `label_replace()` in a PromQL query to normalize inconsistent label values from different exporters. What are the performance implications of doing this at query time versus at ingestion time via relabeling?

7. A team is using `count_over_time()` on a gauge metric to detect anomalies. Why is this approach potentially misleading, and what alternative PromQL patterns would you recommend for anomaly detection on gauges?

8. How would you write a PromQL query to detect a "thundering herd" pattern where many services simultaneously spike in error rate within a 2-minute window?

9. Your `topk()` query in an alert rule is causing alert flapping because different time series enter and exit the top-k set. How do you stabilize this alert without losing signal fidelity?

10. Explain the difference between `sum by()` and `sum without()` in the context of high-cardinality label sets. When does the choice between them significantly impact query performance?

---

## Exporters & Instrumentation

11. A custom application exporter is exposing 50,000+ unique time series due to unbounded label cardinality from user IDs. How do you identify this problem, quantify its impact on Prometheus memory, and remediate it without losing critical observability?

12. The `node_exporter` on a production host is reporting stale metrics for filesystem utilization. The host is healthy and the exporter process is running. Walk through your systematic debugging approach.

13. You're instrumenting a Go microservice with the Prometheus client library. The service handles 10,000 RPS. How do you design your histogram buckets for latency tracking to balance accuracy at the 99th percentile with cardinality overhead?

14. A `blackbox_exporter` probe is intermittently failing for an HTTPS endpoint that appears healthy from application logs. What are the possible causes, and how do you differentiate between TLS, DNS, network, and application-layer issues?

15. Your `kube-state-metrics` deployment is consuming 4GB of memory in a cluster with 500 nodes and 5,000 pods. What strategies do you use to reduce its memory footprint while maintaining the metrics needed for SLO tracking?

16. Explain how the `pushgateway` anti-pattern manifests in production. Describe a real scenario where using Pushgateway caused metric staleness issues and how you resolved it.

17. A third-party JMX exporter for a Kafka cluster is producing inconsistent metric names across different Kafka versions. How do you handle this in a multi-version Kafka environment while maintaining consistent alerting?

18. You need to monitor a legacy application that cannot be instrumented directly. Compare the approaches of using a sidecar exporter, a proxy-based exporter, and a log-based exporter, including their trade-offs in a Kubernetes environment.

19. The `mysqld_exporter` is causing noticeable performance degradation on a production MySQL instance during peak hours. How do you diagnose the impact and tune the exporter's collection intervals and enabled collectors?

20. Describe how you would implement custom business metrics (e.g., order processing rate, payment failures) in a microservices architecture, ensuring the metrics are consistent, meaningful, and don't introduce cardinality explosions.

---

## Federation & Remote Write

21. You're implementing a hierarchical federation setup where regional Prometheus instances feed a global Prometheus. The global instance is missing metrics from one region intermittently. Walk through your debugging process.

22. A remote_write endpoint is experiencing backpressure, causing the Prometheus WAL to grow unboundedly. What metrics do you monitor to detect this condition, and what are your remediation options without data loss?

23. You need to federate metrics from 20 Prometheus instances into a central Thanos or Cortex cluster. Compare federation versus remote_write for this use case, including failure modes and operational complexity.

24. Your remote_write configuration is sending data to a Thanos Receive component, but you're seeing significant write amplification. How do you tune `queue_config` parameters (`max_samples_per_send`, `capacity`, `max_shards`) for optimal throughput?

25. Explain the WAL (Write-Ahead Log) replay mechanism in Prometheus. In a scenario where Prometheus crashes and restarts, how does WAL replay interact with remote_write, and what data loss scenarios are possible?

26. You're using cross-service federation to pull specific metrics from application Prometheus instances into a platform-level Prometheus. The `federate` endpoint is timing out. What are the likely causes and how do you resolve them?

27. Describe the challenges of implementing remote_write to multiple backends simultaneously (e.g., Thanos and a commercial observability platform). How do you handle partial failures and ensure consistency?

28. A remote_write pipeline is dropping samples due to `remote_write: out of order samples`. Explain why this happens in a Prometheus HA setup and how you configure deduplication at the receiving end.

29. How do you implement metric filtering in remote_write to reduce egress costs when writing to a cloud-based metrics backend, while ensuring critical SLO metrics are never dropped?

30. You're migrating from federation to remote_write for a large-scale deployment. What are the key operational differences in terms of data freshness, query latency, and failure isolation?

---

## High Availability & Clustering

31. You're running Prometheus in HA mode with two replicas behind a load balancer. An alert fires from one replica but not the other due to a scrape timing difference. How do you configure Alertmanager to handle this deduplication correctly?

32. Describe the "split-brain" scenario in a Prometheus HA setup. How does Alertmanager's gossip protocol help, and what are its limitations when network partitions occur between Alertmanager instances?

33. Your Prometheus HA pair is consuming 128GB of RAM combined for a 15-day retention period. Walk through your capacity planning methodology and the options for reducing memory footprint without reducing retention.

34. In a Kubernetes environment, a Prometheus pod is being OOMKilled repeatedly. The metrics show memory usage growing linearly over 6 hours before the kill. What are the likely causes and how do you systematically diagnose and fix this?

35. How do you implement a Prometheus HA setup that survives an entire availability zone failure in AWS/GCP/Azure, ensuring no alert gaps and no duplicate alert storms?

36. Explain the trade-offs between running Prometheus with long-term storage via Thanos versus Cortex versus VictoriaMetrics in a multi-tenant, multi-region enterprise environment.

37. You need to implement zero-downtime Prometheus upgrades in a production environment. Describe your strategy, including handling WAL compatibility, configuration changes, and alerting continuity.

38. A Prometheus instance in your HA pair has fallen behind by 30 minutes due to a network issue. When it recovers, how does it handle the backlog, and what is the impact on alerting and dashboards during the catch-up period?

39. Describe how you would architect a Prometheus deployment for a 10,000-node Kubernetes cluster with 1 million active time series, ensuring sub-second query response times for dashboards.

40. How do you handle Prometheus configuration management at scale across 50+ Prometheus instances with different scrape configurations, alerting rules, and retention policies?

---

## Cardinality Issues & Performance

41. A Prometheus instance's memory usage has doubled in 24 hours. Using `prometheus_tsdb_head_series` and related metrics, walk through your investigation to identify the cardinality explosion source.

42. A developer has added a `request_id` label to an HTTP request counter, causing 10 million new time series. The Prometheus instance is now unresponsive. What are your immediate remediation steps and long-term prevention strategies?

43. Explain how Prometheus TSDB chunks work and how chunk encoding affects both storage efficiency and query performance. When would you tune `--storage.tsdb.min-block-duration`?

44. You're seeing high `prometheus_engine_query_duration_seconds` for certain dashboard queries. How do you identify which queries are expensive, and what PromQL or recording rule optimizations would you apply?

45. Describe the impact of high-cardinality labels on Prometheus's inverted index. How does this affect both ingestion performance and query performance differently?

46. Your Prometheus instance is taking 45 seconds to compact a TSDB block. This is causing scrape delays and metric gaps. What are the underlying causes and how do you tune compaction behavior?

47. How do you implement a cardinality governance process in a large organization where multiple teams are adding metrics? Include tooling, policies, and enforcement mechanisms.

48. A service mesh (Istio) is generating millions of time series from its telemetry. How do you selectively reduce Istio metric cardinality while preserving the metrics needed for service-level observability?

49. Explain the difference between active series, head series, and total series in Prometheus TSDB. How does each metric help you understand memory consumption and predict OOM events?

50. You're using `metric_relabel_configs` to drop high-cardinality metrics at scrape time. What are the risks of this approach, and how do you ensure you're not accidentally dropping metrics that are used in active alert rules?

---

## Alertmanager & Alerting Strategy

51. An alert for high error rate is firing and resolving every 5 minutes (flapping). Walk through the Alertmanager configuration changes and PromQL adjustments you would make to eliminate flapping without increasing detection latency.

52. You have a complex routing tree in Alertmanager with 15 routes. An alert is being sent to the wrong receiver. How do you systematically debug Alertmanager routing without triggering real alerts?

53. Describe how you would implement a multi-tier alerting strategy for a payment processing system with different severity levels, escalation paths, and business-hours-aware routing.

54. Your Alertmanager cluster has lost quorum due to a network partition. Alerts are being generated by Prometheus but not delivered. What is your incident response procedure, and how do you prevent this in the future?

55. How do you implement alert correlation in Alertmanager to suppress downstream alerts when a root cause alert (e.g., a database being down) is already firing?

56. A critical alert has a `for: 5m` clause, but the underlying issue resolves in 3 minutes before the alert fires. How do you balance detection sensitivity with alert noise in this scenario?

57. Explain how Alertmanager's `inhibit_rules` work and describe a production scenario where misconfigured inhibition rules caused a critical alert to be silenced during an actual outage.

58. You need to implement alert routing that sends different notifications based on the Kubernetes namespace, team ownership labels, and time of day. Describe your Alertmanager configuration approach.

59. How do you handle alert deduplication when the same underlying issue is detected by multiple monitoring systems (Prometheus, Datadog, and a synthetic monitor) simultaneously?

60. Describe your strategy for managing Alertmanager silences programmatically during planned maintenance windows across a fleet of 200 microservices.

---

## Recording Rules & Rule Management

61. A recording rule that pre-computes a complex aggregation is producing results that differ from the equivalent instant query by 15%. What are the possible causes of this discrepancy?

62. You have 500 recording rules across multiple rule files. The rule evaluation is taking longer than the 15-second evaluation interval. How do you diagnose which rules are slow and optimize the rule evaluation pipeline?

63. Explain the dependency chain problem in recording rules. How do you detect and resolve circular dependencies, and what happens to dependent alerts when a recording rule evaluation fails?

64. How do you implement a recording rule strategy that balances query performance improvement with the operational overhead of maintaining rules across multiple Prometheus instances?

65. A recording rule was modified to change its aggregation labels, causing a metric name collision with the old rule output during a rolling update. How do you handle this migration safely?

---

## Kubernetes Monitoring

66. Your Kubernetes cluster has 1,000 pods, and `kube-state-metrics` is showing pod phase metrics with a 5-minute lag. How do you diagnose whether this is a scrape interval issue, a kube-state-metrics performance issue, or an API server issue?

67. Prometheus is failing to scrape metrics from pods in a specific namespace due to RBAC issues. Walk through the exact RBAC resources you need to verify and the commands you would use to diagnose this.

68. You're implementing monitoring for a stateful application (e.g., Cassandra) running on Kubernetes. How do you handle pod identity in metrics when pods are rescheduled and get new IP addresses?

69. Describe how you would implement Prometheus Operator in a multi-tenant Kubernetes cluster where different teams need isolated Prometheus instances with different retention and scrape configurations.

70. A Kubernetes HPA is scaling a deployment based on custom metrics from Prometheus via the custom metrics API adapter. The scaling is erratic. How do you debug the metric pipeline from Prometheus through the adapter to the HPA?

71. How do you monitor Kubernetes control plane components (etcd, API server, scheduler, controller manager) with Prometheus in a managed Kubernetes service (EKS/GKE/AKS) where you have limited access to control plane nodes?

72. Explain how you would implement monitoring for Kubernetes network policies and CNI plugin performance using Prometheus, including the specific metrics and exporters involved.

73. Your Prometheus instance is being evicted from Kubernetes nodes due to resource pressure. How do you configure PodDisruptionBudgets, resource requests/limits, and node affinity to ensure Prometheus stability?

74. Describe the challenges of monitoring ephemeral Kubernetes jobs and CronJobs with Prometheus, and how you would implement reliable metrics collection for batch workloads.

75. How do you implement cross-cluster monitoring in a multi-cluster Kubernetes environment using Prometheus, ensuring consistent metric naming, label conventions, and alert routing across clusters?

---

## Scaling Challenges & Production Scenarios

76. Your Prometheus instance is ingesting 2 million samples per second and query latency has degraded to 30 seconds for range queries. What architectural changes do you make, and how do you migrate without data loss?

77. During a Black Friday traffic surge, your Prometheus instance ran out of disk space due to unexpected metric volume. Describe your post-incident architecture changes to prevent recurrence.

78. You need to implement Prometheus monitoring for a serverless architecture (AWS Lambda/Google Cloud Functions) where traditional scraping is not possible. What approaches do you use?

79. A microservices team is deploying 50 new services per week. How do you implement auto-discovery and automatic monitoring onboarding without manual Prometheus configuration changes?

80. Describe how you would implement a cost allocation model for Prometheus infrastructure costs in a multi-team environment, attributing storage and compute costs to the teams generating the most metrics.

---

## Basic Questions

1. What is Prometheus and what problem does it solve in a monitoring context?
2. Explain the difference between a counter, gauge, histogram, and summary metric type in Prometheus.
3. What is a scrape interval and how does Prometheus collect metrics from targets?
4. What is the default port Prometheus listens on, and where is its configuration file located?
5. What is a Prometheus exporter? Give three examples of commonly used exporters.
6. How do you define a static scrape target in `prometheus.yml`?
7. What is PromQL and what is it used for?
8. Explain what a label is in Prometheus and why labels are important.
9. What is the purpose of the Alertmanager in the Prometheus ecosystem?
10. How do you check if a Prometheus target is being scraped successfully?
11. What does the `up` metric in Prometheus represent?
12. What is the difference between `rate()` and `irate()` in PromQL?
13. How do you write a basic PromQL query to get the total HTTP request count for a service?
14. What is a recording rule in Prometheus and why would you use one?
15. What is the Pushgateway and when should it be used?
16. How do you define an alert rule in Prometheus?
17. What is the `for` clause in a Prometheus alert rule?
18. What does `prometheus_tsdb_head_series` metric tell you?
19. How do you reload Prometheus configuration without restarting the process?
20. What is service discovery in Prometheus and name two supported service discovery mechanisms?
21. What is the purpose of `relabel_configs` in a scrape configuration?
22. How does Prometheus store data locally, and what is the default retention period?
23. What is the difference between `sum()` and `count()` aggregation operators in PromQL?
24. How do you filter metrics by label value in a PromQL query?
25. What is a Prometheus federation and when would you use it?
26. What is the purpose of the `--web.enable-lifecycle` flag in Prometheus?
27. How do you expose custom application metrics using the Prometheus client library?
28. What is the `node_exporter` and what types of metrics does it expose?
29. How do you configure Alertmanager to send notifications to Slack?
30. What is the difference between `increase()` and `rate()` in PromQL?
31. What does the `absent()` function do in PromQL?
32. How do you query the top 5 services by request rate in PromQL?
33. What is a Prometheus data model and how are time series identified?
34. What is the `kube-state-metrics` exporter used for in Kubernetes monitoring?
35. How do you configure Prometheus to scrape metrics from Kubernetes pods using annotations?
36. What is the difference between `metric_relabel_configs` and `relabel_configs`?
37. How do you check Prometheus's own health and readiness endpoints?
38. What is the purpose of the `honor_labels` option in a scrape configuration?
39. How do you configure a basic Alertmanager route to send alerts to an email receiver?
40. What is the `blackbox_exporter` used for?
41. How do you use the Prometheus HTTP API to query metrics programmatically?
42. What is the `scrape_timeout` setting and what happens when it is exceeded?
43. How do you configure Prometheus to use basic authentication for scraping a target?
44. What is the purpose of the `external_labels` configuration in Prometheus?
45. How do you write a PromQL query to calculate the error rate as a percentage of total requests?

---

## Intermediate Questions

1. How do you implement Prometheus service discovery for AWS EC2 instances using the `ec2_sd_config`?
2. Describe how you would configure Prometheus to scrape metrics from all pods in a Kubernetes namespace that have a specific annotation.
3. How do you implement a multi-tier alerting strategy using Alertmanager's routing tree with different receivers for different severity levels?
4. Explain how Prometheus's WAL (Write-Ahead Log) works and how it affects recovery after a crash.
5. How do you configure Prometheus remote_write to send metrics to a Thanos Receive component?
6. Describe how you would implement Prometheus HA with two replicas and configure Alertmanager deduplication.
7. How do you use `label_replace()` to normalize inconsistent label values from different exporters?
8. Explain how `histogram_quantile()` works and what the accuracy implications of bucket boundaries are.
9. How do you implement recording rules to pre-compute expensive aggregations for dashboard performance?
10. Describe how you would configure Prometheus to monitor a Kubernetes cluster's control plane components.
11. How do you implement Alertmanager inhibition rules to suppress downstream alerts when a root cause alert is firing?
12. How do you tune Prometheus's `--storage.tsdb.retention.size` versus `--storage.tsdb.retention.time`?
13. Explain how Prometheus handles counter resets and how `rate()` accounts for them.
14. How do you implement a Prometheus scrape configuration that uses TLS client certificates for authentication?
15. Describe how you would use `predict_linear()` to alert on disk space exhaustion before it occurs.
16. How do you configure Alertmanager grouping to reduce alert noise during a widespread infrastructure failure?
17. How do you implement Prometheus monitoring for a stateful application running on Kubernetes with persistent volumes?
18. Explain how `subquery` syntax works in PromQL and when you would use it.
19. How do you implement Prometheus federation to pull specific metrics from application-level Prometheus instances into a platform-level instance?
20. Describe how you would configure `metric_relabel_configs` to drop high-cardinality metrics before they are stored.
21. How do you implement Prometheus Operator's `ServiceMonitor` and `PodMonitor` resources for automatic scrape configuration?
22. How do you configure Alertmanager's `repeat_interval` and `group_interval` to balance notification frequency with alert noise?
23. Explain how Prometheus's `staleness` handling works and how it affects dashboards when a target goes down.
24. How do you implement a Prometheus scrape configuration that dynamically discovers targets from a Consul service registry?
25. Describe how you would implement custom Prometheus metrics for tracking business KPIs in a microservices application.
26. How do you use the Prometheus API to query the current state of all active alerts?
27. How do you implement Prometheus monitoring for a Kubernetes Ingress controller, including request rate, error rate, and latency metrics?
28. Explain how `offset` modifier works in PromQL and provide a use case for comparing current metrics with historical baselines.
29. How do you configure Prometheus to scrape metrics from a service mesh (Istio) sidecar proxies?
30. Describe how you would implement a Prometheus-based SLO monitoring setup using recording rules and multi-window burn rate alerts.
31. How do you configure Prometheus's `--query.max-concurrency` and `--query.timeout` flags for a heavily loaded instance?
32. How do you implement Prometheus monitoring for Kubernetes HPA (Horizontal Pod Autoscaler) using custom metrics?
33. Explain how Prometheus handles high-availability for Alertmanager using the gossip protocol.
34. How do you implement a Prometheus scrape configuration that handles targets with self-signed TLS certificates?
35. Describe how you would use `group_left` and `group_right` in PromQL for many-to-one metric joins.
36. How do you configure Prometheus to automatically discover and monitor new Kubernetes namespaces without manual configuration changes?
37. How do you implement Prometheus alerting for Kubernetes node resource pressure (CPU, memory, disk)?
38. Explain how the `@` modifier works in PromQL and when it is useful for incident analysis.
39. How do you implement Prometheus monitoring for a multi-cluster Kubernetes environment using a central Prometheus instance?
40. Describe how you would tune Prometheus's chunk encoding and block compaction settings for a write-heavy workload.

---

## Advanced Questions

1. Design a Prometheus architecture for a 50,000-node infrastructure generating 500 million active time series, including sharding, federation, and long-term storage strategies.
2. How do you implement a Prometheus-based multi-tenant monitoring platform where each tenant has isolated metrics, alerting, and dashboards, with strict data separation?
3. Describe how you would implement a global Prometheus deployment using Thanos with cross-cluster querying, deduplication, and 2-year metric retention at petabyte scale.
4. How do you implement a Prometheus cardinality governance system that automatically detects, alerts on, and enforces cardinality budgets per team and per service?
5. Describe the complete failure modes of a Prometheus HA setup during a network partition, including the impact on alerting, data collection, and query availability.
6. How do you implement a Prometheus-based anomaly detection system that uses statistical methods to detect metric anomalies without relying on static thresholds?
7. Describe how you would implement a zero-trust security model for a Prometheus deployment, including mTLS for all scrape connections, RBAC for metric access, and audit logging.
8. How do you implement a Prometheus deployment that can survive a complete cloud region failure with no data loss and sub-5-minute recovery time?
9. Describe how you would implement a cost-optimized Prometheus architecture that reduces storage costs by 80% while maintaining 2-year metric retention and sub-second query performance.
10. How do you implement Prometheus monitoring for a service mesh at scale (10,000 pods) where Istio telemetry is generating 50 million time series?
11. Describe how you would implement a Prometheus-based capacity planning system that predicts resource exhaustion 30 days in advance across a 1,000-node Kubernetes cluster.
12. How do you implement a Prometheus deployment that handles 10 million samples per second ingestion with consistent sub-100ms query latency for real-time dashboards?
13. Describe how you would implement a Prometheus-based SLO platform that automatically calculates error budgets, burn rates, and SLO compliance for 500 services.
14. How do you implement a Prometheus federation architecture that provides both global aggregated views and local high-resolution data without duplicating storage?
15. Describe how you would implement a Prometheus deployment for a regulated financial services environment, including data encryption, access controls, and audit trails for compliance.
16. How do you implement a Prometheus-based alerting system that uses machine learning to dynamically adjust alert thresholds based on historical patterns and seasonal variations?
17. Describe how you would implement a Prometheus deployment that automatically scales horizontally based on ingestion rate, with automatic shard rebalancing and zero data loss.
18. How do you implement a Prometheus-based observability platform for a multi-cloud environment (AWS, GCP, Azure) with unified metric naming, consistent labeling, and cross-cloud correlation?
19. Describe how you would implement a Prometheus deployment that provides real-time metric streaming to downstream consumers (Kafka, Kinesis) while maintaining local storage for querying.
20. How do you implement a Prometheus-based incident correlation system that automatically identifies related metric anomalies across different services and infrastructure layers during an incident?
21. Describe how you would implement a Prometheus deployment that handles metric backfill for historical data import from legacy monitoring systems without disrupting ongoing ingestion.
22. How do you implement a Prometheus-based change detection system that automatically correlates metric changes with deployment events, configuration changes, and infrastructure modifications?
23. Describe how you would implement a Prometheus deployment for a Kubernetes multi-cluster environment with 100 clusters, providing both per-cluster and cross-cluster observability.
24. How do you implement a Prometheus-based reliability engineering platform that automatically calculates MTTR, MTTF, and availability metrics for all services from incident data?
25. Describe how you would implement a Prometheus deployment that provides sub-second metric freshness for real-time trading applications while maintaining 5-year historical retention.

---

## Rapid-Fire Questions

1. What HTTP method does Prometheus use to scrape metrics from targets?
2. What is the default scrape interval in Prometheus?
3. Which metric type should you use to track the number of HTTP requests processed?
4. What does `rate(http_requests_total[5m])` calculate?
5. What port does `node_exporter` listen on by default?
6. What is the purpose of the `__address__` label in Prometheus service discovery?
7. How do you silence an alert in Alertmanager?
8. What is the difference between `sum()` and `avg()` aggregation operators?
9. What does a Prometheus target in "UNKNOWN" state indicate?
10. What is the `stale marker` in Prometheus TSDB?
11. How do you check the cardinality of a specific metric in Prometheus?
12. What is the purpose of `honor_timestamps` in a scrape configuration?
13. What does `topk(5, metric)` return?
14. What is the Prometheus `/-/healthy` endpoint used for?
15. What is the difference between `recording rules` and `alerting rules`?
16. What does `increase(counter[1h])` calculate?
17. What is the purpose of the `__metrics_path__` label?
18. How do you query all time series with a specific label value in PromQL?
19. What is the default evaluation interval for Prometheus rules?
20. What does `vector(0)` return in PromQL?
21. What is the purpose of `keep_firing_for` in an alert rule?
22. How do you check which version of Prometheus is running?
23. What is the `ALERTS` metric in Prometheus?
24. What does `min_over_time(metric[1h])` calculate?
25. What is the purpose of the `cluster` label in a federated Prometheus setup?
26. What does `absent(up{job="myapp"})` return when the target is down?
27. How do you list all active targets in Prometheus via the API?
28. What is the difference between `label_replace()` and `label_join()`?
29. What does `resets(counter[1h])` calculate?
30. What is the purpose of `--storage.tsdb.allow-overlapping-blocks`?
31. What does `quantile_over_time(0.99, metric[1h])` calculate?
32. How do you check the WAL replay status after a Prometheus restart?
33. What is the `prometheus_rule_evaluation_failures_total` metric used for?
34. What does `clamp_min(metric, 0)` do in PromQL?
35. What is the purpose of the `send_resolved` option in Alertmanager receivers?
36. How do you query the number of active time series in Prometheus?
37. What does `delta(gauge[1h])` calculate?
38. What is the purpose of `--rules.alert.for-outage-tolerance` flag?
39. What does `sort_desc(topk(10, metric))` return?
40. What is the `prometheus_tsdb_compactions_total` metric used for?
41. What does `bool` modifier do in a PromQL comparison?
42. What is the purpose of `group_wait` in Alertmanager routing?
43. How do you check if a recording rule is being evaluated correctly?
44. What does `time()` return in PromQL?
45. What is the purpose of `--web.console.templates` in Prometheus?
46. What does `changes(gauge[1h])` calculate?
47. What is the `prometheus_notifications_dropped_total` metric used for?
48. How do you force a Prometheus TSDB compaction?
49. What does `scalar(single_element_vector)` do in PromQL?
50. What is the purpose of the `match[]` parameter in the Prometheus federation endpoint?
