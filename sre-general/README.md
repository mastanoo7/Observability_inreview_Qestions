# SRE General — Advanced Interview Questions

> **Level:** Senior / Staff SRE | Platform Engineer | Reliability Engineer
> **Focus:** Real-time production scenarios, incident management, reliability engineering, and architecture

---

## SLI / SLO / SLA

1. Your service has a 99.9% availability SLO. During a 30-day month, you've already consumed 80% of your error budget by day 20. How do you communicate this to stakeholders, adjust your deployment velocity, and implement short-term reliability improvements?

2. Describe how you would define meaningful SLIs for a data pipeline service where "availability" is not straightforward — the service processes batch jobs and the user impact of a failure depends on which job fails and when.

3. A business stakeholder is demanding a 99.999% SLA for a non-critical internal tool. How do you explain the cost implications of five-nines availability, and how do you negotiate a more appropriate SLO based on actual user impact?

4. How do you implement SLO-based alerting using burn rate alerts? Explain the multi-window, multi-burn-rate approach and why it's superior to simple threshold-based availability alerts.

5. Your service has separate SLOs for availability (99.9%) and latency (P99 < 500ms). During an incident, availability is within SLO but latency is breaching. How do you calculate the combined error budget impact and communicate it?

6. Describe how you would implement SLO measurement for a service that has both synchronous API calls and asynchronous background processing, where failures in background processing have delayed user impact.

7. How do you handle SLO measurement during planned maintenance windows? Should maintenance downtime count against the error budget, and how do you implement this policy technically?

8. A microservice team wants to set their own SLOs independently. How do you implement a governance process that ensures SLOs are meaningful, measurable, and aligned with user expectations without becoming a bureaucratic bottleneck?

---

## Incident Management & Response

9. You're the incident commander for a P1 outage affecting 50% of users. The root cause is unknown, and you have 10 engineers available. How do you structure the incident response, assign roles, and manage communication simultaneously?

10. During a major incident, your primary monitoring system (Prometheus) is also affected by the outage. How do you maintain situational awareness and coordinate the response without your primary observability tools?

11. Describe how you would implement an on-call rotation for a global team across 4 time zones, including handoff procedures, escalation paths, and preventing on-call burnout.

12. A production incident was resolved, but 3 hours later the same issue recurred because the fix was incomplete. How do you implement incident follow-through processes to prevent recurrence?

13. How do you implement a severity classification system for incidents that accurately reflects user impact, business impact, and urgency, and how do you train teams to correctly classify incidents under pressure?

14. Describe your approach to managing a "slow burn" incident — one where the system is degraded but not completely down, and the degradation is gradually worsening over hours.

15. How do you implement automated incident detection that correlates signals from multiple monitoring systems to generate a single, actionable alert rather than hundreds of individual notifications?

---

## Error Budgets

16. Your team has been consistently burning through the error budget in the first week of each month due to aggressive deployments. How do you implement an error budget policy that automatically adjusts deployment frequency based on remaining budget?

17. Describe how you would implement error budget tracking for a service that depends on three external third-party APIs, where failures in those APIs count against your SLO but are outside your control.

18. How do you handle the situation where a single large incident consumes the entire monthly error budget? What processes do you implement to prevent a single incident from permanently blocking deployments for the rest of the month?

19. Explain how you would implement error budget alerts at different burn rate thresholds (2%, 5%, 10% per hour) and what actions each threshold should trigger in terms of deployment freezes and escalations.

20. How do you implement error budget reporting for a quarterly business review, showing trends, incident contributions, and the correlation between deployment frequency and error budget consumption?

---

## Production Outages & Kubernetes Failures

21. A Kubernetes cluster's etcd cluster has lost quorum due to disk I/O saturation. The API server is unavailable. Walk through your recovery procedure, including how you restore etcd from backup and validate cluster state.

22. Your Kubernetes cluster is experiencing a "cascading failure" where pod evictions are causing more load on remaining pods, which triggers more evictions. How do you break the cascade and stabilize the cluster?

23. A Kubernetes node is in a "NotReady" state and pods are not being rescheduled because the node hasn't been evicted. How do you diagnose the node issue and implement a safe node eviction procedure?

24. Describe how you would diagnose and resolve a situation where Kubernetes DNS (CoreDNS) is intermittently failing, causing random service-to-service communication failures.

