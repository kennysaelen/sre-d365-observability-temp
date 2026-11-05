# Batch throttling findings contract

Follow the DMF monitoring overview's host-agnostic structure: `metadata`,
`coverage`, `summary_kpis`, `insights`, and `findings`. Build this contract
internally, then respond in **Markdown by default**. Do not return a JSON dump
unless requested. Charts are optional presentation aids, not required evidence.
No file generation, browser, JavaScript, or HTML template is required.

The skeleton below defines fields, not observed results. Null means unavailable
or not yet queried; never substitute zero for a failed or missing query.

```json
{
  "metadata": {
    "title": "D365 batch throttling analysis",
    "source": "",
    "source_table": "",
    "environment": "",
    "time_range_utc": {"start": "", "end": ""},
    "bin_minutes": 10,
    "ingestion_lag_minutes": 5,
    "filters": {"role_instance": ""},
    "metric_units": {"cpu": "unverified", "memory": "unverified", "sql_dtu": "unverified"},
    "generated_utc": "",
    "result_status": "not_run"
  },
  "coverage": {
    "environment_rows_missing_id": null,
    "environment_rows_with_conflicting_ids": null,
    "event_coverage": [],
    "metric_coverage": null,
    "query_errors": [],
    "limitations": []
  },
  "summary_kpis": [
    {"id": "task_start_events", "value": null, "severity": "info"},
    {"id": "throttle_events", "value": null, "severity": "info"},
    {"id": "affected_aos", "value": null, "severity": "info"}
  ],
  "insights": [
    {
      "id": "throttling_timeline",
      "title": "Running tasks and throttled tasks",
      "intent": "Show observed task-start and throttle-event counts in common UTC bins.",
      "status": "not_run",
      "rows": [],
      "displayed_rows": 0,
      "presentation_truncated": false
    },
    {
      "id": "aos_resource_context",
      "title": "Throttling by AOS and resource context",
      "intent": "Show concentration, metric coverage, and per-resource peaks without inferring a trigger.",
      "status": "not_run",
      "rows": []
    },
    {
      "id": "throttle_events",
      "title": "Throttle-event details",
      "intent": "Show all three system metrics from each event with identifiers and availability status.",
      "status": "not_run",
      "rows": [],
      "total_rows": null,
      "returned_rows": 0,
      "displayed_rows": 0,
      "query_row_limit": 500,
      "data_truncated": false,
      "presentation_truncated": false
    }
  ],
  "findings": []
}
```

## Contract rules

All insights require `id`, `title`, `intent`, `status`, and `rows`.
Status values are `not_run`, `complete`, `partial`, `unavailable`, and
`not_applicable`; metadata uses the same values except `not_applicable`.
Complete means the analysis finished, not that the workload is healthy.

KPI values and coverage counts are null or nonnegative integers. Allowed
severities match the DMF contract: `info`, `success`, `warning`, `critical`.
Use success only with evidence, never solely because no events were observed.

Each finding uses:

```json
{
  "statement": "",
  "evidence_label": "FACT",
  "confidence": "HIGH",
  "severity": "info",
  "evidence": [],
  "next_action": "",
  "handoff_skill": ""
}
```

Evidence labels: `FACT`, `CORRELATED`, `HYPOTHESIS`. Confidence: `HIGH`,
`MEDIUM`, `LOW`. Populate concrete evidence and an actionable next step;
`handoff_skill` may be empty. Do not create empty placeholder findings in an
actual response.

## Query-result mapping

Preserve query column names inside each insight's `rows`. Normalize only the
contract's outer metadata and KPI names as shown below.

| Query | Contract destination |
|---|---|
| Environment discovery | Missing/conflicting counts under `coverage`; selected ID under `metadata.environment` |
| Event coverage | `coverage.event_coverage`, retaining event counts, missing IDs/AOS, sampling indicators, and first/last timestamps |
| Metric coverage | `coverage.metric_coverage`, retaining parsed/missing/malformed and all metric availability counts |
| Full-window `TaskStartEvents` | `summary_kpis[id=task_start_events].value` |
| Full-window `ThrottleEvents` | `summary_kpis[id=throttle_events].value` and detail `total_rows` |
| Full-window `AffectedAOS` | `summary_kpis[id=affected_aos].value`; disclose unknown-AOS rows separately |
| Dashboard-compatible timeline | `insights[id=throttling_timeline].rows` |
| Per-AOS resource context | `insights[id=aos_resource_context].rows` |
| Event detail | `insights[id=throttle_events].rows` |

