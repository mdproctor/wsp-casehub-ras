# Recall Metric Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #60 — Recall metric — external missed-detection signal
**Issue group:** #60

**Goal:** Enable operators to report missed detections, compute recall alongside precision, and surface as a Micrometer gauge.

**Architecture:** Extend `OutcomeLedger` with `recordMissed(MissedDetectionRecord)`. New `MissedDetectionRecord` type in api/ (separate from `OutcomeRecord`). `OutcomeStatistics` gains `missedCount` + `recall()`. `MissedDetectionRecorder` in runtime/ handles validation and counter increments. REST endpoint `POST /api/ras/feedback/missed` for operator input. JPA persistence via Flyway V9 migration.

**Tech Stack:** Java 21, Quarkus (CDI, JAX-RS, Scheduler), JPA/Hibernate, Flyway, Micrometer, JUnit 5, AssertJ

## Global Constraints

- `recall()` on `OutcomeStatistics` only, NOT on `QualityMetrics` — prevents misleading 1.0 on per-ganglion stats
- Same `FeedbackConfig.retentionPeriod()` time window for precision and recall
- `OutcomeRecord` is NOT modified — non-null contract, dedup semantics preserved
- No auto-tuning for recall — metric only, `DefaultTuningStrategy` unchanged
- DriftDirection deferred — not in scope
- `recordMissed()` is a required method, not default — compile-time enforcement

---

## Batch 1: API + InMemory Foundation

### Task 1: MissedDetectionRecord + OutcomeStatistics extension + OutcomeLedger.recordMissed

**Files:**
- Create: `api/src/main/java/io/casehub/ras/api/MissedDetectionRecord.java`
- Modify: `api/src/main/java/io/casehub/ras/api/OutcomeStatistics.java`
- Modify: `api/src/main/java/io/casehub/ras/api/OutcomeLedger.java`
- Test: `api/src/test/java/io/casehub/ras/api/MissedDetectionRecordTest.java`
- Test: `api/src/test/java/io/casehub/ras/api/OutcomeStatisticsTest.java`

**Interfaces:**
- Produces: `MissedDetectionRecord(situationId, correlationKey, tenancyId, eventTime, reportedBy, reportId, recordedAt)` — immutable record
- Produces: `OutcomeStatistics.missedCount` field, `OutcomeStatistics.recall()` method
- Produces: `OutcomeLedger.recordMissed(MissedDetectionRecord)` — returns `boolean` (true=new, false=dedup)

#### MissedDetectionRecord

- [ ] **Step 1: Write test — record construction and field validation**

```java
@Test
void record_construction() {
    var record = new MissedDetectionRecord("sit-1", "key-1", "tenant-a",
            Instant.parse("2026-08-28T10:00:00Z"), "operator@example.com",
            UUID.randomUUID(), Instant.now());
    assertThat(record.situationId()).isEqualTo("sit-1");
    assertThat(record.correlationKey()).isEqualTo("key-1");
    assertThat(record.tenancyId()).isEqualTo("tenant-a");
    assertThat(record.reportedBy()).isEqualTo("operator@example.com");
}

@Test
void record_rejects_null_situationId() {
    assertThatThrownBy(() -> new MissedDetectionRecord(null, "key", "tenant",
            Instant.now(), "operator", UUID.randomUUID(), Instant.now()))
        .isInstanceOf(NullPointerException.class);
}

@Test
void record_rejects_null_reportId() {
    assertThatThrownBy(() -> new MissedDetectionRecord("sit", "key", "tenant",
            Instant.now(), "operator", null, Instant.now()))
        .isInstanceOf(NullPointerException.class);
}
```

