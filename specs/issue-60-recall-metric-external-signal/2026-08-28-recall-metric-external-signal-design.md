# Recall Metric — External Missed-Detection Signal

**Issue:** casehubio/casehub-ras#60
**Date:** 2026-08-28
**Status:** Design

## Problem

The feedback loop computes precision and noise rate from case outcomes — data RAS itself produces. Recall (the fraction of real situations that RAS successfully detects) requires knowing about false negatives: situations that should have been detected but weren't. RAS cannot compute this from its own outcome data because missed detections are invisible by definition. An external signal is required.

`DriftDirection.UNDER_SENSITIVE` is referenced aspirationally in issues #58 and #60 but was never created. The feedback loop handles over-sensitivity (noise rate > 0.5 → raise threshold) but has no under-sensitivity path.

## Architecture

Extend `OutcomeLedger` — the existing feedback data store — with missed-detection recording. Precision and recall are two projections of the same confusion matrix; they should be computed from a single data source within a single time window.

No new SPI. No new CDI event type. The extension point for future signal sources (ground truth systems, batch imports, streaming integrations) is `OutcomeLedger.recordMissed()` — any adapter calls it directly, mirroring how `CaseOutcomeObserver` → `OutcomeRecorder` → `OutcomeLedger.record()` works for case outcomes.

### Data Flow

```
Operator → REST endpoint → validation → OutcomeLedger.recordMissed()
                                              ↓
FeedbackUpdateJob (every 5m) → FeedbackAnalyzer.analyze()
                                              ↓
                              OutcomeLedger.statistics() → OutcomeStatistics
                                              ↓
                              recall = confirmed / (confirmed + missed)
                                              ↓
                              FeedbackMetrics → ras.feedback.recall gauge
```

## MissedDetectionRecord

New record type in `api/`. Separate from `OutcomeRecord` — missed detections have no `caseId`, `outcomeLabel`, or `classification`.

| Field | Type | Description |
|-------|------|-------------|
| `situationId` | String | Which situation definition should have fired |
| `correlationKey` | String | Domain-level correlation key (e.g., account ID) |
| `tenancyId` | String | Tenant identity |
| `eventTime` | Instant | When the missed event occurred (operator-specified) |
| `reportedBy` | String | Operator identity for audit trail |
| `reportId` | UUID | Deduplication key |
| `recordedAt` | Instant | System-generated timestamp of report filing |

### Deduplication

Composite key: `(situationId, correlationKey, tenancyId, eventTime)`. Duplicate reports for the same miss are idempotent — the same missed event reported N times counts as 1 false negative. `reportId` (UUID) provides an additional external dedup handle.

## OutcomeLedger Extension

New method on existing SPI:

```java
default void recordMissed(MissedDetectionRecord record) {
    throw new UnsupportedOperationException("recordMissed not implemented");
}
```

Default throws `UnsupportedOperationException` rather than silently dropping — callers must know if the implementation doesn't support missed detection recording. This is a pre-release API; the default can be changed to no-op when the feature stabilises.

### statistics() Extension

`OutcomeStatistics` gains a `missedCount` field. The existing `statistics()` method returns the extended record. All counts use the same `FeedbackConfig.retentionPeriod()` time window.

### Existing Methods Unchanged

`record()`, `lastNoiseDismissalTime()`, `countByLabel()`, `ganglionStatistics()`, `distinctTenancies()`, `removeRecordsBefore()` — all unchanged. `removeRecordsBefore()` applies to missed detection records as well (retention cleanup).

## Recall Computation

```java
// On OutcomeStatistics — NOT on QualityMetrics
public double recall() {
    long decisive = confirmedCount + missedCount;
    return decisive == 0 ? Double.NaN : confirmedCount / (double) decisive;
}
```

### Why Not on QualityMetrics

`QualityMetrics` is implemented by both `OutcomeStatistics` (situation-level) and `GanglionOutcomeStatistics` (per-ganglion). Per-ganglion missed detection data does not exist (signals are situation-level only). A `recall()` default on the interface would return `confirmed / (confirmed + 0)` = 1.0 for any ganglion with confirmed outcomes — semantically wrong (implies perfect recall when actually no data exists). For `OutcomeStatistics`, `missedCount == 0` means "no misses reported" (metric accurate given available data); for `GanglionOutcomeStatistics`, it would mean "measurement impossible" — categorically different.

When ganglion-level signals are added in a future enrichment, `recall()` can be promoted to `QualityMetrics`.

### NaN Convention

`recall()` returns `NaN` when `confirmedCount + missedCount == 0`, consistent with `precision()`'s NaN convention. `FeedbackMetrics.setGauge()` suppresses gauge registration for NaN values.

## Validation

On ingestion (`recordMissed` call path):

1. **Situation exists:** `SituationDefinitionRegistry.exists(situationId)` — reject unknown situations
2. **Temporal bounds:** `eventTime` within `FeedbackConfig.retentionPeriod()` of current time — records outside the window would be cleaned up by retention immediately
3. **Deduplication:** composite key `(situationId, correlationKey, tenancyId, eventTime)` — idempotent on duplicate

Validation that the situation wasn't actually detected (cross-referencing `SituationQueryService.history()`) is deferred. Operators are asserting a fact about the real world based on domain knowledge the system doesn't have; the system records the assertion, not second-guesses it.

## Metrics

| Metric | Type | Tags | Purpose |
|--------|------|------|---------|
| `ras.feedback.recall` | gauge | `situation_id`, `tenancy_id` | Recall per situation per tenant |
| `ras.feedback.missed` | counter | `situation_id`, `tenancy_id` | Missed detection reports received |
| `ras.feedback.missed.rejected` | counter | `situation_id`, `reason` | Reports rejected by validation |

