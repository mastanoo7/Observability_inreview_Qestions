# Splunk — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## Indexers & Indexing Architecture

1. Your Splunk indexer cluster is experiencing indexing latency spikes every 4 hours. The spikes correlate with bucket rolling events. How do you diagnose whether this is a disk I/O issue, a replication factor issue, or a search peer load issue?

2. A Splunk indexer is showing a high number of events in the "thawed" state. What does this indicate about your bucket lifecycle, and what are the performance implications for search operations?

3. Describe the internal architecture of a Splunk indexer bucket. How do the rawdata, tsidx, and metadata files interact during indexing and search, and how does understanding this help you troubleshoot performance issues?

4. Your Splunk indexing rate has dropped from 500GB/day to 50GB/day without any change in log volume. Walk through your systematic investigation of the indexing pipeline.

5. How do you implement index-time field extraction versus search-time field extraction in Splunk, and what are the performance trade-offs for each approach in a high-volume production environment?

6. A Splunk indexer is running out of disk space faster than expected. The license usage appears normal. How do you identify which indexes are consuming unexpected space and implement a remediation strategy?

7. Explain how Splunk's bloom filters work in the tsidx files and how they affect search performance. In what scenarios do bloom filters fail to improve search performance?

8. You need to implement a hot-warm-cold-frozen bucket lifecycle for a Splunk deployment that must retain 7 years of compliance data while keeping query performance acceptable for recent data. Describe your architecture.

9. How do you handle index corruption in a Splunk indexer cluster? Describe the steps to identify corrupted buckets, remove them safely, and restore data from replicated copies.

10. Describe how you would implement Splunk SmartStore to offload cold and frozen buckets to S3, including the configuration, performance implications, and failure scenarios.

---

## Forwarders & Data Collection

11. A Universal Forwarder on a critical production server is dropping events during peak load. The forwarder's queue is full. How do you diagnose the root cause and implement both immediate and long-term fixes?

12. Explain the difference between Universal Forwarder, Heavy Forwarder, and Intermediate Forwarder. Describe a production architecture where you would use all three, and the specific role each plays.

13. Your Universal Forwarder is sending duplicate events to the indexer. How do you identify the source of duplication — whether it's a forwarder configuration issue, a load balancer issue, or an indexer-side issue?

14. A Heavy Forwarder is consuming 100% CPU during peak hours. It's performing regex-based routing and field extraction. How do you profile the forwarder's processing pipeline and optimize it?

15. Describe how you would implement forwarder management at scale for 5,000 Universal Forwarders across a hybrid cloud environment, including configuration management, version upgrades, and health monitoring.

16. How do you implement SSL/TLS encryption and certificate-based authentication between Universal Forwarders and Indexers in a large enterprise environment, including certificate rotation without data loss?

17. A forwarder is failing to connect to the indexer cluster after a network change. The forwarder logs show "connection refused" errors. Walk through your debugging process, including the specific configuration files and log locations you would check.

18. Explain how Splunk's Indexer Discovery feature works and how it differs from manual indexer configuration in forwarders. What are the failure modes of Indexer Discovery?

19. You need to collect logs from a Kubernetes cluster using Splunk. Compare the approaches of using a DaemonSet-based Universal Forwarder versus the Splunk Connect for Kubernetes (SC4K) solution, including their trade-offs.

20. How do you implement forwarder load balancing across multiple indexers, and how does Splunk handle indexer failures from the forwarder's perspective?

---

## Search Heads & Search Management

21. Your Splunk Search Head Cluster (SHC) is experiencing search job failures with "search peer not responding" errors. How do you diagnose whether this is a network issue, an indexer performance issue, or a search head captain election problem?

22. Describe the Search Head Cluster captain election process. What happens to in-flight searches during a captain re-election, and how do you minimize the impact on users?

23. A search head is running out of memory due to concurrent searches. How do you implement search concurrency limits, search scheduling priorities, and resource pools to manage search load?

24. Explain how Splunk's search artifact replication works in a Search Head Cluster. What are the storage implications, and how do you tune artifact replication for a cluster with 50 concurrent users?

