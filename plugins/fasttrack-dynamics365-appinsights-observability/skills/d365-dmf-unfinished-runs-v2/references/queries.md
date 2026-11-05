# DMF unfinished-run query library

Examples use Application Insights `customEvents`.

## Schema and coverage probe

```kusto
let _queryNow=now();
customEvents
| where timestamp between (_queryNow-1d .. _queryNow)
| where name startswith "DMF"
| extend cd=todynamic(customDimensions)
| extend PropertyKeys=bag_keys(cd)
| summarize Events=count(),SamplePropertyKeys=take_any(PropertyKeys),
    HasExecutionId=countif(isnotempty(tostring(cd.ExecutionId))),
    HasEnvironmentId=countif(isnotempty(coalesce(tostring(cd.environmentId),tostring(cd.EnvironmentId))))
    by name
| order by Events desc
```

## Detection and enrichment

```kusto
let _queryNow = now();
let _lookback = 7d;
let _ingestionLag = 5m;
let _warningSilence = 15m;
let _criticalSilence = 60m;
let _lookbackStart = _queryNow - _lookback;
let _cutoff = _queryNow - _ingestionLag;
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedDirection = "{{Direction}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedLegalEntity = "{{LegalEntity}}";
let ProjectGuidSuffix = @"-[({]?[a-fA-F0-9]{8}[-]?([a-fA-F0-9]{4}[-]?){3}[a-fA-F0-9]{12}[})]?$";
let AllDmf =
    materialize(
        customEvents
        | where timestamp between (_lookbackStart .. _queryNow)
        | where name startswith "DMF"
        | extend cd = todynamic(customDimensions)
        | extend
            ExecutionId = tolower(trim(" ", tostring(cd.ExecutionId))),
            EnvironmentId = coalesce(tostring(cd.environmentId), tostring(cd.EnvironmentId)),
            EntityName = tostring(cd.EntityName),
            DataProjectName = replace_regex(tostring(cd.DataProjectName),ProjectGuidSuffix,""),
            LegalEntity = tostring(cd.LegalEntity),
            BatchJobId = tostring(cd.BatchJobId),
            ActivityId = tolower(coalesce(tostring(cd.activityId), tostring(cd.ActivityId), tostring(operation_Id))),
            Direction = case(
                name contains "Export", "Export",
                name contains "Import", "Import",
                tostring(cd.DMFOperationType) contains "Export", "Export",
                tostring(cd.DMFOperationType) contains "Import", "Import",
                "Unknown")
        | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
        | project timestamp, name, ExecutionId, EnvironmentId, EntityName, DataProjectName,
            LegalEntity, BatchJobId, ActivityId, Direction, cloud_RoleInstance);
let MissingExecutionIdEvents =
    toscalar(AllDmf
        | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
        | summarize countif(isempty(ExecutionId)));
let StartRows =
    AllDmf
    | where name in~ ("DMFExportJobStart","DMFImportJobStart")
    | where timestamp <= _cutoff
    | where isnotempty(ExecutionId)
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId
    | project ExecutionId,timestamp,EntityName,DataProjectName,LegalEntity,BatchJobId,
        ActivityId,Direction,cloud_RoleInstance;
let StartCounts=StartRows | summarize StartEvents=count() by ExecutionId;
let Starts =
    StartRows
    | summarize arg_max(timestamp,EntityName,DataProjectName,LegalEntity,BatchJobId,
        ActivityId,Direction,cloud_RoleInstance) by ExecutionId
    | project ExecutionId,StartTime=timestamp,EntityName,DataProjectName,LegalEntity,
        BatchJobId,ActivityId,Direction,StartRoleInstance=cloud_RoleInstance
    | join kind=inner StartCounts on ExecutionId;
let Ends =
    AllDmf
    | where isnotempty(ExecutionId)
    | where name in~ ("DMFExportJobEnd","DMFImportJobEnd")
    | summarize EndEvents=count(), EndTime=max(timestamp) by ExecutionId;
let EventsAfterStart =
    AllDmf
    | where isnotempty(ExecutionId)
    | join kind=inner (Starts | project ExecutionId,StartTime) on ExecutionId
    | where timestamp >= StartTime and timestamp <= _cutoff;
let LatestEvent =
    EventsAfterStart
    | summarize arg_max(timestamp,name,cloud_RoleInstance) by ExecutionId
    | project ExecutionId,LastEventTime=timestamp,LastEventName=name,
        LastRoleInstance=cloud_RoleInstance;
let EventCounts =
    EventsAfterStart
    | summarize EventCount=count() by ExecutionId;
let ErrorCounts =
    EventsAfterStart
    | summarize ErrorEvents=countif(name =~ "DMFError") by ExecutionId;
let Unfinished =
    Starts
    | join kind=leftouter Ends on ExecutionId
    | where isnull(EndTime) or EndTime < StartTime
    | join kind=leftouter LatestEvent on ExecutionId
    | join kind=leftouter ErrorCounts on ExecutionId
    | join kind=leftouter EventCounts on ExecutionId
    | extend
        ObservationCutoff=_cutoff,
        ElapsedMinutes=round((_cutoff-StartTime)/1m,1),
        SilenceMinutes=max_of(0.0,round((_cutoff-LastEventTime)/1m,1)),
        HasProgress=coalesce(EventCount,0) > StartEvents
    | extend
        State=case(
            StartTime < _lookbackStart + 1h, "Indeterminate",
            StartEvents > 1, "Indeterminate",
            coalesce(ErrorEvents,0) > 0, "ErroredNoEnd",
            SilenceMinutes < _warningSilence/1m, "Active",
            "Silent"),
        Severity=case(
            StartEvents > 1, "warning",
            coalesce(ErrorEvents,0) > 0 and SilenceMinutes >= _warningSilence/1m, "critical",
            SilenceMinutes >= _criticalSilence/1m, "critical",
            SilenceMinutes >= _warningSilence/1m, "warning",
            "info")
    | where isempty(requestedEntityName) or EntityName =~ requestedEntityName
    | where isempty(requestedProjectName) or DataProjectName =~ replace_regex(requestedProjectName,ProjectGuidSuffix,"")
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity
    | where isempty(requestedDirection) or Direction =~ requestedDirection;
```

## Risk-ordered result

```kusto
Unfinished
| extend SeverityRank=case(Severity=="critical",3,Severity=="warning",2,1)
| sort by SeverityRank desc, SilenceMinutes desc
| take 1000
| project DataProjectName, EntityName, ExecutionId, ActivityId, Direction, StartTime,
    LastEventTime, LastEventName, ElapsedMinutes, SilenceMinutes, State, Severity,
    HasProgress,ErrorEvents, BatchJobId,
    RoleInstance=coalesce(LastRoleInstance,StartRoleInstance),
    MissingExecutionIdEvents
```

Run `Unfinished | summarize Runs=count() by State,Severity` before the limited detail query. This preserves the full summary and reveals detail truncation.

## Restart or crash correlation

Use the appropriate AOS/platform telemetry source. Accept a correlation only when:

```text
RestartOrCrashTime > StartTime
and RestartOrCrashTime >= LastEventTime
and RoleInstance matches
```

Temporal overlap without matching role instance is insufficient.

Match `tolower(trim(" ", node))` from restart telemetry to `tolower(trim(" ", LastRoleInstance))`; fall back to the similarly normalized `StartRoleInstance` only when the latest-event node is unavailable.

## Historical-duration context

For each entity/project with enough history, calculate completed-run p95 from explicitly paired start/end events. Compare `ElapsedMinutes` with that baseline but keep `SilenceMinutes` as the primary current-health signal.
