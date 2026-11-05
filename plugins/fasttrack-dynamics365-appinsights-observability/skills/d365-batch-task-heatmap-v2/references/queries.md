# Batch task heatmap query library

## Source resolution and assembly

Select the adapter internally from the resource and tool:
`QueryAppInsightsByAppId` / `QueryAppInsightsByResourceId` prefer `customEvents`;
`QueryLogAnalyticsByResourceId` / `QueryLogAnalyticsByWorkspaceId` prefer
`AppEvents`. For an explicit generic Kusto target inspect schema metadata, or
probe `customEvents` first if no schema catalog is available.

Probe `<table> | take 0 | getschema`. Confirm time, event-name, and properties
columns and the role-instance column. Try the alternate table on the same
target only after a confirmed missing-table result. Do not retry with a
different dialect for zero rows, access errors, timeouts, or incompatible
schemas. If both exist, retain the tool-preferred family, or `customEvents`
for a generic target without a declared preference. Never union both.

Submit **Parameters + one Source adapter + Normalization + required dataset
definitions + one terminal query**. Environment discovery reads `Candidates`
before filters. All other analysis uses `Starts`/`KnownStarts`. These are
query-local definitions, not persistent tables. Include only the definitions,
not the terminal output of an earlier query. Validate assembled queries when
the selected host supports it.

Replace tokens with typed values or safely escaped KQL literals, never unescaped
user input. Freeze start/end once, require start < end <= now, and default to
24 hours with `_bin=1h`. Keep a positive whole-minute interval if the user
requests a different granularity and record it in `metadata.bin_minutes`.
Do not silently widen an explicitly requested one-hour bucket.

## Parameters

```kusto
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let _bin=1h;
let requestedEnvironmentId=tolower(trim(@"\s+","{{EnvironmentId}}"));
let requestedRoleInstance="{{RoleInstance}}";
let requestedBatchJobId="{{BatchJobId}}";
let requestedClassName="{{ClassName}}";
let requestedLegalEntity="{{LegalEntity}}";
```

## Source adapter: Application Insights

```kusto
let RawRows=customEvents
    | where timestamp >= _startTime and timestamp < _endTime
    | where name == "BatchTaskStart"
    | project EventTime=timestamp,Dimensions=todynamic(customDimensions),
        RoleInstance=tostring(column_ifexists("cloud_RoleInstance","")),
        SampleWeight=todouble(column_ifexists("itemCount",real(null))),
        SourceTable="customEvents";
```

## Source adapter: Log Analytics

```kusto
let RawRows=AppEvents
    | where TimeGenerated >= _startTime and TimeGenerated < _endTime
    | where Name == "BatchTaskStart"
    | project EventTime=TimeGenerated,Dimensions=todynamic(Properties),
        RoleInstance=tostring(column_ifexists("AppRoleInstance","")),
        SampleWeight=todouble(column_ifexists("ItemCount",real(null))),
        SourceTable="AppEvents";
```

`todynamic` handles dynamic properties and JSON strings. Do not change nested
property casing. If metadata identifies another role-instance column, change
only that projection. If none exists, preserve empty RoleInstance and report
that AOS attribution and plotting are unavailable; do not invent an AOS.

## Normalization

```kusto
let Candidates=RawRows
    | extend LowerEnvironment=tolower(trim(@"\s+",tostring(Dimensions.environmentId))),
        UpperEnvironment=tolower(trim(@"\s+",tostring(Dimensions.EnvironmentId)))
    | extend EnvironmentId=coalesce(LowerEnvironment,UpperEnvironment),
        ConflictingEnvironment=isnotempty(LowerEnvironment)
            and isnotempty(UpperEnvironment) and LowerEnvironment != UpperEnvironment
    | extend BatchJobId=coalesce(tostring(Dimensions.BatchJobId),tostring(Dimensions.batchJobId)),
        ClassName=tostring(Dimensions.ClassName),LegalEntity=tostring(Dimensions.LegalEntity);
let Starts=Candidates
    | where isnotempty(requestedEnvironmentId)
        and EnvironmentId == requestedEnvironmentId and not(ConflictingEnvironment)
    | where isempty(requestedRoleInstance) or RoleInstance =~ requestedRoleInstance
    | where isempty(requestedBatchJobId) or BatchJobId == requestedBatchJobId
    | where isempty(requestedClassName) or ClassName =~ requestedClassName
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity;
let KnownStarts=Starts | where isnotempty(trim(@"\s+",RoleInstance));
```

