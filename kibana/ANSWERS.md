# Kibana — Interview Answers

> Companion answers for [README.md](./README.md). Structure mirrors README sections.

---

## Dashboard Troubleshooting & Design

### 1. Dashboard showing "No results found" — systematic debugging

- **Confirm data exists:** Run the same query in Dev Tools or Discover against the backing indices; verify document counts and time range of actual data.
- **Time filter:** Check dashboard global time picker (absolute vs. relative, timezone, "now" drift); a common cause is "Last 15 minutes" when data is historical or delayed.
- **Index pattern / data view:** Validate the pattern matches current index names (`logs-*` vs. `logs-2025.05.17`); refresh field list and confirm the default data view is correct for each panel.
- **Filters and drilldowns:** Inspect pinned filters, hidden dashboard filters, and space-level defaults that may exclude all documents.
- **Field mapping:** Ensure aggregated fields are `keyword` not analyzed `text`; missing or wrong types yield empty aggregations.
- **Kibana saved object references:** Broken links after import/migration can point panels at deleted data views—re-link in panel editor.
- **Kibana server logs:** Check `logging.json` / server logs for query errors, license issues, or security exceptions (`kibana.log`, Elasticsearch response bodies in debug mode).
- **Elasticsearch permissions:** Confirm the Kibana system user and end-user roles can read the target indices; security errors sometimes surface as empty results in UI.

### 2. Dashboard with 30 visualizations taking 3 minutes to load

- **Per-panel timing:** Open Inspector on each visualization; sort by query duration to find the slowest 5–10 panels.
- **Elasticsearch vs. Kibana:** If ES `took` is high, optimize queries (filters, smaller time range, `keyword` aggregations, avoid high-cardinality `terms`); if ES is fast but UI slow, issue is rendering or too many round trips.
- **Consolidate queries:** Merge related metrics into one Lens visualization or use TSVB with multiple series instead of 30 independent searches.
- **Reduce time range defaults:** NOC dashboards rarely need 30 days on load; default to 1–4 hours with optional expansion.
- **Sampling and precision:** Lower `terms` size only where appropriate; use `composite` aggregations for high-cardinality tables.
- **Caching:** Enable Elasticsearch request cache where safe; use transform/summary indices for heavy historical panels.
- **Lazy load / layout:** Split into linked dashboards by domain; load overview first, detail on drilldown.
- **Infrastructure:** Scale Kibana nodes and ensure coordinating nodes aren't saturated; check `search.max_buckets` and circuit breakers.

### 3. NOC dashboard for 500 microservices

- **Hierarchy, not flat list:** Top row = global health (SLO status, open alerts, error budget burn); second row = tier-1 / business-critical services only.
- **Traffic-light semantics:** Green/yellow/red by SLO or alert state; limit to ~20–30 cells operators can scan in seconds.
- **Top-N error sources:** Lens or Data Table showing top 10 services by error rate or log level, not all 500.
- **Alert summary panel:** Count by severity with link to Cases or alert detail; suppress noise via grouping rules upstream.
- **Real-time with guardrails:** 30–60s auto-refresh on summary panels only; heavier analytics on separate drilldown dashboard.
- **Service catalog integration:** Enrich with ownership, tier, and runbook links via runtime fields or external lookups.
- **Consistent time and filters:** Global time picker + optional environment filter (`prod`); document standard layout in runbooks.

### 4. Incorrect data after Elasticsearch mapping change

- **Identify changed fields:** Compare current mapping in Index Management vs. what dashboards expect (type, `keyword` subfield, format).
- **Refresh data view:** Stack Management → Data Views → Refresh field list; review conflicts (e.g., `text` vs. `keyword`).
- **Reconfigure formatters:** Set field formatters (bytes, duration, URL) on the data view for fields showing raw values.
- **Update visualizations:** Re-select aggregation fields using `.keyword` suffix where dynamic mapping flipped.
- **Runtime fields:** Use runtime fields as a bridge if reindex isn't immediate.
- **Communicate breaking changes:** Treat mapping changes like API changes—version data views and run dashboard smoke tests in staging.
- **Reindex if needed:** For irreconcilable type changes, reindex to a new index with correct mapping and alias cutover.

### 5. Dashboard version control and change management

- **Saved objects in Git:** Export dashboards, visualizations, data views via `saved_objects` API or `kibana.savedObjects` NDJSON; store in version control.
- **CI/CD pipeline:** PR review for dashboard changes; import to staging, run automated screenshot or query validation, then promote to prod.
- **Ownership tags:** Use Kibana tags and CODEOWNERS in repo to map dashboards to teams.
- **Promotion workflow:** Dev → staging → prod spaces with restricted prod write access; platform team approves shared NOC boards.
- **Change audit:** Enable audit logging for saved object updates; optional Slack notification on prod imports.
- **Rollback:** Keep previous NDJSON export per release; `saved_objects` import with `overwrite` controlled by pipeline.

### 6. Auto-refresh every 30 seconds without overloading Elasticsearch

- **Scope refresh:** Enable auto-refresh only on lightweight summary dashboards, not 30-panel analytics boards.
- **Align panel queries:** Shared time filter and identical time ranges allow ES request cache reuse.
- **Short lookback windows:** Query last 5–15 minutes of data, not full 24h, on refreshing panels.
- **Pre-aggregate heavy metrics:** Use Metricbeat rollups, transforms, or recording indices for refresh-friendly metrics.
- **Stagger refreshes:** Offset panel refresh timing slightly if using Canvas or custom polling to avoid query thundering herds.
- **Monitor load:** Track `search` thread pool and Kibana response times; back off refresh interval during incidents.
- **Use alerting for detection:** Let alerts evaluate on schedule server-side instead of polling large Discover queries from many browsers.

### 7. Lens aggregation doesn't match raw document count

- **Check aggregation type:** `unique count` (cardinality) ≠ document count; `sum` of a field ≠ doc count unless one doc per event.
- **Sampling:** High-cardinality `terms` may use `shard_size` and approximate counts—compare with `composite` aggregation or `value_count` in Dev Tools.
- **Filters:** Panel-level filters vs. dashboard filters vs. KQL in Lens may exclude subsets.
- **Time zone boundaries:** Bucket boundaries can shift counts at day boundaries.
- **Data quality:** Duplicate `_id`, partial docs, or nested field expansion can skew counts—validate with `_count` API and sample documents.
- **Inspector:** Compare exact DSL Lens generated vs. manual query; adjust metric to `Count of records` for true doc count.

### 8. Drilldown for interactive dashboards

- **Create drilldown:** In Lens or visualization → Add drilldown → Dashboard drilldown or URL drilldown.
- **Pass context:** Carry clicked bucket value (e.g., `service.name`, `host.name`) as filter variables to target dashboard.
- **Target dashboard:** Build detail dashboard with placeholders expecting those filters; test with sample clicks.
- **URL drilldowns:** For external systems (APM, Cases, Grafana), use Kibana context variables in URL template.
- **Discover drilldown:** Link from metric spike to pre-filtered Discover for raw logs.
- **Document in runbooks:** Standard drilldown paths for NOC ("click red service → error logs + traces").

### 9. Correlating logs, Metricbeat metrics, and APM on one timeline

- **Unified Observability apps:** Use Logs, Infrastructure, and APM with linked navigation; enable trace/log correlation in APM settings.
- **Common dimensions:** Normalize `service.name`, `host.name`, `container.id`, and `trace.id` across beats and APM agents.
- **Single dashboard:** Lens layers or TSVB combining index patterns (`logs-*`, `metrics-*`, `apm-*`) with aligned time axis.
- **Annotations:** Mark deploys and incidents on time series via manual annotations or indexed annotation source.
- **Trace id in logs:** Ensure app logs include `trace.id` and `transaction.id` for one-click pivot from log to trace.
- **Cases workflow:** Attach all three signal types to a Case during incident response for shared timeline.

### 10. Geographic distribution of errors with Maps

- **Geo fields:** Require `geo_point` (e.g., `client.geo.location` from GeoIP ingest pipeline or CDN headers).
- **Data prep:** Use Ingest Node GeoIP processor on IP fields; for client-side errors, capture user region with privacy controls.
- **Maps layer:** Create Maps visualization → add Choropleth or Points layer → source from `logs-*` with KQL filter `log.level: error`.
- **Metrics:** Color by count or unique users; use top-N filtering to avoid map clutter.
- **Drilldown:** Click region → filtered dashboard of errors for that geo bounding box.
- **Performance:** Pre-aggregate by `geo_hash` grid for high-volume data; consider transform for hourly geo error counts.

---

## Role-Based Access Control (RBAC)

### 11. Prevent accidental deletion of shared index patterns (100 teams)

- **Separate shared vs. team objects:** Platform-owned data views in a `platform` space; teams clone or reference read-only copies.
- **Custom roles:** Grant `All` on dashboards/visualizations but only `Read` on `index-pattern` / `data-view` for shared objects; `All` on team-owned patterns in team space.
- **Space isolation:** Each team space with its own data views; shared NOC patterns only in `default` or `noc` space with restricted editors.
- **Feature privileges:** Limit `Stack Management → Data Views` to platform role only.
- **Backups:** Daily saved object export to Git; quick restore if deletion occurs.
- **Audit:** Alert on `saved_object_delete` for `index-pattern` type in audit logs.

### 12. Space-based access for security, ops, and executives

- **Create spaces:** `security`, `operations`, `executive` with distinct URLs and branding.
- **Role mapping:** SAML groups → roles: `security_analyst` (Security app + security indices), `ops_engineer` (Infrastructure, Logs, Uptime), `exec_viewer` (read-only dashboards).
- **Feature controls:** Executives get `read` on Dashboard only; disable Dev Tools, Stack Management, and saved object export.
- **Data access:** Elasticsearch roles scoped by index patterns per space (`logs-security-*`, `metrics-*`, `business-kpi-*`).
- **Cross-space navigation:** Disable or limit for executives; provide single landing dashboard per persona.
- **Test matrix:** Validate each persona cannot access other spaces via URL guessing.

### 13. Feature controls to restrict APM, Uptime, SIEM

- **Kibana privileges:** Under Role → Kibana → assign `None` or `Read` per feature (APM, Uptime, Security, ML) per space.
- **Elasticsearch backing:** Even if UI hidden, restrict underlying indices (`apm-*`, `logs-*`, `security-*`) via cluster privileges.
- **Dashboard-only teams:** `Dashboard`, `Discover`, `Visualize` = All; Observability apps = None.
- **License awareness:** Some features require Platinum/Enterprise—combine license and RBAC.
- **API access:** Restrict API keys to minimum feature set for automation accounts.

