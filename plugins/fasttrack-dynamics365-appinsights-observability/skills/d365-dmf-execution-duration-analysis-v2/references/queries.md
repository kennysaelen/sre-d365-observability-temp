# DMF duration query library

Examples use Application Insights `customEvents`. Adapt only after probing the actual schema.

## Schema probe

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
customEvents
| where timestamp between (_startTime .. _endTime)
| where name in~ ("DMFImportJobStart","DMFImportJobEnd","DMFExportJobStart","DMFExportJobEnd")
| extend cd = todynamic(customDimensions)
| summarize
    Events=count(), FirstSeen=min(timestamp), LastSeen=max(timestamp),
    HasExecutionId=countif(isnotempty(tostring(cd.ExecutionId))),
    HasEntity=countif(isnotempty(tostring(cd.EntityName))),
    HasProject=countif(isnotempty(tostring(cd.DataProjectName))),
    HasRecords=countif(isnotnull(tolong(cd.TotalNumberOfRecords)))
    by name
```

## Per-execution dataset

Replace every template token. Empty optional filters mean no filter.

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let _pairingBuffer = 1d;
let _fetchStart = _startTime - _pairingBuffer;
let _fetchEnd = min_of(now(), _endTime + _pairingBuffer);
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedDirection = "{{Direction}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedLegalEntity = "{{LegalEntity}}";
let requestedActivityId = tolower("{{ActivityId}}");
let ProjectGuidSuffix = @"-[({]?[a-fA-F0-9]{8}[-]?([a-fA-F0-9]{4}[-]?){3}[a-fA-F0-9]{12}[})]?$";
let AllEvents =
    customEvents
    | where timestamp between (_fetchStart .. _fetchEnd)
    | where name in~ ("DMFImportJobStart","DMFImportJobEnd","DMFExportJobStart","DMFExportJobEnd")
    | extend cd = todynamic(customDimensions)
    | extend
        ExecutionId = tolower(trim(" ", tostring(cd.ExecutionId))),
        EnvironmentId = coalesce(tostring(cd.environmentId), tostring(cd.EnvironmentId)),
        EntityName = tostring(cd.EntityName),
        DataProjectName = replace_regex(tostring(cd.DataProjectName),ProjectGuidSuffix,""),
        LegalEntity = tostring(cd.LegalEntity),
        TotalNumberOfRecords = tolong(cd.TotalNumberOfRecords),
        ActivityId = tolower(coalesce(tostring(cd.activityId), tostring(cd.ActivityId), tostring(operation_Id))),
        Direction = case(name contains "Export", "Export", name contains "Import", "Import", "Unknown")
    | where isnotempty(ExecutionId)
    | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId;
let ActivityEligibleExecutions =
    AllEvents
    | where isempty(requestedActivityId) or ActivityId == requestedActivityId
    | distinct ExecutionId;
let Events =
    AllEvents
    | join kind=inner ActivityEligibleExecutions on ExecutionId;
let EndByExecutionEntity =
    Events
    | where name endswith "JobEnd"
    | summarize
        EndEvents=count(),
        arg_max(timestamp,TotalNumberOfRecords,EntityName,DataProjectName,LegalEntity)
        by ExecutionId,EntityKey=iff(isempty(EntityName),"(missing)",EntityName)
    | project ExecutionId,EntityKey,EndTime=timestamp,Records=TotalNumberOfRecords,
        EndEntityName=EntityName,EndProjectName=DataProjectName,EndLegalEntity=LegalEntity,
        EndEvents;
let EndByExecution =
    EndByExecutionEntity
    | summarize
        EndTime=max(EndTime),
        RawRecords=sum(Records),
        KnownRecordEntities=countif(isnotnull(Records)),
        EntityNames=make_set_if(EndEntityName,isnotempty(EndEntityName)),
        ProjectNamesAtEnd=make_set_if(EndProjectName,isnotempty(EndProjectName)),
        LegalEntitiesAtEnd=make_set_if(EndLegalEntity,isnotempty(EndLegalEntity)),
        EndEntities=count(),
        DuplicateEndGroups=countif(EndEvents > 1)
        by ExecutionId
    | extend Records=iff(KnownRecordEntities == EndEntities,RawRecords,long(null))
    | project-away RawRecords;
let PerExecution =
    Events
    | summarize
        StartTime = minif(timestamp, name endswith "JobStart"),
        StartEvents = countif(name endswith "JobStart"),
        EnvironmentId = take_any(EnvironmentId),
        EntityNamesAny = make_set_if(EntityName,isnotempty(EntityName)),
        ProjectNamesAny = make_set_if(DataProjectName,isnotempty(DataProjectName)),
        LegalEntitiesAny = make_set_if(LegalEntity,isnotempty(LegalEntity)),
        ActivityId = take_anyif(ActivityId, isnotempty(ActivityId)),
        Direction = take_anyif(Direction,Direction != "Unknown")
        by ExecutionId
    | join kind=leftouter EndByExecution on ExecutionId
    | extend
        EntityNames=iff(array_length(EntityNames)>0,EntityNames,EntityNamesAny),
        ProjectNames=iff(array_length(ProjectNamesAtEnd)>0,ProjectNamesAtEnd,ProjectNamesAny),
        LegalEntities=iff(array_length(LegalEntitiesAtEnd)>0,LegalEntitiesAtEnd,LegalEntitiesAny)
    | extend
        EntityName=case(array_length(EntityNames)==1,tostring(EntityNames[0]),array_length(EntityNames)>1,"(multi-entity)",""),
        DataProjectName=case(array_length(ProjectNames)==1,tostring(ProjectNames[0]),array_length(ProjectNames)>1,"(multiple projects)",""),
        LegalEntity=case(array_length(LegalEntities)==1,tostring(LegalEntities[0]),array_length(LegalEntities)>1,"(multiple legal entities)","")
    | extend PairStatus = case(
        isnull(StartTime), "TruncatedLeft",
        StartEvents > 1 or DuplicateEndGroups > 0, "DuplicateOrRetried",
        isnull(EndTime), "IncompleteOrRunning",
        EndTime < StartTime, "ReversedOrReusedId",
        "Complete")
    | extend Duration = iff(PairStatus == "Complete", EndTime - StartTime, timespan(null))
    | extend DurationSec = todouble(Duration / 1s)
    | extend
        MsPerRecord = iff(PairStatus == "Complete" and Records > 0 and DurationSec >= 5.0, round(Duration / 1ms / todouble(Records), 2), real(null)),
        RecordsPerSec = iff(PairStatus == "Complete" and Records > 0 and DurationSec >= 5.0, round(todouble(Records) / DurationSec, 2), real(null))
    | where
        (PairStatus == "TruncatedLeft" and EndTime between (_startTime .. _endTime))
        or (PairStatus != "TruncatedLeft" and StartTime between (_startTime .. _endTime))
    | where isempty(requestedEntityName) or EntityNames has requestedEntityName
    | where isempty(requestedProjectName) or ProjectNames has replace_regex(requestedProjectName,ProjectGuidSuffix,"")
    | where isempty(requestedLegalEntity) or LegalEntities has requestedLegalEntity
    | where isempty(requestedDirection) or Direction =~ requestedDirection;
```