25. A Kubernetes deployment rollout is stuck because new pods are failing health checks, but the old pods are still running. How do you diagnose the health check failure and implement a safe rollback?

26. How do you implement Kubernetes pod disruption budgets (PDBs) to ensure that rolling updates and node maintenance don't cause service availability to drop below SLO thresholds?

---

## CI/CD Failures & Deployment Safety

27. A CI/CD pipeline deployed a configuration change that caused a production outage. The change passed all automated tests. How do you implement additional safeguards to prevent configuration-only changes from bypassing safety checks?

28. Describe how you would implement a progressive delivery strategy (canary deployments, feature flags) for a high-risk database schema migration that cannot be easily rolled back.

29. How do you implement automated rollback triggers in a CI/CD pipeline that detect production degradation within 5 minutes of a deployment and automatically revert to the previous version?

30. A deployment pipeline is taking 45 minutes end-to-end, causing deployment frequency to drop and increasing the risk of large, risky deployments. How do you optimize the pipeline while maintaining safety?

---

## Scaling Strategies

31. Your service is experiencing a "thundering herd" problem where all instances of a service simultaneously try to refresh a shared cache after expiration. How do you implement cache stampede prevention at scale?

32. Describe how you would implement auto-scaling for a service that has highly variable traffic patterns, including the selection of scaling metrics, scaling policies, and handling of scale-in events safely.

33. How do you implement database connection pooling and management for a microservices architecture where 500 service instances need to connect to a PostgreSQL database without overwhelming the connection limit?

34. A service is experiencing hot partition issues in a distributed database (DynamoDB/Cassandra) during traffic spikes. How do you diagnose the hot partition and implement a sharding strategy to distribute load?

35. Describe how you would implement a global load balancing strategy for a latency-sensitive application that needs to route users to the nearest healthy region while handling regional failures gracefully.

---

## Disaster Recovery

36. Your primary data center has experienced a complete power failure. Describe your DR activation procedure, including the decision criteria for failover, the steps to activate the DR site, and how you validate that the DR environment is serving traffic correctly.

37. How do you implement and regularly test a disaster recovery plan for a stateful application with a 1-hour RPO and 4-hour RTO, including the automation of failover procedures?

38. Describe how you would implement a multi-region active-active architecture for a database-backed application, including conflict resolution, data consistency, and failover procedures.

39. How do you implement chaos engineering practices to proactively test your disaster recovery capabilities, including the design of chaos experiments and the measurement of recovery metrics?

40. A DR test revealed that your recovery time is 8 hours instead of the target 4 hours. How do you systematically identify the bottlenecks in the recovery process and implement improvements?

---

## Observability Architecture

41. You're designing the observability architecture for a new microservices platform that will eventually have 500 services. How do you implement a scalable, cost-effective observability stack that provides metrics, logs, and traces without creating operational overhead for individual teams?

42. Describe how you would implement a unified observability platform that correlates metrics from Prometheus, logs from Elasticsearch, and traces from Jaeger, providing a single pane of glass for incident investigation.

43. How do you implement observability for a serverless architecture where traditional agent-based monitoring is not possible, and where the ephemeral nature of functions makes debugging challenging?

44. Describe your approach to implementing observability for a data pipeline that processes events through multiple stages, ensuring that you can track individual events through the pipeline and identify where failures or delays occur.

---

## Reliability Engineering & Postmortems

45. How do you implement a blameless postmortem process that genuinely identifies systemic issues rather than individual mistakes, and how do you ensure that action items from postmortems are actually completed?

46. Describe how you would implement a reliability review process for new features before they are deployed to production, including the criteria for reliability readiness and the process for waiving requirements.

47. How do you measure and improve the "mean time to detect" (MTTD) and "mean time to resolve" (MTTR) for your service, and what are the most impactful improvements you can make to each metric?

48. A service has had 5 incidents in the past month, all with different root causes. How do you identify systemic reliability issues versus random failures, and how do you prioritize reliability investments?

---

## Cloud-Native Troubleshooting

49. A cloud-native application is experiencing intermittent failures that only occur in production and cannot be reproduced in staging. How do you implement production debugging techniques that don't require code changes or service restarts?

50. Describe how you would implement a comprehensive health check strategy for a microservices architecture, including liveness probes, readiness probes, and startup probes, and how you tune these to avoid false positive failures.