### 14. User bypassing access via saved search index pattern modification

- **Document-level security (DLS):** Elasticsearch role query filters documents (e.g., `team_id: "{{_user.metadata.team_id}}"`).
- **Field-level security (FLS):** Hide sensitive fields (`credit_card`, `ssn`) at query time regardless of dashboard config.
- **Index aliases per tenant:** Users only have alias access to their tenant's indices—changing pattern in UI cannot reveal other aliases.
- **Read-only data views:** Where supported, prevent custom index pattern creation; provide pre-built data views only.
- **Disable saved object edit:** Read-only role for analysts who should not alter queries.
- **Audit saved object changes:** Detect index pattern tampering in audit logs.

### 15. SAML SSO with Okta/Azure AD and role mapping

- **Configure Elasticsearch SAML realm:** IdP metadata, SP entity ID, assertion signing; Kibana uses Elasticsearch security.
- **Kibana config:** `xpack.security.authc.providers.saml.saml1` with `order`, `description`, and RP settings.
- **Attribute mapping:** Map SAML `groups` or `roles` claim to Elasticsearch roles via `role_mapping` API.
- **Just-in-time users:** `xpack.security.authc.realms.saml` with `attributes.principal` mapping to username.
- **Multi-tenant:** Combine SAML roles with space assignment in role definitions (`spaces: ["team-a"]`).
- **Test:** Validate login, logout, session timeout, and group change propagation; document break-glass local accounts.

### 16. API key management for service accounts

- **Create API keys:** Stack Management → API Keys or `POST /_security/api_key` with role descriptors limiting indices and Kibana privileges.
- **Service accounts:** Prefer Elastic service tokens where available for internal stack communication.
- **Rotation:** Dual-key pattern—issue new key, deploy to consumers, revoke old after validation; automate rotation calendar (90 days).
- **Auditing:** Enable audit logs for `authentication_success` / `access_denied` with `api_key` id; ship to SIEM.
- **Storage:** Never embed keys in dashboards; use Kibana encrypted saved objects or external secret manager for connectors.
- **Least privilege:** Separate keys per integration (CI, reporting, alert webhook) for blast-radius control.

### 17. Space with incorrect permissions — audit and fix

- **Export role definitions:** `GET /_security/role` and Kibana role UI for effective privileges per space.
- **Saved object audit:** Review who has `write` on dashboard objects in the space (`saved_objects` permissions).
- **Space settings:** Stack Management → Spaces → verify `Disabled features` and default landing pages.
- **User simulation:** Use "View as" (where available) or test accounts per role to confirm view-only cannot edit.
- **Corrective role:** `Dashboard: Read`, `Visualize: Read`, `Index Patterns: Read` for viewers; editors get `Write` only on team-owned tags.
- **Prevent recurrence:** Space templates managed as code; prod changes via PR only.

### 18. MSP RBAC — customer data isolation

- **Tenant identifier:** Every document tagged with `customer.id`; enforced at ingest.
- **DLS per customer role:** Role query `customer.id: "cust-123"` for each customer-facing role.
- **Space per customer:** Dedicated space with branded dashboards cloning from master template.
- **Separate API keys and SAML tenants:** Map IdP organization claim to role mapping.
- **Prevent cross-tenant aggregations:** No shared indices without DLS; avoid global data views spanning all customers.
- **Operational access:** MSP admin role with break-glass logging and MFA for cross-tenant support.

---

## Visualizations & Analytics

### 19. TSVB for error rate % with moving average

- **Multiple series:** Series A = filter `status_code >= 500`, metric = count; Series B = all requests, metric = count.
- **Pipeline aggregation:** Use `Serial Diff` or formula in Lens alternatively; in TSVB use **Calc** tab: `params.numerator / params.denominator * 100`.
- **Moving average:** Add pipeline aggregation `Moving Average` on the calculated series with window 5–15 buckets.
- **Group by:** Split by `service.name` for per-service error rates.
- **Index:** `metrics-*` or APM transaction indices with `http.response.status_code`.
- **Validate:** Compare TSVB output to Prometheus/recording rule for same interval.

### 20. Vega visualization for custom chart types

- **Enable Vega:** Create visualization → Custom visualization → Vega or Vega-Lite.
- **Data URL:** Point to Elasticsearch query (`%context%` and `%timefield%` placeholders) or transform output.
- **Vega-Lite spec:** Define mark type, encodings, and transforms not available in Lens (e.g., custom heatmap layout, sankey).
- **Test iteratively:** Use Vega editor debug panel; start with small time range.
- **Performance:** Push aggregations to ES in the query portion; avoid fetching raw hits for large datasets.
- **Version control:** Export Vega JSON in saved objects Git repo.

### 21. High-cardinality field sampling in aggregations

- **Symptom:** `terms` aggregation `sum_other_doc_count` high; counts approximate with `shard_size` < precision threshold.
- **Fix accuracy:** Increase `shard_size` and `size`; use `composite` aggregation pagination for exact top-N tables.
- **Trade-off:** Higher memory and slower queries—may hit `search.max_buckets`.
- **Better approach:** Pre-aggregate via transform to hourly `service × status` summary index for dashboards.
- **Visualization choice:** For >10k cardinality, avoid pie charts; use filtered Data Table with search or drilldown.

### 22. Canvas for executive status board

- **Workpad design:** Fixed 1920×1080 layout, brand colors, large fonts for wall displays.
- **Data sources:** Elasticsearch SQL, esqueries, or Timelion functions feeding elements (metric, progress bar, status light).
- **Real-time:** `timelion` interval or workpad auto-refresh every 60s; keep queries lightweight.
- **Custom images:** SVG/PNG logos and status icons bound to data conditions (conditional styling).
- **Reporting:** Schedule PDF/PNG export to email for executives not watching wallboard.
- **Separate from ops dashboards:** Canvas optimized for aesthetics; operational triage stays in Lens/Observability apps.

### 23. Deployment events correlated with error rate

- **Annotation source:** Index deploy events (`event.kind: deployment`, `service.version`, timestamp) or use built-in APM deployment markers.
- **Lens/TSVB:** Enable annotations layer, select annotation index/data view, map `message` and time fields.
- **Visual verification:** Error rate spike within 5–15 minutes of annotation confirms correlation.
- **Alternative:** Vertical line via Timelion `points()` from deploy index.
- **Automation:** CI pipeline writes deploy document to `deployments-*` index on every release.

### 24. Machine learning anomaly detection on logs

- **Job type:** Log rate (`count`) or population analysis on structured fields (`url.path`, `response_code`).
- **Configure in ML app:** Pick data view, bucket span 15m, influencers `service.name`, `host.name`.
- **Calendar:** Exclude planned maintenance to reduce false positives.
- **Integrate dashboards:** Add **ML Annotations** layer or **Anomaly Explorer** embed link; use `ml.anomaly_score` in alert rules.
- **Operationalize:** Alert when record score > 75; route to on-call with influencer context.

### 25. Pie chart large "Other" category

- **Increase buckets:** Raise "Size" to show more slices (still limited for readability).
- **Use "Other" bucket:** Configure minimum doc count or `sum_other_doc_count` threshold to group tail.
- **Better chart types:** Horizontal bar with top 10 + "Other", treemap for hierarchical breakdown, or Data Table with sorting.
- **Filter noise:** KQL exclude health checks and known benign paths before aggregating.
- **Rule of thumb:** >8–10 categories → abandon pie for bar/table.

### 26. SLO compliance dashboard

- **Metrics source:** Elastic APM/OpenTelemetry SLIs, custom `slo.*` fields, or Prometheus federation into ES.
- **Panels:** Gauge for current SLI %, burn rate line (1h/6h windows), remaining error budget bar, historical compliance trend (30/90d).
- **Lens formulas:** `good events / total events * 100`; burn rate = `error_rate / (1 - slo_target)`.
- **Tiered SLOs:** Filter by `service.tier` for critical vs. standard.
- **Alert linkage:** Panel links to active SLO violation alerts and Cases.
- **QBR views:** Monthly rollup index via transform for executive summary.

---

## Alerting & Notification

### 27. Alert flapping — hundreds of notifications per hour

- **Tune conditions:** Widen threshold or require N consecutive violations (`numMatches` in index threshold rule).
- **Group alerts:** Use rule grouping by `service.name` or `host.name` to get one alert per service, not per host.
- **Throttle notifications:** Configure `throttle` (e.g., 1h) and `notify_when: onActionGroupChange` to alert on state transitions only.
- **Flapping detection:** Increase evaluation window; use 5–15 minute lookback for rate-based rules.
- **Maintenance windows:** Suppress during deploys; use active/active dedup with PagerDuty event orchestration.
- **Review recovered alerts:** Disable "notify on recovered" for noisy rules if it adds volume without value.

### 28. Stack rules vs. observability rules

- **Stack rules (Index threshold, Elasticsearch query):** Generic, any index/KQL; use for custom business metrics, log patterns, and cross-index logic.
- **Observability rules (Metric threshold, Log threshold, Uptime):** Pre-built for Elastic Agent/APM/Uptime data models with less config.
- **When to use stack:** Custom indices, complex DSL, non-Observability data (application logs in raw indices).
- **When to use observability:** Standard infra/APM/uptime with curated UIs and faster onboarding.
- **Production pattern:** Observability rules for 80% of infra; stack rules for app-specific SLOs and security use cases.

### 29. Alert rule "Request timeout"

- **Check rule execution log:** Stack Management → Rules → execution history for stack trace and ES response.
- **Elasticsearch:** Slow query? Run DSL in Dev Tools with `profile: true`; check thread pool rejections and GC.
- **Kibana:** Increase `xpack.alerting.rules.run.timeout` cautiously; check Kibana heap and task manager backlog (`GET .kibana_task_manager/_search`).
- **Network:** Latency between Kibana and ES clusters; cross-cluster search timeouts.
- **Simplify query:** Reduce time window, add filters, use summary indices instead of raw logs.
- **Shard availability:** Yellow cluster, unassigned shards cause slow searches.

### 30. Alert when 5xx rate exceeds 1% over 5 minutes per service

