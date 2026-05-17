# Logstash — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## Pipeline Architecture & Bottlenecks

1. Your Logstash pipeline is processing 100,000 events per second but has suddenly dropped to 10,000 events per second. No configuration changes were made. How do you diagnose the bottleneck using Logstash's monitoring API and node stats?

2. Explain how Logstash's pipeline workers, batch size, and batch delay settings interact. How do you tune these parameters for a pipeline that processes large, complex events versus a pipeline that processes many small events?

3. You have a single Logstash pipeline that handles both high-priority security events and low-priority application logs. The security events are being delayed by the application log processing. How do you redesign the pipeline architecture to ensure priority processing?

4. Describe how Logstash's pipeline-to-pipeline communication works using the `pipeline` input and output plugins. In what scenarios does this architecture improve performance compared to a single pipeline?

5. A Logstash pipeline is consuming 100% CPU on all worker threads. How do you identify which filter plugin is causing the CPU spike, and what optimization strategies do you apply?

6. Explain the difference between Logstash's in-memory queue and persistent queue. Describe a production scenario where the in-memory queue caused data loss and how switching to persistent queues resolved it.

7. How do you implement a Logstash pipeline that can handle backpressure from a slow Elasticsearch output without dropping events or causing memory exhaustion?

8. Describe how you would implement a Logstash pipeline for a multi-tenant environment where events from different customers must be routed to different Elasticsearch indices with different retention policies.

9. Your Logstash pipeline is processing events out of order, causing incorrect time-based aggregations in Elasticsearch. How do you implement event ordering without sacrificing throughput?

10. How do you implement Logstash pipeline monitoring and alerting to detect when a pipeline falls behind, when the queue is filling up, or when the output is experiencing errors?

---

## Queue Tuning & Persistent Queues

11. A Logstash persistent queue has grown to 500GB and is consuming all available disk space. The pipeline is processing events but the queue is not draining. How do you diagnose the root cause and implement an emergency remediation?

12. Explain how Logstash's persistent queue checkpoint mechanism works. What happens to in-flight events when Logstash crashes, and how does the checkpoint file ensure at-least-once delivery?

13. How do you tune Logstash's persistent queue settings (`queue.max_bytes`, `queue.checkpoint.writes`, `queue.page_capacity`) for a pipeline that needs to buffer up to 1 hour of events during Elasticsearch maintenance windows?

14. Describe the performance trade-offs between Logstash's persistent queue and using an external message queue (Kafka) as a buffer. When would you choose each approach?

15. A Logstash persistent queue is corrupted after an unexpected system shutdown. How do you recover the queue data, and what are the data loss implications?

16. How do you implement queue monitoring for Logstash to alert when the queue depth exceeds a threshold, indicating that the pipeline is falling behind?

17. Explain how Logstash's `queue.drain` setting affects shutdown behavior. In what scenarios would you enable or disable queue draining, and what are the trade-offs?

---

## Parsing Failures & Grok Debugging

18. A Logstash pipeline is generating a high rate of `_grokparsefailure` tags. The events are being sent to Elasticsearch but without the expected parsed fields. How do you systematically debug the grok pattern and implement a fallback strategy?

19. You need to write a grok pattern for a custom application log format that includes variable-length fields, optional sections, and nested JSON within a syslog envelope. Describe your approach to building and testing the pattern.

20. A grok filter is taking 500ms per event due to catastrophic backtracking in a regex pattern. How do you identify the problematic pattern using Logstash's slow log feature, and how do you rewrite the pattern to eliminate backtracking?

21. Describe how you would implement a grok pattern library for an organization with 50 different application log formats, including version control, testing, and deployment procedures.

22. How do you handle log format changes in production without pipeline downtime? Describe your strategy for deploying new grok patterns when an application changes its log format.

23. A grok pattern that works correctly in the Grok Debugger tool is failing in production Logstash. What are the possible causes of this discrepancy, and how do you diagnose them?

24. How do you implement conditional grok parsing in Logstash to handle multiple log formats from the same source, and what are the performance implications of using multiple grok patterns in sequence?

25. Describe how you would use Logstash's `dissect` filter as an alternative to grok for structured log parsing. When does dissect outperform grok, and what are its limitations?

---

## Throughput Optimization

