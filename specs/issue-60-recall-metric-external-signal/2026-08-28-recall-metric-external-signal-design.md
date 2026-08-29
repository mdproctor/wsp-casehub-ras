# Recall Metric — External Missed-Detection Signal

**Issue:** casehubio/casehub-ras#60
**Date:** 2026-08-28
**Status:** Design

## Problem

The feedback loop computes precision and noise rate from case outcomes — data RAS itself produces. Recall (the fraction of real situations that RAS successfully detects) requires knowing about false negatives: situations that should have been detected but weren't. RAS cannot compute this from its own outcome data because missed detections are invisible by definition. An external signal is required.

`DriftDirection.UNDER_SENSITIVE` is referenced aspirationally in issues #58 and #60 but was never created. The feedback loop handles over-sensitivity (noise rate > 0.5 → raise threshold) but has no under-sensitivity path.

## Architecture

Extend `OutcomeLedger` — the existing feedback data store — with missed-detection recording. Precision and recall are two projections of the same confusion matrix; they should be computed from a single data source within a single time window.

No new SPI. No new CDI event type. The integration point for future signal sources (ground truth systems, batch imports, streaming integrations) is `MissedDetectionRecorder` — the validated ingestion service. Any adapter calls it directly, mirroring how `CaseOutcomeObserver` → `OutcomeRecorder` → `OutcomeLedger.record()` works for case outcomes. `OutcomeLedger.recordMissed()` is the storage-level method; `MissedDetectionRecorder` adds validation, `recordedAt` stamping, and counter increments.

### Data Flow