- **Rule type:** Metric threshold (APM) or Elasticsearch query grouped by `service.name`.
- **Condition:** `percentage() > 1` where numerator = `http.response.status_code >= 500`, denominator = all transactions, window 5m.
- **Per-service:** Group by `service.name`; set individual thresholds via rule templates or multiple rules with KQL filters.
- **Tier overrides:** Critical services 0.5%, internal tools 5%—deploy via saved object CI with parameter files.
- **Test:** Use alert "Test rule" with historical intervals; validate with known incident data.

### 31. Multi-channel notification strategy

- **Action connectors:** Configure PagerDuty (severity 1), Slack (warnings), Email (digest) connectors in Stack Management.
- **Rule actions by severity:** Critical → PagerDuty + Slack; Warning → Slack only; Info → daily email report rule.
- **Connector secrets:** Store in encrypted saved objects; rotate API tokens on schedule.
- **Runbook links:** Each action template includes `{{context.rule.name}}`, reason, and dashboard URL.
- **Digest rule:** Separate scheduled report aggregating sub-threshold events for email summary.

### 32. Maintenance windows

- **Create window:** Stack Management → Maintenance Windows → define schedule, affected services/tags, and scopes.
- **Assign to rules:** Link rules to maintenance window; notifications suppressed but executions may still log.
- **Post-maintenance review:** Enable "run rules during maintenance but don't notify" + export execution log for review; or use Cases to track deferred alerts.
- **Communication:** Auto-post to Slack when window starts/ends.
- **Overlap policy:** Prevent conflicting windows; document change calendar integration.

### 33. False positives on complex KQL alert

- **Test rule:** Use "Test" with time ranges covering known good and bad periods; inspect matched documents.
- **Discover preview:** Copy KQL to Discover; refine exclusions (`NOT healthcheck`, `NOT status: 499`).
- **Threshold tuning:** Require longer duration or higher count before firing.
- **Add influencers:** Group by `error.message` top terms in notification for faster human triage.
- **Iterate with feedback:** Track false positive rate per rule monthly; disable or rewrite worst offenders.

### 34. Case management integration with alerting

- **Enable Cases:** Security/Observability Cases with connector to Jira/ServiceNow.
- **Rule action:** Add "Create case" or webhook action on critical alerts with deduplication key.
- **Case template:** Pre-fill severity, service, dashboard links, and recent similar cases.
- **Bi-directional sync:** Jira connector updates case status; closing case resolves alert if configured.
- **War room:** Attach logs, traces, and dashboards to case timeline for collaborative investigation.

---

## Index Patterns & Data Management

### 35. 500 daily indices `logs-app-YYYY.MM.DD`

- **Data view pattern:** `logs-app-*` with wildcard; enable **Allow hidden and system indices** only if needed.
- **Time field:** `@timestamp`; verify across all indices.
- **Refresh:** Data views auto-pick new indices matching pattern on query; optionally run **Refresh field list** after mapping changes.
- **Performance:** Avoid `logs-app-*` spanning years in one query—use Index Lifecycle Management (ILM) and frozen tier for old data; default dashboard time ranges short.
- **Aliases:** Prefer `logs-app-write` alias for ingest and `logs-app-read` for queries if restructuring.
- **Field caps:** Use `field_caps` API periodically to detect mapping explosions across daily indices.

### 36. Incorrect field types after mapping change

- **Refresh field list:** Updates types from recent indices; conflicts show fields with multiple types.
- **Field conflict resolution:** Pin field type in data view or fix mapping on new indices going forward.
- **Avoid breaking dashboards:** Create new field name (`status_code.keyword`) rather than changing type in place when possible.
- **Reindex strategy:** For breaking changes, new index version `v2` with alias migration.
- **Communicate:** Dashboard owners update panels before decommissioning old field names.

### 37. Data views for multi-cluster setup

- **Cross-cluster search (CCS):** Configure remote clusters; data view pattern `logs:services-*` or `cluster-a:logs-*`.
- **Field aliases:** Normalize differing names via runtime fields (`host.name` vs. `hostname`) in each cluster's data view or unified CCS view.
- **Separate vs. unified:** One CCS data view for global search; cluster-specific views for ops with different mappings.
- **Latency:** CCS queries slower—use for investigative queries, not sub-second NOC refresh.
- **Failover:** Document which remote cluster aliases are required for DR.

### 38. Microservices — per-service indices, cross-service search

- **Pattern strategy:** `logs-*` or `logs-{service}-*` depending on cardinality and retention policies.
- **Common schema:** ECS (Elastic Common Schema) enforced at ingest for `service.name`, `trace.id`, `log.level`.
- **Unified data view:** `logs-*` for platform team; service teams get `logs-{their-service}-*` only via RBAC.
- **Runtime `service.name`:** If embedded only in index name, extract via runtime field from `_index`.
- **Dashboard templates:** Parameterized by `service.name` filter.

### 39. ILM rollover and dashboard continuity

- **Alias-based ingest:** Applications write to `logs-app-write`; ILM rollovers create `logs-app-000042`; dashboards query alias or `logs-app-*`.
- **Data view wildcard:** Ensure pattern matches rolled indices (`logs-app-*` not hardcoded date).
- **Reindex handling:** After rollover, new fields may appear—schedule periodic field refresh.
- **Testing:** Automate check that today's index matches data view after rollover job.
- **Frozen/searchable snapshots:** Update data view tier preferences for historical panels.

### 40. "No cached mapping for this field"

- **Refresh field list:** Pulls mapping from indices matching pattern; run after new fields first appear.
- **Field popularity:** Kibana caches popular fields; low-frequency fields may lazy-load on first use.
- **Performance impact:** Very wide mappings (10k+ fields) slow Kibana and Discover—use `field_mapping_limit` and ingest pruning.
- **Mitigation:** `ignore_malformed`, drop dynamic fields at ingest, use flattened field type for arbitrary keys.
- **Scheduled refresh:** API/automation to refresh data views after deploys that add fields.

---

## Query Optimization & Performance

### 41. Saved search timeout over 7 days

- **Narrow scope:** Reduce time range; add KQL filters (`service.name`, `log.level:error`).
- **Index pattern:** Target specific indices via ILM tier or `logs-app-2025.05.*` instead of full wildcard.
- **Query DSL:** Use `filter` context (no scoring); avoid leading wildcards; prefer `match_phrase` on keyword fields.
- **Exclude heavy fields:** `source: false` with `docvalue_fields` in advanced Discover settings where applicable.
- **Rollups/transforms:** Query summary index for trends; drill to raw only on narrow window.
- **Increase timeout cautiously:** `elasticsearch.requestTimeout` in advanced settings—not a substitute for optimization.

### 42. Inspector tool for query analysis

- **Open Inspector:** Panel menu → Inspect → Requests shows ES query DSL, response time, and shard stats.
- **Statistics tab:** Hit count, cache hit/miss, aggregation breakdown.
- **Compare panels:** Identify high `took` and large `total hits`.
- **Actions:** Copy DSL to Dev Tools; add filters, change aggregation sizes, test with `profile: true`.
- **Request cache:** Check if repeated queries benefit from shard request cache (same filters, no now-based relative times on cache-unfriendly queries).

### 43. Search sessions for long-running queries

- **Enable search sessions:** Stack Management → Advanced Settings → `searchSessions` features enabled.
- **Start session:** Run query in Discover/Lens; option to run in background; receive notification on completion.
- **Use case:** Ad-hoc 30-day forensic search without blocking browser tab.
- **Limits:** Sessions stored in `.kibana_search_sessions`; clean up expired sessions; monitor storage.
- **Governance:** Restrict to power users if cluster load is concern.

### 44. Dashboard making 50 separate queries

- **Shared time filter:** Ensure all panels use dashboard time—not per-panel overrides causing cache misses.
- **Consolidate visualizations:** Combine metrics in one Lens with multiple layers; split only when necessary.
- **Linked dashboards:** Overview (5 queries) + detail dashboards on demand.
- **TTS:** Use TSVB or Timelion single request multi-series where possible.
- **Reporting mode:** Batch export uses optimized path—different from interactive load.
- **Measure:** Inspector request count before/after consolidation.

### 45. Runtime fields — when to use

- **Definition:** Painless scripts computed at query time (e.g., `profit = revenue - cost`, parse JSON subfield).
- **Prefer index-time when:** Field used in heavy aggregations across billions of docs, same computation every query, or strict performance SLAs.
- **Prefer runtime when:** Ad-hoc exploration, rare queries, prototyping, or combining fields without reindex.
- **Trade-off:** Runtime fields increase CPU per query and cannot replace all index-time optimizations.
- **Governance:** Catalog approved runtime fields; avoid unbounded scripts on high-volume dashboards.

### 46. Optimize Discover for 1 billion documents

- **Always filter first:** Require `service.name`, time bounds, and level before broad search.
- **Disable expensive features:** Turn off field statistics on load; limit selected fields in sidebar.
- **Synthetic data views:** Query rollup/transform indices by default; link to raw Discover via drilldown.
- **Frozen tier / searchable snapshots:** For historical search with acceptable latency.
- **Async search:** Use search sessions or async search API for large result sets.
- **Ingest design:** Proper `keyword` vs `text`; avoid mega-documents and excessive nesting.

### 47. Field statistics for data quality

- **Enable:** Discover → field → Visualize distribution / Field statistics (where available in version).
- **Use cases:** Cardinality check (is `user.id` actually unique?), null rates, top values before building pie charts.
- **Pre-dashboard QA:** Sample 24h of data; identify mapping surprises and timestamp drift.
- **Automate:** Elastic ML data frame analytics or periodic transforms writing quality metrics to `data-quality-*` index.

---

## Observability Integrations

### 48. APM end-to-end tracing with log and metric correlation

- **APM Server / Fleet:** Deploy Elastic APM integration; configure agents with `service.name` and central config.
- **Log correlation:** Enable ECS logging with `trace.id`, `transaction.id` in application logs; APM UI shows "View logs" link.
- **Metrics:** Metricbeat or APM metrics for JVM/Go runtime correlated by `service.name` and `host.name`.
- **Unified service view:** Observability → APM → Service → Transactions, Errors, Metrics, Dependencies tabs.
- **OpenTelemetry:** Use OTel SDK exporting to Elastic APM for polyglot consistency.

### 49. Uptime monitoring for 200 global endpoints

- **Synthetics:** Create monitors (HTTP, TCP, ICMP, browser) per endpoint with locations matching user geography.
- **Private locations:** Deploy Elastic Agent-based private locations for internal APIs.
- **Thresholds:** Alert on down status, TLS cert expiry, response time p95; group by `monitor.tags`.
- **Scaling:** Use monitor tags (`region: eu`, `tier: critical`); bulk import monitors via saved objects API.
- **Dashboard:** Uptime app overview + regional breakdown; integrate with Cases for outages.

