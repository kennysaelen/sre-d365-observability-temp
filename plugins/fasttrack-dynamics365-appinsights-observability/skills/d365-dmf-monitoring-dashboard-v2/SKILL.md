---
name: d365-dmf-monitoring-dashboard-v2
description: Produces a host-agnostic health overview of Dynamics 365 Finance and Operations Data Management Framework (DMF) telemetry. Use for environment health checks, daily or periodic digests, import/export mix, completed volume, duration, errors, unfinished runs, coverage, and routing to focused DMF analysis skills.
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

# D365 DMF monitoring dashboard

## Use this skill when

Use this skill as the broad DMF entry point for:

- Environment-wide DMF health over a bounded period.
- Daily or periodic import/export digest.
- Completed executions and record volume.
- Duration, error, and unfinished-run headlines.
- Identifying which focused DMF skill should investigate next.

Do not use this dashboard as a substitute for detailed RCA. It summarizes signals and routes the highest-impact findings.

## Required scope

Obtain a telemetry source, bounded UTC analysis window, and environment scope when the source contains multiple environments. Optional filters include entity, project, execution, activity, legal entity, and direction.

If the source contains more than one environment, require one exact environment ID before building any execution-level dataset.

## Workflow

1. **Choose the narrowest query tool** for the supplied App ID, workspace ID, resource ID, or ADX target.
2. **Confirm the schema and coverage** using `references/queries.md`.
   - Confirm table, event names, custom-dimension casing, role-instance column, and identifier coverage.
   - Report unknown `DMF*` event names rather than silently dropping them.
3. **Build the canonical event datasets.**
   - Use the correctly bounded one-day lookback before `_startTime` to resolve direction.
   - Define `allDMFEvents` explicitly.
   - Normalize GUID-like IDs case-insensitively and data-project suffixes consistently.
   - Drop sensitive identity and tenant fields from dashboard rows.
4. **Calculate headline KPIs.**
   - End-anchored distinct completed executions and completed records from the same deduplicated cohort.
   - Completed record volume after per-execution/entity deduplication.
   - DMF error events and affected executions.
   - Unfinished starts without a matching end, after ingestion-lag protection.
   - Coverage for missing IDs/counts and duplicate end events.
   - Pairing coverage is separate: executions without one unambiguous start are excluded only from duration metrics.
5. **Build focused insights.**
   - Import/export completed-execution mix.
   - Executions and record volume by entity and project.
   - Cumulative duration by entity and project using explicit start/end pairs.
   - Error signatures/projects plus zero-filled trend.
   - Risk-ordered unfinished runs.
   - Slowest complete executions.
6. **Use consistent semantics.**
   - Count distinct execution IDs for execution KPIs.
   - Treat `DMFError` as an error event, not a terminal event or failed execution.
   - Use end-event record counts only and deduplicate before summing.
   - Exclude incomplete pairs from duration aggregates.
   - State that cumulative duration can exceed wall-clock time because runs overlap.
7. **Assign severity contextually.**
   - Do not mark every active unfinished row critical.
   - Error thresholds use affected executions and a zero-filled trend with an absolute floor.
   - Duration thresholds are contextualized by volume and baseline.
   - Surface coverage limitations before confidence claims.
8. **Build the findings contract** in `assets/findings-contract.md`.
9. **Render in the host’s supported format.**
   - Charts are optional presentation aids, not required evidence.
   - Preserve the contract fields across Markdown, HTML, Adaptive Card, or console output.
10. **Route focused follow-up.**
    - Duration/performance: `d365-dmf-execution-duration-analysis-v2`
    - Errors: `d365-dmf-error-rca-v2`
    - Unfinished runs: `d365-dmf-unfinished-runs-v2`
    - Throughput/volume: `d365-dmf-throughput-volume-analysis-v2`
    - One execution: `d365-dmf-execution-drilldown-v2`

## Guardrails

- Never run an unbounded query or mix environments unintentionally.
- Never claim `allDMFEvents` exists unless it is explicitly defined in the submitted query.
- Never infer direction from an execution ID.
- Never count raw end events as distinct executions.
- Never sum repeated final record counts without deduplication.
- Never calculate duration from an unverified single event.
- Never use `DMFError` as a completion signal.
- Never emit raw error messages, user IDs, tenant IDs, custom dimensions, file paths, or source payloads.
- Report windows, lookbacks, ingestion lag, filters, row limits, sampling, missing fields, and truncation.

## Completion criteria

Include:

- Scope and coverage.
- Reconciled KPI totals.
- Direction, entity, project, volume, duration, error, and unfinished-run insights.
- Unknown event types and data-quality limitations.
- Severity with explicit rules.
- Focused handoffs for the most important findings.