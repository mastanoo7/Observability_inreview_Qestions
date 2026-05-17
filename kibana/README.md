# Kibana — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## Dashboard Troubleshooting & Design

1. A Kibana dashboard that was working correctly yesterday is now showing "No results found" for all visualizations. The Elasticsearch cluster is healthy and the data exists when queried directly. Walk through your systematic debugging approach, including index pattern validation, time filter issues, and Kibana server logs.

2. A Kibana dashboard with 30 visualizations is taking 3 minutes to load. How do you identify which visualizations are slow, diagnose whether the bottleneck is Kibana-side or Elasticsearch-side, and implement optimizations?

3. You need to design a Kibana dashboard for a 24/7 NOC team that monitors 500 microservices. The dashboard must show real-time health status, alert counts, and top error sources without overwhelming the operators. How do you structure this?

4. A Kibana dashboard is showing incorrect data after an Elasticsearch index mapping change. Some fields are now showing as "unknown" or displaying raw values instead of formatted values. How do you diagnose and fix this?

5. How do you implement Kibana dashboard version control and change management in an enterprise environment where multiple teams contribute to shared dashboards?

6. Describe how you would implement a Kibana dashboard that automatically refreshes every 30 seconds for real-time monitoring without causing excessive load on the Elasticsearch cluster.

7. A Kibana lens visualization is showing aggregation results that don't match the raw document count. How do you diagnose whether this is a sampling issue, an aggregation configuration error, or a data quality problem?

8. How do you implement Kibana's drilldown feature to create interactive dashboards where clicking on a data point navigates to a more detailed view with pre-filtered context?

9. Describe how you would implement a Kibana dashboard that correlates application logs, infrastructure metrics (from Metricbeat), and APM traces on a single timeline for incident investigation.

10. You need to create a Kibana dashboard that shows the geographic distribution of errors across a global application. How do you implement this using Kibana's Maps feature, and what data requirements are needed?

---

## Role-Based Access Control (RBAC)

11. Your organization has 100 teams using a shared Kibana instance. A team accidentally deleted a shared index pattern used by 50 dashboards. How do you implement RBAC to prevent this while still allowing teams to create their own index patterns?

12. Describe how you would implement Kibana's space-based access control to support a scenario where: the security team has access to security dashboards, the operations team has access to infrastructure dashboards, and executives have access to a read-only business metrics space.

13. How do you implement Kibana's feature controls to restrict specific teams from accessing APM, Uptime, or SIEM features while allowing them to use dashboards and visualizations?

14. A Kibana user is able to see data from indices they shouldn't have access to by modifying the index pattern in a saved search. How do you implement document-level security and field-level security in Elasticsearch to enforce data access controls at the data layer?

15. Describe how you would integrate Kibana with an enterprise SSO provider (Okta/Azure AD) using SAML authentication, including role mapping from SAML attributes to Kibana roles.

16. How do you implement Kibana's API key management for service accounts that need programmatic access to Kibana APIs, including key rotation and access auditing?

17. A Kibana space has been configured with incorrect permissions, allowing users to modify dashboards they should only be able to view. How do you audit the current permissions and implement the correct RBAC configuration?

18. Describe how you would implement a Kibana RBAC model for a managed service provider (MSP) that needs to give each customer access to their own data while preventing cross-customer data access.

---

## Visualizations & Analytics

19. How do you implement a Kibana TSVB (Time Series Visual Builder) visualization for a metric that requires complex mathematical transformations, such as calculating error rate as a percentage of total requests with a moving average overlay?

20. Describe how you would use Kibana's Vega visualization to create a custom chart type that is not available in the standard visualization library, including the data query and rendering configuration.

21. A Kibana aggregation-based visualization is showing incorrect results for a high-cardinality field because Elasticsearch is using sampling. How do you configure the visualization to get accurate results, and what are the performance trade-offs?

22. How do you implement Kibana's Canvas feature to create a pixel-perfect operational status board for executive reporting, including real-time data updates and custom branding?

23. Describe how you would implement a Kibana visualization that shows the correlation between deployment events and error rate changes, using annotations to mark deployment timestamps on a time series chart.

