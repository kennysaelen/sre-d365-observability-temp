# Completed-week page-view duration comparison

## Scope and execution

| Field | Value |
|---|---|
| Authorized application resource / selected environment | |
| Source family and query tool | |
| Duration meaning, confirmed type, and units | Reported page-view duration; ms |
| Frozen observation clock (UTC) | |
| Half-open analysis interval (UTC) | [start, end) |
| Most recent completed-week boundary | Monday 00:00 UTC |
| Completed weeks / adjacent comparisons | 4 / 3 by default |
| Exact page and role filters; unknown-role filter enabled? | |
| Known-synthetic exclusion | |
| Positive sample minimum in both weeks | 30 by default |
| Median absolute / relative thresholds (both required) | 500 ms / 20% by default |
| p95 absolute / relative thresholds (both required) | 500 ms / 20% by default |
| Query validation / runtime execution / truncation | |
| Tool time-range argument, retention, ingestion maturity | |
| Partition count / successful / unavailable | |
| Result-row / chart-cell budget and actual preflight | |

State that current partial week is excluded. For any separately requested
partial interval, add a separate section clearly labeled partial and do not
mix it into the completed-week conclusions.

## Coverage and exclusions

| Cohort / measure | Count | Meaning / limitation |
|---|---:|---|
| Resource rows before environment/page/role selection | | Not all attributed to selected environment |
| Missing / conflicting environment rows | | Resource-scope diagnostics |
| Rows excluded by combined scope selection | | Includes environment, page, and role selectors |
| Scoped views before synthetic filtering | | |
| Known synthetic views / excluded views | | Blank synthetic metadata does not prove human traffic |
| PageRows observed views | | Includes missing page names |
| Missing page-name views | | Coverage only; excluded from named-page comparisons |
| NamedPages observed views | | Page-comparison cohort |
| Positive / zero / missing-unparseable / invalid durations | | Retained events, not sampling-weight estimates |
| Unknown role / unknown weight / sampling-indicated views | | |
| Distinct exact page/role keys | | Includes duration-less keys |
| Expected / actual key-week cells | | Keys times completed weeks |
| Expected / actual key-pair cells | | Keys times (weeks minus one) |
| Unavailable queries or partitions | | Unavailable is not zero |

## Weekly whole-population timeline

These rows use all PageRows, including unnamed pages. Percentiles use raw
positive retained samples only. Do not average per-page metrics.

| UTC week start | Exclusive week end | Observed views | Positive N | Zero | Missing/unparseable | Invalid | Missing names | Median ms | p95 ms | Observed active hours / 168 | Collection caveat |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|

An all-empty week has zero observed counts and null duration percentiles.
Active hours do not prove collection continuity.

## Full-population adjacent-pair accounting

Include every actual joint status for every adjacent pair, with an explicit
total of zero for pairs with no named keys. Totals must reconcile to all keys
times all pairs. Show comparable/unknown/noncomparable evidence counts.

| Previous UTC week | Current UTC week | Median status | p95 status | Comparability | Page/role pairs |
|---|---|---|---|---|---:|

| Current UTC week | Expected pairs | Accounted pairs | Reconciled? | Partition completeness |
|---|---:|---:|---|---|

## Latest completed week versus prior week

Rank the complete final-pair population before limiting display. Keep
Comparable and exploratory Unknown results in distinct sections. Publish
candidate totals and displayed counts. Positive deltas mean slower.

| Rank | Exact page | Exact role / unknown | Previous positive N | Current positive N | Previous median ms | Current median ms | Delta ms | Delta % | Median status | Comparability / evidence |
|---:|---|---|---:|---:|---:|---:|---:|---:|---|---|

### Tail-only and combined p95 signals

| Rank | Exact page | Exact role / unknown | Previous positive N | Current positive N | Previous p95 ms | Current p95 ms | Delta ms | Delta % | p95 status | Median status | Evidence |
|---:|---|---|---:|---:|---:|---:|---:|---:|---:|---|---|

### Unassessed populations

| Status | Full count | Examples (bounded, not full population) | Reason / next check |
|---|---:|---|---|
| Absent in both weeks | | | |
| Newly observed / no longer observed | | | Not proof of real-world start/stop |
| No positive durations (both / previous / current) | | | |
| Insufficient positive samples | | | |
| Collection not comparable | | | |
| Comparability unknown | | | Exploratory only |

## Per-page weekly timeline

Retain the full grid as budget permits; otherwise provide reconciled disjoint
partitions and identify the displayed subset. Do not omit duration-less keys
from population accounting.

| Exact page | Exact role / unknown | UTC week start | Observed views | Positive N | Zero | Missing/unparseable | Invalid | Median ms | p95 ms |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|

## Comparability and version/client evidence

| Page/role or explicitly labeled application scope | Adjacent pair | Decision | Retention / gap evidence | Sampling / positive coverage | App/SDK versions | Browser/client mix | Limits |
|---|---|---|---|---|---|---|---|

Metadata rows are distributions of retained events, not causal attribution.
Preserve unknown metadata as unknown. Do not present app-wide distributions
as proof that every page cohort is comparable.

## Optional charts or equivalent Markdown

- Weekly median and p95 line chart: UTC week start on x, ms on y, exact
  page/role and metric in series, with positive sample counts in the table.
  Null durations are gaps, not zeros or interpolated points.
- Latest-pair median or p95 delta bar chart: signed ms, with exact page/role,
  metric and comparable/exploratory status in labels. Never combine median
  and p95 under one unlabeled value axis.
- If chart tools are unavailable, fail, truncate, or cannot preserve nulls,
  the weekly and delta tables above are the complete fallback. Charts do not
  replace whole-population accounting.

## Assessment and next steps

| Finding | Fact / correlation / hypothesis | Evidence / limits | Next check |
|---|---|---|---|

Use "no threshold signal among N eligible comparable pairs" when supported,
not "stable" or "no regressions." State unassessed populations alongside that
sentence. Thresholds are heuristic, not SLOs or statistical significance.