- [ ] **Step 2: Run tests — verify they fail (class doesn't exist)**

Run: `mvn --batch-mode test -pl api -Dtest=MissedDetectionRecordTest`

- [ ] **Step 3: Implement MissedDetectionRecord**

Create via `ide_create_file`:

```java
package io.casehub.ras.api;

import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

public record MissedDetectionRecord(
        String situationId,
        String correlationKey,
        String tenancyId,
        Instant eventTime,
        String reportedBy,
        UUID reportId,
        Instant recordedAt
) {
    public MissedDetectionRecord {
        Objects.requireNonNull(situationId, "situationId");
        Objects.requireNonNull(correlationKey, "correlationKey");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(eventTime, "eventTime");
        Objects.requireNonNull(reportedBy, "reportedBy");
        Objects.requireNonNull(reportId, "reportId");
        Objects.requireNonNull(recordedAt, "recordedAt");
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=MissedDetectionRecordTest`

#### OutcomeStatistics.missedCount + recall()

- [ ] **Step 5: Write test — recall computation**

Add to existing `OutcomeStatisticsTest` (or create if absent):

```java
@Test
void recall_computed_from_confirmed_and_missed() {
    var stats = new OutcomeStatistics("sit-1", "tenant-a", 10, 2, 7, 1,
            Instant.parse("2026-01-01T00:00:00Z"), 3);
    assertThat(stats.recall()).isCloseTo(0.7, within(0.001));
}

@Test
void recall_nan_when_no_confirmed_or_missed() {
    var stats = new OutcomeStatistics("sit-1", "tenant-a", 5, 0, 0, 5,
            Instant.parse("2026-01-01T00:00:00Z"), 0);
    assertThat(stats.recall()).isNaN();
}

@Test
void recall_one_when_no_missed() {
    var stats = new OutcomeStatistics("sit-1", "tenant-a", 5, 0, 5, 0,
            Instant.parse("2026-01-01T00:00:00Z"), 0);
    assertThat(stats.recall()).isEqualTo(1.0);
}
```

- [ ] **Step 6: Run tests — verify they fail (no missedCount field)**

Run: `mvn --batch-mode test -pl api -Dtest=OutcomeStatisticsTest`

- [ ] **Step 7: Add missedCount to OutcomeStatistics + recall()**

Use `ide_edit_member` to update the record declaration. Add `long missedCount` as the 8th component after `windowStart`. Add `recall()` method. Update the existing 7-arg constructor as a backwards-compat convenience that passes `0` for missedCount:

```java
public OutcomeStatistics(String situationId, String tenancyId, long totalOutcomes,
                         long noiseCount, long confirmedCount, long neutralCount,
                         Instant windowStart) {
    this(situationId, tenancyId, totalOutcomes, noiseCount, confirmedCount,
         neutralCount, windowStart, 0);
}

public double recall() {
    long decisive = confirmedCount + missedCount;
    return decisive == 0 ? Double.NaN : confirmedCount / (double) decisive;
}
```

- [ ] **Step 8: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=OutcomeStatisticsTest`

#### OutcomeLedger.recordMissed

- [ ] **Step 9: Add recordMissed to OutcomeLedger interface**

Use `ide_insert_member` on `OutcomeLedger`:

```java
boolean recordMissed(MissedDetectionRecord record);
```

- [ ] **Step 10: Verify compile — expect failures in implementations**

Run: `mvn --batch-mode compile -DskipTests`
Expected: compile errors in `InMemoryOutcomeLedger` and `JpaOutcomeLedger`

- [ ] **Step 11: Add stub implementations to fix compilation**

Add temporary stubs to both implementations:

InMemoryOutcomeLedger:
```java
@Override
public boolean recordMissed(MissedDetectionRecord record) {
    throw new UnsupportedOperationException("TODO");
}
```

JpaOutcomeLedger: same stub.

- [ ] **Step 12: Verify compile passes**

Run: `mvn --batch-mode compile -DskipTests`

- [ ] **Step 13: Commit**

```bash
git add api/ runtime/src/main/java/io/casehub/ras/runtime/InMemoryOutcomeLedger.java persistence-jpa/
git commit -m "feat(#60): API types — MissedDetectionRecord, OutcomeStatistics.recall(), OutcomeLedger.recordMissed()

Adds MissedDetectionRecord record type. Extends OutcomeStatistics with
missedCount + recall() (on OutcomeStatistics only, not QualityMetrics).
Adds recordMissed(MissedDetectionRecord) to OutcomeLedger SPI.

Refs #60"
```

---

### Task 2: InMemoryOutcomeLedger extension + contract tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/InMemoryOutcomeLedger.java`
- Modify: `api/src/test/java/io/casehub/ras/api/AbstractOutcomeLedgerContractTest.java`

**Interfaces:**
- Consumes: `MissedDetectionRecord` (Task 1), `OutcomeLedger.recordMissed()` (Task 1)
- Produces: Working InMemory implementation with dedup, statistics, distinctTenancies union, removeRecordsBefore

- [ ] **Step 1: Write contract tests for recordMissed**

Add to `AbstractOutcomeLedgerContractTest`:

```java
@Test
void recordMissed_stores_and_counts_in_statistics() {
    ledger.recordMissed(missedDetection("sit-1", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));
    var stats = ledger.statistics("sit-1", "tenant-a",
            Instant.now().minus(Duration.ofHours(1)));
    assertThat(stats.missedCount()).isEqualTo(1);
}

@Test
void recordMissed_dedup_on_composite_key() {
    Instant eventTime = Instant.now().minus(Duration.ofMinutes(5));
    boolean first = ledger.recordMissed(missedDetection("sit-1", "key-1", "tenant-a", eventTime));
    boolean second = ledger.recordMissed(missedDetection("sit-1", "key-1", "tenant-a", eventTime));
    assertThat(first).isTrue();
    assertThat(second).isFalse();
    var stats = ledger.statistics("sit-1", "tenant-a",
            Instant.now().minus(Duration.ofHours(1)));
    assertThat(stats.missedCount()).isEqualTo(1);
}

@Test
void recordMissed_dedup_on_reportId() {
    UUID reportId = UUID.randomUUID();
    var r1 = new MissedDetectionRecord("sit-1", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5)), "op", reportId, Instant.now());
    var r2 = new MissedDetectionRecord("sit-1", "key-2", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(3)), "op", reportId, Instant.now());
    assertThat(ledger.recordMissed(r1)).isTrue();
    assertThat(ledger.recordMissed(r2)).isFalse();
}

@Test
void distinctTenancies_includes_missed_only_tenants() {
    ledger.recordMissed(missedDetection("sit-1", "key-1", "tenant-missed",
            Instant.now().minus(Duration.ofMinutes(5))));
    assertThat(ledger.distinctTenancies("sit-1")).contains("tenant-missed");
}

@Test
void removeRecordsBefore_cleans_missed_detections() {
    Instant old = Instant.now().minus(Duration.ofHours(2));
    ledger.recordMissed(missedDetection("sit-1", "key-1", "tenant-a", old));
    int removed = ledger.removeRecordsBefore("sit-1", Instant.now().minus(Duration.ofHours(1)));
    assertThat(removed).isGreaterThan(0);
    var stats = ledger.statistics("sit-1", "tenant-a", Instant.now().minus(Duration.ofHours(3)));
    assertThat(stats.missedCount()).isEqualTo(0);
}

@Test
void recall_computed_from_outcomes_and_missed() {
    ledger.record(outcome("sit-1", "key-1", "tenant-a", "resolved",
            OutcomeClassification.CONFIRMED));
    ledger.recordMissed(missedDetection("sit-1", "key-2", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));
    var stats = ledger.statistics("sit-1", "tenant-a",
            Instant.now().minus(Duration.ofHours(1)));
    assertThat(stats.confirmedCount()).isEqualTo(1);
    assertThat(stats.missedCount()).isEqualTo(1);
    assertThat(stats.recall()).isCloseTo(0.5, within(0.001));
}
```

Add helper to the contract test:

```java
protected MissedDetectionRecord missedDetection(String situationId, String correlationKey,
                                                  String tenancyId, Instant eventTime) {
    return new MissedDetectionRecord(situationId, correlationKey, tenancyId,
            eventTime, "test-operator", UUID.randomUUID(), Instant.now());
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=AbstractOutcomeLedgerContractTest`
Expected: compilation errors or UnsupportedOperationException

- [ ] **Step 3: Implement InMemoryOutcomeLedger.recordMissed**

Add fields:

```java
private final ConcurrentHashMap<MissedDetectionKey, MissedDetectionRecord> missedStore = new ConcurrentHashMap<>();
private final Set<UUID> seenReportIds = ConcurrentHashMap.newKeySet();

private record MissedDetectionKey(String situationId, String correlationKey, String tenancyId, Instant eventTime) {}
```

Implement `recordMissed`:

```java
@Override
public boolean recordMissed(MissedDetectionRecord record) {
    if (!seenReportIds.add(record.reportId())) {
        return false;
    }
    var key = new MissedDetectionKey(record.situationId(), record.correlationKey(),
                                      record.tenancyId(), record.eventTime());
    return missedStore.putIfAbsent(key, record) == null;
}
```

Update `statistics()` to include missedCount:

```java
long missedCount = missedStore.values().stream()
        .filter(r -> r.situationId().equals(situationId)
                && r.tenancyId().equals(tenancyId)
                && !r.eventTime().isBefore(since))
        .count();
return new OutcomeStatistics(situationId, tenancyId, total, noise, confirmed, neutral, since, missedCount);
```

Update `distinctTenancies()` to union missed detection tenants:

```java
missedStore.values().stream()
        .filter(r -> r.situationId().equals(situationId))
        .map(MissedDetectionRecord::tenancyId)
        .forEach(tenancies::add);
```

Update `removeRecordsBefore()` to clean missed detections:

```java
int missedRemoved = 0;
var missedIt = missedStore.entrySet().iterator();
while (missedIt.hasNext()) {
    var entry = missedIt.next();
    if (entry.getKey().situationId().equals(situationId)
            && entry.getValue().eventTime().isBefore(cutoff)) {
        missedIt.remove();
        seenReportIds.remove(entry.getValue().reportId());
        missedRemoved++;
    }
}
return removed + missedRemoved;
```

- [ ] **Step 4: Run contract tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=InMemoryOutcomeLedgerContractTest`

- [ ] **Step 5: Commit**

```bash
git add api/ runtime/
git commit -m "feat(#60): InMemoryOutcomeLedger — recordMissed with dedup, statistics, retention

Implements recordMissed with composite-key + reportId dedup. statistics()
includes missedCount. distinctTenancies() unions missed-only tenants.
removeRecordsBefore() cleans missed detection records. Contract tests
extended.

Refs #60"
```

---

## Batch 2: Ingestion + Metrics

### Task 3: MissedDetectionRecorder + FeedbackMetrics recall gauge + FeedbackUpdateJob

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionRecorder.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionRecorderTest.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackMetrics.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackUpdateJob.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackUpdateJobTest.java`

**Interfaces:**
- Consumes: `OutcomeLedger.recordMissed()` (Task 1), `SituationDefinitionRegistry.exists()`, `FeedbackConfig`
- Produces: `MissedDetectionRecorder.record(MissedDetectionRecord)` — validated ingestion, returns `RecordResult(boolean isNew, String rejectionReason)`
- Produces: `ras.feedback.recall` gauge, `ras.feedback.missed` counter, `ras.feedback.missed.rejected` counter

#### MissedDetectionRecorder

- [ ] **Step 1: Write tests — validation and recording**

```java
@Test
void record_stores_valid_missed_detection() {
    var result = recorder.record(missedDetection("sit-1", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));
    assertThat(result.accepted()).isTrue();
    assertThat(result.isNew()).isTrue();
}

@Test
void record_rejects_unknown_situation() {
    var result = recorder.record(missedDetection("unknown-sit", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));
    assertThat(result.accepted()).isFalse();
    assertThat(result.rejectionReason()).isEqualTo("UNKNOWN_SITUATION");
}

@Test
void record_rejects_situation_without_feedback_config() {
    var result = recorder.record(missedDetection("sit-no-feedback", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));
    assertThat(result.accepted()).isFalse();
    assertThat(result.rejectionReason()).isEqualTo("FEEDBACK_NOT_CONFIGURED");
}

@Test
void record_rejects_event_outside_retention_window() {
    var result = recorder.record(missedDetection("sit-1", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofDays(60))));
    assertThat(result.accepted()).isFalse();
    assertThat(result.rejectionReason()).isEqualTo("EVENT_OUTSIDE_WINDOW");
}

@Test
void record_rejects_future_event_beyond_skew_tolerance() {
    var result = recorder.record(missedDetection("sit-1", "key-1", "tenant-a",
            Instant.now().plus(Duration.ofMinutes(5))));
    assertThat(result.accepted()).isFalse();
    assertThat(result.rejectionReason()).isEqualTo("EVENT_OUTSIDE_WINDOW");
}

@Test
void record_returns_not_new_on_duplicate() {
    Instant eventTime = Instant.now().minus(Duration.ofMinutes(5));
    recorder.record(missedDetection("sit-1", "key-1", "tenant-a", eventTime));
    var result = recorder.record(missedDetection("sit-1", "key-1", "tenant-a", eventTime));
    assertThat(result.accepted()).isTrue();
    assertThat(result.isNew()).isFalse();
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionRecorderTest`

- [ ] **Step 3: Implement MissedDetectionRecorder**

```java
@ApplicationScoped
public class MissedDetectionRecorder {

    private static final Duration FUTURE_SKEW_TOLERANCE = Duration.ofSeconds(30);

    private final OutcomeLedger ledger;
    private final SituationDefinitionRegistry registry;
    private final RasMetrics metrics;

    @Inject
    public MissedDetectionRecorder(OutcomeLedger ledger,
                                    SituationDefinitionRegistry registry,
                                    RasMetrics metrics) {
        this.ledger = ledger;
        this.registry = registry;
        this.metrics = metrics;
    }

    public RecordResult record(MissedDetectionRecord record) {
        if (!registry.exists(record.situationId())) {
            metrics.missedRejected(record.situationId(), "UNKNOWN_SITUATION");
            return RecordResult.rejected("UNKNOWN_SITUATION");
        }
        FeedbackConfig config = registry.feedbackConfig(record.situationId());
        if (config == null) {
            metrics.missedRejected(record.situationId(), "FEEDBACK_NOT_CONFIGURED");
            return RecordResult.rejected("FEEDBACK_NOT_CONFIGURED");
        }
        Instant now = Instant.now();
        Instant windowStart = now.minus(config.retentionPeriod());
        if (record.eventTime().isBefore(windowStart)
                || record.eventTime().isAfter(now.plus(FUTURE_SKEW_TOLERANCE))) {
            metrics.missedRejected(record.situationId(), "EVENT_OUTSIDE_WINDOW");
            return RecordResult.rejected("EVENT_OUTSIDE_WINDOW");
        }
        boolean isNew = ledger.recordMissed(record);
        if (isNew) {
            metrics.missedRecorded(record.situationId(), record.tenancyId());
        }
        return RecordResult.accepted(isNew);
    }

    public record RecordResult(boolean accepted, boolean isNew, String rejectionReason) {
        static RecordResult accepted(boolean isNew) { return new RecordResult(true, isNew, null); }
        static RecordResult rejected(String reason) { return new RecordResult(false, false, reason); }
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionRecorderTest`

#### FeedbackMetrics + FeedbackUpdateJob recall gauge

- [ ] **Step 5: Add recall gauge + missed counters to FeedbackMetrics**

Use `ide_insert_member` on `FeedbackMetrics`:

```java
public void missedRecorded(String situationId, String tenancyId) {
    counter("ras.feedback.missed", "situation_id", situationId, "tenancy_id", tenancyId);
}

public void missedRejected(String situationId, String reason) {
    counter("ras.feedback.missed.rejected", "situation_id", situationId, "reason", reason);
}
```

Update `recordStatistics()` to add recall gauge:

```java
double recall = stats.recall();
if (!Double.isNaN(recall)) {
    setGauge("ras.feedback.recall",
             Tags.of("situation_id", situationId, "tenancy_id", tenancyId), recall);
}
```

- [ ] **Step 6: Write test — FeedbackUpdateJob pushes recall**

Add to `FeedbackUpdateJobTest`:

```java
@Test
void updateFeedback_pushes_recall_gauge() {
    // Setup: outcome + missed detection in store
    ledger.record(outcome("sit-1", "key-1", "tenant-a", "resolved",
            OutcomeClassification.CONFIRMED));
    ledger.recordMissed(missedDetection("sit-1", "key-2", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5))));

    job.updateFeedback();

    assertThat(meterRegistry.find("ras.feedback.recall")
            .tag("situation_id", "sit-1")
            .tag("tenancy_id", "tenant-a")
            .gauge()).isNotNull();
    assertThat(meterRegistry.find("ras.feedback.recall")
            .tag("situation_id", "sit-1")
            .gauge().value()).isCloseTo(0.5, within(0.001));
}
```

- [ ] **Step 7: Run all feedback tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest='FeedbackUpdateJobTest,MissedDetectionRecorderTest'`

- [ ] **Step 8: Commit**

```bash
git add runtime/
git commit -m "feat(#60): MissedDetectionRecorder + recall gauge

Validated ingestion service with situation-exists, feedback-configured,
temporal-bounds, and future-skew checks. FeedbackMetrics gains recall
gauge + missed/rejected counters. FeedbackUpdateJob pushes recall
alongside precision and noise rate.

Refs #60"
```

---

## Batch 3: JPA Persistence + REST Endpoint

### Task 4: Flyway V9 + JpaOutcomeLedger extension

**Files:**
- Create: `persistence-jpa/src/main/resources/db/ras/migration/V9__missed_detection.sql`
- Modify: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedger.java`
- Create: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/MissedDetectionEntity.java`

**Interfaces:**
- Consumes: `OutcomeLedger.recordMissed()` (Task 1), `MissedDetectionRecord` (Task 1)
- Produces: JPA persistence with dedup, statistics integration, retention cleanup

- [ ] **Step 1: Create Flyway V9 migration**

```sql
CREATE TABLE ras_missed_detection (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    situation_id    VARCHAR(255) NOT NULL,
    correlation_key VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    event_time      TIMESTAMP WITH TIME ZONE NOT NULL,
    reported_by     VARCHAR(255) NOT NULL,
    report_id       UUID NOT NULL,
    recorded_at     TIMESTAMP WITH TIME ZONE NOT NULL,
    UNIQUE(situation_id, correlation_key, tenancy_id, event_time),
    UNIQUE(report_id)
);

CREATE INDEX idx_missed_detection_situation ON ras_missed_detection(situation_id, tenancy_id, event_time);
```

- [ ] **Step 2: Create MissedDetectionEntity**

```java
@Entity
@Table(name = "ras_missed_detection")
public class MissedDetectionEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "situation_id", nullable = false)
    private String situationId;
    @Column(name = "correlation_key", nullable = false)
    private String correlationKey;
    @Column(name = "tenancy_id", nullable = false)
    private String tenancyId;
    @Column(name = "event_time", nullable = false)
    private Instant eventTime;
    @Column(name = "reported_by", nullable = false)
    private String reportedBy;
    @Column(name = "report_id", nullable = false)
    private UUID reportId;
    @Column(name = "recorded_at", nullable = false)
    private Instant recordedAt;
    // getters, setters, no-arg constructor
}
```

- [ ] **Step 3: Implement JpaOutcomeLedger.recordMissed**

Replace stub with INSERT ON CONFLICT:

```java
@Override
@Transactional(TxType.REQUIRED)
public boolean recordMissed(MissedDetectionRecord record) {
    int inserted = em.createNativeQuery(
            "INSERT INTO ras_missed_detection (situation_id, correlation_key, tenancy_id, " +
            "event_time, reported_by, report_id, recorded_at) " +
            "VALUES (:sid, :ck, :tid, :et, :rb, :rid, :ra) " +
            "ON CONFLICT DO NOTHING")
        .setParameter("sid", record.situationId())
        .setParameter("ck", record.correlationKey())
        .setParameter("tid", record.tenancyId())
        .setParameter("et", record.eventTime())
        .setParameter("rb", record.reportedBy())
        .setParameter("rid", record.reportId())
        .setParameter("ra", record.recordedAt())
        .executeUpdate();
    return inserted > 0;
}
```

Update `statistics()` to include missedCount:

```java
long missedCount = ((Number) em.createNativeQuery(
        "SELECT COUNT(*) FROM ras_missed_detection " +
        "WHERE situation_id = :sid AND tenancy_id = :tid AND event_time >= :since")
    .setParameter("sid", situationId)
    .setParameter("tid", tenancyId)
    .setParameter("since", since)
    .getSingleResult()).longValue();
