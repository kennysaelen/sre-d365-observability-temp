# Usage overview queries

## Assembly

Read [page-view-source.md](page-view-source.md). All terminal queries below
require **source Parameters + exactly one adapter + Normalization + Scope +
Overview parameters**, then the stated additional datasets and one terminal.
Never submit a terminal alone or assume a local `let` is a stored table.
Use empty `_pageName` for a whole-scope overview; disclose any optional role
filter. Confirm scope booleans before analysis, not by silently executing
the false-default empty dataset.

Run the source discovery and three source coverage terminals first. This
file's KPI, page aggregates, and timeline operate on `PageRows` or `NamedPages`;
those cohorts differ when page names are missing.

## Overview parameters

```kusto
let _bin=1h;
let _displayLimit=20;
let _minimumDurationSamples=20;
```

Validate positive whole-minute `_bin`, positive integer sample minimum, and
display limit 1..100. Freeze parameters across all queries. Do not silently
widen the analysis interval or drop low-volume pages from usage counts.

## KPI terminal

Assembly: source prefix + Overview parameters + this terminal.

```kusto
PageRows
| summarize ObservedViews=count(),
    NamedViews=countif(isnotempty(trim(@"\s+",PageName))),
    PositiveDurationViews=countif(DurationState == "Positive"),
    ObservedUserIds=count_distinctif(UserKey,isnotempty(UserKey)),
    ObservedSessionIds=count_distinctif(SessionKey,isnotempty(SessionKey)),
    ViewsWithUserId=countif(isnotempty(UserKey)),
    ViewsWithSessionId=countif(isnotempty(SessionKey)),
    MeanMs=avg(DurationMs),MedianMs=percentile(DurationMs,50),
    P95Ms=percentile(DurationMs,95),MaxMs=max(DurationMs),
    DurationSumMs=sum(DurationMs)
| extend MeanSeconds=iff(PositiveDurationViews > 0,MeanMs/1000.0,real(null)),
    MedianSeconds=iff(PositiveDurationViews > 0,MedianMs/1000.0,real(null)),
    P95Seconds=iff(PositiveDurationViews > 0,P95Ms/1000.0,real(null)),
    MaxSeconds=iff(PositiveDurationViews > 0,MaxMs/1000.0,real(null)),
    SumObservedDurationSeconds=iff(PositiveDurationViews > 0,DurationSumMs/1000.0,real(null))
| project-away MeanMs,MedianMs,P95Ms,MaxMs,DurationSumMs
```

If no user/session identifiers are populated, mark the identity metric
unavailable rather than presenting a zero as "zero users." The zero count of
known identifiers and the coverage counts may still be stated literally.
The duration sum is additive measured durations, not wall-clock wait, server
load, or time lost by all users. Do not derive it from a top-N ranking.

## Page aggregates dataset

Include this definition for the next three terminals.

```kusto
let TotalObservedViews=toscalar(PageRows | summarize count());
let PageStats=NamedPages
    | summarize ObservedViews=count(),
        PositiveDurationViews=countif(DurationState == "Positive"),
        ZeroDurationViews=countif(DurationState == "Zero"),
        MissingDurationViews=countif(DurationState == "MissingOrUnparseable"),
        InvalidDurationViews=countif(DurationState == "Invalid"),
        SamplingIndicatedViews=countif(ValidWeight and Weight > 1),
        UnknownWeightViews=countif(not(ValidWeight)),
        ObservedUserIds=count_distinctif(UserKey,isnotempty(UserKey)),
        ObservedSessionIds=count_distinctif(SessionKey,isnotempty(SessionKey)),
        ViewsWithUserId=countif(isnotempty(UserKey)),
        ViewsWithSessionId=countif(isnotempty(SessionKey)),
        MeanMs=avg(DurationMs),MedianMs=percentile(DurationMs,50),
        P95Ms=percentile(DurationMs,95),MaxMs=max(DurationMs)
        by PageName,RoleName
    | extend ShareOfAllObservedViewsPct=iff(TotalObservedViews > 0,
        100.0*ObservedViews/TotalObservedViews,real(null)),
        MeanSeconds=iff(PositiveDurationViews > 0,MeanMs/1000.0,real(null)),
        MedianSeconds=iff(PositiveDurationViews > 0,MedianMs/1000.0,real(null)),
        P95Seconds=iff(PositiveDurationViews > 0,P95Ms/1000.0,real(null)),
        MaxSeconds=iff(PositiveDurationViews > 0,MaxMs/1000.0,real(null))
    | project-away MeanMs,MedianMs,P95Ms,MaxMs;
```

