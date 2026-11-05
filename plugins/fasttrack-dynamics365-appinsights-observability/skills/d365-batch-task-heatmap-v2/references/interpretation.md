# Heatmap interpretation

## Definitions

**Cell = number of retained BatchTaskStart rows for one AOS instance during
one UTC bucket.** Recurring starts for the same persistent task ID remain
separate observations. Repeated emissions or retries may increase counts;
without a verified unique occurrence key do not deduplicate them.

Full AOS instance names identify rows. Never collapse instances by role name,
prefix, or string splitting. Unknown AOS is reported separately.

Zero-filled cells mean **no task-start events observed**. They do not prove
idle capacity, node availability, absence of running tasks, or workload health.
The observed-node list is not a fleet inventory. An AOS absent for the entire
window cannot be discovered from task starts alone. A server first observed
late may not have existed in earlier zero-filled bins.

An empty source query is not equivalent to an unavailable query. Missing data
and failed retrieval must remain unknown/null, not zero.

## Reading the visualization

- Darker cells mean more task starts, not greater CPU use or longer duration.
- A long-running task contributes to its start bucket only, even if it runs
  for hours. Do not repeat it across subsequent buckets.
- A node handling many short tasks can have more starts than one handling
  fewer expensive tasks. Unequal counts do not prove load imbalance.
- A burst may reflect scheduling, recurrence, workload affinity, or available
  nodes. Attribution requires independent scheduler/capacity evidence.
- Partial edge buckets cover fewer minutes; do not compare their counts as if
  they were full-hour rates.
- Keep full dates; never merge 09:00 across different days into a single
  hour-of-day cell unless that is a separately requested analysis.
- Use a shared sequential scale from zero through the global peak for all
  panels. Per-row/per-panel auto-scaling makes unlike counts appear equal.

## Evidence and severity

**FACT**: scoped observed counts, node identities, peak cells, sampling and
missing-field coverage.
**CORRELATED**: independent evidence aligned to the same node and time.
**HYPOTHESIS**: potential scheduling or performance explanation.

Confidence: HIGH for reliable observed counts with clear scope and adequate
coverage; MEDIUM for sampling, partial metadata, or collection uncertainty;
LOW for unsupported causal explanations. Confidence in counts does not imply
confidence in why they differ.

Severity defaults to info. High start counts are not automatically warning or
critical. Elevated severity requires independent impact evidence or a documented
SLO breach. Do not label an absent/zero row success.

## Follow-up

For actual concurrency and effective capacity, use a batch workload overview
with `BatchThreadInfo`. For resource-pressure correlations, use a throttling
analysis with same-node/time evidence. For slow jobs, inspect verified task
durations. Do not change batch threads, priorities, affinity, or resource sizes
from a task-start heatmap alone.