### 50. Infrastructure view for Kubernetes

- **Integrations:** `kubernetes` and `system` integrations on Fleet for node, pod, and container metrics.
- **Inventory:** Infrastructure → Hosts / Pods with CPU, memory, network; filter by `orchestrator.cluster.name`.
- **Log correlation:** Click pod → logs stream filtered by `kubernetes.pod.name`.
- **Unified dashboard:** Combine `metrics-kubernetes.*` and `logs-kubernetes.*` with shared filters.
- **Alerts:** Metric threshold on pod CPU/memory; log threshold for `OOMKilled` events.

### 51. Logs view for incident response

- **Stream logs:** Observability → Logs with real-time tail (`auto refresh`) on tight filters.
- **Categories:** Define log categories by `service.name` / `log.logger` for quick switching.
- **Highlights:** Highlight patterns (`Exception`, `error`, `timeout`) in settings.
- **Contextual links:** Configure links to APM trace, host metrics, and Cases.
- **Anomaly rules:** Log rate ML or threshold rules feeding same workspace.

### 52. Service Map for microservices

- **Requirements:** APM agents with distributed tracing; downstream spans reported (not tail-based sampling only on edges).
- **Map view:** Observability → APM → Service Map; edge thickness = throughput, color = error rate.
- **Incident use:** Identify upstream caller amplifying failures; click node for dependency latency breakdown.
- **Limitations:** Async messaging needs trace context propagation (Kafka headers, etc.) for accurate edges.

### 53. Universal Profiling for CPU hotspots

- **Deploy profiler:** Elastic Universal Profiling agent on hosts or Kubernetes daemonset.
- **No code changes:** eBPF-based sampling of production workloads with low overhead.
- **Analysis:** Flame graphs in Observability → Profiling; filter by `service.name`, host, process.
- **Correlation:** Link hot functions to APM traces and deploy annotations for "what changed."
- **Safety:** Test overhead in staging; exclude sensitive processes if policy requires.

### 54. Fleet management at scale

- **Fleet Server:** Dedicated scalable Fleet Server behind load balancer; multiple for HA.
- **Policies:** Base policy per OS/role; layer integrations (system, logs, security) via policy inheritance.
- **Agent health:** Fleet UI shows uninstalled, offline, failed upgrade; alert on agent check-in missed.
- **Automation:** Terraform/API for policy assignment; golden image with agent pre-installed.
- **Upgrades:** Rolling agent upgrades in waves by environment; pin versions during incidents.

---

## Multi-Space Management

### 55. Space provisioning for 20 teams — automated

- **Space API:** `POST /api/spaces/space` with id, name, disabled features, and initial objects.
- **IaC:** Terraform `elasticstack` provider or custom scripts in CI triggered by new team ticket.
- **Templates:** Export golden space NDJSON (dashboards, data views, roles) and import on create.
- **RBAC bundle:** Create role + role mapping + space in one pipeline job.
- **Catalog:** Service catalog webhook opens PR to `spaces/team-foo/` in Git.
- **Governance:** Naming conventions, quota on number of dashboards, periodic orphan space cleanup.

### 56. Cross-space dashboard sharing

- **Copy to space:** Platform publishes dashboards to all team spaces via API (`copy saved objects`).
- **Read-only in team spaces:** Team role gets `Read` on shared dashboard IDs; `Write` only for platform role in `platform` space.
- **Single source of truth:** Master dashboard in `platform` space; nightly sync job overwrites team copies or uses linked objects where supported.
- **Documentation:** "Do not edit NOC Overview" banner in dashboard description.
- **Alternative:** Default space dashboards with RBAC data separation instead of duplicating objects.

### 57. Space migration when reorganizing teams

- **Export NDJSON:** Include dashboards, visualizations, searches, data views from source space.
- **Import to target:** Map old data view IDs to new ones using `saved_object` references or import with conflict resolution.
- **Reference repair:** Use `saved_objects` `_migrate` or re-import in correct order (data views first).
- **Validation checklist:** Open each dashboard, verify panels load, run smoke queries.
- **Cutover:** Read-only old space during migration; communicate bookmark URL changes.

### 58. Space template system

- **Template repo:** `templates/team-space/` with standard dashboards (service overview, logs, infra).
- **Parameters:** Replace `{{TEAM_NAME}}`, `{{INDEX_PREFIX}}` in NDJSON before import.
- **Provisioning hook:** New space creation pipeline imports template and creates role.
- **Versioning:** Tag template releases; upgrade spaces via diff import with review.
- **Feedback loop:** Teams PR improvements back to template quarterly.

### 59. Programmatic copy-to-space via API

- **`POST /api/saved_objects/_copy_to_spaces`:** Copy dashboards/visualizations between spaces with `includeReferences` and `overwrite`.
- **Automation:** Jenkins/GitHub Action on merge to `dashboards/` triggers copy to all team spaces or tagged subset.
- **Idempotency:** Use consistent object IDs across spaces for easier updates.
- **Audit:** Log API calls; alert on mass delete/copy failures.

---

## Kibana as Code & Automation

### 60. Configuration as code with saved objects API

- **Export:** `GET /api/saved_objects/_export` with types and optional `includeReferencesDeep`.
- **Git:** Store NDJSON; PR review; semantic versioning tags per environment.
- **Import:** `POST /api/saved_objects/_import` with `createNewCopies` or `overwrite` per environment policy.
- **Spaces:** Import into specific space via `?space=` query parameter.
- **Dependencies:** Export data views, index templates, and ingest pipelines together for reproducibility.

### 61. CI/CD pipeline for dashboard deployment

- **Stages:** Lint NDJSON → import to staging Kibana → automated API tests (panel queries return 200, doc count > 0) → manual approval → prod import.
- **Tests:** Curl Kibana saved object APIs; optional Playwright screenshot diff for critical dashboards.
- **Rollback:** Re-import previous NDJSON tag from Git; keep last 5 releases artifacts.
- **Secrets:** CI uses scoped API key with import privileges only to target space.

### 62. Alerting and connectors as code

- **APIs:** `POST /api/alerting/rule`, `/api/actions/connector` with saved object export for rules/connectors.
- **Terraform:** `elasticstack_kibana_alerting_rule` resources where available.
- **Maintenance windows:** `POST /api/alerting/rules/maintenance_window` in same repo as rules.
- **Validation:** Dry-run rule test endpoint in staging before prod promotion.
- **Version lock:** Pin Kibana version in CI to avoid schema drift on upgrade.

### 63. Automated Kibana health monitoring

- **Synthetic checks:** Scheduled query against canary index; alert if zero docs in last 5m (data freshness).
- **Rule execution:** Monitor task manager failure rate; alert on rules in permanent error state.
- **Dashboard availability:** HTTP GET Kibana status API; Playwright login smoke test.
- **Elasticsearch stack monitoring:** Monitor Kibana node CPU, heap, response times via Metricbeat.
- **Meta-alerting:** External Prometheus/Grafana watches Kibana health to avoid single point of blind failure.

### 64. Scheduled PDF reporting

- **Enable reporting:** `xpack.reporting` with object storage or shared volume for report files.
- **Create report:** Dashboard → Share → PDF Reports → schedule cron (daily 8am).
- **Permissions:** Role needs `reporting_user` and access to underlying data.
- **Distribution:** Email connector or webhook to SharePoint/S3 via post-generation script.
- **Scale:** Reporting is resource-intensive—dedicated reporting instances or limit concurrent jobs.

---

## Security & Compliance

### 65. Audit logging for SOC 2

- **Enable ES audit:** `xpack.security.audit.enabled: true` with `audit.log` events for authentication, access granted/denied, admin changes.
- **Kibana events:** Saved object CRUD, space access, config changes, query failures (where configured).
- **Ship logs:** Filebeat → `auditbeat-*` or `logs-elasticsearch.audit-*` to immutable SIEM index.
- **Retention:** Meet SOC 2 retention (often 1 year); WORM/S3 Object Lock for tamper evidence.
- **Review:** Quarterly access reviews correlating audit logs with HR termination list.

### 66. Encrypted saved objects

- **Enable:** `xpack.encryptedSavedObjects` with encryption key in `kibana.yml` (`xpack.encryptedSavedObjects.encryptionKey`).
- **Protects:** API keys in connectors, credentials in actions, sensitive alert config.
- **Key management:** Use KMS-backed keys in production; document key rotation procedure (re-encrypt on rotation).
- **Backup:** Encryption keys must be backed up with snapshots—loss means unreadable connectors.

### 67. IP-based access restrictions

- **Reverse proxy:** nginx/ALB allowlist corporate IP ranges in front of Kibana.
- **Elasticsearch IP filtering:** `transport.host` restrictions for direct ES access separate from Kibana.
- **Remote workers:** VPN or Zero Trust (Cloudflare Access, IAP) required to reach Kibana.
- **CI/CD:** Dedicated egress IPs for deployment pipelines; separate API keys with minimal scope.
- **Document exceptions:** Break-glass procedure for on-call from non-corporate networks with MFA.

### 68. PCI-DSS security review for CDE dashboards

- **Network segmentation:** Kibana for CDE in isolated VPC; no shared spaces with non-CDE data.
- **RBAC + DLS:** Strict least privilege; PAN fields never indexed or masked at ingest.
- **Audit:** All dashboard views and queries logged; real-time alerts on bulk export attempts.
- **Encrypted connectors:** Mandatory encrypted saved objects for any integration touching CDE.
- **Vulnerability management:** Regular Kibana/ES patching; annual penetration test including Kibana SSRF paths.
- **Change control:** All dashboard changes via approved CI/CD; no manual prod edits.

---

## Advanced Kibana Scenarios

### 69. Upgrade broke 30 dashboards — pre-upgrade testing

- **Compatibility matrix:** Review Elastic deprecation logs and Upgrade Assistant warnings for visualization types.
- **Staging upgrade:** Clone prod saved objects to staging cluster on target version; open every critical dashboard.
- **Automated visual QA:** Script panel load tests; export list of deprecated viz types from Upgrade Assistant API.
- **Migration path:** Lens migration tool for legacy visualizations; manual rebuild for unsupported TSVB features.
- **Cutover plan:** Upgrade during maintenance window; keep NDJSON backup; rollback = restore previous Kibana + ES combo.
- **Post-upgrade:** 48h hypercare with dashboard owners validating panels.

