---
name: d365-batch-workload-overview-v2
description: Produces a daily or bounded operational overview of Dynamics 365 Finance and Operations batch workload from customer-accessible Application Insights or Log Analytics telemetry. Use for batch capacity, thread utilization, priority queues, completed and unfinished tasks, slow workloads, failures, exceptions, throttling, and workload distribution.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
  - PlotBarChart
  - PlotColumnChart
  - PlotLineChart
  - PlotPieChart
---

# D365 batch workload overview

## Use this skill when

Use this skill for:

- A daily batch-health report, defaulting to the last 24 hours.
- Batch capacity and effective thread-utilization trends by AOS.
- Priority-based scheduling queue depth and sustained backlog.
- Completed, failed, ambiguous, and apparently unfinished batch tasks.
- Slow task and batch-job rankings.
- Batch failure, exception, and throttling signals.
- Workload distribution by class, job, priority, legal entity, and AOS.

This is an operational overview. Treat causal explanations as hypotheses unless
independent evidence supports them.

## Required knowledge

Use the knowledge files supplied with the Azure SRE Agent:

- `ApplicationInsightsGeneralSchema.json`
- `ApplicationInsightsCustomEvents-Batch.json`
- The current D365 F&O batch-monitoring dashboard JSON, when available

Knowledge files describe expected telemetry; the queried source is the runtime
authority. Probe the runtime schema before relying on a field or event.

## Required scope

Obtain a telemetry source and a bounded UTC analysis window. Default to the last
24 hours only when the user does not supply a window.

If the source contains multiple D365 F&O environments, require one exact
environment ID before producing execution, capacity, queue, failure, or
throttling metrics. Optional filters include batch job ID, class name, legal
entity, priority, AOS role instance, activity ID, and batch task ID.

The user and caller agent remain dialect-agnostic. Resolve table names and
source adapters internally from the supplied or configured telemetry resource;
do not request a dialect flag. Clarify only a genuinely ambiguous target.

## Workflow

1. **Choose the narrowest query tool.**
   - Application Insights App ID or resource ID: use an Application Insights
     query tool.
   - Log Analytics workspace ID or resource ID: use a Log Analytics query tool.
   - Use `kusto_query` only for an explicitly supplied compatible Kusto target.
2. **Resolve the table dialect and schema.**
   - Application Insights uses `customEvents`, `exceptions`, `timestamp`,
     `name`, `customDimensions`, and `cloud_RoleInstance`.
   - Log Analytics uses `AppEvents`, `AppExceptions`, `TimeGenerated`, `Name`,
     `Properties`, and `AppRoleInstance`.
   - Follow automatic source resolution in `references/queries.md`. Probe the
     tool-preferred table schema, then select exactly one source adapter for
     events and one for exceptions internally.
   - Compose parameters, the selected adapter, and shared analysis. Never union
     both table families or require the caller to rewrite queries.
   - Try the alternate table only after a confirmed missing-table result on
     the same target. Zero events do not mean a missing table. Surface access,
     timeout, and incompatible-schema errors rather than switching resources.
   - If exception tables are absent, mark that section unavailable and continue
     with event analysis. Validate assembled queries on the selected surface
     when supported.
   - Confirm event names, field casing, nested `InfoMessage` JSON, environment
     coverage, and observation cadence.
3. **Build the canonical datasets.**
   - Normalize environment, activity, job, and task identifiers
     case-insensitively.
   - Prefer `activityId` as the execution correlation key. Fall back to the
     combination of batch job ID and task ID only when both are present.
   - Pair `BatchTaskStart` with `BatchTaskFinished` or `BatchTaskFailure`.
   - Keep missing, repeated, conflicting, left-truncated, and unfinished pairs
     visible, but exclude them from completed-duration aggregates.
   - Use a pairing buffer around the requested window and report its length.
4. **Measure telemetry coverage before interpreting health.**
   - Report expected event names that are absent.
   - Report missing environment IDs, activity IDs, task IDs, job IDs, role
     instances, and malformed `InfoMessage` payloads.
   - Report first/last sample time and maximum sample gap for thread and queue
     telemetry.
   - Do not interpret missing telemetry as zero workload or healthy capacity.
