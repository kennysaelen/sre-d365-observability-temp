# Exact page/form drilldown queries

## Assembly, privacy, and execution contract

Read `page-view-source.md` first. **Every terminal below is a query suffix**,
not a standalone database-table reference. For every submission concatenate
the following in this exact order:

1. The complete **Parameters** block from `page-view-source.md`, with all
   values resolved and safely encoded as KQL literals.
2. Exactly one complete **Adapter A** or **Adapter L** block.
3. The complete **Normalization** block.
4. The complete **Scope** block.
5. Exactly one entire Q1a, Q1b, Q2, Q3, Q4, Q5, or Q6 block below, including
   all local lets in that block.

Thus `NamedPages`, `Scoped`, and every other dataset are explicit local
`let` bindings in each final query. Do not submit a suffix alone, reference
a previously executed query's variables, or assume installed functions.
The source contract's separately documented discovery/coverage terminals
also require their stated complete prefix.

Require a nonempty, non-whitespace `_pageName`, confirmed source/environment,
start < end <= observation time, and safe labels before all queries here.
The extra exact filter in each block prevents the source contract's optional
empty selector from becoming an unrestricted drilldown. Page and role
matching are case-sensitive exact equality, not `contains` or `=~`.

For Q1 only, use `_roleFilterEnabled=false` before final role selection.
After Q1, use `_roleFilterEnabled=true` with the exact selected `_roleName`
for all other queries. Empty role is an explicit unknown-role choice. If a
caller supplies a role, retain it as the intended choice and confirm it
against discovery without silently substituting another component.

Defaults in suffixes are editable recorded parameters, not inferred values:
`_slowThresholdMs=3000.0`, `_topDimensions=20`, `_sampleLimit=20`.
Only positive finite thresholds and integer limits from 1 through 20 are
permitted. `_binSize` must be 1h or 1d for this default workflow.
Validate complete assembled queries, not these fragments. Inspect the actual
query tool schema and apply its time-range argument to the same frozen window.

User/session identifiers are used only for aggregate distinct counts and
aggregate missing-value coverage after the source adapter. Their values never
appear in output, row filters, group keys, joins, samples, or charts. Missing
rows use `countif(isempty(...))`, not a fabricated empty-string user or an
expensive serialized per-row identifier. With absent identity columns, counts
of known IDs compute to 0;
report the corresponding identity metric as unavailable based on schema.

Names and dimension strings must be verified nonsensitive before display;
omit unsafe dimensions rather than exposing them to discover their content.
Samples omit page names and all identifiers entirely. Markdown/chart labels
must be escaped. No raw property bag or URL is projected by these suffixes.

## Q1a: exact-page role cardinality, before final role freeze

Assembly: Parameters (role filtering false) + Adapter + Normalization + Scope
+ this complete block. Discovery uses `Scoped` before the synthetic filter,
so a component is not silently hidden merely because its retained events are
known synthetic. Only roles inside the already confirmed scope are counted.

```kusto
let ExactPageRoles=Scoped
    | where not(_roleFilterEnabled)
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | summarize ObservedViews=count(),
        EligibleObservedViews=countif(not(_excludeKnownSynthetic) or not(KnownSynthetic))
        by RoleName;
ExactPageRoles
| summarize RoleBuckets=count(),ObservedViews=sum(ObservedViews),
    EligibleObservedViews=sum(EligibleObservedViews)
| extend RoleListCells=RoleBuckets*4
```

If 0, return no matches in this scope and interval; do not widen. If more
than 1, require an exact component choice before final analytics. Unknown
role is one bucket. For Q1b, enforce both 5,000 rows and 20,000 cells or lower
host limits. If over budget, ask for a specific role and verify that choice;
never choose from a truncated list or merge roles.

## Q1b: complete role list when preflight permits

Assembly: Parameters (role filtering false) + Adapter + Normalization + Scope
+ this complete block.

```kusto
let ExactPageRoles=Scoped
    | where not(_roleFilterEnabled)
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | summarize ObservedViews=count(),
        EligibleObservedViews=countif(not(_excludeKnownSynthetic) or not(KnownSynthetic))
        by RoleName;
ExactPageRoles
| extend MissingRole=isempty(RoleName)
| project RoleName,MissingRole,ObservedViews,EligibleObservedViews
| order by ObservedViews desc,RoleName asc
```

