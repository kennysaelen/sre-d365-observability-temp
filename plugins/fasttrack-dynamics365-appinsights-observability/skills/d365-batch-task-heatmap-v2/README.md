# D365 Batch Task Heatmap

Azure SRE Agent skill: `d365-batch-task-heatmap-v2`.

Uses the supplied skill name and the same Markdown-first, five-file structure
as the DMF overview and batch throttling skills. Generates an hourly heatmap of
batch task starts across AOS instances, with the data used to plot it.

## What it measures

Each cell counts **BatchTaskStart events for one full AOS instance in one UTC
hour**. Default interval: last 24 hours. This shows task-start distribution,
not concurrent-running task count, execution duration, or resource utilization.

The response includes:

- Source, environment, UTC interval, filters, and collection coverage.
- Total task starts, known-AOS starts, unknown-AOS starts, and observed AOS count.
- Heatmap using `PlotHeatmap`, plus its underlying Markdown matrix/counts.
- Per-AOS totals/share and the highest-count cells.
- Evidence-qualified interpretation and next actions.

## Source selection and prerequisites

Provide a telemetry resource or an unambiguous configured connection, an
environment, and optionally a time window. The skill selects its source adapter
internally; no table selection or dialect flag is needed from the caller.

| Surface | Table | AOS instance | Time |
|---|---|---|---|
| Application Insights | `customEvents` | `cloud_RoleInstance` | `timestamp` |
| Log Analytics | `AppEvents` | `AppRoleInstance` | `TimeGenerated` |

Both adapters feed shared analysis queries. Only one table family is queried.
The role-instance column's case and availability are verified from runtime
schema. Unknown-AOS events remain visible in coverage but are not plotted as a
real server.

Provide `ApplicationInsightsGeneralSchema.json` and
`ApplicationInsightsCustomEvents-Batch.json` as knowledge. Confirm the query
and plotting tool identifiers in `SKILL.md` against the target Azure SRE space.
Use `ValidateQuery` when supported. The plotting tool is used when available;
if unavailable, the skill returns the matrix and explicitly reports that no
heatmap could be rendered. No HTML renderer or artifact-writing tool is needed.

## Example prompts

```text
Show a heatmap of batch task starts across AOS instances for environment
<environment-id> over the last 24 hours. Include the hourly count data.
```

```text
Using workspace <workspace-id>, show the AOS batch task-start distribution for
environment <environment-id> from 2026-09-14T00:00:00Z to 2026-09-15T00:00:00Z.
Use one-hour buckets and explain the busiest windows.
```

```text
Generate a batch task heatmap for class <class-name> in environment
<environment-id> over the past three days. Keep separate dates and include
per-AOS counts and coverage limitations.
```

## Layout

```text
d365-batch-task-heatmap-v2/
|-- README.md
|-- SKILL.md
|-- references/
|   |-- queries.md
|   `-- interpretation.md
`-- assets/
    `-- findings-contract.md
```

Charts and Markdown derive from the same findings contract. Large datasets
are split into disclosed panels/batches rather than silently limited to top
servers. All panels use the same count/color scale. The contract retains
retrieved rows beyond a shortened Markdown preview.

## Review checklist

- Confirm automatic routing, schema discovery, and empty/error distinctions.
- Assemble every query with one adapter and its required definitions.
- Test missing/conflicting environment IDs, missing AOS, and exact filters.
- Test recurring task IDs and separate nodes with similar hyphenated names.
- Test zero-start cells, all-empty results, all-unknown AOS, and sampled rows.
- Test aligned and nonaligned hourly boundaries, including exclusive end time.
- Confirm grid preflight happens before expansion and batching preserves all
  nodes, time bins, totals, and color scale.
- Reconcile matrix, per-AOS, and per-hour totals; keep unknown AOS separate.
- Confirm PlotHeatmap uses UTC time/full instance/count, not CPU or concurrency.
- Confirm chart unavailability does not hide the numeric matrix or claim a
  chart exists. Report all preview/query limits.

No customer-specific targets or sample telemetry are embedded. Live KQL and
chart execution require the target host's customer telemetry connection.