51. How do you implement distributed rate limiting for a microservices architecture where rate limits need to be enforced globally across all instances of a service, not just per-instance?

52. A cloud provider's managed service (e.g., AWS RDS, Google Cloud SQL) is experiencing degraded performance. How do you diagnose whether the issue is in the managed service, your application's usage patterns, or the network between your application and the service?

53. Describe how you would implement a cost optimization strategy for a cloud-native observability stack that is generating $50,000/month in storage and compute costs, without reducing monitoring coverage for critical services.

54. How do you implement network observability for a service mesh (Istio/Linkerd) to diagnose intermittent connection failures, high latency, and circuit breaker activations between services?

55. A production service is experiencing memory leaks that cause it to be OOMKilled every 6 hours. The leak is not reproducible in development. How do you implement production memory profiling without impacting service performance?

---

## Basic Questions

1. What is Site Reliability Engineering (SRE) and how does it differ from traditional operations?
2. What is the difference between an SLI, SLO, and SLA?
3. What is an error budget and how is it calculated?
4. What is the purpose of a postmortem in SRE?
5. What is the difference between MTTR and MTTF?
6. What is toil in SRE and why is it important to reduce it?
7. What is the purpose of a runbook in incident management?
8. What is the difference between a P1 and P2 incident?
9. What is the purpose of on-call rotation in SRE?
10. What is a change freeze and when would you implement one?
11. What is the difference between availability and reliability?
12. What is a canary deployment and why is it used?
13. What is the purpose of a health check endpoint in a service?
14. What is the difference between a liveness probe and a readiness probe in Kubernetes?
15. What is a circuit breaker pattern and when would you use it?
16. What is the purpose of a load balancer in a distributed system?
17. What is the difference between horizontal and vertical scaling?
18. What is a blue-green deployment and what are its benefits?
19. What is the purpose of a feature flag in software deployment?
20. What is the difference between RTO and RPO in disaster recovery?
21. What is a chaos experiment and what is its purpose?
22. What is the purpose of a service mesh in a microservices architecture?
23. What is the difference between a hard dependency and a soft dependency?
24. What is the purpose of rate limiting in a distributed system?
25. What is the difference between synchronous and asynchronous communication in microservices?
26. What is a dead letter queue and when would you use it?
27. What is the purpose of a retry policy with exponential backoff?
28. What is the difference between a rolling update and a recreate deployment strategy?
29. What is the purpose of a pod disruption budget (PDB) in Kubernetes?
30. What is the difference between a Kubernetes deployment and a statefulset?
31. What is the purpose of resource requests and limits in Kubernetes?
32. What is a Kubernetes namespace and why is it used?
33. What is the purpose of a Kubernetes ConfigMap?
34. What is the difference between a Kubernetes Secret and a ConfigMap?
35. What is the purpose of a Kubernetes HorizontalPodAutoscaler (HPA)?
36. What is a Kubernetes node affinity and when would you use it?
37. What is the purpose of a Kubernetes PersistentVolume (PV)?
38. What is the difference between a Kubernetes Service of type ClusterIP, NodePort, and LoadBalancer?
39. What is the purpose of a Kubernetes Ingress resource?
40. What is the difference between a Kubernetes Job and a CronJob?
41. What is the purpose of a Kubernetes DaemonSet?
42. What is the difference between a Kubernetes rolling update and a blue-green deployment?
43. What is the purpose of a Kubernetes NetworkPolicy?
44. What is the difference between a Kubernetes StatefulSet and a Deployment?
45. What is the purpose of a Kubernetes ServiceAccount?

---

## Intermediate Questions

