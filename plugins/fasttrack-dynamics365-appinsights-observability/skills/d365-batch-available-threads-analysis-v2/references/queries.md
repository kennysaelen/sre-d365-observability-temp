# Available batch threads query library

## Source resolution and assembly

Use one authorized target throughout. `QueryAppInsightsByAppId` and
`QueryAppInsightsByResourceId` prefer `customEvents`;
`QueryLogAnalyticsByResourceId` and `QueryLogAnalyticsByWorkspaceId` prefer
`AppEvents`. For `kusto_query`, inspect target metadata; without a declared
surface, probe `customEvents` first.

Probe `<table> | take 0 | getschema` and confirm time (datetime), event name
(string), and properties (dynamic or JSON string). Check optional full role
instance and sample-weight columns. Missing optional columns use the adapter's
empty/null defaults and must be disclosed. Preserve case-sensitive names.
The supplied general schema documents the Application Insights columns;
Log Analytics equivalents must be confirmed on the runtime target.

Try the alternate table only after a confirmed missing-table error on this
same target. No alternate for zero data, timeout, authorization failure,
incompatible columns, or truncation. Surface errors, not successful zeros.
If both tables are available, keep the tool-preferred one (customEvents for
an otherwise unspecified generic surface). Never union the two views or
fingerprint-deduplicate them. No exception tables or internal telemetry.

The fenced blocks have stable IDs for composition. Each submission consists
of its listed definitions followed by exactly one terminal block:

| Terminal | Required prefix, in order |
|---|---|
| D: discovery | P + exactly one of A/L + E |
| C: coverage and preflight | P + A/L + E + N |
| Q: validation breakdown | P + A/L + E + N |
| S: per-AOS summary | P + A/L + E + N + K |
| T: timeline | P + A/L + E + N + K + B |

Do not include earlier terminal outputs in later queries. Run D, then C,
then Q/S/T subject to preflight. Use `ValidateQuery` on complete submissions
when supported. Local assembly is not KQL compiler validation.

Replace all placeholders with escaped KQL literals or supported bound
parameters. Never interpolate unescaped user text. Require one nonempty
environment for N; only D may run before environment resolution. Freeze
ObservedAtUtc once and start < end <= ObservedAtUtc, all UTC. Use 10-minute
bins by default; `_bin` and `_freshness` may be adjusted explicitly and recorded.
Use positive whole-minute bins and a positive freshness threshold.

### P: parameters

```kusto
let _observedAt=datetime({{ObservedAtUtc}});
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let _bin=10m;
let _freshness=15m;
let _cutoff=min_of(_endTime,_observedAt);
let requestedEnvironmentId=tolower(trim(@"\s+", "{{EnvironmentId}}"));
let requestedRoleInstance="{{RoleInstance}}";
```

### A: Application Insights adapter

```kusto
let RawRows=customEvents
    | where timestamp >= _startTime and timestamp < _endTime
    | where name == "BatchThreadInfo"
    | extend PropertiesBag=parse_json(tostring(customDimensions))
    | project EventTime=timestamp,
        LowerEnvironment=tostring(PropertiesBag.environmentId),
        UpperEnvironment=tostring(PropertiesBag.EnvironmentId),
        InfoText=tostring(PropertiesBag.InfoMessage),
        RoleInstance=tostring(column_ifexists("cloud_RoleInstance","")),
        SampleWeight=todouble(column_ifexists("itemCount",real(null)));
```

### L: Log Analytics adapter

```kusto
let RawRows=AppEvents
    | where TimeGenerated >= _startTime and TimeGenerated < _endTime
    | where Name == "BatchThreadInfo"
    | extend PropertiesBag=parse_json(tostring(Properties))
    | project EventTime=TimeGenerated,
        LowerEnvironment=tostring(PropertiesBag.environmentId),
        UpperEnvironment=tostring(PropertiesBag.EnvironmentId),
        InfoText=tostring(PropertiesBag.InfoMessage),
        RoleInstance=tostring(column_ifexists("AppRoleInstance","")),
        SampleWeight=todouble(column_ifexists("ItemCount",real(null)));
```

Only these necessary fields enter normalization; no identity, job, operation,
stack or raw event-message fields. InfoText is internal parse input and is
never returned by a terminal query.

