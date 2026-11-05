# D365 Batch Throttling Analysis

Azure SRE Agent skill: `d365-batch-throttling-analysis-v2`.

Produces a Markdown-first throttling analysis from customer-accessible D365
Finance and Operations Application Insights or Log Analytics events. Follows
the DMF overview pattern: operating instructions, separate query and
interpretation references, and a host-agnostic findings contract.

`SKILL.md` instructs the agent; this README is for installation and maintenance.
This package is not a Copilot Studio agent or an internal telemetry skill.

## Report contents

- Two-series UTC timeline: **Running tasks** and **Throttled tasks**, default
  10-minute bins over the last 24 hours, presented as a Markdown table with
  an optional chart.
- Counts and coverage from the full analysis window.
- Throttle-event table showing timestamp, event code, full AOS instance,
  activity, job attribution where available, and CPU, memory, SQL DTU values.
- Metric availability, units, resource assessment, findings, and next actions.
- Explicit partial-result, sampling, late-arrival, and detail-limit caveats.

The dashboard's "Running tasks" series counts `BatchTaskStart` events, not
concurrent tasks. "Throttled tasks" counts `BatchThrottled` events, not distinct
tasks. Those definitions are always visible in the report. Actual concurrency
belongs in a separate thread-capacity analysis using `BatchThreadInfo`.

## Automatic source selection

The caller provides or inherits a telemetry resource, environment, and time
window. No dialect flag, table selection, or query rewriting is required.

| Surface | Event table | Properties |
|---|---|---|
| Application Insights query tool | `customEvents` | `customDimensions` |
| Log Analytics query tool | `AppEvents` | `Properties` |

The skill probes the tool-preferred schema and selects one short adapter.
Both adapters feed one shared analysis library. A missing table can trigger a
same-target schema fallback; empty results, access errors, and timeouts cannot.
If both families exist, only the preferred one is queried.

## Knowledge and prerequisites

Provide these files as Azure SRE Agent knowledge:

- `ApplicationInsightsGeneralSchema.json`
- `ApplicationInsightsCustomEvents-Batch.json`

The target space needs one of the query tools listed in `SKILL.md`. Confirm
their exact catalog names before deployment. `ValidateQuery` is used when
supported. `PlotLineChart` is optional, not a prerequisite for completing the
analysis. No browser, HTML template, or artifact-writing tool is required.

Telemetry must be enabled and readable for the requested resource. Numeric
system metrics alone do not establish whether CPU, memory, or SQL caused a
throttle; the supplied schema does not specify units or trigger thresholds.
All three values are shown, with unknown attribution explicit.

## Example prompts

```text
Analyze batch throttling for environment <environment-id> using the configured
telemetry resource over the last 24 hours. Summarize in Markdown with the
running-versus-throttled timeline and per-event CPU, memory, and SQL DTU values.
```

```text
Summarize batch throttling from workspace <workspace-id> for
environment <environment-id>, 2026-09-14T00:00:00Z to 2026-09-15T00:00:00Z.
Identify the AOS instances and time windows with the most throttling.
```

Optional filter: one exact AOS role instance. Job/class/activity filters are
intentionally not applied to the primary counts, since throttle records commonly
carry an empty or zero job ID. The report must not imply job attribution.

## Layout

```text
d365-batch-throttling-analysis-v2/
|-- README.md
|-- SKILL.md
|-- references/
|   |-- queries.md
|   `-- interpretation.md
`-- assets/
    `-- findings-contract.md
```

Build `assets/findings-contract.md` internally and present the result as
Markdown, not a JSON dump. The response follows the DMF overview's structure:
scope and summary, coverage, KPIs, focused insights, and evidence-labeled
findings with next actions.

The default response shows up to 24 timeline rows and 25 event-detail rows.
The query detail limit remains 500. Disclose displayed, retrieved, and matching
counts separately. Preserve all retrieved rows in the contract and use full
query results for KPIs and optional charts. Another host-supported format is
permitted only when requested; it is not a dependency of the analysis.

## Review checklist

- Confirm automatic routing for both telemetry tools and a generic target.
- Confirm a missing preferred table is distinguishable from an empty result,
  access error, timeout, or missing source connection.
- Validate assembled KQL on each target surface.
- Test mixed environment casing, conflicting environment keys, and missing AOS.
- Test JSON-string versus dynamic property bags; missing/malformed `InfoMessage`;
  zero, missing, nonnumeric, nonfinite, and negative metric values.
- Test half-open boundaries, nonaligned partial bins, empty windows, starts
  without throttles, throttles without starts, and multiple events at one time.
- Verify full-window counts reconcile with the timeline, regardless of the
  detail cap; repeated event codes must not be deduplicated.
- Test detail truncation, sampling indicators, and unavailable query sections.
- Confirm the Markdown response has scope, coverage, KPIs, timeline evidence,
  per-AOS resource context, event details, and findings with next actions.
- Confirm three system metric columns, explicit unknown units, unavailable
  rather than zero values, and literal rendering of Markdown-sensitive labels.
- Confirm analysis completes without chart or file-writing tools. If a chart
  is added, confirm exactly two named series using full timeline results.
- Confirm presentation limits do not change the underlying findings contract
  or summary totals.

Live query validation requires a configured customer telemetry connection.
The package itself contains no customer-specific target or live report data.
