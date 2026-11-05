---
name: d365-dmf-execution-duration-analysis-v2
description: Identifies slow or degraded Dynamics 365 Finance and Operations Data Management Framework (DMF) imports and exports using explicitly paired start/end events, duration distributions, completed-record counts, per-record processing time, and comparable-window trends. Use for slow executions, entity or data-project duration rankings, regressions, and efficiency analysis.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
  - PlotBarChart
  - PlotColumnChart
  - PlotScatter
---

# D365 DMF execution-duration analysis

## Use this skill when

Use this skill for:

- Ranked slow DMF import or export executions.
- Duration comparisons by entity or data project.
- Duration-per-record and records-per-second efficiency.
- Current-versus-prior duration regressions.
- Distinguishing long runs caused by volume from inefficient processing.

Use `d365-dmf-unfinished-runs-v2` for runs without a terminal event and `d365-dmf-execution-drilldown-v2` for root-cause reconstruction of one execution.

## Required scope

Obtain a telemetry source and bounded UTC analysis window. Default to the last 24 hours only when the user provides no window. Optional filters include environment, entity, data project, direction, execution ID, and activity ID.

If the source contains more than one environment, an exact environment filter is mandatory; do not aggregate or pair execution IDs across environments.

## Workflow

1. **Choose the narrowest query tool** based on the supplied App ID, workspace ID, resource ID, or ADX target.
2. **Confirm the schema** with `references/queries.md`, including event names, identifier casing, record-count fields, and metadata availability on start and end events.
3. **Build `PerExecution`.**
   - Fetch a bounded buffer before and after the analysis window so pairs crossing a window boundary can be resolved.
   - Pair start and end explicitly by normalized `ExecutionId`.
   - Deduplicate end rows at execution/entity grain; multiple entities in one execution are valid and their final counts are summed.
   - Treat repeated starts, repeated ends for the same execution/entity, or reversed timestamps as ambiguous rather than complete.
   - Keep incomplete, left-truncated, and duplicate/retried executions visible, but exclude them from completed-duration aggregates.
   - Never calculate duration as `max(timestamp) - min(timestamp)` without verifying that both event types are present.
4. **Rank completed executions** by duration and show record volume, milliseconds per record, and records per second.
5. **Report incomplete and truncated executions separately.** Do not represent them as zero-duration runs.
6. **Aggregate cumulative processing time** by entity/project using the full `PerExecution` dataset, not a top-N subset. State that concurrent durations can overlap and therefore exceed wall-clock time.
7. **Compare efficiency.**
   - Use end-event record counts only.
   - Exclude missing/zero record counts and durations shorter than five seconds from per-record rate rankings.
   - Use a scatter plot of records versus duration when helpful.
8. **Analyze regressions** with disjoint, equal-length windows.
   - Compare median and p95 duration, not mean alone.
   - Require at least five completed executions in both windows.
   - Use the formula in `references/queries.md`; positive improvement percentage means faster.
9. **Assign severity contextually.**
   - Always state thresholds.
   - Absolute defaults: 5–15 minutes warning, over 15 minutes critical.
   - Downgrade absolute-duration severity when per-record efficiency is normal for the entity/project.
   - Prefer trailing entity/project baselines when enough history exists.
10. **Apply evidence labels** from `references/interpretation.md`.
11. **Render the result** using `assets/report-template.md`.

## Guardrails

- Never include incomplete executions in duration, per-record, or trend aggregates.
- Never aggregate from a previously limited top-N result.
- Never divide by zero, null records, or nonpositive duration.
- Never infer direction from an execution-ID string.
- Never emit raw `customDimensions`, user identifiers, file paths, or source payloads.
- Treat missing metadata as unavailable, not zero.
- Report query window, pairing buffer, filters, truncation, sampling, and coverage.
- Validate adapted KQL before execution when `ValidateQuery` supports the selected surface.

## Completion criteria

Include:

- Source, analysis window, pairing buffer, filters, and cohort rule.
- Counts of complete, incomplete, truncated, and duplicate/retried executions.
- Slowest complete executions with duration and efficiency.
- Entity/project cumulative duration and distribution.
- Current/prior regression results with sample counts.
- Facts, correlations, hypotheses, confidence, and next actions.