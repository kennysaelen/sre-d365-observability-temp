# Page-view source contract (standalone package copy)

This file is identical in all three Forms v2 packages so each package can be
uploaded independently. Keep copies synchronized when the source contract
changes; no sibling skill or external query function is required.

## Evidence and meaning

Use confirmed runtime schema, available emitter documentation, and these
public telemetry references. No separately supplied reference file is required:

- https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/apppageviews
- https://learn.microsoft.com/en-us/azure/azure-monitor/app/data-model-complete

`pageViews` maps to `AppPageViews`, NOT `AppEvents`. AppPageViews.DurationMs is
milliseconds. Confirm the Application Insights duration type/encoding:
verified numeric milliseconds can be divided by 1000 to display seconds;
a verified timespan needs conversion by `1ms` to canonical milliseconds.
Do not infer units from magnitude. Unrecognized units or encodings block
duration analysis, though observed usage can still be reported.

Application-specific dimensions, system-form exclusions, and the interpretation
"form open" are not universal Dynamics 365 conventions. Require evidence from
the selected application's instrumentation rather than assuming F&O behavior.

Call this **reported page-view duration** until the emitter's documentation
establishes form-open semantics. It is not necessarily server time, time to
interactive, or a complete browser-load waterfall. No telemetry means no
observations, not zero usage in the actual application.

## Resource, environment, and page identity

Use one authorized application resource and one confirmed environment scope
per analysis. A workspace can contain many application resources. Discover
the choices with the bounded scope query; never combine them automatically.
Ask for a choice when ambiguous. A role name is an application component,
not an environment and not an AOS instance.

Verify the environment's top-level property key from supplied application-specific
schema or observed property names. `environmentId` / `EnvironmentId` are
candidates, not required fields in every Dynamics application. If both are
verified, compare them for conflicts. A missing property can be acceptable
only when independent resource configuration establishes an isolated
environment. Do not invent an environment from a tenant ID, URL, or role.
The dynamic index used below takes a literal key, not a dotted JSON path.
Nested environment values require an explicitly validated adapter mapping;
do not silently treat an unresolved nested value as resource isolation.

Group pages by exact `(PageName, RoleName)` within the selected resource and
environment. Keep roles separate; missing role is an explicit unknown bucket,
not proof that all rows came from the same component. Use stable instrumented
names, not URL query strings or record-specific page titles. If names are
dynamic, sensitive, or collide across distinct pages, stop page-level ranking
and request a verified stable identifier/mapping rather than inventing URL
normalization. Missing names remain coverage, not a fabricated form.

## Source selection and assembly

The caller and user do not select the dialect. App Insights query tools prefer
`pageViews`; Log Analytics tools prefer `AppPageViews`. For generic
`kusto_query`, inspect the target metadata and resource boundary first.
Probe the preferred table using `<table> | take 0 | getschema`. Inspect
time/name, duration type, properties, resource identity, and optional columns.
Try the alternate family on the SAME target only on a confirmed missing-table
response, never on zero rows, timeout, access denial, or incompatible schema.
If both exist, use the tool-preferred family, not a union or cross-family dedup.

Every terminal query is submitted as:
**Parameters + exactly one adapter + Normalization + Scope + required local
datasets from queries.md + one terminal**. Discovery needs only Parameters,
adapter, and Normalization. `RawPages`, `Candidates`, `ResourceRows`, `Scoped`,
`PageRows`, and `NamedPages` are local `let` bindings, not physical tables.

Freeze UTC bounds once. Require start < end <= observed time. Use half-open
intervals. Escape user values as KQL literals or use supported typed parameters;
never concatenate unescaped resource/page names. Validate each assembled
query through `ValidateQuery` if available, and disclose limitations.
Local text checks are not service-side KQL compilation.
If the query tool/API has a separate time-range argument, set it to cover the
entire same fetch interval. A portal or tool default of 24 hours must not
silently clip a seven-day or four-week query. Discover the actual argument
schema; do not invent a parameter name. Host time/row limits and shortened
retention are partial coverage, not zero earlier traffic.

## Parameters

Replace every token. Booleans default false until scope is confirmed.
`SingleAppResourceKey` is allowed only when the tool's target is independently
known to be one app (for example an App ID); never supply a workspace ID as it.
`SelectedResourceKey` must match the discovered canonical resource key.
Environment IDs are case-normalized; page/role selectors remain exact.

