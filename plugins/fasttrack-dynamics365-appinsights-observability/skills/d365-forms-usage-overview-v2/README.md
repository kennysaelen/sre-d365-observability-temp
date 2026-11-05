# d365-forms-usage-overview-v2

Standalone Azure SRE Agent skill for observed Dynamics 365 page usage and
reported duration. It is not restricted to F&O and does not rely on internal
Microsoft telemetry. Default: seven days, hourly buckets, top 20 pages.

| File | Purpose |
|---|---|
| `SKILL.md` | Activation, workflow, completion requirements |
| `references/page-view-source.md` | Source adapters, scope, canonical rows, privacy and collection semantics |
| `references/queries.md` | KPI, ranking, and timeline query blocks with assembly recipes |
| `references/interpretation.md` | Evidence rules and limitations |
| `assets/report-template.md` | Markdown response contract and chart guidance |

Upload the complete package; tool declarations do not prove host attachment.
Verify the four resource-query bindings, `kusto_query` if used, `ValidateQuery`,
and chart schemas. `PlotBarChart` and `PlotLineChart` may not exist in an older
agent's tool list; use honest table fallback rather than invented calls.

The source reference cites Microsoft's public page-view model and AppPageViews
schema. Confirm runtime metadata, duration units, and emitter-specific meaning
for the selected application. No separately supplied reference file,
unpublished schema, or F&O-only property is required or assumed.

All three Forms packages carry an identical standalone copy of the source
reference; update those copies together. They have no runtime dependency on
one another. Follow-up skills are recommendations, not mandatory execution.

Output is Markdown with optional charts and supporting aggregates. The skill
counts retained events, not every real user action or all configured forms.
Page duration does not establish a performance regression by itself.

This package does not deploy anything, query a customer, or change the local
observability-agent draft. Registering/enabling Forms on that agent is a
separate explicit action.
