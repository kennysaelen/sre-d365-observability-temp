---
name: d365-forms-form-drilldown-analysis-v2
description: Analyzes one exact Dynamics 365 page or form name in customer Application Insights pageViews or Log Analytics AppPageViews. Use for a bounded page-duration and observed-usage drilldown with hourly and daily trends, browser/client/component context, and privacy-safe slow samples. Compatible instrumentation is required; this is not SQL, X++, browser trace, or weekly regression analysis.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
  - PlotLineChart
  - PlotBarChart
---

# D365 page/form drilldown

## Activation and boundaries

Use for **one exact requested instrumented page/form name** in any Dynamics
365 application with compatible customer page-view telemetry. A name alone
does not prove that it measures form opening. Say **reported page-view
duration** until emitter documentation confirms the measured start/end
semantics.

This standalone Azure SRE Agent package does not use Microsoft-internal F&O
telemetry, local observability agents, Copilot Studio, or internal browser
trace deep-dives. Do not deploy anything or update another agent. Do not
perform environment-wide form discovery, detailed causal tracing, or a
previous-period comparison here. Recommend the forms usage overview for an
unknown name and the weekly regression skill for a previous-period question.

## Inputs and defaults

- Authorized source and confirmed application-resource/environment boundary;
  inherit those selections from the originating investigation when present.
- One nonempty, exact, stable page/form name. Never fuzzy-match, expand to
  similar names, or treat an empty name as a wildcard.
- Exact role/component when needed; an explicitly selected unknown role is
  valid (`_roleFilterEnabled=true`, `_roleName=""`).
- Explicit UTC interval if supplied; otherwise freeze the last seven days
  ending at the observation time. Use `[start, end)`. Never widen automatically.
- Hourly bins (`1h`) by default, plus a separate UTC daily (`1d`) summary.
  A custom bin width requires an explicit recorded choice.
- Optional positive duration screening threshold; default 3,000 ms, explicitly
  not a product SLA. Default 20 slow samples and 20 known dimension buckets.
- Default per-result ceiling: 5,000 rows and 20,000 cells, or smaller host
  limits. Preflight bin counts, dimension cardinality, and returned columns.

If inputs are ambiguous, request the missing choice. If no interaction is
possible, return a bounded blocked/partial report rather than guessing.

## Workflow

1. **Read the source contract.** Load `references/page-view-source.md`.
   Verify the table, required columns, duration encoding, optional dimensions,
   resource identity, and environment mapping. Select exactly one adapter.
   Prefer `pageViews` for Application Insights tools and `AppPageViews` for
   Log Analytics tools. For `kusto_query`, inspect the target metadata first.
   Never fall back because of an empty result, denied access, or timeout.
2. **Freeze scope and time.** Inherit confirmed boundaries. If unresolved,
   use the contract's bounded discovery and coverage queries; do not combine
   resources or environments. Align any query API time-range argument with
   these frozen bounds, using its actual schema, not an invented parameter.
   Environment conflicts and missing environment IDs are not attributable
   to a selected environment. Preserve the known synthetic filter policy.
3. **Resolve page identity and roles.** Validate that the page name and
   dimension labels are stable, nonsensitive telemetry names. With the exact
   page filter retained, run Q1 with role filtering disabled before freezing
   the final role. Count role buckets before returning their list. If the
   page name occurs in multiple roles, require one exact choice; never merge
   them. Missing role is a separate unknown component, not an alias for all
   components. If only one role is observed, freeze that role explicitly and
   record that it is an observed, not exhaustive, identity determination.
4. **Assemble every query locally.** Follow `references/queries.md`. Each
   submitted query must contain Parameters, exactly one Adapter,
   Normalization, Scope, its explicitly defined local lets, and one terminal.
   These are not installed database functions. Replace and escape every
   token. Guard against whitespace-only page names, unconfirmed scope, invalid
   dates, invalid thresholds, or invalid limits before submission.
5. **Validate and preflight.** Invoke `ValidateQuery` where supported for each
   complete assembled query before executing it. Do not describe local text
   checks as service-side compilation. Run Q2; confirm output budgets. Stop
   on truncation. Split oversized trends into disjoint output bucket ranges
   with the same global source bounds, without splitting a bucket's samples.
   Do not average sliced percentiles or sum grouped distinct-user counts.
6. **Collect full-population evidence first.** Run Q3 for the entire selected
   page/role cohort before top-N lists. Report retained observed views,
   positive-duration count, mean/median/p95/max, duration exclusions, sampling
   indicators, observed distinct user/session counts and missing-ID rows.
   Missing columns make identity metrics unavailable, not evidence of zero
   people. Use only aggregate identifier handling; never return identifiers.
7. **Preserve trends and gaps.** Run Q4 twice, hourly and daily. Retain
   partial edge buckets and successful empty buckets. Empty successful
   buckets have observed counts 0 and null duration, not confirmed actual
   zero use. Failed/truncated slices are unavailable; do not fill them with
   zeros. Identify known retention gaps and collection outages separately.
8. **Add bounded context.** Run Q5 separately for browser, client type, role
   instance, app version, and SDK version. Show missing buckets and full
   totals alongside top-N buckets. The exact selected role and Q1 results
   provide component context; do not silently recombine roles. RoleInstance
   is an instrumentation dimension, not necessarily an AOS or server host.
   Run Q6 for at most 20 sanitized slow observations; never output IDs, raw
   URLs, property bags, IPs, tenant IDs, record titles, or sensitive labels.
9. **Interpret proportionately.** Apply `references/interpretation.md`.
   Distinguish a few slow observations from repeated within-window
   distribution changes. Do not claim a weekly regression, platform
   degradation, SQL blocking, X++ cost, browser cause, or business throughput
   from page durations. Mark findings FACT, CORRELATED, or HYPOTHESIS with
   justified confidence and evidence counts.
10. **Render a Markdown report.** Use `assets/report-template.md`. Plot time
    on the X axis and milliseconds on the duration Y axis; plot observed
    views separately. Dimension charts use dimension labels versus either
    observed views or explicitly labeled milliseconds. Preserve null gaps,
    distinguish partial buckets, and escape labels. If chart tools are
    unavailable or would exceed their budgets, provide a bounded table and
    an honest unavailable reason. Do not invent charts or telemetry results.

## Completion checks

The report must identify source/schema provenance, exact page/role and scope,
UTC bounds and bins, duration semantics, full population and exclusions,
identity and dimension coverage, sampling and collection limitations, daily
and hourly trends, safe samples, evidence confidence, and concrete next steps.
Report a no-match or unavailable outcome precisely without widening scope.
No live query or deployment is required to author or review this package.