# Interpret usage and duration

## What the evidence supports

| Observation | Supported statement | Unsupported conclusion |
|---|---|---|
| Retained page-view rows | Observed usage in this source/window | All real opens or all configured forms |
| Distinct telemetry user/session IDs | Observed identity counts with coverage | Unique people, licensed users, or adoption rate |
| Positive page-view duration | Distribution of measured durations | Server execution, time to interactive, or a universal form-open metric |
| High p95 | Long upper tail among retained valid samples | Regression without a comparable earlier period |
| Large view share | Large share of retained events | Business criticality or unsampled traffic share |
| No views | No retained observations in the queried scope | Unused application, downtime, health, or zero real traffic |

FACT describes retrieved aggregates with exact scope and units. CORRELATED
describes coincident volume/duration behavior without causality. HYPOTHESIS
needs corroborating telemetry, not a confident SQL/X++/network diagnosis.

Use unweighted retained observations consistently. If sampling weights vary,
relative retained-view rankings can differ from true traffic rankings. Unknown
weight metadata and excluded synthetic views are coverage limitations.
Do not "correct" duration percentiles or identity counts with ItemCount.

Zero/missing/invalid duration rows count for usage but not duration. A page with
many views and no eligible duration is not fast. Report its availability gap.
Default N>=20 for the slow-page list reduces sparse ranking noise but does not
establish confidence intervals or statistical significance. p95 is especially
unstable at small N; show N next to it.

Per-page names must be stable and safe to expose. Missing names or dynamic
record titles are not automatically grouped into one form. Role names remain
separate component keys; role instances are not necessarily F&O AOS servers.
Do not exclude F&O system forms such as SysOperationSandboxForm,
SysBoxForm, and Dialog globally. Any requested exclusion must be explicit,
application-appropriate, and reflected in coverage.

Do not infer a trend from a single overall average. Use the separate weekly
regression skill for equal completed weeks and the single-form drilldown for
browser/version/component context. Optional charts share the table population,
units, filters, and counts.

## Output and next actions

Keep a proportionate overview rather than a root-cause report. Prioritize
high-usage pages with elevated measured durations as investigation candidates,
not proven sources of business harm. Identify exact page/role keys and suggest
the focused skill. Disclose missing schema, late ingestion, coverage, limits,
and chart failures; no observations are not evidence of health.
