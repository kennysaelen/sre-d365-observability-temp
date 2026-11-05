# D365 DMF Throughput and Volume Analysis

Human-facing documentation for the Azure SRE Agent skill
`d365-dmf-throughput-volume-analysis-v2`.

This skill measures completed DMF record volume, execution size, records per
wall-clock hour, comparable-window volume changes, and execution-level
processing efficiency.

> `README.md` is for people who install, review, test, or maintain the skill.
> The agent's authoritative operating instructions remain in `SKILL.md`.

## When to use it

Use this skill to answer:

- How many records completed?
- Which entity, project, or direction dominates volume?
- Which executions processed the most records?
- How did the records-per-hour trend change?
- Which workloads are new, disappeared, or materially changed?
- Which high-volume executions have poor records-per-second efficiency?

Use `d365-dmf-execution-duration-analysis-v2` for duration regressions and
`d365-dmf-execution-drilldown-v2` for one execution.

## Prerequisites

Provide a telemetry source, bounded UTC window, and one exact environment ID
when the source contains multiple environments. Optional filters include
entity, project, direction, execution, and legal entity.

Confirm the query, validation, and chart tools declared in `SKILL.md` exist in
the target Azure SRE space.

## Example prompts

```text
Analyze completed DMF volume for environment <environment-id> over the last
seven days. Show record totals by entity and project, largest executions, and
the hourly throughput trend.
```

```text
Compare current and previous seven-day DMF volume for environment
<environment-id>. Classify new, disappeared, comparable, and unknown-coverage
workloads using a 1,000-record materiality floor.
```

## Expected output

1. Scope, time windows, filters, thresholds, and count grain.
2. Completed record volume by entity, project, and direction.
3. Missing, zero, duplicate, and conflicting-count coverage.
4. Execution-size distribution and largest execution/entity rows.
5. Zero-filled records-per-hour trend with complete-bin handling.
6. Equal-window volume comparison and materiality assessment.
7. Processing efficiency for executions with valid complete pairs and counts.
8. Evidence labels, confidence, and next actions.

## Package layout

```text
d365-dmf-throughput-volume-analysis-v2/
|-- README.md
|-- SKILL.md
|-- assets/
|   `-- report-template.md
`-- references/
    |-- interpretation.md
    `-- queries.md
```

## Important analysis rules

- Use only import/export job-end events for completed record volume.
- Select one coherent latest end row at execution/entity grain.
- Never sum repeated raw `TotalNumberOfRecords` values.
- Keep entity-level volume filters at entity-row grain.
- Use execution-wide counts only for execution-wide efficiency.
- Distinguish volume from throughput:
  - Volume is completed records in a window.
  - Operational throughput is records per wall-clock hour or bin.
  - Processing efficiency is records per execution second.
- Zero-fill absent trend bins and exclude partial bins from rate comparison.
- Compare disjoint equal-length windows.
- Use `UnknownCoverage` when missing record counts prevent reliable
  new/disappeared classification.
- Do not interpret volume growth as inherently positive or negative.

## Testing and review checklist

- Confirm every declared tool exists in the target Azure SRE space.
- Test duplicate end events with both matching and conflicting record counts.
- Test one execution containing multiple entities.
- Test entity-row filtering without including unrelated entities.
- Test project and direction execution-level filtering.
- Test missing and zero record counts.
- Test a workload present in only the current or prior window.
- Test reused execution IDs in both comparison windows.
- Test aligned and partial hourly bins.
- Test efficiency with incomplete duration pairs and partial record coverage.

## Maintenance guidance

Keep workflow and metric definitions in `SKILL.md`, KQL in
`references/queries.md`, interpretation guidance in
`references/interpretation.md`, and presentation structure in
`assets/report-template.md`.