### E: environment normalization

```kusto
let EnvironmentRows=RawRows
    | extend LowerEnvironment=tolower(trim(@"\s+",LowerEnvironment)),
        UpperEnvironment=tolower(trim(@"\s+",UpperEnvironment))
    | extend EnvironmentId=coalesce(LowerEnvironment,UpperEnvironment),
        ConflictingEnvironment=isnotempty(LowerEnvironment)
            and isnotempty(UpperEnvironment) and LowerEnvironment != UpperEnvironment
    | extend RoleInstance=iff(isempty(trim(@"\s+",RoleInstance)),"",RoleInstance);
```

### D: discovery

```kusto
EnvironmentRows
| summarize SourceRows=count(),
    CandidateIds=make_set_if(EnvironmentId,
        isnotempty(EnvironmentId) and not(ConflictingEnvironment),2),
    MissingEnvironment=countif(isempty(EnvironmentId)),
    ConflictingEnvironmentRows=countif(ConflictingEnvironment)
```

The set cap of two is deliberate: one candidate can be auto-selected; two
means ambiguity, not an exhaustive list. Auto-select only one candidate with
zero conflicts. Missing-ID counts remain disclosed. Do not apply AOS filters
to discovery. If no candidate exists, use an explicit/unambiguous configured
environment or clarify; never invent one. Explicit environment scope excludes
missing/conflicting IDs. Zero scoped rows do not trigger source fallback.

### N: scoped validated snapshots

```kusto
let CountStatus=(raw:string) {
    let value=todouble(raw);
    case(isempty(trim(@"\s+",raw)),"Missing",
        isnull(value) or not(isfinite(value)),"NonNumericOrNonFinite",
        value < 0,"Negative",
        value % 1.0 != 0,"Fractional",
        value > 2147483647.0,"OutsideSchemaIntRange",
        "Valid")
};
let CountValue=(raw:string) {
    iff(CountStatus(raw) == "Valid",tolong(todouble(raw)),long(null))
};
let MatchedEnvironment=EnvironmentRows
    | where isnotempty(requestedEnvironmentId)
        and EnvironmentId == requestedEnvironmentId and not(ConflictingEnvironment);
let Samples=MatchedEnvironment
    | where isempty(requestedRoleInstance) or RoleInstance == requestedRoleInstance
    | extend Info=parse_json(InfoText)
    | extend InfoStatus=case(isempty(trim(@"\s+",InfoText)),"Missing",
        gettype(Info) != "dictionary","Malformed","Parsed")
    | extend Info=iff(InfoStatus == "Parsed",Info,dynamic({}))
    | extend CurrentStatus=CountStatus(tostring(Info.CurrentBatchTasks)),
        MaximumStatus=CountStatus(tostring(Info.MaxThreadCount)),
        ReservedStatus=CountStatus(tostring(Info.ReservedNumberOfThreads)),
        QueueStatus=CountStatus(tostring(Info.TaskQueueCount)),
        CurrentBatchTasks=CountValue(tostring(Info.CurrentBatchTasks)),
        MaxThreadCount=CountValue(tostring(Info.MaxThreadCount)),
        ReservedNumberOfThreads=CountValue(tostring(Info.ReservedNumberOfThreads)),
        TaskQueueCount=CountValue(tostring(Info.TaskQueueCount))
    | extend SampleStatus=case(InfoStatus != "Parsed",strcat("Info:",InfoStatus),
        CurrentStatus != "Valid",strcat("Current:",CurrentStatus),
        MaximumStatus != "Valid",strcat("Maximum:",MaximumStatus),
        ReservedStatus != "Valid",strcat("Reserved:",ReservedStatus),
        ReservedNumberOfThreads > MaxThreadCount,"ReservedExceedsMaximum",
        "Valid")
    | extend ValidAvailable=SampleStatus == "Valid"
    | extend EffectiveCapacity=iff(ValidAvailable,
            MaxThreadCount - ReservedNumberOfThreads,long(null)),
        AvailableThreads=iff(ValidAvailable,
            MaxThreadCount - CurrentBatchTasks - ReservedNumberOfThreads,long(null))
    | extend HeadroomPct=iff(ValidAvailable and EffectiveCapacity > 0,
        100.0 * todouble(AvailableThreads) /
            iff(EffectiveCapacity > 0,todouble(EffectiveCapacity),real(null)),real(null))
    | project EventTime,RoleInstance,InfoStatus,CurrentStatus,MaximumStatus,
        ReservedStatus,QueueStatus,CurrentBatchTasks,MaxThreadCount,
        ReservedNumberOfThreads,TaskQueueCount,SampleStatus,ValidAvailable,
        EffectiveCapacity,AvailableThreads,HeadroomPct,SampleWeight;
```

