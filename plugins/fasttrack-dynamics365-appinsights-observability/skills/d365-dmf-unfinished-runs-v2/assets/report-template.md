# DMF unfinished-run triage

## Scope

| Field | Value |
|---|---|
| Telemetry source | |
| Lookback start | |
| Query time | |
| Ingestion lag | |
| Observation cutoff | |
| Environment and filters | |
| Events missing execution ID | |
| Detail truncation | |

## Summary

| State | Severity | Runs |
|---|---|---:|

## Unfinished executions

| Severity | State | Progress observed | Project | Entity | Execution ID | Start | Last event | Elapsed min | Silent min | Errors | Batch job ID | Node |
|---|---|---|---|---|---|---|---|---:|---:|---:|---|---|

If no rows are found, state: "No DMF starts without a matching end were detected within the stated lookback and ingestion-lag boundary."

## Correlation

| Execution ID | Signal | Evidence label | Confidence | Evidence |
|---|---|---|---|---|

## Safe next actions

Prioritize verification of batch state, node health, current progress, partial commits, and idempotency. Do not issue cancellation or rerun actions.
