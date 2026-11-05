# Batch task duration query library

## Source contract and automatic selection

Only customer-accessible event telemetry is needed. Select `customEvents` for
`QueryAppInsightsByAppId`/`QueryAppInsightsByResourceId`, and `AppEvents` for
`QueryLogAnalyticsByResourceId`/`QueryLogAnalyticsByWorkspaceId`. Generic
`kusto_query` requires an explicit compatible customer target. Probe the
preferred table with `<table> | take 0 | getschema`; confirm time/name/properties.
For an unknown generic target inspect metadata, otherwise prefer `customEvents`.
Probe the alternate table on the **same resource** only for confirmed missing
table, not empty data, timeout, access denial, or incompatible schema. Never
union both families or deduplicate cross-table fingerprints. Keep one selected
adapter for the whole report; the caller never rewrites queries.

| Property | Mapping and availability |
|---|---|
| Events | Exact `BatchTaskStart`, `BatchTaskFinished`, `BatchTaskFailure` |
| Job | `BatchJobId`, string, required exact filter; not occurrence ID |
| Task | Prefer `BatchTaskId`; optional runtime alias `BatchJobTaskId`; contradictory values quarantine both IDs |
| Environment | `environmentId`; optional casing alias `EnvironmentId`; normalize GUID casing, reject conflicts |
| Activity | Prefer custom `activityId`; optional casing alias `ActivityId`; reject conflicts |
| Operation | Optional top-level `operation_Id` / `OperationId`; absent from supplied general schema, fallback disabled by default |
| Duration | `elapsedMilliseconds` string on finished events only; finite nonnegative milliseconds |
| Metadata | `ClassName`, `LegalEntity`, `BatchJobCaption`; caption treated as text despite schema's inconsistent datetime annotation |
| AOS | `cloud_RoleInstance` / `AppRoleInstance`; optional, preserve full names |
| Sampling | `itemCount` / `ItemCount`; optional, count rows without multiplying |
| Retry | No retry-count field in supplied schema; unavailable, never inferred from repeated rows |
| Event identity | `eventId` is an event type code, not a unique emission/attempt ID |

The supplied JSON describes App Insights names; LA top-level mappings follow
the existing batch adapters and require runtime metadata confirmation. Optional
aliases likewise require coverage checks, not an assumption that they exist.
No numeric conversion, trimming, or lowercasing of job/task IDs is performed.
IDs with leading zeros or different case remain different. Whitespace-only
identifiers are unavailable; literal nonblank IDs are preserved exactly.

## Assembly

Submit **Parameters + exactly one adapter + Normalization + required datasets +
one terminal expression**. Definitions are query-local; do not include other
terminal outputs. Discovery needs normalization but no scope/pair definitions.
Coverage, inventory, placement, and emitted diagnostics need Scope. Pair
coverage, ranking, distributions, AOS duration, and trends also need Pairing.
Trend outputs additionally need **Trend datasets**, after **Pairing**. In
particular, `StartCohort` is a `let` binding defined near the end of **Pairing**,
not a stored table/function. Do not submit a fragment starting with it alone.
Replace all `{{Token}}` placeholders; use tool parameters where supported or
correctly escaped KQL literals, never raw user interpolation.

Freeze `_observedAt` once. Require `_startTime < _endTime <= _observedAt` and
nonempty exact job/environment (except environment during discovery). Reject
placeholder job ID `0`; do not broaden it. Default buffer 1 day; disclose and
adjust only deliberately, with the same rule for all periods. Require complete
query results before reporting exact metrics.

Baseline defaults on: normally attempt the immediately preceding equal-window
task comparison. Fetch that window too, and use the shared full horizon for
ambiguity detection, even if no prior pairs survive. This can conservatively
quarantine more keys; report it. Only an explicit opt-out or unavailable prior
source coverage justifies a current-only analysis; disclose it and any different
fetch horizon, and retain an unavailable trend section. `_fetchEnd` is an exclusive
event-time cutoff, not a declaration that tasks stopped then. Ingestion is
limited to frozen observation time where available, **not** historical end;
late arrivals can alter historical results on a later observation.

### Parameters

```kusto
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let _observedAt=datetime({{ObservedAtUtc}});
let _includeBaseline=true;
let _allowOperationFallback=false;
let _pairingBuffer=1d;
let _ingestionLag=5m;
let _trendMinSamples=5;
let _trendPercentThreshold=10.0;
let _trendAbsoluteFloorSec=1.0;
let _trendCoverageConfirmed=false;
let _previousStart=_startTime-(_endTime-_startTime);
let _analysisStart=iff(_includeBaseline,_previousStart,_startTime);
let _fetchStart=_analysisStart-_pairingBuffer;
let _fetchEnd=min_of(_observedAt,_endTime+_pairingBuffer);
let requestedEnvironmentId=tolower(trim(@"\s+","{{EnvironmentId}}"));
let requestedBatchJobId="{{BatchJobId}}";
```

