# D365 Forms: single-page drilldown v2

Standalone Azure SRE Agent skill for one exact page/form name in customer
Application Insights `pageViews` or Log Analytics `AppPageViews`. Compatible
telemetry can come from any Dynamics 365 application; F&O is not assumed.

`SKILL.md` is authoritative agent guidance. This README is for package
reviewers and maintainers. Upload this complete folder, including the bundled
source contract. No sibling package, database function, local observability
agent update, deployment, or Microsoft-internal telemetry is required.

## What it answers

- What reported durations and retained page-view volumes are observed?
- Do a few outliers or repeated slow buckets account for the concern?
- How do hourly/daily patterns and browser, client, app version, SDK version,
  or role-instance distributions vary within this fixed interval?
- How complete are the duration, identity, dimension, and collection data?
- What bounded follow-up evidence is needed?

It does not prove SQL/X++ causes, reconstruct browser traces, measure business
transactions, or perform a week-over-week regression analysis.

## Inputs and examples

Supply an authorized application/workspace resource, an environment mapping
or independently verified isolated resource, and an exact stable page name.
If the same name appears under multiple role names, select one component.
Unknown role can be selected explicitly; it must not mean all roles.

```text
Analyze the exact page name <stable-page-name> in <selected-resource>,
environment <confirmed-environment>. Use the default last seven days.
Show hourly/daily duration and volume, coverage, and bounded client context.
```

```text
Drill into <stable-page-name>, exact role <component>, inheriting the
resource/environment and UTC interval from the current investigation.
Use 3000 ms as a screening threshold, not a service-level objective.
```

Defaults: last seven days, 1-hour trend bins and separate UTC daily bins,
positive reported durations only for performance, known synthetic rows
excluded, 20 slow samples, 20 known dimension buckets plus the missing bucket,
and 5,000 rows/20,000 cells per query result. Smaller tool limits win. Missing,
zero, and invalid durations still contribute to observed usage counts.

## Source schema provenance and limitations

The package uses public telemetry documentation and confirmed runtime schema;
no separately supplied reference file is required. Application-specific
dimensions and system-page exclusions are not universal defaults.

The bundled source contract references:

- https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/apppageviews
- https://learn.microsoft.com/en-us/azure/azure-monitor/app/data-model-complete

Verify the runtime schema and instrumentation. `AppPageViews.DurationMs` is
milliseconds; verify whether `pageViews.duration` is numeric milliseconds or
a timespan before conversion. Unknown units block performance interpretation.
Missing duration leaves a usage-only result. "Reported page-view duration"
is not automatically form-open time, time to interactive, or server time.

A workspace may hold multiple applications. Resource, environment, page,
and role are different boundaries. `TenantId` in AppPageViews is a workspace
identifier, not a Dynamics environment. Role instances are not necessarily
AOS instances. Page names must be stable and safe; sensitive or dynamic names
require a verified safe mapping rather than URL parsing or fuzzy matching.

Observed views are retained telemetry rows, not sampling-adjusted estimates
or business operations. User/session counts are observed nonempty telemetry
identities, not authenticated people; missing identifiers are unknown.
Percentiles use actual retained positive samples without `itemCount`
replication. Sampling, partial retention, late arrivals, unknown synthetic
status, and instrumentation changes limit conclusions. First/last observations
do not prove continuous collection.

## Package contents

```text
d365-forms-form-drilldown-analysis-v2\
  SKILL.md
  README.md
  references\
    page-view-source.md
    queries.md
    interpretation.md
  assets\
    report-template.md
```

Each Forms v2 package carries its own identical source contract. Synchronize
contract copies during maintenance; do not make deployment depend on sibling
folders. Runtime queries concatenate the bundled contract prefix with the
complete local query block. No local let is an installed function.

## Expected report

Scope and provenance; full-population duration and observed usage KPIs;
identity/duration/collection coverage; hourly and daily trends with partial
buckets and gaps; contextual dimension rankings with missing buckets and
full totals; privacy-safe slow observations; evidence-labeled findings and
next actions. A successful empty bucket has 0 observed rows and null duration.
A failed query is unavailable, never zero.

## Validation checklist

- All referenced local files exist; all text is ASCII en-US.
- All eight declared query/validation/chart tools match the target space.
- Every terminal is assembled with Parameters, one Adapter, Normalization,
  Scope, and all of its local lets before submission.
- Reject empty/whitespace page names, unresolved multiple roles, invalid
  intervals, unsafe labels, unresolved resource/environment, and unescaped
  values. Select unknown role explicitly, never implicitly broaden to it.
- Check non-aligned seven-day bounds, exact end exclusion, partial buckets,
  daily boundaries, no rows, sparse traffic, and failed slices.
- Check null/zero/negative/nonfinite duration, missing optional columns,
  missing IDs, mixed sampling weights, known synthetic filtering, and
  repeated IDs across multiple bins without summing distinct counts.
- Confirm top-N leaves full KPIs intact and explicitly reports omitted and
  missing buckets. Samples contain no identifiers or sensitive content.
- Preflight row/cell budgets and stop on truncation. Validate full KQL with
  `ValidateQuery` where supported; local structural checks do not establish
  live KQL compilation, schema compatibility, or query execution success.

For previous-period or weekly change analysis, recommend the weekly
regression skill rather than expanding this package's scope.