26. You need to increase Logstash throughput from 50,000 to 500,000 events per second. Describe your optimization strategy, including pipeline configuration, JVM tuning, hardware considerations, and architectural changes.

27. How do you implement Logstash's `aggregate` filter for stateful event processing (e.g., correlating multi-event transactions) without causing memory leaks or performance degradation?

28. Describe how you would optimize a Logstash pipeline that performs DNS lookups for every event. The DNS lookups are causing high latency and are a bottleneck for throughput.

29. How do you implement event batching in Logstash to reduce the number of Elasticsearch bulk API calls while maintaining acceptable indexing latency?

30. A Logstash pipeline is performing expensive GeoIP lookups for every event. How do you implement caching, sampling, and conditional processing to reduce the GeoIP lookup overhead?

31. Explain how Logstash's JVM heap settings affect pipeline performance. How do you tune heap size and GC settings for a Logstash instance processing high-volume, complex events?

32. How do you implement Logstash pipeline profiling to identify which filters are consuming the most processing time, and what tools do you use beyond the built-in monitoring API?

---

## Scaling Logstash

33. You need to scale Logstash horizontally to handle a 10x increase in log volume. Describe your scaling strategy, including load balancing, state management, and ensuring no duplicate event processing.

34. How do you implement Logstash in a Kubernetes environment with horizontal pod autoscaling, ensuring that pipeline state (persistent queues) is handled correctly when pods scale up and down?

35. Describe the challenges of running stateful Logstash pipelines (using `aggregate` filter or persistent queues) in a horizontally scaled environment, and how you address them.

36. How do you implement blue-green deployments for Logstash pipeline updates to ensure zero-downtime configuration changes?

37. A Logstash cluster is experiencing uneven load distribution where some instances are processing 10x more events than others. How do you diagnose the load imbalance and implement even distribution?

38. Describe how you would implement Logstash autoscaling based on queue depth metrics, including the monitoring setup, scaling triggers, and scale-down safety mechanisms.

---

## Kafka Integration

39. Your Logstash Kafka input is experiencing consumer lag that is growing continuously. The Logstash pipeline appears healthy. How do you diagnose whether the issue is in Kafka consumer configuration, Logstash processing speed, or downstream output performance?

40. Explain how Logstash's Kafka input plugin handles consumer group rebalancing. What happens to in-flight events during a rebalance, and how do you minimize event loss or duplication?

41. How do you implement exactly-once semantics in a Logstash-to-Elasticsearch pipeline using Kafka as the message queue, and what are the limitations of this approach?

42. Describe how you would implement a Logstash pipeline that consumes from multiple Kafka topics with different schemas and routes events to different Elasticsearch indices based on topic and event content.

43. A Logstash Kafka consumer is being repeatedly kicked out of the consumer group due to `max.poll.interval.ms` timeouts. How do you diagnose the root cause and tune the Kafka consumer settings?

44. How do you implement Kafka topic partitioning strategies to optimize Logstash consumer parallelism, and how does the number of Kafka partitions relate to the number of Logstash pipeline workers?

45. Describe how you would implement a Logstash pipeline that uses Kafka as both input and output for a stream processing use case, including handling of processing failures and dead letter queue management.

---

## Dead Letter Queues

46. Your Logstash dead letter queue (DLQ) is filling up rapidly. How do you implement a DLQ monitoring strategy, analyze the failed events, and implement a reprocessing pipeline?

47. Explain how Logstash's DLQ differs from Kafka's dead letter topic. In what scenarios would you use each, and how do you implement a unified error handling strategy?

48. How do you implement automatic DLQ reprocessing in Logstash, including the logic to fix the root cause of failures before reprocessing, and how do you prevent infinite reprocessing loops?

49. A Logstash pipeline is sending events to the DLQ due to Elasticsearch mapping conflicts. How do you diagnose the mapping conflict, fix the pipeline configuration, and reprocess the DLQ events?

50. Describe how you would implement DLQ alerting and reporting to give operations teams visibility into the types and volumes of failed events without requiring manual DLQ inspection.

---

## Filter Plugins & Data Enrichment

51. How do you implement a Logstash pipeline that enriches events with data from an external database (e.g., user information from PostgreSQL) without creating a bottleneck, and how do you handle database connection failures?

