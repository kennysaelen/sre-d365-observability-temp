# Duration interpretation

- **FACT:** directly observed pair, timestamp, count, duration, rate, or status.
- **CORRELATED:** another workload or resource signal overlaps in time and scope.
- **HYPOTHESIS:** an explanation requiring execution drilldown, SQL, batch, dependency, or platform evidence.

Use `HIGH` confidence only when the slow phase or dependency is directly identified. Use `MEDIUM` for multiple aligned signals and `LOW` for duration or timing alone.

Long duration with normal per-record efficiency usually indicates high volume. Long duration with poor per-record efficiency is a stronger optimization signal. Cumulative duration is additive workload and may exceed the analysis window when executions overlap.
