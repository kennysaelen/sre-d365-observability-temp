# D365 DMF Execution-Duration Analysis

Human-facing documentation for the Azure SRE Agent skill
`d365-dmf-execution-duration-analysis-v2`.

This skill identifies slow or degraded DMF imports and exports through explicit
start/end pairing, duration distributions, final record counts, per-record
efficiency, and disjoint current-versus-prior comparisons.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill for:

- Ranking slow completed DMF executions.
- Duration comparisons by entity or data project.
- Milliseconds per record and records-per-second efficiency.
- Current-versus-prior duration regressions.
- Separating high-volume long runs from inefficient processing.

| Different investigation | Skill |
|---|---|
| Run has no terminal event | `d365-dmf-unfinished-runs-v2` |
| One execution needs RCA | `d365-dmf-execution-drilldown-v2` |
| Record volume rather than duration | `d365-dmf-throughput-volume-analysis-v2` |

## Prerequisites

Provide a telemetry source and bounded UTC analysis window. An exact environment
ID is mandatory when the source contains multiple environments. Optional
filters include entity, data project, direction, execution ID, activity ID, and
legal entity.

Confirm the query, validation, and optional chart tools declared in `SKILL.md`
exist in the target Azure SRE space.

## Example prompts

```text
Find the slowest completed DMF executions in environment <environment-id>
during the last seven days. Compare duration with record volume and identify
executions with poor per-record efficiency.
```

```text
Compare DMF duration for data project <project-name> between the current and
previous seven-day periods. Show median, p95, sample counts, and material
regressions.
```

## Expected output

1. Source, analysis window, pairing buffer, filters, and cohort rule.
2. Counts of complete, incomplete, truncated, retried, and ambiguous pairs.
3. Slowest valid executions with duration and record efficiency.
4. Cumulative duration and distributions by entity or project.
5. Equal-window median and p95 comparison with sample counts.
6. Coverage limitations, evidence labels, confidence, and next actions.

## Package layout

```text
d365-dmf-execution-duration-analysis-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## Important analysis rules

- Pair start and end events explicitly by normalized execution ID.
- Use a bounded pairing buffer for executions crossing the requested window.
- Deduplicate final counts at execution/entity grain.
- Treat multiple entities in one execution as valid.
- Exclude repeated starts, duplicate entity ends, reversed timestamps, and
  incomplete pairs from duration aggregates.
- Do not convert incomplete duration to zero.
- Calculate execution efficiency only when every entity row has a final count.
- Exclude missing/zero counts and durations shorter than five seconds from
  rate rankings.
- Compare disjoint equal-length windows and require sufficient samples.
- State that cumulative execution duration can exceed wall-clock time because
  executions overlap.

## Testing and review checklist

- Confirm every declared tool exists in the target Azure SRE space.
- Test pairs crossing both analysis-window boundaries.
- Test multi-entity executions and summed final counts.
- Test repeated starts, duplicate ends, reversed timestamps, and reused IDs.
- Test missing counts on all entities and on only one entity.
- Test entity, project, activity, execution, direction, and legal-entity filters.
- Test current/prior windows with at least and fewer than five samples.
- Confirm top-N limits are applied after full-population aggregation.
- Confirm incomplete and truncated details are reported separately.

## Maintenance guidance

Keep pairing behavior in `SKILL.md`, executable KQL in
`references/queries.md`, severity and confidence guidance in
`references/interpretation.md`, and output structure in
`assets/report-template.md`.
