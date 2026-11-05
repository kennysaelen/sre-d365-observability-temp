# DMF error RCA query library

Adapt table and column names to the selected Azure SRE query surface. These examples use Application Insights-compatible `customEvents`.

## Template rules

Before submitting a query:

1. Replace every `{{...}}` value.
2. Emit an empty string for each unused optional filter.
3. Verify that no `{{` or `}}` remains.
4. Keep the time range bounded.

Example bindings:

```kusto
let _startTime = datetime(2026-09-13T00:00:00Z);
let _endTime = datetime(2026-09-14T00:00:00Z);
let requestedEnvironmentId = "";
let requestedEntityName = "";
let requestedProjectName = "";
let requestedExecutionId = "";
let requestedActivityId = "";
let requestedLegalEntity = "";
```

## Schema probe

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
customEvents
| where timestamp between (_startTime .. _endTime)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| extend PropertyKeys = bag_keys(cd)
| summarize
    EventCount = count(),
    FirstSeen = min(timestamp),
    LastSeen = max(timestamp),
    SamplePropertyKeys = take_any(PropertyKeys),
    HasExecutionId = countif(isnotempty(tostring(cd.ExecutionId))),
    HasLowerActivityId = countif(isnotempty(tostring(cd.activityId))),
    HasUpperActivityId = countif(isnotempty(tostring(cd.ActivityId))),
    HasOperationId = countif(isnotempty(operation_Id))
    by name
| order by EventCount desc
```

## Error-volume precheck

Run this before materializing the dataset. If the count is excessive for the selected tool, narrow the window or add entity, project, execution, or environment filters.

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedActivityId = tolower("{{ActivityId}}");
let requestedLegalEntity = "{{LegalEntity}}";
customEvents
| where timestamp between (_startTime .. _endTime)
| where name == "DMFError"
| extend cd = todynamic(customDimensions)
| extend
    EnvironmentId = coalesce(tostring(cd.environmentId), tostring(cd.EnvironmentId)),
    EntityName = tostring(cd.EntityName),
    DataProjectName = tostring(cd.DataProjectName),
    ExecutionId = tostring(cd.ExecutionId),
    ActivityId = tolower(case(
        isnotempty(tostring(cd.activityId)), tostring(cd.activityId),
        isnotempty(tostring(cd.ActivityId)), tostring(cd.ActivityId),
        tostring(operation_Id))),
    LegalEntity = tostring(cd.LegalEntity)
| where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
| where isempty(requestedEntityName) or EntityName =~ requestedEntityName
| where isempty(requestedProjectName) or DataProjectName =~ requestedProjectName
| where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId
| where isempty(requestedActivityId) or ActivityId == requestedActivityId
| where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity
| summarize ErrorEvents = count()
```

## Reusable filtered dataset