`_allowOperationFallback` may become true only with recorded independent
attempt-level emitter/schema evidence, stable on both endpoints and not reused
across scheduled runs. No fallback to task ID alone, nearest timestamps, or
activity-only grouping. `_ingestionLag` is a reporting caveat, not a filter
that changes one event family's window.

Trend thresholds are configurable heuristics, not significance tests. Require
integer `_trendMinSamples >= 5`, finite positive percent/seconds thresholds,
positive `_pairingBuffer`, and nonnegative `_ingestionLag`; disclose changes.
Set `_trendCoverageConfirmed=true` only after complete discovery, current/prior
event and pair coverage establish retained source coverage, sufficient retention
and ingestion lag, no unresolved unassigned scope/ID gaps affecting this job,
and no partial/truncated queries. This is an analyst-confirmed query parameter,
not an invented telemetry field. Record the evidence or leave it false; weights
of one alone do not prove completeness. Task-specific guards still apply.
Failed/partial queries make affected outputs unavailable: this flag cannot make
partial counts valid. Do not change adapters, job/environment, or time bounds
between these checks and trend queries.

### Adapter: Application Insights

```kusto
let RawRows=customEvents
    | where timestamp >= _fetchStart and timestamp < _fetchEnd
    | where name in ("BatchTaskStart","BatchTaskFinished","BatchTaskFailure")
    | extend IngestedAt=ingestion_time()
    | where isnull(IngestedAt) or IngestedAt <= _observedAt
    | project EventTime=timestamp,EventName=name,Dimensions=todynamic(customDimensions),
        RoleInstance=tostring(column_ifexists("cloud_RoleInstance","")),
        OperationId=tostring(column_ifexists("operation_Id","")),
        SampleWeight=todouble(column_ifexists("itemCount",real(null))),IngestedAt;
```

### Adapter: Log Analytics

```kusto
let RawRows=AppEvents
    | where TimeGenerated >= _fetchStart and TimeGenerated < _fetchEnd
    | where Name in ("BatchTaskStart","BatchTaskFinished","BatchTaskFailure")
    | extend IngestedAt=ingestion_time()
    | where isnull(IngestedAt) or IngestedAt <= _observedAt
    | project EventTime=TimeGenerated,EventName=Name,Dimensions=todynamic(Properties),
        RoleInstance=tostring(column_ifexists("AppRoleInstance","")),
        OperationId=tostring(column_ifexists("OperationId","")),
        SampleWeight=todouble(column_ifexists("ItemCount",real(null))),IngestedAt;
```

Optional columns yield empty/null, never invented nodes. If the surface does not
support `ingestion_time()`, project `IngestedAt=datetime(null)` and disclose no
ingestion cutoff guarantee; do not silently change the requested event interval.

### Normalization

```kusto
let Normalized=RawRows
    | extend EnvLower=tolower(trim(@"\s+",tostring(Dimensions.environmentId))),
        EnvUpper=tolower(trim(@"\s+",tostring(Dimensions.EnvironmentId))),
        BatchJobId=tostring(Dimensions.BatchJobId),
        TaskPrimary=tostring(Dimensions.BatchTaskId),
        TaskAlias=tostring(Dimensions.BatchJobTaskId),
        ActLower=tolower(trim(@"\s+",tostring(Dimensions.activityId))),
        ActUpper=tolower(trim(@"\s+",tostring(Dimensions.ActivityId))),
        OperationId=tolower(trim(@"\s+",OperationId)),
        ClassName=tostring(Dimensions.ClassName),
        ElapsedText=tostring(Dimensions.elapsedMilliseconds)
    | extend EnvironmentId=coalesce(EnvLower,EnvUpper),
        EnvironmentConflict=isnotempty(EnvLower) and isnotempty(EnvUpper) and EnvLower != EnvUpper,
        TaskPrimary=iff(isempty(trim(@"\s+",TaskPrimary)),"",TaskPrimary),
        TaskAlias=iff(isempty(trim(@"\s+",TaskAlias)),"",TaskAlias),
        BatchJobId=iff(isempty(trim(@"\s+",BatchJobId)),"",BatchJobId),
        CustomActivity=coalesce(ActLower,ActUpper),
        ActivityConflict=isnotempty(ActLower) and isnotempty(ActUpper) and ActLower != ActUpper,
        EmittedRawMs=todouble(ElapsedText)
    | extend BatchTaskId=coalesce(TaskPrimary,TaskAlias),
        TaskConflict=isnotempty(TaskPrimary) and isnotempty(TaskAlias) and TaskPrimary != TaskAlias,
        ActivityId=iff(isnotempty(CustomActivity),CustomActivity,iff(_allowOperationFallback,OperationId,"")),
        ActivitySource=case(isnotempty(CustomActivity),"custom",
            _allowOperationFallback and isnotempty(OperationId),"operation","missing"),
        EmittedStatus=case(isempty(ElapsedText),"Missing",
            isnull(EmittedRawMs) or not(isfinite(EmittedRawMs)) or EmittedRawMs < 0,"Invalid","Valid")
    | extend EmittedElapsedMs=iff(EmittedStatus == "Valid",EmittedRawMs,real(null)),
        InCurrentWindow=EventTime >= _startTime and EventTime < _endTime;
```

