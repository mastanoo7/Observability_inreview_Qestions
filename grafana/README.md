# Grafana — Advanced SRE Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Observability Engineer
> **Focus:** Real-time production scenarios, troubleshooting, architecture, and scaling

---

## Dashboard Design & Troubleshooting

1. A Grafana dashboard that was working perfectly yesterday is now showing "No data" for all panels. The underlying Prometheus datasource is healthy and returning data via direct queries. Walk through your systematic debugging approach.

2. You're designing a dashboard for a payment processing system that needs to display real-time transaction rates, error rates, and latency percentiles across 50 microservices. How do you structure the dashboard to avoid performance issues while maintaining usability?

3. A Grafana dashboard with 40 panels is taking 45 seconds to load. Users are complaining about performance. How do you diagnose which panels are slow, and what optimization strategies do you apply?

4. Explain how Grafana's query caching works and how you would configure it to reduce load on a Prometheus backend that is being queried by 200 concurrent dashboard users.

5. You need to create a dashboard that shows the same metrics for different environments (dev, staging, prod) without duplicating the dashboard. How do you implement this using Grafana variables and templating?

6. A dashboard panel is showing data with a 10-minute delay compared to the actual metric values in Prometheus. The panel uses a `$__range` variable. What are the possible causes and how do you resolve them?

7. How do you implement a "fleet overview" dashboard for 500 Kubernetes nodes that allows drill-down to individual node metrics without creating separate dashboards for each node?

8. Describe your approach to dashboard version control and change management in an enterprise environment where multiple teams contribute to shared dashboards.

9. A Grafana dashboard is showing incorrect data after a Prometheus recording rule was renamed. How do you identify all dashboards affected by this change and implement a migration strategy?

10. You need to implement a real-time incident dashboard that automatically highlights anomalous metrics during an active incident. How do you design this using Grafana's features?

---

## Alerting & Notification

11. Your Grafana alerting system is generating 500 alert notifications per hour due to flapping. Describe your strategy for implementing alert grouping, deduplication, and routing to reduce noise while maintaining signal quality.

12. Explain the architectural differences between Grafana Legacy Alerting and Grafana Unified Alerting (Grafana 8+). In what production scenarios would the migration from legacy to unified alerting cause alert gaps?

13. A Grafana alert that uses a multi-dimensional query is firing for the wrong time series instances. How do you debug label matching in Grafana alert conditions and fix the targeting?

14. You need to implement alert routing in Grafana that sends PagerDuty notifications for critical alerts, Slack notifications for warnings, and email digests for informational alerts, with different routing for business hours vs. after hours.

15. Describe how you would implement alert silencing and maintenance windows programmatically in Grafana using the API, integrated with your CI/CD pipeline for deployment windows.

16. A Grafana alert is configured with a `for` duration of 5 minutes, but it's firing immediately without waiting. What are the possible causes in Grafana Unified Alerting, and how do you fix this?

17. How do you implement alert escalation policies in Grafana when the primary on-call engineer doesn't acknowledge an alert within 15 minutes?

18. Explain how Grafana's contact points and notification policies interact. Describe a scenario where a misconfigured notification policy caused critical alerts to be silently dropped.

19. You're using Grafana alerting with multiple datasources (Prometheus, Loki, and Elasticsearch). How do you implement correlated alerts that fire only when conditions are met across multiple datasources simultaneously?

20. Describe your approach to testing Grafana alert rules in a staging environment before promoting them to production, including how you simulate alert conditions without affecting production systems.

---

## Loki Integration

21. Your Grafana dashboards using Loki are showing log query timeouts for queries spanning more than 24 hours. How do you diagnose whether this is a Loki configuration issue, a query complexity issue, or a Grafana timeout issue?

22. You're implementing a correlated view in Grafana that links Prometheus metrics to Loki logs using exemplars. Walk through the configuration required on both the Prometheus and Loki sides, and how you configure the datasource linking in Grafana.

23. A Loki LogQL query in Grafana is returning results but the log volume panel shows significantly fewer entries than expected. How do you diagnose sampling, ingestion, and query-time filtering issues?

24. How do you implement a Grafana dashboard that shows the correlation between deployment events (from Loki) and metric anomalies (from Prometheus) on the same timeline?

25. Describe how you would use Grafana's Explore mode to perform a live incident investigation, correlating logs from Loki with traces from Tempo and metrics from Prometheus.

