# Weekly regression queries

## Assembly, gates, and naming

Read [page-view-source.md](page-view-source.md) first. `P` below means its
**Parameters + exactly one adapter + Normalization + Scope**, in that order.
Those named sections contain complete local bindings for `RawPages`,
`Candidates`, `ResourceRows`, `Scoped`, `PageRows`, and `NamedPages`.
They are NOT physical datasets. The only physical table in an assembled
query is the tool-selected `pageViews` or `AppPageViews`.

Every code block labeled Module is appended once, in the listed order, before
one terminal. Each terminal below has an exact recipe. No external function
or prior query session state is assumed. For discovery and source coverage
terminals, use the recipes in the source contract, not the comparison modules.
Replace all source tokens and freeze UTC bounds once. Escape KQL string
literals correctly; do not paste unescaped names or evidence into queries.

| Terminal | Exact recipe |
|---|---|
| Window/control gate | P + A + T1 |
| Cardinality preflight (each proposed partition) | P + A + B + T2 |
| Explicit exclusion totals | P + T3 |
| Evidence gate | P + A + B + D + T4 |
| Complete page-week timeline | P + A + B + C + T5 |
| All adjacent page/pair details | P + A + B + C + D + E + T6 |
| Joint status rollup, including empty pairs | P + A + B + C + D + E + T7 |
| Pair reconciliation totals | P + A + B + C + D + E + T8 |
| Final-pair median ranking | P + A + B + C + D + E + T9 |
| Final-pair p95 ranking | P + A + B + C + D + E + T10 |
| Whole-population weekly timeline | P + A + F + T11 |
| Version/client mix preflight | P + A + G + T12 |
| Version/client mix distributions | P + A + G + T13 |

Run T1-T4 before comparisons. A failing gate stops interpretation. Run T2
before C constructs any page-week grid. Run T12 before exporting T13.
An empty result after a failure is not an empty population. Schema, access,
retention, query truncation, partial failures, and API time-range limits must
be checked independently. Use `ValidateQuery` on assembled queries when
available, not on fragments containing unresolved local bindings.

## Frozen time parameters

The source Parameters block receives literal `_observedAtUtc`, `_startTime`,
and `_endTime` values. If the user supplies no interval, compute:

- `end = startofweek(observedAtUtc - 1d) + 1d`
- `start = end - 28d`

Kusto `startofweek()` starts on Sunday. Subtracting one day before calling it,
then adding one day, anchors to Monday. Do not use `startofweek(observedAtUtc)
+ 1d`, which points to a future Monday on Sundays. Do not replace supplied
explicit boundaries with these defaults. T1 rejects nonaligned, future,
partial-week, shorter-than-two-week, and longer-than-twelve-week windows.

Reference: https://learn.microsoft.com/en-us/kusto/query/startofweek-function

## Module A: Controls and UTC week spine

Use positive numeric thresholds; all settings are explicit and reported.
The chart-cell estimate later conservatively counts six output fields per
page/week row (page, role, time, two metrics, sample count).

```kusto
let _minimumPositiveSamples=30;
let _medianAbsoluteMs=500.0;
let _medianRelativePct=20.0;
let _p95AbsoluteMs=500.0;
let _p95RelativePct=20.0;
let _groupBudget=5000;
let _chartCellBudget=20000;
let _partitionCount=1;
let _partitionId=0;
let _topN=20;
let _completedBoundary=startofweek(_observedAtUtc - 1d) + 1d;
let _weekCount=tolong((_endTime - _startTime) / 7d);
let _windowValid=isnotnull(_observedAtUtc)
    and isnotnull(_startTime) and isnotnull(_endTime)
    and _startTime < _endTime and _endTime <= _observedAtUtc
    and _startTime == startofday(_startTime)
    and _endTime == startofday(_endTime)
    and dayofweek(_startTime) == 1d and dayofweek(_endTime) == 1d
    and _endTime <= _completedBoundary
    and _weekCount >= 2 and _weekCount <= 12;
let _controlsValid=_minimumPositiveSamples >= 1
    and _medianAbsoluteMs > 0 and _medianRelativePct > 0
    and _p95AbsoluteMs > 0 and _p95RelativePct > 0
    and _groupBudget >= 1 and _groupBudget <= 5000
    and _chartCellBudget >= 1 and _chartCellBudget <= 20000
    and _partitionCount >= 1 and _partitionId >= 0 and _partitionId < _partitionCount
    and _topN >= 1 and _topN <= _groupBudget;
let Weeks=range WeekIndex from 0 to 11 step 1
    | where _windowValid and _controlsValid and WeekIndex < _weekCount
    | extend WeekStart=_startTime + WeekIndex * 7d
    | project WeekIndex,WeekStart,WeekEnd=WeekStart + 7d;
let PairWeeks=Weeks
    | where WeekIndex > 0
    | project PreviousWeekStart=WeekStart - 7d,CurrentWeekStart=WeekStart;
```