25. How do you implement search head pooling for different user groups (operations, security, compliance) to ensure that heavy compliance searches don't impact real-time operational dashboards?

26. A scheduled search that runs every 5 minutes is consistently taking 8 minutes to complete. How do you diagnose the search performance issue and optimize it without changing the search frequency?

27. Describe how you would implement Splunk's workload management feature to prioritize real-time searches over historical searches during incident response.

28. How do you handle search head failover in a non-clustered Splunk deployment? What data is lost during failover, and how do you minimize the impact?

---

## Clustering & High Availability

29. Your Splunk indexer cluster has lost replication factor for 30% of its buckets after an indexer failure. How do you assess the data loss risk, restore replication factor, and prevent this scenario in the future?

30. Explain the difference between replication factor and search factor in Splunk clustering. Describe a scenario where having a search factor lower than the replication factor caused a production issue.

31. A Splunk cluster master (now called Cluster Manager) is unresponsive. What is the impact on indexing and searching, and what are your options for recovery?

32. How do you implement a Splunk multi-site cluster for disaster recovery across two data centers, including the configuration of site replication factor and site search factor?

33. Describe the process of adding a new indexer peer to an existing Splunk cluster without disrupting ongoing indexing or searching operations.

34. How do you perform a rolling upgrade of a Splunk indexer cluster from one version to another, ensuring zero data loss and minimal search disruption?

35. Explain how Splunk's excess bucket removal works and how it can cause data loss if misconfigured. Describe a production scenario where this caused an incident.

---

## SPL Queries & Search Optimization

36. A SPL query using `stats count by host, sourcetype` across 30 days of data is timing out. How do you optimize this query using time-based filtering, index selection, and summary indexing?

37. Explain the difference between `tstats` and `stats` in Splunk. In what scenarios does `tstats` provide a 10x or greater performance improvement, and what are its limitations?

38. You need to write a SPL query to detect brute force login attempts across a distributed application where login events are spread across multiple sourcetypes and indexes. Write the query and explain your approach.

39. How do you use Splunk's `join` command effectively, and when should you avoid it in favor of `lookup`, `append`, or subsearches for better performance?

40. Describe how you would implement a SPL query to calculate the 95th percentile of API response times across 100 microservices, grouped by service and time bucket, for a 30-day trend analysis.

41. A SPL query using `transaction` command is consuming excessive memory. How do you rewrite it using `stats` or `streamstats` to achieve the same result with better performance?

42. How do you implement SPL macros and workflow actions to create reusable query components that can be shared across teams while maintaining version control?

43. Explain how Splunk's `eval` command handles null values and how null propagation can cause incorrect results in complex SPL queries. Provide a production example.

44. You need to implement a SPL query that detects anomalies in metric data using statistical methods (e.g., standard deviation, moving average). How do you implement this without using Splunk's ML Toolkit?

45. Describe how you would use Splunk's `map` command and when it's appropriate versus using subsearches or lookups for correlating data across multiple searches.

---

## Ingestion Bottlenecks & Parsing Issues

46. Your Splunk deployment is experiencing a 2-hour indexing lag during peak hours. The license usage is within limits. How do you identify the bottleneck in the ingestion pipeline — from forwarder to indexer?

47. A new log source is being indexed with incorrect timestamps, causing events to appear in the wrong time buckets. How do you diagnose the timestamp parsing issue and fix it without re-indexing existing data?

48. Describe how Splunk's line breaking and event merging works. You're seeing multi-line Java stack traces being split into individual events. How do you configure `SHOULD_LINEMERGE` and related settings to fix this?

49. A sourcetype's events are being indexed with `_raw` containing unparsed data because the transforms are not being applied. Walk through the Splunk parsing pipeline (input, parsing, indexing, search) to identify where the failure is occurring.

50. How do you implement structured logging in Splunk to improve search performance and field extraction reliability, and what changes do you make to `props.conf` and `transforms.conf`?

51. Your Splunk deployment is receiving logs with inconsistent timestamp formats from different application versions. How do you implement a robust timestamp normalization strategy?

52. Describe how you would diagnose and fix a situation where Splunk is incorrectly identifying the sourcetype for events, causing them to be parsed with the wrong configuration.

