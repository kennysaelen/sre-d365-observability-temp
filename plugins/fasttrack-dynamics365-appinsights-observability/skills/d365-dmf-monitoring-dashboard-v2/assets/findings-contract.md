# DMF dashboard findings contract

```json
{
  "metadata": {
    "title": "D365 DMF telemetry",
    "source": "",
    "environment": "",
    "time_range_utc": {"start": "", "end": ""},
    "direction_lookback_start_utc": "",
    "ingestion_lag_minutes": 5,
    "filters": {},
    "generated_utc": ""
  },
  "coverage": {
    "events_missing_execution_id": 0,
    "completed_rows_missing_record_count": 0,
    "partial_record_executions": 0,
    "duplicate_end_groups": 0,
    "conflicting_count_groups": 0,
    "completed_executions_without_start": 0,
    "completed_executions_with_repeated_starts": 0,
    "completed_executions_with_reversed_pair": 0,
    "unknown_event_types": []
  },
  "summary_kpis": [
    {"id": "completed_executions", "value": 0, "severity": "info"},
    {"id": "completed_records", "value": 0, "severity": "info"},
    {"id": "error_events", "value": 0, "affected_executions": 0, "severity": "info"},
    {"id": "unfinished_runs", "value": 0, "severity": "info"}
  ],
  "insights": [
    {"id": "direction_mix", "title": "", "intent": "", "rows": []},
    {"id": "executions_per_entity", "title": "", "intent": "", "rows": []},
    {"id": "records_per_entity", "title": "", "intent": "", "rows": []},
    {"id": "duration_per_entity", "title": "", "intent": "", "rows": []},
    {"id": "executions_per_project", "title": "", "intent": "", "rows": []},
    {"id": "records_per_project", "title": "", "intent": "", "rows": []},
    {"id": "errors", "title": "", "intent": "", "rows": []},
    {"id": "error_trend", "title": "", "intent": "", "rows": []},
    {"id": "unfinished_runs", "title": "", "intent": "", "rows": []},
    {"id": "slowest_executions", "title": "", "intent": "", "rows": []}
  ],
  "findings": [
    {
      "statement": "",
      "evidence_label": "FACT",
      "confidence": "HIGH",
      "severity": "info",
      "evidence": [],
      "handoff_skill": ""
    }
  ]
}
```

Allowed severities: `info`, `success`, `warning`, `critical`. Every insight requires `id`, `title`, `intent`, and `rows`. Renderers may add visualization hints without changing evidence.