Use this block at the start of each analysis query:

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedEnvironmentId = "{{EnvironmentId}}";
let requestedEntityName = "{{EntityName}}";
let requestedProjectName = "{{DataProjectName}}";
let requestedExecutionId = "{{ExecutionId}}";
let requestedActivityId = tolower("{{ActivityId}}");
let requestedLegalEntity = "{{LegalEntity}}";
let DmfErrors = materialize(
    customEvents
    | where timestamp between (_startTime .. _endTime)
    | where name == "DMFError"
    | extend cd = todynamic(customDimensions)
    | extend
        LegalEntity = tostring(cd.LegalEntity),
        ErrorMessage = tostring(cd.ErrorMessage),
        ErrorCategory = tostring(cd.ErrorCategory),
        ExecutionId = tostring(cd.ExecutionId),
        EntityName = tostring(cd.EntityName),
        DataProjectName = tostring(cd.DataProjectName),
        ExecutionMode = tostring(cd.ExecutionMode),
        StagingRecId = tostring(cd.StagingRecId),
        ActivityId = tolower(case(
            isnotempty(tostring(cd.activityId)), tostring(cd.activityId),
            isnotempty(tostring(cd.ActivityId)), tostring(cd.ActivityId),
            tostring(operation_Id))),
        ActivityIdSource = case(
            isnotempty(tostring(cd.activityId)), "customDimensions.activityId",
            isnotempty(tostring(cd.ActivityId)), "customDimensions.ActivityId",
            isnotempty(tostring(operation_Id)), "operation_Id",
            "unavailable"),
        BatchJobId = tostring(cd.BatchJobId),
        EnvironmentId = coalesce(
            tostring(cd.environmentId),
            tostring(cd.EnvironmentId))
    | extend RedactedErrorMessage = replace_regex(
        ErrorMessage,
        @"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}",
        "<email>")
    | extend RedactedErrorMessage = replace_regex(
        RedactedErrorMessage,
        @"[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}",
        "<guid>")
    | extend RedactedErrorMessage = replace_regex(
        RedactedErrorMessage,
        @"(?i)Bearer\s+[A-Za-z0-9._~+/=-]+",
        "Bearer <redacted>")
    | extend RedactedErrorMessage = replace_regex(
        RedactedErrorMessage,
        @"(?i)(token|secret|password|api[_-]?key)\s*[:=]\s*[^\s,;]+",
        @"\1=<redacted>")
    | where isempty(requestedEnvironmentId) or EnvironmentId =~ requestedEnvironmentId
    | where isempty(requestedEntityName) or EntityName =~ requestedEntityName
    | where isempty(requestedProjectName) or DataProjectName =~ requestedProjectName
    | where isempty(requestedExecutionId) or ExecutionId =~ requestedExecutionId
    | where isempty(requestedActivityId) or ActivityId == requestedActivityId
    | where isempty(requestedLegalEntity) or LegalEntity =~ requestedLegalEntity
    | project
        timestamp,
        name,
        LegalEntity,
        ErrorMessage,
        RedactedErrorMessage,
        ErrorCategory,
        ExecutionId,
        EntityName,
        DataProjectName,
        ExecutionMode,
        StagingRecId,
        ActivityId,
        ActivityIdSource,
        cloud_RoleInstance,
        BatchJobId,
        EnvironmentId
);
```

The telemetry source plus bounded time range are mandatory scope. Optional empty filters are safe because each is independently guarded.

## Overall impact

Append to the reusable dataset:

```kusto
DmfErrors
| summarize
    ErrorEvents = count(),
    Executions = dcountif(ExecutionId, isnotempty(ExecutionId)),
    Entities = dcountif(EntityName, isnotempty(EntityName)),
    Projects = dcountif(DataProjectName, isnotempty(DataProjectName)),
    LegalEntities = dcountif(LegalEntity, isnotempty(LegalEntity)),
    RoleInstances = dcountif(cloud_RoleInstance, isnotempty(cloud_RoleInstance))
```

## Errors by project

```kusto
DmfErrors
| extend Project = iff(isempty(DataProjectName), "(missing)", DataProjectName)
| summarize
    ErrorEvents = count(),
    Executions = dcountif(ExecutionId, isnotempty(ExecutionId)),
    Entities = dcountif(EntityName, isnotempty(EntityName))
    by Project
| top 20 by ErrorEvents desc
```

## Errors by entity

```kusto
DmfErrors
| extend Entity = iff(isempty(EntityName), "(missing)", EntityName)
| summarize
    ErrorEvents = count(),
    Executions = dcountif(ExecutionId, isnotempty(ExecutionId)),
    Projects = dcountif(DataProjectName, isnotempty(DataProjectName))
    by Entity
| top 20 by ErrorEvents desc
```

## Recurring signatures and blast radius

Normalize volatile values conservatively. Preserve short numbers because they can be meaningful error codes.

```kusto
DmfErrors
| extend NormalizedMessage = replace_regex(
    RedactedErrorMessage,
    @"[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}",
    "<guid>")
| extend NormalizedMessage = replace_regex(
    NormalizedMessage,
    @"0x[0-9a-fA-F]+",
    "<hex>")
| extend NormalizedMessage = replace_regex(
    NormalizedMessage,
    @"\b[0-9]{6,}\b",
    "<id>")
| extend ErrorSignature = substring(trim(" ", NormalizedMessage), 0, 500)
| extend Category = iff(isempty(ErrorCategory), "(missing)", ErrorCategory)
| summarize
    ErrorEvents = count(),
    Executions = dcountif(ExecutionId, isnotempty(ExecutionId)),
    Entities = dcountif(EntityName, isnotempty(EntityName)),
    Projects = dcountif(DataProjectName, isnotempty(DataProjectName)),
    LegalEntities = dcountif(LegalEntity, isnotempty(LegalEntity)),
    RoleInstances = dcountif(cloud_RoleInstance, isnotempty(cloud_RoleInstance)),
    SampleMessage = take_any(RedactedErrorMessage)
    by Category, ErrorSignature
| top 20 by ErrorEvents desc
```

## Zero-filled trend and anomaly heuristic

Select `_binSize` based on the requested window. Typical values are `15m`, `1h`, or `1d`.

```kusto
let _binSize = 1h;
let ErrorTrend =
    DmfErrors
    | summarize NumberOfErrors = count() by TimeBucket = bin(timestamp, _binSize);
let AllBuckets =
    range TimeBucket from bin(_startTime, _binSize) to bin(_endTime, _binSize) step _binSize;
let FilledTrend =
    AllBuckets
    | join kind=leftouter ErrorTrend on TimeBucket
    | project
        TimeBucket,
        NumberOfErrors = coalesce(NumberOfErrors, 0)
    | order by TimeBucket asc;
