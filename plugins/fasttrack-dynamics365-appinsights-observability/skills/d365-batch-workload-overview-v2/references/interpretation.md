# Batch workload interpretation

## Evidence labels

- **FACT**: Directly observed event, count, timestamp, duration, configuration,
  queue value, or resource value.
- **CORRELATED**: Independent signals align in the same environment, time
  window, and preferably AOS or activity.
- **HYPOTHESIS**: A plausible explanation that needs additional evidence.

Do not use causal wording for correlation alone.

## Confidence

| Confidence | Use when |
|---|---|
| HIGH | Expected events are present, environment scope is exact, pairing and sampling coverage are strong, and fields are consistently populated. |
| MEDIUM | The conclusion is useful but relies on partial identifiers, sparse samples, fallback correlation, or one supporting signal. |
| LOW | Required events or fields are materially absent, the source is truncated/sampled, or competing explanations remain. |

## Severity

Severity is contextual and must state the rule used.

- **critical**: sustained capacity exhaustion or queue growth with impact,
  repeated failed executions affecting important workloads, or widespread
  throttling with aligned resource pressure.
- **warning**: material degradation, intermittent saturation, sustained
  nonzero backlog, elevated failure rate, or a meaningful baseline regression.
- **info**: normal workload observations, isolated events without demonstrated
  impact, or coverage limitations.
- **success**: explicitly healthy evidence with adequate coverage.

Do not infer health from absent telemetry.

## Metric semantics

- Task starts per bucket measure arrival/throughput, not active concurrency.
- `CurrentBatchTasks` from `BatchThreadInfo` is the concurrency signal.
- Effective capacity is maximum threads minus reserved threads.
- Cumulative task duration can exceed wall-clock time because tasks overlap.
- A failed correlated execution comes from `BatchTaskFailure`.
- `AppExceptions`/`exceptions` rows are supporting exception signals and can
  overlap with failure events.
- A missing terminal event can mean active, delayed ingestion, truncated
  observation, lost telemetry, or a genuinely unfinished task.
- A nonzero queue sample is not a sustained backlog without duration and sample
  cadence evidence.
- CPU, memory, and SQL DTU values from `BatchThrottled.InfoMessage` are emitted
  context values. Confirm units/ranges from live data before assigning
  threshold-based severity.

## Recommended default heuristics

Use defaults only when no environment-specific baseline or SLO exists:

- Apply at least five minutes of ingestion-lag protection.
- Require at least three consecutive nonzero queue samples before calling a
  backlog sustained.
- Treat p95 effective utilization of 85% or more as warning context and 95% or
  more as critical context only when sample coverage is adequate.
- Treat a failure-rate increase as actionable only with an explicit
  denominator, at least three failed executions, and either a material baseline
  increase or important-workload impact.
- Do not calculate percentage change from a zero baseline.

These are triage heuristics, not product limits.