1. How do you implement multi-window, multi-burn-rate SLO alerting to detect both fast and slow error budget consumption?
2. Describe how you would implement an on-call escalation policy that balances response time requirements with engineer well-being.
3. How do you implement a blameless postmortem process that identifies systemic issues rather than individual mistakes?
4. Explain how you would design a reliability review process for new features before production deployment.
5. How do you implement error budget policies that automatically adjust deployment velocity based on remaining budget?
6. Describe how you would implement chaos engineering for a Kubernetes-based microservices platform, including experiment design and safety mechanisms.
7. How do you implement a progressive delivery strategy using feature flags and canary deployments for a high-risk database migration?
8. Explain how you would design a disaster recovery plan with 1-hour RPO and 4-hour RTO for a stateful application.
9. How do you implement automated rollback triggers in a CI/CD pipeline based on production error rate metrics?
10. Describe how you would implement a capacity planning process for a service expecting 10x traffic growth over 12 months.
11. How do you implement Kubernetes pod disruption budgets to ensure rolling updates don't breach SLO thresholds?
12. Explain how you would design a multi-region active-active architecture for a latency-sensitive application.
13. How do you implement a service dependency map to identify blast radius during incident response?
14. Describe how you would implement a toil reduction program that measures and systematically eliminates operational toil.
15. How do you implement a production readiness review (PRR) checklist for new services joining the platform?
16. Explain how you would design a Kubernetes cluster autoscaling strategy that balances cost efficiency with performance.
17. How do you implement a database connection pooling strategy for a microservices architecture with 500 service instances?
18. Describe how you would implement a distributed rate limiting system that enforces global rate limits across all service instances.
19. How do you implement a Kubernetes node drain procedure that ensures zero downtime during node maintenance?
20. Explain how you would design a CI/CD pipeline that includes automated reliability testing before production deployment.
21. How do you implement a service mesh (Istio) for traffic management, including canary routing and circuit breaking?
22. Describe how you would implement a Kubernetes cluster upgrade strategy for a production cluster with zero downtime.
23. How do you implement a secrets management strategy for a Kubernetes-based microservices platform?
24. Explain how you would design a multi-cloud failover strategy that can switch traffic between cloud providers within 5 minutes.
25. How do you implement a Kubernetes resource quota strategy for a multi-tenant cluster with 50 teams?
26. Describe how you would implement a production debugging strategy for intermittent failures that cannot be reproduced in staging.
27. How do you implement a Kubernetes network policy strategy for a zero-trust microservices architecture?
28. Explain how you would design an incident severity classification system that accurately reflects user impact.
29. How do you implement a deployment pipeline that automatically validates infrastructure changes before applying them to production?
30. Describe how you would implement a Kubernetes storage strategy for stateful applications with different performance requirements.
31. How do you implement a service catalog that provides self-service infrastructure provisioning for development teams?
32. Explain how you would design a log aggregation strategy for a 1,000-node Kubernetes cluster.
33. How do you implement a Kubernetes admission controller for enforcing security and reliability policies?
34. Describe how you would implement a distributed tracing strategy for a microservices architecture with 200 services.
35. How do you implement a Kubernetes cluster federation strategy for a multi-region deployment?
36. Explain how you would design a backup and restore strategy for a Kubernetes-based stateful application.
37. How do you implement a Kubernetes pod security policy (or Pod Security Admission) for a multi-tenant cluster?
38. Describe how you would implement a service mesh observability strategy for monitoring mTLS, retries, and circuit breakers.
39. How do you implement a Kubernetes custom resource definition (CRD) for managing application-specific operational concerns?
40. Explain how you would design a cost optimization strategy for a Kubernetes cluster running on cloud infrastructure.

---

## Advanced Questions