52. Describe how you would use the Logstash `jdbc_streaming` filter for real-time event enrichment, including connection pooling, caching strategies, and handling of slow database queries.

53. How do you implement a Logstash pipeline that performs real-time IP reputation lookups using a threat intelligence feed, including cache management and feed update procedures?

54. Explain how Logstash's `ruby` filter can be used for complex event transformations, and what are the performance implications and security considerations of using inline Ruby code in production pipelines?

55. How do you implement event deduplication in Logstash using the `fingerprint` filter and Elasticsearch's document ID, and what are the trade-offs in terms of performance and deduplication accuracy?

---

## Input/Output Plugins & Integration

56. Your Logstash `elasticsearch` output is experiencing high indexing latency due to Elasticsearch cluster pressure. How do you implement output buffering, retry logic, and circuit breaking to handle Elasticsearch degradation gracefully?

57. How do you implement a Logstash pipeline that reads from S3 for historical log reprocessing while simultaneously processing real-time events from Kafka, ensuring that the historical processing doesn't impact real-time latency?

58. Describe how you would implement a Logstash pipeline that outputs to multiple destinations simultaneously (Elasticsearch, S3, and a Kafka topic for downstream consumers) with different filtering and formatting for each destination.

59. How do you implement the Logstash `http` output plugin for sending events to a webhook endpoint, including retry logic, authentication, and handling of rate limiting responses?

60. Explain how you would implement a Logstash pipeline that processes Windows Event Log data from multiple sources, normalizing the event format and enriching with Active Directory information.

---

## Kubernetes & Cloud Deployment

61. How do you deploy Logstash on Kubernetes with persistent queues, ensuring that queue data is preserved when pods are rescheduled, and how do you handle pod disruptions during maintenance?

62. Describe how you would implement Logstash configuration management on Kubernetes using ConfigMaps, including hot-reload of pipeline configurations without pod restarts.

63. How do you implement Logstash resource management on Kubernetes, including CPU and memory requests/limits, JVM heap sizing, and handling of OOMKilled events?

64. Describe how you would implement a Logstash deployment that automatically scales based on Kafka consumer lag metrics using Kubernetes HPA with custom metrics.

65. How do you implement Logstash pipeline monitoring on Kubernetes, including the collection of pipeline metrics, JVM metrics, and integration with Prometheus and Grafana?

---

## Security & Compliance

66. How do you implement TLS encryption for all Logstash inputs and outputs, including certificate management, rotation procedures, and handling of certificate expiration in production?

67. Describe how you would implement data masking and redaction in Logstash pipelines to remove PII (personally identifiable information) before events are stored in Elasticsearch, including handling of nested JSON structures.

68. How do you implement Logstash pipeline access controls to prevent unauthorized pipeline configuration changes, and how do you audit pipeline configuration changes for compliance?

69. Describe how you would implement a Logstash pipeline that handles GDPR data subject deletion requests, including the identification and removal of events containing specific user data.

---

## Advanced Troubleshooting Scenarios

70. A Logstash pipeline that was processing events correctly suddenly starts producing events with incorrect timestamps. The events are being indexed into the wrong time-based indices. How do you diagnose the timestamp issue and fix it without reprocessing all historical data?

71. Your Logstash instance is experiencing memory leaks that cause it to be OOMKilled every 6 hours. How do you diagnose the memory leak using JVM heap dumps and Logstash monitoring metrics?

72. A Logstash pipeline is intermittently dropping events with no error messages in the logs. How do you implement comprehensive event tracking to identify where events are being lost in the pipeline?

73. Describe how you would implement a Logstash pipeline that processes structured logs from a microservices architecture, correlating events from different services using a distributed trace ID.

74. How do you implement Logstash pipeline testing in a CI/CD environment, including unit testing of filter configurations, integration testing with real data, and performance benchmarking?

75. A Logstash upgrade caused a breaking change in a filter plugin that is used by 20 pipelines. How do you implement a rollback strategy and plan a safe migration to the new plugin version?

76. Describe how you would implement a Logstash pipeline that handles log rotation events correctly, ensuring that no events are lost when log files are rotated and that duplicate events are not generated.

77. How do you implement a Logstash pipeline that processes high-volume metrics data and routes it to both Elasticsearch (for log correlation) and a time-series database (for metrics storage) with different data formats?