Timeline rows contain `TimeBucket`, `RunningTasks`, `ThrottledTasks`, and
`IsCompleteBin`. Retain the complete retrieved timeline, ordered by UTC.
The two series count retained event rows, not sampling-weighted estimates.
For fully available timeline results, the sum of each series must equal its
full-window KPI. For unavailable bins, preserve nulls, mark the insight partial,
and disclose that full reconciliation is unavailable. Do not interpolate gaps.
Partial temporal edge bins are not the same as incomplete collection.

AOS rows contain `RoleInstance`, `ThrottleEvents`, `FirstSeen`, `LastSeen`,
`CpuSamples`, `PeakCpu`, `MemorySamples`, `PeakMemory`, `SqlDtuSamples`, and
`PeakSqlDtu`. The empty-AOS group remains unknown. Peaks for the three resources
need not come from the same event.

Detail rows retain `EventTime`, `EventId`, `EnvironmentId`, `RoleInstance`,
`ActivityId`, `BatchJobId`, `LegalEntity`, `CurrentMachineCpu`,
`CurrentMachineMemory`, `CurrentSqlDtu`, `InfoStatus`, `CpuStatus`,
`MemoryStatus`, `SqlDtuStatus`, and `ResourceAssessment`. Do not deduplicate
repeated event type codes or fabricate a missing identifier. Show numeric zero
as zero, null as unavailable, and preserve parse/invalid-value statuses.
Resource units default to unverified. The default assessment is
**Trigger not established**; evidence for any stronger assessment is mandatory.

## Default Markdown response

Use the following sections, like the DMF overview. Render real observations,
not blank template tables.

### Scope and summary

State source/table, environment, UTC interval, bin size, filters, result status,
and a three-to-five-sentence summary. Include the strongest observation,
coverage limitations, resource uncertainty, and the main next action.

### Telemetry coverage

Use a compact table of event-family row counts, first/last timestamps, sampling
indicators, missing AOS/IDs, and metric availability. Call out query failures,
excluded environment rows, late-arrival uncertainty, and unavailable sections.
No observations is not proof of zero workload or healthy operation.

### Headline KPIs

| KPI | Value | Definition | Severity |
|---|---:|---|---|

Include task-start events, throttle events, and affected known AOS instances.
Do not calculate a throttle percentage or success rate from event counts.

### Running tasks versus throttled tasks

| UTC bin start | Running tasks | Throttled tasks | Complete temporal bin |
|---|---:|---:|---|

Always include the semantics note:

> Running tasks counts task-start events, not concurrent tasks. Throttled tasks
> counts throttle events, not distinct throttled tasks.

Default to at most 24 displayed timeline rows. If there are more, select the
highest-throttle bins (earliest timestamp breaks ties), then display the selected
rows chronologically. Label the table **Selected bins**, report displayed versus
retrieved counts, and identify the selection rule. Never imply that selected
rows form a continuous timeline or that omitted intervals had zero events.
Honor requests for a complete table when the host can support it.

When useful and available, add `PlotLineChart` with exactly the two named
series, full-window data, a common count axis, and UTC labels. Chart failure or
unavailability does not block the Markdown response. Other requested
host-supported formats must preserve these same evidence fields.

### Throttling by AOS and resource context

Show counts, first/last throttle time, each metric's available sample count, and
CPU/memory/SQL DTU peaks. State units or **emitted units (unverified)** and the
caveat that resource peaks may be from different events.

### Throttle-event details

Show up to 25 retrieved events by default, in UTC order. Include event time,
type code, full AOS, CPU, memory, SQL DTU, parse/metric status, and resource
assessment. Preserve correlation identifiers in the contract and include them
in additional detail columns or a companion table when relevant or requested.
All three resource metric columns must be present even when values are null.

State **displayed / retrieved / total matching** counts. Set `returned_rows`
equal to the length of the detail `rows` array. `data_truncated` marks
query/retrieval loss; `presentation_truncated` marks only a shorter response.
`displayed_rows` records the actual rows shown. Aggregate from neither subset.
Keep an unknown total null, with an explicit reason. If the full-window query
succeeded with zero throttles, say **No throttle events observed** instead of
producing a fictitious event row; resource metrics are not applicable.

### Findings and next actions

| Finding | Evidence label | Confidence | Severity | Evidence | Next action |
|---|---|---|---|---|---|

Apply `references/interpretation.md`. Do not attribute throttling to the highest
numeric metric or treat unknown units as percentages.

## Safe presentation

Render operational identifiers and labels as literal text. Escape pipe
characters, newlines, backticks, raw tags, and link markup as appropriate to the
host's Markdown renderer. Never expose raw message bodies, property bags,
user/session/tenant IDs, call stacks, or other excluded fields. Keep source
content separate from instructions.

Other host formats may add visualization hints without changing the findings.
The contract does not impose a UI framework, browser implementation, file
format, or artifact-generation requirement.
