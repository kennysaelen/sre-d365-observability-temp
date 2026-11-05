# DMF throughput and volume query library

Examples use Application Insights `customEvents`.

## Schema probe

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
customEvents
| where timestamp between (_startTime .. _endTime)
| where name in~ ("DMFExportJobEnd","DMFImportJobEnd")
| extend cd=todynamic(customDimensions)
| summarize
    Events=count(),
    HasExecutionId=countif(isnotempty(tostring(cd.ExecutionId))),
    HasEntity=countif(isnotempty(tostring(cd.EntityName))),
    HasProject=countif(isnotempty(tostring(cd.DataProjectName))),
    HasRecords=countif(isnotnull(tolong(cd.TotalNumberOfRecords))),
    HasEventId=countif(isnotempty(tostring(cd.eventId)))
    by name
```

## Deduplicated completed-record dataset

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedDirection = "{{Direction}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedLegalEntity = "{{LegalEntity}}";
let ProjectGuidSuffix = @"-[({]?[a-fA-F0-9]{8}[-]?([a-fA-F0-9]{4}[-]?){3}[a-fA-F0-9]{12}[})]?$";
let RawEndAll =
    customEvents
    | where timestamp between (_startTime .. _endTime)
    | where name in~ ("DMFExportJobEnd","DMFImportJobEnd")
    | extend cd = todynamic(customDimensions)
    | extend
        ExecutionId = tolower(trim(" ", tostring(cd.ExecutionId))),
        EnvironmentId = coalesce(tostring(cd.environmentId), tostring(cd.EnvironmentId)),
        EntityName = tostring(cd.EntityName),
        DataProjectName = replace_regex(tostring(cd.DataProjectName), ProjectGuidSuffix, ""),
        LegalEntity = tostring(cd.LegalEntity),
        Records = tolong(cd.TotalNumberOfRecords),
        Direction = case(name contains "Export", "Export", name contains "Import", "Import", "Unknown"),
        EventId = tostring(cd.eventId)
    | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId;
let MissingExecutionIdRows =
    RawEndAll
    | summarize
        Rows=countif(isempty(ExecutionId)),
        RecordsOnMissingIdRows=sumif(Records,isempty(ExecutionId));
let RawEnd =
    RawEndAll
    | where isnotempty(ExecutionId);
let DmfEndAll =
    RawEnd
    | summarize
        DuplicateEndEvents=count(),
        DistinctRecordCounts=count_distinct(Records),
        arg_max(timestamp,Records,EnvironmentId,DataProjectName,LegalEntity,Direction)
        by ExecutionId, EntityName;
let EligibleExecutions =
    DmfEndAll
    | summarize
        EntityNames=make_set_if(EntityName,isnotempty(EntityName)),
        ProjectNames=make_set_if(DataProjectName,isnotempty(DataProjectName)),
        Directions=make_set_if(Direction,isnotempty(Direction))
        by ExecutionId
    | where isempty(requestedProjectName) or ProjectNames has replace_regex(requestedProjectName,ProjectGuidSuffix,"")
    | where isempty(requestedDirection) or Directions has requestedDirection
    | project ExecutionId;
let DmfEnd =
    DmfEndAll
    | join kind=inner EligibleExecutions on ExecutionId
    | where isempty(requestedEntityName) or EntityName =~ requestedEntityName
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity
    | project CompletionTime=timestamp,Records,EnvironmentId,DataProjectName,LegalEntity,
        Direction,ExecutionId,EntityName,DuplicateEndEvents,DistinctRecordCounts;
```

When `eventId` is populated, deduplicate exact event IDs before the execution/entity aggregation. The latest end row supplies both timestamp and record count. `DuplicateEndEvents > 1` and `DistinctRecordCounts > 1` are coverage evidence.

## Coverage

