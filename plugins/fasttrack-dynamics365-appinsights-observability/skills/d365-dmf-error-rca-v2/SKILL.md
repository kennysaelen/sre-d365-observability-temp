---
name: d365-dmf-error-rca-v2
description: Analyzes Dynamics 365 Finance and Operations Data Management Framework (DMF) error telemetry across a bounded time window. Use for failed or partially successful imports and exports, recurring DMFError signatures, affected entities or data projects, error trends, blast radius, and evidence-based root-cause analysis.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
  - PlotColumnChart
  - PlotLineChart
---

# D365 DMF error root-cause analysis

## Use this skill when

Use this skill to analyze **multiple DMF error events** within a bounded period, including:

- Import or export failures and partial successes.
- Error spikes by entity, data project, legal entity, or execution.
- Recurring error signatures and their blast radius.
- Whether failures appear isolated, data-specific, configuration-related, or systemic.
- Error-rate analysis when a trustworthy record-volume denominator is available.

Do not use this skill for:

- A detailed reconstruction of one known execution; use `d365-dmf-execution-drilldown-v2`.
- Finding unfinished runs without error telemetry; use `d365-dmf-unfinished-runs`.
- Environment-wide telemetry without a bounded time range and telemetry source.

## Telemetry assumptions

`DMFError` is an error signal, not automatically a failed execution or a failed-record count. An execution can retry, partially succeed, or emit multiple error events for one underlying problem. Verify execution outcome using terminal DMF events, staging and target statuses, and record/error counts before classifying impact.

## Required scope

Obtain:

1. The Application Insights application, Log Analytics workspace, Azure resource, or ADX database to query.
2. A bounded time window; default to the last 24 hours only when the user supplies none.
3. Any available environment, entity, data-project, execution, activity, or legal-entity filters.

Always report the effective source, time window, filters, and observation cutoff.

## Workflow

1. **Choose the narrowest query tool.**
   - Prefer App ID or workspace ID tools when those identifiers are known.
   - Otherwise use the corresponding resource ID tool.
   - Use `kusto_query` only for an Azure Data Explorer source.

2. **Confirm the schema.**
   - Run the bounded schema probe in `references/queries.md`.
   - Confirm whether the table is `customEvents` or `AppEvents`.
   - Confirm timestamp, name, property-bag, operation/activity, user, and role-instance columns.
   - Verify the exact casing and presence of each custom dimension before interpreting zero rows.

3. **Build the filtered error dataset.**
   - Load `references/queries.md`.
   - Define `_startTime` and `_endTime` explicitly.
   - Apply the time and `DMFError` predicates first.
   - Apply only populated optional filters.
   - Use case-insensitive exact matching for GUID-like IDs and names; never use substring matching for identifiers.
   - Run the error-volume precheck before materializing a large result. Narrow the window or add filters when the volume is excessive.
   - Never submit a query that still contains `{{` or `}}`.

4. **Establish the overall impact.**
   - Count error events and distinct executions.
   - Count affected entities, projects, legal entities, and role instances.
   - Identify executions with the most error events.
   - Do not label error-event count as failed-record count unless telemetry semantics prove one event per failed record.

5. **Rank affected projects and entities.**
   - Report error events and distinct executions together.
   - Separate missing project/entity values rather than silently combining them with named workloads.
   - Use the column-chart tool only when a chart improves comparison.

6. **Group recurring error signatures.**
   - Normalize GUIDs, hexadecimal addresses, and long record-like numeric identifiers.
   - Preserve short numeric values because they may be meaningful error codes, HTTP statuses, or SQL codes.
   - Retain `ErrorCategory` as a separate grouping dimension.
   - Show one representative message and the normalization rule used.

7. **Analyze the time trend.**
   - Select a bin size appropriate to the window.
   - Fill empty time buckets with zero before calculating the median.
   - Exclude partial first and last buckets from the median baseline and identify them in the output.
   - Treat `max(5 events, 3 x median)` as a triage heuristic, not a statistical proof of anomaly.
   - Use the line-chart tool when there are enough buckets to show a meaningful trend.

8. **Quantify blast radius.**
   - For each top signature, report error events and distinct executions, entities, projects, legal entities, and nodes.
   - Distinguish a repeated error in one execution from the same signature across many executions.

9. **Determine execution outcomes.**
   - Query the related non-error DMF events for the affected execution IDs.
   - Pad the outcome query beyond the error-analysis window so executions that finish later are not misclassified.
   - Report the separate outcome-query window and its observation cutoff.
   - Classify each execution as completed, partial success, failed, unfinished, or indeterminate.
   - Use the latest non-null status and count values and identify their source event.

10. **Calculate error rates only with a valid denominator.**
    - Prefer terminal per-execution `StagingErrorCount`, `TargetErrorCount`, and `TotalNumberOfRecords`.
    - Do not sum repeated cumulative counts across events.
    - Do not divide raw `DMFError` event count by record volume unless one-event-per-record semantics are verified.
    - Report staging and target error rates separately by default. Combine them only when telemetry semantics prove that their populations do not overlap.
    - If no trustworthy denominator exists, report counts only and state that an error rate is unavailable.

11. **Investigate likely causes.**
    - Load `references/interpretation.md`.
    - Correlate spikes with deployment, batch, dependency, SQL, AOS restart, or resource-pressure telemetry only when available.
    - Label findings as `FACT`, `CORRELATED`, or `HYPOTHESIS`.
    - Assign `HIGH`, `MEDIUM`, or `LOW` confidence using the documented criteria.

12. **Provide drilldown evidence.**
    - Return a newest-first detail table capped at 1,000 rows.
    - State whether the result was truncated.
    - Redact secrets, tokens, personal data, and source-record payload values from messages.
    - Recommend `d365-dmf-execution-drilldown-v2` for the most representative or highest-impact execution.

13. **Render the result** using `assets/report-template.md`.

## Guardrails

- Never run an unbounded telemetry query.
- Never conclude that no errors exist until the event schema and query window are verified.
- Never equate an error event with a failed execution or failed record without corroboration.
- Never normalize away short numeric error codes.
- Never claim causation from temporal overlap alone.
- Preserve exact counts, IDs, timestamps, categories, and statuses in evidence tables.
- Treat null values as unavailable, not zero.
- Report sampling, retention limits, query errors, row limits, and truncation.
- Redact sensitive values before presenting or persisting error details.
- Use UTC unless the user requests another time zone; state any conversion.

## Completion criteria

The analysis is complete only when it includes:

- Source, time window, filters, and result limits.
- Overall impact and blast radius.
- Ranked projects, entities, and recurring signatures.
- A zero-filled trend and clearly labeled anomaly heuristic.
- Execution-outcome evidence for the highest-impact failures.
- Error rates or an explicit explanation of why they cannot be calculated.
- Facts, correlations, hypotheses, confidence, and prioritized next actions.