Integer controls must be integer literals; reject fractional values before
assembly. Do not override budgets upward. Nonfinite threshold inputs are
invalid before assembly. The source scope confirmation is a separate T1 gate.

### T1: Window/control gate

```kusto
print ObservedAtUtc=_observedAtUtc,StartUtc=_startTime,EndUtc=_endTime,
    CompletedBoundaryUtc=_completedBoundary,CompletedWeeks=_weekCount,
    AdjacentComparisons=_weekCount - 1,
    WindowValid=_windowValid,ControlsValid=_controlsValid,
    ScopeConfirmed=_scopeConfirmed,
    ResourceSelected=isnotempty(_selectedResourceKey),
    EnvironmentSelected=isnotempty(_selectedEnvironmentId) or _resourceEnvironmentIsolated
```

All five Boolean checks must be true. Scope flags require independent
confirmation as specified by the source contract, not just setting them true
to pass the gate. Empty telemetry cannot confirm isolation or collection.

## Module B: Full key union and disjoint partitions

Hashing is only for work partitioning, not a new page identity. Its input
includes the page-name length so concatenated exact keys remain unambiguous.
Hash collisions place keys on the same partition; they never merge keys.
All full original page and role values remain grouping keys.

```kusto
let AllPageKeys=NamedPages
    | summarize by PageName,RoleName
    | extend PartitionId=hash(strcat(tostring(strlen(PageName)),":",PageName,RoleName),_partitionCount);
let PageKeys=AllPageKeys
    | where PartitionId == _partitionId
    | project PageName,RoleName;
```

### T2: Cardinality preflight

```kusto
print TotalKeys=toscalar(AllPageKeys | count),
    PartitionKeys=toscalar(PageKeys | count),
    PartitionId=_partitionId,PartitionCount=_partitionCount,
    CompletedWeeks=_weekCount
| extend TotalWeeklyGroups=TotalKeys * CompletedWeeks,
    TotalPairGroups=TotalKeys * (CompletedWeeks - 1),
    PartitionWeeklyGroups=PartitionKeys * CompletedWeeks,
    PartitionPairGroups=PartitionKeys * (CompletedWeeks - 1)
| extend FullWeeklyChartCells=TotalWeeklyGroups * 6,
    PartitionWeeklyChartCells=PartitionWeeklyGroups * 6
| extend PartitionFits=PartitionWeeklyGroups <= _groupBudget
    and PartitionPairGroups <= _groupBudget
    and PartitionWeeklyChartCells <= _chartCellBudget
```

If a partition does not fit, increase `_partitionCount`, preflight **every**
partition ID, and restart with that fixed partition count. Keep exact scope,
bounds, thresholds, and evidence fixed. Do not change partition count halfway
through a run. Reconcile partition key totals to TotalKeys; weekly rows to
TotalKeys times weeks; and pair rows to TotalKeys times (weeks minus one).
Respect any smaller tool limits and operational compute/time limits too.

The event-time window is frozen, but storage is not an ingestion snapshot.
Record query execution times and late-arrival risk. Inconsistent key/count
reconciliation across repeated reads requires rerunning affected results;
do not disguise it as confirmed coverage.