78. You're implementing Logstash as part of a compliance logging architecture that requires tamper-evident logs. How do you implement cryptographic signing of log events in the Logstash pipeline?

---

## Basic Questions

1. What is Logstash and what is its role in the ELK stack?
2. What are the three main components of a Logstash pipeline?
3. What is a Logstash input plugin and give three examples?
4. What is a Logstash filter plugin and give three examples?
5. What is a Logstash output plugin and give three examples?
6. What is the purpose of the `grok` filter in Logstash?
7. How do you start Logstash with a specific configuration file?
8. What is the purpose of the `mutate` filter in Logstash?
9. What is a Logstash codec and give two examples?
10. How do you test a Logstash configuration without starting the full pipeline?
11. What is the purpose of the `date` filter in Logstash?
12. How do you configure Logstash to read from a file input?
13. What is the purpose of the `if` conditional in a Logstash pipeline?
14. What is the Logstash `beats` input plugin used for?
15. How do you configure Logstash to output to Elasticsearch?
16. What is the purpose of the `add_field` option in Logstash filters?
17. What is a Logstash pipeline worker and what does it do?
18. How do you configure Logstash's logging level for debugging?
19. What is the purpose of the `remove_field` option in Logstash filters?
20. What is the Logstash `stdin` input plugin used for?
21. How do you configure Logstash to parse JSON log events?
22. What is the purpose of the `tags` field in Logstash events?
23. What is the Logstash `stdout` output plugin used for?
24. How do you configure Logstash to add a timestamp to events?
25. What is the purpose of the `@metadata` field in Logstash?
26. How do you configure Logstash to drop specific events using a filter?
27. What is the Logstash `tcp` input plugin used for?
28. How do you configure Logstash to parse syslog format events?
29. What is the purpose of the `rename` option in the Logstash `mutate` filter?
30. What is the Logstash `http` input plugin used for?
31. How do you configure Logstash to route events to different outputs based on field values?
32. What is the purpose of the `gsub` option in the Logstash `mutate` filter?
33. What is the Logstash `kafka` input plugin used for?
34. How do you configure Logstash's pipeline batch size?
35. What is the purpose of the `convert` option in the Logstash `mutate` filter?
36. How do you configure Logstash to handle multiline log events?
37. What is the Logstash `elasticsearch` input plugin used for?
38. How do you configure Logstash to add GeoIP information to events?
39. What is the purpose of the `split` filter in Logstash?
40. How do you configure Logstash to use environment variables in configuration files?
41. What is the Logstash `s3` input plugin used for?
42. How do you configure Logstash to parse CSV format events?
43. What is the purpose of the `clone` filter in Logstash?
44. How do you configure Logstash's dead letter queue?
45. What is the Logstash `http` output plugin used for?

---

## Intermediate Questions

