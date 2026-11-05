# D365 available batch threads analysis

Standalone Azure SRE Agent skill: `d365-batch-available-threads-analysis-v2`.
Markdown-first, five files, no Copilot Studio or HTML/artifact dependency.

## Measure and output

Uses `BatchThreadInfo` snapshots and the supplied dashboard formula:

`AvailableThreads = MaxThreadCount - CurrentBatchTasks - ReservedNumberOfThreads`

This is dashboard-derived signed headroom, not task-start throughput or
guaranteed all-work scheduler capacity. Reserved execution may overlap current
tasks; the supplied schema does not resolve that behavior. Negative results
are retained as headroom deficits, not labeled physical negative threads.

Default: one exact environment, last 24 hours frozen once, 10-minute UTC bins.
Optional: one exact full AOS instance. No job filter. Each node keeps its full
name and separate graph series. Bucket medians are accompanied by min/max,
valid/invalid counts, coverage, and zero/negative observations. Empty or
invalid-only bins have null headroom, never zero or carried-forward values.

The response includes scope and coverage, timeline graph via `PlotLineChart`
when supported, matching underlying Markdown data, per-AOS summaries and latest
configuration, freshness as of the requested window, and evidence-labeled
findings. A missing graph is explicitly disclosed, not silently promised.

## Setup

Supply an authorized customer telemetry resource or an unambiguous configured
connection. Provide these two knowledge files, and no additional knowledge:

- `ApplicationInsightsGeneralSchema.json`
- `ApplicationInsightsCustomEvents-Batch.json`

Confirm the tools in `SKILL.md` are exposed by the Azure SRE host. Runtime
metadata controls source mapping; the skill internally selects one adapter:

| Surface | Table | Time | Properties | Full AOS |
|---|---|---|---|---|
| Application Insights | customEvents | timestamp | customDimensions | cloud_RoleInstance |
| Log Analytics | AppEvents | TimeGenerated | Properties | AppRoleInstance |

Only confirmed missing-table errors permit same-target alternate-table
probing. Zero rows and query/access failures never mean successful empty
capacity. The user/caller is not asked to pick a dialect.

## Example prompts

```text
Show available batch threads over the last 24 hours for environment
<environment-id> using resource <resource-id>. Include the timeline graph,
underlying data, and last observed configuration for each AOS.
```

```text
Analyze available batch threads for environment <environment-id> from
2026-09-14T00:00:00Z to 2026-09-15T00:00:00Z, restricted to the exact AOS
<full-role-instance>. Explain observed zero/negative headroom and gaps.
```

## Package

- `SKILL.md`: activation, tools, scope, execution and reporting workflow.
- `references/queries.md`: adapters, validation, composable bounded queries.
- `references/interpretation.md`: semantics, evidence and operational caveats.
- `assets/findings-contract.md`: DMF-style metadata, coverage, KPIs, insights,
  findings, chart mapping, and Markdown delivery contract.
- `README.md`: setup and validation notes.

## Validation and limitations

Local validation checks query-block composition with each source adapter,
schema mappings, frontmatter and local links, plus fixtures for numeric
validation, signed negatives, zero capacity, latest-invalid snapshots,
duplicate/conflicting times, null/mixed bins, full AOS names, and historical
freshness. These are offline structural and behavioral checks using existing
Node/PowerShell builtins, NOT server-side KQL compilation or live chart tests.
Authoring validation passed **29 offline checks**, including all **10**
adapter/terminal combinations.
No live telemetry source is configured for package authoring. Validate the
assembled queries and chart capabilities on the destination host before use.

Preflight bounds query rows and cells; it never trims away low-headroom nodes
or treats truncation as healthy zeros. Complete query data is retained in the
contract, even when the Markdown view is an explicitly labeled preview.
