# Task-duration interpretation

## Evidence and confidence

- **FACT:** observed event count, ID coverage, explicit validated pair duration,
  endpoint AOS, or finite emitted elapsed value. Counts describe retained rows.
- **CORRELATED:** an independently retrieved signal aligns in time and scope.
- **HYPOTHESIS:** SQL blocking, resource pressure, scheduling constraints, slow
  code, retry behavior, or imbalance requiring further evidence.

High confidence can describe a directly observed duration with sound coverage,
not its cause. Use medium/low confidence when missing IDs, uncertain activity
semantics, sampling, cutoff, or query truncation affects interpretation. Do not
assign high-confidence root cause from duration or AOS concentration alone.

## Compact classification

| Classification | Evidence | Next action |
|---|---|---|
| SLOW_COMPLETED_TASK | Valid completed pair exceeds stated contextual threshold | Inspect that task/class/activity in customer dependency or exception telemetry |
| REPEATED_OR_CONFLICTING | Repeated endpoints, contradictory aliases, reused correlation key, or finish plus failure | Confirm emitter/retry semantics; do not choose nearest endpoints |
| FAILED_ATTEMPT_OBSERVED | One start and one failure, correctly ordered | Investigate failure and subsequent attempts; do not call the job failed |
| TERMINAL_NOT_OBSERVED | Start but no terminal by fetch cutoff | Check collection/retention, buffer, lag, and authoritative batch history |
| DURATION_DISAGREEMENT | Emitted versus paired elapsed differs materially | Confirm clock precision and emitter scope, rather than replacing pair duration |
| AOS_CONCENTRATION | Most known event placements or assigned pair-seconds on one node | Check scheduling/affinity and capacity independently |
| INSUFFICIENT_EVIDENCE | Missing/conflicting IDs, sampling, too few baseline pairs | Repair or confirm telemetry before drawing performance conclusions |

Use an explicit customer SLA or task baseline first. Optional triage default:
completed pair duration >=300 seconds is a review candidate, not a failure or
automatic critical incident. Emitted disagreement default is strictly greater
than `max(1000 ms, 5% of paired ms)`; disclose it. Zero paired or emitted duration
is valid; a zero baseline suppresses percentage division, not sample counts.
Critical severity requires documented business impact or SLA evidence.

## Counting and time semantics

Distinct persistent task IDs count observed task definitions, not runs or
attempts. Candidate compound keys are not execution counts. A valid pair
identifies an **observed unambiguous task attempt within the fetched horizon**;
it is not proof of globally unique activity reuse behavior.

Duration statistics use starts in `[start, end)` with eligible terminals in the
buffer. Boundary terminal groups are coverage only. Repeated groups are selected
by any start in the window; never hide a repeated start because the group's
earliest start lies before it. Task inventory and raw AOS placement use events
in the requested window and can therefore have different totals from the start
cohort. State both denominators. Failure row count includes unpaired/repeated
signals; paired failures count only valid failed attempts.

Summed completed task seconds can exceed wall-clock time and are not concurrency.
If useful, describe first/last selected task event times as an **observed task
envelope**, including gaps and potentially many scheduled runs. It is not a
job-occurrence duration and must never be labeled as one.

Missing terminals are right-censored at the event cutoff, even for a historical
request. Missing starts can be left-truncated, absent, sampled, or mismatched,
not proof of a currently running task. Five minutes is an initial ingestion-lag
caveat, not a promise of completeness; use measured delay if available. The
adapter accepts events ingested by frozen observation time, not necessarily by
the historical end. Do not claim an as-of historical ingestion snapshot.

`itemCount`/`ItemCount > 1` is a sampling indicator. Missing/invalid weights mean
unknown state; weights of one do not prove complete collection. Never scale
pair counts or reconstruct missing endpoints from sampling weights. Repeated
events may be duplicate emission or retry/reuse; no deduplication is justified
by the event type code `eventId`.

## AOS caution

One-AOS pair attribution requires both endpoints known and identical. Different
endpoint nodes are reported separately, not evidence of task migration. Unknown
nodes are not a real extra server. No allocation of an entire pair's duration
across multiple nodes without additional evidence. Class metadata missing or
conflicting remains visible; do not drop endpoints to obtain a clean class.

## Per-task execution-duration trend

Normally compare completed attempts of each persistent BatchTaskId in the
current and immediately preceding equal, disjoint start windows, scoped to the
same exact job/environment. Use the shared full fetch horizon from **Parameters**
through **Pairing** in `queries.md`; never independently pair the two periods.
`StartCohort` is that query's local Pairing result, not a telemetry table.
Repeated keys across either period remain ambiguous. Preserve the union of task
IDs in event coverage, even if no correlation group/completed pair survives.
Event presence is not evidence of a new/deleted task definition: CurrentOnly
and PriorOnly mean observed only in that interval.