let CompleteBuckets =
    FilledTrend
    | where TimeBucket >= _startTime
    | where TimeBucket + _binSize <= _endTime;
let MedianErrors = coalesce(
    toscalar(CompleteBuckets | summarize percentile(NumberOfErrors, 50)),
    0.0);
let AnomalyThreshold = max_of(5.0, 3.0 * MedianErrors);
FilledTrend
| extend
    IsPartialBucket = TimeBucket < _startTime or TimeBucket + _binSize > _endTime,
    MedianErrors,
    AnomalyThreshold,
    IsAnomaly = not(IsPartialBucket) and NumberOfErrors >= AnomalyThreshold
| order by TimeBucket asc
```

This is a triage heuristic. It is not a statistical anomaly model.

## Most affected executions

```kusto
DmfErrors
| where isnotempty(ExecutionId)
| summarize
    ErrorEvents = count(),
    Categories = make_set(ErrorCategory, 20),
    Entities = make_set(EntityName, 20),
    Projects = make_set(DataProjectName, 20),
    FirstError = min(timestamp),
    LastError = max(timestamp),
    SampleMessage = take_any(RedactedErrorMessage)
    by ExecutionId, ActivityId
| top 20 by ErrorEvents desc
```

## Execution outcome evidence

Replace `{{ExecutionIdsJson}}` with a JSON array containing the highest-impact execution IDs, for example `["5637148930","5637148931"]`.

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let _outcomeStart = _startTime - 2h;
let _outcomeEnd = _endTime + 4h;
let _observationCutoff = min_of(now(), _outcomeEnd);
let targetExecutionIds = dynamic({{ExecutionIdsJson}});
customEvents
| where timestamp between (_outcomeStart .. _outcomeEnd)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| extend
    ExecutionId = tostring(cd.ExecutionId),
    ActivityId = tolower(case(
        isnotempty(tostring(cd.activityId)), tostring(cd.activityId),
        isnotempty(tostring(cd.ActivityId)), tostring(cd.ActivityId),
        tostring(operation_Id))),
    TotalNumberOfRecords = tolong(cd.TotalNumberOfRecords),
    StagingErrorCount = tolong(cd.StagingErrorCount),
    TargetErrorCount = tolong(cd.TargetErrorCount),
    StagingStatus = tostring(cd.StagingStatus),
    TargetStatus = tostring(cd.TargetStatus),
    EntityName = tostring(cd.EntityName),
    DataProjectName = tostring(cd.DataProjectName)
| where ExecutionId in~ (targetExecutionIds)
| project
    timestamp,
    name,
    ExecutionId,
    ActivityId,
    EntityName,
    DataProjectName,
    TotalNumberOfRecords,
    StagingErrorCount,
    TargetErrorCount,
    StagingStatus,
    TargetStatus
| order by ExecutionId asc, timestamp desc
```

For each execution, take each count or status from the latest event where that field is non-null or non-empty. Record the source event and timestamp. Do not assume the final event carries every value. Use `_observationCutoff` when evaluating whether a run is unfinished, and report `_outcomeStart` through `_outcomeEnd` as the outcome-query window.

## Detail export

```kusto
DmfErrors
| project
    timestamp,
    LegalEntity,
    StagingRecId,
    ExecutionId,
    ActivityId,
    ActivityIdSource,
    ErrorCategory,
    RedactedMessage = RedactedErrorMessage,
    DataProjectName,
    EntityName,
    ExecutionMode,
    cloud_RoleInstance,
    BatchJobId,
    EnvironmentId
| top 1000 by timestamp desc
```

Do not include `user_Id` by default. Add it only when identity attribution is required and authorized. Review the mechanically redacted output before presenting or persisting it because no regular expression can identify every sensitive value.

## Outcome and error-rate rules

For the highest-impact execution IDs, query all related DMF events and retrieve the latest non-null:

- `TotalNumberOfRecords`
- `StagingErrorCount`
- `TargetErrorCount`
- `StagingStatus`
- `TargetStatus`

Calculate staging and target rates separately when the counts are known to describe the same execution and entity:

```text
staging error rate = staging error count / total number of records
target error rate = target error count / total number of records
```

Do not sum cumulative counts repeated across events. Calculate a combined rate only when telemetry semantics prove the staging and target error populations do not overlap.

## AppEvents adaptation

Some workspaces expose `AppEvents` rather than `customEvents`. Confirm the schema, then map:

- `timestamp` to `TimeGenerated`
- `name` to `Name`
- `customDimensions` to `Properties`
- `operation_Id` to `OperationId`
- `cloud_RoleInstance` to the available role-instance column

If `Properties` is a JSON string rather than a dynamic value, use `parse_json(Properties)`.
