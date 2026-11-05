# D365 Batch Workload Overview

Human-facing documentation for the Azure SRE Agent skill
`d365-batch-workload-overview-v2`.

This skill creates a daily or bounded operational report from Dynamics 365
Finance and Operations batch telemetry emitted to a customer's Application
Insights resource or Log Analytics workspace. It covers workload, duration,
capacity, priority queues, failures, exceptions, and throttling without relying
on Microsoft-internal telemetry.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## Supported telemetry surfaces

| Surface | Event tables | Time/name/properties columns |
|---|---|---|
| Application Insights analytics | `customEvents`, `exceptions` | `timestamp`, `name`, `customDimensions` |
| Log Analytics workspace | `AppEvents`, `AppExceptions` | `TimeGenerated`, `Name`, `Properties` |

The skill resolves the dialect internally from the supplied or configured
resource and a schema probe. Neither the user nor the caller agent selects
tables or rewrites queries. One small source adapter is selected for events
and one for exceptions; all downstream analysis stays shared in `queries.md`.

Each operational query reads one table family without fuzzy unions or
cross-table deduplication. Missing exception tables are reported as unavailable,
not as zero exceptions. An empty event result does not trigger a dialect switch,
and access errors or timeouts are surfaced rather than hidden by fallback.

## Knowledge files

Add these files to the Azure SRE Agent's knowledge:

- `ApplicationInsightsGeneralSchema.json`
- `ApplicationInsightsCustomEvents-Batch.json`
- The current D365 F&O batch-monitoring dashboard JSON

The batch schema currently documents:

- `BatchTaskStart`
- `BatchTaskFinished`
- `BatchTaskFailure`
- `BatchThrottled`
- `BatchQueuesDetails`
- `BatchThreadInfo`

The skill probes the live source because telemetry versions, field casing, and
field population can vary.

## Prerequisites

The Azure SRE Agent space must provide a supported telemetry query tool. This
skill declares:

- `kusto_query`
- `QueryAppInsightsByAppId`
- `QueryAppInsightsByResourceId`
- `QueryLogAnalyticsByResourceId`
- `QueryLogAnalyticsByWorkspaceId`
- `ValidateQuery`
- Optional chart tools listed in `SKILL.md`

Confirm these identifiers against the target Azure SRE Agent tool catalog.

The request should provide:

- An Application Insights App ID/resource ID, Log Analytics workspace
  ID/resource ID, or explicitly compatible Kusto target.
- A bounded UTC time range, or permission to use the last 24 hours.
- One exact D365 F&O environment ID when the source contains multiple
  environments.
- Optional job, class, task, activity, legal-entity, priority, or AOS filters.

## Example prompts

```text
Create the daily D365 batch workload report for environment <environment-id>
from Application Insights app <app-id>. Use the last 24 hours and compare it
with the preceding 24 hours.
```

```text
Review batch workload for environment <environment-id> in Log Analytics
workspace <workspace-id> from 2026-09-14T00:00:00Z through
2026-09-15T00:00:00Z. Include slow jobs, failures, thread utilization, queue
depth, throttling, and telemetry coverage.
```

```text
Analyze batch class <class-name> for environment <environment-id> over the last
seven days. Show completed and unfinished task executions, p95 duration,
failure signals, AOS distribution, and throttling correlation.
```

## Expected report

1. Scope and telemetry coverage.
2. A short executive summary.
3. Headline workload and health KPIs.
4. Slowest completed tasks and job-level workload.
5. Effective thread capacity and utilization by AOS.
6. Priority queue depth and sustained-backlog evidence.
7. Failed task executions and supporting exception telemetry.
8. Throttling events and resource-value coverage.
9. Baseline comparison, findings, confidence, and next actions.

## Package layout

```text
d365-batch-workload-overview-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## Testing checklist

- Confirm the folder name matches the `name` in `SKILL.md`.
- Confirm all declared tools exist in the target Azure SRE Agent space.
- Validate the query library against one Application Insights resource and one
  Log Analytics workspace.
- Assemble each query with exactly one source adapter and the shared analysis.
- Test automatic routing from each supported tool, a generic target, a missing
  preferred table, and targets exposing both table families.
- Confirm zero events do not trigger fallback; access errors and timeouts
  remain visible; absent exception tables produce an unavailable section.
- Confirm dynamic and JSON-string properties normalize identically, optional
  operation/method metadata stays optional, and repeated task events remain
  visible to the pairing logic.
- Test sources containing multiple environment IDs.
- Test missing activity IDs and the job/task fallback key.
- Test repeated starts, both terminal types, missing terminals, and events
  crossing the report boundary.
- Test missing or malformed `InfoMessage`.
- Test nonpositive capacity and current-task counts above effective capacity.
- Test sparse queue/thread sampling and a historical report window.
- Confirm raw messages, stacks, identities, and property bags are absent from
  rendered output.