24. How do you implement Kibana's machine learning features (anomaly detection) to automatically detect unusual patterns in log data, and how do you integrate ML results into operational dashboards?

25. A Kibana pie chart visualization is showing a large "Other" category that obscures important data. How do you configure the visualization to show more buckets, and when should you use a different visualization type instead?

26. Describe how you would implement a Kibana dashboard that shows SLO compliance metrics, including error budget burn rate, remaining error budget, and historical SLO performance trends.

---

## Alerting & Notification

27. Your Kibana alerting rules are generating hundreds of notifications per hour due to alert flapping. How do you implement alert conditions, thresholds, and notification throttling to reduce noise while maintaining signal quality?

28. Explain the difference between Kibana's stack rules (index threshold, ES query) and Kibana's observability rules (log threshold, metric threshold, uptime). When would you use each type in a production environment?

29. A Kibana alert rule is failing to execute with "Request timeout" errors. How do you diagnose whether this is a Kibana server issue, an Elasticsearch query performance issue, or a network issue?

30. How do you implement Kibana alerting for a scenario where you need to alert when the rate of 5xx errors exceeds 1% of total requests over a 5-minute window, with different thresholds for different services?

31. Describe how you would implement Kibana's action connectors for a multi-channel notification strategy that sends PagerDuty alerts for critical issues, Slack messages for warnings, and email digests for informational alerts.

32. How do you implement Kibana alert maintenance windows to suppress notifications during planned maintenance, and how do you ensure that alerts that fire during maintenance are reviewed after the window closes?

33. A Kibana alert that uses a complex KQL query is producing false positives. How do you debug the alert condition using Kibana's alert testing features and refine the query to reduce false positives?

34. Describe how you would implement Kibana's case management integration with alerting to automatically create incident tickets when critical alerts fire.

---

## Index Patterns & Data Management

35. You have 500 time-based indices following the pattern `logs-app-YYYY.MM.DD`. How do you create and manage Kibana index patterns that span all these indices efficiently, and how do you handle index pattern refresh when new indices are created?

36. An index pattern in Kibana is showing incorrect field types because the underlying Elasticsearch mapping has changed. How do you refresh the index pattern without breaking existing dashboards that use the old field types?

37. How do you implement Kibana's data views (formerly index patterns) for a multi-cluster setup where the same data exists in different Elasticsearch clusters with slightly different field names?

38. Describe how you would implement a Kibana index pattern strategy for a microservices architecture where each service writes to its own index, but operators need to search across all services simultaneously.

39. How do you handle Kibana index pattern management when Elasticsearch ILM is rolling over indices and creating new indices with different names, ensuring that dashboards continue to work without manual updates?

40. A Kibana data view is showing "No cached mapping for this field" errors for recently added fields. How do you implement automatic field mapping refresh, and what are the implications for dashboard performance?

---

## Query Optimization & Performance

41. A Kibana saved search is timing out for queries spanning more than 7 days of data. How do you optimize the query using KQL filters, index pattern scoping, and Elasticsearch query DSL tuning?

42. How do you use Kibana's Inspector tool to analyze the Elasticsearch queries generated by visualizations, and what information does it provide for optimizing slow queries?

43. Describe how you would implement Kibana's search sessions feature to allow users to run long-running queries in the background without blocking the UI.

44. A Kibana dashboard is making 50 separate Elasticsearch queries for each page load. How do you reduce the query count using dashboard-level time filters, shared queries, and visualization consolidation?

45. How do you implement Kibana's runtime fields to add computed fields to documents at query time, and when is this preferable to adding fields at index time?

46. Describe how you would optimize Kibana's Discover view for a high-volume index (1 billion documents) to provide fast search results without overwhelming Elasticsearch.

47. How do you implement Kibana's field statistics feature to understand data distribution and identify data quality issues before building visualizations?

---

## Observability Integrations

48. How do you configure Kibana's APM integration to provide end-to-end transaction tracing, including the correlation of APM traces with application logs and infrastructure metrics?

49. Describe how you would implement Kibana's Uptime monitoring for a global application with 200 endpoints, including the configuration of synthetic monitors and alert thresholds.

50. How do you use Kibana's Infrastructure view to monitor Kubernetes cluster health, including the correlation of pod metrics, node metrics, and application logs in a unified view?