## Q2: frozen-cohort cardinality and result-budget preflight

Assembly: Parameters (role filtering true) + Adapter + Normalization + Scope
+ this complete block. The whole result is one row. The dimension expressions
count an empty bucket separately without merging it with a literal label.

```kusto
let _binSize=1h;
let _topDimensions=20;
let _sampleLimit=20;
let Focus=NamedPages
    | where _roleFilterEnabled
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | where RoleName == _roleName;
Focus
| summarize ObservedViews=count(),PositiveDurationViews=countif(DurationState == "Positive"),
    BrowserBuckets=count_distinctif(Browser,isnotempty(Browser))+tolong(countif(isempty(Browser))>0),
    ClientTypeBuckets=count_distinctif(ClientType,isnotempty(ClientType))+tolong(countif(isempty(ClientType))>0),
    RoleInstanceBuckets=count_distinctif(RoleInstance,isnotempty(RoleInstance))+tolong(countif(isempty(RoleInstance))>0),
    AppVersionBuckets=count_distinctif(AppVersion,isnotempty(AppVersion))+tolong(countif(isempty(AppVersion))>0),
    SDKVersionBuckets=count_distinctif(SDKVersion,isnotempty(SDKVersion))+tolong(countif(isempty(SDKVersion))>0)
| extend TrendBuckets=tolong(ceiling((_endTime-bin(_startTime,_binSize))/_binSize))
| extend TrendCells=TrendBuckets*14,
    MaximumDimensionResultRows=_topDimensions+1,
    MaximumDimensionResultCells=(_topDimensions+1)*17,
    MaximumSampleRows=_sampleLimit,MaximumSampleCells=_sampleLimit*6
```

Repeat with `_binSize=1d` for daily output sizing. A non-aligned seven-day
hourly interval can yield 169 buckets, not 168. Retain partial edges.
The dimensional cardinalities are full-cohort values, not sampled lists.
Top-N output is bounded after full aggregation; do not request an unbounded
cross-product of dimensions. High cardinality may still exceed execution
limits: disclose failures and request an explicitly narrower scope rather
than processing a partial first page. Q3 is one row/19 cells; Q4 has 14
columns; Q5 has 17 columns; Q6 has 6 columns. Honor all host limits.

## Q3: full population, duration and identity coverage

Assembly: Parameters (role filtering true) + Adapter + Normalization + Scope
+ this complete block. Run before every ranking or sample. Identifier
values remain inside aggregate expressions and are not returned.

```kusto
let _slowThresholdMs=3000.0;
let Focus=NamedPages
    | where _roleFilterEnabled
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | where RoleName == _roleName;
Focus
| summarize ObservedViews=count(),
    PositiveDurationViews=countif(DurationState == "Positive"),
    ZeroDurationViews=countif(DurationState == "Zero"),
    MissingDurationViews=countif(DurationState == "MissingOrUnparseable"),
    InvalidDurationViews=countif(DurationState == "Invalid"),
    MeanMs=avg(DurationMs),MedianMs=percentile(DurationMs,50),
    P95Ms=percentile(DurationMs,95),MaximumMs=max(DurationMs),
    SlowViews=countif(DurationMs>_slowThresholdMs),
    ObservedUsers=count_distinctif(UserKey,isnotempty(UserKey)),
    ObservedSessions=count_distinctif(SessionKey,isnotempty(SessionKey)),
    MissingUserViews=countif(isempty(UserKey)),
    MissingSessionViews=countif(isempty(SessionKey)),
    SamplingIndicatedViews=countif(ValidWeight and Weight>1),
    UnknownWeightViews=countif(not(ValidWeight)),
    FirstObserved=min(EventTime),LastObserved=max(EventTime)
| extend ThresholdMs=_slowThresholdMs,
    MeanMs=iff(PositiveDurationViews > 0,MeanMs,real(null)),
    MedianMs=iff(PositiveDurationViews > 0,MedianMs,real(null)),
    P95Ms=iff(PositiveDurationViews > 0,P95Ms,real(null)),
    MaximumMs=iff(PositiveDurationViews > 0,MaximumMs,real(null))
```

Reconcile duration-state counts to `ObservedViews`. Positive `DurationMs`
alone enters statistics because other durations are null in the source
normalization. Empty input still yields a count row with null durations.
Do not translate undefined real values returned by a client into zero.
Use schema evidence to distinguish absent identity columns from observed
empty identifiers. No sum of per-bucket distinct counts replaces Q3.