Never top-N each partition then claim a global top-N. For partitioned runs,
retrieve all T6 rows within budget, concatenate disjoint keys, reconcile
T7/T8 totals, and apply T9/T10's exact sort/rank/limit logic once to the complete
merged population. No averaging or merging of already-computed percentiles
across time/sample slices is allowed. Each page/role's entire interval belongs
to exactly one partition. T11-T13 use the global cohort, not page partitions,
and run once unless separately narrowed and explicitly relabeled.

### T3: Explicit exclusion totals

Use the source contract's coverage terminals as well, including missing and
conflicting environment, missing optional fields, and weight diagnostics.
The following differences are exact within one successful query.

```kusto
print ResourceObservedRows=toscalar(ResourceRows | count),
    ScopedBeforeSyntheticRows=toscalar(Scoped | count),
    SelectedPageRows=toscalar(PageRows | count),
    NamedPageRows=toscalar(NamedPages | count)
| extend CombinedScopeExcludedRows=ResourceObservedRows - ScopedBeforeSyntheticRows,
    SyntheticExcludedRows=ScopedBeforeSyntheticRows - SelectedPageRows,
    MissingPageNameRows=SelectedPageRows - NamedPageRows
```

CombinedScopeExcludedRows includes environment conflicts/mismatches/missing
IDs and exact page/role selection; it is not a count attributed to the chosen
environment. Do not add overlapping diagnostic exclusion counts together.

## Module C: Complete page-week grid

Count all named observations first. Compute duration statistics separately
from positive rows only; this avoids relying on empty/all-null aggregate
defaults. After the outer joins, coalesce **counts only**, not durations or
timestamps. Thus an absent week has zero observed counts and null percentiles.
A week with observations but no positive durations stays visible.

```kusto
let SliceNamed=NamedPages
    | join kind=leftsemi (PageKeys) on PageName,RoleName;
let SliceWeekRows=SliceNamed
    | extend WeekStart=startofweek(EventTime - 1d) + 1d;
let WeeklyCounts=SliceWeekRows
    | summarize ObservedViews=count(),
        PositiveN=countif(DurationState == "Positive"),
        ZeroN=countif(DurationState == "Zero"),
        MissingN=countif(DurationState == "MissingOrUnparseable"),
        InvalidN=countif(DurationState == "Invalid"),
        SamplingIndicatedN=countif(ValidWeight and Weight > 1),
        UnknownWeightN=countif(not(ValidWeight)),
        FirstObserved=min(EventTime),LastObserved=max(EventTime)
        by PageName,RoleName,WeekStart;
let WeeklyDurations=SliceWeekRows
    | where DurationState == "Positive"
    | summarize MedianMs=percentile(DurationMs,50),P95Ms=percentile(DurationMs,95)
        by PageName,RoleName,WeekStart;
let PageGrid=PageKeys
    | extend GridJoinKey=1
    | join kind=inner (Weeks | extend GridJoinKey=1) on GridJoinKey
    | project PageName,RoleName,WeekIndex,WeekStart,WeekEnd;
let PageWeekly=PageGrid
    | join kind=leftouter (WeeklyCounts) on PageName,RoleName,WeekStart
    | join kind=leftouter (WeeklyDurations) on PageName,RoleName,WeekStart
    | project PageName,RoleName,WeekIndex,WeekStart,WeekEnd,
        ObservedViews=coalesce(ObservedViews,long(0)),
        PositiveN=coalesce(PositiveN,long(0)),
        ZeroN=coalesce(ZeroN,long(0)),
        MissingN=coalesce(MissingN,long(0)),
        InvalidN=coalesce(InvalidN,long(0)),
        SamplingIndicatedN=coalesce(SamplingIndicatedN,long(0)),
        UnknownWeightN=coalesce(UnknownWeightN,long(0)),
        MedianMs,P95Ms,FirstObserved,LastObserved;
```

Reference: https://learn.microsoft.com/en-us/kusto/query/percentiles-aggregation-function
Percentiles are Kusto estimates over retained, unweighted positive samples.

### T5: Complete page-week timeline

```kusto
PageWeekly
| order by PageName asc,RoleName asc,WeekStart asc
```

