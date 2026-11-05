# Throttling evidence and interpretation

## Dashboard compatibility

The supplied dashboard counts `BatchTaskStart` and `BatchThrottled` events in
10-minute bins. Preserve its two display labels but always explain:

| Display series | Meaning | Not a measure of |
|---|---|---|
| Running tasks | Observed task-start events in the bin | Simultaneously executing tasks, successful tasks |
| Throttled tasks | Observed throttle events in the bin | Distinct tasks, failed tasks, unique throttle episodes |

One task can encounter multiple throttle events. A throttle can occur before
any start. These counts are not a common population; never calculate
`throttles / (starts + throttles)` as a throttle rate, success rate, or SLA.
Identical `eventId` values such as `Batch00005` identify the event type and must
not collapse separate events.

## Resource context, not automatic trigger identification

`BatchThrottled.InfoMessage` contains:

| Property | Report column | Meaning supported by supplied schema |
|---|---|---|
| `CurrentMachineCpu` | CPU | Machine CPU usage context at emission |
| `CurrentMachineMemory` | Memory | Machine memory usage context at emission |
| `CurrentSqlDtu` | SQL DTU | SQL DTU usage context at emission |

The schema types are strings and units are unspecified. Preserve numeric
precision rather than rounding before interpretation. Do not append `%`, GB,
or a percent-of-limit interpretation until supported by authoritative runtime
documentation or confirmed source configuration. Values that happen to lie
between 0 and 100 are not by themselves proof of percentage units.

Default each event's resource assessment to **Trigger not established**.
If units and applicable thresholds are established, record their provenance
in coverage/findings and list every exceeding resource (CPU, memory, SQL DTU,
or multiple) as **CORRELATED: threshold exceeded**. Do not call the numerically
largest resource the trigger. Missing one metric does not exonerate it.

Only an explicit runtime signal with documented trigger semantics supports a
FACT about cause. The supplied event has no such dedicated field. Do not invent
`ThrottleReason`, Boolean trigger flags, or threshold fields. Never copy raw
`EventMessage` to the response in an attempt to explain a cause.

## Data-quality states

- `InfoStatus=Missing`: no InfoMessage.
- `InfoStatus=Malformed`: supplied InfoMessage is not a JSON object.
- `InfoStatus=Parsed`: a JSON object was parsed; individual metrics may still
  be absent or invalid.
- Metric status `Present`: finite, nonnegative numeric value, including zero.
- `Missing`, `NonNumericOrNonFinite`, or `Negative`: render unavailable and
  retain the status; never coerce to zero or silently clamp.
- `ItemCount > 1`: evidence that retained rows represent sampled/aggregated
  items. Report observed-row counts, not reconstructed task counts.
- `ItemCount` absent or equal to one does not prove lossless telemetry.

The event table, full-window summary, timeline, and per-AOS summaries must use
the same scope. Never infer zero throttle events for a job from `BatchJobId=0`.
Do not roll missing AOS into a real named server. Unknown environment rows are
excluded with disclosure, not pooled with the selected environment.

## Evidence, confidence, and severity

**FACT** is reserved for observations: event counts, timestamps, metric values,
availability, and confirmed identifiers. **CORRELATED** describes matching
resource observations or verified threshold exceedances at throttling time.
**HYPOTHESIS** describes possible causation or workload impact requiring
independent evidence.

Use HIGH confidence for the stated observation when queries completed with
adequate scope and coverage; MEDIUM when limited by missing fields or sampling;
LOW for causal theories without independent evidence. Confidence in an observed
event does not imply confidence in its cause.

Severity defaults to info for isolated observations and coverage limitations.
Use warning for repeated bursts or significant resource pressure with supporting
evidence. Use critical only with documented sustained impact or an applicable
SLO breach; raw counts alone are insufficient. State the actual threshold,
window, impact evidence, and limitations for elevated severity.

## Operational next actions

For CPU or memory correlations, investigate the same full AOS instance and
time window. For SQL correlations, inspect authorized database telemetry for
that environment and interval. For unexplained throttling, obtain documented
trigger semantics and source sampling/configuration information. These are
recommendations only: do not change batch threads, priorities, resource sizing,
or throttling controls automatically.