26. Your Loki datasource in Grafana is showing "context deadline exceeded" errors for certain log queries. The Loki cluster appears healthy. What are the Grafana-side configuration parameters you would tune?

27. How do you implement log-based alerting in Grafana using Loki, and what are the limitations compared to metric-based alerting in terms of reliability and latency?

28. Explain how Grafana's derived fields feature works with Loki logs. Describe a production use case where you used derived fields to create clickable links from log entries to distributed traces.

---

## Tempo Integration & Distributed Tracing

29. You're using Grafana with Tempo for distributed tracing. A specific trace is showing a 10-second gap between two spans. How do you use Grafana's trace visualization to identify whether this is a network latency issue, a processing delay, or a clock synchronization problem?

30. How do you configure Grafana to enable trace-to-metrics and metrics-to-trace navigation, and what are the requirements on the Prometheus and Tempo sides to make this work?

31. Describe how you would implement a Grafana dashboard that shows the RED (Rate, Errors, Duration) metrics for each service in a distributed trace, derived from Tempo's TraceQL metrics.

32. A Tempo datasource in Grafana is returning incomplete traces — some spans are missing. How do you diagnose whether this is a sampling issue, an ingestion issue, or a query issue in Tempo?

33. How do you use Grafana's Service Graph panel with Tempo to visualize service dependencies and identify cascading failures during an incident?

---

## Mimir Integration

34. You've migrated from Prometheus to Grafana Mimir for long-term metrics storage. Some Grafana dashboards are showing data gaps at the boundary between Prometheus local storage and Mimir. How do you diagnose and resolve this?

35. Describe how you configure Grafana to query Mimir in a multi-tenant environment where different teams should only see their own metrics, using Mimir's tenant isolation features.

36. A Grafana dashboard querying Mimir is significantly slower than the same dashboard querying Prometheus directly. What are the architectural reasons for this, and how do you optimize Mimir query performance?

37. How do you implement a Grafana datasource configuration that automatically routes short-range queries to Prometheus (for freshness) and long-range queries to Mimir (for historical data)?

38. Explain how Grafana's query splitting feature interacts with Mimir's query-frontend. In what scenarios does query splitting improve performance, and when does it cause issues?

---

## RBAC & Multi-Tenancy

39. Your organization has 50 teams using a shared Grafana instance. A team accidentally modified a shared infrastructure dashboard. How do you implement RBAC to prevent this while still allowing teams to view shared dashboards?

40. Describe how you would implement Grafana's role-based access control to support a scenario where: SREs can edit all dashboards, developers can view dashboards in their team's folder, and executives can only view a specific executive summary dashboard.

41. You need to implement data source permissions in Grafana so that the production Prometheus datasource is only accessible to SRE team members, while developers can only query the staging datasource.

42. How do you implement Grafana multi-tenancy using organizations versus folders, and what are the trade-offs in terms of data isolation, user management, and operational overhead?

43. Describe how you would integrate Grafana with an enterprise SSO provider (Okta/Azure AD) and map LDAP/SAML groups to Grafana roles, including handling role synchronization when group memberships change.

44. A Grafana user is able to see metrics from a datasource they shouldn't have access to by crafting direct API calls. How do you audit this security issue and implement proper data source-level access controls?

45. How do you implement Grafana's team-based dashboard permissions in a way that allows teams to have private dashboards while also contributing to shared organizational dashboards?

---

## Datasource Optimization & Performance

46. A Grafana instance with 300 active users is causing excessive load on the backend Prometheus instance. How do you implement query caching, rate limiting, and query optimization to reduce the impact?

47. You have multiple Prometheus instances (one per Kubernetes cluster) as datasources in Grafana. How do you implement a unified view across all clusters without creating a separate federation layer?

48. Describe how you would implement Grafana's mixed datasource feature to create a dashboard that combines metrics from Prometheus, logs from Elasticsearch, and traces from Jaeger in a single view.

49. A Grafana dashboard is making 150 individual queries to Prometheus for each page load. How do you refactor the dashboard to reduce query count using recording rules, query grouping, and panel optimization?

50. How do you implement Grafana's query inspector to diagnose slow queries, and what information does it provide that helps you optimize both the Grafana query and the backend datasource?

---

## Incident Visualization & Real-Time Troubleshooting