The four fields are `int` in the supplied schema: accept numeric strings or
JSON numbers only after finite/nonnegative/integer validation, within signed
32-bit positive range. Out-of-schema range is explicit, never integer overflow
or truncation. Arithmetic uses `long`. Do not truncate fractional inputs.
If runtime evidence establishes a wider integer type, review this bound
explicitly before changing it. Missing optional queue is not a headroom failure.

`SampleStatus` records the first blocking reason; retain all four field
statuses for the complete diagnosis. Parsed individual values may still be
shown on an invalid latest row, with the row's invalid status. Derived headroom
and capacity remain null when the required snapshot is invalid.

### C: coverage and preflight

```kusto
let NodeCount=toscalar(Samples | where isnotempty(RoleInstance)
    | summarize by RoleInstance | count);
let Buckets=tolong(ceiling((_endTime - bin(_startTime,_bin)) / _bin));
let EnvironmentRowCount=toscalar(MatchedEnvironment | count);
let EnvironmentMissingAOS=toscalar(MatchedEnvironment
    | summarize N=countif(isempty(RoleInstance)) | project N);
Samples
| summarize ScopedRows=count(),ValidSamples=countif(ValidAvailable),
    InvalidSamples=countif(not(ValidAvailable)),
    KnownAOSRows=countif(isnotempty(RoleInstance)),
    MissingAOSRows=countif(isempty(RoleInstance)),
    KnownAOSValidSamples=countif(isnotempty(RoleInstance) and ValidAvailable),
    KnownAOSInvalidSamples=countif(isnotempty(RoleInstance) and not(ValidAvailable)),
    QueueUnavailable=countif(QueueStatus != "Valid"),
    SamplingIndicatedRows=countif(SampleWeight > 1 and isfinite(SampleWeight)),
    SamplingUnknownRows=countif(isnull(SampleWeight) or not(isfinite(SampleWeight))
        or SampleWeight < 1 or SampleWeight % 1.0 != 0),
    FirstObserved=min(EventTime),LastObserved=max(EventTime)
| extend ObservedAOS=NodeCount,EnvironmentRowsBeforeAOSFilter=EnvironmentRowCount,
    EnvironmentMissingAOSBeforeFilter=EnvironmentMissingAOS,
    BucketCount=Buckets,ExpectedTimelineRows=NodeCount * Buckets,
    EstimatedTimelineCells=NodeCount * Buckets * 32,
    EstimatedSummaryCells=NodeCount * 48
```

Before S/T, require output <= **10,000 rows and 250,000 scalar cells per
query**, or smaller actual host limits. Conservative cell estimates use 32
columns for T and 48 for S. Before building B, also require <=5,000 time
buckets. These are output safety budgets, not telemetry health thresholds.
No raw snapshot retrieval is needed.

If budgets fail, increase the common whole-minute bucket size automatically
and disclose it, preserving the window. Rerun C. If node count alone remains
too large, report timeline/summary unavailable due to budget and request a
narrower exact AOS scope; do not execute an oversized grid, silently choose
top nodes, or truncate to apparent success. Coverage can still be reported.
For a source scan timeout/resource limit, surface the error rather than
retrying with a different table or pretending no samples exist.

Inspect tool truncation/partial-result metadata for every query. A truncated
result is partial even if row count appears within budget. No missing grid
rows may be inferred healthy. If independent queries differ due to late
arrival, record unreconciled/partial coverage and rerun the same frozen scope
once when feasible; never silently reconcile by dropping observations.

### Q: validation breakdown

```kusto
Samples
| summarize Samples=count() by InfoStatus,CurrentStatus,MaximumStatus,
    ReservedStatus,QueueStatus,SampleStatus
| order by Samples desc
```