// Pass missedCount to OutcomeStatistics 8-arg constructor
```

Update `distinctTenancies()` to union:

```java
@SuppressWarnings("unchecked")
List<String> missedTenancies = em.createNativeQuery(
        "SELECT DISTINCT tenancy_id FROM ras_missed_detection WHERE situation_id = :sid")
    .setParameter("sid", situationId)
    .getResultList();
tenancies.addAll(missedTenancies);
```

Update `removeRecordsBefore()`:

```java
int missedRemoved = em.createNativeQuery(
        "DELETE FROM ras_missed_detection WHERE situation_id = :sid AND event_time < :cutoff")
    .setParameter("sid", situationId)
    .setParameter("cutoff", cutoff)
    .executeUpdate();
return outcomeRemoved + missedRemoved;
```

- [ ] **Step 4: Run JPA tests**

Run: `mvn --batch-mode test -pl persistence-jpa`

- [ ] **Step 5: Commit**

```bash
git add persistence-jpa/
git commit -m "feat(#60): JPA persistence — Flyway V9, MissedDetectionEntity, JpaOutcomeLedger extension

Flyway V9 creates ras_missed_detection table with composite-key + reportId
UNIQUE constraints. INSERT ON CONFLICT for dedup. statistics() includes
missedCount. distinctTenancies() unions. removeRecordsBefore() cleans both.

