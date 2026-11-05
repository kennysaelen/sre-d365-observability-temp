---
name: d365-dmf-execution-drilldown-v2
description: Reconstructs and analyzes one Dynamics 365 Finance and Operations Data Management Framework (DMF) execution from Application Insights or Log Analytics telemetry. Use when an ExecutionId or ActivityId is known and the user needs an event timeline, duration, record counts, errors, legal-entity and batch context, unfinished-run analysis, or evidence-based root-cause findings.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
---

# D365 DMF execution drilldown

## Use this skill when

Use this skill for a **single DMF execution** when at least one of these identifiers is known:

- DMF `ExecutionId`
- Telemetry `ActivityId`

Typical requests include:

- Why did this import or export fail?
- Reconstruct the complete timeline for this execution.
- Which phase was slow?
- How many records reached staging and target?
- Is the execution completed, partially successful, failed, or unfinished?

Do not use this skill for environment-wide DMF discovery, trend analysis, or finding an unknown execution. Use the appropriate DMF discovery, error-RCA, or unfinished-runs skill first, then return with a candidate identifier.

## Required inputs

Obtain:

1. An `ExecutionId` or `ActivityId`.
2. The Application Insights application, Log Analytics workspace, or Azure resource to query.
3. A time hint if available.

If the identifier is missing, ask for it or route to a discovery skill. Never run an unrestricted environment-wide query from this skill.

## Workflow

1. **Choose the narrowest query tool.**
   - Use the App ID or workspace ID tool when that identifier is known.
   - Otherwise, use the corresponding resource ID tool.
   - Use `kusto_query` only when the telemetry source is an Azure Data Explorer database rather than Azure Monitor.

2. **Establish a bounded search window.**
   - Start with the supplied incident window.
   - If only an identifier is known, start with a 48-hour window around the best available time hint.
   - Widen progressively only when no matching event is found.
   - Define `_startTime` and `_endTime` explicitly in every submitted query.
   - Use `_endTime` as the observation cutoff for unfinished-run duration.
   - State the final queried window and telemetry source in the result.

3. **Confirm the telemetry schema and identifier field.**
   - Run the bounded schema probe in `references/queries.md`.
   - Confirm the event table, property-bag column, timestamp column, role-instance column, and identifier casing.
   - Check both `activityId` and `ActivityId`; Application Insights may instead carry the correlation in `operation_Id`.
   - Do not interpret zero rows as "execution not found" until the schema and identifier field have been verified.

4. **Validate and run the primary timeline query.**
   - Load and adapt `references/queries.md`.
   - Use `ValidateQuery` when the selected query surface supports it.
   - Filter by the supplied identifier before projecting or sorting the full event set.
   - Do not treat an empty identifier as a wildcard.
   - Replace every template value before execution; never submit a query containing `{{` or `}}`.

5. **Verify identifier integrity.**
   - Confirm that returned events belong to one logical execution.
   - If one `ActivityId` maps to multiple execution IDs, separate them and explain the relationship.
   - If no exact match exists, report that result and the searched window; do not substitute a similar execution.

6. **Reconstruct the event sequence.**
   - Sort events ascending by timestamp.
   - Identify start, progress/staging, target, error, and terminal events from the observed event names.
   - Calculate the gap from each event to the next event.
   - Calculate gaps independently for each execution ID; the terminal row has no next-event gap.
   - Treat a large telemetry gap as the location of elapsed time, not proof of the underlying cause.

7. **Extract execution metadata.**
   - Data project, entity, legal entity, and execution mode.
   - Batch job ID and caption.
   - AOS role instance.
   - Staging and target record counts, error counts, and statuses.
   - Error message and staging record ID when present.
   - For each count or status, use the latest event carrying a non-null value and identify that source event. Do not choose an arbitrary first or last row.

8. **Classify the outcome.**
   - **Completed:** a terminal success event is present and statuses show success.
   - **Partial success:** records succeeded but staging or target errors are nonzero, or statuses indicate partial completion.
   - **Failed:** a terminal failure/error is present or no records succeeded after a fatal error.
   - **Unfinished:** a start event exists without a terminal event by the observation cutoff.
   - **Indeterminate:** telemetry is incomplete or contradictory.

9. **Calculate duration.**
   - Completed execution: terminal timestamp minus start timestamp.
   - Unfinished execution: observation cutoff minus start timestamp.
   - If start or terminal telemetry is missing, label the duration as a lower bound or unavailable.

10. **Apply evidence and confidence labels.**
   - Load `references/interpretation.md`.
   - Label each root-cause statement as `FACT`, `CORRELATED`, or `HYPOTHESIS`.
   - Assign `HIGH`, `MEDIUM`, or `LOW` confidence using the documented criteria.

11. **Investigate likely contention only when evidence warrants it.**
   - Query overlapping executions on the same role instance, entity, or data project.
   - Correlate with batch, SQL, dependency, or platform telemetry when those tools and data are available.

12. **Render the result** using `assets/report-template.md`.

## Guardrails

- Never expose an unbounded telemetry query.
- Never infer success solely from the presence of a terminal event; verify statuses and counts.
- Never infer failure solely from an error event; later events may show retry or partial success.
- Never claim that a telemetry gap proves SQL blocking, resource starvation, or an external dependency delay without corroborating evidence.
- Preserve exact IDs, timestamps, event names, counts, and status values in the evidence tables.
- Identify null or missing fields as unavailable rather than converting them to zero.
- Use UTC timestamps unless the user requests a local time zone; state any conversion explicitly.
- Report tool errors, truncation, sampling, retention limits, and incomplete time coverage.

## Completion criteria

The analysis is complete only when it contains:

- The queried source and time window.
- The exact identifier or identifier mapping used.
- A chronological timeline with event-to-event gaps.
- Metadata, record counts, errors, and statuses.
- Duration and outcome classification.
- A root-cause assessment that separates facts from hypotheses.
- Confidence and a specific next action.