### 70. Lens formulas for Apdex, burn rate, health scores

- **Apdex:** `countIf(response.duration < 500ms, "satisfied") + countIf(..., "tolerating") * 0.5) / count()`.
- **Burn rate:** `(error_rate - slo_target) / (1 - slo_target)` over sliding windows using formula and time shift.
- **Health score:** Weighted average of normalized metrics (latency, errors, saturation) with `clamp` in formula.
- **Validate:** Compare formula output to offline spreadsheet on sample interval.
- **Document:** Include formula definitions in dashboard markdown panel for maintainability.

### 71. Incident management with PagerDuty, Jira, Slack

- **Cases as hub:** Create case from alert; link dashboards, traces, and runbooks.
- **Connectors:** PagerDuty for sev-1, Jira for ticket sync, Slack for channel updates.
- **Workflow:** Alert → Case → auto-assign on-call → Jira epic → Slack war room channel.
- **Resolution:** Close case resolves linked alerts; postmortem template attached to Jira issue.
- **Metrics:** Track MTTR via case timestamps for quarterly review.

### 72. Custom branding and white-labeling

- **Kibana.yml:** `server.publicBaseUrl`, logo paths, `xpack.customBranding` (license-dependent) for logo, colors, login screen.
- **Per-space branding:** Different space themes for MSP customers where license supports advanced customization.
- **CSS (limited):** Some versions allow limited theme overrides; prefer supported branding APIs.
- **DNS:** Customer-specific subdomain `customer-a.observability.com` routing to their space.
- **Legal:** Custom footer with privacy policy and support contacts.

### 73. Discover timeline for incident reconstruction

- **Multi-source data view:** `logs-*` with filters per subsystem; align on `@timestamp`.
- **Chronological view:** Sort ascending; narrow to incident window ±15 minutes.
- **Surrounding documents:** "View surrounding documents" for context around key error line.
- **Correlate:** Filter by shared `trace.id`, `user.id`, or `transaction.id` across services.
- **Annotations:** Overlay deploy and config change markers.
- **Export:** Save search for postmortem attachment.

### 74. Threshold anomaly detection for seasonal business metrics

- **ML job with calendars:** Define business hours and holiday calendars to exclude expected dips/spikes.
- **Category:** Use `day_of_week` and `hour_of_day` as influencers for retail seasonality.
- **Length of season:** Set `model_plot_config` and bucket span appropriate for daily/weekly cycles (e.g., 1h buckets, 4 weeks training).
- **Alert:** When anomaly score exceeds threshold for 3 consecutive buckets.
- **Fallback:** Static thresholds during known events (Black Friday) with manual override calendar entries.

### 75. High memory from concurrent complex queries

- **Kibana scaling:** Horizontal scale Kibana nodes behind load balancer; sticky sessions optional.
- **Resource limits:** `elasticsearch.requestTimeout`, max concurrent searches per user (custom proxy if needed).
- **Query governance:** Discourage unbounded Discover; provide curated dashboards and summary indices.
- **Session management:** Reduce `session.idleTimeout`; limit saved searches returning huge field lists.
- **Monitoring:** Track Kibana NodeJS heap; identify heavy dashboards via APM on Kibana itself.
- **Offload:** Reporting and async search for batch workloads vs. interactive.

### 76. Golden signals SRE dashboard suite with service discovery

- **Data:** APM metrics (latency p50/p95/p99), transaction rate (traffic), error rate, and infra saturation (CPU/memory/thread pools).
- **Service discovery:** APM auto-discovers services; filter `environment: production`.
- **Dashboard per tier:** Auto-generate from template using `service.name` list from service catalog API.
- **Drilldown:** Service overview → traces → logs → host metrics.
- **SLO overlay:** Error budget burn on error panel; latency SLO line on latency chart.

### 77. Transforms for summarized indices

- **Create transform:** Continuous pivot aggregating hourly metrics by `service.name` into `metrics-summary-*`.
- **Use in Kibana:** Data view on destination index for fast long-term trends.
- **ILM on destination:** Hot warm cold delete aligned with retention needs.
- **Lifecycle:** Update transform query when schema changes; reindex destination if pivot breaking.
- **Monitoring:** Alert on transform failure via Stack Monitoring or rule on `transform.health`.

### 78. Monitoring the Elastic Stack from Kibana

- **Stack Monitoring:** Enable on cluster; Metricbeat collection; view in Monitoring UI (Elasticsearch, Kibana, Logstash, Beats).
- **Alerts:** Built-in stack rules for cluster health, license expiry, disk watermark.
- **Logstash:** Pipeline monitoring dashboards for throughput and failed events.
- **Kibana health:** Response times, saved object count, task manager backlog.
- **External mirror:** Also ship stack metrics to corporate Prometheus for unified alerting.

---

## Basic Questions

### 1. What is Kibana and what is its primary role in the Elastic Stack?

Kibana is the visualization and management UI for the Elastic Stack. Its primary role is to explore, analyze, and present data stored in Elasticsearch through dashboards, Discover, and specialized apps (APM, Security, etc.). It also provides stack administration (index management, Fleet, alerting) in one interface.

### 2. What is a Kibana index pattern and how do you create one?

An index pattern (now often called a **data view**) tells Kibana which Elasticsearch indices to search and which field holds the time series (`@timestamp`). Create one in **Stack Management → Data Views → Create data view** by entering an index pattern like `logs-*` and selecting the time field.

### 3. What is the Kibana Discover view used for?

Discover is used for interactive log and document exploration: searching with KQL, filtering, viewing raw fields, and inspecting individual events. It is the primary tool for ad-hoc investigation before building visualizations.

### 4. What is the difference between Kibana's Visualize and Dashboard features?

**Visualize** (or Lens) builds individual charts and tables from Elasticsearch aggregations. **Dashboard** combines multiple saved visualizations, controls, and filters on one screen for operational monitoring and sharing.

### 5. What is KQL (Kibana Query Language) and how does it differ from Lucene query syntax?

KQL is a simplified, user-friendly query language (`field:value`, `and`/`or`, ranges) optimized for filters in Discover and Lens. Lucene syntax offers more powerful full-text operators but is harder to read; Kibana often exposes both, with KQL as the default in modern versions.

### 6. How do you create a basic bar chart visualization in Kibana?

Open **Visualize Library → Create visualization → Lens** (or legacy Vertical Bar), select a data view, choose **Vertical Bar**, set a metric (e.g., Count), and a bucket aggregation (e.g., Terms on `host.name`). Save and add to a dashboard.

### 7. What is the purpose of Kibana's time filter?

The time filter limits all queries to a specific time range so dashboards and Discover only show relevant recent data. It is essential for time-series indices and performance, preventing full-index scans.

### 8. What is a Kibana saved search and when would you use it?

A saved search stores a Discover query (KQL filters, columns, sort order) for reuse. Use it when multiple visualizations or dashboards need the same base query, or to share a standard investigation view with a team.

### 9. How do you share a Kibana dashboard with a team member?

Share the dashboard URL (optionally with snapshot input state), export/import the saved object, or grant access via the same **space** and **role**. For external users, use reporting (PDF/PNG) or embed codes where licensing permits.

### 10. What is the purpose of Kibana's Lens editor?

Lens is Kibana's primary drag-and-drop visualization editor that suggests chart types based on field types. It simplifies building aggregations, formulas, and drilldowns without writing query DSL manually.

### 11. What is a Kibana space and what is it used for?

A space is an isolated workspace with its own saved objects (dashboards, data views) and access controls. Teams use spaces to separate environments or organizational units while sharing one Kibana cluster.

### 12. How do you create a Kibana alert based on an index threshold?

Go to **Stack Management → Rules → Create rule → Index threshold**, select indices, define a count or aggregation threshold over a time window, and attach actions (email, Slack, etc.). Schedule the check interval and save the rule.

### 13. What is the purpose of Kibana's Maps feature?

Maps visualizes geospatial data on layered maps (choropleths, points, tracks). It is used for geographic analysis such as request origins, asset locations, or regional error distribution.

### 14. How do you configure Kibana's default index pattern?

Set the default data view in **Stack Management → Data Views** by clicking the star on the desired data view, or set `kibana.defaultAppId` / data view preferences per space. Discover uses the default when none is selected.

### 15. What is the Kibana Canvas feature used for?

Canvas creates pixel-perfect, presentation-style live displays (workpads) with custom layouts, branding, and typography. It is commonly used for NOC wallboards and executive screens rather than exploratory analytics.

### 16. How do you add a filter to a Kibana dashboard?

Click **Add filter** in the KQL bar, select field and operator, or click a chart segment and choose **Create filter**. Filters can be pinned to all panels or edited per dashboard.

### 17. What is the purpose of Kibana's TSVB (Time Series Visual Builder)?

TSVB builds advanced time series charts with multiple indices, pipeline aggregations, annotations, and math across series. It is used when Lens lacks specialized time-series or pipeline features needed for complex operational charts.

### 18. How do you configure Kibana to connect to an Elasticsearch cluster?

Set `elasticsearch.hosts` in `kibana.yml` (and credentials via `elasticsearch.username` / password or service account token). Ensure Kibana's service account has the `kibana_system` role and network connectivity to Elasticsearch HTTP port.

### 19. What is the Kibana Timelion feature used for?

Timelion plots time series using a functional expression language (`.es()`, `.label()`). It is used for chained time series math and custom charts, though Lens and TSVB have largely superseded it for new work.

### 20. How do you create a Kibana data table visualization?

In Lens, choose **Table**, add rows grouped by terms (e.g., `service.name`) and metrics (count, average latency). Alternatively use legacy **Data Table** visualization from the Visualize library.

### 21. What is the purpose of Kibana's field formatter?

Field formatters control how values display (bytes, percentages, URLs, colors) without changing stored data. They make dashboards human-readable and consistent across visualizations using the same data view.

### 22. How do you configure Kibana's dark mode?

Enable **Dark mode** under **Stack Management → Advanced Settings → Theme** (or user profile appearance settings in newer versions). Some deployments set `xpack.dashboardUi.darkMode` defaults in `kibana.yml`.

### 23. What is the Kibana APM feature used for?

APM monitors application performance: traces, transaction latency, errors, and dependencies. It helps developers and SREs find slow endpoints, failure hotspots, and distributed tracing issues.

### 24. How do you create a Kibana pie chart visualization?

