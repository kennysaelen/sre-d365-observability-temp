# DMF error interpretation guide

## Evidence labels

- **FACT:** directly observed event, count, status, identifier, timestamp, or resource measurement.
- **CORRELATED:** an independent signal overlaps in time and scope but does not prove causation.
- **HYPOTHESIS:** a plausible explanation requiring another query, log, or system owner to confirm.

## Confidence

- **HIGH:** direct exception, validated status/count evidence, dependency failure, SQL evidence, deployment change, or platform event explains the failure.
- **MEDIUM:** multiple independent signals align, but the failing operation is not directly observed.
- **LOW:** based mainly on timing, message wording, missing telemetry, or one ambiguous signal.

## Outcome rules

| Outcome | Required evidence |
|---|---|
| Completed | Terminal success event and successful staging/target statuses. |
| Partial success | Some records succeeded while staging or target errors are nonzero, or a terminal status explicitly indicates partial completion. |
| Failed | Terminal failure, fatal error with no later recovery, or failed staging/target status. |
| Unfinished | Start exists but no terminal event is present by the observation cutoff. |
| Indeterminate | Required start, terminal, count, or status telemetry is missing or contradictory. |

## Error-pattern interpretation

| Pattern | Likely scope | Required follow-up |
|---|---|---|
| One signature, one execution, one entity | Isolated data or execution issue | Drill into the execution and affected staging records. |
| One signature across many executions of one entity | Entity mapping, validation, or source-data pattern | Review entity configuration and representative records. |
| One signature across projects and entities | Shared platform, integration, authentication, or capacity issue | Correlate dependencies, SQL, AOS, and deployment telemetry. |
| Many signatures beginning simultaneously | Shared outage, deployment, malformed source batch, or telemetry change | Correlate the first spike with platform and change events. |
| Error events followed by successful terminal status | Retry or recovered/partial condition | Report the errors and final outcome separately. |

## Root-cause categories

Use a category only when supported by evidence:

- Source-data validation
- Entity mapping or configuration
- Security or permission
- Duplicate or conflicting data
- Business-logic validation
- Integration or dependency
- SQL or resource contention
- AOS or batch-process interruption
- Deployment regression
- Unknown

## Privacy and evidence handling

- Error messages may contain customer record values, email addresses, URLs, tokens, or identifiers.
- Redact sensitive values in narrative and exported tables while retaining enough structure to identify the signature.
- Do not expose `user_Id` unless identity attribution is necessary and authorized.
- Preserve the unredacted evidence only in the governed telemetry system, not in generated reports.

## Limitations to disclose

- Query source, time range, and observation cutoff.
- Sampling, ingestion delay, retention boundary, or row-limit effects.
- Missing event or custom-dimension fields.
- Whether non-error DMF events were available for outcome verification.
- Whether deployment, SQL, dependency, AOS, and batch telemetry were available for correlation.
