# DMF dashboard query library

Examples use Application Insights `customEvents`. Adapt after probing the actual schema.

## Canonical datasets

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let _directionLookbackStart = _startTime - 1d;
let _queryNow = now();
let _ingestionLag = 5m;
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedActivityId = tolower("{{ActivityId}}");
let requestedLegalEntity = "{{LegalEntity}}";
let requestedDirection = "{{Direction}}";
let ProjectGuidSuffix = @"-[({]?[a-fA-F0-9]{8}[-]?([a-fA-F0-9]{4}[-]?){3}[a-fA-F0-9]{12}[})]?$";
let RawDmfEvents =
    customEvents
    | where timestamp between (_directionLookbackStart .. _endTime)
    | where name startswith "DMF"
    | extend cd=todynamic(customDimensions)
    | extend
        ExecutionId=tolower(trim(" ",tostring(cd.ExecutionId))),
        EnvironmentId=coalesce(tostring(cd.environmentId),tostring(cd.EnvironmentId)),
        ActivityId=tolower(coalesce(tostring(cd.activityId),tostring(cd.ActivityId),tostring(operation_Id))),
        DataProjectName=replace_regex(tostring(cd.DataProjectName),ProjectGuidSuffix,""),
        EntityName=tostring(cd.EntityName),
        LegalEntity=tostring(cd.LegalEntity),
        ExecutionMode=tostring(cd.ExecutionMode),
        DMFOperationType=tostring(cd.DMFOperationType),
        BatchJobId=tostring(cd.BatchJobId),
        TotalNumberOfRecords=tolong(cd.TotalNumberOfRecords),
        StagingErrorCount=tolong(cd.StagingErrorCount),
        TargetErrorCount=tolong(cd.TargetErrorCount),
        ErrorType=tostring(cd.ErrorType),
        ExceptionType=tostring(cd.ExceptionType)
    | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId;
let DirectionByExecution =
    RawDmfEvents
    | where name in~ ("DMFExportJobStart","DMFImportJobStart")
    | where isnotempty(ExecutionId)
    | extend DirectionCandidate=case(
        DMFOperationType contains "Export","Export",
        DMFOperationType contains "Import","Import",
        name contains "Export","Export",
        name contains "Import","Import",
        "Unknown")
    | where DirectionCandidate != "Unknown"
    | summarize arg_max(timestamp,DirectionCandidate) by ExecutionId
    | project ExecutionId,Direction=DirectionCandidate;
let allDMFEvents =
    RawDmfEvents
    | where timestamp between (_startTime .. _endTime)
    | join kind=leftouter DirectionByExecution on ExecutionId
    | extend Direction=case(
        isnotempty(Direction),Direction,
        DMFOperationType contains "Export","Export",
        DMFOperationType contains "Import","Import",
        name contains "Export","Export",
        name contains "Import","Import",
        "Unknown")
    | project timestamp,name,cloud_RoleInstance,BatchJobId,
        DataProjectName,EntityName,ExecutionId,ExecutionMode,LegalEntity,
        TotalNumberOfRecords,StagingErrorCount,TargetErrorCount,ActivityId,
        EnvironmentId,Direction,ErrorType,ExceptionType;
let ExecutionMetadata =
    allDMFEvents
    | where isnotempty(ExecutionId)
    | summarize
        EntityNames=make_set_if(EntityName,isnotempty(EntityName)),
        ProjectNames=make_set_if(DataProjectName,isnotempty(DataProjectName)),
        LegalEntities=make_set_if(LegalEntity,isnotempty(LegalEntity)),
        ActivityIds=make_set_if(ActivityId,isnotempty(ActivityId)),
        DataProjectName=take_anyif(DataProjectName,isnotempty(DataProjectName)),
        Direction=take_anyif(Direction,Direction != "Unknown"),
        ActivityId=take_anyif(ActivityId,isnotempty(ActivityId))
        by ExecutionId
    | where isempty(requestedEntityName) or EntityNames has requestedEntityName
    | where isempty(requestedProjectName) or ProjectNames has replace_regex(requestedProjectName,ProjectGuidSuffix,"")
    | where isempty(requestedLegalEntity) or LegalEntities has requestedLegalEntity
    | where isempty(requestedActivityId) or ActivityIds has requestedActivityId
    | where isempty(requestedDirection) or Direction =~ requestedDirection;
let filteredDMFEvents =
    allDMFEvents
    | join kind=inner (ExecutionMetadata | project ExecutionId) on ExecutionId;
let EndByExecutionEntity =
    allDMFEvents
    | where name in~ ("DMFImportJobEnd","DMFExportJobEnd")
    | where isnotempty(ExecutionId)
    | summarize
        EndEvents=count(),
        DistinctRecordCounts=count_distinct(TotalNumberOfRecords),
        arg_max(timestamp,TotalNumberOfRecords,LegalEntity)
        by ExecutionId,EntityName
    | project ExecutionId,EntityName,EndTime=timestamp,Records=TotalNumberOfRecords,
        EndLegalEntity=LegalEntity,EndEvents,DistinctRecordCounts;
