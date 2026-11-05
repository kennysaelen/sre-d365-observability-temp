# Dynamics 365 page-view usage overview

## Scope and short assessment

| Field | Value |
|---|---|
| Customer application resource / selected adapter | |
| Dynamics application / environment evidence | |
| UTC interval [start, end) / observed time | |
| Reported duration meaning and unit conversion | |
| Page identity / role filter / synthetic policy | |
| Bucket size / query and display limits | |

## Collection coverage and KPIs

| Metric | Value / limitation |
|---|---|
| Retained page views / named views / missing names | |
| Observed page-role keys | |
| Known user IDs / views with IDs / identity limitation | |
| Known session IDs / views with IDs | |
| Positive / zero / missing / invalid duration rows | |
| Mean / median / p95 / maximum reported duration, seconds | |
| Sampling indicated / unknown weight rows | |
| Missing role / excluded synthetic rows | |
| Resource-level missing/conflicting environments, not attributed to selection | |
| Query errors, truncation, ingestion/retention limitations | |

Use unavailable, not zero, for unavailable metrics. No populated identity
columns means identity counts are unavailable, not no users. Do not expose IDs.

## Most-used pages

Optional top-usage bar chart using the same retained-view counts.

| Page | Role component | Observed views | Share of all observed views | Positive-duration N | Median s | p95 s | Identity coverage |
|---|---|---:|---:|---:|---:|---:|---|

State full key count versus displayed keys, remaining named views, and missing-
name views. Shares use all selected observed views as the denominator.

## Slow-page candidates

Default ranking: p95 descending, then median; minimum 20 positive samples.
State overrides and number of excluded low-sample/unavailable keys.

| Page | Role | Views / positive N | Mean s | Median s | p95 s | Maximum s | Missing/zero/invalid duration | Coverage |
|---|---|---|---:|---:|---:|---:|---|---|

This is not a regression or causal ranking.

## Hourly activity and reported duration

Show volume and duration in separate charts/panels when supported. Use UTC
dates, null duration gaps, and visible partial-edge buckets.

| Bucket UTC | Partial? | Retained views | Positive-duration N | Median s | p95 s |
|---|---|---:|---:|---:|---:|

Chart unavailability must be explicit. If the displayed table is abbreviated,
state full row count and displayed range; compute KPIs from the full population.

## Findings and focused follow-up

| Finding | FACT / CORRELATED / HYPOTHESIS | Confidence and evidence | Next action |
|---|---|---|---|

Weekly shifts: `d365-forms-regression-analysis-v2`.
One page: `d365-forms-form-drilldown-analysis-v2`.
Do not force an RCA conclusion or create HTML artifacts.