## Slowest completed executions

```kusto
PerExecution
| where PairStatus == "Complete"
| top 100 by DurationSec desc
| project EnvironmentId,EntityName, DataProjectName, ExecutionId, Direction, StartTime, EndTime,
    DurationSec=round(DurationSec,1), Records, MsPerRecord, RecordsPerSec, ActivityId
```

## Pairing coverage

```kusto
PerExecution
| summarize Executions=count() by PairStatus
| order by Executions desc
```

## Cumulative duration by entity or project

```kusto
PerExecution
| where PairStatus == "Complete"
| extend Workload = iff(isempty(EntityName), "(missing)", EntityName)
| summarize
    Executions=count(),
    CumulativeDurationSec=round(sum(DurationSec),1),
    MedianDurationSec=round(percentile(DurationSec,50),1),
    P95DurationSec=round(percentile(DurationSec,95),1),
    MedianMsPerRecord=round(percentile(MsPerRecord,50),2)
    by Workload, Direction
| top 25 by CumulativeDurationSec desc
```

Use `DataProjectName` instead of `EntityName` for the project view.

## Equal-window duration regression

Set `_currentStart` and `_currentEnd` to the requested window. Build `PerExecution` over `_previousStart` through `_currentEnd`, using the same pairing buffer and filters, then run:

```kusto
let _currentStart = datetime({{CurrentStartUtc}});
let _currentEnd = datetime({{CurrentEndUtc}});
let _windowLength = _currentEnd - _currentStart;
let _previousStart = _currentStart - _windowLength;
PerExecution
| where PairStatus == "Complete"
| where StartTime between (_previousStart .. _currentEnd)
| extend Workload=case(
    isnotempty(EntityName),EntityName,
    isnotempty(DataProjectName),DataProjectName,
    "(missing)")
| summarize
    CurrentN=countif(StartTime >= _currentStart),
    PreviousN=countif(StartTime < _currentStart),
    CurrentMedianSec=percentileif(DurationSec,50,StartTime >= _currentStart),
    PreviousMedianSec=percentileif(DurationSec,50,StartTime < _currentStart),
    CurrentP95Sec=percentileif(DurationSec,95,StartTime >= _currentStart),
    PreviousP95Sec=percentileif(DurationSec,95,StartTime < _currentStart)
    by Workload,Direction
| where CurrentN >= 5 and PreviousN >= 5
| extend ImprovementPct=iff(
    isnull(PreviousMedianSec) or PreviousMedianSec == 0.0,
    real(null),
    round(100.0*(PreviousMedianSec-CurrentMedianSec)/PreviousMedianSec,1))
| order by ImprovementPct asc
```

Positive improvement means faster; negative means regression. Do not compare overlapping 7-day and 14-day windows.
