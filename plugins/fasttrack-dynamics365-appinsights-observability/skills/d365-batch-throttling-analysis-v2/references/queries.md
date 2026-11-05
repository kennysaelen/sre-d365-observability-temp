# Batch throttling query library

One shared analysis library; internally select one of two small source adapters.
No cross-table unions or fingerprint deduplication.

## Automatic source resolution

| Selected query tool | Preferred event table |
|---|---|
| `QueryAppInsightsByAppId`, `QueryAppInsightsByResourceId` | `customEvents` |
| `QueryLogAnalyticsByResourceId`, `QueryLogAnalyticsByWorkspaceId` | `AppEvents` |
| `kusto_query` | Discover on the explicitly supplied compatible target |

Use `<table> | take 0 | getschema` for the preferred table. Confirm time, name,
and properties columns. Resolve role-instance mapping from metadata; if absent,
project an empty string and disclose that AOS analysis is unavailable.
For a generic target use schema metadata; if none is available, probe
`customEvents` first and `AppEvents` only after a confirmed missing-table error.
On any target, the alternate table may be probed after a genuine missing-table
result on the same target. Zero data rows are not a missing table. Do not fall
back for authentication/authorization failures, timeouts, or incompatible
schemas; surface the error. Never silently switch resources.

If both schemas resolve, retain the tool-preferred table (or `customEvents`
for a generic target without a declared preference). The caller supplies no
dialect selection. No exception table is needed for this skill.

## Query assembly

Submit **Parameters + one Source adapter + Normalization + the required
dataset definitions + one terminal query**. Let definitions are query-local.
Do not paste terminal outputs of earlier queries into a later query.
Environment discovery needs only Parameters + one Source adapter + discovery.
Coverage, KPIs, and timeline need the common prefix through Normalization.
Resource coverage, AOS resource summaries, and detail need the Throttles
definition as well. Replace every placeholder; quote/escape user-supplied
strings as KQL literals (prefer tool parameters when supported), never raw
string interpolation. `EnvironmentId` is required except for discovery.

Freeze UTC bounds once. Require start < end <= now and a positive bin size.
Defaults: 24 hours and 10 minutes. Keep <=5,000 timeline bins; widen the common
bin size for longer requests and disclose it. Preserve the requested window
when widening bins. Use whole-minute bins and split very long requests into
disclosed report windows if needed.
Set `metadata.bin_minutes` to the actual `_bin` size in minutes.
Queries use `[start, end)`, avoiding an extra end-point bin.
Do not use separate moving `ago()` windows for different result sections.

## Parameters

```kusto
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let _bin=10m;
let requestedEnvironmentId=tolower(trim(@"\s+", "{{EnvironmentId}}"));
let requestedRoleInstance="{{RoleInstance}}";
```

## Source adapter: Application Insights

```kusto
let RawRows=customEvents
    | where timestamp >= _startTime and timestamp < _endTime
    | where name in ("BatchTaskStart","BatchThrottled")
    | project EventTime=timestamp, EventName=name,
        Dimensions=todynamic(customDimensions),
        RoleInstance=tostring(column_ifexists("cloud_RoleInstance","")),
        OperationId=tostring(column_ifexists("operation_Id","")),
        SampleWeight=todouble(column_ifexists("itemCount",real(null))),
        SourceTable="customEvents";
```

## Source adapter: Log Analytics

```kusto
let RawRows=AppEvents
    | where TimeGenerated >= _startTime and TimeGenerated < _endTime
    | where Name in ("BatchTaskStart","BatchThrottled")
    | project EventTime=TimeGenerated, EventName=Name,
        Dimensions=todynamic(Properties),
        RoleInstance=tostring(column_ifexists("AppRoleInstance","")),
        OperationId=tostring(column_ifexists("OperationId","")),
        SampleWeight=todouble(column_ifexists("ItemCount",real(null))),
        SourceTable="AppEvents";
```