---

## Scaling & Performance

53. Your Splunk deployment needs to scale from 500GB/day to 5TB/day ingestion. Describe your scaling strategy, including indexer capacity planning, search head scaling, and network architecture changes.

54. How do you implement Splunk's distributed search to optimize query performance across a large indexer cluster, including the role of search factor and the impact of data distribution?

55. Describe how you would implement Splunk's summary indexing to pre-compute expensive searches and improve dashboard performance for executive-level reporting.

56. A Splunk deployment is experiencing high CPU usage on indexers during search operations. How do you balance indexing and searching workloads, and what configuration changes reduce search-induced indexing latency?

57. How do you implement Splunk's data model acceleration to improve the performance of pivot-based searches and dashboards, and what are the storage and CPU trade-offs?

58. Describe your approach to Splunk capacity planning for a 3-year growth projection, including storage tiering, compute scaling, and license management.

---

## SIEM Use Cases & Security

59. You're using Splunk as a SIEM and need to detect lateral movement in your network. Describe the SPL queries and correlation searches you would implement to identify this attack pattern.

60. How do you implement Splunk's Enterprise Security (ES) framework for a financial services company, including the configuration of notable events, risk scores, and automated response actions?

61. Describe how you would use Splunk to implement real-time detection of data exfiltration attempts, including the data sources, SPL queries, and alerting thresholds you would use.

62. How do you implement Splunk's User Behavior Analytics (UBA) integration with ES to detect insider threats, and what are the data requirements and model training considerations?

63. A Splunk ES correlation search is generating 1,000 notable events per hour, overwhelming the SOC team. How do you implement risk-based alerting to reduce noise while maintaining detection coverage?

64. Describe how you would implement Splunk's Adaptive Response framework to automatically contain a compromised host when a specific security alert fires.

---

## Production Outage Analysis

65. During a production outage, your Splunk search heads are unavailable because they're on the same network segment as the affected systems. How do you access log data for incident investigation?

66. You need to perform a post-incident analysis of a 4-hour outage that affected 10 microservices. Describe your SPL-based investigation strategy, including the queries you would use to establish a timeline of events.

67. How do you implement Splunk's real-time search capabilities for live incident monitoring, and what are the performance implications of running multiple real-time searches simultaneously?

68. Describe how you would use Splunk to implement automated incident detection that correlates application errors, infrastructure metrics, and user impact signals to generate a single, actionable alert.

69. A critical SPL-based alert failed to fire during an outage because the underlying data was delayed by 30 minutes. How do you implement monitoring for data ingestion lag and adjust alert logic to account for late-arriving data?

70. How do you implement Splunk's IT Service Intelligence (ITSI) for service-level monitoring, including the configuration of KPI base searches, service trees, and episode review?

---

## Splunk Cloud & Kubernetes Integration

71. You're migrating from Splunk Enterprise to Splunk Cloud. What are the key architectural differences, and how do you handle data sovereignty requirements for logs that must remain in specific geographic regions?

72. Describe how you would implement Splunk monitoring for a Kubernetes cluster, including the collection of pod logs, Kubernetes events, metrics, and control plane logs.

73. How do you implement Splunk's HTTP Event Collector (HEC) at scale for a microservices architecture where 500 services need to send logs directly to Splunk?

74. Describe the challenges of using Splunk in a multi-cloud environment and how you implement consistent log collection, parsing, and searching across AWS, GCP, and Azure.

75. How do you implement Splunk's Observability Cloud (formerly SignalFx) integration with Splunk Enterprise for a unified view of metrics, logs, and traces?

76. A Splunk HEC endpoint is being overwhelmed with requests during a traffic spike, causing event loss. How do you implement HEC load balancing, token management, and backpressure handling?

77. Describe how you would implement Splunk's Data Stream Processor (DSP) for real-time data routing, filtering, and enrichment before data reaches the indexers.

78. How do you implement compliance logging in Splunk for a healthcare organization, ensuring HIPAA-compliant data handling, access controls, and audit trails for all log access?

---

## Basic Questions

