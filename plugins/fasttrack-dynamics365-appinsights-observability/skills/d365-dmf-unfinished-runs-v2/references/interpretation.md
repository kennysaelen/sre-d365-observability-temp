# Unfinished-run interpretation

- **FACT:** start without end in the observed lookback, latest event, silence, errors, batch state, or restart event.
- **CORRELATED:** matching-node process interruption or resource event overlaps the execution.
- **HYPOTHESIS:** the run is hung, dead, blocked, or safe to rerun.

`Active` means recent telemetry, not guaranteed health. `Silent` means no recent telemetry, not proven failure. `ErroredNoEnd` is higher concern but still requires outcome or batch confirmation.

Cancellation and rerun can leave partial commits or duplicate target records. Recommend them only after a human verifies batch state, progress, idempotency, and recovery procedure.