### Discovery terminal (before environment/job exclusion)

```kusto
Normalized
| summarize FetchedRows=count(),CurrentWindowRows=countif(InCurrentWindow),
    MissingJob=countif(isempty(BatchJobId)),MissingTask=countif(isempty(BatchTaskId)),
    TaskAliasConflicts=countif(TaskConflict),ActivityAliasConflicts=countif(ActivityConflict)
    by EnvironmentId,EnvironmentConflict
| order by FetchedRows desc
```

Missing/conflicting environment rows stay visible here and are not attributable
to a selected job/environment. Discovery covers the full fetch horizon and
separately the current window. Auto-select only one nonempty nonconflicting
environment with no conflicts; otherwise require exact scope. Blank environment
rows still prevent a claim of complete source attribution. Missing job rows are
not assigned to the requested job. Successful discovery with no rows does not
authorize removing the job filter.

### Scope

```kusto
let Scoped=Normalized
    | where isnotempty(requestedEnvironmentId) and isnotempty(requestedBatchJobId)
        and requestedBatchJobId != "0"
    | where EnvironmentId == requestedEnvironmentId and not(EnvironmentConflict)
        and BatchJobId == requestedBatchJobId;
let WindowEvents=Scoped | where InCurrentWindow;
```

### Event coverage terminal

```kusto
WindowEvents
| summarize Rows=count(),StartRows=countif(EventName == "BatchTaskStart"),
    FinishRows=countif(EventName == "BatchTaskFinished"),
    FailureRows=countif(EventName == "BatchTaskFailure"),
    DistinctPersistentTasks=count_distinctif(BatchTaskId,isnotempty(BatchTaskId) and not(TaskConflict)),
    MissingTask=countif(isempty(BatchTaskId)),TaskAliasConflicts=countif(TaskConflict),
    MissingActivity=countif(isempty(ActivityId)),ActivityAliasConflicts=countif(ActivityConflict),
    OperationOnlyRows=countif(isempty(CustomActivity) and isnotempty(OperationId)),
    MissingClass=countif(isempty(ClassName)),MissingAOS=countif(isempty(RoleInstance)),
    SamplingIndicatedRows=countif(SampleWeight > 1 and isfinite(SampleWeight)),
    SamplingUnknownRows=countif(isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1),
    UnknownIngestionRows=countif(isnull(IngestedAt)),
    FirstObservedTaskEvent=min(EventTime),LastObservedTaskEvent=max(EventTime)
```

Zero counts are legitimate only on successful complete queries. Missing family
counts mean no rows observed, not proven absence of workload/failures. First/last
values optionally describe an observed task envelope, **not job duration**.

### Persistent task inventory terminal

```kusto
WindowEvents
| extend TaskIdentityStatus=case(TaskConflict,"ConflictingTaskAliases",
    isempty(BatchTaskId),"MissingTaskId","KnownTask")
| summarize Rows=count(),Starts=countif(EventName == "BatchTaskStart"),
    Finishes=countif(EventName == "BatchTaskFinished"),Failures=countif(EventName == "BatchTaskFailure"),
    Classes=make_set_if(ClassName,isnotempty(ClassName),20),MissingClassRows=countif(isempty(ClassName)),
    KnownAOS=count_distinctif(RoleInstance,isnotempty(RoleInstance)),
    MissingAOSRows=countif(isempty(RoleInstance)),
    MissingActivityRows=countif(isempty(ActivityId)),ActivityConflictRows=countif(ActivityConflict)
    by TaskIdentityStatus,BatchTaskId
| order by Rows desc,BatchTaskId asc
| take 500
```

Class lists are presentation-capped at 20; do not infer completeness from them.
The exact persistent task KPI comes from full coverage, not this limited table.
Conflicting/missing statuses are coverage buckets, not real task identifiers.
Keeping status separate avoids collisions with literal task ID strings.

### Emitted elapsed diagnostics terminal

```kusto
WindowEvents
| where EventName == "BatchTaskFinished"
| summarize FinishRows=count(),ValidEmittedRows=countif(EmittedStatus == "Valid"),
    MissingEmittedRows=countif(EmittedStatus == "Missing"),
    InvalidEmittedRows=countif(EmittedStatus == "Invalid"),
    ZeroEmittedRows=countif(EmittedStatus == "Valid" and EmittedElapsedMs == 0),
    MedianEmittedMs=percentile(EmittedElapsedMs,50),P95EmittedMs=percentile(EmittedElapsedMs,95)
```

This is a finish-event diagnostic cohort, not paired execution duration.

### Pairing

