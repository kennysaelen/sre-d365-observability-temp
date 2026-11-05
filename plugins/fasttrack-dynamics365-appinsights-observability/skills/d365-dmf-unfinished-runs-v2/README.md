# D365 DMF Unfinished-Run Triage

Human-facing documentation for the Azure SRE Agent skill
`d365-dmf-unfinished-runs-v2`.

This skill detects DMF executions with a start event but no matching terminal
event and distinguishes recent activity, silence, errors, telemetry ambiguity,
and corroborated process interruption.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill when a DMF import or export appears:

- Stuck or unexpectedly long-running.
- Abandoned or interrupted.
- Started but never completed.
- Erroring without a terminal event.
- Present in an environment-wide unfinished-run summary.

An unfinished telemetry sequence is not automatically a hung process. Use
`d365-dmf-execution-drilldown-v2` when one execution needs detailed RCA.

## Prerequisites

Provide:

- A telemetry source.
- One exact environment ID when the source contains multiple environments.
- A configurable start-event lookback, default seven days.
- An ingestion/observation lag, default five minutes.
- Optional entity, project, direction, execution, or legal-entity filters.

Confirm the query and validation tools declared in `SKILL.md` exist in the
target Azure SRE space.

## Example prompts

```text
Find unfinished DMF executions in environment <environment-id> over the last
seven days. Use a five-minute ingestion lag, classify state and severity, and
show silence, errors, batch context, and node context.
```

```text
Investigate whether DMF execution <execution-id> is still active, silent,
errored without completion, interrupted by an AOS restart, or indeterminate.
Do not cancel or rerun it.
```

## Expected output

1. Source, lookback, query time, observation cutoff, lag, and filters.
2. Missing-identifier and lookback-boundary coverage.
3. Total unfinished count and state/severity summary.
4. Risk-ordered execution details with elapsed and silence time.
5. Progress, errors, batch job, direction, and role-instance context.
6. Restart/crash and batch-state correlation when telemetry is available.
7. Evidence labels, confidence, and safe next actions.

## Package layout

```text
d365-dmf-unfinished-runs-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## State model

| State | Meaning |
|---|---|
| `Active` | No terminal event and telemetry remains recent. |
| `Silent` | No terminal event and no recent telemetry beyond the warning threshold. |
| `ErroredNoEnd` | One or more error events occurred after the selected start, with no matching end. |
| `ProcessInterrupted` | Matching-node restart or crash evidence corroborates interruption. |
| `Indeterminate` | Retention, identifiers, retries, or contradictory evidence prevent a reliable conclusion. |

## Important analysis rules

- Only matching import/export job-end events are terminal.
- `DMFError` is not terminal.
- Select the latest relevant start and require a later matching end.
- Treat repeated starts as ambiguous unless a retry discriminator exists.
- Restrict latest-event and error enrichment to events after the selected start.
- Apply ingestion lag before classifying recent starts.
- Use silence as the primary current-health signal; elapsed time alone is not
  proof of a hang.
- Require matching normalized role instance and valid timing for restart/crash
  correlation.
- Keep the skill read-only; do not cancel, restart, or rerun executions.

## Testing and review checklist

- Confirm every declared tool exists in the target Azure SRE space.
- Test recent starts inside and outside the ingestion-lag cutoff.
- Test completed, active, silent, errored-no-end, and indeterminate sequences.
- Test repeated starts and an older end followed by a newer retry.
- Test starts near the lookback boundary.
- Test missing execution IDs and missing metadata.
- Test role-instance normalization and restart timing.
- Test empty batch job ID as a possible interactive execution.
- Confirm risk ordering occurs before the detail limit.
- Confirm no action mutates or reruns a DMF job.

## Maintenance guidance

Keep state and safety behavior in `SKILL.md`, detection and correlation KQL in
`references/queries.md`, evidence guidance in
`references/interpretation.md`, and presentation structure in
`assets/report-template.md`.