51. During a production incident, your Grafana dashboards are also slow because the Prometheus backend is under stress. How do you design your monitoring architecture to ensure dashboards remain available during incidents?

52. Describe how you would build a Grafana "incident war room" dashboard that automatically populates with relevant metrics, logs, and traces when a PagerDuty incident is created.

53. You need to implement a Grafana dashboard that shows the blast radius of a failing service — which other services are affected, what their error rates are, and what the user impact is. How do you design this?

54. How do you use Grafana's annotations feature to mark deployment events, incidents, and configuration changes on dashboards, and how do you automate annotation creation from your CI/CD pipeline?

55. Describe your approach to implementing a Grafana SLO dashboard that shows error budget burn rate, remaining error budget, and projected SLO breach time in real-time.

56. A Grafana dashboard shows a metric spike at 2 AM every night. How do you use Grafana's features to investigate whether this is a scheduled job, a cron-based process, or an external attack pattern?

57. How do you implement Grafana's state timeline panel to visualize the health state transitions of 100 services over a 30-day period for a reliability review?

58. Describe how you would use Grafana's Geomap panel to visualize the geographic distribution of errors and latency across a globally distributed application.

---

## Grafana as Code & Automation

59. Your organization wants to manage all Grafana dashboards as code using GitOps. Describe your implementation using Grafonnet, Jsonnet, or Terraform, including the CI/CD pipeline for dashboard deployment.

60. How do you implement automated dashboard testing to ensure that dashboard changes don't break existing panels or introduce query errors before they reach production?

61. Describe how you would use the Grafana HTTP API to programmatically create dashboards, manage users, and configure datasources as part of a new environment provisioning workflow.

62. You need to migrate 200 dashboards from one Grafana instance to another with different datasource UIDs. How do you automate this migration while updating all datasource references?

63. How do you implement Grafana dashboard snapshots as part of your incident postmortem process, ensuring that the state of dashboards during an incident is preserved for later analysis?

---

## Kubernetes & Cloud Integration

64. How do you deploy Grafana on Kubernetes with high availability, persistent storage for dashboards, and automatic datasource provisioning using ConfigMaps?

65. Describe how you would implement Grafana's Kubernetes app plugin to provide a unified view of Kubernetes cluster health, including node status, pod metrics, and namespace-level resource utilization.

66. You're using Grafana Cloud for a multi-cloud environment spanning AWS, GCP, and Azure. How do you implement a unified observability view that normalizes metric names and labels across cloud providers?

67. How do you configure Grafana to automatically provision dashboards and datasources when a new Kubernetes namespace is created, using the Grafana operator or sidecar provisioning?

68. Describe how you would implement Grafana alerting for Kubernetes pod disruption events, correlating pod evictions with node resource pressure metrics and application error rates.

---

## Advanced Features & Production Scenarios

69. A Grafana plugin used by 50 dashboards has a critical security vulnerability. How do you manage the plugin update process without breaking existing dashboards or causing downtime?

70. Describe how you would implement Grafana's reporting feature to automatically generate and distribute weekly SLO reports to stakeholders, including PDF generation and email delivery.

71. How do you use Grafana's transformation feature to perform client-side data manipulation, and in what scenarios is this preferable to server-side query transformations?

72. You need to implement a Grafana dashboard that shows cost attribution for cloud resources, correlating infrastructure metrics with billing data from AWS Cost Explorer. How do you approach this?

73. Describe how you would implement Grafana's playlist feature for a network operations center (NOC) display, rotating through critical infrastructure dashboards with appropriate refresh intervals.

74. How do you implement Grafana's public dashboards feature securely for sharing operational metrics with external stakeholders without exposing sensitive infrastructure data?

75. A Grafana instance is running out of database storage because dashboard versions are accumulating. How do you implement a dashboard version retention policy and clean up historical versions without losing current dashboards?

76. Describe how you would implement a Grafana-based chaos engineering dashboard that shows the impact of chaos experiments on service reliability metrics in real-time.

77. How do you implement Grafana's alerting contact points to integrate with a custom incident management system via webhooks, including retry logic and failure handling?

78. You're implementing Grafana for a financial services company with strict compliance requirements. How do you implement audit logging, data retention policies, and access controls to meet SOC 2 and PCI-DSS requirements?

---

## Basic Questions

