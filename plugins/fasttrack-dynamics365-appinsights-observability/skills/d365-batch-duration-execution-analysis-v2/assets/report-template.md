# Batch task duration analysis — one BatchJobId

> Current telemetry contains task start/finished/failure signals only.
> Scheduled job occurrences, job-level duration, and final job status are not
> established. BatchJobId and persistent BatchTaskId can repeat across runs.

## Scope and short summary

| Field | Value |
|---|---|
| Customer source / selected adapter | |
| Environment / exact BatchJobId | |
| Current UTC interval [start, end) | |
| Prior equal interval / unavailable or explicitly disabled reason | |
| Frozen observation time / event cutoff / fetch interval | |
| Pairing buffer / ingestion-lag caveat / retention | |
| Duration cohort | Starts in interval; terminals may be in buffer |
| Correlation / operation-fallback evidence | |
| Query completeness / query and display limits | |

## Coverage and counting

| Metric | Current value | Definition / limitation |
|---|---:|---|
| Retained starts / finishes / failures | | Event-window rows; not concurrency or final job outcomes |
| Distinct persistent tasks | | Exact known string IDs observed in event window |
| Candidate correlation groups | | Groups with a window start; not all valid executions |
| Valid completed / failed attempts | | Explicit exclusive endpoint pairs in start cohort |
| Missing/conflicting environment or job scope | | Unscoped discovery counts, not attributed to the selected job |
| Missing/conflicting task or activity | | Scoped rows; excluded/quarantined for pairing |
| Repeated / conflicting / reversed groups | | Excluded from duration |
| Start without terminal / boundary-only terminal groups | | Not observed by cutoff; coverage, not zero duration |
| Missing AOS / unknown or indicated sampling | | Known-node and collection coverage |
| Emitted elapsed availability / invalid values | | Finish rows; finite nonnegative values only |
| Retry-count availability | | Unavailable in supplied schema |

Do not fill unavailable results with zero. Record totals versus displayed rows
and all unavailable sections. Event-window and start-cohort counts differ.

## Participating persistent tasks

| Task ID | Class evidence | Start / finish / failure rows | Known AOS / missing AOS rows | Correlation coverage |
|---|---|---|---|---|

## Slowest valid completed task attempts

| Task ID | Activity / provenance | Class | Start UTC | Finish UTC | Paired seconds | Emitted seconds / delta | Endpoint AOS / attribution |
|---|---|---|---|---|---:|---|---|

## Task duration distributions

| Task ID | Valid completed N | Sum task seconds | Median seconds | p95 seconds | Max seconds | Classification |
|---|---:|---:|---:|---:|---:|---|

Summed task seconds are additive completed work, not job duration, wall-clock
time, or concurrency. Optional charts use these full-population aggregates.

## AOS distribution

| Full AOS instance or unassigned status | Event-window starts / finishes / failures | Distinct persistent tasks | Start-cohort completed N / sum seconds | Coverage |
|---|---|---:|---|---|

Do not join the two cohort counts as though they measured the same population.
Unknown AOS is not a physical server; multi-node endpoint pairs are unassigned.

## Incomplete, ambiguous, and boundary evidence

| Task / activity | Period / coverage cohort | Pair status | Endpoint counts | Start / terminal UTC | Cutoff age / boundary note | Next action |
|---|---|---|---|---|---|---|

## Per-task execution-duration trend

Always include this section, even when prior data or the comparison is unavailable.
Use the union of known task IDs observed in either interval, including tasks
with no usable pairs. Missing/conflicting task IDs remain coverage buckets above.

| Trend configuration / evidence | Value |
|---|---|
| Min completed N each / percent threshold / absolute seconds floor | 5 / 10% / 1 s (configurable heuristics) |
| Shared pairing horizon / equal per-start follow-up bound | |
| Collection coverage confirmed / evidence or reason not confirmed | |
| Current / prior task populations / total versus displayed | |

| Task | Current N / prior N | Current / prior median s | Current / prior p95 s | Improvement % (+ faster) | Trend status / population status | Coverage flags / class sets |
|---|---|---|---|---:|---|---|

| Full-population task roll-up | Count |
|---|---:|
| Improved (positive/faster) | |
| Regressed (negative/slower) | |
| Stable (within heuristic thresholds) | |
| Incomparable | |
| Insufficient (including zero prior median) | |
| Total union of observed known task IDs | |

Only comparable tasks receive directional or Stable labels. Improved requires
>=+10% and >=1 s faster; Regressed requires <=-10% and >=1 s slower.
N>=5 each is a minimum, not significance; p95 can be unstable. Statuses are
median-based; call out contrary/worsening p95 tail evidence. Keep
InsufficientSamples, MissingDuration, and ZeroPriorMedian reasons explicit.
Percentage is unavailable for insufficient/incomparable evidence or zero prior
median; never replace null with zero. A successful empty cohort has N=0 but no
duration, unlike an unavailable/partial query whose metrics are unavailable.
With no comparable tasks, say so; zero classified improvements is not stability.
Do not average percentages or call this a job-wide trend.

### Worst comparable regressions

| Task / class | Median slowdown s | Improvement % (negative) | Current / prior N | Current / prior median and p95 s | Supporting attempt / next check |
|---|---:|---:|---|---|---|

Use full-population ranking before the display limit. Investigate corresponding
completed-attempt evidence; incomplete tasks have no observed full-duration
trend and are not proven currently running. Report class, collection, follow-up,
workload-volume/configuration unknowns, and confidence. These are descriptive
duration shifts, not proof of a task-code cause or statistical significance.

## Assessment and handoffs

| Task / finding | Classification | FACT / CORRELATED / HYPOTHESIS | Confidence / severity | Evidence / threshold | Next action |
|---|---|---|---|---|---|

Future extension: job-occurrence metrics require confirmed occurrence IDs and
job lifecycle events with documented retry semantics. No occurrence metrics
are inferred in this report.
