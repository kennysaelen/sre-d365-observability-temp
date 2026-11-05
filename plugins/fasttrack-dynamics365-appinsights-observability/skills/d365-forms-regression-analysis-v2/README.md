# D365 forms regression analysis v2

Standalone Azure SRE Agent skill for customer Application Insights `pageViews`
or Log Analytics `AppPageViews`. It does not use Microsoft-internal telemetry,
Copilot Studio, or agent configuration changes.

## Package

- [SKILL.md](SKILL.md): trigger, workflow, tool allowlist, and guardrails.
- [Source contract](references/page-view-source.md): self-contained local copy
  of the shared Forms source adapters, scope rules, units, privacy, and limits.
- [Queries](references/queries.md): exact assembly recipes, weekly grid,
  adjacent comparisons, population summaries, and optional chart data.
- [Interpretation](references/interpretation.md): heuristics and evidence rules.
- [Report template](assets/report-template.md): Markdown output and chart fallback.

Upload the entire package together. There are no runtime references to sibling
skills, external KQL functions, or invented physical datasets. The source
contract is maintained in sync with the other Forms v2 packages, but this
package runs independently once that local copy is present.

## Defaults

Freeze UTC observation time, end at the most recent Monday 00:00 UTC, and
start 28 days earlier. Analyze four completed Monday-Sunday weeks with three
nonoverlapping adjacent comparisons. For example, an observation clock of
`2026-09-19T23:41:02Z` gives `[2026-08-17T00:00:00Z,
2026-09-14T00:00:00Z)`, not a partial week ending September 19.

Allow explicit 2-12 completed UTC weeks; do not round supplied boundaries.
Retain all observed exact page/role keys, even if no positive duration exists.
Require 30 retained positive samples in both weeks for heuristic eligibility.
Median flags require both 500 ms absolute and 20% relative change; p95 is a
separate tail signal. A positive signed delta means slower.

Collection/cohort comparability defaults to Unknown. Only evidence-backed
Comparable page/pairs receive strong directional labels. Unknown comparisons
can expose exploratory potential signals. Thresholds are neither SLOs nor
tests of statistical significance. No flagged regression does not mean stable.

## Execution and validation

Select a customer query tool based on the authorized target. The source
contract chooses its table family from that tool and confirmed schema, not
from a user-selected dialect. Confirm resource/environment isolation, source
duration units, and complete time-range arguments before querying. Assemble
each query using the exact dependency matrix in `references/queries.md`.
Validate it with `ValidateQuery` when the target supports validation.

Run preflight before page-week grids; partition complete keys if needed.
Never draw whole-population conclusions from a top-N or truncated result.
Charts are optional; all required findings can be delivered using the template.
Do not return raw identity values, URLs, custom property bags, or record titles.

This package is authored and locally checked without accessing live customer
telemetry or deploying anything. Local structural and synthetic-model checks
are not service-side KQL compilation. Service validation, runtime schema
compatibility, and collection comparability must be established at execution.

Local authoring checks covered ASCII, Markdown links/fences, the exact
eight-tool frontmatter, and dependency closure for all 13 terminals with both
source adapters (26 assemblies). A synthetic model exercised 13 exact
page/role keys, 52 weekly cells, and 39 adjacent pairs, including empty input,
duration-less pages, unknown roles, boundary dates, both threshold gates,
tail-only signals, evidence states, and full-population reconciliation.