The six field statuses permit at most 6^4 combinations, plus six
ReservedExceedsMaximum alternatives when the three required fields are valid,
plus two nonparsed Info states: **1,304 rows / 9,128 cells** maximum.
This fits the default budgets. Check smaller host limits before running Q;
if they cannot accommodate this bound, report that breakdown unavailable
rather than returning an unlabeled truncated diagnosis.

### K: known nodes and timestamp ambiguity

```kusto
let KnownSamples=Samples | where isnotempty(RoleInstance);
let TimeVariants=KnownSamples
    | summarize RowsWithValues=count() by RoleInstance,EventTime,
        CurrentBatchTasks,MaxThreadCount,ReservedNumberOfThreads,TaskQueueCount,
        InfoStatus,CurrentStatus,MaximumStatus,ReservedStatus,QueueStatus,SampleStatus;
let TimeKeys=TimeVariants
    | summarize RowsAtTime=sum(RowsWithValues),ValueVariants=count()
        by RoleInstance,EventTime;
```

This grouping flags duplicate timestamps and conflicting normalized metric
tuples, NOT event identity and NOT deduplication. All retained rows still
contribute equally to snapshot statistics. Equivalent invalid raw values
can normalize to the same tuple; conflict detection is conservative. Do not
claim duplicate timestamps prove duplicate delivery or independent samples.

### S: per-AOS summary

```kusto
let Statistics=KnownSamples
    | summarize Samples=count(),ValidSamples=countif(ValidAvailable),
        InvalidSamples=countif(not(ValidAvailable)),
        MinAvailable=min(AvailableThreads),
        MedianAvailable=percentile(AvailableThreads,50),
        P95Available=percentile(AvailableThreads,95),
        MaxAvailable=max(AvailableThreads),
        ZeroSamples=countif(ValidAvailable and AvailableThreads == 0),
        NegativeSamples=countif(ValidAvailable and AvailableThreads < 0),
        FirstObserved=min(EventTime),LastValidObserved=maxif(EventTime,ValidAvailable)
        by RoleInstance;
let Ambiguity=TimeKeys
    | summarize RepeatedTimestampGroups=countif(RowsAtTime > 1),
        DuplicateExcessRows=sum(RowsAtTime - 1),
        ConflictingTimestampGroups=countif(ValueVariants > 1) by RoleInstance;
let Latest=KnownSamples
    | summarize arg_max(EventTime,CurrentBatchTasks,MaxThreadCount,
        ReservedNumberOfThreads,TaskQueueCount,EffectiveCapacity,AvailableThreads,
        HeadroomPct,ValidAvailable,SampleStatus,InfoStatus,CurrentStatus,
        MaximumStatus,ReservedStatus,QueueStatus,SampleWeight) by RoleInstance
    | join kind=leftouter TimeKeys on RoleInstance,EventTime
    | project RoleInstance,LatestTime=EventTime,CurrentBatchTasks,MaxThreadCount,
        ReservedNumberOfThreads,TaskQueueCount,EffectiveCapacity,
        LatestAvailable=AvailableThreads,LatestHeadroomPct=HeadroomPct,
        LatestValid=ValidAvailable,SampleStatus,InfoStatus,CurrentStatus,
        MaximumStatus,ReservedStatus,QueueStatus,SampleWeight,
        LatestTimestampRows=RowsAtTime,LatestValueVariants=ValueVariants
    | extend AgeMinutes=(_cutoff - LatestTime) / 1m,
        Freshness=iff(_cutoff - LatestTime > _freshness,"Stale","WithinHeuristic"),
        LatestAmbiguous=LatestValueVariants > 1;
Statistics
| join kind=inner Latest on RoleInstance
| join kind=inner Ambiguity on RoleInstance
| project-away RoleInstance1,RoleInstance2
| order by RoleInstance asc
```

Latest means latest **in the requested window**, not lifetime latest. If that
row is invalid, show it as invalid and show `LastValidObserved` separately;
never populate latest metrics from the earlier valid row. All config values
come from the same arg_max row. Conflicting ties make the displayed latest
row an arbitrary coherent candidate, not authoritative current configuration.
Do not aggregate tied candidates field-by-field.

WithinHeuristic only concerns timestamp recency relative to `_cutoff`, not
validity or health. Report it independently of LatestValid/LatestAmbiguous.
Unobserved nodes have no summary row; they are not proven absent or fresh.