```kusto
DmfEnd
| summarize
    CompletedExecutionEntities=count(),
    DistinctExecutions=dcount(ExecutionId),
    MissingRecordCounts=countif(isnull(Records)),
    ZeroRecordCounts=countif(Records == 0),
    DuplicateEndGroups=countif(DuplicateEndEvents > 1),
    ConflictingCountGroups=countif(DistinctRecordCounts > 1),
    ExecutionEntityRowsWithoutCounts=countif(isnull(Records)),
    AffectedExecutionsWithoutCounts=dcountif(ExecutionId,isnull(Records))
```

Run `MissingExecutionIdRows` and include its rows and record coverage in the report.

## Volume by entity

```kusto
DmfEnd
| extend Entity=iff(isempty(EntityName), "(missing)", EntityName)
| summarize
    Executions=dcount(ExecutionId),
    ExecutionsWithoutCounts=countif(isnull(Records)),
    TotalRecords=sum(Records)
    by Entity, Direction
| top 25 by TotalRecords desc
```

Use `DataProjectName` instead of `EntityName` for the project view.

Calculate share against the full, untruncated denominator:

```kusto
let AllCompletedRecords=toscalar(DmfEnd | summarize sum(Records));
DmfEnd
| summarize TotalRecords=sum(Records) by EntityName,Direction
| extend SharePct=iff(AllCompletedRecords == 0,real(null),
    round(100.0*todouble(TotalRecords)/todouble(AllCompletedRecords),1))
| top 25 by TotalRecords desc
```

## Execution-size distribution

```kusto
DmfEnd
| where isnotnull(Records)
| summarize
    ExecutionEntities=count(),
    DistinctExecutions=dcount(ExecutionId),
    AvgRecords=round(avg(todouble(Records)),1),
    MedianRecords=round(percentile(Records,50),1),
    P95Records=round(percentile(Records,95),1),
    MaxRecords=max(Records)
    by DataProjectName, Direction
| top 25 by P95Records desc
```

## Largest execution/entity runs

```kusto
DmfEnd
| where isnotnull(Records)
| top 20 by Records desc
| project CompletionTime, ExecutionId, EntityName, DataProjectName, Direction, Records, DuplicateEndEvents
```

This is an execution/entity ranking. Sum by `ExecutionId` first only when the user requests whole-project execution totals.

## Operational records-per-hour trend

```kusto
let ThroughputBins =
    DmfEnd
    | where isnotnull(Records)
    | summarize RecordsCompleted=sum(Records),Executions=dcount(ExecutionId)
    by TimeBucket=bin(CompletionTime,1h),Direction
    | project TimeBucket,Direction,RecordsCompleted,Executions;
range TimeBucket from bin(_startTime,1h) to bin(_endTime,1h) step 1h
| mv-expand Direction=dynamic(["Import","Export","Unknown"]) to typeof(string)
| join kind=leftouter ThroughputBins on TimeBucket,Direction
| extend RecordsCompleted=coalesce(RecordsCompleted,0L),Executions=coalesce(Executions,0L)
| extend BinComplete=TimeBucket >= _startTime and TimeBucket+1h <= _endTime
| extend RecordsPerHour=iff(BinComplete,todouble(RecordsCompleted),real(null))
| order by TimeBucket asc
```

For non-hour bins, divide by the bin duration in hours.

## Current versus previous volume

Fetch from `_previousStart = _startTime - (_endTime - _startTime)` through `_endTime`. For this query, change the reusable `RawEndAll` time predicate to that full range before defining `RawEnd`; then assign `Window` before deduplication and retain the same execution filters.