1. Design an SRE organization structure for a 500-engineer company with 200 microservices, including team topologies, SLO ownership, and escalation paths.
2. How do you implement a reliability engineering platform that automatically measures, tracks, and reports SLO compliance for 500 services across multiple teams?
3. Describe how you would implement a global incident management system that coordinates response across 10 time zones with automated escalation and war room creation.
4. How do you implement a chaos engineering program at enterprise scale, including experiment design, safety controls, blast radius management, and organizational buy-in?
5. Describe how you would design a multi-region active-active architecture for a financial services application with strict consistency requirements and sub-100ms latency.
6. How do you implement a zero-downtime database migration strategy for a PostgreSQL database with 10TB of data and 10,000 concurrent connections?
7. Describe how you would implement a Kubernetes multi-cluster management platform for 100 clusters across 5 cloud providers, with unified policy enforcement and observability.
8. How do you implement a production debugging platform that enables engineers to safely debug production issues without impacting service availability?
9. Describe how you would implement a reliability-as-code framework that allows teams to define, test, and enforce reliability requirements in their CI/CD pipelines.
10. How do you implement a capacity planning platform that uses machine learning to predict resource requirements and automatically provision infrastructure ahead of demand?
11. Describe how you would implement a disaster recovery automation platform that can execute complex multi-step recovery procedures without human intervention.
12. How do you implement a service mesh at scale for a 10,000-pod Kubernetes environment, including performance optimization and observability?
13. Describe how you would implement a progressive delivery platform that supports canary deployments, A/B testing, and feature flags with automatic rollback based on SLO metrics.
14. How do you implement a cost optimization platform for a cloud-native infrastructure that automatically identifies and eliminates waste while maintaining reliability targets?
15. Describe how you would implement a security reliability engineering program that integrates security requirements into SLO definitions and incident response procedures.
16. How do you implement a platform engineering team's internal developer platform (IDP) that provides self-service infrastructure provisioning with built-in reliability guardrails?
17. Describe how you would implement a reliability testing framework that automatically generates and executes reliability tests based on service dependency graphs.
18. How do you implement a Kubernetes operator for managing the complete lifecycle of a complex stateful application, including upgrades, backups, and failure recovery?
19. Describe how you would implement a global load balancing strategy for a latency-sensitive application with users in 50 countries and strict data residency requirements.
20. How do you implement a production readiness framework that automatically evaluates new services against reliability, security, and operational standards before allowing production deployment?
21. Describe how you would implement a postmortem-driven reliability improvement program that tracks action item completion and measures the impact of reliability investments.
22. How do you implement a Kubernetes cluster autoscaling strategy that can scale from 100 to 10,000 nodes within 10 minutes in response to traffic spikes?
23. Describe how you would implement a distributed systems debugging platform that provides causal analysis of complex failure scenarios involving multiple services and infrastructure components.
24. How do you implement a reliability SLA management system for a B2B SaaS platform with different SLA tiers for different customers?
25. Describe how you would implement a GitOps-based infrastructure management platform that ensures all infrastructure changes are reviewed, tested, and audited before deployment.

---

## Rapid-Fire Questions

1. What does SRE stand for?
2. What is the formula for calculating error budget?
3. What is the difference between MTTR and MTTD?
4. What does SLI stand for?
5. What is the purpose of a postmortem action item?
6. What is toil in SRE?
7. What does RTO stand for in disaster recovery?
8. What is the purpose of a circuit breaker pattern?
9. What does RPO stand for in disaster recovery?
10. What is the difference between availability and reliability?
11. What is a canary deployment?
12. What does MTTF stand for?
13. What is the purpose of a feature flag?
14. What is a blue-green deployment?
15. What does P99 latency mean?
16. What is the purpose of a health check endpoint?
17. What is a rolling deployment?
18. What does CRD stand for in Kubernetes?
19. What is the purpose of a Kubernetes liveness probe?
20. What is a Kubernetes readiness probe?
21. What does HPA stand for in Kubernetes?
22. What is the purpose of a Kubernetes PodDisruptionBudget?
23. What is a Kubernetes DaemonSet used for?
24. What does PVC stand for in Kubernetes?
25. What is the purpose of a Kubernetes NetworkPolicy?
26. What is a Kubernetes StatefulSet used for?
27. What does RBAC stand for in Kubernetes?
28. What is the purpose of a Kubernetes ServiceAccount?
29. What is a Kubernetes Ingress resource?
30. What does CNI stand for in Kubernetes?
31. What is the purpose of a Kubernetes ConfigMap?
32. What is a Kubernetes Secret used for?
33. What does OOM stand for in Kubernetes?
34. What is the purpose of Kubernetes resource requests?
35. What is a Kubernetes resource limit?
36. What does VPA stand for in Kubernetes?
37. What is the purpose of a Kubernetes node taint?
38. What is a Kubernetes node affinity?
39. What does CSI stand for in Kubernetes?
40. What is the purpose of a Kubernetes admission webhook?
41. What is a Kubernetes operator?
42. What does CRD stand for?
43. What is the purpose of a Kubernetes namespace?
44. What is a Kubernetes pod priority class?
45. What does QoS stand for in Kubernetes pod scheduling?
46. What is the purpose of a Kubernetes horizontal pod autoscaler?
47. What is a Kubernetes cluster autoscaler?
48. What does ETCD store in a Kubernetes cluster?
49. What is the purpose of a Kubernetes API server?
50. What is a Kubernetes controller manager responsible for?