1. What is Splunk and what is its primary use case in enterprise IT?
2. What is the difference between a Splunk indexer, search head, and forwarder?
3. What is a Splunk index and how does it organize data?
4. What is SPL (Search Processing Language) and what is it used for?
5. What is a Universal Forwarder and how does it differ from a Heavy Forwarder?
6. How do you perform a basic keyword search in Splunk?
7. What is a Splunk sourcetype and why is it important?
8. What is the purpose of the `index` field in a Splunk search?
9. How do you use the `stats` command in SPL?
10. What is a Splunk saved search and when would you use it?
11. What is the purpose of the `host` field in Splunk events?
12. How do you use the `table` command in SPL to format search results?
13. What is a Splunk dashboard and how do you create one?
14. What is the purpose of the `eval` command in SPL?
15. How do you use the `rex` command to extract fields from log data?
16. What is a Splunk alert and how do you configure one?
17. What is the purpose of the `timechart` command in SPL?
18. How do you use the `dedup` command to remove duplicate events?
19. What is a Splunk lookup and when would you use it?
20. What is the purpose of the `where` command in SPL?
21. How do you configure a Splunk forwarder to send data to an indexer?
22. What is the purpose of the `props.conf` file in Splunk?
23. How do you use the `top` command to find the most common values?
24. What is a Splunk field extraction and how do you create one?
25. What is the purpose of the `transforms.conf` file in Splunk?
26. How do you use the `transaction` command in SPL?
27. What is a Splunk data model and when would you use it?
28. How do you configure Splunk to monitor a log file on a local system?
29. What is the purpose of the `inputs.conf` file in Splunk?
30. How do you use the `sort` command in SPL?
31. What is a Splunk knowledge object and give three examples?
32. How do you use the `rename` command in SPL?
33. What is the purpose of Splunk's HTTP Event Collector (HEC)?
34. How do you use the `head` and `tail` commands in SPL?
35. What is a Splunk index-time field extraction versus a search-time field extraction?
36. How do you configure Splunk's `outputs.conf` for a Universal Forwarder?
37. What is the purpose of the `_internal` index in Splunk?
38. How do you use the `makeresults` command in SPL for testing?
39. What is a Splunk macro and how do you create one?
40. How do you use the `iplocation` command in SPL?
41. What is the purpose of Splunk's `_audit` index?
42. How do you configure Splunk's time zone settings for correct timestamp parsing?
43. What is the `splunkd.log` file and what information does it contain?
44. How do you use the `fieldsummary` command to understand data quality?
45. What is the purpose of Splunk's `_metrics` index?

---

## Intermediate Questions

