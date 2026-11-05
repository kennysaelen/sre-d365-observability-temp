# DMF telemetry query library

Adapt table names and parameter syntax to the selected Azure SRE query tool. The examples assume Application Insights-compatible `customEvents`.

## Parameters

Provide one non-empty identifier:

- `requestedExecutionId`
- `requestedActivityId`

Provide a bounded `_startTime` and `_endTime`.

Before running any query:

1. Replace every `{{...}}` template value.
2. Emit the supplied identifier and emit an empty string for the unused identifier.
3. Verify that no `{{` or `}}` remains.

Example bindings:

```kusto
let _startTime = datetime(2026-09-13T00:00:00Z);
let _endTime = datetime(2026-09-15T00:00:00Z);
let requestedExecutionId = "5637148930";
let requestedActivityId = "";
```

## Schema and identifier probe

Run this bounded probe before concluding that an identifier has no matching telemetry:

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
customEvents
| where timestamp between (_startTime .. _endTime)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| project
    timestamp,
    name,
    ExecutionId = tostring(cd.ExecutionId),
    activityId = tostring(cd.activityId),
    ActivityId = tostring(cd.ActivityId),
    operation_Id,
    cloud_RoleInstance,
    customDimensions
| take 5
```

## Primary execution timeline

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedExecutionId = "{{ExecutionId}}";
let requestedActivityId = tolower("{{ActivityId}}");
customEvents
| where timestamp between (_startTime .. _endTime)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| extend
    ExecutionId = tostring(cd.ExecutionId),
    ActivityId = tolower(coalesce(
        tostring(cd.activityId),
        tostring(cd.ActivityId),
        tostring(operation_Id)))
| where
    (isnotempty(requestedExecutionId) and ExecutionId == requestedExecutionId)
    or (isnotempty(requestedActivityId) and ActivityId == requestedActivityId)
| extend
    BatchJobCaption = tostring(cd.BatchJobCaption),
    BatchJobId = tostring(cd.BatchJobId),
    DataProjectName = tostring(cd.DataProjectName),
    EntityName = tostring(cd.EntityName),
    ExecutionMode = tostring(cd.ExecutionMode),
    LegalEntity = tostring(cd.LegalEntity),
    NumOfStagingNew = tolong(cd.NumOfStagingNew),
    NumOfTargetNew = tolong(cd.NumOfTargetNew),
    NumOfTargetUpdated = tolong(cd.NumOfTargetUpdated),
    StagingErrorCount = toint(cd.StagingErrorCount),
    StagingStatus = tostring(cd.StagingStatus),
    TargetErrorCount = toint(cd.TargetErrorCount),
    TargetStatus = tostring(cd.TargetStatus),
    TotalNumberOfRecords = tolong(cd.TotalNumberOfRecords),
    ErrorMessage = tostring(cd.ErrorMessage),
    StagingRecId = tostring(cd.StagingRecId)
| project
    timestamp,
    name,
    ExecutionId,
    ActivityId,
    DataProjectName,
    EntityName,
    LegalEntity,
    ExecutionMode,
    BatchJobId,
    BatchJobCaption,
    cloud_RoleInstance,
    NumOfStagingNew,
    NumOfTargetNew,
    NumOfTargetUpdated,
    TotalNumberOfRecords,
    StagingErrorCount,
    TargetErrorCount,
    StagingStatus,
    TargetStatus,
    StagingRecId,
    ErrorMessage
| partition hint.strategy=native by ExecutionId
    (
        order by timestamp asc
        | serialize
        | extend GapToNext = next(timestamp) - timestamp
    )
| order by ExecutionId asc, timestamp asc
```

Emit the identifier bindings in one of these exact forms:

```kusto
// Execution ID supplied
let requestedExecutionId = "5637148930";
let requestedActivityId = "";

// Activity ID supplied
let requestedExecutionId = "";
let requestedActivityId = tolower("8ac1d8c0-0000-0000-0000-000000000000");
```

The non-empty guards prevent accidental wildcard behavior. `GapToNext` is null for the final event in each execution.

## Identifier mapping check

Run this check when the caller supplies an `ActivityId`, or when the primary query returns unexpected identifiers:

```kusto
let _startTime = datetime({{StartTimeUtc}});
let _endTime = datetime({{EndTimeUtc}});
let requestedActivityId = tolower("{{ActivityId}}");
customEvents
| where timestamp between (_startTime .. _endTime)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| extend
    ExecutionId = tostring(cd.ExecutionId),
    ActivityId = tolower(coalesce(
        tostring(cd.activityId),
        tostring(cd.ActivityId),
        tostring(operation_Id)))
| where isnotempty(requestedActivityId) and ActivityId == requestedActivityId
| summarize
    EventCount = count(),
    FirstEvent = min(timestamp),
    LastEvent = max(timestamp),
    EventNames = make_set(name, 50)
    by ExecutionId, ActivityId
| order by FirstEvent asc
```

## Concurrent execution check

Use only after the primary execution's start, end, role instance, and entity are known:

```kusto
let _executionStart = datetime({{ExecutionStartUtc}});
let _executionEnd = datetime({{ExecutionEndUtc}});
let investigatedExecutionId = "{{ExecutionId}}";
let investigatedRoleInstance = "{{RoleInstance}}";
let investigatedEntityName = "{{EntityName}}";
customEvents
| where timestamp between (_executionStart .. _executionEnd)
| where name startswith "DMF"
| extend cd = todynamic(customDimensions)
| extend
    ExecutionId = tostring(cd.ExecutionId),
    EntityName = tostring(cd.EntityName)
| where isnotempty(ExecutionId)
| where ExecutionId != investigatedExecutionId
| where
    (isnotempty(investigatedRoleInstance) and cloud_RoleInstance == investigatedRoleInstance)
    or (isnotempty(investigatedEntityName) and EntityName == investigatedEntityName)
| summarize
    EventCount = count(),
    FirstEvent = min(timestamp),
    LastEvent = max(timestamp),
    RoleInstances = make_set(cloud_RoleInstance, 20),
    EventNames = make_set(name, 50)
    by ExecutionId, EntityName
| order by FirstEvent asc
```

## Query adaptation rules

- Some workspaces expose `AppEvents` instead of `customEvents`. Map:
  - `timestamp` to `TimeGenerated`
  - `name` to `Name`
  - `customDimensions` to `Properties`
  - `cloud_RoleInstance` to the available role-instance column
- If `Properties` is a JSON string rather than a dynamic value, use `parse_json(Properties)`.
- Inspect the actual schema before changing column names.
- Keep the identifier and time predicates early in the query.
- Do not use `contains` or substring matching for execution identifiers.
- If results can exceed the tool limit, summarize first or query the execution in smaller bounded windows.