Existing gauges (`ras.feedback.precision`, `ras.feedback.noise_rate`) are unchanged. `FeedbackUpdateJob` pushes recall alongside them.

## Persistence

### InMemoryOutcomeLedger (runtime/)

`@DefaultBean`. New `ConcurrentHashMap<MissedDetectionKey, MissedDetectionRecord>` alongside existing outcome storage. `MissedDetectionKey` is the composite `(situationId, correlationKey, tenancyId, eventTime)`. `statistics()` counts missed records in the retention window.

### JpaOutcomeLedger (persistence-jpa/)

New table `ras_missed_detection` (Flyway V9):

```sql
CREATE TABLE ras_missed_detection (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    situation_id  VARCHAR(255) NOT NULL,
    correlation_key VARCHAR(255) NOT NULL,
    tenancy_id    VARCHAR(255) NOT NULL,
    event_time    TIMESTAMP WITH TIME ZONE NOT NULL,
    reported_by   VARCHAR(255) NOT NULL,
    report_id     UUID NOT NULL,
    recorded_at   TIMESTAMP WITH TIME ZONE NOT NULL,
    UNIQUE(situation_id, correlation_key, tenancy_id, event_time)
);

CREATE INDEX idx_missed_detection_situation ON ras_missed_detection(situation_id, tenancy_id);
```

`INSERT ON CONFLICT` for deduplication (same pattern as `ras_outcome_record`). `statistics()` query joins outcome count with missed detection count in the same time window.

## Tuning Interaction

No auto-tuning for recall. `FeedbackTuningStrategy` gains no new methods. `DefaultTuningStrategy` is unchanged. Operators observe the `ras.feedback.recall` gauge and adjust situation definitions or thresholds manually.

### DriftDirection

Deferred to a follow-on design. `DriftDirection` does not exist in the codebase. Designing it requires decisions about the entire drift classification model — thresholds for when each direction applies, how OVER_SENSITIVE and UNDER_SENSITIVE interact, whether they're mutually exclusive. This is a separate concern from the recall metric itself.

## Module Placement

| Artifact | Module | Rationale |
|----------|--------|-----------|
| `MissedDetectionRecord` | `api/` | Domain type |
| `OutcomeLedger.recordMissed()` | `api/` | SPI extension |
| `OutcomeStatistics.missedCount` + `recall()` | `api/` | Domain type extension |
| `InMemoryOutcomeLedger` extension | `runtime/` | Default implementation |
| `FeedbackAnalyzer` extension | `runtime/` | Recall in analysis output |
| `FeedbackMetrics` recall gauge | `runtime/` | Micrometer integration |
| `FeedbackUpdateJob` recall push | `runtime/` | Scheduled update |
| Validation logic | `runtime/` | Registration check + temporal bounds |
| `JpaOutcomeLedger` extension | `persistence-jpa/` | JPA persistence |
| Flyway V9 migration | `persistence-jpa/` | Schema extension |

## Testing Strategy

| Test | Scope | Verifies |
|------|-------|----------|
| `MissedDetectionRecordTest` | Unit | Record construction, field validation |
| `OutcomeStatisticsRecallTest` | Unit | recall() computation, NaN on empty, same-window consistency with precision |
| `InMemoryOutcomeLedgerMissedTest` | Unit | recordMissed, dedup, statistics with missed count, retention cleanup |
| `FeedbackAnalyzerRecallTest` | Unit | analyze() returns missedCount, recall gauge value |
| `FeedbackUpdateJobRecallTest` | Unit | recall gauge pushed alongside precision/noise_rate |
| `AbstractOutcomeLedgerContractTest` extension | Contract | recordMissed contract across implementations |
| `JpaOutcomeLedgerMissedTest` | Integration | Flyway V9, INSERT ON CONFLICT dedup, statistics query |
| Validation tests | Unit | Unknown situation rejected, out-of-window rejected, dedup idempotent |

## Epistemological Limitations

Operator-reported recall is inherently biased. Operators can only report misses they independently notice — systematic blind spots (entire event patterns the system never processes) remain invisible. This is not a flaw in the design; it's an inherent property of any external false-negative signal. Adding future sources (ground truth systems, periodic audit sampling) mitigates but never eliminates this bias. The recall metric should be interpreted as a lower bound on the true miss rate.

## Known Gaps

1. **Per-ganglion recall:** Requires ganglion-level missed detection signals (future enrichment on D2)
2. **DriftDirection classification:** Deferred to separate design (D8)
3. **F1 score gauge:** Operators compute externally from precision + recall gauges; one-line addition when needed
4. **Trigger-history cross-reference:** Validation that the situation wasn't actually detected is deferred — operators assert domain facts the system can't verify

## References

- `OutcomeLedger.java` — SPI being extended
- `OutcomeRecord.java` — existing outcome record (NOT being modified)
- `OutcomeStatistics.java` — gains `missedCount` + `recall()`
- `QualityMetrics.java` — NOT gaining `recall()` (see design rationale)
- `FeedbackAnalyzer.java` — gains recall in analysis output
- `FeedbackMetrics.java` — gains recall gauge
- `FeedbackUpdateJob.java` — pushes recall gauge
- `InMemoryOutcomeLedger.java` — runtime/ default impl
- `JpaOutcomeLedger.java` — persistence-jpa/ impl
- `AbstractOutcomeLedgerContractTest.java` — contract test extension
- `DefaultTuningStrategy.java` — unchanged (no auto-tuning for recall)
- Issue #60 — recall metric requirement
- Issue #58 — three signal sources identified (closed as duplicate)
- Issue #40 — feedback loop design (requirement 4: precision/recall per ganglion)
- `2026-08-06-ras-feedback-loop-design.md` — feedback loop spec
- `2026-08-21-per-ganglion-quality-metrics-design.md` — quality metrics spec
