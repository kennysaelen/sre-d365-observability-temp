# Batch workload query library

The supplied schema files describe Application Insights names. The queries
below normalize both Application Insights and Log Analytics:

| Canonical field | Application Insights | Log Analytics |
|---|---|---|
| Event table | `customEvents` | `AppEvents` |
| Exception table | `exceptions` | `AppExceptions` |
| Time | `timestamp` | `TimeGenerated` |
| Event name | `name` | `Name` |
| Properties | `customDimensions` | `Properties` |
| AOS instance | `cloud_RoleInstance` | `AppRoleInstance` |
| Operation | `operation_Id` | `OperationId` |

## Automatic source resolution

The skill selects the adapter internally. Neither the user nor the caller
agent needs to supply table names, a dialect flag, or rewritten KQL. Use the
supplied resource or the unambiguous configured resource and keep that target
fixed throughout discovery and analysis.

| Selected tool | Preferred event/exception tables |
|---|---|
| `QueryAppInsightsByAppId`, `QueryAppInsightsByResourceId` | `customEvents`, `exceptions` |
| `QueryLogAnalyticsByResourceId`, `QueryLogAnalyticsByWorkspaceId` | `AppEvents`, `AppExceptions` |
| `kusto_query` | Discover the schema on the explicitly supplied target |

Probe the preferred event table with `<table> | take 0 | getschema`, using
exactly one table name from the mapping. This returns metadata, not raw events.
Confirm required time, name, properties, and role-instance columns. For an
unknown generic target, inspect available schema metadata first; if unavailable,
probe `customEvents` first and `AppEvents` only on a confirmed missing-table
response. A zero-row data result is not a missing table.

If the preferred table is genuinely absent, probe the alternate table on the
same target and use its adapter when the schema matches. Do not switch resources
or treat authorization failures, timeouts, or incompatible schemas as empty
telemetry. Surface those errors. When both families are exposed, retain the
tool-preferred family; for a generic target without a declared dialect, prefer
`customEvents` when both schemas match. Never concatenate or union both views.

Probe the corresponding exception table separately. If it is absent, probe the
alternate exception table on the same target; its adapter may be selected
independently. If neither exists, report exception telemetry as unavailable and
continue event analysis, not as zero exceptions. Ask for clarification only if
the target resource itself is ambiguous, not to choose a dialect.

Replace every `{{Token}}` before validation or execution. Every submitted query
must include its required `let` definitions; `BatchEvents`, `TaskExecutions`,
and `BatchExceptionSignals` are not persistent tables.

## Common event adapter