```kusto
let UnsafeTasks=Scoped
    | where TaskConflict or ActivityConflict or isempty(ActivityId)
    | extend CandidateTasks=pack_array(TaskPrimary,TaskAlias)
    | mv-expand UnsafeTaskId=CandidateTasks to typeof(string)
    | where isnotempty(UnsafeTaskId)
    | distinct EnvironmentId,BatchJobId,UnsafeTaskId;
let Tagged=Scoped
    | join kind=leftouter (UnsafeTasks | extend Unsafe=true)
        on EnvironmentId,BatchJobId,$left.BatchTaskId == $right.UnsafeTaskId
    | extend CorrelationUsable=isnotempty(BatchTaskId) and isnotempty(ActivityId)
        and not(TaskConflict) and not(ActivityConflict) and not(coalesce(Unsafe,false));
let Groups=Tagged
    | where CorrelationUsable
    | summarize Starts=countif(EventName == "BatchTaskStart"),
        Finishes=countif(EventName == "BatchTaskFinished"),Failures=countif(EventName == "BatchTaskFailure"),
        StartTime=minif(EventTime,EventName == "BatchTaskStart"),
        TerminalTime=minif(EventTime,EventName != "BatchTaskStart"),
        CurrentStarts=countif(EventName == "BatchTaskStart" and InCurrentWindow),
        PriorStarts=countif(EventName == "BatchTaskStart" and EventTime >= _previousStart and EventTime < _startTime),
        CurrentTerminals=countif(EventName != "BatchTaskStart" and InCurrentWindow),
        PriorTerminals=countif(EventName != "BatchTaskStart" and EventTime >= _previousStart and EventTime < _startTime),
        StartAOS=take_anyif(RoleInstance,EventName == "BatchTaskStart"),
        TerminalAOS=take_anyif(RoleInstance,EventName != "BatchTaskStart"),
        ClassCount=count_distinctif(ClassName,isnotempty(ClassName)),
        ClassValue=take_anyif(ClassName,isnotempty(ClassName)),
        MissingClassEvents=countif(isempty(ClassName)),
        EmittedMs=maxif(EmittedElapsedMs,EventName == "BatchTaskFinished"),
        InvalidEmittedEvents=countif(EventName == "BatchTaskFinished" and EmittedStatus == "Invalid"),
        SamplingIndicatedEvents=countif(SampleWeight > 1 and isfinite(SampleWeight)),
        SamplingUnknownEvents=countif(isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1)
        by EnvironmentId,BatchJobId,BatchTaskId,ActivitySource,ActivityId
    | extend PairStatus=case(Starts > 1 or Finishes > 1 or Failures > 1,"RepeatedEvents",
        Finishes > 0 and Failures > 0,"ConflictingTerminals",
        Starts == 0,"TerminalWithoutStart",Finishes+Failures == 0,"TerminalNotObserved",
        TerminalTime < StartTime,"ReversedTimestamps",Finishes == 1,"Completed","FailedObservedAttempt")
    | extend PairedElapsedMs=iff(PairStatus in ("Completed","FailedObservedAttempt"),
            todouble((TerminalTime-StartTime)/1ms),real(null)),
        ClassLabel=case(ClassCount == 0,"(missing)",ClassCount > 1,"(conflicting)",ClassValue),
        AOSAttribution=case(isempty(StartAOS) or isempty(TerminalAOS),"MissingEndpointAOS",
            StartAOS != TerminalAOS,"DifferentEndpointAOS","OneAOS")
    | extend DurationSec=iff(PairStatus == "Completed",PairedElapsedMs/1000.0,real(null)),
        FailedAttemptSec=iff(PairStatus == "FailedObservedAttempt",PairedElapsedMs/1000.0,real(null)),
        EmittedDeltaMs=iff(PairStatus == "Completed" and isnotnull(EmittedMs),
            EmittedMs-PairedElapsedMs,real(null)),
        AOSLabel=iff(AOSAttribution == "OneAOS",StartAOS,AOSAttribution)
    | extend EmittedDisagrees=isnotnull(EmittedDeltaMs)
        and abs(EmittedDeltaMs) > max_of(1000.0,0.05*PairedElapsedMs);
let StartCohort=Groups
    | extend Periods=pack_array(
        iff(CurrentStarts > 0,"Current",""),
        iff(_includeBaseline and PriorStarts > 0,"Prior",""))
    | mv-expand Period=Periods to typeof(string)
    | where isnotempty(Period);
let BoundaryGroups=Groups
    | extend Periods=pack_array(
        iff(CurrentStarts == 0 and CurrentTerminals > 0,"Current",""),
        iff(_includeBaseline and PriorStarts == 0 and PriorTerminals > 0,"Prior",""))
    | mv-expand Period=Periods to typeof(string)
    | where isnotempty(Period);
let Completed=StartCohort | where PairStatus == "Completed";
```

