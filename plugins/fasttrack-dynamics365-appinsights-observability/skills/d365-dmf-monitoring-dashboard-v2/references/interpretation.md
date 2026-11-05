# Dashboard interpretation

Use:

- **FACT** for observed events, counts, durations, statuses, and coverage.
- **CORRELATED** for independent signals aligned in time and scope.
- **HYPOTHESIS** for causal explanations requiring focused analysis.

Severity is not based on raw volume alone. Evaluate affected executions, trend baseline, duration/volume context, silence, and data quality. Unknown event types and missing identifiers reduce confidence.

Cumulative duration is additive workload, not exclusive wall-clock occupancy. Error events are not failed executions. Unfinished starts require ingestion-lag and silence handling.