```
Operator → REST endpoint → MissedDetectionRecorder → OutcomeLedger.recordMissed()
                           (validation + counters)
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
| `reportId` | UUID | Client-generated idempotency key |
| `recordedAt` | Instant | System-generated timestamp of report filing |

### Deduplication

Two deduplication mechanisms at different levels:

1. **Domain-level:** Composite key `(situationId, correlationKey, tenancyId, eventTime)` enforced by UNIQUE constraint. The same missed event reported N times counts as 1 false negative.
2. **Transport-level:** `reportId` (UUID) enforced by UNIQUE constraint. Client-generated idempotency key for safe HTTP retries — a retried request with the same `reportId` returns the existing record. The REST endpoint returns `DUPLICATE` status for both dedup paths.

## OutcomeLedger Extension

New required method on existing SPI:

```java
boolean recordMissed(MissedDetectionRecord record);
```

Returns `true` if a new record was stored, `false` if deduplicated (composite key or `reportId` conflict). `MissedDetectionRecorder` uses this to return 201 vs 200 from the REST endpoint and to increment `ras.feedback.missed` only on genuinely new records.

The existing `record()` remains `void` — its caller (`OutcomeRecorder`) is a CDI event observer with no response semantics and no selective counter. The asymmetry is intentional: different callers, different requirements.

Required, not default. There are exactly two implementations (`InMemoryOutcomeLedger`, `JpaOutcomeLedger`), both in-project. Compile-time enforcement is strictly better than a runtime `UnsupportedOperationException`.

### statistics() Extension

`OutcomeStatistics` gains a `missedCount` field. The existing `statistics()` method returns the extended record. All counts use the same `FeedbackConfig.retentionPeriod()` time window.

Implementation mechanics — `statistics()` now queries two data sources to build one record:

**JPA:** Two queries in the same `@Transactional` method. The existing `GROUP BY classification` query on `ras_outcome_record` is unchanged. A second query counts missed detections:

```sql
SELECT COUNT(*) FROM ras_missed_detection
WHERE situation_id = :sid AND tenancy_id = :tid AND event_time >= :since
```

The `event_time` filter aligns with the `closed_at >= :since` filter on outcome records — both represent when the real-world event occurred.

**In-memory:** `statistics()` counts entries from the missed detection `ConcurrentHashMap` where `eventTime >= since`, in addition to the existing outcome record counts.

### Existing Method Impact

**Unchanged:** `record()`, `lastNoiseDismissalTime()`, `countByLabel()`, `ganglionStatistics()` — no changes.

**Changed — `distinctTenancies()`:** Must union tenants from both `ras_outcome_record` and `ras_missed_detection`. Without this, tenants with missed detection reports but no case outcomes are invisible to `FeedbackUpdateJob` — their recall is never computed.

- **JPA:** `SELECT DISTINCT tenancy_id FROM ras_outcome_record WHERE situation_id = :sid UNION SELECT DISTINCT tenancy_id FROM ras_missed_detection WHERE situation_id = :sid`
- **In-memory:** Union keys from both the outcome `store` and the missed detection `ConcurrentHashMap`.

**Changed — `removeRecordsBefore()`:** Extends to clean up missed detection records alongside outcome records.

- **JPA:** Second `DELETE FROM ras_missed_detection WHERE situation_id = :sid AND event_time < :cutoff` in the same `@Transactional` method. Uses `event_time` (when the event occurred), consistent with `closed_at` on outcome records.
- **In-memory:** Iterates both stores, removes expired entries.
- **Return value:** Sum of deleted records from both sources. Callers (`FeedbackUpdateJob`) use this only for logging/metrics.

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

## REST Endpoint

RAS currently has no REST endpoints. This is the first JAX-RS resource in the project.

### Endpoint

`POST /api/ras/feedback/missed`

### Request Body

```json
{
    "situationId": "fire-risk-detector",
    "correlationKey": "ACC-12345",
    "tenancyId": "tenant-prod",
    "eventTime": "2026-08-28T14:30:00Z",
    "reportedBy": "operator@example.com",
    "reportId": "550e8400-e29b-41d4-a716-446655440000"
}
```

All fields required. `reportId` is client-generated (UUID). `eventTime` is the operator's assertion of when the missed event occurred. `recordedAt` is system-generated at ingestion time and not part of the request.

### Response

**201 Created** — new record stored:
```json
{
    "reportId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RECORDED",
    "recordedAt": "2026-08-28T14:31:02Z"
}
```

**200 OK** — idempotent duplicate (same composite key or same `reportId`):
```json
{
    "reportId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "DUPLICATE"
}
```

**400 Bad Request** — validation failure:
```json
{
    "error": "UNKNOWN_SITUATION",
    "message": "Situation 'nonexistent' is not registered"
}
```

Error codes: `UNKNOWN_SITUATION`, `FEEDBACK_NOT_CONFIGURED`, `EVENT_OUTSIDE_WINDOW`, `INVALID_REQUEST`.

### Ingestion Service

`MissedDetectionRecorder` in `runtime/` — mirrors `OutcomeRecorder`'s pattern (thin adapter → validation → ledger storage). The REST resource delegates to this service. The service:

1. Validates the request (see Validation below) — rejects with counter increment on failure
2. Sets `recordedAt` to `Instant.now()`
3. Calls `OutcomeLedger.recordMissed()` — returns `true` (new) or `false` (duplicate)
4. Increments `ras.feedback.missed` counter only when `recordMissed()` returns `true`
5. Returns the new/duplicate status to the REST resource for 201 vs 200

This separation keeps the REST resource thin (HTTP concerns only) and the ingestion logic independently testable. `MissedDetectionRecorder` is the integration point for all signal sources — future adapters (batch import, streaming consumer) call it directly, not the REST resource or `OutcomeLedger.recordMissed()`.

## Validation

On ingestion (`MissedDetectionRecorder` call path):

1. **Situation exists:** `SituationDefinitionRegistry.exists(situationId)` — reject unknown situations with `UNKNOWN_SITUATION`
2. **Feedback configured:** `registry.feedbackConfig(situationId)` is non-null — reject with `FEEDBACK_NOT_CONFIGURED`. A situation can exist but have no feedback configuration (mirroring `OutcomeRecorder.onOutcome()` which returns silently on null config). Here we reject with an actionable error rather than silently dropping the report.
3. **Temporal bounds:** `eventTime` must satisfy `now - retentionPeriod <= eventTime <= now + 30s`. The lower bound prevents records that would be immediately cleaned up by retention. The upper bound (with 30-second clock-skew tolerance) prevents future-dated events from persisting indefinitely and inflating `missedCount` in every statistics window until the clock passes them. Reject with `EVENT_OUTSIDE_WINDOW`.
4. **Deduplication:** composite key `(situationId, correlationKey, tenancyId, eventTime)` — idempotent on duplicate

Validation that the situation wasn't actually detected (cross-referencing `SituationQueryService.history()`) is deferred. Operators are asserting a fact about the real world based on domain knowledge the system doesn't have; the system records the assertion, not second-guesses it.

## Metrics

| Metric | Type | Tags | Purpose |
|--------|------|------|---------|
| `ras.feedback.recall` | gauge | `situation_id`, `tenancy_id` | Recall per situation per tenant |
| `ras.feedback.missed` | counter | `situation_id`, `tenancy_id` | Missed detection reports received |
| `ras.feedback.missed.rejected` | counter | `situation_id`, `reason` | Reports rejected by validation |

**Increment locations:**

- `ras.feedback.recall` — gauge pushed by `FeedbackUpdateJob` via `FeedbackMetrics.recordStatistics()`, alongside precision and noise rate
- `ras.feedback.missed` — counter incremented by `MissedDetectionRecorder` when `recordMissed()` returns `true` (new record, not duplicate)
- `ras.feedback.missed.rejected` — counter incremented by `MissedDetectionRecorder` when validation rejects a report

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
    UNIQUE(situation_id, correlation_key, tenancy_id, event_time),
    UNIQUE(report_id)
);

CREATE INDEX idx_missed_detection_situation ON ras_missed_detection(situation_id, tenancy_id, event_time);
```