The group-by columns are the compound key; there is no delimiter-concatenated
key collision. Multiple tasks sharing one activity remain separate. Reused
activity/task combinations across runs are excluded as repeated, never split
into invented occurrences. No deduplication or pairing by nearest time occurs.
Missing-activity/conflicting rows taint matching persistent task IDs across the
fetch horizon, intentionally losing some usable pairs to avoid false matches.
Rows with no task cannot be linked to a known task; count and disclose this
residual uncertainty. Custom/operation provenance is never bridged.
Alias conflicts, missing environment/job, sampling, and undiscovered activity
reuse can limit confidence even for structurally valid pairs.

### Correlation exclusion coverage terminal

```kusto
Tagged
| summarize FetchedRows=count(),CurrentWindowRows=countif(InCurrentWindow),
    UnusableFetchedRows=countif(not(CorrelationUsable)),
    UnusableCurrentRows=countif(InCurrentWindow and not(CorrelationUsable)),
    QuarantinedCurrentRows=countif(InCurrentWindow and coalesce(Unsafe,false))
```

### Start-cohort coverage terminal

```kusto
StartCohort
| summarize CandidateGroups=count(),ValidCompleted=countif(PairStatus == "Completed"),
    ValidFailedAttempts=countif(PairStatus == "FailedObservedAttempt")
    by Period,PairStatus
| order by Period asc,PairStatus asc
```

Candidate groups count only usable-key groups with at least one start in that
period; excluded rows remain in the previous coverage result. A repeated group
with starts in both periods is visible in both, but never in completed metrics.

### Boundary and incomplete detail terminal

Run once using `BoundaryGroups`, and once replacing that source with
`StartCohort | where PairStatus != "Completed"`. Label the coverage cohort.

```kusto
BoundaryGroups
| extend CutoffAgeSec=iff(PairStatus == "TerminalNotObserved",
    todouble((_fetchEnd-StartTime)/1s),real(null))
| project Period,BatchTaskId,ActivityId,ActivitySource,PairStatus,Starts,Finishes,Failures,
    StartTime,TerminalTime,FailedAttemptSec,CutoffAgeSec
| order by Period asc,BatchTaskId asc,ActivityId asc
| take 500
```

Also obtain full counts via `BoundaryGroups | summarize Groups=count() by
Period,PairStatus` before the detail limit. Boundary-only valid completions are
not part of that period's duration cohort.

### Slowest completed terminal

```kusto
Completed
| where Period == "Current"
| top 100 by DurationSec desc
| project BatchTaskId,ActivityId,ActivitySource,ClassLabel,MissingClassEvents,
    StartTime,TerminalTime,DurationSec,EmittedMs,EmittedDeltaMs,EmittedDisagrees,
    StartAOS,TerminalAOS,AOSAttribution,SamplingIndicatedEvents,SamplingUnknownEvents
```

### Per-task distribution terminal

```kusto
Completed
| where Period == "Current"
| summarize CompletedN=count(),SumTaskSeconds=sum(DurationSec),
    MedianSec=percentile(DurationSec,50),P95Sec=percentile(DurationSec,95),MaxSec=max(DurationSec),
    EmittedComparableN=countif(isnotnull(EmittedMs)),
    EmittedDisagreementN=countif(EmittedDisagrees) by BatchTaskId
| order by SumTaskSeconds desc,BatchTaskId asc
```

Use `Completed | where Period == "Current" | summarize CompletedN=count(),
SumTaskSeconds=sum(DurationSec)` for full-job **task-work** totals. If N=0,
duration distributions/sums are unavailable for performance interpretation,
not evidence of zero-duration job work.

### AOS event placement terminal

```kusto
WindowEvents
| summarize Rows=count(),Starts=countif(EventName == "BatchTaskStart"),
    Finishes=countif(EventName == "BatchTaskFinished"),Failures=countif(EventName == "BatchTaskFailure"),
    PersistentTasks=count_distinctif(BatchTaskId,isnotempty(BatchTaskId) and not(TaskConflict))
    by RoleInstance
| order by Rows desc
```

Blank role is an unknown bucket, not a node. Obtain the full known AOS count with
`WindowEvents | summarize KnownAOS=count_distinctif(RoleInstance,
isnotempty(RoleInstance)),MissingAOSRows=countif(isempty(RoleInstance))`.
Starts measure event throughput/placement, not active concurrency.

### AOS completed duration terminal

```kusto
Completed
| where Period == "Current"
| summarize CompletedN=count(),SumTaskSeconds=sum(DurationSec),
    MedianSec=percentile(DurationSec,50),P95Sec=percentile(DurationSec,95)
    by AOSAttribution,AOSLabel
| order by SumTaskSeconds desc
```

Attribute pair-seconds to a node only for `OneAOS`; other buckets are unassigned.
Sum all buckets to reconcile completed N and task-seconds to the full population.
Do not divide by thread capacity or infer AOS CPU, SQL, or node-level causation.

### Prior-period coverage assembly

Before interpreting trends, run **Event coverage terminal** and **Persistent
task inventory terminal** for each period. Each is a separate query assembled
as **Parameters + selected Adapter: Application Insights OR Adapter: Log
Analytics + Normalization + Scope + that one terminal**. For Prior only, replace
the `WindowEvents` binding in **Scope** with:

```kusto
let WindowEvents=Scoped
    | where EventTime >= _previousStart and EventTime < _startTime;
```

This binding alone is not executable: `Scoped` comes from **Scope**, which
depends on **Normalization** and the selected adapter. Keep
`_includeBaseline=true`, frozen bounds, and all other parameters unchanged.
Label the output Prior. Do not replace `InCurrentWindow` or any **Pairing**
definition. Run **Start-cohort coverage terminal** separately using
**Parameters + one selected adapter + Normalization + Scope + Pairing +
Start-cohort coverage terminal**; it returns both Period values. The full
**Correlation exclusion coverage terminal** assembly has the same prefix
through Pairing. None of these outputs is a persisted dataset.

### Trend datasets

Append this complete block after **Parameters + exactly one selected Adapter:
Application Insights OR Adapter: Log Analytics + Normalization + Scope +
Pairing**. Then append exactly one of **Per-task trend terminal**, **Trend
roll-up terminal**, or **Worst regressions terminal**. Use the unchanged current
`WindowEvents` binding from Scope; trend periods are assigned explicitly here.
`Tagged`, `StartCohort`, `BoundaryGroups`, and their columns are all query-local bindings from
**Pairing**. `TrendEvents`, `TrendEventStats`, `TrendGroupStats`,
`TrendBoundaryStats`, `TrendClassStats`, and `TaskTrends` below are new query-local bindings only,
never assumed tables, stored functions, or separately submitted queries.

The task universe comes from known, nonconflicting IDs in either event window,
not only `Completed` or `StartCohort`: unpairable and terminal-only tasks must
remain visible. Missing/conflicting task IDs remain separate coverage buckets,
not invented task IDs; unresolved attribution keeps coverage unconfirmed.
Class evidence includes raw event-window rows and start-cohort endpoints in the
buffer. Compare uncropped sets, verify the set-size cap against exact counts,
and independently reject conflicting/missing endpoint class evidence.

Pairing is unchanged. For a directional comparison, require a fully observed
current buffer plus ingestion lag and all selected start groups completed
within the same per-start `_pairingBuffer` bound in both periods. This prevents
extra prior follow-up from silently admitting longer prior attempts. Do not
discard longer completions or incomplete groups: show the original completed
N/median/p95, flag the task incomparable (or insufficient), and retain detail.
Never shorten the shared horizon or split groups to improve eligibility.