let CompletedVolume =
    EndByExecutionEntity
    | join kind=inner ExecutionMetadata on ExecutionId
    | where isempty(requestedEntityName) or EntityName =~ requestedEntityName
    | where isempty(requestedLegalEntity) or EndLegalEntity =~ requestedLegalEntity;
let CompletedExecutions =
    CompletedVolume
    | summarize
        EndTime=max(EndTime),
        RawRecords=sum(Records),
        KnownRecordEntities=countif(isnotnull(Records)),
        Entities=count_distinct(EntityName),
        EntityNames=make_set_if(EntityName,isnotempty(EntityName)),
        EndEvents=sum(EndEvents),
        DuplicateEndGroups=countif(EndEvents > 1),
        ConflictingCountGroups=countif(DistinctRecordCounts > 1),
        DataProjectName=take_any(DataProjectName),
        Direction=take_any(Direction),
        ActivityId=take_any(ActivityId)
        by ExecutionId
    | extend Records=iff(KnownRecordEntities == Entities,RawRecords,long(null))
    | project-away RawRecords
    | extend EntityLabel=case(
        array_length(EntityNames)==1,tostring(EntityNames[0]),
        array_length(EntityNames)>1,"(multi-entity)",
        "(missing)");
let StartByExecution =
    RawDmfEvents
    | where name in~ ("DMFImportJobStart","DMFExportJobStart")
    | where isnotempty(ExecutionId)
    | summarize StartTime=min(timestamp),StartEvents=count() by ExecutionId;
let CompletedPairs =
    CompletedExecutions
    | join kind=leftouter StartByExecution on ExecutionId
    | extend IsComplete=isnotnull(StartTime) and StartEvents == 1
        and DuplicateEndGroups == 0 and EndTime >= StartTime
    | extend DurationSec=iff(IsComplete,todouble((EndTime-StartTime)/1s),real(null));
```

Every query must include the required `let` definitions. `allDMFEvents` is not a persistent table.

## KPIs

```kusto
let CompletedExecutionCount=toscalar(CompletedExecutions | count);
let CompletedRecords=toscalar(CompletedExecutions | summarize sum(Records));
let PartialRecordExecutions=toscalar(CompletedExecutions | summarize countif(KnownRecordEntities < Entities));
let ErrorEvents=toscalar(filteredDMFEvents | where name =~ "DMFError" | summarize count());
let ErrorExecutions=toscalar(filteredDMFEvents | where name =~ "DMFError" | summarize count_distinct(ExecutionId));
print CompletedExecutions=CompletedExecutionCount,CompletedRecords,PartialRecordExecutions,
    ErrorEvents,ErrorExecutions
```

When `PartialRecordExecutions > 0`, label `CompletedRecords` as a known-record lower bound rather than an exact total.

## Direction mix

```kusto
CompletedExecutions
| summarize Executions=count() by Direction
```

## Executions and volume by workload

```kusto
CompletedVolume
| summarize Executions=count_distinct(ExecutionId),Records=sum(Records) by EntityName,Direction
| top 25 by Records desc
```

Use `DataProjectName` for the project view.

## Cumulative duration

```kusto
CompletedPairs
| where IsComplete
| summarize Executions=count(),CumulativeDurationSec=round(sum(DurationSec),1),
    MedianDurationSec=round(percentile(DurationSec,50),1),
    P95DurationSec=round(percentile(DurationSec,95),1)
    by EntityLabel,Direction
| top 25 by CumulativeDurationSec desc
```

## Error trend

```kusto
let ErrorTrend=filteredDMFEvents
    | where name =~ "DMFError"
    | summarize ErrorEvents=count(),AffectedExecutions=count_distinctif(ExecutionId,isnotempty(ExecutionId))
        by TimeBucket=bin(timestamp,1h);
range TimeBucket from bin(_startTime,1h) to bin(_endTime,1h) step 1h
| join kind=leftouter ErrorTrend on TimeBucket
| extend IsCompleteBin=TimeBucket >= _startTime and TimeBucket+1h <= _endTime
| project TimeBucket,IsCompleteBin,ErrorEvents=coalesce(ErrorEvents,0),AffectedExecutions=coalesce(AffectedExecutions,0)
| order by TimeBucket asc
```

Use only complete bins for anomaly baselines. Use an absolute floor plus a multiple of the zero-filled median. Treat the result as a triage heuristic.

## Unfinished-run summary

This dashboard summary is self-contained; use `d365-dmf-unfinished-runs-v2` for detailed triage.

```kusto
let _cutoff=min_of(_endTime,_queryNow-_ingestionLag);
let StartRows=RawDmfEvents
    | where name in~ ("DMFImportJobStart","DMFExportJobStart")
    | where timestamp between (_startTime .. _cutoff)
    | where isnotempty(ExecutionId)
    | join kind=inner (ExecutionMetadata | project ExecutionId) on ExecutionId;
