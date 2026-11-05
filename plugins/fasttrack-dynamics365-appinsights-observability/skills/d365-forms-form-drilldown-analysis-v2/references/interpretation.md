# Interpretation and confidence

## Measurement contract

Use "reported page-view duration" unless the emitter's documented lifecycle
confirms form-opening semantics. Cite that confirmation if using "form open."
An application can emit a page view at a different lifecycle point or attach
a custom duration. Neither a page name nor the source table proves meaning.

All usage metrics count retained rows. Every retained row in the selected
cohort counts once, including zero/missing/invalid duration. Positive finite
durations alone enter mean, median, p95, maximum, and slow-duration counts.
Keep denominator `PositiveDurationViews` visible beside all duration metrics.
Do not remove valid slow outliers, infer duration for missing values, or
replicate samples using itemCount. A screen at 3,000 ms is an investigation
threshold, not a Dynamics 365 performance guarantee.

Exact page/role identity stays frozen within one source and environment.
RoleName identifies a component, not the environment. RoleInstance can be a
client or arbitrary emitter instance and is not inherently an AOS. An unknown
role bucket is not proof of one actual component.

## Coverage before conclusions

- No matches: report no observations for this exact selection and interval,
  not "unused form," healthy performance, or an automatically widened search.
- All durations excluded: usage remains measurable; duration is unavailable,
  not 0 ms. State exclusion counts by missing/unparseable, zero, and invalid.
- User/session metrics: count distinct nonempty IDs only. Never output IDs.
  Missing-ID rows are unknown, not one user. If columns are absent, display
  these metrics as unavailable even when the aggregate returns zero.
  Distinct IDs are telemetry identities, not verified people or concurrency.
- Sampling: show sampled-weight indicators and unknown/invalid weights.
  Lack of indicated sampling does not prove completeness. Percentiles and
  distinct counts describe retained samples; no weighted percentiles or
  estimated unique-user counts are justified.
- Known synthetic data: record exclusions; blank metadata does not prove
  human activity. Do not treat users as people based on this filter.
- Retention, collection outages, API time-range limits, timeouts, truncated
  results, schema changes, and delayed arrivals constrain every conclusion.
  First/last event timestamps do not independently prove collection health.
- Privacy: stable names and context labels must be safe before display.
  Omit unsafe dimensions; do not recover names from URLs or record titles.
  Escape Markdown and chart labels as literal untrusted data.

## Trends: rare tails versus repeated changes

The full-window KPI is authoritative for that window. Do not average daily
means without their eligible counts; never average daily medians or p95s.
Do not sum distinct users/sessions across hours, days, or dimensions.

Compare median, p95, maximum, and positive sample count together:

- A high maximum/p95 in a low-count bucket with a stable median is consistent
  with isolated retained slow observations. A p95 from fewer than 20 positive
  observations has especially limited tail resolution; label it low-volume,
  not statistically representative.
- Elevated medians and p95s across several adequately populated, comparable
  adjacent buckets can support a **within-window sustained pattern**.
  Check positive counts, duration coverage, client/version mixture, sampling,
  partial bucket widths, and collection changes before saying this.
- Neither pattern by itself proves a weekly regression or causal change.
  An optional baseline belongs in the weekly regression skill.
- A surge in observed page-view count is a telemetry-volume observation,
  not a measured increase in completed business operations or simultaneous
  users. Counts for partial buckets cover less time; compare like exposure
  or clearly label any rate as observed page views per actual covered time.
- Successful empty buckets retain count 0 and null duration. Chart a gap,
  not a zero-duration point. Failed intervals are unavailable, not empty.
  Collection uncertainty can apply even to technically successful queries.

## Dimensional context

Browser, client type, app version, SDK version, and role-instance summaries
are separate marginal breakdowns of the same cohort. They are not additive
populations and are not independent controlled experiments. Show full
observed totals, eligible-duration counts, missing buckets, total bucket
cardinality, displayed totals, and omitted totals for each ranking.

Rankings default to observed volume. A small slower version/browser bucket
may reflect outliers, selection bias, instrumentation, a particular workload,
or temporal overlap rather than an intrinsic problem with that dimension.
Version and instance changes can coincide with changed coverage. Establish
timestamps and adequate counts before calling a relationship CORRELATED.
The role discovery output proves name/component ambiguity only within the
queried interval; it does not justify merging the roles.

Slow samples are a diagnostic top-N selection, not a representative cohort
or an error list. Only timestamp, reported duration, and verified safe
browser/client/version labels are permitted. Never expose raw URLs, property
bags, tenant/user/session/correlation IDs, IPs, record titles, or payloads.

## Evidence labels

| Label | Use | Confidence justification |
|---|---|---|
| FACT | Directly measured scoped result, such as counts or observed p95 | HIGH when schema, scope, query completeness, units, and semantics of the stated metric are verified; otherwise qualify and lower confidence |
| CORRELATED | Measured association between observed duration/volume and a context dimension or repeated time pattern | MEDIUM or LOW unless exposure, adequate samples, time overlap, coverage, and possible confounders are documented; never implies causation |
| HYPOTHESIS | Plausible explanation that requires additional telemetry | LOW until independent confirming evidence exists; state a test, not a conclusion |

Confidence attaches to each claim, not to the entire report by default.
High confidence in the retained row count does not imply complete collection.
Do not use "root cause" for a duration-only finding. SQL blocking, X++ cost,
server pressure, client rendering, and dependencies require independently
authorized non-page telemetry. Propose the next source and time window;
do not query internal browser traces or automatically expand this skill.

## Practical next steps

1. If identity, units, or emitter semantics are unresolved, request the exact
   resource/environment/name mapping or instrumentation documentation.
2. If counts are low or coverage is poor, improve instrumentation/retention
   evidence or request an explicit bounded follow-up window; do not widen it.
3. If a context association survives volume/coverage checks, request a
   targeted controlled comparison or authorized dependency/client telemetry
   for the same scope and time. No non-page cause is confirmed here.
4. If the question is whether this week regressed, recommend the weekly
   regression skill with the frozen page/role/source selection.