The explicit projection in Module C avoids depending on suffixed join-key
column names. Case-sensitive aliases are used consistently in later modules.

## Module D: Explicit comparability evidence

This is a local input table, not a physical source. Leave it empty until
evidence exists. For each reviewed page/pair, insert exactly one row using the
**current** week's Monday as CurrentWeekStart. Decisions are case-sensitive:
`Comparable`, `Unknown`, or `NotComparable`. Include a sanitized evidence
summary explaining collection and cohort assessment. See
[interpretation.md](interpretation.md). Do not fill the table from sample
counts or query success alone. The blank default makes all pairs Unknown.

```kusto
let EvidenceRaw=datatable(
    PageName:string,RoleName:string,CurrentWeekStart:datetime,
    Comparability:string,EvidenceSummary:string
)[];
let Evidence=EvidenceRaw
    | summarize EvidenceRows=count(),Decision=take_any(Comparability),
        EvidenceSummary=take_any(EvidenceSummary)
        by PageName,RoleName,CurrentWeekStart
    | extend Comparability=iff(EvidenceRows == 1
        and Decision in ("Comparable","Unknown","NotComparable")
        and isnotempty(trim(@"\s+",EvidenceSummary)),Decision,"Unknown")
    | project PageName,RoleName,CurrentWeekStart,Comparability,EvidenceSummary;
```

### T4: Evidence gate

```kusto
print EvidenceRows=toscalar(EvidenceRaw | count),
    DuplicateKeys=toscalar(EvidenceRaw
        | summarize N=count() by PageName,RoleName,CurrentWeekStart
        | where N > 1 | count),
    InvalidDecisions=toscalar(EvidenceRaw
        | where Comparability !in ("Comparable","Unknown","NotComparable") | count),
    BlankEvidence=toscalar(EvidenceRaw
        | where isempty(trim(@"\s+",EvidenceSummary)) | count),
    UnknownPageKeys=toscalar(EvidenceRaw
        | join kind=leftanti (AllPageKeys | project PageName,RoleName)
            on PageName,RoleName | count),
    InvalidPairWeeks=toscalar(EvidenceRaw
        | join kind=leftanti (PairWeeks | project CurrentWeekStart)
            on CurrentWeekStart | count)
| extend EvidenceValid=DuplicateKeys == 0 and InvalidDecisions == 0
    and BlankEvidence == 0 and UnknownPageKeys == 0 and InvalidPairWeeks == 0
```

Stop and correct evidence if EvidenceValid is false. The defensive
single-row aggregation in D additionally prevents accidental join fan-out.
An empty evidence table passes this gate but confirms **nothing**.

## Module E: Every adjacent comparison and both metric labels

The two sides come from the same complete page-week grid, not inner joins
between positive-duration samples. Inner joining these grids preserves every
key/pair because each key has every week. Each join is followed by explicit
projection; no later expression assumes a generated suffixed column name.