For resource/environment conflict coverage and before-synthetic counts, use
the source contract's `ResourceRows` and `Scoped` coverage terminals with
their full prefix. They are separate denominators. For this exact-page
drilldown Q3 already supplies the selected page/role identity and duration
coverage, so repeating the general `PageRows` coverage terminal is unnecessary.

## Q4: hourly or daily trends with partial buckets and empty gaps

Assembly: Parameters (role filtering true) + Adapter + Normalization + Scope
+ this complete block. Execute once with `1h` and once with `1d`.
No hour-of-day folding, rolling smoothing, or removal of partial days.

```kusto
let _binSize=1h;
let Focus=NamedPages
    | where _roleFilterEnabled
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | where RoleName == _roleName;
let BucketStats=Focus
    | summarize ObservedViews=count(),
        PositiveDurationViews=countif(DurationState == "Positive"),
        MeanMs=avg(DurationMs),MedianMs=percentile(DurationMs,50),
        P95Ms=percentile(DurationMs,95),
        ObservedUsers=count_distinctif(UserKey,isnotempty(UserKey)),
        ObservedSessions=count_distinctif(SessionKey,isnotempty(SessionKey))
        by BucketStart=bin(EventTime,_binSize);
let BucketGrid=range BucketStart from bin(_startTime,_binSize) to _endTime step _binSize
    | where BucketStart < _endTime
    | extend BucketEnd=BucketStart+_binSize
    | extend CoveredStart=max_of(BucketStart,_startTime),
        CoveredEnd=min_of(BucketEnd,_endTime)
    | extend PartialBucket=CoveredStart != BucketStart or CoveredEnd != BucketEnd;
BucketGrid
| join kind=leftouter BucketStats on BucketStart
| project BucketStart,BucketEnd,CoveredStart,CoveredEnd,PartialBucket,
    ObservedViews=coalesce(ObservedViews,tolong(0)),
    PositiveDurationViews=coalesce(PositiveDurationViews,tolong(0)),
    MeanMs=iff(PositiveDurationViews > 0,MeanMs,real(null)),
    MedianMs=iff(PositiveDurationViews > 0,MedianMs,real(null)),
    P95Ms=iff(PositiveDurationViews > 0,P95Ms,real(null)),
    ObservedUsers=coalesce(ObservedUsers,tolong(0)),
    ObservedSessions=coalesce(ObservedSessions,tolong(0)),
    QueryStatus="Succeeded",CollectionContinuity="NotEstablishedByQuery"
| order by BucketStart asc
```

The literal query status is valid only if execution actually succeeds without
truncation. A failed query produces no valid rows: the report must explicitly
mark its requested interval unavailable. Never fabricate a successful grid
for a failed query. Known retention/outage limits must annotate the result
even if it succeeds. Identity metrics may still be unavailable from schema.

If a longer explicitly supplied interval exceeds budgets, add a bounded
output range to `BucketGrid` for disjoint UTC bucket slices while preserving
the same source `_startTime`/`_endTime` and complete `BucketStats` computation.
Specify each slice's bounds as additional local lets and validate the entire
assembly. Reconcile output coverage, avoid overlap, and keep failed slices
unavailable. Compute full-window KPIs separately via Q3. Do not derive them
from chart points. Never silently increase bin width to force a result.

Charts: time on X; duration in ms on its own Y axis; retained observed views
on a separate chart. Null duration stays a gap. Do not interpolate outages.
If converting to seconds, divide every duration value by 1000 and label
every corresponding column/axis in seconds, never mix ms and seconds.

## Q5: one context dimension, full totals before top-N

Assembly: Parameters (role filtering true) + Adapter + Normalization + Scope
+ this complete block. Run separately for each verified safe dimension:
`Browser`, `ClientType`, `RoleInstance`, `AppVersion`, and `SDKVersion`.
Require `_dimension` to be one of these exact values before submission;
do not silently accept an unknown selector. Missing optional columns generate
an explicit missing bucket rather than an invented dimension.