Missing task/activity IDs do not remove start rows. Exact full source instance
strings identify nodes; do not shorten them or treat `cloud_RoleName` as AOS.
Optional metadata filters cannot attribute rows whose metadata is absent.
Disclose that filtered counts cover only matching observed metadata.

## Environment discovery

```kusto
Candidates
| summarize Rows=count() by EnvironmentId,ConflictingEnvironment
| order by Rows desc
```

Choose automatically only with one nonempty ID and no conflicting keys.
Otherwise use the caller's explicit environment or clarify ambiguity. Retain
missing/conflicting-ID counts in coverage even though those rows are excluded.

## Coverage and KPIs

```kusto
Starts
| summarize TaskStartEvents=count(),
    KnownAOSStarts=countif(isnotempty(trim(@"\s+",RoleInstance))),
    UnknownAOSStarts=countif(isempty(trim(@"\s+",RoleInstance))),
    ObservedAOS=count_distinctif(RoleInstance,isnotempty(trim(@"\s+",RoleInstance))),
    FirstSeen=min(EventTime),LastSeen=max(EventTime),
    RowsWithSamplingIndicator=countif(SampleWeight > 1),
    RowsWithUnknownSampleWeight=countif(
        isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1),
    MissingJobId=countif(isempty(BatchJobId)),MissingClass=countif(isempty(ClassName))
```

Counts represent retained events. Do not weight counts using SampleWeight or
interpret no sampling indicator as proof of completeness. Report possible late
arrivals when the end is near now (initial five-minute caveat). Do not move the
window differently across queries.

## Grid preflight

```kusto
let _nodeCount=toscalar(KnownStarts | summarize count_distinct(RoleInstance));
let _binCount=tolong(ceiling((_endTime-bin(_startTime,_bin))/_bin));
print ObservedAOS=_nodeCount,TimeBuckets=_binCount,ExpectedCells=_nodeCount*_binCount
| extend FitsQueryBudget=TimeBuckets <= 5000 and ExpectedCells <= 20000
```

Run before the matrix query. If over budget, retrieve the observed node list
(`KnownStarts | distinct RoleInstance | order by RoleInstance asc`) and batch
deterministic, disjoint node subsets, each within the budget. Apply each
subset to `KnownStarts` for grid construction. Keep original time scope.
When >5,000 bins, split query windows at bucket boundaries and keep a frozen
full-scope node list for the batch; do not rediscover nodes per slice, which
would omit zero cells. If a slice contains no starts for a listed node, its
successful query must still yield zero cells for that node.

Do not silently `take` grid rows, select only busiest nodes, or truncate a
matrix to fit the tool. Honor a smaller host limit. Merge results by exact
`RoleInstance,TimeBucket` without overlap and reconcile against full-scope
KPIs. Missing/failed slices remain null and mark the result partial, not zero.

## Matrix datasets

```kusto
let Nodes=KnownStarts | distinct RoleInstance;
let Buckets=range TimeBucket from bin(_startTime,_bin) to _endTime step _bin
    | where TimeBucket < _endTime
    | extend IsCompleteBin=TimeBucket >= _startTime and TimeBucket+_bin <= _endTime;
let CellCounts=KnownStarts
    | summarize TaskStarts=count() by RoleInstance,TimeBucket=bin(EventTime,_bin);
let HeatmapCells=Nodes
    | extend GridKey=1
    | join kind=inner (Buckets | extend GridKey=1) on GridKey
    | project RoleInstance,TimeBucket,IsCompleteBin
    | join kind=leftouter CellCounts on RoleInstance,TimeBucket
    | project RoleInstance,TimeBucket,TaskStarts=coalesce(TaskStarts,0),IsCompleteBin;
```