### B: bucket dataset and null grid

```kusto
let BucketStats=KnownSamples
    | summarize Samples=count(),ValidSamples=countif(ValidAvailable),
        InvalidSamples=countif(not(ValidAvailable)),
        MinAvailable=min(AvailableThreads),
        MedianAvailable=percentile(AvailableThreads,50),
        MaxAvailable=max(AvailableThreads),
        ZeroSamples=countif(ValidAvailable and AvailableThreads == 0),
        NegativeSamples=countif(ValidAvailable and AvailableThreads < 0),
        FirstObserved=min(EventTime),LastObserved=max(EventTime),
        FirstNonpositive=minif(EventTime,ValidAvailable and AvailableThreads <= 0),
        LastNonpositive=maxif(EventTime,ValidAvailable and AvailableThreads <= 0)
        by RoleInstance,TimeBucket=bin(EventTime,_bin);
let BucketAmbiguity=TimeKeys
    | summarize DuplicateExcessRows=sum(RowsAtTime - 1),
        ConflictingTimestampGroups=countif(ValueVariants > 1)
        by RoleInstance,TimeBucket=bin(EventTime,_bin);
let Nodes=KnownSamples | summarize by RoleInstance | extend JoinKey=1;
let Times=range TimeBucket from bin(_startTime,_bin) to _endTime step _bin
    | where TimeBucket < _endTime | extend JoinKey=1;
let Timeline=Nodes
    | join kind=inner Times on JoinKey
    | project RoleInstance,TimeBucket
    | join kind=leftouter BucketStats on RoleInstance,TimeBucket
    | join kind=leftouter BucketAmbiguity on RoleInstance,TimeBucket
    | project RoleInstance,TimeBucket,
        IntervalStart=max_of(TimeBucket,_startTime),
        IntervalEnd=min_of(TimeBucket + _bin,_endTime),
        IsCompleteBin=TimeBucket >= _startTime and TimeBucket + _bin <= _endTime,
        Samples=coalesce(Samples,0),ValidSamples=coalesce(ValidSamples,0),
        InvalidSamples=coalesce(InvalidSamples,0),
        MinAvailable,MedianAvailable,MaxAvailable,
        ZeroSamples=coalesce(ZeroSamples,0),NegativeSamples=coalesce(NegativeSamples,0),
        FirstObserved,LastObserved,FirstNonpositive,LastNonpositive,
        DuplicateExcessRows=coalesce(DuplicateExcessRows,0),
        ConflictingTimestampGroups=coalesce(ConflictingTimestampGroups,0)
    | extend CoverageState=case(Samples == 0,"NoObservation",
        ValidSamples == 0,"InvalidOnly",InvalidSamples > 0,"Mixed","ValidObserved"),
        ValidSampleFraction=iff(Samples > 0,
            todouble(ValidSamples) / iff(Samples > 0,todouble(Samples),real(null)),
            real(null));
```

### T: timeline

```kusto
Timeline
| order by RoleInstance asc,TimeBucket asc
```

Empty successful bins have zero **observation counts** but NULL thread
gauges. Invalid-only bins have positive sample counts and null gauges.
Mixed bins retain valid-row statistics and explicitly flag invalid coverage.
No expected cadence is asserted by ValidSampleFraction.
Missing rows from incomplete retrieval are unavailable, not inferred bins.

Derive the nonpositive-interval insight from the same T rows where ZeroSamples
or NegativeSamples > 0. Keep interval bounds and first/last observed
nonpositive timestamps; these are NOT continuous episodes. Do not subtract
timestamps or multiply buckets/sample fractions into saturation duration.

Reconcile successful full results: sum T.Samples = sum S.Samples =
C.KnownAOSRows; likewise valid/invalid counts with the known-AOS counters.
C.ScopedRows = C.KnownAOSRows + C.MissingAOSRows and Valid + Invalid.
Sum Q.Samples = C.ScopedRows. T unique (RoleInstance,TimeBucket) count must
equal C.ExpectedTimelineRows. Min/max across T per node must match S min/max
(null for no valid rows). Median/p95 are calculated on original unweighted
samples, NOT averages/percentiles of bucket medians. Never sum thread gauges.