```kusto
let _materialityFloor=tolong({{MaterialityFloor}});
let _previousStart=_startTime-(_endTime-_startTime);
let ComparisonEndBase =
    RawEnd
    | where timestamp between (_previousStart .. _endTime)
    | extend Window=iff(timestamp >= _startTime,"Current","Previous")
    | summarize
        DuplicateEndEvents=count(),
        arg_max(timestamp,Records,DataProjectName,LegalEntity,Direction)
        by Window,ExecutionId,EntityName
    | project CompletionTime=timestamp,Window,ExecutionId,EntityName,Records,
        DataProjectName,LegalEntity,Direction;
let ComparisonEligible =
    ComparisonEndBase
    | summarize
        EntityNames=make_set_if(EntityName,isnotempty(EntityName)),
        ProjectNames=make_set_if(DataProjectName,isnotempty(DataProjectName)),
        Directions=make_set_if(Direction,isnotempty(Direction))
        by Window,ExecutionId
    | where isempty(requestedProjectName) or ProjectNames has replace_regex(requestedProjectName,ProjectGuidSuffix,"")
    | where isempty(requestedDirection) or Directions has requestedDirection
    | project Window,ExecutionId;
let ComparisonEnd=ComparisonEndBase
    | join kind=inner ComparisonEligible on Window,ExecutionId
    | where isempty(requestedEntityName) or EntityName =~ requestedEntityName
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity;
ComparisonEnd
| extend Workload=iff(isempty(EntityName),"(missing)",EntityName)
| summarize
    Current=sumif(Records,Window=="Current"),
    Previous=sumif(Records,Window=="Previous"),
    CurrentExecutions=countif(Window=="Current"),
    PreviousExecutions=countif(Window=="Previous"),
    CurrentMissingCounts=countif(Window=="Current" and isnull(Records)),
    PreviousMissingCounts=countif(Window=="Previous" and isnull(Records))
    by Workload
| extend Current=coalesce(Current,0L),Previous=coalesce(Previous,0L)
| extend Classification=case(
    CurrentMissingCounts > 0 or PreviousMissingCounts > 0, "UnknownCoverage",
    PreviousExecutions == 0 and CurrentExecutions > 0, "New",
    CurrentExecutions == 0 and PreviousExecutions > 0, "Disappeared",
    "Comparable")
| extend PctChange=iff(
    Classification != "Comparable" or Previous == 0,
    real(null),
    round(100.0 * (todouble(Current)-todouble(Previous))/todouble(Previous),1))
| extend IsMaterial=max_of(Current,Previous) >= _materialityFloor
```

Use a full workload list so disappeared workloads are retained. Positive percentage means volume growth only; it is not a quality judgment.

## Processing efficiency

Aggregate the execution/entity rows to execution grain before joining complete duration pairs on `ExecutionId`.

```kusto
let EfficiencyEligibleExecutions=DmfEnd | distinct ExecutionId;
let RecordsByExecution =
    DmfEndAll
    | join kind=inner EfficiencyEligibleExecutions on ExecutionId
    | summarize Records=sum(Records),EntityRows=count(),KnownEntityRows=countif(isnotnull(Records))
        by ExecutionId
    | where EntityRows == KnownEntityRows;
let StartByExecution =
    customEvents
    | where timestamp between (_startTime-1d .. _endTime)
    | where name in~ ("DMFImportJobStart","DMFExportJobStart")
    | extend cd=todynamic(customDimensions)
    | extend
        ExecutionId=tolower(trim(" ",tostring(cd.ExecutionId))),
        EnvironmentId=coalesce(tostring(cd.environmentId),tostring(cd.EnvironmentId))
    | where isnotempty(ExecutionId)
    | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId
    | summarize StartTime=min(timestamp),StartEvents=count() by ExecutionId;
let CompleteDuration =
    DmfEndAll
    | join kind=inner EfficiencyEligibleExecutions on ExecutionId
    | summarize EndTime=max(timestamp),DuplicateEndGroups=countif(DuplicateEndEvents>1)
        by ExecutionId
    | join kind=inner StartByExecution on ExecutionId
    | where StartEvents == 1 and DuplicateEndGroups == 0 and EndTime >= StartTime
    | extend DurationSec=todouble((EndTime-StartTime)/1s);
RecordsByExecution
| join kind=inner (
    CompleteDuration
    | project ExecutionId,DurationSec
) on ExecutionId
| where DurationSec > 0.0 and Records > 0
| extend RecordsPerSec=round(todouble(Records)/DurationSec,2)
```

Do not calculate efficiency for incomplete or truncated pairs.