51. Describe how you would implement Kibana's Logs view for real-time log streaming during incident response, including the configuration of log categories, highlights, and contextual links to related metrics.

52. How do you implement Kibana's Service Map feature for a microservices architecture to visualize service dependencies and identify cascading failure patterns during incidents?

53. Describe how you would use Kibana's Profiling feature (Universal Profiling) to identify CPU hotspots in production applications without requiring code instrumentation.

54. How do you implement Kibana's Fleet management for deploying and managing Elastic Agents across a large fleet of hosts, including policy management and agent health monitoring?

---

## Multi-Space Management

55. Your organization has grown to 20 teams, each requiring isolated Kibana spaces with their own dashboards, index patterns, and saved searches. How do you implement a space provisioning process that can be automated and managed as code?

56. Describe how you would implement cross-space dashboard sharing in Kibana, where a shared infrastructure dashboard is visible in all team spaces but only editable by the platform team.

57. How do you implement Kibana space migration when reorganizing teams, including moving dashboards, saved searches, and index patterns between spaces without breaking references?

58. Describe how you would implement a Kibana space template system that automatically creates a standard set of dashboards and index patterns when a new team space is provisioned.

59. How do you implement Kibana's copy-to-space feature programmatically using the Kibana API to synchronize dashboards across multiple spaces?

---

## Kibana as Code & Automation

60. How do you implement Kibana configuration as code using the Kibana saved objects API, including the export, version control, and import of dashboards, visualizations, and index patterns?

61. Describe how you would implement a CI/CD pipeline for Kibana dashboard deployment, including automated testing of dashboard configurations and rollback procedures.

62. How do you use the Kibana API to programmatically create and manage alerting rules, action connectors, and maintenance windows as part of an infrastructure-as-code workflow?

63. Describe how you would implement automated Kibana health monitoring that checks dashboard availability, alert rule execution, and data freshness, and alerts the operations team when issues are detected.

64. How do you implement Kibana's reporting feature to automatically generate and distribute PDF reports of key dashboards to stakeholders on a scheduled basis?

---

## Security & Compliance

65. How do you implement Kibana's audit logging to track all user actions, including dashboard views, data queries, and configuration changes, for compliance with SOC 2 requirements?

66. Describe how you would implement Kibana's encrypted saved objects feature to protect sensitive data (such as API keys in action connectors) stored in the Kibana index.

67. How do you implement IP-based access restrictions for Kibana to limit access to specific network ranges, and how do you handle access for remote workers and cloud-based CI/CD systems?

68. Describe how you would implement a Kibana security review process for a PCI-DSS compliant environment, including the controls needed for cardholder data environment (CDE) monitoring dashboards.

---

## Advanced Kibana Scenarios

69. A Kibana upgrade broke 30 dashboards because a visualization type was deprecated. How do you implement a pre-upgrade testing process and a migration strategy for deprecated visualization types?

70. How do you implement Kibana's Lens formula feature to create complex calculated metrics, such as Apdex scores, error budget burn rates, and composite health scores?

71. Describe how you would implement a Kibana-based incident management workflow that integrates with PagerDuty, Jira, and Slack, using Kibana cases and alerting as the central coordination point.

72. How do you implement Kibana's custom branding and white-labeling for a managed service offering where each customer sees a customized Kibana interface?

73. Describe how you would use Kibana's Discover timeline feature to reconstruct the sequence of events leading up to a production incident, including correlating events from multiple log sources.

74. How do you implement Kibana's threshold-based anomaly detection for business metrics (e.g., order volume, payment processing rate) that have strong seasonal patterns?

75. A Kibana instance is experiencing high memory usage due to a large number of concurrent users running complex queries. How do you implement query resource limits, user session management, and Kibana scaling to handle the load?

76. Describe how you would implement a Kibana-based SRE dashboard suite that covers all four golden signals (latency, traffic, errors, saturation) for a microservices architecture, with automatic service discovery.

77. How do you implement Kibana's transform feature to create summarized indices for long-term trend analysis, and how do you manage the lifecycle of transform destination indices?

78. Describe how you would implement a Kibana monitoring strategy for the Elastic Stack itself, including monitoring Elasticsearch cluster health, Logstash pipeline performance, and Kibana server health from within Kibana.