```kusto
let TrendEvents=Tagged
    | extend Period=case(InCurrentWindow,"Current",
        _includeBaseline and EventTime >= _previousStart and EventTime < _startTime,"Prior","")
    | where isnotempty(Period) and isnotempty(BatchTaskId) and not(TaskConflict);
let TrendEventStats=TrendEvents
    | summarize CurrentRows=countif(Period == "Current"),PriorRows=countif(Period == "Prior"),
        UnusableRows=countif(not(CorrelationUsable)),
        MissingClassRows=countif(isempty(trim(@"\s+",ClassName))),
        SamplingIndicatedRows=countif(SampleWeight > 1 and isfinite(SampleWeight)),
        SamplingUnknownRows=countif(isnull(SampleWeight) or not(isfinite(SampleWeight)) or SampleWeight < 1)
        by BatchTaskId;
let TrendGroupStats=StartCohort
    | summarize CurrentGroups=countif(Period == "Current"),PriorGroups=countif(Period == "Prior"),
        CurrentN=countif(Period == "Current" and PairStatus == "Completed"),
        PriorN=countif(Period == "Prior" and PairStatus == "Completed"),
        CurrentMedianSec=percentileif(DurationSec,50,Period == "Current" and PairStatus == "Completed"),
        PriorMedianSec=percentileif(DurationSec,50,Period == "Prior" and PairStatus == "Completed"),
        CurrentP95Sec=percentileif(DurationSec,95,Period == "Current" and PairStatus == "Completed"),
        PriorP95Sec=percentileif(DurationSec,95,Period == "Prior" and PairStatus == "Completed"),
        NonCompletedGroups=countif(PairStatus != "Completed"),
        BeyondFollowupGroups=countif(PairStatus == "Completed" and TerminalTime > StartTime+_pairingBuffer),
        MissingClassGroups=countif(MissingClassEvents > 0 or ClassCount == 0),
        ConflictingClassGroups=countif(ClassCount > 1),
        PairSamplingIssueGroups=countif(SamplingIndicatedEvents > 0 or SamplingUnknownEvents > 0)
        by BatchTaskId;
let TrendBoundaryStats=BoundaryGroups
    | summarize UnpairedBoundaryGroups=countif(PairStatus !in ("Completed","FailedObservedAttempt"))
        by BatchTaskId;
let TrendClassStats=union
    (TrendEvents | project BatchTaskId,Period,ClassName=trim(@"\s+",ClassName)),
    (StartCohort | project BatchTaskId,Period,ClassName=trim(@"\s+",ClassValue))
    | summarize CurrentClasses=make_set_if(ClassName,Period == "Current" and isnotempty(ClassName),1048576),
        PriorClasses=make_set_if(ClassName,Period == "Prior" and isnotempty(ClassName),1048576),
        CurrentClassCount=count_distinctif(ClassName,Period == "Current" and isnotempty(ClassName)),
        PriorClassCount=count_distinctif(ClassName,Period == "Prior" and isnotempty(ClassName)),
        BlankClassEvidence=countif(isempty(ClassName))
        by BatchTaskId;
let TaskTrends=TrendEventStats
    | join kind=leftouter TrendGroupStats on BatchTaskId
    | join kind=leftouter TrendBoundaryStats on BatchTaskId
    | join kind=leftouter TrendClassStats on BatchTaskId
    | extend CurrentN=coalesce(CurrentN,long(0)),PriorN=coalesce(PriorN,long(0)),
        CurrentGroups=coalesce(CurrentGroups,long(0)),PriorGroups=coalesce(PriorGroups,long(0)),
        NonCompletedGroups=coalesce(NonCompletedGroups,long(0)),
        BeyondFollowupGroups=coalesce(BeyondFollowupGroups,long(0)),
        UnpairedBoundaryGroups=coalesce(UnpairedBoundaryGroups,long(0)),
        MissingClassGroups=coalesce(MissingClassGroups,long(0)),
        ConflictingClassGroups=coalesce(ConflictingClassGroups,long(0)),
        PairSamplingIssueGroups=coalesce(PairSamplingIssueGroups,long(0))
    | extend PopulationStatus=case(CurrentRows == 0,"PriorOnly",PriorRows == 0,"CurrentOnly","Both"),
        ClassSetsDiffer=array_length(set_difference(CurrentClasses,PriorClasses)) > 0
            or array_length(set_difference(PriorClasses,CurrentClasses)) > 0,
        ClassSetTruncated=CurrentClassCount != array_length(CurrentClasses)
            or PriorClassCount != array_length(PriorClasses),
        InvalidDuration=isnull(CurrentMedianSec) or not(isfinite(CurrentMedianSec)) or CurrentMedianSec < 0
            or isnull(PriorMedianSec) or not(isfinite(PriorMedianSec)) or PriorMedianSec < 0
            or isnull(CurrentP95Sec) or not(isfinite(CurrentP95Sec)) or CurrentP95Sec < 0
            or isnull(PriorP95Sec) or not(isfinite(PriorP95Sec)) or PriorP95Sec < 0
    | extend CoverageFlags=set_difference(pack_array(
        iff(not(_includeBaseline),"BaselineDisabled",""),
        iff(not(_trendCoverageConfirmed),"CollectionCoverageUnconfirmed",""),
        iff(_fetchEnd < _endTime+_pairingBuffer
            or _observedAt < _endTime+_pairingBuffer+_ingestionLag,"UnequalOrImmatureFollowup",""),
        iff(UnusableRows > 0,"UnusableCorrelationRows",""),
        iff(NonCompletedGroups > 0,"NonCompletedStartGroups",""),
        iff(UnpairedBoundaryGroups > 0,"UnpairedBoundaryGroups",""),
        iff(BeyondFollowupGroups > 0,"BeyondEqualFollowupBound",""),
        iff(MissingClassRows+MissingClassGroups+BlankClassEvidence > 0,"MissingClassMetadata",""),
        iff(ConflictingClassGroups > 0,"ConflictingClassMetadata",""),
        iff(ClassSetsDiffer,"ChangedClassSets",""),
        iff(ClassSetTruncated,"ClassSetTruncated",""),
        iff(SamplingIndicatedRows+SamplingUnknownRows+PairSamplingIssueGroups > 0,"SamplingOrUnknownWeights","")),
        dynamic([""]))
    | extend ComparisonStatus=case(CurrentN < _trendMinSamples or PriorN < _trendMinSamples,"InsufficientSamples",
        InvalidDuration,"MissingDuration",PriorMedianSec == 0,"ZeroPriorMedian",
        array_length(CoverageFlags) > 0,"Incomparable","Comparable")
    | extend MedianImprovementSec=iff(not(InvalidDuration),
            PriorMedianSec-CurrentMedianSec,real(null)),
        SafePriorMedianSec=iff(ComparisonStatus == "Comparable" and PriorMedianSec > 0,
            PriorMedianSec,1.0)
    | extend ImprovementPct=iff(ComparisonStatus == "Comparable",
        100.0*coalesce(MedianImprovementSec,0.0)/SafePriorMedianSec,real(null))
    | extend TrendStatus=case(ComparisonStatus != "Comparable",ComparisonStatus,
        ImprovementPct >= _trendPercentThreshold and MedianImprovementSec >= _trendAbsoluteFloorSec,"Improved",
        ImprovementPct <= -_trendPercentThreshold and MedianImprovementSec <= -_trendAbsoluteFloorSec,"Regressed",
        "Stable")
    | extend TrendBucket=iff(TrendStatus in ("InsufficientSamples","MissingDuration","ZeroPriorMedian"),
        "Insufficient",TrendStatus);
```