`todynamic` also parses JSON-string properties. Only top-level projections
change between sources; do not rename case-sensitive nested property keys.
Optional role, operation, and sampling fields remain empty/null when absent.
An empty role instance is unknown, not a synthesized server identity.

## Environment discovery

```kusto
RawRows
| extend LowerEnvironment=tolower(trim(@"\s+",tostring(Dimensions.environmentId))),
    UpperEnvironment=tolower(trim(@"\s+",tostring(Dimensions.EnvironmentId)))
| extend EnvironmentId=coalesce(LowerEnvironment,UpperEnvironment),
    ConflictingEnvironment=isnotempty(LowerEnvironment)
        and isnotempty(UpperEnvironment) and LowerEnvironment != UpperEnvironment
| summarize Rows=count() by EnvironmentId,ConflictingEnvironment
| order by Rows desc
```

Auto-select only when exactly one nonempty ID exists and there are no
conflicting keys. If no events exist, use the explicit/configured environment;
do not invent one. If unknown, ask for scope. Report missing-ID and
conflicting-ID totals even though scoped queries exclude them. Do not apply
an AOS filter to discovery; it could hide a multi-environment source.

## Normalization

```kusto
let Events=RawRows
    | extend LowerEnvironment=tolower(trim(@"\s+",tostring(Dimensions.environmentId))),
        UpperEnvironment=tolower(trim(@"\s+",tostring(Dimensions.EnvironmentId)))
    | extend EnvironmentId=coalesce(LowerEnvironment,UpperEnvironment),
        ConflictingEnvironment=isnotempty(LowerEnvironment)
            and isnotempty(UpperEnvironment) and LowerEnvironment != UpperEnvironment
    | where isnotempty(requestedEnvironmentId)
        and EnvironmentId == requestedEnvironmentId and not(ConflictingEnvironment)
    | where isempty(requestedRoleInstance) or RoleInstance =~ requestedRoleInstance
    | extend ActivityId=tolower(trim(@"\s+",coalesce(
            tostring(Dimensions.activityId),tostring(Dimensions.ActivityId),OperationId))),
        RawJobId=coalesce(tostring(Dimensions.BatchJobId),tostring(Dimensions.batchJobId)),
        LegalEntity=tostring(Dimensions.LegalEntity),
        EventId=coalesce(tostring(Dimensions.eventId),tostring(Dimensions.EventId))
    | extend BatchJobId=iff(trim(@"\s+",RawJobId) == "0","",trim(@"\s+",RawJobId));
```

Do not filter this dataset by job, activity, class, or legal entity: the
environment/AOS-level throttle signal may not populate those fields.
No event deduplication is applied. Report observed rows, with sampling caveats.

## Event coverage

```kusto
let Observed=Events
    | summarize Rows=count(),FirstSeen=min(EventTime),LastSeen=max(EventTime),
        MissingAOS=countif(isempty(RoleInstance)),
        MissingActivity=countif(isempty(ActivityId)),
        MissingEventType=countif(isempty(EventId)),
        RowsWithSamplingIndicator=countif(SampleWeight > 1),
        RowsWithUnknownSampleWeight=countif(
            isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1)
        by EventName;
datatable(EventName:string)["BatchTaskStart","BatchThrottled"]
| join kind=leftouter Observed on EventName
| project EventName,Rows=coalesce(Rows,0),FirstSeen,LastSeen,
    MissingAOS=coalesce(MissingAOS,0),MissingActivity=coalesce(MissingActivity,0),
    MissingEventType=coalesce(MissingEventType,0),
    RowsWithSamplingIndicator=coalesce(RowsWithSamplingIndicator,0),
    RowsWithUnknownSampleWeight=coalesce(RowsWithUnknownSampleWeight,0)
```

These are sparse event streams, not heartbeats. Do not use gaps between
throttles as proof of missing collection or sustained throttling.
An absent family can be quiet or not collected. Surface this uncertainty.

## Full-window KPIs