For batched execution, replace `Nodes` with the explicit frozen batch list
using a safely constructed `datatable(RoleInstance:string)[...]`; include
only those nodes in CellCounts, and use the batch's bucket-aligned time slice.
Preserve original partial-edge labels; only the first/last global bins may
be partial. Do not run the unmodified unbounded grid after failed preflight.

## Heatmap data

```kusto
HeatmapCells
| order by RoleInstance asc,TimeBucket asc
```

Map `TimeBucket` to X, `RoleInstance` to Y, `TaskStarts` to color. Use the
returned numeric counts, not a chart inferred from narrative or the top-cells
query. Confirm returned cells equal preflight ExpectedCells on complete results.
No known AOS means an empty matrix, not a chart of invented nodes. The unknown
AOS count remains in the KPIs and hourly totals.

## Per-AOS distribution

```kusto
let _knownTotal=toscalar(KnownStarts | count);
HeatmapCells
| summarize TaskStarts=sum(TaskStarts),PeakHourlyStarts=max(TaskStarts),
    BinsWithStarts=countif(TaskStarts > 0),ObservedBins=count()
    by RoleInstance
| extend ShareOfKnownStartsPct=iff(_knownTotal > 0,
    round(100.0*todouble(TaskStarts)/todouble(_knownTotal),2),real(null))
| order by TaskStarts desc,RoleInstance asc
```

For custom granularity rename the display label for `PeakHourlyStarts` to
**peak starts per bucket**; the returned field remains unchanged.
Share is a proportion of observed known-AOS starts, not utilization, success,
or a share of all tasks. Round only for display. Aggregate over the full merged
grid after batching; do not compare batch-local percentage denominators.

## Hourly totals including unknown AOS

```kusto
let HourCounts=Starts
    | summarize KnownAOSStarts=countif(isnotempty(trim(@"\s+",RoleInstance))),
        UnknownAOSStarts=countif(isempty(trim(@"\s+",RoleInstance)))
        by TimeBucket=bin(EventTime,_bin);
Buckets
| join kind=leftouter HourCounts on TimeBucket
| project TimeBucket,KnownAOSStarts=coalesce(KnownAOSStarts,0),
    UnknownAOSStarts=coalesce(UnknownAOSStarts,0),IsCompleteBin
| extend TaskStartEvents=KnownAOSStarts+UnknownAOSStarts
| order by TimeBucket asc
```

Needs `Buckets` from Matrix datasets, but not the grid expansion. For large
requests retrieve totals in disjoint time slices with the same global bins.

## Highest-count cells

```kusto
HeatmapCells
| where TaskStarts > 0
| sort by TaskStarts desc,TimeBucket asc,RoleInstance asc
| take 10
```

This is a separate summary only. Do not feed these ten cells to PlotHeatmap as
if they represented the full matrix.

## Presentation and reconciliation

Use `assets/findings-contract.md`. Keep full UTC dates and ordered bins. With
nonaligned boundaries, the first bucket may precede start and the final bucket
is partial; a 24-hour interval can legitimately include 25 hourly buckets.

On complete results:

- Matrix cells = observed known nodes x overlapping bins.
- Matrix sum = per-AOS sum = KnownAOSStarts.
- Hourly KnownAOSStarts sum = matrix sum.
- KnownAOSStarts + UnknownAOSStarts = TaskStartEvents.

On incomplete queries, missing cells are unavailable/null. Do not render them
with zero intensity. Record query errors and do not claim reconciliation.
Plot panels must use the same global count scale, preserving zero and unknown.
The Markdown response must include the numeric data behind the plotted cells,
with any preview omissions and the complete retrieved dataset distinguished.