5. **Calculate headline workload KPIs.**
   - Distinct correlated task executions started in the report window.
   - Completed, failed, unfinished, ambiguous, and left-truncated executions.
   - Completion and failure rates only when their denominators are explicit.
   - Task starts per time bucket as throughput, never as concurrent-running
     tasks.
6. **Analyze slow workload.**
   - Rank only unambiguous completed task pairs by observed duration.
   - Prefer the emitted `elapsedMilliseconds` as a validation signal, not as a
     replacement for pairing; report material disagreement.
   - Aggregate job-level workload from the full paired-task dataset before
     limiting results.
7. **Analyze thread capacity.**
   - Effective capacity is `MaxThreadCount - ReservedNumberOfThreads`.
   - Utilization is `CurrentBatchTasks / EffectiveCapacity`.
   - Available threads are `EffectiveCapacity - CurrentBatchTasks`.
   - Report invalid samples where capacity is nonpositive or current tasks
     exceed effective capacity; do not silently clamp them.
   - Show median, p95, and peak utilization by AOS plus a time trend.
8. **Analyze queue depth.**
   - Include critical, high, normal, low, reserved-capacity, ready-buffer, and
     prioritized-ready-buffer fields when present.
   - Report peak, p95, median, nonzero duration, and longest sustained nonzero
     sequence. A single sample is not proof of sustained backlog.
9. **Analyze throttling.**
   - Count `BatchThrottled` events and affected AOS instances.
   - Trend throttles beside task-start throughput, but do not label starts as
     running-task concurrency.
   - Summarize CPU, memory, and SQL DTU values from `InfoMessage` only after
     conversion and coverage checks.
   - Do not attribute throttles to a job when the job ID is empty or `0`.
10. **Analyze failures without double counting.**
    - `BatchTaskFailure` proves a failed correlated task execution.
    - Batch-mode rows in `exceptions` or `AppExceptions` are supporting
      exception telemetry and may overlap with failure events.
    - Report the two signal families separately unless a defensible correlation
      key proves uniqueness.
    - Group only by approved exception type/source/class signatures. Redact raw
      messages, call stacks, user IDs, tenant IDs, IP data, and payloads.
11. **Compare with a baseline when requested or useful.**
    - Use the immediately preceding disjoint window of equal length.
    - Compare rates, median/p95 duration, peak/p95 utilization, p95 queue depth,
      failures, and throttles.
    - Report sample counts and suppress percentage changes when the baseline is
      zero or coverage is materially different.
12. **Render the report** with `assets/report-template.md` and the evidence rules
    in `references/interpretation.md`.

## Guardrails

- Never run an unbounded query or mix environments.
- Never use internal Microsoft service telemetry; query only the
  customer-provided Application Insights, Log Analytics, or compatible target.
- Never assume that a knowledge-file field is populated at runtime.
- Never calculate task duration from unrelated minimum and maximum timestamps.
- Never treat `BatchTaskStart` counts as active concurrency.
- Never treat a missing terminal event near the observation cutoff as a proven
  stuck task; apply ingestion lag and silence rules.
- Never claim that `BatchTaskFailure` and exception-table rows are unique
  failures without correlation evidence.
- Never expose raw exception messages, event messages, call stacks,
  `customDimensions`/`Properties`, user IDs, tenant IDs, session IDs, IP data,
  or source payloads.
- Never assign critical severity solely because an absolute count is high.
- Report source, UTC window, environment, filters, pairing buffer, ingestion
  lag, row limits, sampling, truncation, and coverage limitations.

## Completion criteria

Include:

- Scope, source dialect, UTC window, environment, filters, and data coverage.
- Headline task-execution and health KPIs with explicit denominators.
- Slowest completed tasks and job-level workload.
- Thread-capacity configuration and utilization by AOS.
- Priority queue depth and sustained-backlog evidence.
- Failure events and exception telemetry as separate signal families.
- Throttling trend and resource-value coverage.
- Current-versus-baseline comparison when available.
- Evidence-labeled findings, confidence, severity, and concrete next actions.