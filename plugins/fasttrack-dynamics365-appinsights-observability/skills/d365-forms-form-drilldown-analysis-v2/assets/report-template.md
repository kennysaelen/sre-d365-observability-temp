# Page/form drilldown: {{SafeExactPageName}}

## Summary

{{OneParagraphOutcomeIncludingLimits}}

Status: {{CompletePartialNoObservationsOrBlocked}}
Measurement: {{ReportedPageViewDurationOrConfirmedFormOpenWithCitation}}
Observation cutoff: {{ObservedAtUtc}}

## Scope and provenance

| Setting | Value |
|---|---|
| Source/tool/table | {{AuthorizedSourceAndSelectedTable}} |
| Schema and duration encoding evidence | {{RuntimeSchemaAndEmitterEvidence}} |
| Application resource / environment | {{SafeScopeLabelsAndVerification}} |
| Exact page / selected role | {{SafeExactPageAndRoleOrExplicitUnknown}} |
| Role discovery | {{ObservedRoleBucketCountAndResolution}} |
| UTC interval | [{{StartTimeUtc}}, {{EndTimeUtc}}) |
| Bin widths | {{HourlyBinWidth}} and UTC 1d |
| Synthetic policy | {{KnownSyntheticPolicy}} |
| Duration screening threshold | {{ThresholdMs}} ms; screening only, not SLA |
| Query validation / execution | {{ValidatedExecutedUnavailableWithReasons}} |
| Result budgets | {{RowsCellsAndHostLimits}} |

Do not include tenant IDs, user/session IDs, raw URLs, or resource strings
that disclose restricted identifiers. Scope labels must still permit an
authorized operator to identify the selected source safely.

## Full selected population

| Metric | Value |
|---|---|
| Retained observed page views | {{ObservedViews}} |
| Positive finite duration samples | {{PositiveDurationViews}} |
| Mean / median / p95 / maximum | {{MeanMs}} / {{MedianMs}} / {{P95Ms}} / {{MaxMs}} ms |
| Above screening threshold | {{SlowViews}} / {{PositiveDurationViews}} eligible samples |
| Observed distinct users / sessions | {{ObservedUsersOrUnavailable}} / {{ObservedSessionsOrUnavailable}} |
| Missing user / session rows | {{MissingUserViews}} / {{MissingSessionViews}} |
| Zero / missing-unparseable / invalid durations | {{ZeroViews}} / {{MissingViews}} / {{InvalidViews}} |
| Sampling-indicated / unknown-weight rows | {{SamplingIndicatedViews}} / {{UnknownWeightViews}} |
| First / last retained observation | {{FirstObservedOrNone}} / {{LastObservedOrNone}} |

These are retained telemetry observations, not authenticated people,
business-operation throughput, or sampling-adjusted duration statistics.
Missing durations do not reduce observed usage. Null metrics mean unavailable,
not zero. Whole-window distinct counts are not sums of grouped counts.

## Coverage and exclusions

{{TableOfSchemaOptionalColumnsDurationIdentityAndDimensionCoverage}}

{{ResourceLevelEnvironmentConflictAndMissingCountsWithSeparateDenominators}}
{{ExactPageRoleBeforeSyntheticAndExcludedSyntheticCounts}}
{{KnownRetentionCollectionOutagesLateArrivalAndInstrumentationLimits}}
{{QueryFailuresTruncationAndUnavailableIntervalsOrNone}}

Resource-wide exclusion counts are not selected-environment/page counts.
Successful queries and first/last events do not prove continuous collection.

## Hourly and daily trends

{{HourlyDurationChartOrExplicitUnavailableReasonAndBoundedTable}}
{{HourlyObservedViewsChartOrExplicitUnavailableReasonAndBoundedTable}}
{{DailySummaryTable}}

Axes: UTC time; duration in ms; observed page-view counts on a separate chart.
Retain null-duration gaps and partial-edge bucket labels. Empty successful
buckets mean 0 observed views, not confirmed zero actual activity. Failed
buckets are unavailable. Mark low-volume duration buckets and chart omissions.

## Browser, client, instance, and version context

{{OneTablePerAvailableDimensionIncludingMissingBucket}}

Each table: dimension value, missing flag, observed views, eligible durations,
mean/median/p95 ms, share of full cohort, full bucket cardinality, displayed
views, and omitted views. State ranking criterion and top-N limit.
Include browser, client type, role instance, app version, SDK version, and
the frozen role/component context. Report unavailable/unsafe dimensions.
Do not sum across dimensions or label RoleInstance an AOS without evidence.

## Limited slow-observation samples

Diagnostic selection: top {{ReturnedSamples}} of {{EligibleSlowViews}}
retained observations above {{ThresholdMs}} ms; limit {{SampleLimit}}.

| UTC timestamp | Reported duration (ms) | Safe browser | Safe client type | Safe app version | Safe SDK version |
|---|---:|---|---|---|---|
| {{EventTime}} | {{DurationMs}} | {{Browser}} | {{ClientType}} | {{AppVersion}} | {{SDKVersion}} |

No identifiers, URLs, property bags, IPs, record titles, or payloads.
This deliberately slow-biased sample is not a representative distribution.

## Findings and confidence

| Label | Finding | Evidence / denominator / coverage | Confidence and justification |
|---|---|---|---|
| {{FACT_CORRELATED_HYPOTHESIS}} | {{Claim}} | {{MeasuredEvidenceAndLimits}} | {{HIGH_MEDIUM_LOWAndWhy}} |

{{LowVolumeOutliersVersusRepeatedWithinWindowPatternAssessment}}
No weekly regression or SQL/X++/browser/server cause is established by
page-view duration alone.

## Next steps

| Priority | Next action | Evidence needed | Owner or decision |
|---|---|---|---|
| {{Priority}} | {{BoundedAction}} | {{SourceWindowAndQuestion}} | {{OwnerOrInputNeeded}} |

{{RecommendWeeklyRegressionSkillOnlyIfPreviousPeriodQuestionApplies}}
{{UnresolvedInputsAndExplicitNoAutomaticScopeWidening}}