1. What is Grafana and what is its primary use case in a monitoring stack?
2. What is a Grafana datasource and how do you add one?
3. What is the difference between a Grafana panel and a dashboard?
4. How do you create a new dashboard in Grafana?
5. What is a Grafana variable and how does it make dashboards dynamic?
6. What is the purpose of the time range picker in Grafana dashboards?
7. How do you share a Grafana dashboard with a team member?
8. What is Grafana Loki and how does it differ from Prometheus?
9. What is the default port Grafana listens on?
10. How do you add a Prometheus datasource to Grafana?
11. What is a Grafana alert and how does it differ from a Prometheus alert?
12. What is the purpose of Grafana's Explore view?
13. How do you create a simple time series panel in Grafana?
14. What is a Grafana annotation and when would you use it?
15. What is the difference between Grafana's bar chart and histogram panel types?
16. How do you configure a Grafana dashboard to auto-refresh every 30 seconds?
17. What is Grafana's stat panel used for?
18. How do you import a pre-built dashboard from Grafana.com?
19. What is the purpose of Grafana's table panel?
20. How do you configure a threshold in a Grafana panel to change color based on value?
21. What is Grafana's alert state history and how do you access it?
22. What is the difference between Grafana OSS and Grafana Enterprise?
23. How do you create a Grafana folder to organize dashboards?
24. What is the purpose of Grafana's dashboard links feature?
25. How do you configure Grafana to use SMTP for email alert notifications?
26. What is a Grafana playlist and when would you use it?
27. How do you duplicate a panel in a Grafana dashboard?
28. What is the purpose of Grafana's `$__interval` variable?
29. How do you configure Grafana's data source health check?
30. What is Grafana Tempo and what type of data does it store?
31. How do you create a Grafana alert rule based on a Prometheus query?
32. What is the purpose of the Grafana `$__timeFilter` variable?
33. How do you configure Grafana's organization settings?
34. What is a Grafana contact point and how does it relate to alerting?
35. How do you use Grafana's query inspector to debug a slow panel?
36. What is the purpose of Grafana's `$__range` variable?
37. How do you configure Grafana to display data from multiple datasources in a single panel?
38. What is Grafana Mimir and how does it extend Prometheus?
39. How do you configure Grafana's user authentication using local accounts?
40. What is the purpose of Grafana's dashboard version history?
41. How do you configure a Grafana panel to show the last known value when data is missing?
42. What is the difference between Grafana's `rate` and `irate` functions when used with Prometheus datasource?
43. How do you configure Grafana's notification policies?
44. What is the purpose of Grafana's transformation feature?
45. How do you export a Grafana dashboard as JSON for backup?

---

## Intermediate Questions