Refs #60"
```

---

### Task 5: REST endpoint + integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionResource.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionResourceTest.java`

**Interfaces:**
- Consumes: `MissedDetectionRecorder.record()` (Task 3)
- Produces: `POST /api/ras/feedback/missed` — 201/200/400 responses

- [ ] **Step 1: Write test — REST endpoint responses**

```java
@Test
void post_valid_missed_detection_returns_201() {
    var result = resource.reportMissed(new MissedDetectionRequest(
            "sit-1", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5)),
            "operator@example.com",
            UUID.randomUUID()));
    assertThat(result.getStatus()).isEqualTo(201);
}

@Test
void post_duplicate_returns_200() {
    Instant eventTime = Instant.now().minus(Duration.ofMinutes(5));
    UUID reportId = UUID.randomUUID();
    resource.reportMissed(new MissedDetectionRequest(
            "sit-1", "key-1", "tenant-a", eventTime, "op", reportId));
    var result = resource.reportMissed(new MissedDetectionRequest(
            "sit-1", "key-1", "tenant-a", eventTime, "op", UUID.randomUUID()));
    assertThat(result.getStatus()).isEqualTo(200);
}

@Test
void post_unknown_situation_returns_400() {
    var result = resource.reportMissed(new MissedDetectionRequest(
            "unknown", "key-1", "tenant-a",
            Instant.now().minus(Duration.ofMinutes(5)),
            "op", UUID.randomUUID()));
    assertThat(result.getStatus()).isEqualTo(400);
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionResourceTest`

