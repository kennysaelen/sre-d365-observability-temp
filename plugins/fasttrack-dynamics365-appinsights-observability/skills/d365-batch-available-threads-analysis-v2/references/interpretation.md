# Evidence and interpretation

## Knowledge boundary

The batch schema defines `BatchThreadInfo` (`Batch00007`) and nested
`InfoMessage` integer fields:

| Field | Unit and schema meaning |
|---|---|
| CurrentBatchTasks | Count of batch tasks currently being processed |
| TaskQueueCount | Count of tasks currently queued waiting for processing |
| MaxThreadCount | Maximum count of threads available for processing |
| ReservedNumberOfThreads | Count of threads reserved for high-priority tasks |

The general schema establishes Application Insights field names, including
`cloud_RoleInstance` and `itemCount`. Runtime metadata confirms the selected
surface. No other knowledge or scheduler implementation is assumed.

The user's dashboard supplied `maximum - current - reserved`. Preserve that
formula, but not its lossy `split(cloud_RoleInstance, '-')[0]` node naming.
The schema does not establish whether CurrentBatchTasks includes work executing
on reserved capacity. Therefore this formula is **dashboard-derived headroom**,
not guaranteed spare slots for every task type or all-work schedulable capacity.

## Counts, validity, and signs

- Validate every input independently. Missing, nonnumeric, nonfinite, negative,
  fractional, or outside-schema-int-range inputs are unavailable, not zero.
  InfoMessage must parse as a JSON object. Retain reason codes.
- Required headroom inputs are current, maximum, and reserved. Queue is optional
  and never subtracted. Unknown queue does not mean an empty queue.
- Reserved > maximum makes the required snapshot invalid. Retain the parsed
  counts and explicit inconsistency reason, but derived headroom is null.
- Current > maximum - reserved yields a valid signed negative formula result,
  NOT automatically corrupt telemetry. It can reflect demand/reservation
  mismatch or unknown reservation-overlap semantics. It is not physical
  negative threads, proof of overload, or proof of scheduler failure.
- EffectiveCapacity = maximum - reserved for a valid snapshot. It may be zero.
  Available may then be zero or negative. HeadroomPct is null at zero capacity;
  otherwise it is a signed **formula ratio**, not authoritative utilization.
- Zero headroom is one observed zero-formula snapshot. Positive headroom does
  not prove task eligibility or scheduler availability; negative/zero does not
  establish customer impact. Never clamp signed values for chart cosmetics.

## Sampling, gaps, and aggregation

Every retained event contributes once, unweighted. `ItemCount > 1` is a
sampling/aggregation indicator; do not multiply gauges or reconstruct sample
populations. Missing/invalid weights are coverage flags; a value of one is
not proof of lossless collection. Sampling may hide brief minima.

Statistics describe retained observations, not exact time-weighted exposure.
KQL percentile estimates are sample-distribution summaries; neither p95 nor
median substitutes for the minimum. A negative min with positive median must
remain visible in the evidence and findings. Sample fractions cannot establish
percentage of wall-clock time exhausted or sustained saturation.

Per-node bucket median is a deliberate bounded representation of the original
dashboard's sampled timeline, not the exact instantaneous path. Keep min/max,
sample counts, first/last observation times, and coverage. Do not sum gauges
across time or unsynchronized nodes. Different emission rates bias pooled
sample distributions; do not create a fleet headroom statistic.

| Coverage state | Meaning |
|---|---|
| NoObservation | No retained snapshot in the bin; headroom unknown |
| InvalidOnly | Snapshots observed, none with valid required inputs |
| Mixed | Valid summary accompanied by invalid samples; incomplete coverage |
| ValidObserved | Required inputs valid in all retained rows; cadence still unknown |

NoObservation may reflect collection gaps, sampling, restart, a node that did
not exist for that period, or unknown emission cadence. Do not choose a cause
without evidence. Partial first/last bins are explicitly labeled. Never
zero-fill available values, carry prior values forward, or interpolate through
null bins. A near-live window may miss late arrivals; initially disclose a
five-minute ingestion-lag caution as a heuristic, not a verified source SLA.

Repeated identical-time values and conflicting same-time values are flagged
without deduplication. Repeated timestamps are not independent elapsed-time
evidence. Conflicting latest timestamps mean current configuration is ambiguous:
the displayed candidate remains one coherent row, not combined maxima.

## Freshness and attribution

Latest is the latest observed row in scope, including invalid rows. Show last
valid timestamp separately; never use it to make an invalid latest observation
look healthy. Age is relative to min(requested end, frozen observed-at time).
Default stale threshold is 15 minutes; adjust when a source's emission cadence
is known and disclose the chosen threshold and basis. A historical sample is
not stale merely because the report is generated days later.

All full-AOS names remain separate. Missing/whitespace-only AOS rows are
unassigned coverage, not one synthetic server. An exact AOS filter may exclude
them; also show environment-level missing-AOS counts before that filter.
Observed nodes do not equal complete inventory. There is no job attribution,
task occurrence, completion metric, or lifecycle-derived concurrency here.

## Findings and next actions

- **FACT**: specific snapshot/bucket min, sample counts, null coverage, invalid
  reason, or latest-time age. Qualify approximate percentile statistics.
- **CORRELATED**: independently verified queue pressure or impact in the same
  full-AOS/window. Queue counts alone do not prove a capacity bottleneck.
- **HYPOTHESIS**: reservation overlap, demand pressure, collection gaps, or
  misconfiguration needing independent verification.

Use HIGH confidence for a verified observation with complete retrieval; MEDIUM
for observations limited by missing fields/sampling/ambiguity; LOW for
unsupported causal explanations. Observation confidence is not cause confidence.
Default severity is info; warning requires stated pressure or substantial
quality evidence. Critical requires separately documented impact/SLO breach,
never a negative value alone.

Investigate the same full AOS and UTC window, confirm emission cadence and
reservation execution semantics, and inspect independent queue/throttle/resource
evidence if needed. Do not automatically increase threads, change priorities,
modify reserved capacity, or restart servers. Keep this report focused; other
investigations are recommendations, not required skill dependencies.