```kusto
let _observedAtUtc=datetime({{ObservedAtUtc}});
let _startTime=datetime({{StartTimeUtc}});
let _endTime=datetime({{EndTimeUtc}});
let _selectedResourceKey=tolower("{{SelectedResourceKey}}");
let _singleAppResourceKey=tolower("{{SingleAppResourceKey}}");
let _environmentProperty="{{EnvironmentProperty}}";
let _alternateEnvironmentProperty="{{AlternateEnvironmentProperty}}";
let _selectedEnvironmentId=tolower("{{EnvironmentId}}");
let _resourceEnvironmentIsolated=false;
let _scopeConfirmed=false;
let _aiDurationEncoding="milliseconds";
let _pageName="{{ExactPageNameOrEmpty}}";
let _roleName="{{ExactRoleNameOrEmpty}}";
let _roleFilterEnabled=false;
let _excludeKnownSynthetic=true;
```

For a verified timespan AI column set `_aiDurationEncoding="timespan"`.
If the duration column is absent, preserve null duration and report that
performance analysis is unavailable. Optional columns below must stay absent/
unknown rather than be invented from unrelated fields.

## Adapter A: Application Insights

```kusto
let RawPages=pageViews
    | where timestamp >= _startTime and timestamp < _endTime
    | project EventTime=timestamp,PageName=tostring(name),
        SourceKey=tolower(coalesce(tostring(column_ifexists("_ResourceId","")),_singleAppResourceKey)),
        P=parse_json(tostring(column_ifexists("customDimensions",dynamic({})))),
        RawDuration=tostring(column_ifexists("duration","")),
        Weight=todouble(column_ifexists("itemCount",real(null))),
        RoleName=tostring(column_ifexists("cloud_RoleName","")),
        RoleInstance=tostring(column_ifexists("cloud_RoleInstance","")),
        Browser=tostring(column_ifexists("client_Browser","")),
        ClientType=tostring(column_ifexists("client_Type","")),
        AppVersion=tostring(column_ifexists("application_Version","")),
        SDKVersion=tostring(column_ifexists("sdkVersion","")),
        UserKey=tostring(column_ifexists("user_Id","")),
        SessionKey=tostring(column_ifexists("session_Id","")),
        SyntheticSource=tostring(column_ifexists("operation_SyntheticSource",""))
    | extend RawDurationMs=case(
        _aiDurationEncoding == "milliseconds",todouble(RawDuration),
        _aiDurationEncoding == "timespan",todouble(totimespan(RawDuration)/1ms),
        real(null));
```

## Adapter L: Log Analytics

```kusto
let RawPages=AppPageViews
    | where TimeGenerated >= _startTime and TimeGenerated < _endTime
    | project EventTime=TimeGenerated,PageName=tostring(Name),
        SourceKey=tolower(tostring(column_ifexists("_ResourceId",""))),
        P=parse_json(tostring(column_ifexists("Properties",dynamic({})))),
        RawDurationMs=todouble(column_ifexists("DurationMs",real(null))),
        Weight=todouble(column_ifexists("ItemCount",real(null))),
        RoleName=tostring(column_ifexists("AppRoleName","")),
        RoleInstance=tostring(column_ifexists("AppRoleInstance","")),
        Browser=tostring(column_ifexists("ClientBrowser","")),
        ClientType=tostring(column_ifexists("ClientType","")),
        AppVersion=tostring(column_ifexists("AppVersion","")),
        SDKVersion=tostring(column_ifexists("SDKVersion","")),
        UserKey=tostring(column_ifexists("UserId","")),
        SessionKey=tostring(column_ifexists("SessionId","")),
        SyntheticSource=tostring(column_ifexists("SyntheticSource",""));
```

Do not substitute `TenantId` for a Dynamics environment: in AppPageViews it is
the workspace ID. If `_ResourceId` is absent from a workspace-wide surface,
resolve a verified application-resource mapping or stop; missing identity is
not permission to mix resources.

## Normalization

```kusto
let Candidates=RawPages
    | extend EnvironmentPrimary=tolower(trim(@"\s+",iff(isnotempty(_environmentProperty),tostring(P[_environmentProperty]),""))),
        EnvironmentAlternate=tolower(trim(@"\s+",iff(isnotempty(_alternateEnvironmentProperty),tostring(P[_alternateEnvironmentProperty]),"")))
    | extend EnvironmentId=coalesce(EnvironmentPrimary,EnvironmentAlternate),
        EnvironmentConflict=isnotempty(EnvironmentPrimary) and isnotempty(EnvironmentAlternate)
            and EnvironmentPrimary != EnvironmentAlternate
    | extend DurationState=case(isnull(RawDurationMs),"MissingOrUnparseable",
        not(isfinite(RawDurationMs)) or RawDurationMs < 0,"Invalid",
        RawDurationMs == 0,"Zero","Positive"),
        ValidWeight=isnotnull(Weight) and isfinite(Weight) and Weight >= 1 and Weight == round(Weight,0),
        KnownSynthetic=isnotempty(SyntheticSource)
    | extend DurationMs=iff(DurationState == "Positive",RawDurationMs,real(null));
```