- [ ] **Step 3: Implement MissedDetectionResource**

```java
@Path("/api/ras/feedback/missed")
@ApplicationScoped
public class MissedDetectionResource {

    @Inject MissedDetectionRecorder recorder;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public Response reportMissed(MissedDetectionRequest request) {
        var record = new MissedDetectionRecord(
                request.situationId(), request.correlationKey(), request.tenancyId(),
                request.eventTime(), request.reportedBy(), request.reportId(), Instant.now());

        var result = recorder.record(record);
        if (!result.accepted()) {
            return Response.status(400)
                    .entity(Map.of("error", result.rejectionReason(),
                                   "message", describeRejection(result.rejectionReason(), request.situationId())))
                    .build();
        }
        if (result.isNew()) {
            return Response.status(201)
                    .entity(Map.of("reportId", record.reportId(),
                                   "status", "RECORDED",
                                   "recordedAt", record.recordedAt()))
                    .build();
        }
        return Response.ok(Map.of("reportId", record.reportId(), "status", "DUPLICATE")).build();
    }

    private String describeRejection(String reason, String situationId) {
        return switch (reason) {
            case "UNKNOWN_SITUATION" -> "Situation '" + situationId + "' is not registered";
            case "FEEDBACK_NOT_CONFIGURED" -> "Situation '" + situationId + "' has no feedback configuration";
            case "EVENT_OUTSIDE_WINDOW" -> "eventTime is outside the retention window";
            default -> reason;
        };
    }

    public record MissedDetectionRequest(
            String situationId, String correlationKey, String tenancyId,
            Instant eventTime, String reportedBy, UUID reportId) {}
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionResourceTest`

- [ ] **Step 5: Run full test suite**

Run: `mvn --batch-mode test`

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "feat(#60): REST endpoint POST /api/ras/feedback/missed

First JAX-RS resource in RAS. Delegates to MissedDetectionRecorder for
validation. Returns 201 (new), 200 (duplicate), 400 (validation failure)
with structured error responses.

Closes #60"
```

---

## References

- `2026-08-28-recall-metric-external-signal-design.md` — design spec
- `OutcomeLedger.java` — SPI being extended
- `OutcomeStatistics.java` — gains missedCount + recall()
- `InMemoryOutcomeLedger.java` — runtime/ default impl
- `JpaOutcomeLedger.java` — persistence-jpa/ impl
- `FeedbackAnalyzer.java` — analyze() returns OutcomeStatistics (no code change needed — statistics() already returns missedCount)
- `FeedbackMetrics.java` — gains recall gauge
- `FeedbackUpdateJob.java` — pushes recall gauge
- `AbstractOutcomeLedgerContractTest.java` — contract test extension
- `OutcomeRecorder.java` — pattern reference for MissedDetectionRecorder
- GitHub #60 — Recall metric issue