In Lens, select **Pie** chart type, configure a slice dimension (terms aggregation) and a metric (count or sum). Limit slice count to keep the chart readable.

### 25. What is the purpose of Kibana's tag cloud visualization?

A tag cloud shows term frequency with font size proportional to count. It is useful for spotting prominent keywords in unstructured text such as log messages or tags.

### 26. How do you configure Kibana's session timeout?

Set `xpack.security.session.idleTimeout` and `lifespan` in `kibana.yml` (e.g., `30m`). Elasticsearch realm session settings may also apply for SSO integrations.

### 27. What is the Kibana Uptime feature used for?

Uptime (Synthetics) monitors endpoint availability and response time from global or private locations. It alerts when websites or APIs fail or degrade.

### 28. How do you create a Kibana metric visualization?

In Lens, use **Metric** or **Gauge** chart types with a single aggregated value (e.g., total errors last 15m). Legacy **Metric** visualization also shows one big number per aggregation.

### 29. What is the purpose of Kibana's heat map visualization?

Heat maps show density of values across two dimensions (e.g., hour of day vs. day of week) using color intensity. They reveal patterns like peak traffic hours or recurring failure windows.

### 30. How do you configure Kibana's default time zone?

Set `dateFormat:tz` in **Advanced Settings** or `xpack.encryptedSavedObjects`/`uiSettings` overrides per space. User profiles can also set a preferred timezone for display.

### 31. What is the Kibana SIEM/Security feature used for?

The Security app provides SIEM capabilities: detection rules, alerts, timelines, and cases for threat hunting and incident response. It analyzes security events and logs indexed in Elasticsearch.

### 32. How do you create a Kibana coordinate map visualization?

Use **Maps** or legacy **Coordinate Map** visualization, select a `geo_point` field, and choose basemap layer. Configure metrics for point size or color based on document values.

### 33. What is the purpose of Kibana's goal and gauge visualization?

Goal and gauge charts show a single metric against a target threshold (e.g., 99.9% SLO). They communicate progress or compliance at a glance on executive dashboards.

### 34. How do you configure Kibana's reporting feature for PDF generation?

Enable `xpack.reporting` in `kibana.yml`, assign users the `reporting_user` role, then use **Share → PDF Reports** on a dashboard to generate or schedule reports. Configure object storage for large deployments.

### 35. What is the Kibana Logs feature used for?

The Observability **Logs** app provides a tailored log streaming and categorization experience with highlights and links to metrics and traces. It is optimized for operators during incidents.

### 36. How do you create a Kibana vertical bar chart with multiple metrics?

In Lens, add multiple metrics to a bar chart or use **Break down by** for split series. TSVB also supports multiple series in one panel with different filters per series.

### 37. What is the purpose of Kibana's region map visualization?

Region maps color countries or regions based on aggregated metrics (e.g., errors per country). They require a recognized region field join (ISO codes) or Elasticsearch supported shapes.

### 38. How do you configure Kibana's index pattern field refresh?

In **Stack Management → Data Views**, open the data view and click **Refresh field list** to pull new fields from Elasticsearch mappings. Automate via API after schema changes.

### 39. What is the Kibana Infrastructure feature used for?

Infrastructure shows hosts, pods, and containers with CPU, memory, and network metrics from Elastic Agents. It helps correlate infrastructure saturation with application issues.

### 40. How do you create a Kibana line chart with multiple series?

In Lens, choose **Line** chart, add breakdown by terms or multiple metrics. Set different colors and axis options for comparing series over time.

### 41. What is the purpose of Kibana's saved objects management?

Saved Objects management lists and exports dashboards, visualizations, data views, and rules. Administrators use it for migration, cleanup, and troubleshooting broken references.

### 42. How do you configure Kibana's advanced settings?

Go to **Stack Management → Advanced Settings** to tune Kibana and UI behavior (timeouts, default routes, query preferences). Changes apply per space or globally depending on setting.

### 43. What is the Kibana Dev Tools console used for?

Dev Tools provides a console for Elasticsearch REST API calls and sometimes runtime fields testing. Engineers use it to run queries, inspect mappings, and troubleshoot outside the visual UI.

### 44. How do you create a Kibana area chart visualization?

In Lens, select **Area** as the chart type with a date histogram on the X-axis and one or more metrics. Stacked area shows composition over time.

### 45. What is the purpose of Kibana's index management feature?

Index Management in Stack Management shows indices, mappings, settings, and ILM policies. Operators use it to inspect shard health, force rollover, and troubleshoot storage issues.

---

## Intermediate Questions

### 1. How do you implement Kibana's RBAC for multi-team dashboard access?

Define Elasticsearch roles with index privileges per team and Kibana privileges per space (Dashboard/Discover read vs. write). Map SSO groups to roles, use spaces for isolation, and restrict Stack Management to platform admins. Test effective permissions with representative user accounts.

### 2. Describe SAML authentication for enterprise SSO.

Configure Elasticsearch SAML realm with IdP metadata, set Kibana as SP, and map SAML attributes (groups) to Elasticsearch roles via role mappings. Enable SAML as an auth provider in `kibana.yml`, test login/logout flows, and maintain break-glass local users for IdP outages.

### 3. How do you implement alerting for log threshold with multi-channel notifications?

Create a **Log threshold** rule in Observability or an **Elasticsearch query** rule, define count threshold and grouping, then attach multiple connectors (PagerDuty, Slack, email) with severity-based conditions. Use throttling and maintenance windows to control noise.

### 4. Explain Lens formula for complex calculated metrics.

Lens formulas let you combine metrics with arithmetic (`divide`, `multiply`) and functions like `moving_average` on aggregated results. Define numerator/denominator for rates and use currency or percent formatters; validate against known SQL or PromQL calculations.

### 5. How do you implement space-based multi-tenancy?

Create a space per team with dedicated saved objects and landing pages. Assign roles that limit users to their space and backing indices. Automate provisioning with the Spaces API and export/import templates for consistency.

### 6. Configure APM for end-to-end transaction tracing.

Deploy APM Server or Fleet APM integration, instrument apps with Elastic APM agents or OpenTelemetry, and ensure distributed tracing headers propagate across services. Enable log correlation and service maps; configure sampling appropriately for production volume.

### 7. Machine learning anomaly detection for logs.

Create ML jobs for log population or count anomalies on key fields (`url.path`, `log.level`). Use calendars for known patterns, set alerts on anomaly scores, and embed results in operational dashboards via anomaly layers or dedicated ML views.

### 8. Runtime fields vs. index-time fields.

Runtime fields compute values at query time with Painless—flexible but more expensive per query. Index-time fields (ingest pipelines or explicit mappings) are preferred for high-cardinality aggregations on huge datasets. Use runtime for prototyping and narrow admin queries.

### 9. Dashboard drilldown for interactive exploration.

Configure dashboard drilldowns from Lens visualizations to pass clicked filters (e.g., `service.name`) to a detail dashboard or Discover. Document standard drill paths for operators and test variable passing across spaces.

### 10. Fleet management for Elastic Agent at scale.

Deploy HA Fleet Servers, define layered agent policies per workload, and use tags for grouping. Monitor agent health in Fleet UI, automate upgrades in waves, and integrate agent install into image build pipelines.

### 11. Transforms for summarized reporting indices.

Create a continuous transform pivoting raw metrics into hourly summaries by service. Point dashboards at the destination index for fast long-range charts and apply ILM for retention on the summary index.

### 12. Search sessions for long-running queries.

Enable search sessions in advanced settings so users run large Discover queries in the background. Monitor session storage and set governance policies for who may run multi-day searches against raw indices.

### 13. Case management for incident tracking.

Enable Cases with connectors to Jira/ServiceNow, create cases from alerts with linked dashboards and observability data, and assign owners. Use cases as the single timeline during incidents and close linked alerts when resolved.

### 14. Osquery integration for endpoint security.

Deploy Osquery Manager integration via Fleet on endpoints, define scheduled queries packs, and analyze results in Kibana Security. Correlate Osquery findings with host logs and vulnerability data for investigation.

### 15. Alerting with action connectors for automated response.

Configure connectors (webhook, Slack, PagerDuty, Jira) and attach to rules with conditional actions. Use connector secrets in encrypted saved objects and test with rule simulation before enabling production notifications.

### 16. Data view field statistics for data quality.

Use field statistics in Discover to inspect cardinality, top values, and distributions before building charts. Identify mapping issues (high cardinality keywords, null rates) early to avoid misleading visualizations.

### 17. Canvas workpads for executive dashboards.

Build fixed-layout Canvas workpads with branded elements fed by Elasticsearch queries, schedule PNG/PDF exports, and display on wallboards. Keep data queries lightweight and separate operational triage dashboards in Lens.

### 18. Cross-cluster search for multiple clusters.

Register remote clusters in Elasticsearch, then create data views using `remote_cluster:index-*` syntax. Account for increased latency and network failure modes; use CCS primarily for investigative rather than sub-second monitoring.

### 19. Vega visualization for custom charts.

Author Vega or Vega-Lite JSON with Elasticsearch query data sources for charts not available in Lens. Version control specs in Git and test performance with aggregations pushed to Elasticsearch rather than client-side rendering of raw hits.

### 20. Aggregation-based visualizations vs. Lens.

Legacy aggregation visualizations offer mature specific chart types but less guidance; Lens provides guided analysis, formulas, and better maintainability. Prefer Lens for new dashboards unless a legacy feature (specific TSVB pipeline) is required.

### 21. Dashboard-to-dashboard navigation with context.

Use drilldowns, links dashboard panel, or URL templates with context variables (`{{value}}`) to navigate between dashboards while preserving filters. Standardize variable names across the dashboard suite.

### 22. Elasticsearch query optimization for slow panels.

Use Inspector to copy DSL, add filter context, reduce time range, avoid high-cardinality terms, and query summary indices. Increase shard request cache effectiveness with consistent time filters across panels.

### 23. Role-based feature controls.

In role definitions, set per-feature privileges (APM, Security, ML) to `None`, `Read`, or `All` for each space. Combine with Elasticsearch index restrictions so hidden features cannot be accessed via API.

### 24. Service map for microservice dependencies.

APM Service Map builds dependency graphs from trace data. Ensure instrumentation reports outbound spans and messaging trace context; use the map during incidents to find upstream callers and downstream failures.

### 25. Alerting for Kubernetes pod failures with Metricbeat.

Create metric threshold rules on `kubernetes.pod.status` or `kubernetes.container.cpu` metrics, grouped by pod/deployment. Complement with log threshold rules for `OOMKilled` and cluster-level alerts on node conditions.