Show current/prior completed N, median and p95 seconds for each task, including
null durations where no valid pair exists. A successful empty baseline gives
PriorN=0, not a zero median. An inaccessible, failed, or partial query gives
unavailable metrics, not zero counts. If comparison is disabled, explain why.

Default configurable **heuristics**, applied in this order:

1. Fewer than 5 completed pairs in either period: **InsufficientSamples**.
2. Null/nonfinite/negative median or p95: **MissingDuration** (insufficient).
3. Prior median zero: **ZeroPriorMedian** (insufficient percentage baseline);
   keep absolute duration evidence, but never divide or label Stable/Improved.
4. Any comparability flag: **Incomparable**, with specific flags and low
   confidence; do not present a signed percentage as a valid trend.
5. Otherwise calculate `MedianImprovementSec = prior median - current median`
   and `ImprovementPct = 100 * MedianImprovementSec / prior median`.
   **Improved** (positive/faster) requires >=+10% AND >=+1 second;
   **Regressed** (negative/slower) requires <=-10% AND <=-1 second.
   **Stable** means neither two-part threshold is met, not identical performance.
   Use unrounded values for classification; format signs/rounding afterward.
   These are median-based statuses; p95 is separate tail evidence. Call out a
   worsening p95 even when the median improves rather than hiding the tail.

Retain coverage flags even when a sample/baseline gate takes precedence. In
**Trend datasets**, class sets come from all known-task event rows in each
window plus the corresponding start-cohort endpoint evidence, not a top-N
inventory. Missing/whitespace-only class metadata, conflicting endpoint
classes, changed sets, or truncated sets disallow directional labels. Sampling
indicators/unknown weights and unusable correlation rows also disallow them.
Counts of one do not prove collection completeness: confirm retention, ingest
lag, source/query completeness, and unassigned scope/ID coverage independently
before setting `_trendCoverageConfirmed=true`. Leave it false when uncertain.
Partial query results invalidate affected metrics entirely, not just confidence.

The comparison uses a strict equal-follow-up guard without changing pairing:
the current window must have its full disclosed buffer observed (and ingestion
lag allowed), and every selected start group in both periods must be completed
with terminal no later than `StartTime + _pairingBuffer`. Longer completed
durations remain in the descriptive N/median/p95, but make the task
Incomparable; they are not dropped to manufacture improvement. Noncompleted
start groups and unpaired boundary groups similarly flag possible survivor
bias or missing endpoints; valid boundary-only pairs are not a defect.
Minimum-sample failures still
report insufficiency, including when conservative repeated-key exclusions leave
no usable pairs. Do not shorten the shared fetch horizon to hide reuse. A
near-real-time interval usually fails follow-up and cannot get a confident
directional label; report unavailable comparison or select a disclosed mature
window only if the caller authorizes different bounds.

Roll up counts of **Improved / Regressed / Stable / Incomparable / Insufficient**;
the last includes InsufficientSamples, MissingDuration, and ZeroPriorMedian.
Include current-only/prior-only tasks, and say explicitly when there are no
comparable tasks. Never average percentages, pool changing task populations
into a job-wide trend, infer occurrences, or equate task counts with runs.
Rank comparable regressions by median slowdown seconds then negative percentage,
retain both cohorts' N/median/p95 and class evidence, and drill into corresponding
slow completed attempts before an evidence-based handoff.

These are descriptive cohort comparisons, not statistical significance tests
or a fitted time-series trend. Minimum N=5 is only a gate; p95 with five samples
is unstable. Same class does not prove constant input volume/configuration.
A slower task is not proof of a task-code regression: volume, data shape,
contention, scheduling, or configuration may differ and require separate
evidence. Incomplete attempts have no observed full duration; never describe
these results as full-duration trends of tasks currently running.

## Handoffs and future schema

- `d365-batch-workload-overview-v2`: independent thread, queue, resource, and
  broad batch-health context.
- `d365-batch-throttling-analysis-v2`: environment/AOS throttling context.
  Throttle events with empty/zero job IDs are not attributable to this job.
- Customer exception/dependency or code investigation: pass only exact
  environment/job/task/activity, AOS, bounded UTC interval, pair status, and
  sanitized class/type. No raw messages, stacks, or property bags.

These handoffs are recommendations, not automatic queries in this skill.
Add job-occurrence analysis only after new lifecycle events and an explicit
occurrence key are confirmed in runtime schema, including retry and terminal
semantics. Then version and validate that contract separately.