```kusto
Events
| summarize TaskStartEvents=countif(EventName == "BatchTaskStart"),
    ThrottleEvents=countif(EventName == "BatchThrottled"),
    AffectedAOS=count_distinctif(RoleInstance,
        EventName == "BatchThrottled" and isnotempty(RoleInstance)),
    ThrottleRowsWithoutAOS=countif(EventName == "BatchThrottled" and isempty(RoleInstance))
```

`AffectedAOS` counts known nodes; separately disclose unknown-node events.
Do not call it the full affected-node count when node IDs are missing.

## Dashboard-compatible timeline

```kusto
let Counts=Events
    | summarize RunningTasks=countif(EventName == "BatchTaskStart"),
        ThrottledTasks=countif(EventName == "BatchThrottled")
        by TimeBucket=bin(EventTime,_bin);
range TimeBucket from bin(_startTime,_bin) to _endTime step _bin
| where TimeBucket < _endTime
| join kind=leftouter Counts on TimeBucket
| project TimeBucket,RunningTasks=coalesce(RunningTasks,0),
    ThrottledTasks=coalesce(ThrottledTasks,0),
    IsCompleteBin=TimeBucket >= _startTime and TimeBucket + _bin <= _endTime
| order by TimeBucket asc
```

Use these exact two columns in the Markdown timeline and series in optional charts:
`RunningTasks` -> **Running tasks**, `ThrottledTasks` -> **Throttled tasks**.
The two series count starts and throttle emissions, respectively, NOT concurrency
and NOT unique tasks. Keep an explicit note with the table or chart. Do not add CPU,
memory, or SQL as extra series on this count axis.

Zero-fill only when the query succeeded without truncation. If retrieval is
partial or a subwindow fails, represent its bins as null, record the failure,
and do not interpolate across gaps. Reconcile full timeline sums with
`TaskStartEvents` and `ThrottleEvents`. Never generate a chart from the capped
event-detail rows. Edge bins retain observed counts, with `IsCompleteBin=false`.
Display UTC and the common bin interval. Counts have no resource units.

## Throttles dataset

```kusto
let MetricStatus=(raw:string) {
    let value=todouble(raw);
    case(isempty(trim(@"\s+",raw)),"Missing",
        isnull(value) or not(isfinite(value)),"NonNumericOrNonFinite",
        value < 0,"Negative","Present")
};
let MetricValue=(raw:string) {
    let value=todouble(raw);
    iff(isnotnull(value) and isfinite(value) and value >= 0,value,real(null))
};
let Throttles=Events
    | where EventName == "BatchThrottled"
    | extend InfoText=tostring(Dimensions.InfoMessage)
    | extend Info=parse_json(InfoText)
    | extend InfoStatus=case(isempty(InfoText),"Missing",
            gettype(Info) != "dictionary","Malformed","Parsed")
    | extend Info=iff(InfoStatus == "Parsed",Info,dynamic({}))
    | extend CurrentMachineCpu=MetricValue(tostring(Info.CurrentMachineCpu)),
        CurrentMachineMemory=MetricValue(tostring(Info.CurrentMachineMemory)),
        CurrentSqlDtu=MetricValue(tostring(Info.CurrentSqlDtu)),
        CpuStatus=MetricStatus(tostring(Info.CurrentMachineCpu)),
        MemoryStatus=MetricStatus(tostring(Info.CurrentMachineMemory)),
        SqlDtuStatus=MetricStatus(tostring(Info.CurrentSqlDtu));
```

The metric property names and nested-JSON parsing follow the supplied batch
schema and dashboard. Zero is retained. Missing, invalid, negative, and infinite
values yield null plus a diagnostic status. Do not round before assessment.
Malformed or missing payloads have all three metrics null with `Missing`
metric statuses; `InfoStatus` preserves the reason. A parsed object with bad
individual values instead reports their individual invalid statuses.

## Metric coverage

