# DMF interpretation guide

## Evidence levels

Use these labels in findings:

- **FACT:** directly present in telemetry, such as an event, timestamp, count, status, identifier, or node.
- **CORRELATED:** independently observed telemetry overlaps in time and scope but does not prove causation.
- **HYPOTHESIS:** a plausible explanation that requires another query or data source.

## Timeline interpretation

- A long gap after a staging event localizes elapsed time to the following phase, but it does not identify the internal operation.
- Repeated events may indicate retries, polling, partitioned processing, or duplicate telemetry. Determine which pattern applies before counting attempts.
- An error followed by successful terminal telemetry may represent a recovered retry or partial failure.
- A start event without a terminal event may mean the run is unfinished, telemetry was lost, retention excluded the terminal event, or the process terminated.

## Count interpretation

- Treat missing counts as unavailable, not zero.
- Do not add staging and target counts together; they describe different phases.
- Compare `TotalNumberOfRecords` with staging/target counts only after confirming they share the same entity and execution.
- Partial success requires evidence from statuses, counts, or subsequent successful events.

## Root-cause confidence

- **HIGH:** direct error, stack, dependency failure, SQL evidence, or platform event explains the failed or slow phase.
- **MEDIUM:** multiple independent signals align in time and scope, but the exact causal operation is not directly observed.
- **LOW:** the conclusion is based primarily on timing, absence of events, or a single ambiguous field.

## Common follow-up paths

| Signal | Follow-up |
|---|---|
| Long gap with high dependency duration | Query the dependency type, target, result code, and operation correlation. |
| Long gap with SQL symptoms | Query SQL duration, blocking, throttling, and database resource pressure for the exact execution window. |
| Unfinished run on a replaced AOS node | Check process restarts, crashes, memory pressure, and batch task state. |
| Multiple executions on the same node/entity | Quantify overlap and resource contention before claiming interference. |
| Staging errors | Retrieve each `DMFError`, `StagingRecId`, and message; group identical errors without losing counts. |
| Target errors after successful staging | Focus on entity validation, target writes, business logic, and downstream dependencies. |

## Reporting limitations

Always disclose:

- Query time range and observation cutoff.
- Data source and table.
- Sampling, truncation, or retention limitations.
- Missing start, end, status, or count fields.
- Whether related batch, SQL, dependency, or platform telemetry was available.
