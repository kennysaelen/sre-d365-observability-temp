# Batch Task Priority Distribution

Skill: `d365-batch-task-priority-distribution-v2`.

Returns a pie chart of **distinct observed batch jobs grouped by their task
priorities**, plus a two-to-three-sentence summary. Defaults to the last
24 hours and automatically selects Application Insights or Log Analytics.

Each nonzero job ID counts once. Multiple known priorities produce a
**Mixed priorities** slice; missing task priorities produce
**Unknown/incomplete**. Raw numeric priority codes are retained unless an
authoritative label mapping is available. This is not a task-event count,
job-execution count, or inventory of all configured batch jobs.

Example: "Show the batch job priority distribution for environment
<environment-id> over the last 24 hours."

Provide the general and batch Application Insights schema JSON knowledge files
and a supported query connection. `PlotPieChart` produces the visualization;
if unavailable, the skill returns a compact table instead. No HTML or report
framework is required.

Package: `SKILL.md` contains the workflow; `references/queries.md` contains
source adapters and the shared aggregation.