Assemble each event query in this order: **parameters below + exactly one event
source adapter + shared event normalization + required analysis definitions +
one terminal result expression**. References to "Common event adapter" below
mean this assembled prefix. Keep analysis definitions only (not an earlier
query's terminal result) when reusing datasets such as `ThreadSamples`.
Validate the assembled query on the selected surface when supported.

### Event parameters

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let _pairingBuffer = 1d;
let _fetchStart = _startTime - _pairingBuffer;
let _fetchEnd = min_of(now(), _endTime + _pairingBuffer);
let _queryNow = now();
let _ingestionLag = 5m;
let requestedEnvironmentId = tolower("{{EnvironmentId}}");
let requestedBatchJobId = tolower("{{BatchJobId}}");
let requestedBatchTaskId = tolower("{{BatchTaskId}}");
let requestedActivityId = tolower("{{ActivityId}}");
let requestedClassName = "{{ClassName}}";
let requestedLegalEntity = "{{LegalEntity}}";
let requestedPriority = "{{Priority}}";
let requestedRoleInstance = "{{RoleInstance}}";
```

### Event source: Application Insights

```kusto
let RawEventRows = customEvents
    | where timestamp between (_fetchStart .. _fetchEnd)
    | project
        EventTime=timestamp,
        EventName=name,
        Dimensions=todynamic(customDimensions),
        RoleInstance=tostring(cloud_RoleInstance),
        OperationId=tolower(tostring(column_ifexists("operation_Id",""))),
        SourceTable="customEvents";
```

### Event source: Log Analytics

```kusto
let RawEventRows = AppEvents
    | where TimeGenerated between (_fetchStart .. _fetchEnd)
    | project
        EventTime=TimeGenerated,
        EventName=Name,
        Dimensions=todynamic(Properties),
        RoleInstance=tostring(AppRoleInstance),
        OperationId=tolower(tostring(column_ifexists("OperationId",""))),
        SourceTable="AppEvents";
```

Both adapters return the same columns. `todynamic` (equivalent to `parse_json`)
handles either a dynamic property bag or a JSON string. Adapt only the source
projection if schema discovery identifies a different role-instance column.
If no role-instance column exists, explicitly project `RoleInstance=""` and
report AOS-specific analysis as unavailable; do not combine unknown AOS rows as
one server. Missing optional operation IDs remain empty and appear in activity
coverage when no custom activity ID is present.

### Shared event normalization

```kusto
let BatchEvents = RawEventRows
    | where EventName in~ (
        "BatchTaskStart",
        "BatchTaskFinished",
        "BatchTaskFailure",
        "BatchThrottled",
        "BatchQueuesDetails",
        "BatchThreadInfo")
    | extend Info=todynamic(tostring(Dimensions.InfoMessage))
    | extend
        EnvironmentId=tolower(coalesce(
            tostring(Dimensions.environmentId),
            tostring(Dimensions.EnvironmentId))),
        ActivityId=tolower(coalesce(
            tostring(Dimensions.activityId),
            tostring(Dimensions.ActivityId),
            OperationId)),
        BatchJobId=tolower(coalesce(
            tostring(Dimensions.BatchJobId),
            tostring(Dimensions.batchJobId))),
        BatchTaskId=tolower(coalesce(
            tostring(Dimensions.BatchJobTaskId),
            tostring(Dimensions.BatchTaskId),
            tostring(Dimensions.batchTaskId))),
        BatchJobCaption=tostring(Dimensions.BatchJobCaption),
        ClassName=tostring(Dimensions.ClassName),
        LegalEntity=tostring(Dimensions.LegalEntity),
        Priority=tostring(Dimensions.Priority),
        Critical=tobool(Dimensions.Critical),
        EventId=tostring(Dimensions.eventId),
        ExceptionType=tostring(Dimensions.ExceptionType),
        ExceptionSource=tostring(Dimensions.ExceptionSource),
        EmittedElapsedMs=todouble(Dimensions.elapsedMilliseconds)
    | extend CorrelationKey=case(
        isnotempty(ActivityId), strcat("activity:",ActivityId),
        isnotempty(BatchJobId) and isnotempty(BatchTaskId),
            strcat("task:",BatchJobId,":",BatchTaskId),
        "")
    | where isempty(requestedEnvironmentId) or EnvironmentId == requestedEnvironmentId
    | where isempty(requestedBatchJobId) or BatchJobId == requestedBatchJobId
    | where isempty(requestedBatchTaskId) or BatchTaskId == requestedBatchTaskId
    | where isempty(requestedActivityId) or ActivityId == requestedActivityId
    | where isempty(requestedClassName) or ClassName =~ requestedClassName
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity
    | where isempty(requestedPriority) or Priority =~ requestedPriority
    | where isempty(requestedRoleInstance) or RoleInstance =~ requestedRoleInstance;
```

Only one source table is read, so cross-table fingerprinting and deduplication
are unnecessary. Preserve repeated source events: the task-pairing logic below
must still identify repeated starts or terminals as ambiguous, not silently
discard them. Analytical `summarize` operations remain unchanged.

## Schema and coverage probe

Run this before health interpretation.

```kusto
// Include the Common event adapter.
let WindowEvents=BatchEvents
    | where EventTime between (_startTime .. _endTime);
WindowEvents
| extend InfoText=tostring(Dimensions.InfoMessage)
| summarize
    Events=count(),
    FirstSeen=min(EventTime),
    LastSeen=max(EventTime),
    Environments=make_set_if(EnvironmentId,isnotempty(EnvironmentId),20),
    MissingEnvironment=countif(isempty(EnvironmentId)),
    MissingActivity=countif(isempty(ActivityId)),
    MissingJob=countif(isempty(BatchJobId)),
    MissingTask=countif(isempty(BatchTaskId)),
    MissingRoleInstance=countif(isempty(RoleInstance)),
    MissingInfoMessage=countif(
        EventName in~ ("BatchThreadInfo","BatchQueuesDetails","BatchThrottled")
        and isempty(InfoText)),
    MalformedInfoMessage=countif(
        EventName in~ ("BatchThreadInfo","BatchQueuesDetails","BatchThrottled")
        and isnotempty(InfoText)
        and gettype(Info) != "dictionary")
    by EventName,SourceTable
| order by EventName asc
```

Expected events can legitimately be absent in a quiet window. Absence becomes a
coverage limitation when the report requires that signal.

## Paired task-execution dataset

```kusto
// Include the Common event adapter.
let TaskEventRows=BatchEvents
    | where EventName in~ ("BatchTaskStart","BatchTaskFinished","BatchTaskFailure")
    | where isnotempty(CorrelationKey);
let TaskExecutions=TaskEventRows
    | summarize
        StartTime=minif(EventTime,EventName =~ "BatchTaskStart"),
        FinishTime=maxif(EventTime,EventName =~ "BatchTaskFinished"),
        FailureTime=maxif(EventTime,EventName =~ "BatchTaskFailure"),
        StartEvents=countif(EventName =~ "BatchTaskStart"),
        FinishEvents=countif(EventName =~ "BatchTaskFinished"),
        FailureEvents=countif(EventName =~ "BatchTaskFailure"),
        EnvironmentId=take_anyif(EnvironmentId,isnotempty(EnvironmentId)),
        ActivityId=take_anyif(ActivityId,isnotempty(ActivityId)),
        BatchJobId=take_anyif(BatchJobId,isnotempty(BatchJobId)),
        BatchTaskId=take_anyif(BatchTaskId,isnotempty(BatchTaskId)),
        BatchJobCaption=take_anyif(BatchJobCaption,isnotempty(BatchJobCaption)),
        ClassName=take_anyif(ClassName,isnotempty(ClassName)),
        LegalEntity=take_anyif(LegalEntity,isnotempty(LegalEntity)),
        Priority=take_anyif(Priority,isnotempty(Priority)),
        RoleInstance=take_anyif(RoleInstance,isnotempty(RoleInstance)),
        ExceptionType=take_anyif(ExceptionType,isnotempty(ExceptionType)),
        ExceptionSource=take_anyif(ExceptionSource,isnotempty(ExceptionSource)),
        EmittedElapsedMs=maxif(EmittedElapsedMs,EventName =~ "BatchTaskFinished")
        by CorrelationKey
    | extend TerminalTime=max_of(FinishTime,FailureTime)
    | extend PairStatus=case(
        StartEvents > 1 or FinishEvents > 1 or FailureEvents > 1,"RepeatedEvents",
        FinishEvents > 0 and FailureEvents > 0,"ConflictingTerminal",
        StartEvents == 0 and FinishEvents + FailureEvents == 1,"LeftTruncated",
        StartEvents == 1 and FinishEvents == 0 and FailureEvents == 0,"UnfinishedOrRunning",
        StartEvents == 1 and FailureEvents == 1,"Failed",
        StartEvents == 1 and FinishEvents == 1,"Completed",
        "Ambiguous")
    | extend PairStatus=iff(
        isnotnull(StartTime) and isnotnull(TerminalTime) and TerminalTime < StartTime,
        "ReversedTimestamps",
        PairStatus)
    | extend ObservedDurationMs=iff(
        PairStatus in ("Completed","Failed"),
        todouble((TerminalTime-StartTime)/1ms),
        real(null))
    | extend DurationDifferencePct=iff(
        PairStatus == "Completed"
        and EmittedElapsedMs > 0.0
        and ObservedDurationMs >= 0.0,
        round(100.0*(ObservedDurationMs-EmittedElapsedMs)/EmittedElapsedMs,1),
        real(null))
    | where
        (StartTime between (_startTime .. _endTime))
        or (PairStatus == "LeftTruncated" and TerminalTime between (_startTime .. _endTime));
```

### Pairing coverage and KPIs

```kusto
// Include the Common event adapter and TaskExecutions.
let _cutoff=min_of(_endTime,_queryNow-_ingestionLag);
let EligibleStarted=TaskExecutions
    | where StartTime between (_startTime .. _cutoff);
EligibleStarted
| summarize
    Started=count(),
    Completed=countif(PairStatus == "Completed"),
    Failed=countif(PairStatus == "Failed"),
    Unfinished=countif(PairStatus == "UnfinishedOrRunning"),
    Ambiguous=countif(PairStatus in (
        "RepeatedEvents","ConflictingTerminal","ReversedTimestamps","Ambiguous"))
| extend
    CompletionRatePct=iff(
        Started > 0,
        round(100.0*todouble(Completed)/todouble(Started),1),
        real(null)),
    FailureRatePct=iff(
        Started > 0,
        round(100.0*todouble(Failed)/todouble(Started),1),
        real(null))
```

Also report left-truncated rows separately:

```kusto
TaskExecutions
| summarize Executions=count() by PairStatus
| order by Executions desc
```

An unfinished row near `_cutoff` is not automatically stuck. Rank with silence,
class baseline, and historical-window context.

### Apparently unfinished tasks

```kusto
// Include the Common event adapter and TaskExecutions.
let _cutoff=min_of(_endTime,_queryNow-_ingestionLag);
TaskExecutions
| where PairStatus == "UnfinishedOrRunning"
| where StartTime <= _cutoff
| extend AgeMinutes=round((_cutoff-StartTime)/1m,1)
| extend State=case(
    StartTime < _startTime + _pairingBuffer,"BoundaryRisk",
    AgeMinutes < 15.0,"RecentlyStarted",
    AgeMinutes < 60.0,"Silent15m",
    "Silent60m")
| top 100 by AgeMinutes desc
| project
    State,
    AgeMinutes,
    EnvironmentId,
    BatchJobCaption,
    ClassName,
    BatchJobId,
    BatchTaskId,
    ActivityId,
    RoleInstance,
    Priority,
    StartTime
```

`Silent15m` and `Silent60m` are triage states, not proof that the task is stuck.

## Slowest completed tasks

```kusto
// Include the Common event adapter and TaskExecutions.
TaskExecutions
| where PairStatus == "Completed"
| top 100 by ObservedDurationMs desc
| project
    BatchJobCaption,
    ClassName,
    BatchJobId,
    BatchTaskId,
    ActivityId,
    RoleInstance,
    Priority,
    StartTime,
    FinishTime,
    ObservedDurationMs=round(ObservedDurationMs,0),
    EmittedElapsedMs=round(EmittedElapsedMs,0),
    DurationDifferencePct
```

Flag duration disagreement only after choosing a documented threshold, such as
an absolute difference over 30 seconds and a relative difference over 10%.

## Batch-job workload summary

Build this from the full `TaskExecutions` dataset, never from a top-N subset.

```kusto
// Include the Common event adapter and TaskExecutions.
TaskExecutions
| where StartTime between (_startTime .. _endTime)
| extend JobLabel=case(
    isnotempty(BatchJobCaption),BatchJobCaption,
    isnotempty(ClassName),ClassName,
    isnotempty(BatchJobId),strcat("Job ",BatchJobId),
    "(unidentified)")
| summarize
    Tasks=count(),
    Completed=countif(PairStatus == "Completed"),
    Failed=countif(PairStatus == "Failed"),
    Unfinished=countif(PairStatus == "UnfinishedOrRunning"),
    Ambiguous=countif(PairStatus !in (
        "Completed","Failed","UnfinishedOrRunning","LeftTruncated")),
    CumulativeDurationMs=round(sumif(ObservedDurationMs,PairStatus == "Completed"),0),
    MedianDurationMs=round(percentileif(ObservedDurationMs,50,PairStatus == "Completed"),0),
    P95DurationMs=round(percentileif(ObservedDurationMs,95,PairStatus == "Completed"),0),
    MaxDurationMs=round(maxif(ObservedDurationMs,PairStatus == "Completed"),0),
    AOSInstances=make_set_if(RoleInstance,isnotempty(RoleInstance),20)
    by JobLabel,BatchJobId,ClassName
| top 50 by CumulativeDurationMs desc
```

## Task-start throughput and AOS distribution

```kusto
// Include the Common event adapter.
BatchEvents
| where EventTime between (_startTime .. _endTime)
| where EventName =~ "BatchTaskStart"
| summarize TaskStarts=count() by TimeBucket=bin(EventTime,10m),RoleInstance
| order by TimeBucket asc
```

This is arrival/throughput data, not concurrent-running task count.

## Thread capacity and utilization

```kusto
// Include the Common event adapter.
let ThreadSamples=BatchEvents
    | where EventTime between (_startTime .. _endTime)
    | where EventName =~ "BatchThreadInfo"
    | extend
        CurrentBatchTasks=toint(Info.CurrentBatchTasks),
        TaskQueueCount=toint(Info.TaskQueueCount),
        MaxThreadCount=toint(Info.MaxThreadCount),
        ReservedNumberOfThreads=toint(Info.ReservedNumberOfThreads)
    | extend EffectiveCapacity=MaxThreadCount-ReservedNumberOfThreads
    | extend
        AvailableThreads=EffectiveCapacity-CurrentBatchTasks,
        UtilizationPct=iff(
            EffectiveCapacity > 0
                and CurrentBatchTasks >= 0
                and CurrentBatchTasks <= EffectiveCapacity,
            100.0*todouble(CurrentBatchTasks)/todouble(EffectiveCapacity),
            real(null)),
        InvalidSample=isnull(CurrentBatchTasks)
            or isnull(MaxThreadCount)
            or isnull(ReservedNumberOfThreads)
            or EffectiveCapacity <= 0
            or CurrentBatchTasks < 0
            or CurrentBatchTasks > EffectiveCapacity;
ThreadSamples
| summarize
    Samples=count(),
    FirstSample=min(EventTime),
    LastSample=max(EventTime),
    MaxThreadsMin=min(MaxThreadCount),
    MaxThreadsMax=max(MaxThreadCount),
    ReservedThreadsMin=min(ReservedNumberOfThreads),
    ReservedThreadsMax=max(ReservedNumberOfThreads),
    EffectiveCapacityMin=min(EffectiveCapacity),
    EffectiveCapacityMax=max(EffectiveCapacity),
    MedianCurrentTasks=round(percentile(CurrentBatchTasks,50),1),
    P95CurrentTasks=round(percentile(CurrentBatchTasks,95),1),
    MedianUtilizationPct=round(percentile(UtilizationPct,50),1),
    P95UtilizationPct=round(percentile(UtilizationPct,95),1),
    PeakUtilizationPct=round(max(UtilizationPct),1),
    MinimumAvailableThreads=min(AvailableThreads),
    InvalidSamples=countif(InvalidSample)
    by RoleInstance
| order by PeakUtilizationPct desc
```

Time trend:

```kusto
ThreadSamples
| project
    EventTime,
    RoleInstance,
    CurrentBatchTasks,
    EffectiveCapacity,
    AvailableThreads,
    UtilizationPct,
    InvalidSample
| order by EventTime asc
```

Sample-gap coverage:

```kusto
ThreadSamples
| sort by RoleInstance asc,EventTime asc
| serialize
| extend PreviousRole=prev(RoleInstance),PreviousTime=prev(EventTime)
| extend GapMinutes=iff(
    RoleInstance == PreviousRole,
    todouble((EventTime-PreviousTime)/1m),
    real(null))
| summarize
    Samples=count(),
    FirstSample=min(EventTime),
    LastSample=max(EventTime),
    MaxGapMinutes=round(max(GapMinutes),1)
    by RoleInstance
```

## Priority queue depth

```kusto
// Include the Common event adapter.
let QueueSamples=BatchEvents
    | where EventTime between (_startTime .. _endTime)
    | where EventName =~ "BatchQueuesDetails"
    | project
        EventTime,
        RoleInstance,
        Critical=toint(Info.BatchCriticalSchedulingQueue),
        High=toint(Info.BatchHighSchedulingQueue),
        Normal=toint(Info.BatchNormalSchedulingQueue),
        Low=toint(Info.BatchLowSchedulingQueue),
        ReservedCapacity=toint(Info.BatchReservedCapacitySchedulingQueue),
        ReadyBuffer=toint(Info.ReadyTasksBuffer),
        ReadyBufferWithPriorities=toint(Info.ReadyTasksBufferWithPriorities);
QueueSamples
| project EventTime,RoleInstance,QueueValues=pack(
    "Critical",Critical,
    "High",High,
    "Normal",Normal,
    "Low",Low,
    "ReservedCapacity",ReservedCapacity,
    "ReadyBuffer",ReadyBuffer,
    "ReadyBufferWithPriorities",ReadyBufferWithPriorities)
| mv-expand QueueName=bag_keys(QueueValues)
| extend QueueName=tostring(QueueName),QueueDepth=toint(QueueValues[QueueName])
| summarize
    Samples=count(),
    MissingSamples=countif(isnull(QueueDepth)),
    MedianDepth=round(percentile(QueueDepth,50),1),
    P95Depth=round(percentile(QueueDepth,95),1),
    PeakDepth=max(QueueDepth),
    NonzeroSamples=countif(QueueDepth > 0),
    FirstNonzero=minif(EventTime,QueueDepth > 0),
    LastNonzero=maxif(EventTime,QueueDepth > 0)
    by RoleInstance,QueueName
| order by PeakDepth desc
```

For the chart:

```kusto
QueueSamples
| project
    EventTime,
    RoleInstance,
    Critical,
    High,
    Normal,
    Low,
    ReservedCapacity,
    ReadyBuffer,
    ReadyBufferWithPriorities
| order by EventTime asc
```

To claim sustained backlog, additionally inspect ordered samples and require at
least three consecutive nonzero observations at the observed cadence.

Longest observed nonzero sequence:

```kusto
let QueueSeries=QueueSamples
    | project EventTime,RoleInstance,QueueValues=pack(
        "Critical",Critical,
        "High",High,
        "Normal",Normal,
        "Low",Low,
        "ReservedCapacity",ReservedCapacity,
        "ReadyBuffer",ReadyBuffer,
        "ReadyBufferWithPriorities",ReadyBufferWithPriorities)
    | mv-expand QueueName=bag_keys(QueueValues)
    | extend QueueName=tostring(QueueName),QueueDepth=toint(QueueValues[QueueName])
    | sort by RoleInstance asc,QueueName asc,EventTime asc
    | serialize
    | extend RestartGroup=coalesce(
        RoleInstance != prev(RoleInstance) or QueueName != prev(QueueName),
        true)
    | extend StartsNonzeroRun=iff(
        QueueDepth > 0
            and (RestartGroup or isnull(prev(QueueDepth)) or prev(QueueDepth) <= 0),
        1,
        0)
    | extend RunId=row_cumsum(StartsNonzeroRun,RestartGroup);
QueueSeries
| where QueueDepth > 0
| summarize
    ConsecutiveSamples=count(),
    RunStart=min(EventTime),
    RunEnd=max(EventTime),
    PeakDepth=max(QueueDepth)
    by RoleInstance,QueueName,RunId
| summarize arg_max(ConsecutiveSamples,RunStart,RunEnd,PeakDepth)
    by RoleInstance,QueueName
| order by ConsecutiveSamples desc
```

Queue sample-gap coverage:

```kusto
QueueSamples
| sort by RoleInstance asc,EventTime asc
| serialize
| extend PreviousRole=prev(RoleInstance),PreviousTime=prev(EventTime)
| extend GapMinutes=iff(
    RoleInstance == PreviousRole,
    todouble((EventTime-PreviousTime)/1m),
    real(null))
| summarize
    Samples=count(),
    FirstSample=min(EventTime),
    LastSample=max(EventTime),
    MaxGapMinutes=round(max(GapMinutes),1)
    by RoleInstance
```

## Throttling trend and resource context

```kusto
// Include the Common event adapter.
let Throttles=BatchEvents
    | where EventTime between (_startTime .. _endTime)
    | where EventName =~ "BatchThrottled"
    | extend
        CurrentMachineCpu=todouble(Info.CurrentMachineCpu),
        CurrentMachineMemory=todouble(Info.CurrentMachineMemory),
        CurrentSqlDtu=todouble(Info.CurrentSqlDtu),
        AttributableBatchJobId=iff(
            isnotempty(BatchJobId) and BatchJobId != "0",
            BatchJobId,
            "");
let Starts=BatchEvents
    | where EventTime between (_startTime .. _endTime)
    | where EventName =~ "BatchTaskStart"
    | summarize TaskStarts=count() by TimeBucket=bin(EventTime,10m);
let ThrottleTrend=Throttles
    | summarize
        Throttles=count(),
        AffectedAOS=dcountif(RoleInstance,isnotempty(RoleInstance)),
        AttributableJobs=dcountif(AttributableBatchJobId,isnotempty(AttributableBatchJobId)),
        CpuSamples=countif(isnotnull(CurrentMachineCpu)),
        MemorySamples=countif(isnotnull(CurrentMachineMemory)),
        SqlDtuSamples=countif(isnotnull(CurrentSqlDtu)),
        MedianCpu=round(percentile(CurrentMachineCpu,50),1),
        PeakCpu=round(max(CurrentMachineCpu),1),
        MedianMemory=round(percentile(CurrentMachineMemory,50),1),
        PeakMemory=round(max(CurrentMachineMemory),1),
        MedianSqlDtu=round(percentile(CurrentSqlDtu,50),1),
        PeakSqlDtu=round(max(CurrentSqlDtu),1)
        by TimeBucket=bin(EventTime,10m);
range TimeBucket from bin(_startTime,10m) to bin(_endTime,10m) step 10m
| join kind=leftouter ThrottleTrend on TimeBucket
| join kind=leftouter Starts on TimeBucket
| project
    TimeBucket,
    Throttles=coalesce(Throttles,0),
    TaskStarts=coalesce(TaskStarts,0),
    AffectedAOS=coalesce(AffectedAOS,0),
    AttributableJobs=coalesce(AttributableJobs,0),
    CpuSamples=coalesce(CpuSamples,0),
    MemorySamples=coalesce(MemorySamples,0),
    SqlDtuSamples=coalesce(SqlDtuSamples,0),
    MedianCpu,
    PeakCpu,
    MedianMemory,
    PeakMemory,
    MedianSqlDtu,
    PeakSqlDtu
| order by TimeBucket asc
```

Do not interpret `TaskStarts` as active concurrency. Confirm units and expected
ranges for CPU, memory, and SQL DTU from live values before thresholding.

## Failed task executions

```kusto
// Include the Common event adapter and TaskExecutions.
TaskExecutions
| where PairStatus == "Failed"
| extend ApprovedSignature=case(
    ExceptionType matches regex @"^[A-Za-z0-9_.-]{1,160}$",ExceptionType,
    ExceptionSource matches regex @"^[A-Za-z0-9_.-]{1,160}$",ExceptionSource,
    ClassName matches regex @"^[A-Za-z0-9_.-]{1,160}$",strcat("Class:",ClassName),
    "(redacted/unavailable)")
| project
    FailureTime,
    EnvironmentId,
    BatchJobCaption,
    ClassName,
    BatchJobId,
    BatchTaskId,
    ActivityId,
    RoleInstance,
    Priority,
    ApprovedSignature,
    ObservedDurationMs
| top 100 by FailureTime desc
```

## Supporting exception telemetry

Assemble the exception query separately: **exception parameters + exactly one
exception source adapter + shared exception analysis**. Do not paste the event
parameters into the same submission; that would redeclare variables.

### Exception parameters

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedEnvironmentId = tolower("{{EnvironmentId}}");
let requestedBatchJobId = tolower("{{BatchJobId}}");
```

### Exception source: Application Insights

```kusto
let RawExceptionRows = exceptions
    | where timestamp between (_startTime .. _endTime)
    | project
        ExceptionTime=timestamp,
        ExceptionType=tostring(type),
        MethodName=tostring(column_ifexists("method","")),
        Dimensions=todynamic(customDimensions),
        RoleInstance=tostring(cloud_RoleInstance),
        OperationId=tolower(tostring(column_ifexists("operation_Id",""))),
        SourceTable="exceptions";
```

### Exception source: Log Analytics

```kusto
let RawExceptionRows = AppExceptions
    | where TimeGenerated between (_startTime .. _endTime)
    | project
        ExceptionTime=TimeGenerated,
        ExceptionType=tostring(ExceptionType),
        MethodName=tostring(column_ifexists("Method","")),
        Dimensions=todynamic(Properties),
        RoleInstance=tostring(AppRoleInstance),
        OperationId=tolower(tostring(column_ifexists("OperationId",""))),
        SourceTable="AppExceptions";
```

`AppExceptions.Method` maps to canonical `MethodName`; do not assume the
physical column is named `MethodName`. Apply the same role-instance and
property-bag handling as the event adapters. Missing optional method metadata
does not suppress exception rows. Required columns must pass the schema probe.

### Shared exception analysis

```kusto
let BatchExceptionSignals=RawExceptionRows
    | extend
        EnvironmentId=tolower(coalesce(
            tostring(Dimensions.environmentId),
            tostring(Dimensions.EnvironmentId))),
        ExecutionMode=tostring(Dimensions.ExecutionMode),
        BatchJobId=tolower(coalesce(
            tostring(Dimensions.BatchJobId),
            tostring(Dimensions.batchJobId))),
        ActivityId=tolower(coalesce(
            tostring(Dimensions.activityId),
            tostring(Dimensions.ActivityId),
            OperationId)),
        BatchJobCaption=tostring(Dimensions.BatchJobCaption),
        ClassName=tostring(Dimensions.ClassName)
    | where ExecutionMode =~ "Batch"
    | where isempty(requestedEnvironmentId) or EnvironmentId == requestedEnvironmentId
    | where isempty(requestedBatchJobId) or BatchJobId == requestedBatchJobId
    | extend ApprovedSignature=case(
        ExceptionType matches regex @"^[A-Za-z0-9_.-]{1,160}$",ExceptionType,
        MethodName matches regex @"^[A-Za-z0-9_.:-]{1,160}$",strcat("Method:",MethodName),
        isnotempty(ClassName),strcat("Class:",ClassName),
        "(redacted/unavailable)");
BatchExceptionSignals
| summarize
    ExceptionEvents=count(),
    AffectedActivities=dcountif(ActivityId,isnotempty(ActivityId)),
    AffectedJobs=dcountif(BatchJobId,isnotempty(BatchJobId)),
    FirstSeen=min(ExceptionTime),
    LastSeen=max(ExceptionTime),
    AOSInstances=make_set_if(RoleInstance,isnotempty(RoleInstance),20)
    by ApprovedSignature,SourceTable
| top 25 by ExceptionEvents desc
```

Do not project raw outer messages, call stacks, dimensions, user identifiers,
or session identifiers.

## Equal-window baseline

For a requested current window, define the immediately preceding disjoint
window:

```kusto
let _currentStart = datetime({{StartTimeUtc}});
let _currentEnd = datetime({{EndTimeUtc}});
let _windowLength = _currentEnd-_currentStart;
let _previousStart = _currentStart-_windowLength;
let _previousEnd = _currentStart;
```

Build the same canonical datasets over `_previousStart` through `_currentEnd`
with identical environment and optional filters. Label rows by the start or
sample timestamp:

```kusto
| extend Window=case(
    EventTime >= _currentStart and EventTime <= _currentEnd,"Current",
    EventTime >= _previousStart and EventTime < _previousEnd,"Previous",
    "Outside")
| where Window != "Outside"
```

For task executions, use `StartTime`; for thread, queue, and throttle samples,
use `EventTime`. Compare:

- Started/completed/failed task executions and explicit rates.
- Median and p95 completed duration with at least five completed tasks in each
  window.
- p95 and peak utilization with comparable sample counts and cadence.
- p95 and peak queue depth.
- Throttle events and affected AOS instances.

Do not calculate a percentage change from a zero baseline. Do not compare
windows when telemetry coverage or sample cadence differs materially.