### 26. ILM integration for data tier visualization.

Use Index Management and index lifecycle phase metadata in visualizations; query `data_stream` or ILM explain API for index age. Build dashboards showing index size per tier and ILM action failures.

### 27. Profiling for CPU hotspots.

Deploy Universal Profiling agents, collect stack samples in production, and analyze flame graphs filtered by service and host. Correlate findings with APM slow transactions and recent deployments.

### 28. Discover timeline for incident sequences.

Search across `logs-*` with ascending time sort, narrow to incident window, and use surrounding documents and shared `trace.id` to reconstruct event order. Save searches for postmortem evidence.

### 29. Saved object export/import for migration.

Export NDJSON from source environment including references, import to target with overwrite policy, and validate dashboards. Store exports in Git as part of CI/CD promotion between dev, staging, and prod.

### 30. Audit logging for SOC 2.

Enable Elasticsearch audit logs for authentication and data access, ship to immutable storage, retain per policy, and review quarterly. Include Kibana saved object changes and admin actions in scope.

### 31. Threshold anomaly detection for business metrics.

Configure ML univariate or multivariate jobs on business KPIs with calendars for seasonality. Tune bucket span and alert thresholds; integrate anomaly scores into executive dashboards.

### 32. Field-level security with Elasticsearch.

Define roles with `field_security` grant/exclude lists so sensitive fields never return to unauthorized users. Kibana respects FLS automatically—users cannot bypass via visualization field selection.

### 33. Reporting schedule for automated distribution.

Schedule PDF reports on key dashboards via Reporting UI or API, deliver through email connectors, and monitor reporting queue health. Restrict reporting privileges to prevent data exfiltration.

### 34. Synthetics monitoring for global endpoints.

Create HTTP/browser monitors from multiple Elastic Cloud locations and private agents, tag by region and criticality, and alert on downtime or SLA breaches. Import monitors as code for hundreds of endpoints.

### 35. Data table with computed columns for SLO reporting.

Use Lens formulas or runtime fields for SLI percentage, error budget remaining, and burn rate columns in a table breakdown by service. Link rows to detail dashboards filtered by service.

### 36. Timelion expressions for time series calculations.

Timelion uses functions like `.es(metric=avg:system.cpu.user).movingaverage(10)` chained in expressions. Useful for legacy dashboards; new work often uses Lens formulas or TSVB instead.

### 37. Tag-based dashboard organization

Apply Kibana tags to dashboards and visualizations for team, domain, and criticality. Filter the dashboard list by tag and enforce tagging standards in CI when importing saved objects.

### 38. Custom branding for white-labeled platforms.

Configure `xpack.customBranding` and space-specific themes (license permitting), custom logos, and public base URLs per tenant. Combine with spaces and SSO for MSP-style experiences.

### 39. Alert suppression during incidents

Use alert grouping, throttling, and maintenance windows to suppress duplicate pages. Track suppressed alert counts in Cases and review after incident resolution to catch missed signals.

### 40. Universal Profiling in the Elastic Stack

Universal Profiling uses eBPF to sample production CPU stacks without code changes. It integrates with APM and Logs for correlation, helping find hotspots that metrics alone cannot explain.

---

## Advanced Questions

### 1. Kibana deployment for 500-team enterprise

- **Central platform cluster:** Multi-node Kibana behind LB with dedicated Fleet/APM components as needed.
- **Automated space provisioning:** API/Terraform pipeline from service catalog with template NDJSON per team.
- **Central RBAC:** SAML → role mapping; standard role templates (viewer, editor, admin) with index DLS by `team.id`.
- **GitOps dashboards:** All saved objects in Git; CI promotes staging → prod with smoke tests.
- **SSO and audit:** Mandatory SAML, audit logs to SIEM, encrypted saved objects for connectors.
- **Governance:** Platform team owns shared objects; teams own team-space content with quotas and tagging standards.

### 2. Kibana-based SOC platform

- **Security app:** Detection rules, threat intel feeds, and correlation across `logs-*`, `security-*` indices.
- **Cases workflow:** Alert → case → analyst assignment → timeline → closure with Jira sync.
- **Automated response:** Webhook actions to SOAR, isolation playbooks via API connectors.
- **ML and SIEM:** Population anomalies, rare process execution, lateral movement detections.
- **Compliance:** Full audit trail, RBAC by analyst tier, air-gapped evidence export.

### 3. Observability suite for 1,000 microservices

- **APM auto-discovery:** Unified `service.name` schema; service catalog enriches criticality.
- **SLO app / custom indices:** Error budget tracking per tier with burn-rate alerts.
- **Correlation:** Trace-log-metric linking and Cases for incident correlation.
- **Service map at scale:** Sampling strategy that preserves dependency edges for critical services.
- **Tiered dashboards:** Executive summary + auto-generated per-service dashboards from templates.

### 4. Dashboard availability during ES degradation

- **Degraded mode messaging:** Banner when cluster yellow/red; show last successful query timestamp.
- **Cached screenshots:** Scheduled reporting to object storage for read-only fallback NOC view.
- **Query routing:** Read from replica tier or cross-cluster replica; shorten default time ranges automatically.
- **Critical panels first:** Lightweight summary indices on dedicated coordinating nodes.
- **Graceful failure:** Panel-level error messages instead of blank dashboard; meta-monitoring external to stack.

### 5. Compliance monitoring for financial services (SOX, PCI, GDPR)

- **Data classification at ingest:** Tags drive DLS and retention policies.
- **Separate spaces/indices:** PCI CDE isolated; GDPR pseudonymization for PII fields.
- **Immutable audit logs:** WORM storage for access and admin events.
- **Unified compliance dashboards:** Control effectiveness metrics, access reviews, policy violations.
- **Reporting:** Scheduled evidence exports for auditors with signed PDFs and chain of custody.

### 6. ML at scale across 10,000 metrics

- **Job templates:** Auto-create ML jobs from metric catalog with sensible bucket spans.
- **Hierarchy:** Population jobs across related metrics; prioritize tier-1 services.
- **Alert routing:** Route by influencer and service tier; suppress flapping with calendars.
- **False positive feedback loop:** Analyst labels feed model retraining and threshold tuning.
- **Resource governance:** Dedicated ML nodes; limit concurrent open jobs; auto-close unused jobs.

### 7. Multi-cloud unified Kibana deployment

- **Cross-cluster search or federated search:** Query AWS, GCP, Azure clusters from one Kibana.
- **Normalized schema:** ECS everywhere; runtime fields map cloud-specific metadata to common names.
- **Global dashboards:** CCS data views with `cloud.provider` breakdown panels.
- **Latency handling:** Regional Kibana instances with local data preferred; global only for DR/analytics.
- **Identity:** Single SSO with roles granting per-cloud index access.

### 8. Incident response platform with war room dashboards

- **Auto-populate case dashboard:** Webhook from PagerDuty creates case and attaches service dashboard.
- **Signal correlation:** Pre-built panel combining errors, latency, deploys, and infra for affected service.
- **Progress tracking:** Case status, checklist, and timeline exported for postmortem.
- **Communication:** Slack connector updates channel; dashboard URL pinned.
- **Post-incident:** One-click postmortem export of Discover saved searches and annotations.

### 9. MSP RBAC for 200 customers

- **DLS on `customer.id`:** Every query filtered; no shared unsecured indices.
- **Space + branding per customer:** Cloned templates, custom login where required.
- **SAML org mapping:** Customer IdP groups map to isolated roles.
- **MSP break-glass:** Logged, MFA, time-limited cross-customer support role.
- **Automated provisioning:** Customer onboarding pipeline creates space, role, indices, and dashboards.

### 10. 50,000 concurrent users, sub-second dashboards

- **Horizontally scale Kibana:** 20+ nodes, connection pooling to ES, CDN for static assets.
- **Elasticsearch tuning:** Dedicated coordinating nodes, shard sizing, summary indices for dashboards.
- **Aggressive caching:** Short default time windows, identical queries across panels, avoid `now` in cache keys where possible.
- **Query governance:** Disable ad-hoc heavy Discover for general users; curated dashboards only.
- **Load testing:** Gatling/k6 against Kibana APIs before launch; continuous SLO on p95 panel load time.

### 11. DevSecOps platform in Kibana

- **Integrate CI scan results:** Index SAST/DAST findings; overlay deploy events on vulnerability trends.
- **Runtime security:** Elastic Defend alerts in Security app linked to affected services in APM.
- **Unified timeline:** Deploy → vulnerability introduced → exploit attempt detection.
- **Gates:** Block deploy if critical findings spike (external CI, displayed in Kibana).

### 12. APM for polyglot 20-language architecture

- **OpenTelemetry:** Standardize on OTel SDKs exporting to Elastic APM Server.
- **Consistent `service.name`:** Enforced via central agent config and service catalog.
- **Trace propagation:** W3C tracecontext across HTTP, gRPC, and messaging.
- **Unified service map:** Language-agnostic dependency graph; language shown as metadata.
- **Sampling:** Tail-based sampling for errors; head sampling for high-volume health paths.

### 13. Automated compliance reporting

- **Scheduled Canvas/PDF reports:** Weekly control dashboards per framework.
- **Evidence collection:** Saved Discover searches exported with timestamp and user audit trail.
- **Regulatory submission:** Signed reports stored in immutable bucket with retention locks.
- **Drift detection:** Alert when control metric leaves compliance band.

### 14. Alerting at enterprise scale (50,000 rules)

- **Rule governance:** Templates, naming standards, mandatory owners and tags.
- **Deduplication:** Group keys and PagerDuty event orchestration; global rate limits per connector.
- **Performance:** Dedicated alerting nodes, optimize ES queries, summary indices for thresholds.
- **Lifecycle:** Auto-disable unused rules; quarterly review of firing frequency.
- **Sharding:** Multiple Kibana instances by domain if task manager bottlenecks.

### 15. Capacity planning platform

- **Historical metrics in ES:** Ingest cloud and infra metrics; transforms for weekly growth rates.
- **Forecasting:** ML forecast jobs on CPU, disk, request rates per service.
- **Recommendations dashboard:** Projected exhaustion dates and suggested scale actions.
- **Integration:** Export recommendations to Terraform/Ticket API for approval workflow.

### 16. Security analytics for zero-trust

