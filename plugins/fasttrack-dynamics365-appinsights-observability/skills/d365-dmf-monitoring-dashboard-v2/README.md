# D365 DMF Monitoring Dashboard

Human-facing documentation for the Azure SRE Agent skill
`d365-dmf-monitoring-dashboard-v2`.

This skill creates a bounded health overview of Dynamics 365 Finance and
Operations Data Management Framework (DMF) telemetry. It summarizes completed
executions, processed-record volume, duration, errors, unfinished runs, and
telemetry coverage, then routes detailed findings to the appropriate focused
DMF skill.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill for:

- Environment-wide DMF health checks.
- Daily or periodic DMF operational digests.
- Import versus export workload summaries.
- Completed execution and record-volume reporting.
- High-level duration, error, and unfinished-run signals.
- Identifying which focused DMF skill should investigate a finding.

Use a focused skill instead when the request is already limited to one problem:

| Investigation | Skill |
|---|---|
| Duration or performance regression | `d365-dmf-execution-duration-analysis-v2` |
| Error root-cause analysis | `d365-dmf-error-rca-v2` |
| Unfinished or apparently stuck runs | `d365-dmf-unfinished-runs-v2` |
| Throughput or record volume | `d365-dmf-throughput-volume-analysis-v2` |
| One execution | `d365-dmf-execution-drilldown-v2` |

## Prerequisites

The Azure SRE Agent space must provide a supported telemetry query tool for the
target source. The skill currently declares these tool identifiers:

- `kusto_query`
- `QueryAppInsightsByAppId`
- `QueryAppInsightsByResourceId`
- `QueryLogAnalyticsByResourceId`
- `QueryLogAnalyticsByWorkspaceId`
- `ValidateQuery`
- Optional chart tools listed in `SKILL.md`

Confirm the exact identifiers against the target Azure SRE space's tool
catalog before deployment.

The request should provide:

- A telemetry source, such as an Application Insights App ID, Azure resource
  ID, Log Analytics workspace ID, or supported ADX target.
- A bounded UTC start and end time.
- One exact environment ID when the source contains multiple environments.
- Optional entity, data project, execution, activity, legal-entity, or
  direction filters.

## Example prompts

```text
Show DMF health for environment <environment-id> during the last 24 hours.
Include completed executions, processed records, errors, unfinished runs,
slowest executions, and telemetry coverage.
```

```text
Create a DMF daily digest for environment <environment-id> from
2026-09-13T00:00:00Z through 2026-09-14T00:00:00Z. Break down imports and
exports by entity and data project, and identify findings that need focused
follow-up.
```

```text
Review DMF activity for data project <project-name> in environment
<environment-id> over the past seven days. Highlight volume, duration, errors,
unfinished runs, and data-quality limitations.
```

## Expected output

The skill produces:

1. Scope, source, time window, filters, and ingestion-lag information.
2. Telemetry coverage and data-quality limitations.
3. Reconciled headline KPIs:
   - Completed executions.
   - Completed records.
   - Error events and affected executions.
   - Unfinished runs.
4. Direction, entity, project, volume, duration, error, and unfinished-run
   insights.
5. Evidence-labeled findings with confidence and severity.
6. Handoffs to focused DMF skills where deeper analysis is required.

Completed-record totals are labeled as a known-record lower bound when one or
more execution/entity end rows lack a final record count.

## Package layout

```text
d365-dmf-monitoring-dashboard-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- findings-contract.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

| File | Purpose |
|---|---|
| `README.md` | Human-facing overview, installation context, and usage examples. |
| `SKILL.md` | Agent activation criteria, workflow, guardrails, tools, and completion criteria. |
| `references/queries.md` | Reusable KQL datasets and query patterns. |
| `references/interpretation.md` | Evidence, confidence, and severity guidance. |
| `assets/findings-contract.md` | Host-agnostic structured output contract. |

## Important analysis rules

- Always use a bounded query window.
- Never combine environments when pairing execution IDs.
- Treat only DMF import/export job-end events as terminal completion events.
- Treat `DMFError` as an error signal, not proof that an execution failed or
  completed.
- Select a coherent latest end row before aggregating final record counts.
- Preserve execution/entity grain for record volume.
- Exclude incomplete, repeated, or reversed start/end pairs from duration
  metrics.
- Report missing identifiers, missing counts, duplicates, unknown event names,
  sampling, and truncation.
- Do not emit raw error messages, custom dimensions, user IDs, tenant IDs,
  source payloads, or file paths.

## Testing and review checklist

Before publishing a revision:

- Confirm the folder name matches the `name` value in `SKILL.md`.
- Confirm every declared tool exists in the target Azure SRE space.
- Replace all query placeholders before validation or execution.
- Validate representative KQL through the available query-validation tool.
- Test single-entity and multi-entity executions.
- Test missing record counts and duplicate end events.
- Test an execution without a start event in the pairing lookback.
- Test a historical window so unfinished-run silence uses the historical
  observation cutoff rather than the current time.
- Confirm error output contains only approved type/signature values.
- Confirm the rendered output preserves the fields defined in
  `assets/findings-contract.md`.

## Maintenance guidance

Keep agent behavior in `SKILL.md` and detailed KQL in `references/queries.md`.
Update this README only when human-facing usage, prerequisites, package layout,
or maintenance procedures change. This avoids maintaining the same operational
instructions in multiple files.