---

## Basic Questions

1. What is Kibana and what is its primary role in the Elastic Stack?
2. What is a Kibana index pattern and how do you create one?
3. What is the Kibana Discover view used for?
4. What is the difference between Kibana's Visualize and Dashboard features?
5. What is KQL (Kibana Query Language) and how does it differ from Lucene query syntax?
6. How do you create a basic bar chart visualization in Kibana?
7. What is the purpose of Kibana's time filter?
8. What is a Kibana saved search and when would you use it?
9. How do you share a Kibana dashboard with a team member?
10. What is the purpose of Kibana's Lens editor?
11. What is a Kibana space and what is it used for?
12. How do you create a Kibana alert based on an index threshold?
13. What is the purpose of Kibana's Maps feature?
14. How do you configure Kibana's default index pattern?
15. What is the Kibana Canvas feature used for?
16. How do you add a filter to a Kibana dashboard?
17. What is the purpose of Kibana's TSVB (Time Series Visual Builder)?
18. How do you configure Kibana to connect to an Elasticsearch cluster?
19. What is the Kibana Timelion feature used for?
20. How do you create a Kibana data table visualization?
21. What is the purpose of Kibana's field formatter?
22. How do you configure Kibana's dark mode?
23. What is the Kibana APM feature used for?
24. How do you create a Kibana pie chart visualization?
25. What is the purpose of Kibana's tag cloud visualization?
26. How do you configure Kibana's session timeout?
27. What is the Kibana Uptime feature used for?
28. How do you create a Kibana metric visualization?
29. What is the purpose of Kibana's heat map visualization?
30. How do you configure Kibana's default time zone?
31. What is the Kibana SIEM/Security feature used for?
32. How do you create a Kibana coordinate map visualization?
33. What is the purpose of Kibana's goal and gauge visualization?
34. How do you configure Kibana's reporting feature for PDF generation?
35. What is the Kibana Logs feature used for?
36. How do you create a Kibana vertical bar chart with multiple metrics?
37. What is the purpose of Kibana's region map visualization?
38. How do you configure Kibana's index pattern field refresh?
39. What is the Kibana Infrastructure feature used for?
40. How do you create a Kibana line chart with multiple series?
41. What is the purpose of Kibana's saved objects management?
42. How do you configure Kibana's advanced settings?
43. What is the Kibana Dev Tools console used for?
44. How do you create a Kibana area chart visualization?
45. What is the purpose of Kibana's index management feature?

---

## Intermediate Questions