let StartCounts=StartRows | summarize StartEvents=count() by ExecutionId;
let Started=StartRows
    | summarize StartTime=max(timestamp) by ExecutionId
    | join kind=inner StartCounts on ExecutionId;
let Ended=RawDmfEvents
    | where name in~ ("DMFImportJobEnd","DMFExportJobEnd")
    | where timestamp <= _cutoff and isnotempty(ExecutionId)
    | summarize EndTime=max(timestamp) by ExecutionId;
let EventsAfterStart=RawDmfEvents
    | where timestamp <= _cutoff and isnotempty(ExecutionId)
    | join kind=inner (Started | project ExecutionId,StartTime) on ExecutionId
    | where timestamp >= StartTime;
let Latest=EventsAfterStart
    | summarize arg_max(timestamp,name) by ExecutionId;
let ErrorCounts=EventsAfterStart
    | summarize ErrorEvents=countif(name =~ "DMFError") by ExecutionId;
Started
| join kind=leftouter Ended on ExecutionId
| where isnull(EndTime) or EndTime < StartTime
| join kind=leftouter Latest on ExecutionId
| join kind=leftouter ErrorCounts on ExecutionId
| extend SilenceMinutes=max_of(0.0,round((_cutoff-timestamp)/1m,1))
| extend State=case(
    StartTime < _startTime+1h,"Indeterminate",
    StartEvents > 1,"Indeterminate",
    coalesce(ErrorEvents,0)>0,"ErroredNoEnd",
    SilenceMinutes < 15.0,"Active",
    "Silent")
| extend Severity=case(
    StartTime < _startTime+1h or StartEvents > 1,"warning",
    coalesce(ErrorEvents,0)>0 and SilenceMinutes >= 15.0,"critical",
    SilenceMinutes >= 60.0,"critical",
    SilenceMinutes >= 15.0,"warning",
    "info")
| summarize Executions=count() by State,Severity
```

## Data-quality coverage

```kusto
let TotalDmfEvents=toscalar(allDMFEvents | count);
let MissingExecutionId=toscalar(allDMFEvents | summarize countif(isempty(ExecutionId)));
let CompletedEntityRows=toscalar(CompletedVolume | count);
let CompletedRowsMissingCounts=toscalar(CompletedVolume | summarize countif(isnull(Records)));
let DuplicateEndGroups=toscalar(CompletedVolume | summarize countif(EndEvents > 1));
let ConflictingCountGroups=toscalar(CompletedVolume | summarize countif(DistinctRecordCounts > 1));
let CompletedWithoutStart=toscalar(CompletedPairs | summarize countif(isnull(StartTime)));
let CompletedWithRepeatedStarts=toscalar(CompletedPairs | summarize countif(StartEvents > 1));
let CompletedWithReversedPair=toscalar(CompletedPairs | summarize countif(isnotnull(StartTime) and EndTime < StartTime));
print TotalDmfEvents,MissingExecutionId,CompletedEntityRows,CompletedRowsMissingCounts,
    DuplicateEndGroups,ConflictingCountGroups,CompletedWithoutStart,
    CompletedWithRepeatedStarts,CompletedWithReversedPair
```

## Redacted error signatures

```kusto
filteredDMFEvents
| where name =~ "DMFError"
| extend Signature=case(
    ExceptionType matches regex @"^[A-Za-z0-9_.-]{1,120}$",ExceptionType,
    ErrorType matches regex @"^[A-Za-z0-9_.-]{1,120}$",ErrorType,
    "(redacted/unavailable)")
| summarize ErrorEvents=count(),AffectedExecutions=count_distinctif(ExecutionId,isnotempty(ExecutionId))
    by Signature=iff(isempty(Signature),"(unavailable)",Signature)
| top 10 by ErrorEvents desc
```

## Slowest complete executions

```kusto
CompletedPairs
| where IsComplete
| top 10 by DurationSec desc
| project EntityLabel,DataProjectName,ExecutionId,Direction,StartTime,EndTime,
    DurationSec=round(DurationSec,1),Records,ActivityId
```

## Unknown event names

```kusto
allDMFEvents
| where name !in~ ("DMFImportJobStart","DMFImportJobEnd","DMFExportJobStart","DMFExportJobEnd","DMFError")
| summarize Events=count() by name
| order by Events desc
```

This discovery is scoped to the environment and current window but intentionally runs before execution-based entity/project filtering so unknown events and missing identifiers remain visible.