```kusto
let PreviousWeeks=PageWeekly
    | project PageName,RoleName,CurrentWeekStart=WeekStart + 7d,
        PreviousViews=ObservedViews,PreviousPositiveN=PositiveN,
        PreviousZeroN=ZeroN,PreviousMissingN=MissingN,PreviousInvalidN=InvalidN,
        PreviousSamplingN=SamplingIndicatedN,PreviousUnknownWeightN=UnknownWeightN,
        PreviousMedianMs=MedianMs,PreviousP95Ms=P95Ms;
let CurrentWeeks=PageWeekly
    | where WeekIndex > 0
    | project PageName,RoleName,CurrentWeekStart=WeekStart,
        CurrentViews=ObservedViews,CurrentPositiveN=PositiveN,
        CurrentZeroN=ZeroN,CurrentMissingN=MissingN,CurrentInvalidN=InvalidN,
        CurrentSamplingN=SamplingIndicatedN,CurrentUnknownWeightN=UnknownWeightN,
        CurrentMedianMs=MedianMs,CurrentP95Ms=P95Ms;
let Comparisons=CurrentWeeks
    | join kind=inner (PreviousWeeks) on PageName,RoleName,CurrentWeekStart
    | project PageName,RoleName,PreviousWeekStart=CurrentWeekStart - 7d,CurrentWeekStart,
        PreviousViews,CurrentViews,PreviousPositiveN,CurrentPositiveN,
        PreviousZeroN,CurrentZeroN,PreviousMissingN,CurrentMissingN,
        PreviousInvalidN,CurrentInvalidN,PreviousSamplingN,CurrentSamplingN,
        PreviousUnknownWeightN,CurrentUnknownWeightN,
        PreviousMedianMs,CurrentMedianMs,PreviousP95Ms,CurrentP95Ms
    | join kind=leftouter (Evidence) on PageName,RoleName,CurrentWeekStart
    | project PageName,RoleName,PreviousWeekStart,CurrentWeekStart,
        PreviousViews,CurrentViews,PreviousPositiveN,CurrentPositiveN,
        PreviousZeroN,CurrentZeroN,PreviousMissingN,CurrentMissingN,
        PreviousInvalidN,CurrentInvalidN,PreviousSamplingN,CurrentSamplingN,
        PreviousUnknownWeightN,CurrentUnknownWeightN,
        PreviousMedianMs,CurrentMedianMs,PreviousP95Ms,CurrentP95Ms,
        Comparability=coalesce(Comparability,"Unknown"),
        EvidenceSummary=coalesce(EvidenceSummary,"No pair-specific comparability evidence")
    | extend SampleEligibility=case(
        PreviousViews == 0 and CurrentViews == 0,"AbsentBoth",
        PreviousViews == 0,"NewlyObserved",
        CurrentViews == 0,"NoLongerObserved",
        PreviousPositiveN == 0 and CurrentPositiveN == 0,"NoPositiveBoth",
        PreviousPositiveN == 0,"NoPositivePrevious",
        CurrentPositiveN == 0,"NoPositiveCurrent",
        PreviousPositiveN < _minimumPositiveSamples
            or CurrentPositiveN < _minimumPositiveSamples,"InsufficientPositiveSamples",
        "Eligible")
    | extend MedianDeltaMs=iff(PreviousPositiveN > 0 and CurrentPositiveN > 0,
            CurrentMedianMs - PreviousMedianMs,real(null)),
        P95DeltaMs=iff(PreviousPositiveN > 0 and CurrentPositiveN > 0,
            CurrentP95Ms - PreviousP95Ms,real(null))
    | extend MedianDeltaPct=iff(PreviousMedianMs > 0,
            100.0 * MedianDeltaMs / PreviousMedianMs,real(null)),
        P95DeltaPct=iff(PreviousP95Ms > 0,
            100.0 * P95DeltaMs / PreviousP95Ms,real(null))
    | extend MedianCandidate=SampleEligibility == "Eligible"
            and isnotnull(MedianDeltaMs) and isnotnull(MedianDeltaPct)
            and abs(MedianDeltaMs) >= _medianAbsoluteMs
            and abs(MedianDeltaPct) >= _medianRelativePct,
        P95Candidate=SampleEligibility == "Eligible"
            and isnotnull(P95DeltaMs) and isnotnull(P95DeltaPct)
            and abs(P95DeltaMs) >= _p95AbsoluteMs
            and abs(P95DeltaPct) >= _p95RelativePct
    | extend MedianStatus=case(
        SampleEligibility != "Eligible",SampleEligibility,
        Comparability == "NotComparable","CollectionNotComparable",
        Comparability == "Unknown" and MedianCandidate and MedianDeltaMs > 0,"ExploratoryPotentialMedianRegression",
        Comparability == "Unknown" and MedianCandidate and MedianDeltaMs < 0,"ExploratoryPotentialMedianImprovement",
        Comparability == "Unknown","ExploratoryNoMedianThresholdSignal",
        MedianCandidate and MedianDeltaMs > 0,"MedianRegression",
        MedianCandidate and MedianDeltaMs < 0,"MedianImprovement",
        "NoMedianThresholdSignal"),
        P95Status=case(
        SampleEligibility != "Eligible",SampleEligibility,
        Comparability == "NotComparable","CollectionNotComparable",
        Comparability == "Unknown" and P95Candidate and P95DeltaMs > 0,"ExploratoryPotentialTailRegression",
        Comparability == "Unknown" and P95Candidate and P95DeltaMs < 0,"ExploratoryPotentialTailImprovement",
        Comparability == "Unknown","ExploratoryNoTailThresholdSignal",
        P95Candidate and P95DeltaMs > 0,"TailRegression",
        P95Candidate and P95DeltaMs < 0,"TailImprovement",
        "NoTailThresholdSignal");
```

