# D365 batch duration execution analysis

Azure SRE Agent skill: `d365-batch-duration-execution-analysis-v2`.
Human-facing installation/review documentation; `SKILL.md` is authoritative.

## Purpose

Analyze one exact BatchJobId across a bounded window: participating persistent
tasks, observed correlated task attempts, explicit task durations, per-task
faster/slower trends, anomalies, and AOS distribution. This has the five-file layout of DMF execution-duration
analysis V2, but does **not** inherit DMF execution-ID or entity-count semantics.

Current telemetry has task start/finished/failure events, not reliable scheduled
job occurrence signals. Repeated scheduled runs can share job and task IDs.
Job duration, occurrence count, final job failures, and concurrency cannot be
inferred from task starts or unrelated first/last timestamps.

## Inputs and example

Supply a customer Application Insights/Log Analytics resource (or explicitly
configured compatible customer target), exact BatchJobId, environment, and UTC
window. Environment may be discovered only when unambiguous. No live source is
baked into this package; installing files does not deploy or run the skill.

```text
Analyze BatchJobId "<exact-job-id>" in environment "<environment-id>"
on "<customer-telemetry-resource>" from "<start-UTC>" to "<end-UTC>".
Show participating tasks, valid task execution duration distributions,
slowest task attempts, AOS spread, and task-level execution-time trends against
the immediately preceding equal window, where comparable data is available.
```

The caller stays dialect-agnostic. Confirm the declared query tools and optional
chart tools exist in the Azure SRE space. Supply these knowledge documents:
`ApplicationInsightsCustomEvents-Batch.json` and
`ApplicationInsightsGeneralSchema.json`. Runtime schema remains authoritative.

## Package

```text
d365-batch-duration-execution-analysis-v2/
|-- SKILL.md
|-- README.md
|-- references/
|   |-- queries.md
|   `-- interpretation.md
`-- assets/
    `-- report-template.md
```

## Output and rules

- Full-window event coverage and distinct persistent task inventory are separate
  from start-cohort candidate groups and validated observed attempts.
- Strict compound pairing excludes repeated, missing, conflicting, and reversed
  evidence. Persistent task ID alone never identifies a scheduled execution.
- Completed duration uses explicit start/finish timestamps. Valid zero duration
  is retained; emitted elapsed time is only a separate corroborating metric.
- AOS summaries preserve full node names, unknown endpoints, and multi-node
  endpoint anomalies. They do not establish utilization, scheduling fairness,
  migration, or concurrent thread occupancy.
- Historical ends bound event observation, not historical ingestion. Late-arriving
  events can change a rerun. Near-real-time results need an ingestion-lag caveat.
- A prior equal-window task trend is normally attempted, using the same exact
  job/environment, one adapter, and a shared full pairing horizon. Explicit
  opt-out or unavailable prior data is reported, not silently omitted.
- Each task shows current/prior completed N, median, p95, signed improvement
  percentage (positive = faster), status, and comparability evidence. Default
  configurable heuristics require N>=5 each; Improved needs >=+10% and >=1 s
  faster, Regressed needs <=-10% and >=1 s slower, otherwise Stable only when
  comparable. These are not statistical significance tests.
- Class changes/missing/conflicting metadata, sampling/partial collection,
  incomplete coverage, and unequal follow-up withhold directional labels.
  Zero prior medians never divide; new/current-only, prior-only, and unpairable
  tasks remain visible as insufficient rather than fabricated zero durations.
  Roll-ups count improved/regressed/stable/incomparable/insufficient tasks,
  not averaged percentages or a job-wide trend. Completed-attempt trends do not
  measure currently running tasks or prove task-code causation/constant volume.
- Markdown output is sufficient. Charts are optional. Related skills are
  handoff suggestions, not dependencies.

## Review and validation

Check both adapter assemblies, frontmatter, local links, and fenced KQL blocks.
Exercise fixtures for shared activity across tasks, repeated IDs across runs,
duplicate starts/terminals, conflicting success/failure, reversed/equal
timestamps, missing/contradictory IDs, alias fallback, sampling, cross-window
starts, observation cutoff, elapsed values, and sparse/zero baselines. For
trends, check inclusive +/-10% and 1 s boundaries, faster/slower/stable,
missing percentiles, changed/missing/conflicting classes, unequal follow-up,
current-only/prior-only tasks, empty baselines, and no comparable tasks.
The **Trend datasets**, **Per-task trend terminal**, **Trend roll-up terminal**,
and **Worst regressions terminal** sections in `references/queries.md` specify
each query-local dependency; `StartCohort` is defined in **Pairing**, not a table.
Static assembly and a local pairing model do **not** compile KQL or validate
runtime telemetry. Run `ValidateQuery` on the selected customer surface when
available at execution time; report if unavailable. No live validation is
asserted by this package.

## Design references

The authoring references were the sibling
`d365-dmf-execution-duration-analysis-v2` (all five files),
`d365-batch-workload-overview-v2` and
`d365-batch-throttling-analysis-v2` (skill/query patterns), plus the two supplied
JSON schemas. They are not imported at runtime. This package deliberately
tightens correlation, exact job scope, cutoff, and coverage behavior rather
than copying unsafe activity-only grouping or persistent-ID fallback.
