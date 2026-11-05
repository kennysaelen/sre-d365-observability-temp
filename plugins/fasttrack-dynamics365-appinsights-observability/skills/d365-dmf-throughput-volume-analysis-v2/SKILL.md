---
name: d365-dmf-throughput-volume-analysis-v2
description: Analyzes Dynamics 365 Finance and Operations Data Management Framework (DMF) completed-record volume and throughput by entity, data project, direction, and execution. Use for record totals, import/export mix, largest runs, records-per-hour trends, current-versus-prior volume shifts, and records-per-second efficiency.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
  - PlotColumnChart
  - PlotPieChart
  - PlotLineChart
---

# D365 DMF throughput and volume analysis

## Use this skill when

Use this skill to answer:

- How many records were completed?
- Which entity, data project, or direction dominates volume?
- What is the records-per-hour trend?
- Which executions processed the most records?
- Is volume materially higher or lower than the previous comparable period?
- Which high-volume executions have poor records-per-second efficiency?

Use `d365-dmf-execution-duration-analysis-v2` for duration regressions and `d365-dmf-execution-drilldown-v2` for one execution.

## Required scope

Obtain a telemetry source, bounded UTC window, and environment scope when the telemetry source contains multiple environments. Optional filters include entity, project, direction, execution, and legal entity.

If the source contains more than one environment, require one exact environment ID before grouping or joining by execution ID.

## Workflow

1. Choose the narrowest query tool for the supplied resource identifier.
2. Confirm the event schema and final-count semantics using `references/queries.md`.
3. Build the deduplicated `DmfEnd` dataset.
   - Use only import/export `JobEnd` events for completed-record volume.
   - Deduplicate duplicate ingestion or retries before aggregating.
   - Preserve execution/entity grain when one execution covers multiple entities.
   - Normalize data-project GUID suffixes consistently.
4. Report volume by entity, project, and direction from the per-execution/entity dataset.
5. Report workload distributions using execution count, median, p95, average, and maximum records—not average alone.
6. Report the largest executions with project, entity, direction, completion time, and record count for direct handoff.
7. Measure throughput explicitly.
   - Operational throughput: completed records per wall-clock hour/bin.
   - Processing efficiency: records per execution second, only when a complete start/end pair and positive duration exist.
   - Report partial bins but exclude them from rate comparisons.
   - Do not call a raw window total "throughput."
8. Compare disjoint equal-length current and prior windows.
   - Fetch both windows explicitly.
   - Exclude or normalize partial first/last bins.
   - Classify new and disappeared workloads separately from percentage change.
   - Use `UnknownCoverage` when missing record counts prevent a reliable classification.
   - Require an absolute materiality floor before flagging a percentage swing.
   - State the materiality floor used; default to 1,000 records.
9. Calculate shares and changes using floating-point arithmetic and guard zero denominators.
10. Apply evidence/confidence guidance from `references/interpretation.md`.
11. Render with `assets/report-template.md`.

## Guardrails

- Never sum repeated `TotalNumberOfRecords` values directly from raw events.
- Never mix environments or tenants in one result unless explicitly requested.
- Never silently drop executions with missing record counts; report coverage.
- Never divide by zero or nonpositive duration.
- Never compare a partial current day with a complete prior day without normalization.
- Never interpret volume growth as good or bad without workload context.
- Never emit tenant IDs, user IDs, raw custom dimensions, file paths, or source data.
- State source, windows, filters, count grain, missing-count coverage, and thresholds.

## Completion criteria

Include:

- Scope and count-grain definition.
- Completed record volume by entity, project, and direction.
- Missing/zero-count coverage.
- Execution-size distribution and largest executions.
- Records-per-hour trend.
- Comparable-window change with new/disappeared/material workloads.
- Processing efficiency when valid duration pairs are available.
- Evidence labels, confidence, and next actions.