### T6: Complete adjacent comparisons

```kusto
Comparisons
| order by CurrentWeekStart asc,PageName asc,RoleName asc
```

Numeric deltas for small/noncomparable samples are descriptive only. Candidate
Booleans indicate sample/threshold arithmetic, not collection confirmation.
Only the final status and its evidence determine the permitted wording.

### T7: Whole-population joint status rollup

```kusto
PairWeeks
| join kind=leftouter (
    Comparisons
    | summarize PagePairs=count()
        by CurrentWeekStart,SampleEligibility,Comparability,MedianStatus,P95Status
) on CurrentWeekStart
| project PreviousWeekStart,CurrentWeekStart,
    SampleEligibility=coalesce(SampleEligibility,"NoNamedPageKeys"),
    Comparability=coalesce(Comparability,"Unknown"),
    MedianStatus=coalesce(MedianStatus,"NoNamedPageKeys"),
    P95Status=coalesce(P95Status,"NoNamedPageKeys"),
    PagePairs=coalesce(PagePairs,long(0))
| order by CurrentWeekStart asc,SampleEligibility asc,Comparability asc,
    MedianStatus asc,P95Status asc
```

This is joint median/tail accounting, not two separately summed populations.
For partitions, sum PagePairs by identical status keys across successful
partitions. Drop zero-count NoNamedPageKeys placeholders unless the **global**
pair has zero keys. Keep a global zero row if the entire input is empty.

### T8: Pair reconciliation

```kusto
PairWeeks
| join kind=leftouter (
    Comparisons
    | summarize AccountedPairs=count(),
        SampleEligiblePairs=countif(SampleEligibility == "Eligible"),
        ComparablePairs=countif(Comparability == "Comparable"),
        UnknownComparabilityPairs=countif(Comparability == "Unknown"),
        NotComparablePairs=countif(Comparability == "NotComparable")
        by CurrentWeekStart
) on CurrentWeekStart
| project PreviousWeekStart,CurrentWeekStart,
    ExpectedPairs=toscalar(PageKeys | count),
    AccountedPairs=coalesce(AccountedPairs,long(0)),
    SampleEligiblePairs=coalesce(SampleEligiblePairs,long(0)),
    ComparablePairs=coalesce(ComparablePairs,long(0)),
    UnknownComparabilityPairs=coalesce(UnknownComparabilityPairs,long(0)),
    NotComparablePairs=coalesce(NotComparablePairs,long(0))
| extend Reconciled=ExpectedPairs == AccountedPairs
    and AccountedPairs == ComparablePairs + UnknownComparabilityPairs + NotComparablePairs
| order by CurrentWeekStart asc
```

Reconcile these counts with T7 and T2, not only with each other. Even a
reconciled empty population does not prove that source collection was healthy.

### T9: Final-pair median regression ranking

Use this terminal only for an unsliced full-population run (`_partitionCount=1`).
For a partitioned run use the complete merged T6 population as described in B.
Comparable candidates sort before Unknown candidates; keep them visibly
separated in the report. NotComparable pairs remain in T6-T8, not this ranking.

```kusto
let RankedCandidates=Comparisons
    | where CurrentWeekStart == _endTime - 7d
    | where MedianCandidate and MedianDeltaMs > 0
        and Comparability in ("Comparable","Unknown")
    | extend EvidenceOrder=iff(Comparability == "Comparable",0,1)
    | order by EvidenceOrder asc,MedianDeltaMs desc,MedianDeltaPct desc,
        PageName asc,RoleName asc
    | serialize PopulationRank=row_number();
RankedCandidates
| extend FullCandidateCount=toscalar(RankedCandidates | count)
| order by PopulationRank asc
| take _topN
```