1. How do you implement Grafana dashboard templating with chained variables where selecting a cluster filters the available namespace options?
2. Describe how you would configure Grafana's unified alerting to route alerts to different teams based on label matchers.
3. How do you implement Grafana's RBAC to allow developers to view dashboards but not edit datasources or alert rules?
4. Explain how Grafana's query caching works and how you would configure it to reduce load on a Prometheus backend.
5. How do you implement a Grafana dashboard that shows SLO compliance with error budget burn rate using Prometheus recording rules?
6. Describe how you would configure Grafana to correlate Loki logs with Prometheus metrics using derived fields and data links.
7. How do you implement Grafana's mixed datasource feature to combine metrics from Prometheus and logs from Elasticsearch in a single panel?
8. Explain how Grafana's alert evaluation works and how the evaluation interval affects alert detection latency.
9. How do you implement Grafana's state timeline panel to visualize service health state transitions over time?
10. Describe how you would configure Grafana's Tempo datasource to enable trace-to-metrics and metrics-to-trace navigation.
11. How do you implement Grafana's dashboard provisioning using ConfigMaps in a Kubernetes deployment?
12. Explain how Grafana's `$__rate_interval` variable differs from `$__interval` and when each should be used.
13. How do you implement Grafana's alerting silence rules programmatically using the Grafana API?
14. Describe how you would configure Grafana's LDAP integration for enterprise SSO authentication.
15. How do you implement Grafana's multi-value variable to filter metrics across multiple services simultaneously?
16. Explain how Grafana's Loki datasource handles log volume queries differently from log content queries.
17. How do you implement Grafana's service graph panel to visualize microservice dependencies from Tempo trace data?
18. Describe how you would configure Grafana's alert contact points for PagerDuty integration with proper severity mapping.
19. How do you implement Grafana's dashboard as code using Jsonnet and Grafonnet?
20. Explain how Grafana's query splitting feature works and when it improves performance for long time range queries.
21. How do you implement Grafana's data source permissions to restrict access to production metrics for specific teams?
22. Describe how you would configure Grafana's Mimir datasource for multi-tenant metric queries.
23. How do you implement Grafana's exemplar support to navigate from metrics to traces?
24. Explain how Grafana's alert state machine works, including the pending, firing, and resolved states.
25. How do you implement Grafana's reporting feature to generate scheduled PDF reports of key dashboards?
26. Describe how you would configure Grafana's Kubernetes monitoring using the Kubernetes app plugin.
27. How do you implement Grafana's public dashboards feature for sharing metrics with external stakeholders?
28. Explain how Grafana's transformation pipeline works and provide three examples of useful transformations.
29. How do you implement Grafana's alert notification templates to customize the content of alert messages?
30. Describe how you would configure Grafana's high availability deployment with a shared database backend.
31. How do you implement Grafana's dashboard search and tagging to organize hundreds of dashboards?
32. Explain how Grafana's Loki label browser works and how you use it to explore log streams.
33. How do you implement Grafana's canvas panel for custom operational status boards?
34. Describe how you would configure Grafana's Azure Monitor datasource for multi-subscription monitoring.
35. How do you implement Grafana's alert grouping to reduce notification noise during widespread incidents?
36. Explain how Grafana's Flux query language differs from InfluxQL when querying InfluxDB datasources.
37. How do you implement Grafana's heatmap panel to visualize latency distribution over time?
38. Describe how you would configure Grafana's Google Cloud Monitoring datasource for GKE cluster monitoring.
39. How do you implement Grafana's variable refresh options to automatically update variable values when the time range changes?
40. Explain how Grafana's Prometheus datasource handles recording rules differently from raw metric queries.

---

## Advanced Questions

1. Design a Grafana architecture for a 500-team enterprise with isolated spaces, centralized datasource management, SSO integration, and automated dashboard provisioning via GitOps.
2. How do you implement a Grafana-based observability platform that provides unified visibility across metrics (Mimir), logs (Loki), and traces (Tempo) with automatic correlation for incident investigation?
3. Describe how you would implement Grafana's multi-tenancy at scale for a managed service provider with 200 customers, each requiring complete data isolation and custom branding.
4. How do you implement a Grafana deployment that maintains dashboard availability during Prometheus backend failures, including fallback datasources and cached data display?
5. Describe how you would implement a Grafana-based SRE platform that automatically generates SLO dashboards, error budget burn rate alerts, and incident correlation views for 500 services.
6. How do you implement Grafana's alerting at enterprise scale with 10,000 alert rules, ensuring sub-minute evaluation latency and reliable notification delivery?
7. Describe how you would implement a Grafana deployment for a financial services company with strict data residency requirements, ensuring that metrics from different regions never cross geographic boundaries.
8. How do you implement a Grafana-based incident management platform that automatically creates war room dashboards, correlates relevant metrics, and tracks incident timeline annotations?
9. Describe how you would implement Grafana's RBAC model for a complex enterprise with nested team hierarchies, shared infrastructure dashboards, and team-specific application dashboards.
10. How do you implement a Grafana deployment that handles 10,000 concurrent dashboard users without degrading query performance or causing backend overload?
11. Describe how you would implement a Grafana-based cost observability platform that correlates infrastructure metrics with cloud billing data to provide per-service cost attribution.
12. How do you implement Grafana's alerting for a Kubernetes multi-cluster environment where the same alert rules need to be applied consistently across 50 clusters with cluster-specific routing?
13. Describe how you would implement a Grafana deployment that automatically provisions dashboards, datasources, and alert rules when a new microservice is deployed via CI/CD.
14. How do you implement Grafana's Loki integration at scale for a platform generating 10TB of logs per day, ensuring fast log search and correlation with metrics?
15. Describe how you would implement a Grafana-based chaos engineering observability platform that visualizes the impact of chaos experiments on service reliability in real-time.
16. How do you implement Grafana's security hardening for a production deployment, including TLS termination, secret management, audit logging, and vulnerability scanning?
17. Describe how you would implement a Grafana deployment that provides real-time business intelligence dashboards combining operational metrics with business KPIs from multiple data sources.
18. How do you implement Grafana's plugin management at enterprise scale, including plugin security review, version pinning, and automated updates across 50 Grafana instances?
19. Describe how you would implement a Grafana-based capacity planning platform that uses historical metrics to forecast resource requirements and generate automated scaling recommendations.
20. How do you implement Grafana's disaster recovery strategy, ensuring that all dashboards, alert rules, and configurations can be restored within 30 minutes of a complete Grafana instance failure?
21. Describe how you would implement a Grafana deployment that supports A/B testing of dashboard designs, measuring which dashboard layouts lead to faster incident resolution times.
22. How do you implement Grafana's integration with a service catalog to automatically link dashboards to service ownership information, runbooks, and on-call schedules?
23. Describe how you would implement a Grafana-based compliance monitoring platform for a healthcare organization, ensuring HIPAA-compliant data access controls and audit trails.
24. How do you implement Grafana's Tempo integration for a distributed tracing platform handling 1 billion spans per day, ensuring fast trace search and correlation with logs and metrics?
25. Describe how you would implement a Grafana deployment that provides executive-level business dashboards with automatic data freshness indicators and SLA compliance summaries.