1. How do you implement Splunk's `tstats` command for accelerated searches over data models?
2. Describe how you would configure Splunk's index clustering with a replication factor of 3 and search factor of 2.
3. How do you implement Splunk's summary indexing to pre-compute expensive searches for dashboard performance?
4. Explain how Splunk's bucket lifecycle (hot, warm, cold, frozen) works and how you configure transitions.
5. How do you implement Splunk's Search Head Clustering (SHC) for high availability?
6. Describe how you would configure Splunk's forwarder load balancing across multiple indexers.
7. How do you implement Splunk's role-based access control to restrict data access by index and sourcetype?
8. Explain how Splunk's `join` command works and when you should use `lookup` instead for better performance.
9. How do you implement Splunk's data model acceleration to improve pivot search performance?
10. Describe how you would configure Splunk's SmartStore to offload cold buckets to S3.
11. How do you implement Splunk's `streamstats` command for running calculations over event streams?
12. Explain how Splunk's `eventstats` command differs from `stats` and when each is appropriate.
13. How do you implement Splunk's alert suppression to prevent duplicate notifications for the same issue?
14. Describe how you would configure Splunk's multi-site clustering for disaster recovery.
15. How do you implement Splunk's `predict` command for time series forecasting?
16. Explain how Splunk's `cluster` command works for grouping similar events.
17. How do you implement Splunk's workflow actions to create interactive links from search results?
18. Describe how you would configure Splunk's Indexer Discovery for automatic forwarder-to-indexer routing.
19. How do you implement Splunk's `anomalydetection` command for statistical anomaly detection?
20. Explain how Splunk's `map` command works and when it is appropriate to use.
21. How do you implement Splunk's data integrity control to detect and prevent data tampering?
22. Describe how you would configure Splunk's search scheduler to manage concurrent search load.
23. How do you implement Splunk's `geostats` command for geographic data visualization?
24. Explain how Splunk's `accum` command works for running totals.
25. How do you implement Splunk's field aliases to normalize inconsistent field names across sourcetypes?
26. Describe how you would configure Splunk's Heavy Forwarder for data routing and filtering.
27. How do you implement Splunk's `bucket` command for time-based aggregations?
28. Explain how Splunk's `appendcols` command works for combining search results.
29. How do you implement Splunk's `inputlookup` and `outputlookup` for lookup table management?
30. Describe how you would configure Splunk's Deployment Server for managing forwarder configurations at scale.
31. How do you implement Splunk's `sendalert` command for programmatic alert triggering?
32. Explain how Splunk's `sistats` command differs from `stats` for distributed searches.
33. How do you implement Splunk's `typelearner` command for automatic field type detection?
34. Describe how you would configure Splunk's index-time anonymization for PII data masking.
35. How do you implement Splunk's `contingency` command for cross-tabulation analysis?
36. Explain how Splunk's `multikv` command works for parsing multi-value key-value data.
37. How do you implement Splunk's `crawl` command for file system discovery?
38. Describe how you would configure Splunk's alert actions for automated remediation.
39. How do you implement Splunk's `xyseries` command for pivot table creation?
40. Explain how Splunk's `trendline` command works for moving average calculations.

---

## Advanced Questions

1. Design a Splunk architecture for a global enterprise ingesting 50TB/day across 5 data centers, with multi-site clustering, SmartStore, and compliance data retention for 7 years.
2. How do you implement a Splunk-based SIEM platform for a financial services company, including real-time threat detection, automated response, and compliance reporting for SOX and PCI-DSS?
3. Describe how you would implement Splunk's Machine Learning Toolkit (MLTK) for predictive anomaly detection in infrastructure metrics, including model training, validation, and production deployment.
4. How do you implement a Splunk deployment that handles 1 million events per second ingestion with sub-second search latency for real-time security monitoring?
5. Describe how you would implement Splunk's IT Service Intelligence (ITSI) for a complex service dependency tree with 500 services, including KPI configuration, service health scoring, and episode management.
6. How do you implement a Splunk-based observability platform that correlates application logs, infrastructure metrics, and distributed traces for end-to-end incident investigation?
7. Describe how you would implement Splunk's federated search across multiple Splunk deployments in different security zones, ensuring data sovereignty and access controls.
8. How do you implement a Splunk deployment that provides real-time fraud detection for a payment processing system, including the SPL queries, alert logic, and automated response actions?
9. Describe how you would implement Splunk's data fabric search for querying data across Splunk, S3, and external databases without data movement.
10. How do you implement a Splunk-based compliance monitoring platform for GDPR, including data subject request handling, consent tracking, and breach detection?
11. Describe how you would implement Splunk's User Behavior Analytics (UBA) for insider threat detection, including data requirements, model configuration, and integration with ES notable events.
12. How do you implement a Splunk deployment that automatically scales indexer capacity based on ingestion volume, using cloud auto-scaling and Splunk's dynamic data self-storage?
13. Describe how you would implement Splunk's Adaptive Response framework for automated incident containment, including playbook design and integration with SOAR platforms.
14. How do you implement a Splunk-based network security monitoring platform that detects lateral movement, data exfiltration, and command-and-control communications in real-time?
15. Describe how you would implement Splunk's mission control for unified security operations, integrating ES, UBA, and SOAR into a single incident management workflow.
16. How do you implement a Splunk deployment for a healthcare organization with HIPAA compliance requirements, including PHI data masking, access auditing, and breach notification workflows?
17. Describe how you would implement Splunk's Observability Cloud integration with Splunk Enterprise for a unified view of infrastructure metrics, application logs, and distributed traces.
18. How do you implement a Splunk-based DevSecOps platform that integrates with CI/CD pipelines to detect security vulnerabilities in code deployments and infrastructure changes?
19. Describe how you would implement Splunk's data stream processor for real-time data enrichment, routing, and transformation before data reaches the indexers.
20. How do you implement a Splunk deployment that provides real-time business intelligence for a retail company, correlating point-of-sale transactions, inventory data, and customer behavior logs?
21. Describe how you would implement Splunk's multi-cloud monitoring for an organization spanning AWS, GCP, and Azure, with unified security monitoring and compliance reporting.
22. How do you implement a Splunk-based IoT monitoring platform for a manufacturing company with 100,000 sensors, including real-time anomaly detection and predictive maintenance?
23. Describe how you would implement Splunk's risk-based alerting (RBA) framework to reduce SOC alert fatigue while maintaining detection coverage for critical threats.
24. How do you implement a Splunk deployment that handles the ingestion and analysis of encrypted log data, including key management and selective decryption for authorized investigations?
25. Describe how you would implement Splunk's content pack deployment strategy for a managed security service provider (MSSP) serving 100 customers with different security requirements.