1. How do you implement Kibana's RBAC to support a multi-team environment with different dashboard access levels?
2. Describe how you would configure Kibana's SAML authentication for enterprise SSO integration.
3. How do you implement Kibana's alerting rules for log threshold monitoring with multi-channel notifications?
4. Explain how Kibana's Lens formula feature works for creating complex calculated metrics.
5. How do you implement Kibana's space-based multi-tenancy for isolating different team's dashboards and data?
6. Describe how you would configure Kibana's APM integration for end-to-end transaction tracing.
7. How do you implement Kibana's machine learning anomaly detection for log-based anomaly identification?
8. Explain how Kibana's runtime fields work and when they are preferable to index-time fields.
9. How do you implement Kibana's dashboard drilldown feature for interactive data exploration?
10. Describe how you would configure Kibana's Fleet management for Elastic Agent deployment at scale.
11. How do you implement Kibana's transform feature for creating summarized indices for reporting?
12. Explain how Kibana's search sessions work for long-running background queries.
13. How do you implement Kibana's case management for incident tracking and collaboration?
14. Describe how you would configure Kibana's Osquery integration for endpoint security monitoring.
15. How do you implement Kibana's alerting with action connectors for automated incident response?
16. Explain how Kibana's data view (index pattern) field statistics help with data quality assessment.
17. How do you implement Kibana's Canvas workpad for executive-level operational dashboards?
18. Describe how you would configure Kibana's cross-cluster search for querying multiple Elasticsearch clusters.
19. How do you implement Kibana's Vega visualization for custom chart types not available in standard panels?
20. Explain how Kibana's aggregation-based visualizations differ from Lens-based visualizations in terms of capabilities.
21. How do you implement Kibana's dashboard-to-dashboard navigation with context preservation?
22. Describe how you would configure Kibana's Elasticsearch query optimization for slow dashboard panels.
23. How do you implement Kibana's role-based feature controls to restrict access to specific Kibana features?
24. Explain how Kibana's service map works for visualizing microservice dependencies from APM data.
25. How do you implement Kibana's alerting for Kubernetes pod failures using Metricbeat data?
26. Describe how you would configure Kibana's index lifecycle management integration for data tier visualization.
27. How do you implement Kibana's profiling feature for CPU hotspot identification in production applications?
28. Explain how Kibana's Discover timeline feature helps reconstruct incident sequences.
29. How do you implement Kibana's saved object export/import for dashboard migration between environments?
30. Describe how you would configure Kibana's audit logging for compliance with SOC 2 requirements.
31. How do you implement Kibana's threshold-based anomaly detection for business metric monitoring?
32. Explain how Kibana's field-level security integration with Elasticsearch restricts data visibility.
33. How do you implement Kibana's reporting schedule for automated dashboard distribution to stakeholders?
34. Describe how you would configure Kibana's Synthetics monitoring for global endpoint availability checking.
35. How do you implement Kibana's data table with computed columns for SLO compliance reporting?
36. Explain how Kibana's Timelion expressions work for complex time series calculations.
37. How do you implement Kibana's tag-based dashboard organization for large-scale dashboard management?
38. Describe how you would configure Kibana's custom branding for a white-labeled observability platform.
39. How do you implement Kibana's alert suppression rules to prevent duplicate notifications during incidents?
40. Explain how Kibana's Universal Profiling integrates with the Elastic Stack for continuous profiling.

---

## Advanced Questions

1. Design a Kibana deployment for a 500-team enterprise with automated space provisioning, centralized RBAC management, SSO integration, and GitOps-based dashboard deployment.
2. How do you implement a Kibana-based security operations center (SOC) platform that provides real-time threat detection, incident management, and automated response workflows?
3. Describe how you would implement Kibana's observability suite for a microservices platform with 1,000 services, providing automatic service discovery, SLO tracking, and incident correlation.
4. How do you implement a Kibana deployment that maintains dashboard availability during Elasticsearch cluster degradation, including fallback strategies and cached data display?
5. Describe how you would implement a Kibana-based compliance monitoring platform for a financial services company, meeting SOX, PCI-DSS, and GDPR requirements simultaneously.
6. How do you implement Kibana's machine learning features at scale for automated anomaly detection across 10,000 metrics, with intelligent alert routing and false positive suppression?
7. Describe how you would implement a Kibana deployment for a multi-cloud environment, providing unified visibility across AWS, GCP, and Azure with normalized metrics and consistent dashboards.
8. How do you implement a Kibana-based incident response platform that automatically populates war room dashboards, correlates relevant signals, and tracks incident resolution progress?
9. Describe how you would implement Kibana's RBAC model for a managed service provider with 200 customers, ensuring complete data isolation and customer-specific dashboard customization.
10. How do you implement a Kibana deployment that handles 50,000 concurrent users with consistent sub-second dashboard load times, including query optimization and caching strategies?
11. Describe how you would implement a Kibana-based DevSecOps platform that integrates security scanning results, deployment events, and runtime security alerts in a unified view.
12. How do you implement Kibana's APM integration for a polyglot microservices architecture with 20 different programming languages, ensuring consistent trace correlation and service mapping?
13. Describe how you would implement a Kibana deployment that automatically generates and distributes compliance reports, including evidence collection, audit trail documentation, and regulatory submission.
14. How do you implement Kibana's alerting at enterprise scale with 50,000 alert rules, ensuring reliable evaluation, deduplication, and routing without performance degradation?
15. Describe how you would implement a Kibana-based capacity planning platform that uses historical metrics to forecast infrastructure requirements and generate automated scaling recommendations.
16. How do you implement Kibana's security analytics for a zero-trust network architecture, correlating identity, network, and endpoint telemetry to detect lateral movement and data exfiltration?
17. Describe how you would implement a Kibana deployment for a healthcare organization with HIPAA requirements, including PHI data masking, access auditing, and breach detection workflows.
18. How do you implement Kibana's Fleet management for a 10,000-node infrastructure, including automated agent deployment, policy management, and health monitoring?
19. Describe how you would implement a Kibana-based SRE platform that automatically calculates error budgets, tracks SLO compliance, and generates reliability reports for quarterly business reviews.
20. How do you implement Kibana's disaster recovery strategy, ensuring that all dashboards, alert rules, and configurations can be restored within 15 minutes of a complete Kibana instance failure?
21. Describe how you would implement a Kibana deployment that provides real-time business intelligence by combining operational metrics with business KPIs from external databases and APIs.
22. How do you implement Kibana's integration with a CMDB to automatically enrich monitoring dashboards with service ownership, business criticality, and dependency information?
23. Describe how you would implement a Kibana-based cost observability platform that correlates infrastructure metrics with cloud billing data to provide per-service cost attribution and optimization recommendations.
24. How do you implement Kibana's Canvas for a network operations center (NOC) display that shows real-time infrastructure health across 50 data centers with automatic incident highlighting?
25. Describe how you would implement a Kibana deployment that supports A/B testing of alert thresholds, measuring which configurations lead to better signal-to-noise ratios and faster incident resolution.