---

## Rapid-Fire Questions

1. What is the default port for Grafana?
2. What file format does Grafana use to export dashboards?
3. What is the purpose of Grafana's `$__from` and `$__to` variables?
4. What does the Grafana `stat` panel display?
5. What is the difference between Grafana's `alert` and `recording rule`?
6. How do you reset a Grafana admin password from the command line?
7. What is the purpose of Grafana's `mixed` datasource?
8. What does the Grafana `gauge` panel type display?
9. What is Grafana's `provisioning` directory used for?
10. What is the difference between Grafana OSS and Grafana Cloud?
11. What does `$__interval` represent in a Grafana query?
12. How do you enable Grafana's anonymous access for public dashboards?
13. What is the purpose of Grafana's `overrides` in panel configuration?
14. What does the Grafana `news` panel display?
15. What is the purpose of Grafana's `repeat` feature for panels?
16. How do you configure Grafana's session timeout?
17. What is the difference between Grafana's `bar gauge` and `gauge` panel types?
18. What does Grafana's `Explore` mode allow you to do?
19. What is the purpose of Grafana's `link` panel type?
20. How do you configure Grafana to use a PostgreSQL database instead of SQLite?
21. What is the purpose of Grafana's `fieldConfig` in panel JSON?
22. What does the Grafana `logs` panel display?
23. What is the purpose of Grafana's `threshold` configuration in panels?
24. How do you configure Grafana's `smtp` settings for email notifications?
25. What is the difference between Grafana's `contact point` and `notification policy`?
26. What does Grafana's `flame graph` panel visualize?
27. What is the purpose of Grafana's `dashboard uid`?
28. How do you configure Grafana's `rendering` service for PDF report generation?
29. What is the purpose of Grafana's `alerting` folder in provisioning?
30. What does the Grafana `traces` panel display?
31. What is the purpose of Grafana's `correlations` feature?
32. How do you configure Grafana's `cookie_samesite` setting?
33. What is the difference between Grafana's `alert rule` and `alert instance`?
34. What does Grafana's `geomap` panel display?
35. What is the purpose of Grafana's `data source uid` in dashboard JSON?
36. How do you configure Grafana's `max_open_files` setting?
37. What is the purpose of Grafana's `dashboard tags`?
38. What does Grafana's `xy chart` panel display?
39. What is the purpose of Grafana's `alert group` in unified alerting?
40. How do you configure Grafana's `allow_embedding` setting?
41. What is the difference between Grafana's `panel` and `row`?
42. What does Grafana's `histogram` panel display?
43. What is the purpose of Grafana's `snapshot` feature?
44. How do you configure Grafana's `auto_assign_org_role` setting?
45. What is the purpose of Grafana's `alert evaluation group`?
46. What does Grafana's `candlestick` panel display?
47. What is the purpose of Grafana's `library panel`?
48. How do you configure Grafana's `login_maximum_inactive_lifetime_duration`?
49. What is the difference between Grafana's `folder` and `general` dashboard location?
50. What does Grafana's `trend` panel display?
