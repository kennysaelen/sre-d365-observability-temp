---
name: d365-forms-regression-analysis-v2
description: Compares reported Dynamics 365 page-view duration across completed Monday-Sunday UTC weeks using customer Application Insights or Log Analytics telemetry. Preserves all observed page and role keys, compares every adjacent week, and distinguishes heuristic median and p95 signals from missing samples and noncomparable collection.
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

# D365 forms weekly regression analysis

## Purpose and boundary

Use this standalone Azure SRE Agent skill to investigate weekly changes in
**reported page-view duration** in customer-owned telemetry. The only physical
page-view sources are `pageViews` and `AppPageViews`. This is not an internal
Microsoft F&O telemetry workflow or a Copilot Studio agent. Do not change agent
configuration, deploy resources, or infer server-side causes from page views.
Do not call the metric "form-open time" without confirmed emitter semantics.

## Required inputs and time model

1. Confirm one authorized application resource, one environment, and duration
   type/units using [the source contract](references/page-view-source.md).
   Keep exact `(PageName, RoleName)` keys separate, including the unknown-role
   bucket. An explicit role filter uses `_roleFilterEnabled`, including when
   the selected role is the empty/unknown role. Do not use a role as an
   environment identifier.
2. Freeze one UTC observation clock for the entire run. By default, the end is
   `startofweek(observedAtUtc - 1d) + 1d`, the most recent Monday at 00:00 UTC.
   The start is end minus 28 days. Query `[start, end)`: four completed
   Monday-Sunday weeks, three adjacent comparisons. The current partial week
   is excluded, including when the observation clock is on Sunday.
3. Honor an explicit interval only if it contains 2-12 completed UTC weeks,
   with both boundaries Monday at 00:00 and end no later than that Monday
   anchor. Do not silently round an explicit interval or replace it with the
   default. For a partial-week request, ask for an alternative full-week
   interval or report the requested partial interval separately without
   applying or labeling this full-week comparison.
4. Default heuristics: at least 30 retained positive-duration samples in
   **both** weeks, absolute median delta at least 500 ms, and relative median
   delta at least 20%. Both delta tests must pass. p95 has separate thresholds,
   defaulting to 500 ms and 20% with the same sample eligibility. All are
   configurable, disclosed heuristics, not SLOs or statistical significance.

## Workflow

1. Read [source selection, normalization, scope, privacy, and budgets](references/page-view-source.md).
   Probe the tool-preferred source, verify the schema and duration encoding,
   and resolve environment ambiguity before interpreting results. A missing
   duration column permits coverage reporting, not fabricated duration data.
2. Freeze literal UTC parameters and construct the prefix exactly as specified.
   Make any tool/API time-range argument cover the entire interval. A portal
   default must not clip earlier weeks. Validate each assembled query with
   `ValidateQuery` when supported; disclose when unavailable. A failed,
   truncated, or timed-out query is unavailable, never zero observations.
3. Run the window gate, source coverage queries, and cardinality preflight in
   [queries](references/queries.md). Enforce 5,000 grouped rows / 20,000 chart
   cells or smaller host limits before constructing page-week/pair grids.
   Partition by complete disjoint page/role keys if necessary, never by
   positive-duration keys or a displayed top-N list. Reconcile every partition.
4. Build the union of keys observed anywhere in the interval, including pages
   having only missing, zero, or invalid durations. Cross each key with every
   week. Preserve counts separately from nullable median/p95. Then compare
   every adjacent week, not overlapping rolling averages or each week versus
   a shared baseline. Include absent-both, newly observed, disappeared,
   no-positive-duration, and insufficient-sample outcomes.
5. Assess collection and cohort comparability per page/pair. Check retention,
   outages, ingestion/late-arrival limitations, sampling, schema/units,
   emitter/SDK and application versions, browser/client mix, missing roles,
   and positive-duration coverage. Successful queries and observed traffic do
   not establish continuous collection. Record evidence using the explicit
   evidence table; its default is **Unknown**, not Comparable.
6. Produce median and tail classifications independently. Strong directional
   labels require Comparable evidence. Unknown evidence permits only clearly
   exploratory potential signals. NotComparable evidence blocks directional
   claims even when arithmetic thresholds pass. Do not call an absence of
   flagged regressions "stable."
7. Reconcile the whole population: all page/pair statuses, all exclusions,
   all weeks, and all query partitions. Focus the narrative on the last pair
   (latest completed week versus its predecessor), but retain the full weekly
   timeline and all adjacent-pair summaries. Compute any application-level
   duration trend directly from the full raw positive sample cohort, never
   by averaging per-page percentages, medians, or p95 values.
8. Rank the complete eligible population deterministically before limiting
   display. Publish full counts and displayed counts, threshold settings,
   unavailable slices, and comparability limits. If any partition failed,
   global ranks and totals are unavailable; a partial diagnostic must be
   labeled partial, not a whole-population result.
9. Render [the report template](assets/report-template.md) and apply
   [interpretation rules](references/interpretation.md). Charts are optional:
   use `PlotLineChart` for weekly median/p95 and `PlotBarChart` for final-pair
   deltas, preserving units, UTC dates, null gaps, and exploratory labels.
   If plotting is unavailable or rejects nulls, use the equivalent Markdown
   tables; do not interpolate or replace missing duration with zero.

## Completion criteria

- Explicit source/environment scope and confirmed units; exact frozen bounds.
- Four weekly rows and three adjacent comparisons by default, without the
  current partial week; explicit full-week overrides handled without rounding.
- Every observed page key and every adjacent pair accounted for, including
  duration-less and absent populations, with full collection/exclusion counts.
- Median and p95 changes with both sample counts and separate eligibility,
  evidence, and direction labels; positive change means slower.
- Whole-population status rollup, latest-pair focus, full timeline, and no
  top-N-based conclusions or "no regressions means stable" claim.
- Query/validation/truncation state, comparability evidence, privacy-safe
  outputs, and actionable next checks rather than unsupported root causes.