---

## Rapid-Fire Questions

1. What is the default port for Kibana?
2. What is the purpose of Kibana's `kibana.yml` configuration file?
3. What does KQL stand for in Kibana?
4. What is the purpose of Kibana's `_g` URL parameter?
5. What does Kibana's `Discover` view show by default?
6. What is the purpose of Kibana's `saved_objects` API?
7. What does the Kibana `Dev Tools` console allow you to do?
8. What is the purpose of Kibana's `index_pattern` saved object type?
9. What does Kibana's `Lens` editor replace?
10. What is the purpose of Kibana's `spaces` feature?
11. What does Kibana's `Canvas` feature allow you to create?
12. What is the purpose of Kibana's `alerting` feature?
13. What does Kibana's `APM` feature monitor?
14. What is the purpose of Kibana's `Fleet` feature?
15. What does Kibana's `Maps` feature display?
16. What is the purpose of Kibana's `Uptime` feature?
17. What does Kibana's `TSVB` stand for?
18. What is the purpose of Kibana's `reporting` feature?
19. What does Kibana's `Timelion` feature provide?
20. What is the purpose of Kibana's `role` management?
21. What does Kibana's `Profiling` feature analyze?
22. What is the purpose of Kibana's `Cases` feature?
23. What does Kibana's `Osquery` integration provide?
24. What is the purpose of Kibana's `Synthetics` feature?
25. What does Kibana's `Security` app provide?
26. What is the purpose of Kibana's `Index Management` feature?
27. What does Kibana's `Snapshot and Restore` feature manage?
28. What is the purpose of Kibana's `ILM` policy editor?
29. What does Kibana's `Cross-Cluster Replication` feature manage?
30. What is the purpose of Kibana's `Watcher` feature?
31. What does Kibana's `Machine Learning` feature provide?
32. What is the purpose of Kibana's `Transforms` feature?
33. What does Kibana's `Rollup Jobs` feature create?
34. What is the purpose of Kibana's `Enrich Policies` feature?
35. What does Kibana's `Remote Clusters` feature manage?
36. What is the purpose of Kibana's `License Management` feature?
37. What does Kibana's `Stack Monitoring` feature display?
38. What is the purpose of Kibana's `Upgrade Assistant`?
39. What does Kibana's `Painless Lab` feature provide?
40. What is the purpose of Kibana's `Search Profiler`?
41. What does Kibana's `Grok Debugger` feature help with?
42. What is the purpose of Kibana's `Logstash Pipelines` management?
43. What does Kibana's `Beats Central Management` feature provide?
44. What is the purpose of Kibana's `Advanced Settings`?
45. What does Kibana's `Saved Objects` management allow you to do?
46. What is the purpose of Kibana's `API Keys` management?
47. What does Kibana's `Audit Logs` feature track?
48. What is the purpose of Kibana's `Session Management`?
49. What does Kibana's `Maintenance Windows` feature provide?
50. What is the purpose of Kibana's `Action Connectors` feature?
