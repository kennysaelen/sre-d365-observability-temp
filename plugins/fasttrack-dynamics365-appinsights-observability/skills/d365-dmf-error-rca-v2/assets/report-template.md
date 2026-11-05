# DMF error root-cause analysis

## Scope

| Field | Value |
|---|---|
| Telemetry source | |
| Error-analysis window (UTC) | |
| Outcome-query window (UTC) | |
| Observation cutoff | |
| Environment filter | |
| Entity/project/execution filters | |
| Result limit or truncation | |

## Executive finding

In no more than five sentences, state the error scope, dominant signature, affected executions, verified outcomes, strongest root-cause evidence, confidence, and immediate next action.

## Impact

| Metric | Value |
|---|---:|
| Error events | |
| Distinct executions | |
| Entities | |
| Data projects | |
| Legal entities | |
| AOS role instances | |

Clarify whether error events can be interpreted as failed records.

## Top data projects

| Data project | Error events | Executions | Entities |
|---|---:|---:|---:|

## Top entities

| Entity | Error events | Executions | Projects |
|---|---:|---:|---:|

## Recurring signatures

| Category | Normalized signature | Error events | Executions | Entities | Projects | Sample message |
|---|---|---:|---:|---:|---:|---|

Normalization applied: email addresses and sensitive tokens are redacted; GUIDs become `<guid>`, hexadecimal values become `<hex>`, numeric identifiers of six or more digits become `<id>`, and signatures are capped at 500 characters. Sample messages must be redacted.

## Error trend

Include the time-series chart or table, bin size, zero-filled median, heuristic threshold, and flagged buckets.

## Highest-impact executions

| Execution ID | Activity ID (source) | Error events | Outcome | Staging error rate | Target error rate | Evidence |
|---|---|---:|---|---:|---:|---|

Use `Unavailable` when the outcome or denominator cannot be verified.

## Root-cause assessment

| Finding | Evidence label | Confidence | Supporting evidence |
|---|---|---|---|

## Error detail

| Timestamp (UTC) | Execution ID | Entity | Project | Legal entity | Category | Redacted message |
|---|---|---|---|---|---|---|

State when the table is truncated.
Confirm that messages were redacted before inclusion.

## Limitations

List missing telemetry, sampling, ingestion delay, retention boundaries, query failures, unavailable correlation sources, and denominator limitations.

## Recommended actions

List prioritized actions. Identify a representative execution for `d365-dmf-execution-drilldown-v2` when deeper analysis is required.
