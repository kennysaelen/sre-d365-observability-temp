# D365 DMF Error Root-Cause Analysis

Human-facing documentation for the Azure SRE Agent skill
`d365-dmf-error-rca-v2`.

This skill analyzes Dynamics 365 Finance and Operations Data Management
Framework (DMF) errors over a bounded period. It identifies recurring error
signatures, affected executions and workloads, blast radius, trends, execution
outcomes, and evidence-supported likely causes.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill for:

- DMF import or export error spikes.
- Recurring error signatures.
- Errors affecting multiple executions, entities, projects, or legal entities.
- Distinguishing isolated data errors from broader systemic patterns.
- Error-rate analysis when reliable terminal record counts are available.

Use another skill when the scope is different:

| Investigation | Skill |
|---|---|
| One known execution | `d365-dmf-execution-drilldown-v2` |
| Runs with no terminal event | `d365-dmf-unfinished-runs-v2` |
| Environment-wide DMF overview | `d365-dmf-monitoring-dashboard-v2` |

## Prerequisites

The Azure SRE Agent space must provide one supported Application Insights, Log
Analytics, Azure resource, or ADX query tool declared in `SKILL.md`. Confirm the
exact tool identifiers against the target space before deployment.

The request should provide:

- A telemetry source.
- A bounded UTC start and end time.
- One exact environment ID when the source contains multiple environments.
- Optional entity, data project, execution, activity, or legal-entity filters.

## Example prompts

```text
Analyze DMF errors for environment <environment-id> over the last 24 hours.
Group recurring signatures, quantify affected executions and entities, verify
execution outcomes, and identify the highest-confidence likely causes.
```

```text
Investigate the DMF error spike for data project <project-name> between
2026-09-13T08:00:00Z and 2026-09-13T14:00:00Z. Include blast radius, trend,
representative executions, and safe next actions.
```

## Expected output

1. Query source, window, filters, limits, and coverage.
2. Error events and distinct affected executions.
3. Ranked projects, entities, legal entities, and recurring signatures.
4. A zero-filled error trend with partial-bin handling.
5. Blast radius for the most important signatures.
6. Verified execution outcomes for representative failures.
7. Error rates when a trustworthy denominator exists, or an explicit reason
   why rates are unavailable.
8. `FACT`, `CORRELATED`, and `HYPOTHESIS` findings with confidence and actions.

## Package layout

```text
d365-dmf-error-rca-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## Important analysis rules

- `DMFError` is an error event, not automatically a failed execution or failed
  record.
- Verify outcomes with terminal events, statuses, and counts.
- Use exact, case-insensitive matching for identifier-like filters.
- Normalize volatile signature values without removing meaningful short error
  codes.
- Do not sum repeated cumulative record or error counts.
- Do not calculate an error rate from raw event count unless one-event-per-
  failed-record semantics are proven.
- Redact secrets, personal data, source values, paths, tokens, and payloads.
- Treat temporal alignment as correlation, not proof of causation.

## Testing and review checklist

- Confirm folder and `SKILL.md` names are aligned.
- Confirm every declared tool exists in the target Azure SRE space.
- Validate the schema against both `customEvents` and `AppEvents` variants.
- Test optional filters set to values and to empty strings.
- Test repeated errors in one execution versus one signature across many runs.
- Test later successful completion after an earlier error.
- Test missing terminal counts and overlapping staging/target populations.
- Confirm empty trend buckets are zero-filled and partial bins are excluded
  from the baseline.
- Confirm report details are redacted and row-limit truncation is disclosed.

## Maintenance guidance

Keep operational behavior in `SKILL.md`, KQL in `references/queries.md`,
interpretation rules in `references/interpretation.md`, and presentation
structure in `assets/report-template.md`.