1. How do you implement Logstash's persistent queue to prevent data loss during pipeline restarts?
2. Describe how you would configure Logstash's pipeline-to-pipeline communication for event routing.
3. How do you implement Logstash's `aggregate` filter for correlating multi-event transactions?
4. Explain how Logstash's `dissect` filter differs from `grok` and when each is more appropriate.
5. How do you implement Logstash's Kafka input with consumer group management for parallel processing?
6. Describe how you would configure Logstash's Elasticsearch output with index lifecycle management integration.
7. How do you implement Logstash's `ruby` filter for complex event transformations?
8. Explain how Logstash's queue checkpoint mechanism ensures at-least-once delivery.
9. How do you implement Logstash's `jdbc_streaming` filter for real-time event enrichment from a database?
10. Describe how you would configure Logstash's pipeline workers and batch settings for optimal throughput.
11. How do you implement Logstash's `fingerprint` filter for event deduplication?
12. Explain how Logstash's `translate` filter works for field value mapping.
13. How do you implement Logstash's `throttle` filter for rate limiting event processing?
14. Describe how you would configure Logstash's S3 output for archiving events to object storage.
15. How do you implement Logstash's `xml` filter for parsing XML log events?
16. Explain how Logstash's `useragent` filter works for parsing user agent strings.
17. How do you implement Logstash's `cidr` filter for IP address range matching?
18. Describe how you would configure Logstash's Kafka output with custom partitioning strategies.
19. How do you implement Logstash's `csv` filter for parsing structured CSV data?
20. Explain how Logstash's `elapsed` filter works for measuring event processing time.
21. How do you implement Logstash's `prune` filter for removing unwanted fields?
22. Describe how you would configure Logstash's HTTP output with authentication and retry logic.
23. How do you implement Logstash's `de_dot` filter for handling field names with dots.
24. Explain how Logstash's `sleep` filter works and when it is useful.
25. How do you implement Logstash's `truncate` filter for limiting field value lengths.
26. Describe how you would configure Logstash's monitoring API for pipeline health tracking.
27. How do you implement Logstash's `urldecode` filter for URL-encoded field values.
28. Explain how Logstash's `kv` filter works for key-value pair parsing.
29. How do you implement Logstash's `json_encode` filter for serializing fields to JSON strings.
30. Describe how you would configure Logstash's pipeline configuration reloading for zero-downtime updates.
31. How do you implement Logstash's `metrics` filter for counting and timing events.
32. Explain how Logstash's `collate` filter works for merging events from multiple inputs.
33. How do you implement Logstash's `extractnumbers` filter for numeric field extraction.
34. Describe how you would configure Logstash's Elasticsearch output with custom index templates.
35. How do you implement Logstash's `tld` filter for top-level domain extraction from URLs.
36. Explain how Logstash's `environment` filter works for adding environment variables to events.
37. How do you implement Logstash's `range` filter for field value validation.
38. Describe how you would configure Logstash's dead letter queue reprocessing pipeline.
39. How do you implement Logstash's `uuid` filter for generating unique event identifiers.
40. Explain how Logstash's `zeromq` input plugin works for high-performance message passing.

---

## Advanced Questions

1. Design a Logstash architecture for processing 10 million events per second from 5,000 microservices, including pipeline design, scaling strategy, and failure handling.
2. How do you implement a Logstash-based data enrichment platform that performs real-time lookups against multiple external data sources (threat intelligence, CMDB, user directory) without creating bottlenecks?
3. Describe how you would implement a Logstash deployment that handles exactly-once processing semantics using Kafka transactions and Elasticsearch document IDs.
4. How do you implement a Logstash pipeline that automatically adapts its processing logic based on the schema of incoming events, handling schema evolution without pipeline restarts?
5. Describe how you would implement a Logstash-based compliance logging platform that ensures tamper-evident logs with cryptographic signing and chain-of-custody tracking.
6. How do you implement a Logstash deployment that can survive a complete Elasticsearch cluster failure, buffering events for up to 24 hours and replaying them when the cluster recovers?
7. Describe how you would implement a Logstash pipeline for a financial services company that processes transaction logs, detects fraud patterns in real-time, and routes suspicious events to a SIEM.
8. How do you implement a Logstash deployment that automatically scales based on Kafka consumer lag, with zero-downtime scaling and proper state management for stateful filters?
9. Describe how you would implement a Logstash-based data masking platform that automatically detects and redacts PII from log events using machine learning-based entity recognition.
10. How do you implement a Logstash pipeline that processes logs from 100 different application types, each with different formats, using a dynamic configuration system that loads parsing rules from a database?
11. Describe how you would implement a Logstash deployment for a healthcare organization that processes HL7 medical records, ensuring HIPAA compliance with field-level encryption and access auditing.
12. How do you implement a Logstash pipeline that correlates security events across multiple log sources to detect multi-stage attack patterns, using the `aggregate` filter and external state storage?
13. Describe how you would implement a Logstash deployment that provides real-time data quality monitoring, detecting and alerting on parsing failures, missing fields, and data anomalies.
14. How do you implement a Logstash-based event streaming platform that routes events to multiple downstream consumers (Elasticsearch, Kafka, S3, SIEM) with different filtering and transformation for each?
15. Describe how you would implement a Logstash deployment that handles log backfill for historical data migration, processing years of archived logs while maintaining real-time processing for current events.
16. How do you implement a Logstash pipeline that performs real-time log classification using a machine learning model, routing events to different processing paths based on classification results?
17. Describe how you would implement a Logstash deployment for a multi-cloud environment, collecting logs from AWS CloudWatch, GCP Cloud Logging, and Azure Monitor with unified normalization.
18. How do you implement a Logstash-based observability pipeline that automatically extracts metrics from unstructured log data and forwards them to Prometheus for alerting?
19. Describe how you would implement a Logstash deployment that provides end-to-end event tracing, adding trace IDs to all events and correlating them with distributed traces in Jaeger or Zipkin.
20. How do you implement a Logstash pipeline that handles high-cardinality log data, automatically detecting and dropping or sampling high-volume low-value events to control Elasticsearch storage costs?
21. Describe how you would implement a Logstash deployment that processes IoT sensor data from 1 million devices, handling protocol translation, data normalization, and real-time anomaly detection.
22. How do you implement a Logstash-based audit logging platform that captures all database operations, API calls, and user actions, providing immutable audit trails for compliance investigations?
23. Describe how you would implement a Logstash deployment that automatically recovers from pipeline failures, replaying events from Kafka without duplicates and maintaining processing order guarantees.
24. How do you implement a Logstash pipeline that performs real-time geospatial enrichment, adding location context to events and routing them to region-specific Elasticsearch clusters for data sovereignty?
25. Describe how you would implement a Logstash deployment that provides a self-service pipeline configuration platform, allowing application teams to define their own parsing rules and routing logic without SRE involvement.

