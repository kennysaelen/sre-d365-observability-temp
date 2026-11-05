---
name: d365-dmf-unfinished-runs-v2
description: Detects and triages Dynamics 365 Finance and Operations Data Management Framework (DMF) executions with a start event but no matching end event. Use for apparently stuck, hung, abandoned, interrupted, or long-running imports and exports, including silence-duration, error, batch, and AOS restart correlation.
tools:
  - kusto_query
  - QueryAppInsightsByAppId
  - QueryAppInsightsByResourceId
  - QueryLogAnalyticsByResourceId
  - QueryLogAnalyticsByWorkspaceId
  - ValidateQuery
---

# D365 DMF unfinished-run triage

## Use this skill when

Use this skill when a DMF execution appears stuck, never completed, or has been running unexpectedly long, or as the unfinished-run component of a DMF health check.

An unfinished telemetry sequence is not automatically a hung process. It can represent a healthy long-running execution, ingestion delay, missing telemetry, process interruption, or an execution that started before the selected lookback.

## Required scope

Obtain the telemetry source and environment scope. Use a configurable start-event lookback, default seven days, and an observation lag, default five minutes. Optional filters include entity, project, direction, execution, and legal entity.

If the source contains more than one environment, require one exact environment ID before correlating execution IDs.

## Workflow

1. Choose the narrowest query tool for the supplied resource identifier.
2. Confirm the DMF schema and identifier coverage with `references/queries.md`.
3. Build the bounded `Unfinished` dataset.
   - Scan starts and all related DMF events over the same lookback.
   - Scan terminal events through the current query time.
   - Normalize and exclude empty execution IDs; report excluded-event coverage.
   - Treat only matching `JobEnd` as terminal.
   - Keep `DMFError` as an error signal; an error does not prove execution completion.
   - Replace every unused query placeholder with `""`; never submit a query containing `{{` or `}}`.
4. Apply ingestion-lag protection.
   - Set `_queryNow=now()` and `_cutoff=_queryNow-_ingestionLag`.
   - Do not classify starts newer than `_cutoff`.
   - Report the query time, cutoff, lag, and lookback.
5. Measure both:
   - `ElapsedTime`: cutoff minus start time.
   - `SilenceTime`: cutoff minus the latest event for the execution.
6. Classify state.
   - `Active`: no recent telemetry gap and no end; distinguish start-only from observed progress in the output.
   - `Silent`: no recent telemetry beyond the warning threshold.
   - `ErroredNoEnd`: one or more errors and no end.
   - `ProcessInterrupted`: corroborated AOS/batch restart or crash after the start/latest event.
   - `Indeterminate`: incomplete identifiers, retention boundary, or contradictory evidence.
7. Assign severity using silence and historical duration context.
   - Default silence thresholds: under 15 minutes `info`, 15–60 minutes `warning`, over 60 minutes `critical`.
   - If enough history exists, compare elapsed time with completed-run p95 for the same entity/project.
   - A long elapsed time with recent progress is not automatically critical.
8. Enrich each row with start metadata, last event, error count, batch job ID/caption, role instance, and direction.
9. Correlate carefully.
   - A restart/crash must occur after the start and at or after the latest execution event to support interruption.
   - On accepted matching-node restart/crash evidence, reclassify the row as `ProcessInterrupted`.
   - Use `Indeterminate` when boundary, identifier, or retention coverage prevents a reliable state.
   - Empty `BatchJobId` can indicate an interactive execution and is not itself anomalous.
   - Use batch status as independent confirmation when available.
10. Remain read-only.
    - Do not cancel, restart, or rerun an execution from this skill.
    - Suggest cancellation only after current batch state, silence, and partial-commit risk are verified and a human explicitly approves.
11. Apply `FACT`, `CORRELATED`, and `HYPOTHESIS` labels from `references/interpretation.md`.
12. Render using `assets/report-template.md`.

## Guardrails

- Never use `DMFError` as a terminal event.
- Never classify from elapsed time alone.
- Never use a start-event window narrower than the terminal/latest-event window.
- Never include empty execution IDs in joins.
- Never use a fixed historical end time when determining current state.
- Never return an arbitrary `limit`; order highest-risk rows first and disclose truncation.
- Never recommend rerun without addressing duplicate and partial-commit risk.
- Report source, lookback, lag, cutoff, coverage, filters, and unavailable correlations.

## Completion criteria

Include:

- Query provenance and identifier coverage.
- Total unfinished count and severity/state summary.
- Ordered unfinished-run table with elapsed and silence time.
- Errors, batch context, and node context.
- Restart/crash and batch-status correlation when available.
- A clean no-findings statement when appropriate.
- Evidence labels, confidence, and safe next actions.