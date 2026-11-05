# Interpreting completed-week page-view comparisons

## What the metrics mean

The metric is reported page-view duration. Confirm its emitter semantics
before calling it form-open, browser-load, or server processing time.
Only finite durations greater than zero enter percentile calculations.
Missing/unparseable, zero, and invalid durations remain observed-view coverage.
Do not remove slow outliers or duplicate-looking emissions without an
explicit, separately justified telemetry policy.

Kusto percentile estimates are approximate. Median and p95 describe retained
positive samples, not all real application operations. Do not multiply samples
by `itemCount`/`ItemCount`, invent weighted percentiles, or use weight estimates
to meet sample thresholds. Unknown weights and sampling indications are
reported explicitly. No known sampling is not proof of complete collection.

## Time and signs

Use disjoint Monday 00:00 through next Monday 00:00 UTC intervals, with the
upper boundary excluded. Four weeks produce W1-W2, W2-W3, and W3-W4 comparisons.
A week can be a member of two adjacent comparisons, but its observations are
not pooled into overlapping rolling means. Monday boundaries are computed
from a frozen observation clock, not a changing `now()` per query.

For positive medians:

`deltaMs = currentMedianMs - previousMedianMs`

`deltaPct = 100 * deltaMs / previousMedianMs`

Use the same formulas independently for p95. Positive means slower; negative
means faster. An absent or nonpositive denominator yields null, not infinity
or a 100% claim. Display ms throughout, or convert all displayed values to
seconds consistently and disclose that conversion. Never mix threshold units.

## Sample and threshold heuristics

Default minimum: 30 retained positive durations in each week. The median
candidate rule is `abs(deltaMs) >= 500 AND abs(deltaPct) >= 20`. The same
default numbers apply to the separate p95 candidate rule. Settings can differ
for median and tail. Publish all overrides. A 30-sample p95 can be noisy and
does not gain statistical significance from passing the median sample gate.

These are investigation heuristics, not SLO compliance, confidence intervals,
causal tests, or statistically significant regressions. Values below either
threshold mean "no threshold signal in this eligible comparison," not stable
performance, no user impact, or proof of no regression. A page with fewer
samples is unassessed, not healthy.

## Comparability evidence

Set Comparable only when evidence supports the same page semantics, duration
units/encoding, relevant client cohort, and sufficiently comparable collection
in both weeks. Include evidence for retention and known collection gaps,
instrumentation/SDK changes, sampling policy/rates, and ingestion maturity.
Expected low/no traffic is not itself a telemetry outage. An active-hour
count or first/last observation is only an observation pattern, never proof
that collection was healthy during every empty hour.

App version, SDK version, browser, and client type changes are evidence to
investigate. A new app version is a correlation, not a demonstrated cause.
Version changes may be precisely what the comparison investigates; explain
why collection semantics remain comparable rather than automatically rejecting
every deployment. Mixed/missing dimensions weaken inference. App-wide mix
summaries do not prove page-specific comparability: drill into the exact
page/role using the same source prefix when needed.

The evidence table is explicit and sparse. Unlisted page/pairs are Unknown;
never promote them because a query succeeded, both weeks have 30 samples, or
the default evidence table is empty. Duplicated/invalid evidence rows fail the
gate and must be corrected. Evidence notes must be sanitized.

| Evidence state | Permitted assessment |
|---|---|
| Comparable | Median regression/improvement or p95 tail regression/improvement if that metric passes both thresholds |
| Unknown | Exploratory potential median/tail signal only; no confirmed direction or coverage claim |
| NotComparable | Numeric deltas may be shown as descriptive, but suppress directional assessment |

Even a Comparable heuristic regression is an observed distribution change,
not a root cause or a claim of statistical significance.

## Exhaustive page/pair accounting

Create the key union from **all named observed views** across the entire
interval, never from the positive-duration cohort. Cross it with every week.
A key appearing only in W4 still has statuses for W1-W2 and W2-W3.

Apply this precedence before median/tail interpretation:

1. `AbsentBoth`: neither week has observed views for this key.
2. `NewlyObserved`: previous has none, current has observed views.
3. `NoLongerObserved`: previous has observed views, current has none.
4. `NoPositiveBoth`, `NoPositivePrevious`, or `NoPositiveCurrent`: both weeks
   have observations but one/both lack usable duration.
5. `InsufficientPositiveSamples`: positive counts exist but fail the threshold.
6. Eligible sample pair: use comparability and metric-specific thresholds.

Newly observed does not mean a new feature was released. No longer observed
does not mean usage stopped. Either can reflect collection/retention gaps,
renames, role changes, filtering, or actual usage changes. Never match renamed
keys by guesswork. They remain separate unless a verified mapping is supplied.

Median and p95 have separate labels, so median may have no threshold signal
while the tail regresses. Preserve both results. Every page/pair gets exactly
one joint status-rollup row contribution; do not add independent median and
tail totals and call the result a page count.

## Whole-population and exclusion accounting

Report keys, key-week cells, key-pair cells, and counts for every actual joint
status, plus zero total comparisons when there are no named keys. Include
observed-view, positive, zero, missing/unparseable, and invalid duration counts.
Also report missing page names, unknown roles, synthetic exclusions,
out-of-environment/missing/conflicting environment exclusions, sampling
indicators, unknown weights, and failed slices at their proper scope.

The weekly population query covers **all PageRows**, including missing names;
the page-comparison grid covers NamedPages only. Label those cohorts and
reconcile their difference. Scope diagnostics on ResourceRows are not traffic
attributed to the selected environment. Do not conflate them.

An optional application duration trend must calculate median/p95 from the
entire raw positive population in each week. Averaging page medians, p95
values, or signed page-change percentages is invalid. Whole-population
percentiles can change because page mix changed; they do not establish
within-page regressions. Raw duration sums, if used separately, are not
elapsed wall-clock time or a backend service-load measure.

## Output and failure rules

- Prioritize the final completed-week pair, but show all weeks and all adjacent
  status summaries. Preserve zero-view weeks with null durations.
- Rank all candidates deterministically before limiting display; distinguish
  comparable results from exploratory candidates. Disclose displayed versus
  total candidate counts.
- Reconcile disjoint page partitions. Do not aggregate percentiles across
  sample/time slices. A page's entire weekly sample set stays on one partition.
- A failed/truncated query is unavailable. Missing partition data cannot be
  filled with zeros, used for global ranks, or labeled complete.
- Escape report labels literally. Do not return raw user/session IDs, URLs,
  property bags, record titles, IPs, or sensitive dimension values.
- If charts fail, preserve the complete report in Markdown, including UTC
  week dates, sample counts, nulls, signed deltas, and evidence labels.
- Separate facts, correlations, hypotheses, and next checks. Page views alone
  cannot prove an X++ method, SQL query, AOS, network, or infrastructure cause.