---

## Rapid-Fire Questions

1. What is the default Splunk web interface port?
2. What SPL command do you use to count events by field value?
3. What is the purpose of the `_time` field in Splunk?
4. What does `index=main sourcetype=syslog` search for?
5. What is the difference between `stats` and `eventstats` in SPL?
6. What SPL command extracts fields using regex patterns?
7. What is the purpose of Splunk's `_raw` field?
8. What does the `head 10` command do in SPL?
9. What is a Splunk bucket and what are its states?
10. What is the purpose of Splunk's `props.conf` `TIME_FORMAT` setting?
11. What SPL command do you use to calculate a moving average?
12. What is the difference between Splunk's `search` and `where` commands?
13. What is the purpose of Splunk's `_indextime` field?
14. What does `| stats count by host` return?
15. What is the Splunk `kvstore` used for?
16. What is the purpose of Splunk's `transforms.conf` `REGEX` setting?
17. What SPL command do you use to join two searches?
18. What is the difference between Splunk's `rex` and `extract` commands?
19. What is the purpose of Splunk's `_sourcetype` field?
20. What does `| timechart span=1h count` display?
21. What is the Splunk `fishbucket` used for?
22. What is the purpose of Splunk's `inputs.conf` `monitor` stanza?
23. What SPL command do you use to remove duplicate events?
24. What is the difference between Splunk's `append` and `appendcols` commands?
25. What is the purpose of Splunk's `_meta` field?
26. What does `| top limit=10 sourcetype` return?
27. What is the Splunk `dispatch` directory used for?
28. What is the purpose of Splunk's `outputs.conf` `tcpout` stanza?
29. What SPL command do you use to calculate percentiles?
30. What is the difference between Splunk's `eval` and `fieldformat` commands?
31. What is the purpose of Splunk's `_subsecond` field?
32. What does `| rare limit=5 host` return?
33. What is the Splunk `lookups` directory used for?
34. What is the purpose of Splunk's `props.conf` `BREAK_ONLY_BEFORE` setting?
35. What SPL command do you use to create a lookup table from search results?
36. What is the difference between Splunk's `transaction` and `stats` commands for session analysis?
37. What is the purpose of Splunk's `_cd` field?
38. What does `| makeresults count=10` generate?
39. What is the Splunk `modinput` used for?
40. What is the purpose of Splunk's `transforms.conf` `DEST_KEY` setting?
41. What SPL command do you use to format numbers and dates?
42. What is the difference between Splunk's `iplocation` and `geoip` commands?
43. What is the purpose of Splunk's `_bkt` field?
44. What does `| sistats count by host` do differently from `stats`?
45. What is the Splunk `splunk_search_history` index used for?
46. What is the purpose of Splunk's `props.conf` `MAX_EVENTS` setting?
47. What SPL command do you use to pivot data into a matrix format?
48. What is the difference between Splunk's `search` mode and `fast` mode?
49. What is the purpose of Splunk's `_serial` field?
50. What does `| gentimes start=-7d increment=1d` generate?