**Schema note:** The existing `ras_outcome_record` (V7) uses `GENERATED BY DEFAULT AS IDENTITY` and `TIMESTAMP` (without time zone) for `closed_at`. The new table uses the stricter `GENERATED ALWAYS AS IDENTITY` and `TIMESTAMP WITH TIME ZONE`. Both choices are correct for the new table. Aligning V7's types is pre-existing tech debt not introduced by this spec.

`INSERT ON CONFLICT DO NOTHING` for deduplication on both the composite key and `report_id` (same pattern as `ras_outcome_record`'s `case_id` dedup). `statistics()` runs a second `COUNT(*)` query against `ras_missed_detection` in the same `@Transactional` method.

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
| `MissedDetectionResource` | `runtime/` | JAX-RS endpoint (POST /api/ras/feedback/missed) |
| `MissedDetectionRecorder` | `runtime/` | Ingestion service (validation + storage + counters) |
| `InMemoryOutcomeLedger` extension | `runtime/` | Default implementation |
| `FeedbackAnalyzer` extension | `runtime/` | Recall in analysis output |
| `FeedbackMetrics` recall gauge | `runtime/` | Micrometer integration |
| `FeedbackUpdateJob` recall push | `runtime/` | Scheduled update |
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
| `MissedDetectionRecorderTest` | Unit | Validation (unknown situation, out-of-window), dedup idempotent, counter increments |
| `MissedDetectionResourceTest` | Integration | HTTP 201/200/400 responses, request body parsing, error response format |
| `MissedDetectionIngestionIT` | Integration | End-to-end: POST → validation → storage → statistics reflect missedCount |

## Epistemological Limitations

Operator-reported recall is inherently biased. Operators can only report misses they independently notice — systematic blind spots (entire event patterns the system never processes) remain invisible. This is not a flaw in the design; it's an inherent property of any external false-negative signal. Adding future sources (ground truth systems, periodic audit sampling) mitigates but never eliminates this bias. The recall metric should be interpreted as a lower bound on the true miss rate.

## Scope

Issue #60 specifies four requirements: ingestion path, recall computation, UNDER_SENSITIVE wiring, and recall gauge. This spec delivers three of four — UNDER_SENSITIVE wiring is deferred to the DriftDirection design (#62). Issue #60 remains open until that work is complete.

## Known Gaps

1. **Per-ganglion recall:** Requires ganglion-level missed detection signals (#63)
2. **DriftDirection classification:** Deferred to separate design (#62)
3. **F1 score gauge:** Operators compute externally from precision + recall gauges; one-line addition when needed
4. **Trigger-history cross-reference:** Validation that the situation wasn't actually detected is deferred (#64) — operators assert domain facts the system can't verify

## References

- `OutcomeLedger.java` — SPI being extended
- `OutcomeRecord.java` — existing outcome record (NOT being modified)
- `OutcomeStatistics.java` — gains `missedCount` + `recall()`
- `QualityMetrics.java` — NOT gaining `recall()` (see design rationale)
- `FeedbackAnalyzer.java` — gains recall in analysis output
- `FeedbackMetrics.java` — gains recall gauge + missed counters
- `FeedbackUpdateJob.java` — pushes recall gauge
- `InMemoryOutcomeLedger.java` — runtime/ default impl
- `JpaOutcomeLedger.java` — persistence-jpa/ impl
- `MissedDetectionResource.java` — new REST endpoint
- `MissedDetectionRecorder.java` — new ingestion service
- `SituationDefinitionRegistry.java` — `exists()` method for validation
- `AbstractOutcomeLedgerContractTest.java` — contract test extension
- `DefaultTuningStrategy.java` — unchanged (no auto-tuning for recall)
- Issue #60 — recall metric requirement
- Issue #58 — three signal sources identified (closed as duplicate)
- Issue #40 — feedback loop design (requirement 4: precision/recall per ganglion)
- Issue #62 — DriftDirection classification (deferred from this spec)
- Issue #63 — per-ganglion recall (deferred from this spec)
- Issue #64 — trigger-history cross-reference (deferred from this spec)
- `2026-08-06-ras-feedback-loop-design.md` — feedback loop spec
- `2026-08-21-per-ganglion-quality-metrics-design.md` — quality metrics spec
