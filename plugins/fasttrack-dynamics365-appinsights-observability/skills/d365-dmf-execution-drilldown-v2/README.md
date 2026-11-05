# D365 DMF Execution Drilldown

Human-facing documentation for the execution-drilldown package in
`d365-dmf-execution-drilldown-v2`.

This skill reconstructs one Dynamics 365 Finance and Operations DMF execution
from telemetry. It produces a chronological event timeline, event-to-event
gaps, metadata, record counts, statuses, errors, duration, outcome, and an
evidence-labeled root-cause assessment.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill when a DMF `ExecutionId` or telemetry `ActivityId` is already
known and the user needs to understand one execution.

Use a discovery skill first when the identifier is unknown:

| Investigation | Skill |
|---|---|
| Environment-wide discovery | `d365-dmf-monitoring-dashboard-v2` |
| Error-driven discovery | `d365-dmf-error-rca-v2` |
| Unfinished-run discovery | `d365-dmf-unfinished-runs-v2` |
| Duration ranking | `d365-dmf-execution-duration-analysis-v2` |

## Prerequisites

The request must provide:

- A DMF `ExecutionId` or telemetry `ActivityId`.
- An Application Insights application, Log Analytics workspace, Azure
  resource, or supported ADX target.
- A time hint or bounded incident window when available.
- One exact environment ID when the source contains multiple environments.

Confirm the query and validation tools declared in `SKILL.md` exist in the
target Azure SRE space.

## Example prompts

```text
Reconstruct DMF execution <execution-id> in environment <environment-id>.
Show the complete timeline, event gaps, record counts, errors, duration,
outcome, and evidence-supported root cause.
```

```text
Investigate telemetry activity <activity-id> around
2026-09-13T10:00:00Z. Determine whether it maps to one or multiple DMF
executions and analyze each exact match separately.
```

## Expected output

1. Telemetry source, searched window, and exact identifier mapping.
2. Chronological DMF timeline with the gap to the next event.
3. Data project, entity, legal entity, execution mode, batch, and role-instance
   context.
4. Latest non-null record counts, error counts, and statuses with source events.
5. Completed, partial-success, failed, unfinished, or indeterminate outcome.
6. Duration or a clearly labeled lower bound.
7. Root-cause findings separated into facts, correlations, and hypotheses.
8. Confidence and a specific next action.

## Package layout

```text
d365-dmf-execution-drilldown-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## Important analysis rules

- Never run an unrestricted environment-wide query from this skill.
- Verify the telemetry schema and identifier field before declaring no match.
- Never treat an empty identifier as a wildcard.
- Do not substitute a similar execution when no exact match exists.
- Calculate event gaps independently for each execution.
- A large telemetry gap identifies where elapsed time accumulated; it does not
  prove SQL blocking, resource pressure, or dependency delay.
- Verify success or failure with terminal events, statuses, and counts.
- Use the latest event carrying a non-null status or count and identify it.
- Preserve exact evidence while redacting sensitive payload values.

## Testing and review checklist

- Confirm the folder name and intended deployed skill identifier.
- Confirm every declared tool exists in the target Azure SRE space.
- Test exact `ExecutionId`, exact `ActivityId`, and `operation_Id` fallback.
- Test one activity mapping to multiple execution IDs.
- Test no-match behavior and progressive window widening.
- Test completed, partial-success, failed, unfinished, and contradictory cases.
- Test missing start or terminal events and lower-bound duration labeling.
- Confirm timeline ordering and terminal-row gap handling.
- Confirm all query placeholders are replaced before validation.

## Maintenance guidance

Keep activation and workflow rules in `SKILL.md`, query patterns in
`references/queries.md`, confidence guidance in
`references/interpretation.md`, and report structure in
`assets/report-template.md`.