The rank exists before `take`. If no candidates exist, report the zero total
derived from T6/T7; no returned row is not a status message. To show
improvements, run the same terminal with `MedianDeltaMs < 0`, sort
`MedianDeltaMs asc,MedianDeltaPct asc`, and label the output improvements.
Do not call absence of candidates stable.

### T10: Final-pair p95 regression ranking

Same full-population/partition rule as T9. A median-only candidate list cannot
be used as the input to this query: tail-only signals would disappear.

```kusto
let RankedTailCandidates=Comparisons
    | where CurrentWeekStart == _endTime - 7d
    | where P95Candidate and P95DeltaMs > 0
        and Comparability in ("Comparable","Unknown")
    | extend EvidenceOrder=iff(Comparability == "Comparable",0,1)
    | order by EvidenceOrder asc,P95DeltaMs desc,P95DeltaPct desc,
        PageName asc,RoleName asc
    | serialize PopulationRank=row_number();
RankedTailCandidates
| extend FullCandidateCount=toscalar(RankedTailCandidates | count)
| order by PopulationRank asc
| take _topN
```

For tail improvements replace the direction and sort with
`P95DeltaMs < 0`, `P95DeltaMs asc,P95DeltaPct asc`.
If a caller wants older-pair detail, replace the final-week equality with one
explicit CurrentWeekStart from PairWeeks; do not change the weekly cohort.

## Module F: Whole-population weekly statistics and coverage

This module uses **all PageRows**, including missing names, after the same
resource/environment/role/page/synthetic scope. It is independent of B/C and
does not aggregate from per-page results. Empty weeks come from the spine.
Counts of observed hours are exact but do not establish collection continuity.

```kusto
let PopulationWeekRows=PageRows
    | extend WeekStart=startofweek(EventTime - 1d) + 1d;
let PopulationCounts=PopulationWeekRows
    | summarize ObservedViews=count(),
        PositiveN=countif(DurationState == "Positive"),
        ZeroN=countif(DurationState == "Zero"),
        MissingN=countif(DurationState == "MissingOrUnparseable"),
        InvalidN=countif(DurationState == "Invalid"),
        MissingPageNames=countif(isempty(trim(@"\s+",PageName))),
        UnknownRoleN=countif(isempty(RoleName)),
        SamplingIndicatedN=countif(ValidWeight and Weight > 1),
        UnknownWeightN=countif(not(ValidWeight)),
        FirstObserved=min(EventTime),LastObserved=max(EventTime)
        by WeekStart;
let PopulationDurations=PopulationWeekRows
    | where DurationState == "Positive"
    | summarize MedianMs=percentile(DurationMs,50),P95Ms=percentile(DurationMs,95)
        by WeekStart;
let PopulationHours=PopulationWeekRows
    | summarize by WeekStart,ObservedHour=bin(EventTime,1h)
    | summarize ObservedActiveHours=count() by WeekStart;
let PopulationWeekly=Weeks
    | join kind=leftouter (PopulationCounts) on WeekStart
    | join kind=leftouter (PopulationDurations) on WeekStart
    | join kind=leftouter (PopulationHours) on WeekStart
    | project WeekStart,WeekEnd,
        ObservedViews=coalesce(ObservedViews,long(0)),
        PositiveN=coalesce(PositiveN,long(0)),
        ZeroN=coalesce(ZeroN,long(0)),
        MissingN=coalesce(MissingN,long(0)),
        InvalidN=coalesce(InvalidN,long(0)),
        MissingPageNames=coalesce(MissingPageNames,long(0)),
        UnknownRoleN=coalesce(UnknownRoleN,long(0)),
        SamplingIndicatedN=coalesce(SamplingIndicatedN,long(0)),
        UnknownWeightN=coalesce(UnknownWeightN,long(0)),
        ObservedActiveHours=coalesce(ObservedActiveHours,long(0)),
        MedianMs,P95Ms,FirstObserved,LastObserved;
```

