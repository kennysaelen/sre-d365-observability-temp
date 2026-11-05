# D365 batch workload report

## Scope

| Field | Value |
|---|---|
| Telemetry source | |
| Source dialect/table family | |
| Environment | |
| Analysis window (UTC) | |
| Baseline window (UTC) | |
| Pairing buffer | |
| Ingestion lag | |
| Filters | |

## Executive summary

Write three to five sentences. State the dominant workload observation, the
strongest health or risk signal, whether evidence is sustained, and the most
important next action. Include material coverage limitations.

## Telemetry coverage

| Signal | Events/samples | First seen | Last seen | Missing key fields | Maximum sample gap | Assessment |
|---|---:|---|---|---:|---|---|

## Headline KPIs

| KPI | Value | Denominator/definition | Baseline | Severity |
|---|---:|---|---:|---|
| Correlated task executions started | | | | |
| Completed task executions | | | | |
| Failed task executions | | | | |
| Apparently unfinished tasks | | | | |
| Ambiguous/truncated task pairs | | | | |
| Throttling events | | | | |
| Peak queue depth | | | | |
| Peak effective thread utilization | | | | |

## Slowest completed tasks

| Class/job | Job ID | Task ID | Activity ID | AOS | Start | End | Duration | Emitted duration | Difference | Priority |
|---|---|---|---|---|---|---|---:|---:|---:|---|

## Batch-job workload

| Job/class | Tasks | Completed | Failed | Unfinished | Cumulative duration | Median | p95 | Maximum |
|---|---:|---:|---:|---:|---:|---:|---:|---:|

## Thread capacity and utilization

| AOS | Samples | Max-thread range | Reserved-thread range | Effective-capacity range | Median current | p95 utilization | Peak utilization | Invalid samples |
|---|---:|---:|---:|---:|---:|---:|---:|---:|

Add a utilization time chart when supported.

## Queue depth

| Queue | Samples | Median | p95 | Peak | Nonzero samples | Longest sustained sequence | Assessment |
|---|---:|---:|---:|---:|---:|---|---|

Add a queue-depth time chart when supported.

## Failed task executions

| Time | Class/job | Job ID | Task ID | Activity ID | AOS | Exception type/source | Evidence |
|---|---|---|---|---|---|---|---|

Do not include raw messages or call stacks.

## Supporting exception telemetry

| Exception signature | Events | Affected activities | First seen | Last seen | Correlation note |
|---|---:|---:|---|---|---|

Do not add these rows to the failed-task total unless uniqueness is proven.

## Throttling

| Time bucket | Throttles | Task starts (throughput) | Affected AOS | CPU coverage | Memory coverage | SQL DTU coverage |
|---|---:|---:|---:|---:|---:|---:|

Add a throttling trend when supported. Do not label task starts as concurrent
running tasks.

## Current versus baseline

| Metric | Current | Previous | Absolute change | Percentage change | Coverage note |
|---|---:|---:|---:|---:|---|

## Findings and actions

| Finding | Evidence label | Confidence | Severity | Evidence | Next action |
|---|---|---|---|---|---|