Percentages include missing-name views in the denominator. Known page shares
therefore need not sum to 100%. Group-level distinct IDs must not be summed
to recover scope-wide identities.

## Population and display coverage terminal

Assembly: source prefix + Overview parameters + Page aggregates dataset.

```kusto
PageStats
| summarize TotalPageRoleKeys=count(),NamedViews=sum(ObservedViews),
    DurationRankEligibleKeys=countif(PositiveDurationViews >= _minimumDurationSamples),
    LowSampleOrUnavailableKeys=countif(PositiveDurationViews < _minimumDurationSamples)
```

This is a full-population count, not the size of a limited result. If a full
page table is requested, preflight against the 5,000-row result budget in the
source contract. The default top lists need not export every grouped row.
Reconcile returned usage-top counts and remaining named views, plus missing
names, to the full-scope total. Report slow-rank excluded key count separately.

## Top usage terminal

Assembly: source prefix + Overview parameters + Page aggregates dataset.

```kusto
PageStats
| order by ObservedViews desc,PageName asc,RoleName asc
| take _displayLimit
```

## Slow-page terminal

Assembly: source prefix + Overview parameters + Page aggregates dataset.

```kusto
PageStats
| where PositiveDurationViews >= _minimumDurationSamples
| order by P95Seconds desc,MedianSeconds desc,PageName asc,RoleName asc
| take _displayLimit
```

No eligible keys means no sample-qualified duration ranking, not no slow
pages. Do not hide available low-sample data when the user asks for it; label
it descriptive/low-sample, outside the qualified ranking.

## Timeline preflight terminal

Assembly: source prefix + Overview parameters; no PageStats needed.

```kusto
let ExpectedBins=tolong(ceiling((_endTime-bin(_startTime,_bin))/_bin));
print TimeBuckets=ExpectedBins,FitsBudget=ExpectedBins <= 5000
```

Do not run an oversized grid after failed preflight. Split at bucket boundaries
with frozen original start/end for partial-edge labeling; never average
percentiles when merging slices. Failed slices remain unavailable.

## Timeline datasets

```kusto
let Buckets=range BucketUtc from bin(_startTime,_bin) to _endTime step _bin
    | where BucketUtc < _endTime
    | extend PartialBucket=BucketUtc < _startTime or BucketUtc+_bin > _endTime;
let ObservedBuckets=PageRows
    | summarize ObservedViews=count(),PositiveDurationViews=countif(DurationState == "Positive"),
        MedianMs=percentile(DurationMs,50),P95Ms=percentile(DurationMs,95)
        by BucketUtc=bin(EventTime,_bin);
```

## Timeline terminal

Assembly: source prefix + Overview parameters + Timeline datasets.

```kusto
Buckets
| join kind=leftouter ObservedBuckets on BucketUtc
| extend ObservedViews=coalesce(ObservedViews,0),
    PositiveDurationViews=coalesce(PositiveDurationViews,0)
| project BucketUtc,PartialBucket,ObservedViews,PositiveDurationViews,
    MedianSeconds=iff(PositiveDurationViews > 0,MedianMs/1000.0,real(null)),
    P95Seconds=iff(PositiveDurationViews > 0,P95Ms/1000.0,real(null))
| order by BucketUtc asc
```

Volume includes unnamed views and matches scope KPIs. No duration interpolation
or zero-filled duration lines. Sum of observed bucket counts must equal
ObservedViews for a complete, stable query snapshot. Late ingestion between
queries can break reconciliation: state the observation/retrieval times and
rerun a bounded consistent snapshot where supported; do not change totals by
hand or claim transactional consistency across separate query calls.