### T11: Whole-population weekly timeline

```kusto
PopulationWeekly
| order by WeekStart asc
```

If an application-level adjacent duration delta is needed, subtract adjacent
T11 statistics using the same signed formula, after checking counts and
collection comparability. Label it an overall retained-positive-sample
distribution comparison susceptible to page-mix changes; it is not an average
page regression rate. All-zero/null duration input still returns each week
with observed counts and null percentiles.

## Module G: Application/version/SDK/client mix evidence

This is a named-page population diagnostic, not automatic comparability
approval. Each event contributes once to each dimension. Do not sum counts
across dimensions and call that view volume. Empty values remain unknown.
Check labels for privacy before returning them. RoleInstance, UserKey, and
SessionKey are intentionally not exported.

```kusto
let MixRows=union
    (NamedPages | project WeekStart=startofweek(EventTime - 1d) + 1d,
        Dimension="AppVersion",Value=AppVersion,DurationState),
    (NamedPages | project WeekStart=startofweek(EventTime - 1d) + 1d,
        Dimension="SDKVersion",Value=SDKVersion,DurationState),
    (NamedPages | project WeekStart=startofweek(EventTime - 1d) + 1d,
        Dimension="Browser",Value=Browser,DurationState),
    (NamedPages | project WeekStart=startofweek(EventTime - 1d) + 1d,
        Dimension="ClientType",Value=ClientType,DurationState);
let MixStats=MixRows
    | summarize ObservedViews=count(),PositiveN=countif(DurationState == "Positive")
        by WeekStart,Dimension,Value;
let MixTotals=MixStats
    | summarize DimensionViews=sum(ObservedViews) by WeekStart,Dimension;
```

### T12: Version/client mix cardinality preflight

```kusto
print MixGroups=toscalar(MixStats | count)
| extend Fits=MixGroups <= _groupBudget
```

If Fits is false, narrow diagnostics explicitly to one exact page/role or
dimension/week and rerun the preflight. State that it is a diagnostic subset,
not population evidence. Do not drop unreturned metadata groups silently.

### T13: Version/client mix distributions

```kusto
MixStats
| join kind=inner (MixTotals) on WeekStart,Dimension
| project WeekStart,Dimension,Value,ObservedViews,PositiveN,
    RetainedViewSharePct=100.0 * ObservedViews / DimensionViews
| order by WeekStart asc,Dimension asc,ObservedViews desc,Value asc
```

Every existing MixStats group has at least one event, so DimensionViews is
positive; empty weeks yield no mix groups and remain explicit in T11.
For page-specific investigation, rerun P with `_pageName` set to the exact
page and `_roleFilterEnabled=true` plus the exact role (including empty).
Preserve the same UTC bounds. Label this narrower diagnostic cohort and do
not substitute it for the earlier global status rollup.

## Optional chart contract

No new KQL dependency is required. Use already-validated complete results:

- From T5, select the full weekly history of each globally selected page/role.
  Chart series are exact page/role plus metric; values are MedianMs or P95Ms;
  UTC WeekStart is x. Include PositiveN alongside the chart in Markdown.
  Retain null gaps; never zero-fill or interpolate missing duration.
- From T9/T10, plot signed delta ms, with page/role, metric, and evidence
  label in each bar. Do not chart `CollectionNotComparable` as a regression.
- The chart subset may be top-N for readability **after** global ranking, but
  retain T7/T8 global accounting. Recheck actual serialized chart cells against
  the 20,000-cell or smaller host budget before calling a plotting tool.
- Discover actual plotting tool argument schemas rather than inventing
  arguments. If unsupported or unsuccessful, use the report-template tables
  with dates, counts, units, nulls, and labels. Plot failure does not erase the
  analysis; query failure does prevent interpreting affected values.

## Local validation versus service validation

The package can be checked locally for links, fences, frontmatter, dependency
closure, ASCII, and synthetic cohort/time/threshold behavior. Those checks
cannot establish live source fields, service limits, KQL compilation, sampling
policy, retention, or collection continuity. Explicitly record whether
`ValidateQuery` and actual customer-source execution succeeded at runtime.