This skill uses positive durations to define the performance cohort.
Zero-duration rows still count as observed page views but are excluded from
duration statistics with an explicit count. Do not silently drop slow outliers,
deduplicate emissions, or multiply duration samples by sampling weight.

## Discovery terminal (before scope)

```kusto
Candidates
| summarize ObservedRows=count() by SourceKey,EnvironmentId,EnvironmentConflict,RoleName
| order by ObservedRows desc
| take 1001
```

If more than 1,000 groups are returned or results are truncated, do not choose
from an incomplete discovery result. Narrow to a supplied resource and rerun.
Report missing source/environment and conflicting-environment coverage.
If no environment mapping is verified, this query does NOT prove isolation.
No rows cannot discover or confirm an environment.

## Scope

Set `_scopeConfirmed=true` only after resource/environment scope is resolved.
Set `_resourceEnvironmentIsolated=true` only from independent resource
configuration evidence, not just the absence of environment metadata.
When a specific environment is selected, missing-ID rows remain excluded.
Names that are whitespace-only are missing; do not trim nonblank page names
or merge case variants without a documented emitter-specific rule.

```kusto
let ResourceRows=Candidates | where isnotempty(_selectedResourceKey) and SourceKey == _selectedResourceKey;
let Scoped=ResourceRows
    | where _scopeConfirmed and not(EnvironmentConflict)
    | where (isnotempty(_selectedEnvironmentId) and EnvironmentId == _selectedEnvironmentId)
        or (isempty(_selectedEnvironmentId) and _resourceEnvironmentIsolated)
    | where not(_roleFilterEnabled) or RoleName == _roleName
    | where isempty(_pageName) or PageName == _pageName;
let PageRows=Scoped | where not(_excludeKnownSynthetic) or not(KnownSynthetic);
let NamedPages=PageRows | where isnotempty(trim(@"\s+",PageName));
```

## Scope and collection coverage terminals

Run separately, with the same parameters/adapter. Never attribute unscoped
rows to the chosen environment or count the two cohorts as interchangeable.

```kusto
ResourceRows
| summarize ResourceRows=count(),MissingEnvironmentRows=countif(isempty(EnvironmentId)),
    ConflictingEnvironmentRows=countif(EnvironmentConflict)
```

```kusto
Scoped
| summarize BeforeSyntheticFilter=count(),KnownSyntheticRows=countif(KnownSynthetic)
```

```kusto
PageRows
| summarize ObservedViews=count(),MissingPageNames=countif(isempty(trim(@"\s+",PageName))),
    PositiveDurationViews=countif(DurationState == "Positive"),
    ZeroDurationViews=countif(DurationState == "Zero"),
    MissingDurationViews=countif(DurationState == "MissingOrUnparseable"),
    InvalidDurationViews=countif(DurationState == "Invalid"),
    SamplingIndicatedViews=countif(ValidWeight and Weight > 1),
    UnknownWeightViews=countif(not(ValidWeight)),
    MissingRoleViews=countif(isempty(RoleName)),
    MissingUserViews=countif(isempty(UserKey)),MissingSessionViews=countif(isempty(SessionKey)),
    FirstObserved=min(EventTime),LastObserved=max(EventTime)
```

Missing optional columns are reported from schema preflight as well as row
coverage. No sampling indication is not proof of complete collection.
First/last timestamps and a successful query do not establish continuous
collection or sufficient retention. Record known instrumentation changes,
outages, retention boundaries, and late-arrival risk separately.
Known synthetic views are excluded by default; blank synthetic metadata does
not establish that a view was human. User/session identifiers may be used
inside aggregates but NEVER returned. Distinct IDs are observed telemetry
identities, not authenticated people; missing IDs do not form one user.

## Budgets, output, and failures

Default query-result budget: 5,000 grouped rows / 20,000 chart cells; honor
smaller host limits. Preflight cardinality before grids or full result export.
Split by deterministic disjoint page/role keys or time buckets if needed.
Never compute exact percentiles by averaging percentile outputs from slices:
merge raw eligible samples within budget or compute each entire group's
percentiles server-side. Preserve frozen global bounds and reconcile totals.
Failed slices are unavailable, not zero. Do not analyze a silently truncated
result as complete or rank only the first returned subset.

`take` is permitted only after full-population ranking or for an explicitly
labeled diagnostic sample. Report full totals versus displayed rows. Counts
are retained events; sampling-weight estimates, if requested, are separately
labeled and only valid where all included weights are finite positive integers.
Do not fabricate estimated unique users or weighted duration percentiles.

Keep raw URLs, property bags, user/session IDs, IPs, record titles, tenant IDs,
and error text out of reports. Treat labels as literal data and escape Markdown
and chart labels. Do not emit raw rows just to discover identity fields.