```kusto
Throttles
| summarize ThrottleEvents=count(),ParsedInfo=countif(InfoStatus == "Parsed"),
    MissingInfo=countif(InfoStatus == "Missing"),MalformedInfo=countif(InfoStatus == "Malformed"),
    CpuPresent=countif(CpuStatus == "Present"),CpuMissing=countif(CpuStatus == "Missing"),
    CpuInvalid=countif(CpuStatus in ("Negative","NonNumericOrNonFinite")),
    MemoryPresent=countif(MemoryStatus == "Present"),MemoryMissing=countif(MemoryStatus == "Missing"),
    MemoryInvalid=countif(MemoryStatus in ("Negative","NonNumericOrNonFinite")),
    SqlDtuPresent=countif(SqlDtuStatus == "Present"),SqlDtuMissing=countif(SqlDtuStatus == "Missing"),
    SqlDtuInvalid=countif(SqlDtuStatus in ("Negative","NonNumericOrNonFinite")),
    UnattributedJobRows=countif(isempty(BatchJobId))
```

Convert these counts to coverage statements in the report. When no throttles
were observed, skip detail/resource queries if desired, keep the zero count
from the successful KPI query, and explain that metric coverage is not
applicable. Do not show three metric zeros as if observations were collected.

## Per-AOS resource context

```kusto
Throttles
| summarize ThrottleEvents=count(),FirstSeen=min(EventTime),LastSeen=max(EventTime),
    CpuSamples=countif(CpuStatus == "Present"),PeakCpu=max(CurrentMachineCpu),
    MemorySamples=countif(MemoryStatus == "Present"),PeakMemory=max(CurrentMachineMemory),
    SqlDtuSamples=countif(SqlDtuStatus == "Present"),PeakSqlDtu=max(CurrentSqlDtu)
    by RoleInstance
| order by ThrottleEvents desc
```

The empty-role group is unknown, not an additional known server. Add
concentration and resource coverage to findings. Peaks for different resources
can come from different events; never assemble them into a fictional incident
row or infer that all peaks occurred simultaneously. Use the detail query for
co-occurring values. Do not sum resource metrics across AOS instances.

## Event detail

```kusto
Throttles
| project EventTime,EventId,EnvironmentId,RoleInstance,ActivityId,BatchJobId,
    LegalEntity,CurrentMachineCpu,CurrentMachineMemory,CurrentSqlDtu,
    InfoStatus,CpuStatus,MemoryStatus,SqlDtuStatus,
    ResourceAssessment="Trigger not established"
| order by EventTime asc,RoleInstance asc,ActivityId asc
| take 500
```

The limit caps retrieved detail, not aggregate analysis. In the
`throttle_events` insight, set `total_rows` to the full-window `ThrottleEvents`,
`returned_rows` to retrieved rows, and `data_truncated` when returned < total
or the tool reports truncation. Set `displayed_rows` to the rows actually shown
in Markdown (default up to 25) and `presentation_truncated` when displayed <
returned. Retain retrieved rows in the insight's `rows` array. If additional detail is
required, retrieve disjoint half-open slices and reconcile counts; do not page
only by `EventTime > lastTime`, which skips tied timestamps. If a single
timestamp exceeds the service cap, use the host's supported export/paging or
disclose the unresolved truncation. Never deduplicate by event type code.

This is the dashboard-like detail table with sensitive user/session fields and
raw `EventMessage` removed. All three metric columns remain visible. Add a
resource assessment only according to `references/interpretation.md`; absent
verified thresholds, keep the default. No query asserts that a metric caused
throttling.

## Output handoff

Map query output to `assets/findings-contract.md`. Store source table, exact UTC
scope, bin interval, collection uncertainty, sampling indicators, missing keys,
metric units, excluded rows, late-arrival caveat, and detail truncation in the
contract. The same canonical findings support both source adapters. Return
Markdown tables and findings by default. A timeline table is sufficient;
charts are optional and must not replace the underlying evidence. Apply
presentation limits only after aggregation, disclose omitted rows, and retain
the complete retrieved timeline for reconciliation and optional charts.
