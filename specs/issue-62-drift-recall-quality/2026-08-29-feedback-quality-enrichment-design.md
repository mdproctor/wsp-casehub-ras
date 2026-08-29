# Feedback Quality Enrichment — DriftDirection, Per-Ganglion Recall, Trigger Cross-Reference

**Issues:** casehubio/casehub-ras#62, #63, #64
**Date:** 2026-08-29
**Status:** Design

## Problem

The feedback loop (#40) computes precision and noise rate. The recall metric (#60) added missed detection signals and a situation-level recall gauge. Three enrichments were deferred:

1. **DriftDirection classification (#62):** Raw gauge values (precision, recall, noise rate) require operators to interpret drift manually. No single signal says "this situation is getting worse and here's how." `DriftDirection` does not exist in the codebase.

2. **Per-ganglion missed detection signals (#63):** Missed detection reports are situation-level only. Per-ganglion recall requires knowing which specific ganglion(s) failed to fire — data MissedDetectionRecord does not carry.

3. **Trigger-history cross-reference (#64):** Operators report missed detections without the system checking whether the situation was actually detected. A simple advisory cross-reference would surface obvious operator errors.

## DriftDirection Classification (#62)

### Enum

New enum in `api/`:

```java
public enum DriftDirection {
    OVER_SENSITIVE,
    UNDER_SENSITIVE,
    BOTH_DRIFTING,
    STABLE,
    INSUFFICIENT_DATA
}
```

OVER_SENSITIVE and UNDER_SENSITIVE measure independent axes of the confusion matrix (FP rate vs FN rate). A detector CAN be simultaneously noisy and miss real events — BOTH_DRIFTING captures this distinct "miscalibrated" state where threshold adjustment is the wrong action and ganglion reconfiguration is needed.

### Classification Logic

`FeedbackAnalyzer.classifyDrift(OutcomeStatistics, FeedbackConfig)`:

1. If `totalOutcomes < MIN_DRIFT_OUTCOMES (10)` → `INSUFFICIENT_DATA`
2. `overSensitive = !NaN(noiseRate) && noiseRate > config.overSensitiveThreshold()`
3. `underSensitive = !NaN(recall) && recall < config.underSensitiveThreshold()`
   - Recall NaN (no confirmed + no missed) → `underSensitive = false`
   - Additionally requires `(confirmedCount + missedCount) >= MIN_RECALL_SAMPLES (3)` — prevents trivial-data classification
4. If `overSensitive && underSensitive` → `BOTH_DRIFTING`
5. If `overSensitive` → `OVER_SENSITIVE`
6. If `underSensitive` → `UNDER_SENSITIVE`
7. Else → `STABLE`

`MIN_DRIFT_OUTCOMES = 10` matches `DefaultTuningStrategy.MIN_OUTCOMES_THRESHOLD`. `MIN_RECALL_SAMPLES = 3` prevents a single missed detection report from triggering UNDER_SENSITIVE.

Threshold semantics: `overSensitiveThreshold` is a noise rate **ceiling** (above = over-sensitive). `underSensitiveThreshold` is a recall **floor** (below = under-sensitive).

### FeedbackConfig Extension

Two new optional fields:

```java
// On FeedbackConfig record
double overSensitiveThreshold,    // default 0.5, noise rate ceiling
double underSensitiveThreshold,   // default 0.5, recall floor
Duration crossRefWindow           // default PT1H, for trigger cross-reference
```

Validation: both thresholds in (0.0, 1.0]. `crossRefWindow` positive.

YAML:

```yaml
feedback:
  noiseLabels: [dismissed]
  confirmedLabels: [escalated]
  suppressionCooldown: PT6H
  learningRate: 0.1
  retentionPeriod: P90D
  tuningEnabled: true
  overSensitiveThreshold: 0.4    # optional, default 0.5
  underSensitiveThreshold: 0.7   # optional, default 0.5
  crossRefWindow: PT2H           # optional, default PT1H
```

### Metric

`ras.feedback.drift` — gauge tagged with `direction`, `situation_id`, `tenancy_id`. Published by `FeedbackUpdateJob` via `FeedbackMetrics` every 5 minutes. Each `(situationId, tenancyId)` pair produces exactly one gauge per cycle — `FeedbackMetrics` uses a dedicated `setDriftGauge()` that re-registers the gauge with the current direction tag, replacing the previous direction. Internally, the drift gauge uses a separate holder keyed by `(situationId, tenancyId)` (without direction) to avoid stale gauges accumulating when the direction changes.

### BOTH_DRIFTING Auto-Tuning Guard

When `classifyDrift()` returns `BOTH_DRIFTING`, `FeedbackUpdateJob.processTenant()` skips both threshold adjustment and prior adjustment for that `(situationId, tenancyId)` pair. Auto-tuning is inappropriate when the detection model is fundamentally miscalibrated — raising thresholds worsens recall, and prior adjustment based on skewed data propagates the miscalibration.

OVER_SENSITIVE-only auto-tuning is unchanged. UNDER_SENSITIVE has no auto-tuning (D4 from #60).

## Per-Ganglion Missed Detection Signals (#63)

### MissedDetectionRecord Extension

Optional `List<String> ganglionIds` field. Null/empty = situation-level miss (backwards compatible). Non-empty = operator specifies which ganglion(s) should have fired.

```java
public record MissedDetectionRecord(
        String situationId,
        String correlationKey,
        String tenancyId,
        Instant eventTime,
        String reportedBy,
        UUID reportId,
        Instant recordedAt,
        List<String> ganglionIds    // new, nullable
) {
    // Existing 7-arg constructor preserved (defaults ganglionIds to null)
    public MissedDetectionRecord(String situationId, String correlationKey,
            String tenancyId, Instant eventTime, String reportedBy,
            UUID reportId, Instant recordedAt) {
        this(situationId, correlationKey, tenancyId, eventTime,
             reportedBy, reportId, recordedAt, null);
    }
}
```

### Validation

`MissedDetectionRecorder` validates ganglionIds when present:
- Each ganglionId must exist in the situation's chain mode referenced ganglia
- Unknown ganglionIds rejected with `UNKNOWN_GANGLION` error code

### Deduplication

Dedup key remains the situation-level composite `(situationId, correlationKey, tenancyId, eventTime)`. A ganglion-enriched report after a situation-level report for the same event is rejected as duplicate. Late enrichment is not supported — operators who want ganglion attribution must include it in their initial report.

### REST Endpoint Extension

`POST /api/ras/feedback/missed` request body gains optional `ganglionIds`:

```json
{
    "situationId": "fire-risk-detector",
    "correlationKey": "ACC-12345",
    "tenancyId": "tenant-prod",
    "eventTime": "2026-08-28T14:30:00Z",
    "reportedBy": "operator@example.com",
    "reportId": "550e8400-e29b-41d4-a716-446655440000",
    "ganglionIds": ["temp-classifier", "smoke-detector"]
}
```

### GanglionOutcomeStatistics Extension

`GanglionOutcomeStatistics` gains `missedCount` field and its own `recall()` method:

```java
public record GanglionOutcomeStatistics(
        String ganglionId,
        long totalOutcomes,
        long noiseCount,
        long confirmedCount,
        long neutralCount,
        long missedCount           // new
) implements QualityMetrics {

    // Existing 5-arg constructor preserved (defaults missedCount to 0)
    public GanglionOutcomeStatistics(String ganglionId, long totalOutcomes,
            long noiseCount, long confirmedCount, long neutralCount) {
        this(ganglionId, totalOutcomes, noiseCount, confirmedCount, neutralCount, 0);
    }

    public double recall() {
        long decisive = confirmedCount + missedCount;
        return decisive == 0 ? Double.NaN : confirmedCount / (double) decisive;
    }
}
```

### Why recall() Is NOT on QualityMetrics

`recall()` stays on `OutcomeStatistics` (situation-level) and `GanglionOutcomeStatistics` (per-ganglion) as separate methods — NOT promoted to the `QualityMetrics` interface.

Most operators will file situation-level missed reports (no ganglionIds). For ganglia without explicit missed reports, `missedCount` stays 0 and a `recall()` default on `QualityMetrics` would return `confirmedCount / (confirmedCount + 0) = 1.0` — misleadingly implying perfect recall when no data exists. This is the exact problem #60 D6 was designed to prevent.

The semantic difference: situation-level `missedCount == 0` means "no misses reported" (metric accurate given available data). Per-ganglion `missedCount == 0` means "measurement not available" (no ganglion-level reports filed). Different semantics, different methods.

Promotion to `QualityMetrics` is gated on adoption evidence — sufficient ganglion-level missed detection reports in production.

### OutcomeLedger Extension

`ganglionStatistics()` returns extended `GanglionOutcomeStatistics` with `missedCount`. This is a two-query union: the existing `ras_outcome_record` JSONB aggregation query (#59) produces per-ganglion outcome counts, and a second query against `ras_missed_detection` produces per-ganglion missed counts. Results are merged by ganglionId into the same `Map<String, GanglionOutcomeStatistics>` — the missed count from the second query is set on the corresponding entry from the first query (defaulting to 0 for ganglia with no missed reports).

### Persistence

**JPA — ras_missed_detection table extension (Flyway V10):**

```sql
ALTER TABLE ras_missed_detection
  ADD COLUMN ganglion_ids JSONB;
```

Nullable. Existing rows have NULL (situation-level misses). `JpaOutcomeLedger.ganglionStatistics()` joins against `ras_missed_detection` for per-ganglion missed counts:

```sql
SELECT elem AS ganglion_id, COUNT(*)
FROM ras_missed_detection,
     jsonb_array_elements_text(ganglion_ids) elem
WHERE situation_id = :sid AND tenancy_id = :tid
  AND event_time >= :since
GROUP BY elem
```

Rows with NULL `ganglion_ids` are automatically excluded by `jsonb_array_elements_text`.

**InMemoryOutcomeLedger:** `ganglionStatistics()` filters missed detection records by ganglionIds presence and counts per ganglionId.

### Metrics

| Metric | Type | Tags | Purpose |
|--------|------|------|---------|
| `ras.feedback.ganglion.recall` | gauge | `ganglion_id`, `situation_id`, `tenancy_id` | Per-ganglion recall |

Published by `FeedbackUpdateJob` via `FeedbackMetrics.recordGanglionStatistics()`. NaN suppresses gauge registration (existing convention).

## Trigger-History Cross-Reference (#64)

### Advisory Warning in MissedDetectionRecorder

`MissedDetectionRecorder` gains an optional `Instance<SituationQueryService>` constructor dependency. After all validation passes but before storage, when SituationQueryService is resolvable:

1. Query `history(tenancyId, situationId, correlationKey, eventTime - crossRefWindow, eventTime + crossRefWindow)`
2. Filter results for `changeType == TRIGGERED`
3. If TRIGGERED events found: set `possiblyDetected = true` and `lastTriggerTime` on RecordResult

The record is stored regardless. The cross-reference is informational — the system surfaces a potential operator error without overriding operator judgment.

### Cross-Reference Window

`FeedbackConfig.crossRefWindow()` — configurable `Duration`, default `PT1H`. One hour before and after the reported `eventTime`. Wide enough for clock skew and event processing delays, narrow enough to be meaningful.

### Retention Mismatch Caveat

If `ras.event-history.retention` (default P30D) is shorter than `FeedbackConfig.retentionPeriod`, the cross-reference returns "not detected" for events whose trigger history has been cleaned up. The `possiblyDetected` response must carry this caveat: absence of trigger history does not confirm a miss — events older than the event history retention period cannot be cross-referenced.

### RecordResult Extension

```java
public record RecordResult(
        boolean accepted,
        boolean isNew,
        String rejectionReason,
        boolean possiblyDetected,           // new
        Instant lastTriggerTime             // new, nullable
) {
    // Existing factory methods preserved with defaults
    static RecordResult accepted(boolean isNew) {
        return new RecordResult(true, isNew, null, false, null);
    }
    static RecordResult accepted(boolean isNew, boolean possiblyDetected,
                                  Instant lastTriggerTime) {
        return new RecordResult(true, isNew, null, possiblyDetected, lastTriggerTime);
    }
    static RecordResult rejected(String reason) {
        return new RecordResult(false, false, reason, false, null);
    }
}
```

### REST Response Extension

`POST /api/ras/feedback/missed` response gains optional `possiblyDetected` and `lastTriggerTime`:

**201 Created** (with advisory):
```json
{
    "reportId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RECORDED",
    "recordedAt": "2026-08-28T14:31:02Z",
    "possiblyDetected": true,
    "lastTriggerTime": "2026-08-28T14:25:00Z",
    "advisory": "A trigger was found for this situation within the cross-reference window. The report has been recorded. If the detection was correct, this report may inflate the missed count."
}
```

**201 Created** (no advisory):
```json
{
    "reportId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RECORDED",
    "recordedAt": "2026-08-28T14:31:02Z",
    "possiblyDetected": false
}
```

## Module Placement

| Component | Module | Notes |
|-----------|--------|-------|
| `DriftDirection` | `api/` | New enum |
| `FeedbackConfig` (change) | `api/` | +overSensitiveThreshold, +underSensitiveThreshold, +crossRefWindow |
| `MissedDetectionRecord` (change) | `api/` | +ganglionIds |
| `GanglionOutcomeStatistics` (change) | `api/` | +missedCount, +recall() |
| `FeedbackAnalyzer` (change) | `runtime/` | +classifyDrift() |
| `FeedbackUpdateJob` (change) | `runtime/` | +drift publication, +BOTH_DRIFTING guard |
| `FeedbackMetrics` (change) | `runtime/` | +drift gauge, +ganglion recall gauge |
| `MissedDetectionRecorder` (change) | `runtime/` | +ganglionId validation, +cross-reference |
| `MissedDetectionResource` (change) | `runtime/` | +ganglionIds in request, +possiblyDetected in response |
| `InMemoryOutcomeLedger` (change) | `runtime/` | +ganglion missed counts in ganglionStatistics() |
| `JpaOutcomeLedger` (change) | `persistence-jpa/` | +ganglion missed counts SQL |
| Flyway V10 | `persistence-jpa/` | ALTER TABLE ras_missed_detection ADD ganglion_ids JSONB |
| `YamlSituationDefinitionProvider` (change) | `runtime/` | +new FeedbackConfig fields parsing |

## Testing Strategy

### DriftDirection (#62)

| Test | Scope | Verifies |
|------|-------|----------|
| `DriftDirectionTest` | Unit | Enum values, coverage |
| `FeedbackAnalyzerDriftTest` | Unit | classifyDrift() logic: all 5 states, NaN handling, threshold configurability, MIN_DRIFT_OUTCOMES guard, MIN_RECALL_SAMPLES guard |
| `FeedbackUpdateJobDriftTest` | Unit | Drift gauge published, BOTH_DRIFTING suppresses threshold/prior adjustment |
| `FeedbackMetricsDriftTest` | Unit | ras.feedback.drift gauge registration and tag values |

### Per-Ganglion Recall (#63)

| Test | Scope | Verifies |
|------|-------|----------|
| `MissedDetectionRecordGanglionTest` | Unit | 8-arg constructor, 7-arg backwards compat, ganglionIds null safety |
| `GanglionOutcomeStatisticsRecallTest` | Unit | recall() computation, NaN on empty, 6-arg and 5-arg constructors |
| `MissedDetectionRecorderGanglionTest` | Unit | ganglionId validation against chain mode, UNKNOWN_GANGLION rejection |
| `AbstractOutcomeLedgerContractTest` extension | Contract | ganglionStatistics() with missed counts, null ganglionIds gracefully skipped |
| `JpaOutcomeLedgerGanglionMissedTest` | Integration | Flyway V10, JSONB ganglion_ids, ganglionStatistics() SQL |
| `MissedDetectionResourceGanglionTest` | Integration | Request with ganglionIds, 400 on unknown ganglion |

### Trigger Cross-Reference (#64)

| Test | Scope | Verifies |
|------|-------|----------|
| `MissedDetectionRecorderCrossRefTest` | Unit | possiblyDetected true/false, SituationQueryService absent (graceful), window calculation |
| `MissedDetectionResourceCrossRefTest` | Integration | Response includes possiblyDetected and advisory |

### FeedbackConfig Extension

| Test | Scope | Verifies |
|------|-------|----------|
| `FeedbackConfigDriftTest` | Unit | New field validation, defaults, YAML parsing |

## References

- `api/src/main/java/io/casehub/ras/api/QualityMetrics.java` — interface NOT gaining recall()
- `api/src/main/java/io/casehub/ras/api/OutcomeStatistics.java` — existing recall(), unchanged
- `api/src/main/java/io/casehub/ras/api/GanglionOutcomeStatistics.java` — gains missedCount + recall()
- `api/src/main/java/io/casehub/ras/api/MissedDetectionRecord.java` — gains ganglionIds
- `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java` — gains drift thresholds + crossRefWindow
- `api/src/main/java/io/casehub/ras/api/OutcomeLedger.java` — ganglionStatistics() returns extended stats
- `api/src/main/java/io/casehub/ras/api/SituationQueryService.java` — history() for cross-reference
- `runtime/src/main/java/io/casehub/ras/runtime/FeedbackAnalyzer.java` — gains classifyDrift()
- `runtime/src/main/java/io/casehub/ras/runtime/FeedbackUpdateJob.java` — drift publication + BOTH_DRIFTING guard
- `runtime/src/main/java/io/casehub/ras/runtime/FeedbackMetrics.java` — drift gauge + ganglion recall gauge
- `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionRecorder.java` — ganglion validation + cross-reference
- `runtime/src/main/java/io/casehub/ras/runtime/DefaultTuningStrategy.java` — unchanged
- `docs/specs/issue-60-recall-metric-external-signal/2026-08-28-recall-metric-external-signal-design.md` — parent spec
- `docs/specs/issue-59-per-ganglion-quality-metrics/2026-08-21-per-ganglion-quality-metrics-design.md` — per-ganglion quality
- `docs/specs/issue-40-ras-feedback-loop/2026-08-06-ras-feedback-loop-design.md` — feedback loop foundation
- Issue #62 — DriftDirection classification
- Issue #63 — per-ganglion missed detection signals
- Issue #64 — trigger-history cross-reference
- Issue #60 D4 — no auto-tuning for recall
- Issue #60 D6 — recall() not on QualityMetrics rationale
- Issue #60 D7 — cross-reference deferred rationale