- **Correlate identity (Okta), network (VPC flow), endpoint (Defend) in SIEM timelines.
- **Detect lateral movement:** Unusual auth patterns, new service accounts, anomalous east-west traffic.
- **UEBA:** ML on user behavior baselines.
- **Response:** Automated case creation and network isolation playbooks.

### 17. Healthcare HIPAA deployment

- **PHI masking:** Ingest pipelines redact/mask; FLS for residual sensitive fields.
- **BAA-compliant hosting:** Encrypted at rest and in transit; no PHI in non-production without synthetic data.
- **Access auditing:** Every access logged; anomaly alerts on bulk downloads.
- **Breach workflow:** Cases with regulatory notification checklist.

### 18. Fleet for 10,000 nodes

- **Tiered Fleet Servers:** Regional HA pairs; agents phone home to nearest.
- **Policy hierarchy:** Base + role-specific integrations; minimal integration set per host type.
- **Auto-install:** Golden AMI/cloud-init with enrollment token from vault.
- **Health SLO:** Alert on >1% agents offline; automated remediation playbooks.

### 19. SRE platform with error budgets and QBR reports

- **SLO data model:** OpenSLO metrics ingested to ES; burn rate alerts in Kibana.
- **Dashboards:** Per-service error budget, tier rollups, quarterly trend for leadership.
- **Reliability reports:** Scheduled Canvas/PDF for QBR with narrative annotations.
- **GitOps:** SLO definitions and dashboards versioned together.

### 20. Disaster recovery in 15 minutes

- **Saved objects backup:** Continuous export to S3/Git; ES snapshot of `.kibana_*` indices.
- **Runbook:** Standby Kibana + ES restore from snapshot; DNS cutover.
- **Config as code:** Full NDJSON + `kibana.yml` in vault; automated import script tested monthly.
- **RTO validation:** Quarterly DR drill measuring import completion time.
- **Alerting parity:** Restore connectors from encrypted backups with key management.

### 21. Real-time business intelligence in Kibana

- **ES|QL / transforms:** Join operational metrics with business KPI indices.
- **External data:** Ingest from warehouses via Logstash/API pollers on schedule.
- **Dashboards:** Side-by-side ops and revenue metrics with shared time filter.
- **Latency:** Near-real-time ingest pipelines for KPI updates under 1 minute.

### 22. CMDB integration for enrichment

- **Enrich processor:** Match `service.name` to CMDB lookup index for owner, tier, dependencies.
- **Runtime fields:** Display business criticality on all operational dashboards.
- **Dependency graph:** CMDB relationships augment APM service map context.
- **Sync job:** Nightly CMDB export to Elasticsearch enrich index.

### 23. Cost observability platform

- **Ingest billing CUR/API data:** Normalize per `service.name`, `k8s.namespace`, `cloud.account`.
- **Correlate with metrics:** Cost per request, cost per GB ingested in Elasticsearch itself.
- **Dashboards:** Top spenders, anomaly alerts on daily cost spikes, rightsizing recommendations.
- **Chargeback:** Transform for monthly team allocation reports.

### 24. Canvas NOC for 50 data centers

- **Regional workpads:** Grid of DC status lights fed by per-DC health indices.
- **Auto-highlight:** Red border when DC crosses error or capacity threshold.
- **Rotation:** Auto-cycle workpads on timer for wall display.
- **Performance:** Pre-aggregated health scores per DC; 60s refresh maximum.

### 25. A/B testing alert thresholds

- **Duplicate rules:** Variant rules tagged `experiment: A` with different thresholds, routed to test Slack channel.
- **Metrics:** Track MTTD, false positive rate, and missed incidents per variant over 30 days.
- **Analysis dashboard:** Compare alert volume and incident outcomes by experiment tag.
- **Promotion:** Winning threshold merged to production rule via Git PR; loser disabled.

---

## Rapid-Fire Questions

### 1. What is the default port for Kibana?

5601.

### 2. What is the purpose of Kibana's `kibana.yml` configuration file?

Primary Kibana server configuration: Elasticsearch connection, security, ports, logging, and feature toggles.

### 3. What does KQL stand for in Kibana?

Kibana Query Language.

### 4. What is the purpose of Kibana's `_g` URL parameter?

Encodes global Kibana state (commonly time range and filters) in shareable URLs.

### 5. What does Kibana's `Discover` view show by default?

Documents from the selected data view within the active time range, with the configured columns and search query.

### 6. What is the purpose of Kibana's `saved_objects` API?

Programmatically export, import, and manage Kibana saved objects (dashboards, visualizations, data views, rules).

### 7. What does the Kibana `Dev Tools` console allow you to do?

Execute Elasticsearch REST API requests and troubleshoot queries directly from the browser.

### 8. What is the purpose of Kibana's `index_pattern` saved object type?

Defines which Elasticsearch indices Kibana searches and the time field (now stored as **data view** saved objects).

### 9. What does Kibana's `Lens` editor replace?

Largely replaces the legacy Visualize editor for new chart creation with a guided, unified experience.

### 10. What is the purpose of Kibana's `spaces` feature?

Multi-tenant workspace isolation for saved objects, access control, and separate landing pages.

### 11. What does Kibana's `Canvas` feature allow you to create?

Pixel-perfect, presentation-style live workpads for wallboards and executive displays.

### 12. What is the purpose of Kibana's `alerting` feature?

Define rules that evaluate data on a schedule and trigger actions (notifications, webhooks, cases) when conditions are met.

### 13. What does Kibana's `APM` feature monitor?

Application performance: transactions, traces, errors, and service dependencies.

### 14. What is the purpose of Kibana's `Fleet` feature?

Central management of Elastic Agents: policies, upgrades, and integrations.

### 15. What does Kibana's `Maps` feature display?

Geospatial data on layered maps using `geo_point` and shape fields.

### 16. What is the purpose of Kibana's `Uptime` feature?

Synthetic monitoring of endpoint availability and response times (now part of **Observability → Synthetics**).

### 17. What does Kibana's `TSVB` stand for?

Time Series Visual Builder.

### 18. What is the purpose of Kibana's `reporting` feature?

Generate and schedule PNG/PDF/CSV exports of dashboards and saved searches.

### 19. What does Kibana's `Timelion` feature provide?

Time series charts using Timelion expression syntax (`.es()` functions).

### 20. What is the purpose of Kibana's `role` management?

Define Elasticsearch/Kibana privileges controlling index access and feature permissions per user.

### 21. What does Kibana's `Profiling` feature analyze?

Production CPU flame graphs via Universal Profiling (eBPF-based stack sampling).

### 22. What is the purpose of Kibana's `Cases` feature?

Collaborative incident tracking with evidence, comments, and external system connectors.

### 23. What does Kibana's `Osquery` integration provide?

Scheduled Osquery SQL packs on endpoints managed through Fleet for security and compliance visibility.

### 24. What is the purpose of Kibana's `Synthetics` feature?

Browser and HTTP synthetic monitors for proactive availability testing from multiple locations.

### 25. What does Kibana's `Security` app provide?

SIEM: detection rules, alerts, timelines, and security analytics for threat detection and response.

### 26. What is the purpose of Kibana's `Index Management` feature?

View and manage Elasticsearch indices, data streams, mappings, and ILM policies.

### 27. What does Kibana's `Snapshot and Restore` feature manage?

Elasticsearch snapshot repositories, snapshots, and restore operations for backup/DR.

### 28. What is the purpose of Kibana's `ILM` policy editor?

Create and manage Index Lifecycle Management policies for hot-warm-cold-delete automation.

### 29. What does Kibana's `Cross-Cluster Replication` feature manage?

Replication of indices between Elasticsearch clusters for DR and geo-redundancy.

### 30. What is the purpose of Kibana's `Watcher` feature?

Legacy alerting/automation in Elasticsearch (largely superseded by Kibana alerting in modern stacks).

### 31. What does Kibana's `Machine Learning` feature provide?

Anomaly detection, forecasting, and data frame analytics on Elasticsearch data.

### 32. What is the purpose of Kibana's `Transforms` feature?

Pivot or latest-doc transforms creating summarized destination indices for analytics.

### 33. What does Kibana's `Rollup Jobs` feature create?

Downsampled rollup indices with reduced granularity for historical metrics storage.

### 34. What is the purpose of Kibana's `Enrich Policies` feature?

Define enrich processors that add lookup data to incoming documents (e.g., CMDB metadata).

### 35. What does Kibana's `Remote Clusters` feature manage?

Configuration of cross-cluster search and cross-cluster replication remote connections.

### 36. What is the purpose of Kibana's `License Management` feature?

View, update, and monitor Elastic license level and expiration.

### 37. What does Kibana's `Stack Monitoring` feature display?

Health and metrics for Elasticsearch, Kibana, Logstash, Beats, and APM components.

### 38. What is the purpose of Kibana's `Upgrade Assistant`?

Pre-upgrade checks, deprecation warnings, and migration guidance for Elasticsearch and Kibana.

### 39. What does Kibana's `Painless Lab` feature provide?

Interactive editor to test Painless scripts used in ingest, runtime fields, and Watcher.

### 40. What is the purpose of Kibana's `Search Profiler`?

Profile Elasticsearch query execution to analyze slow search performance.

### 41. What does Kibana's `Grok Debugger` feature help with?

Test Grok patterns for Logstash/ingest pipeline log parsing before deployment.

### 42. What is the purpose of Kibana's `Logstash Pipelines` management?

View and manage centrally configured Logstash pipelines (when using Elastic Agent/Logstash management).

### 43. What does Kibana's `Beats Central Management` feature provide?

Legacy centralized Beat configuration (largely replaced by Fleet for Elastic Agent).

### 44. What is the purpose of Kibana's `Advanced Settings`?

UI-level Kibana configuration overrides (timeouts, formatting, default routes) per space or globally.

### 45. What does Kibana's `Saved Objects` management allow you to do?

Browse, export, import, and delete saved objects; resolve reference conflicts during migration.

### 46. What is the purpose of Kibana's `API Keys` management?

Create and revoke API keys for programmatic access with specific role restrictions.

### 47. What does Kibana's `Audit Logs` feature track?

User authentication, authorization decisions, and administrative actions for security compliance.

### 48. What is the purpose of Kibana's `Session Management`?

Control user session idle timeout and lifespan for security compliance.

### 49. What does Kibana's `Maintenance Windows` feature provide?

Scheduled periods that suppress alert notifications during planned maintenance.

### 50. What is the purpose of Kibana's `Action Connectors` feature?

Store and manage integration credentials for alert actions (Slack, PagerDuty, email, webhooks).