```kusto
let _dimension="Browser";
let _topDimensions=20;
let Focus=NamedPages
    | where _roleFilterEnabled
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | where RoleName == _roleName;
let DimensionSummary=Focus
    | extend DimensionValue=case(_dimension=="Browser",Browser,
        _dimension=="ClientType",ClientType,
        _dimension=="RoleInstance",RoleInstance,
        _dimension=="AppVersion",AppVersion,
        _dimension=="SDKVersion",SDKVersion,"")
    | summarize ObservedViews=count(),
        PositiveDurationViews=countif(DurationState == "Positive"),
        MeanMs=avg(DurationMs),MedianMs=percentile(DurationMs,50),
        P95Ms=percentile(DurationMs,95) by DimensionValue
    | extend MissingDimension=isempty(DimensionValue),
        MeanMs=iff(PositiveDurationViews > 0,MeanMs,real(null)),
        MedianMs=iff(PositiveDurationViews > 0,MedianMs,real(null)),
        P95Ms=iff(PositiveDurationViews > 0,P95Ms,real(null));
let FullObservedViews=toscalar(DimensionSummary | summarize sum(ObservedViews));
let FullPositiveDurationViews=toscalar(DimensionSummary | summarize sum(PositiveDurationViews));
let FullDimensionBuckets=toscalar(DimensionSummary | count);
let MissingDimensionViews=toscalar(DimensionSummary
    | summarize sumif(ObservedViews,MissingDimension));
let Displayed=union
    (DimensionSummary | where not(MissingDimension)
        | order by ObservedViews desc,DimensionValue asc | take _topDimensions),
    (DimensionSummary | where MissingDimension);
let DisplayedObservedViews=toscalar(Displayed | summarize sum(ObservedViews));
Displayed
| extend Dimension=_dimension,
    FullObservedViews=FullObservedViews,
    FullPositiveDurationViews=FullPositiveDurationViews,
    FullDimensionBuckets=FullDimensionBuckets,
    MissingDimensionViews=MissingDimensionViews,
    DisplayedObservedViews=DisplayedObservedViews,
    OmittedObservedViews=FullObservedViews-DisplayedObservedViews,
    ShareOfFullObserved=iff(FullObservedViews>0,todouble(ObservedViews)/FullObservedViews,real(null)),
    Ranking="ObservedViews descending",TopKnownBucketLimit=_topDimensions
| project Dimension,DimensionValue,MissingDimension,ObservedViews,
    PositiveDurationViews,MeanMs,MedianMs,P95Ms,ShareOfFullObserved,
    FullObservedViews,FullPositiveDurationViews,FullDimensionBuckets,
    MissingDimensionViews,DisplayedObservedViews,OmittedObservedViews,
    Ranking,TopKnownBucketLimit
| order by ObservedViews desc,DimensionValue asc
```

Missing is represented by an empty value **and a boolean**, not by replacing
the raw value with a label that could collide with a real string. Render
that boolean as a clearly marked unknown bucket. It is included even if
outside the top 20 known buckets. There is no row for an empty cohort; use
Q3's no-observations result, not an invented context distribution.

FullObservedViews must reconcile with Q3. Per-dimension tables overlap each
other and must not be summed. Role context is the single selected component
and the separate Q1 discovery result, not a new cross-role performance table.
Marginal differences are associations with substantial confounding possible;
they are not proof of browser, version, or host causation.

## Q6: limited slow diagnostic observations without IDs

Assembly: Parameters (role filtering true) + Adapter + Normalization + Scope
+ this complete block. Only run after verifying these labels safe; otherwise
remove unsafe optional projections and record the omission. Keep six or
fewer columns and at most 20 rows. Role instance is deliberately absent from
this event sample even when it is available in aggregate context.

```kusto
let _slowThresholdMs=3000.0;
let _sampleLimit=20;
let Focus=NamedPages
    | where _roleFilterEnabled
    | where isnotempty(trim(@"\s+",_pageName)) and PageName == _pageName
    | where RoleName == _roleName;
Focus
| where DurationState == "Positive" and DurationMs>_slowThresholdMs
| project EventTime,DurationMs,Browser,ClientType,AppVersion,SDKVersion
| order by DurationMs desc,EventTime asc
| take _sampleLimit
```

Report returned sample rows / Q3 SlowViews and the screening threshold.
Samples are deliberately biased toward slow observations, not representative
traffic and not necessarily distinct visits. Retained duplicate emissions
are not silently deduplicated. Never add URL, property bag, error text, page
title, tenant ID, user ID, session ID, activity/correlation ID, or IP columns.
