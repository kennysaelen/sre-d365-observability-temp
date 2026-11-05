# Priority distribution queries

## Automatic source selection

Application Insights tools prefer `customEvents`; Log Analytics tools prefer
`AppEvents`. Probe the preferred table with `<table> | take 0 | getschema`.
For an explicitly supplied generic Kusto target, inspect schema metadata or
probe `customEvents` first. Try the alternate table on the same target only
after a confirmed missing-table response, not for an empty result, timeout,
or authorization error. If both exist, use the tool-preferred family (or
`customEvents` for a generic target without a declared preference).

Assemble parameters + exactly one adapter + normalization + aggregation +
one output query. Environment discovery needs only through normalization.
Replace placeholders using typed parameters or escaped KQL literals.
Require start < end <= now. No user-facing dialect choice or fuzzy union.

## Parameters

```kusto
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let requestedEnvironmentId=tolower(trim(@"\s+","{{EnvironmentId}}"));
```

## Adapter: Application Insights

```kusto
let RawRows=customEvents
    | where timestamp >= _startTime and timestamp < _endTime
    | where name == "BatchTaskStart"
    | project Dimensions=todynamic(customDimensions),
        SampleWeight=todouble(column_ifexists("itemCount",real(null)));
```

## Adapter: Log Analytics

```kusto
let RawRows=AppEvents
    | where TimeGenerated >= _startTime and TimeGenerated < _endTime
    | where Name == "BatchTaskStart"
    | project Dimensions=todynamic(Properties),
        SampleWeight=todouble(column_ifexists("ItemCount",real(null)));
```

## Normalization

```kusto
let Candidates=RawRows
    | extend LowerEnvironment=tolower(trim(@"\s+",tostring(Dimensions.environmentId))),
        UpperEnvironment=tolower(trim(@"\s+",tostring(Dimensions.EnvironmentId)))
    | extend EnvironmentId=coalesce(LowerEnvironment,UpperEnvironment),
        ConflictingEnvironment=isnotempty(LowerEnvironment)
            and isnotempty(UpperEnvironment) and LowerEnvironment != UpperEnvironment;
```

## Environment discovery

```kusto
Candidates
| summarize Rows=count() by EnvironmentId,ConflictingEnvironment
```

Auto-select only one nonempty environment with no conflicting keys. Otherwise
use the explicit environment or clarify scope. Missing/conflicting keys are
excluded from scoped analysis; disclose material exclusions briefly.

## Shared aggregation

```kusto
let Starts=Candidates
    | where isnotempty(requestedEnvironmentId)
        and EnvironmentId == requestedEnvironmentId and not(ConflictingEnvironment)
    | extend BatchJobId=trim(@"\s+",coalesce(
            tostring(Dimensions.BatchJobId),tostring(Dimensions.batchJobId))),
        PriorityValue=tolower(trim(@"\s+",tostring(Dimensions.Priority)));
let Jobs=Starts
    | where isnotempty(BatchJobId) and BatchJobId != "0"
    | summarize MissingPriorityRows=countif(isempty(PriorityValue)),
        Priorities=make_set_if(PriorityValue,isnotempty(PriorityValue),2)
        by BatchJobId
    | extend Priority=case(
        MissingPriorityRows > 0 or array_length(Priorities) == 0,"Unknown/incomplete",
        array_length(Priorities) > 1,"Mixed priorities",
        strcat("Priority: ",tostring(Priorities[0])));
```

Two distinct values are enough to detect mixed priority; the capped set does
not truncate job counts. Prefix raw values so they cannot be confused with
the generated mixed/unknown categories. Unknown takes precedence over mixed
when a job also has missing priorities. Do not infer Normal from missing data.
If an authoritative mapping is available, normalize aliases before this
aggregation; do not merge already-aggregated slices inconsistently.

## Pie chart data

```kusto
let _total=toscalar(Jobs | count);
Jobs
| summarize Jobs=count() by Priority
| extend Percentage=iff(_total > 0,round(100.0*todouble(Jobs)/todouble(_total),2),real(null))
| order by Jobs desc,Priority asc
```

Pass category `Priority` and value `Jobs` to PlotPieChart. Percentages use all
identifiable observed jobs, including mixed/unknown slices. Rounded percentages
may differ slightly from 100%; use counts as chart weights. No top-N cap.

## Summary and coverage

```kusto
let _jobs=toscalar(Jobs | count);
Starts
| summarize StartEvents=count(),
    UnidentifiedJobRows=countif(isempty(BatchJobId) or BatchJobId == "0"),
    RowsWithSamplingIndicator=countif(SampleWeight > 1),
    RowsWithUnknownSampleWeight=countif(
        isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1)
| extend ObservedJobs=_jobs
```

Reconcile sum of slice counts with ObservedJobs. Use results for the short
summary, not an extra diagnostic report. Sampling can hide jobs or priorities:
do not weight job counts by SampleWeight or claim full collection. Successful
empty results mean no identifiable jobs observed, not all jobs have zero
priority. If query results are truncated, do not plot a partial distribution
as complete; recover the full aggregation or disclose the limitation.