N=0 above is a known absence of valid completions only after a successful
complete query, including tasks with no `StartCohort` rows. Never coalesce
percentiles to zero. The internal safe denominator (1 only for ineligible rows)
and numerator guard prevent zero/null division even under eager evaluation;
the outer eligibility gate returns null for those rows. These guards are not
substituted evidence: unavailable comparisons stay null, not zero or infinity.
Keep absolute median differences as descriptive evidence, not a directional
claim when comparability fails. Classify on unrounded values.

### Per-task trend terminal

Complete assembly: **Parameters + exactly one Adapter: Application Insights OR
Adapter: Log Analytics + Normalization + Scope + Pairing + Trend datasets +
this terminal**. `TaskTrends` is defined in **Trend datasets** above; it is not a
table. Use identical frozen parameters and confirmed coverage for all three
trend terminals. Return full task aggregates; apply the normal 25-row display
limit only after receiving complete results, and report hidden rows. If tool
limits prevent complete retrieval, follow the full-horizon partition rules in
SKILL.md or mark the affected output unavailable.

```kusto
TaskTrends
| project BatchTaskId,PopulationStatus,CurrentRows,PriorRows,CurrentGroups,PriorGroups,
    CurrentN,PriorN,CurrentMedianSec,PriorMedianSec,CurrentP95Sec,PriorP95Sec,
    MedianImprovementSec,ImprovementPct,TrendStatus,TrendBucket,CoverageFlags,
    CurrentClasses,PriorClasses,CurrentClassCount,PriorClassCount,
    UnusableRows,NonCompletedGroups,UnpairedBoundaryGroups,BeyondFollowupGroups,MissingClassRows,
    MissingClassGroups,ConflictingClassGroups,SamplingIndicatedRows,
    SamplingUnknownRows,PairSamplingIssueGroups
| order by TrendStatus asc,ImprovementPct asc,BatchTaskId asc
```

### Trend roll-up terminal

Complete assembly: **Parameters + exactly one Adapter: Application Insights OR
Adapter: Log Analytics + Normalization + Scope + Pairing + Trend datasets +
this terminal**. It consumes the query-local `TaskTrends`, not detail output.
Compute before any presentation limit; never average percentages or pool
changing populations into a job-wide duration trend.

```kusto
TaskTrends
| summarize TotalTasks=count(),ImprovedTasks=countif(TrendBucket == "Improved"),
    RegressedTasks=countif(TrendBucket == "Regressed"),StableTasks=countif(TrendBucket == "Stable"),
    IncomparableTasks=countif(TrendBucket == "Incomparable"),
    InsufficientTasks=countif(TrendBucket == "Insufficient"),
    ComparableTasks=countif(ComparisonStatus == "Comparable"),
    CurrentOnlyTasks=countif(PopulationStatus == "CurrentOnly"),
    PriorOnlyTasks=countif(PopulationStatus == "PriorOnly"),
    BothPeriodTasks=countif(PopulationStatus == "Both")
```

The five trend bucket counts must sum to TotalTasks; the three population
counts must also sum to it. A successful empty union yields zero tasks, not
evidence of stable performance. If ComparableTasks=0 say “No comparable task
duration trends”; explain insufficiency/coverage, even if completed pairs exist.
If the baseline query fails, do not emit this roll-up as though it were complete.

### Worst regressions terminal

Complete assembly: **Parameters + exactly one Adapter: Application Insights OR
Adapter: Log Analytics + Normalization + Scope + Pairing + Trend datasets +
this terminal**. It consumes the query-local `TaskTrends`; it does not consume
the per-task terminal's display-limited rows.

```kusto
TaskTrends
| where TrendStatus == "Regressed"
| extend MedianSlowdownSec=-MedianImprovementSec
| project BatchTaskId,CurrentClasses,PriorClasses,CurrentN,PriorN,
    CurrentMedianSec,PriorMedianSec,CurrentP95Sec,PriorP95Sec,
    MedianSlowdownSec,ImprovementPct,TrendStatus,CoverageFlags
| order by MedianSlowdownSec desc,ImprovementPct asc,BatchTaskId asc
| take 25
```

Keep both cohorts' evidence and corresponding completed-attempt details in the
handoff. For a task drilldown use **Parameters + exactly one Adapter: Application
Insights OR Adapter: Log Analytics + Normalization + Scope + Pairing**, then
`Completed | where BatchTaskId ==
"<escaped-exact-task-id>" | order by DurationSec desc | take 100` as the single
terminal; retain Period in output. Filter only after full-horizon Pairing,
never before. A lack of comparable regressions is not proof of improvement.
Neither duration shifts nor unchanged classes establish constant workload
volume or task-code causation. These are completed-attempt comparisons, never
observed full durations of currently running/incomplete tasks.