---

## Rapid-Fire Questions

1. What is the default port for Logstash's Beats input?
2. What does the `grok` filter do in Logstash?
3. What is the purpose of the `@timestamp` field in Logstash events?
4. What does `%{COMBINEDAPACHELOG}` match in a grok pattern?
5. What is the purpose of Logstash's `--config.test_and_exit` flag?
6. What does the `mutate` filter's `convert` option do?
7. What is the purpose of Logstash's persistent queue?
8. What does `_grokparsefailure` tag indicate?
9. What is the purpose of Logstash's `--pipeline.workers` setting?
10. What does the `date` filter do in Logstash?
11. What is the purpose of Logstash's `--pipeline.batch.size` setting?
12. What does `_jsonparsefailure` tag indicate?
13. What is the purpose of Logstash's `@metadata` field?
14. What does the `fingerprint` filter do in Logstash?
15. What is the purpose of Logstash's `--config.reload.automatic` flag?
16. What does the `dissect` filter do in Logstash?
17. What is the purpose of Logstash's dead letter queue?
18. What does `_dateparsefailure` tag indicate?
19. What is the purpose of Logstash's `--path.queue` setting?
20. What does the `aggregate` filter do in Logstash?
21. What is the purpose of Logstash's `--pipeline.batch.delay` setting?
22. What does the `clone` filter do in Logstash?
23. What is the purpose of Logstash's `--log.level` setting?
24. What does the `split` filter do in Logstash?
25. What is the purpose of Logstash's `--queue.type` setting?
26. What does the `translate` filter do in Logstash?
27. What is the purpose of Logstash's `--queue.max_bytes` setting?
28. What does the `throttle` filter do in Logstash?
29. What is the purpose of Logstash's `--pipeline.id` setting?
30. What does the `prune` filter do in Logstash?
31. What is the purpose of Logstash's `--path.config` setting?
32. What does the `ruby` filter do in Logstash?
33. What is the purpose of Logstash's `--node.name` setting?
34. What does the `kv` filter do in Logstash?
35. What is the purpose of Logstash's `--http.port` setting?
36. What does the `useragent` filter do in Logstash?
37. What is the purpose of Logstash's `--queue.checkpoint.writes` setting?
38. What does the `geoip` filter do in Logstash?
39. What is the purpose of Logstash's `--pipeline.unsafe_shutdown` setting?
40. What does the `cidr` filter do in Logstash?
41. What is the purpose of Logstash's `--config.reload.interval` setting?
42. What does the `elapsed` filter do in Logstash?
43. What is the purpose of Logstash's `--pipeline.ecs_compatibility` setting?
44. What does the `truncate` filter do in Logstash?
45. What is the purpose of Logstash's `--queue.page_capacity` setting?
46. What does the `de_dot` filter do in Logstash?
47. What is the purpose of Logstash's `--modules` setting?
48. What does the `metrics` filter do in Logstash?
49. What is the purpose of Logstash's `--cloud.id` setting?
50. What does the `uuid` filter do in Logstash?
