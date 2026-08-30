# Feedback Quality Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #62 — DriftDirection classification model design
**Issue group:** #62, #63, #64

**Goal:** Add drift direction classification, per-ganglion missed detection signals with recall computation, and trigger-history cross-reference advisory to the RAS feedback loop.

**Architecture:** Extends existing feedback loop types in api/ (FeedbackConfig, FeedbackTuningStrategy, MissedDetectionRecord, GanglionOutcomeStatistics) with new fields and methods. DriftDirection is a 5-value enum classified by FeedbackTuningStrategy.classifyDrift() and published as a state-gauge set. Per-ganglion recall uses optional ganglionIds on MissedDetectionRecord with JSONB persistence. Trigger cross-reference is an advisory warning in MissedDetectionRecorder using SituationQueryService.history().

**Tech Stack:** Java 21 records, Quarkus CDI, Micrometer gauges, Flyway V10, PostgreSQL JSONB, JAX-RS

## Global Constraints

- Java records for immutable domain types — backwards-compatible constructors for all extended records
- TDD: failing test → verify fail → implement → verify pass → commit
- `mvn --batch-mode install` must pass after each task
- IntelliJ MCP for all code navigation and structural editing
- No new SPIs — extend existing OutcomeLedger and FeedbackTuningStrategy
- recall() stays OFF QualityMetrics interface (false-1.0 prevention)

---

## Batch 1: API Foundation

### Task 1: DriftDirection enum + FeedbackConfig extension

**Files:**
- Create: `api/src/main/java/io/casehub/ras/api/DriftDirection.java`
- Modify: `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java`
- Test: `api/src/test/java/io/casehub/ras/api/DriftDirectionTest.java`
- Test: `api/src/test/java/io/casehub/ras/api/FeedbackConfigDriftTest.java`
- Modify: `api/src/test/java/io/casehub/ras/api/FeedbackConfigTest.java` (update for new constructor)

**Interfaces:**
- Produces: `DriftDirection` enum with 5 values (OVER_SENSITIVE, UNDER_SENSITIVE, BOTH_DRIFTING, STABLE, INSUFFICIENT_DATA)
- Produces: `FeedbackConfig` with 3 new fields: `double overSensitiveThreshold()`, `double underSensitiveThreshold()`, `Duration crossRefWindow()`
- Produces: `FeedbackConfig` 6-arg backwards-compatible constructor (defaults: 0.5, 0.5, PT1H)

- [ ] **Step 1: Write DriftDirection enum test**

```java
// api/src/test/java/io/casehub/ras/api/DriftDirectionTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DriftDirectionTest {
    @Test
    void enumHasFiveValues() {
        assertEquals(5, DriftDirection.values().length);
        assertNotNull(DriftDirection.OVER_SENSITIVE);
        assertNotNull(DriftDirection.UNDER_SENSITIVE);
        assertNotNull(DriftDirection.BOTH_DRIFTING);
        assertNotNull(DriftDirection.STABLE);
        assertNotNull(DriftDirection.INSUFFICIENT_DATA);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=DriftDirectionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — DriftDirection class does not exist

- [ ] **Step 3: Create DriftDirection enum**

```java
// api/src/main/java/io/casehub/ras/api/DriftDirection.java
package io.casehub.ras.api;

public enum DriftDirection {
    OVER_SENSITIVE,
    UNDER_SENSITIVE,
    BOTH_DRIFTING,
    STABLE,
    INSUFFICIENT_DATA
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=DriftDirectionTest`
Expected: PASS

- [ ] **Step 5: Write FeedbackConfig extension tests**

```java
// api/src/test/java/io/casehub/ras/api/FeedbackConfigDriftTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class FeedbackConfigDriftTest {
    @Test
    void nineArgConstructorStoresNewFields() {
        var config = new FeedbackConfig(
                Set.of("dismissed"), Set.of("escalated"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), true,
                0.4, 0.7, Duration.ofHours(2));
        assertEquals(0.4, config.overSensitiveThreshold());
        assertEquals(0.7, config.underSensitiveThreshold());
        assertEquals(Duration.ofHours(2), config.crossRefWindow());
    }

    @Test
    void sixArgConstructorDefaultsNewFields() {
        var config = new FeedbackConfig(
                Set.of("dismissed"), Set.of("escalated"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), false);
        assertEquals(0.5, config.overSensitiveThreshold());
        assertEquals(0.5, config.underSensitiveThreshold());
        assertEquals(Duration.ofHours(1), config.crossRefWindow());
    }

    @Test
    void overSensitiveThresholdValidation() {
        assertThrows(IllegalArgumentException.class, () ->
                new FeedbackConfig(Set.of("n"), Set.of("c"),
                        Duration.ofHours(1), 0.1, Duration.ofDays(1), false,
                        0.0, 0.5, Duration.ofHours(1)));
        assertThrows(IllegalArgumentException.class, () ->
                new FeedbackConfig(Set.of("n"), Set.of("c"),
                        Duration.ofHours(1), 0.1, Duration.ofDays(1), false,
                        1.1, 0.5, Duration.ofHours(1)));
    }

    @Test
    void underSensitiveThresholdValidation() {
        assertThrows(IllegalArgumentException.class, () ->
                new FeedbackConfig(Set.of("n"), Set.of("c"),
                        Duration.ofHours(1), 0.1, Duration.ofDays(1), false,
                        0.5, 0.0, Duration.ofHours(1)));
    }

    @Test
    void crossRefWindowValidation() {
        assertThrows(IllegalArgumentException.class, () ->
                new FeedbackConfig(Set.of("n"), Set.of("c"),
                        Duration.ofHours(1), 0.1, Duration.ofDays(1), false,
                        0.5, 0.5, Duration.ZERO));
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=FeedbackConfigDriftTest`
Expected: FAIL — 9-arg constructor does not exist

- [ ] **Step 7: Extend FeedbackConfig**

Modify `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java`:

Add 3 new fields to the record: `double overSensitiveThreshold`, `double underSensitiveThreshold`, `Duration crossRefWindow`. Add validation in compact constructor:
- `overSensitiveThreshold` in (0.0, 1.0]
- `underSensitiveThreshold` in (0.0, 1.0]
- `crossRefWindow` positive

Add 6-arg backwards-compatible constructor that defaults new fields to `0.5, 0.5, Duration.ofHours(1)`.

- [ ] **Step 8: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl api -Dtest="FeedbackConfigDriftTest,FeedbackConfigTest"`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/ras/api/DriftDirection.java api/src/main/java/io/casehub/ras/api/FeedbackConfig.java api/src/test/java/io/casehub/ras/api/DriftDirectionTest.java api/src/test/java/io/casehub/ras/api/FeedbackConfigDriftTest.java
```

Message: `feat(#62): DriftDirection enum + FeedbackConfig drift thresholds + crossRefWindow`

### Task 2: FeedbackTuningStrategy.classifyDrift() default method

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/FeedbackTuningStrategy.java`
- Test: `api/src/test/java/io/casehub/ras/api/FeedbackTuningStrategyDriftTest.java`

**Interfaces:**
- Consumes: `DriftDirection` enum (Task 1), `OutcomeStatistics.recall()`, `FeedbackConfig.overSensitiveThreshold()`, `FeedbackConfig.underSensitiveThreshold()`
- Produces: `FeedbackTuningStrategy.classifyDrift(OutcomeStatistics, FeedbackConfig) → DriftDirection` default method

- [ ] **Step 1: Write classifyDrift tests**

```java
// api/src/test/java/io/casehub/ras/api/FeedbackTuningStrategyDriftTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class FeedbackTuningStrategyDriftTest {

    private final FeedbackTuningStrategy strategy = new FeedbackTuningStrategy() {
        @Override
        public OptionalDouble adjustThreshold(OutcomeStatistics s, double t, FeedbackConfig c) {
            return OptionalDouble.empty();
        }
        @Override
        public Optional<double[]> adjustPriors(double[] p, long[] o, FeedbackConfig c) {
            return Optional.empty();
        }
    };

    private FeedbackConfig config(double overThreshold, double underThreshold) {
        return new FeedbackConfig(Set.of("n"), Set.of("c"),
                Duration.ofHours(1), 0.1, Duration.ofDays(90), false,
                overThreshold, underThreshold, Duration.ofHours(1));
    }

    private OutcomeStatistics stats(long total, long noise, long confirmed, long neutral, long missed) {
        return new OutcomeStatistics("sit", "t1", total, noise, confirmed, neutral, Instant.EPOCH, missed);
    }

    @Test
    void insufficientData_belowMinOutcomes() {
        assertEquals(DriftDirection.INSUFFICIENT_DATA,
                strategy.classifyDrift(stats(9, 5, 2, 2, 0), config(0.5, 0.5)));
    }

    @Test
    void stable_bothWithinBounds() {
        assertEquals(DriftDirection.STABLE,
                strategy.classifyDrift(stats(20, 5, 10, 5, 2), config(0.5, 0.5)));
    }

    @Test
    void overSensitive_highNoiseRate() {
        // noise=12/20=0.6 > 0.5, recall=8/(8+1)=0.89 > 0.5
        assertEquals(DriftDirection.OVER_SENSITIVE,
                strategy.classifyDrift(stats(20, 12, 8, 0, 1), config(0.5, 0.5)));
    }

    @Test
    void underSensitive_lowRecall() {
        // noise=2/20=0.1 < 0.5, recall=3/(3+5)=0.375 < 0.5
        assertEquals(DriftDirection.UNDER_SENSITIVE,
                strategy.classifyDrift(stats(20, 2, 3, 15, 5), config(0.5, 0.5)));
    }

    @Test
    void bothDrifting_highNoiseAndLowRecall() {
        // noise=12/20=0.6 > 0.5, recall=2/(2+5)=0.286 < 0.5
        assertEquals(DriftDirection.BOTH_DRIFTING,
                strategy.classifyDrift(stats(20, 12, 2, 6, 5), config(0.5, 0.5)));
    }

    @Test
    void recallNaN_noConfirmedNoMissed_notUnderSensitive() {
        // noise=12/20=0.6 > 0.5, recall=NaN (0 confirmed, 0 missed)
        assertEquals(DriftDirection.OVER_SENSITIVE,
                strategy.classifyDrift(stats(20, 12, 0, 8, 0), config(0.5, 0.5)));
    }

    @Test
    void minRecallSamples_belowThreshold() {
        // recall=1/(1+1)=0.5 but confirmed+missed=2 < MIN_RECALL_SAMPLES(3)
        assertEquals(DriftDirection.STABLE,
                strategy.classifyDrift(stats(20, 5, 1, 14, 1), config(0.5, 0.5)));
    }

    @Test
    void configurableThresholds() {
        // noise=4/20=0.2, overThreshold=0.15 → over-sensitive
        assertEquals(DriftDirection.OVER_SENSITIVE,
                strategy.classifyDrift(stats(20, 4, 10, 6, 3), config(0.15, 0.5)));
        // recall=10/(10+3)=0.77, underThreshold=0.8 → under-sensitive
        assertEquals(DriftDirection.UNDER_SENSITIVE,
                strategy.classifyDrift(stats(20, 1, 10, 9, 3), config(0.5, 0.8)));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=FeedbackTuningStrategyDriftTest`
Expected: FAIL — classifyDrift method does not exist

- [ ] **Step 3: Implement classifyDrift default method**

Add to `FeedbackTuningStrategy.java`:

```java
default DriftDirection classifyDrift(OutcomeStatistics statistics, FeedbackConfig config) {
    int MIN_DRIFT_OUTCOMES = 10;
    int MIN_RECALL_SAMPLES = 3;

    if (statistics.totalOutcomes() < MIN_DRIFT_OUTCOMES) {
        return DriftDirection.INSUFFICIENT_DATA;
    }

    boolean overSensitive = !Double.isNaN(statistics.noiseRate())
            && statistics.noiseRate() > config.overSensitiveThreshold();

    boolean underSensitive = false;
    double recall = statistics.recall();
    if (!Double.isNaN(recall)
            && (statistics.confirmedCount() + statistics.missedCount()) >= MIN_RECALL_SAMPLES
            && recall < config.underSensitiveThreshold()) {
        underSensitive = true;
    }

    if (overSensitive && underSensitive) return DriftDirection.BOTH_DRIFTING;
    if (overSensitive) return DriftDirection.OVER_SENSITIVE;
    if (underSensitive) return DriftDirection.UNDER_SENSITIVE;
    return DriftDirection.STABLE;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=FeedbackTuningStrategyDriftTest`
Expected: PASS

- [ ] **Step 5: Commit**

Message: `feat(#62): FeedbackTuningStrategy.classifyDrift() default method`

### Task 3: MissedDetectionRecord.ganglionIds + GanglionOutcomeStatistics.missedCount + recall()

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/MissedDetectionRecord.java`
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionOutcomeStatistics.java`
- Test: `api/src/test/java/io/casehub/ras/api/MissedDetectionRecordGanglionTest.java`
- Test: `api/src/test/java/io/casehub/ras/api/GanglionOutcomeStatisticsRecallTest.java`

**Interfaces:**
- Produces: `MissedDetectionRecord` with `List<String> ganglionIds()` (nullable) + 7-arg backwards-compat constructor
- Produces: `GanglionOutcomeStatistics` with `long missedCount()` + `double recall()` + 5-arg backwards-compat constructor

- [ ] **Step 1: Write MissedDetectionRecord ganglionIds test**

```java
// api/src/test/java/io/casehub/ras/api/MissedDetectionRecordGanglionTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class MissedDetectionRecordGanglionTest {
    @Test
    void eightArgConstructorStoresGanglionIds() {
        var record = new MissedDetectionRecord("sit", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now(),
                List.of("ganglion-a", "ganglion-b"));
        assertEquals(List.of("ganglion-a", "ganglion-b"), record.ganglionIds());
    }

    @Test
    void sevenArgConstructorDefaultsNull() {
        var record = new MissedDetectionRecord("sit", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now());
        assertNull(record.ganglionIds());
    }

    @Test
    void nullGanglionIdsAllowed() {
        var record = new MissedDetectionRecord("sit", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now(), null);
        assertNull(record.ganglionIds());
    }

    @Test
    void emptyGanglionIdsAllowed() {
        var record = new MissedDetectionRecord("sit", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now(), List.of());
        assertTrue(record.ganglionIds().isEmpty());
    }
}
```

- [ ] **Step 2: Write GanglionOutcomeStatistics recall test**

```java
// api/src/test/java/io/casehub/ras/api/GanglionOutcomeStatisticsRecallTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class GanglionOutcomeStatisticsRecallTest {
    @Test
    void recallWithMissedData() {
        var stats = new GanglionOutcomeStatistics("g1", 10, 2, 5, 3, 3);
        assertEquals(5.0 / 8.0, stats.recall(), 0.0001);
    }

    @Test
    void recallNaN_whenMissedCountZero() {
        var stats = new GanglionOutcomeStatistics("g1", 10, 2, 5, 3, 0);
        assertTrue(Double.isNaN(stats.recall()));
    }

    @Test
    void recallNaN_whenBothZero() {
        var stats = new GanglionOutcomeStatistics("g1", 0, 0, 0, 0, 0);
        assertTrue(Double.isNaN(stats.recall()));
    }

    @Test
    void recallZero_allMissedNoConfirmed() {
        var stats = new GanglionOutcomeStatistics("g1", 0, 0, 0, 0, 5);
        assertEquals(0.0, stats.recall(), 0.0001);
    }

    @Test
    void fiveArgConstructorDefaultsMissedToZero() {
        var stats = new GanglionOutcomeStatistics("g1", 10, 2, 5, 3);
        assertEquals(0, stats.missedCount());
        assertTrue(Double.isNaN(stats.recall()));
    }

    @Test
    void precisionAndNoiseRateUnchanged() {
        var stats = new GanglionOutcomeStatistics("g1", 10, 3, 5, 2, 4);
        assertEquals(5.0 / 8.0, stats.precision(), 0.0001);
        assertEquals(3.0 / 10.0, stats.noiseRate(), 0.0001);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest="MissedDetectionRecordGanglionTest,GanglionOutcomeStatisticsRecallTest"`
Expected: FAIL

- [ ] **Step 4: Extend MissedDetectionRecord**

Add `List<String> ganglionIds` as 8th field. Add 7-arg constructor that defaults `ganglionIds` to `null`. In compact constructor: do NOT requireNonNull on ganglionIds (it's nullable). Defensive copy when non-null: `ganglionIds = ganglionIds != null ? List.copyOf(ganglionIds) : null`.

- [ ] **Step 5: Extend GanglionOutcomeStatistics**

Add `long missedCount` as 6th field. Add 5-arg backwards-compat constructor defaulting `missedCount` to 0. Add `recall()` method:

```java
public double recall() {
    if (missedCount == 0) return Double.NaN;
    long decisive = confirmedCount + missedCount;
    return confirmedCount / (double) decisive;
}
```

- [ ] **Step 6: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl api -Dtest="MissedDetectionRecordGanglionTest,GanglionOutcomeStatisticsRecallTest,MissedDetectionRecordTest,OutcomeStatisticsTest"`
Expected: PASS

- [ ] **Step 7: Commit**

Message: `feat(#63): MissedDetectionRecord.ganglionIds + GanglionOutcomeStatistics.missedCount + recall()`

---

## Batch 2: Persistence + Recording

### Task 4: Flyway V10 + ledger ganglion missed counts

**Files:**
- Create: `persistence-jpa/src/main/resources/db/ras/migration/V10__missed_detection_ganglion_ids.sql`
- Modify: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedger.java`
- Modify: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/MissedDetectionEntity.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/InMemoryOutcomeLedger.java`
- Modify: `api/src/test/java/io/casehub/ras/api/AbstractOutcomeLedgerContractTest.java` (if exists, else create test)

**Interfaces:**
- Consumes: `MissedDetectionRecord.ganglionIds()` (Task 3), `GanglionOutcomeStatistics` 6-arg constructor (Task 3)
- Produces: Extended `ganglionStatistics()` returning `GanglionOutcomeStatistics` with `missedCount` from both outcome records AND missed detection records

- [ ] **Step 1: Write contract test for ganglionStatistics with missed counts**

Add test methods to `AbstractOutcomeLedgerContractTest` (or create a new test class if needed):

```java
@Test
void ganglionStatistics_includesMissedCountsFromGanglionLevelReports() {
    // Record an outcome with ganglion contribution
    ledger().record(new OutcomeRecord("sit1", "ck", "t1", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID(),
            List.of(new GanglionContribution("g1", 0.8, DetectionSignal.DETECTED))));
    // Record a ganglion-level missed detection
    ledger().recordMissed(new MissedDetectionRecord("sit1", "ck2", "t1",
            Instant.now(), "op", UUID.randomUUID(), Instant.now(),
            List.of("g1")));

    var stats = ledger().ganglionStatistics("sit1", "t1", Instant.EPOCH);
    var g1 = stats.get("g1");
    assertNotNull(g1);
    assertEquals(1, g1.confirmedCount());
    assertEquals(1, g1.missedCount());
}

@Test
void ganglionStatistics_situationLevelMissDoesNotAffectGanglionCounts() {
    ledger().recordMissed(new MissedDetectionRecord("sit1", "ck", "t1",
            Instant.now(), "op", UUID.randomUUID(), Instant.now()));
    var stats = ledger().ganglionStatistics("sit1", "t1", Instant.EPOCH);
    assertTrue(stats.isEmpty());
}

@Test
void ganglionStatistics_ganglionOnlyInMissedDetection() {
    // A ganglion that appears ONLY in missed detections (never in outcomes)
    ledger().recordMissed(new MissedDetectionRecord("sit1", "ck", "t1",
            Instant.now(), "op", UUID.randomUUID(), Instant.now(),
            List.of("g-never-fired")));
    var stats = ledger().ganglionStatistics("sit1", "t1", Instant.EPOCH);
    var gNever = stats.get("g-never-fired");
    assertNotNull(gNever);
    assertEquals(0, gNever.confirmedCount());
    assertEquals(1, gNever.missedCount());
    assertEquals(0.0, gNever.recall(), 0.0001);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api`
Expected: FAIL — new test methods fail

- [ ] **Step 3: Create Flyway V10 migration**

```sql
-- V10__missed_detection_ganglion_ids.sql
ALTER TABLE ras_missed_detection
  ADD COLUMN ganglion_ids JSONB;
```

- [ ] **Step 4: Update MissedDetectionEntity**

Add `ganglionIds` column mapping (JSONB, nullable) to `MissedDetectionEntity`.

- [ ] **Step 5: Update JpaOutcomeLedger.recordMissed()**

Add `ganglion_ids` to the INSERT statement:

```sql
INSERT INTO ras_missed_detection (situation_id, correlation_key, tenancy_id,
event_time, reported_by, report_id, recorded_at, ganglion_ids)
VALUES (:sid, :ck, :tid, :et, :rb, :rid, :ra, CAST(:gids AS jsonb))
ON CONFLICT DO NOTHING
```

Serialize `record.ganglionIds()` via `ObjectMapper` — NULL when ganglionIds is null or empty.

- [ ] **Step 6: Update JpaOutcomeLedger.ganglionStatistics()**

After the existing outcome query populates `counts` map (changing array size from `long[3]` to `long[4]`), run a second query:

```sql
SELECT elem AS ganglion_id, COUNT(*)
FROM ras_missed_detection,
     jsonb_array_elements_text(ganglion_ids) elem
WHERE situation_id = :sid AND tenancy_id = :tid
  AND event_time >= :since
GROUP BY elem
```

Merge results: `long[] c = counts.computeIfAbsent(ganglionId, k -> new long[4]); c[3] += count;`

Update the result construction to use the 6-arg constructor: `new GanglionOutcomeStatistics(id, c[0]+c[1]+c[2], c[0], c[1], c[2], c[3])`.

- [ ] **Step 7: Update InMemoryOutcomeLedger.ganglionStatistics()**

Change `long[3]` to `long[4]`. After the outcome loop, add missed detection loop:

```java
for (MissedDetectionRecord r : missedStore.values()) {
    if (!r.situationId().equals(situationId) || !r.tenancyId().equals(tenancyId)
            || r.eventTime().isBefore(since) || r.ganglionIds() == null) continue;
    for (String gid : r.ganglionIds()) {
        long[] c = counts.computeIfAbsent(gid, k -> new long[4]);
        c[3]++;
    }
}
```

Update result construction to use 6-arg constructor with `c[3]` as missedCount.

- [ ] **Step 8: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl api,runtime,persistence-jpa`
Expected: PASS

- [ ] **Step 9: Commit**

Message: `feat(#63): Flyway V10 ganglion_ids JSONB + ledger ganglion missed counts`

### Task 5: MissedDetectionRecorder ganglion validation + DefaultTuningStrategy threshold coherence

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionRecorder.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/DefaultTuningStrategy.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionRecorderGanglionTest.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/DefaultTuningStrategyTest.java` (or new test class)

**Interfaces:**
- Consumes: `MissedDetectionRecord.ganglionIds()` (Task 3), `SituationDefinitionRegistry` for ganglion validation, `FeedbackConfig.overSensitiveThreshold()` (Task 1)
- Produces: UNKNOWN_GANGLION rejection in `RecordResult`, threshold-coherent `adjustThreshold()`

- [ ] **Step 1: Write ganglion validation test**

```java
// runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionRecorderGanglionTest.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class MissedDetectionRecorderGanglionTest {

    @Test
    void rejectsUnknownGanglionId() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());
        when(registry.referencedGanglia("sit1")).thenReturn(Set.of("g1", "g2"));

        var ledger = mock(OutcomeLedger.class);
        var recorder = new MissedDetectionRecorder(ledger, registry);

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now(),
                List.of("g1", "unknown-ganglion"));

        var result = recorder.record(record);
        assertFalse(result.accepted());
        assertEquals("UNKNOWN_GANGLION", result.rejectionReason());
    }

    @Test
    void acceptsValidGanglionIds() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());
        when(registry.referencedGanglia("sit1")).thenReturn(Set.of("g1", "g2"));

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);
        var recorder = new MissedDetectionRecorder(ledger, registry);

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now(),
                List.of("g1", "g2"));

        var result = recorder.record(record);
        assertTrue(result.accepted());
    }

    @Test
    void nullGanglionIdsSkipsValidation() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);
        var recorder = new MissedDetectionRecorder(ledger, registry);

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now());

        var result = recorder.record(record);
        assertTrue(result.accepted());
        verify(registry, never()).referencedGanglia(any());
    }

    private FeedbackConfig feedbackConfig() {
        return new FeedbackConfig(Set.of("n"), Set.of("c"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), false);
    }
}
```

- [ ] **Step 2: Write DefaultTuningStrategy threshold coherence test**

Add test to verify `adjustThreshold` uses `config.overSensitiveThreshold()` instead of hardcoded 0.5:

```java
@Test
void adjustThreshold_usesConfiguredOverSensitiveThreshold() {
    var config = new FeedbackConfig(Set.of("n"), Set.of("c"),
            Duration.ofHours(1), 0.1, Duration.ofDays(90), true,
            0.3, 0.5, Duration.ofHours(1));
    // noiseRate = 8/20 = 0.4 — above 0.3 threshold
    var stats = new OutcomeStatistics("s", "t", 20, 8, 10, 2, Instant.EPOCH, 0);
    var result = new DefaultTuningStrategy().adjustThreshold(stats, 0.7, config);
    assertTrue(result.isPresent());
}

@Test
void adjustThreshold_noActionBelowConfiguredThreshold() {
    var config = new FeedbackConfig(Set.of("n"), Set.of("c"),
            Duration.ofHours(1), 0.1, Duration.ofDays(90), true,
            0.6, 0.5, Duration.ofHours(1));
    // noiseRate = 8/20 = 0.4 — below 0.6 threshold
    var stats = new OutcomeStatistics("s", "t", 20, 8, 10, 2, Instant.EPOCH, 0);
    var result = new DefaultTuningStrategy().adjustThreshold(stats, 0.7, config);
    assertTrue(result.isEmpty());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest="MissedDetectionRecorderGanglionTest,DefaultTuningStrategyTest"`
Expected: FAIL

- [ ] **Step 4: Add referencedGanglia() to SituationDefinitionRegistry**

Add `Set<String> referencedGanglia(String situationId)` method that returns the ganglia referenced by the situation's chain mode. If situation not found, return empty set.

- [ ] **Step 5: Add ganglion validation to MissedDetectionRecorder.record()**

After feedback config validation, before temporal bounds check:

```java
if (record.ganglionIds() != null && !record.ganglionIds().isEmpty()) {
    Set<String> validGanglia = registry.referencedGanglia(record.situationId());
    for (String gid : record.ganglionIds()) {
        if (!validGanglia.contains(gid)) {
            if (metrics != null) metrics.missedRejected(record.situationId(), "UNKNOWN_GANGLION");
            return RecordResult.rejected("UNKNOWN_GANGLION");
        }
    }
}
```

- [ ] **Step 6: Update DefaultTuningStrategy.adjustThreshold()**

Replace hardcoded `0.5` with `config.overSensitiveThreshold()`:

```java
double threshold = config.overSensitiveThreshold();
if (Double.isNaN(noiseRate) || noiseRate <= threshold) return OptionalDouble.empty();
double adjustment = config.learningRate() * (noiseRate - threshold);
```

- [ ] **Step 7: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS

- [ ] **Step 8: Commit**

Message: `feat(#63): MissedDetectionRecorder ganglion validation + DefaultTuningStrategy threshold coherence`

---

## Batch 3: Drift Runtime + YAML

### Task 6: FeedbackMetrics drift gauge + ganglion recall gauge

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackMetrics.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackMetricsDriftTest.java`

**Interfaces:**
- Consumes: `DriftDirection` enum (Task 1), `GanglionOutcomeStatistics.recall()` (Task 3)
- Produces: `FeedbackMetrics.setDriftGauges(DriftDirection, String situationId, String tenancyId)`, `recordGanglionStatistics()` publishes `ras.feedback.ganglion.recall`

- [ ] **Step 1: Write drift gauge and ganglion recall tests**

```java
// runtime/src/test/java/io/casehub/ras/runtime/FeedbackMetricsDriftTest.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.DriftDirection;
import io.casehub.ras.api.GanglionOutcomeStatistics;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FeedbackMetricsDriftTest {
    @Test
    void setDriftGauges_registersAllFiveDirections() {
        var registry = new SimpleMeterRegistry();
        var metrics = new FeedbackMetrics(registry);
        metrics.setDriftGauges(DriftDirection.OVER_SENSITIVE, "sit1", "t1");

        Gauge active = registry.find("ras.feedback.drift")
                .tag("direction", "OVER_SENSITIVE")
                .tag("situation_id", "sit1").gauge();
        assertNotNull(active);
        assertEquals(1.0, active.value());

        Gauge inactive = registry.find("ras.feedback.drift")
                .tag("direction", "STABLE")
                .tag("situation_id", "sit1").gauge();
        assertNotNull(inactive);
        assertEquals(0.0, inactive.value());
    }

    @Test
    void setDriftGauges_directionChange_updatesCorrectly() {
        var registry = new SimpleMeterRegistry();
        var metrics = new FeedbackMetrics(registry);
        metrics.setDriftGauges(DriftDirection.OVER_SENSITIVE, "sit1", "t1");
        metrics.setDriftGauges(DriftDirection.STABLE, "sit1", "t1");

        Gauge overSensitive = registry.find("ras.feedback.drift")
                .tag("direction", "OVER_SENSITIVE")
                .tag("situation_id", "sit1").gauge();
        assertEquals(0.0, overSensitive.value());

        Gauge stable = registry.find("ras.feedback.drift")
                .tag("direction", "STABLE")
                .tag("situation_id", "sit1").gauge();
        assertEquals(1.0, stable.value());
    }

    @Test
    void recordGanglionStatistics_publishesRecallGauge() {
        var registry = new SimpleMeterRegistry();
        var metrics = new FeedbackMetrics(registry);
        var stats = new GanglionOutcomeStatistics("g1", 10, 2, 5, 3, 3);

        metrics.recordGanglionStatistics("g1", "sit1", "t1", stats);

        Gauge recall = registry.find("ras.feedback.ganglion.recall")
                .tag("ganglion_id", "g1").gauge();
        assertNotNull(recall);
        assertEquals(5.0 / 8.0, recall.value(), 0.0001);
    }

    @Test
    void recordGanglionStatistics_suppressesRecallGaugeWhenNaN() {
        var registry = new SimpleMeterRegistry();
        var metrics = new FeedbackMetrics(registry);
        var stats = new GanglionOutcomeStatistics("g1", 10, 2, 5, 3, 0);

        metrics.recordGanglionStatistics("g1", "sit1", "t1", stats);

        Gauge recall = registry.find("ras.feedback.ganglion.recall")
                .tag("ganglion_id", "g1").gauge();
        assertNull(recall);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=FeedbackMetricsDriftTest`
Expected: FAIL

- [ ] **Step 3: Implement setDriftGauges and ganglion recall gauge**

Add to `FeedbackMetrics`:

```java
private final ConcurrentHashMap<String, Map<DriftDirection, AtomicReference<Double>>> driftHolders =
        new ConcurrentHashMap<>();

public void setDriftGauges(DriftDirection direction, String situationId, String tenancyId) {
    if (meterRegistry == null) return;
    String holderKey = situationId + "|" + tenancyId;
    Map<DriftDirection, AtomicReference<Double>> holders = driftHolders.computeIfAbsent(holderKey, k -> {
        Map<DriftDirection, AtomicReference<Double>> m = new java.util.EnumMap<>(DriftDirection.class);
        for (DriftDirection d : DriftDirection.values()) {
            AtomicReference<Double> ref = new AtomicReference<>(0.0);
            Tags tags = Tags.of("direction", d.name(), "situation_id", situationId, "tenancy_id", tenancyId);
            meterRegistry.gauge("ras.feedback.drift", tags, ref, AtomicReference::get);
            m.put(d, ref);
        }
        return m;
    });
    for (var entry : holders.entrySet()) {
        entry.getValue().set(entry.getKey() == direction ? 1.0 : 0.0);
    }
}
```

Add `ras.feedback.ganglion.recall` to `recordGanglionStatistics()`:

```java
setGauge("ras.feedback.ganglion.recall", tags, stats.recall());
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=FeedbackMetricsDriftTest`
Expected: PASS

- [ ] **Step 5: Commit**

Message: `feat(#62): FeedbackMetrics drift state-gauge + per-ganglion recall gauge`

### Task 7: FeedbackUpdateJob drift publication + BOTH_DRIFTING guard + YAML parsing

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackUpdateJob.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackUpdateJobDriftTest.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java` (extend)

**Interfaces:**
- Consumes: `FeedbackTuningStrategy.classifyDrift()` (Task 2), `FeedbackMetrics.setDriftGauges()` (Task 6), `FeedbackConfig` new fields (Task 1)
- Produces: Drift gauge publication in observational block, BOTH_DRIFTING guard in tuning block

- [ ] **Step 1: Write FeedbackUpdateJob drift tests**

```java
// runtime/src/test/java/io/casehub/ras/runtime/FeedbackUpdateJobDriftTest.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import java.time.Duration;
import java.time.Instant;
import java.util.*;

class FeedbackUpdateJobDriftTest {

    @Test
    void processTenant_publishesDriftGauge() {
        var registry = mock(SituationDefinitionRegistry.class);
        var ledger = mock(OutcomeLedger.class);
        var analyzer = mock(FeedbackAnalyzer.class);
        var tuningStrategy = mock(FeedbackTuningStrategy.class);
        var feedbackState = new FeedbackState();
        var feedbackMetrics = mock(FeedbackMetrics.class);
        var config = feedbackConfig();

        var stats = new OutcomeStatistics("sit1", "t1", 20, 5, 10, 5, Instant.EPOCH, 2);
        when(analyzer.analyze("sit1", "t1", config)).thenReturn(stats);
        when(analyzer.ganglionAnalyze("sit1", "t1", config)).thenReturn(Map.of());
        when(tuningStrategy.classifyDrift(stats, config)).thenReturn(DriftDirection.STABLE);
        when(registry.definition("sit1")).thenReturn(
                new SituationDefinition("sit1", Set.of("e"), null, null,
                        new ChainMode.Threshold(List.of("g1"), 0.5),
                        new TriggerAction.CreateCase("ns", "cn", "1.0", Map.of()),
                        new TriggerMode.FireOnce(), null, null, Map.of(), config));

        var job = new FeedbackUpdateJob(registry, ledger, analyzer, tuningStrategy, feedbackState, feedbackMetrics);
        // Call processTenant via reflection or make it package-visible
        // Verify drift gauges are published
        verify(feedbackMetrics).setDriftGauges(DriftDirection.STABLE, "sit1", "t1");
    }

    @Test
    void processTenant_bothDrifting_skipsThresholdAdjustment() {
        // Setup where classifyDrift returns BOTH_DRIFTING
        // Verify tuningStrategy.adjustThreshold() is NOT called
        // Verify tuningStrategy.adjustPriors() is NOT called
    }

    private FeedbackConfig feedbackConfig() {
        return new FeedbackConfig(Set.of("n"), Set.of("c"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), true);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=FeedbackUpdateJobDriftTest`
Expected: FAIL — setDriftGauges not called yet

- [ ] **Step 3: Update FeedbackUpdateJob.processTenant()**

Add drift classification in the observational block (before `tuningEnabled` guard):

```java
DriftDirection drift = tuningStrategy.classifyDrift(stats, config);
feedbackMetrics.setDriftGauges(drift, situationId, tenancyId);
```

Add BOTH_DRIFTING guard after `tuningEnabled` check:

```java
if (!config.tuningEnabled()) return;
if (drift == DriftDirection.BOTH_DRIFTING) return;
```

The `drift` variable must be declared before the `tuningEnabled` guard so it's accessible in both blocks.

- [ ] **Step 4: Update YamlSituationDefinitionProvider.parseFeedbackConfig()**

Add parsing for the 3 new FeedbackConfig fields with defaults:

```java
double overSensitiveThreshold = map.containsKey("overSensitiveThreshold")
    ? ((Number) map.get("overSensitiveThreshold")).doubleValue() : 0.5;
double underSensitiveThreshold = map.containsKey("underSensitiveThreshold")
    ? ((Number) map.get("underSensitiveThreshold")).doubleValue() : 0.5;
Duration crossRefWindow = map.containsKey("crossRefWindow")
    ? Duration.parse(map.get("crossRefWindow").toString()) : Duration.ofHours(1);
return new FeedbackConfig(
        new LinkedHashSet<>(noiseLabels),
        new LinkedHashSet<>(confirmedLabels),
        suppressionCooldown, learningRate, retentionPeriod, tuningEnabled,
        overSensitiveThreshold, underSensitiveThreshold, crossRefWindow);
```

- [ ] **Step 5: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS

- [ ] **Step 6: Commit**

Message: `feat(#62): FeedbackUpdateJob drift publication + BOTH_DRIFTING guard + YAML parsing`

---

## Batch 4: Cross-Reference + REST

### Task 8: MissedDetectionRecorder trigger cross-reference

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionRecorder.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionRecorderCrossRefTest.java`

**Interfaces:**
- Consumes: `SituationQueryService.history()`, `FeedbackConfig.crossRefWindow()` (Task 1), `SituationEvent.changeType()`
- Produces: `RecordResult` with `possiblyDetected`, `lastTriggerTime`, `crossRefConclusive`

- [ ] **Step 1: Write cross-reference tests**

```java
// runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionRecorderCrossRefTest.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class MissedDetectionRecorderCrossRefTest {

    @Test
    void possiblyDetected_whenTriggerFoundInHistory() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);

        var queryService = mock(SituationQueryService.class);
        Instant eventTime = Instant.now();
        when(queryService.history(eq("t1"), eq("sit1"), eq("ck"),
                any(Instant.class), any(Instant.class)))
                .thenReturn(List.of(situationEvent("sit1", "ck", "t1",
                        SituationChangeEvent.ChangeType.TRIGGERED, eventTime.minusSeconds(60))));

        @SuppressWarnings("unchecked")
        Instance<SituationQueryService> qsInstance = mock(Instance.class);
        when(qsInstance.isResolvable()).thenReturn(true);
        when(qsInstance.get()).thenReturn(queryService);

        var recorder = new MissedDetectionRecorder(ledger, registry, null, qsInstance,
                Duration.ofDays(30));

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                eventTime, "op", UUID.randomUUID(), Instant.now());
        var result = recorder.record(record);

        assertTrue(result.accepted());
        assertTrue(result.possiblyDetected());
        assertTrue(result.crossRefConclusive());
        assertNotNull(result.lastTriggerTime());
    }

    @Test
    void notDetected_noTriggerInHistory() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);

        var queryService = mock(SituationQueryService.class);
        when(queryService.history(any(), any(), any(), any(Instant.class), any(Instant.class)))
                .thenReturn(List.of());

        @SuppressWarnings("unchecked")
        Instance<SituationQueryService> qsInstance = mock(Instance.class);
        when(qsInstance.isResolvable()).thenReturn(true);
        when(qsInstance.get()).thenReturn(queryService);

        var recorder = new MissedDetectionRecorder(ledger, registry, null, qsInstance,
                Duration.ofDays(30));

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now());
        var result = recorder.record(record);

        assertTrue(result.accepted());
        assertFalse(result.possiblyDetected());
        assertTrue(result.crossRefConclusive());
    }

    @Test
    void crossRefNotConclusive_eventOutsideHistoryRetention() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);

        @SuppressWarnings("unchecked")
        Instance<SituationQueryService> qsInstance = mock(Instance.class);
        when(qsInstance.isResolvable()).thenReturn(true);

        var recorder = new MissedDetectionRecorder(ledger, registry, null, qsInstance,
                Duration.ofDays(30));

        // Event 60 days ago — outside 30-day retention
        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now().minus(Duration.ofDays(60)), "op", UUID.randomUUID(), Instant.now());
        // Note: event is within FeedbackConfig retention (90 days) but outside event history retention (30 days)
        var result = recorder.record(record);

        assertTrue(result.accepted());
        assertFalse(result.possiblyDetected());
        assertFalse(result.crossRefConclusive());
    }

    @Test
    void graceful_whenSituationQueryServiceAbsent() {
        var registry = mock(SituationDefinitionRegistry.class);
        when(registry.exists("sit1")).thenReturn(true);
        when(registry.feedbackConfig("sit1")).thenReturn(feedbackConfig());

        var ledger = mock(OutcomeLedger.class);
        when(ledger.recordMissed(any())).thenReturn(true);

        @SuppressWarnings("unchecked")
        Instance<SituationQueryService> qsInstance = mock(Instance.class);
        when(qsInstance.isResolvable()).thenReturn(false);

        var recorder = new MissedDetectionRecorder(ledger, registry, null, qsInstance,
                Duration.ofDays(30));

        var record = new MissedDetectionRecord("sit1", "ck", "t1",
                Instant.now(), "op", UUID.randomUUID(), Instant.now());
        var result = recorder.record(record);

        assertTrue(result.accepted());
        assertFalse(result.possiblyDetected());
        assertFalse(result.crossRefConclusive());
    }

    private FeedbackConfig feedbackConfig() {
        return new FeedbackConfig(Set.of("n"), Set.of("c"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), false);
    }

    private SituationEvent situationEvent(String sitId, String ck, String tid,
            SituationChangeEvent.ChangeType changeType, Instant eventTime) {
        return new SituationEvent(sitId, ck, tid, changeType, eventTime,
                eventTime, 0.8, 1, 0, Map.of(), Map.of());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionRecorderCrossRefTest`
Expected: FAIL

- [ ] **Step 3: Extend RecordResult**

Add 3 new fields: `boolean possiblyDetected`, `Instant lastTriggerTime` (nullable), `boolean crossRefConclusive`. Update factory methods with defaults.

- [ ] **Step 4: Add Instance<SituationQueryService> + eventHistoryRetention to MissedDetectionRecorder constructor**

Add `Instance<SituationQueryService> queryServiceInstance` and `@ConfigProperty(name = "ras.event-history.retention", defaultValue = "P30D") Duration eventHistoryRetention` as constructor parameters. Add a package-private constructor for tests that accepts these directly.

- [ ] **Step 5: Implement cross-reference logic**

After `boolean isNew = ledger.recordMissed(record)`, add:

```java
boolean possiblyDetected = false;
Instant lastTriggerTime = null;
boolean crossRefConclusive = false;

if (queryServiceInstance != null && queryServiceInstance.isResolvable()) {
    Instant historyWindowStart = Instant.now().minus(eventHistoryRetention);
    if (!record.eventTime().isBefore(historyWindowStart)) {
        crossRefConclusive = true;
        Duration crossRefWindow = config.crossRefWindow();
        var history = queryServiceInstance.get().history(
                record.tenancyId(), record.situationId(), record.correlationKey(),
                record.eventTime().minus(crossRefWindow),
                record.eventTime().plus(crossRefWindow));
        for (var event : history) {
            if (event.changeType() == SituationChangeEvent.ChangeType.TRIGGERED) {
                possiblyDetected = true;
                lastTriggerTime = event.eventTime();
                break;
            }
        }
    }
}
return RecordResult.accepted(isNew, possiblyDetected, lastTriggerTime, crossRefConclusive);
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionRecorderCrossRefTest`
Expected: PASS

- [ ] **Step 7: Commit**

Message: `feat(#64): MissedDetectionRecorder trigger cross-reference + RecordResult extension`

### Task 9: MissedDetectionResource full extension

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionResource.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/MissedDetectionResourceTest.java`

**Interfaces:**
- Consumes: `MissedDetectionRecord` 8-arg constructor (Task 3), `RecordResult.possiblyDetected()`, `RecordResult.lastTriggerTime()`, `RecordResult.crossRefConclusive()` (Task 8)
- Produces: REST response with `ganglionIds` in request, `possiblyDetected`/`crossRefConclusive`/`advisory` in response

- [ ] **Step 1: Write resource extension tests**

Extend `MissedDetectionResourceTest` with:

```java
@Test
void reportMissed_withGanglionIds_returnsCreated() {
    // POST with ganglionIds in request body
    // Verify 201 response
}

@Test
void reportMissed_unknownGanglion_returnsBadRequest() {
    // POST with unknown ganglionId
    // Verify 400 with error=UNKNOWN_GANGLION
}

@Test
void reportMissed_possiblyDetected_includesAdvisory() {
    // Setup recorder to return possiblyDetected=true
    // Verify response includes possiblyDetected, lastTriggerTime, advisory
}

@Test
void reportMissed_ganglionLevel_differentAdvisoryText() {
    // POST with ganglionIds + possiblyDetected=true
    // Verify advisory mentions "other ganglia may have detected"
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionResourceTest`
Expected: FAIL

- [ ] **Step 3: Update MissedDetectionRequest**

Add `List<String> ganglionIds` field to `MissedDetectionRequest` record.

- [ ] **Step 4: Update MissedDetectionResource.reportMissed()**

Pass `request.ganglionIds()` to MissedDetectionRecord 8-arg constructor. Update response building:

```java
if (result.isNew()) {
    Map<String, Object> response = new LinkedHashMap<>();
    response.put("reportId", record.reportId().toString());
    response.put("status", "RECORDED");
    response.put("recordedAt", recordedAt.toString());
    response.put("possiblyDetected", result.possiblyDetected());
    response.put("crossRefConclusive", result.crossRefConclusive());
    if (result.lastTriggerTime() != null) {
        response.put("lastTriggerTime", result.lastTriggerTime().toString());
    }
    if (result.possiblyDetected()) {
        boolean ganglionLevel = request.ganglionIds() != null && !request.ganglionIds().isEmpty();
        response.put("advisory", ganglionLevel
                ? "A trigger was found for this situation within the cross-reference window, suggesting other ganglia may have detected the event. The per-ganglion miss has been recorded."
                : "A trigger was found for this situation within the cross-reference window. The report has been recorded. If the detection was correct, this report may inflate the missed count.");
    } else if (!result.crossRefConclusive()) {
        response.put("advisory", "The event occurred outside the trigger history retention window. Cross-reference results may be incomplete.");
    }
    return Response.status(201).entity(response).build();
}
```

Add `UNKNOWN_GANGLION` to `describeRejection()`.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=MissedDetectionResourceTest`
Expected: PASS

- [ ] **Step 6: Full build verification**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 7: Commit**

Message: `feat(#64): MissedDetectionResource ganglionIds + cross-reference advisory response`

---

## References

- `specs/issue-62-drift-recall-quality/2026-08-29-feedback-quality-enrichment-design.md` — design spec
- `specs/issue-62-drift-recall-quality/decisions.md` — 8 design decisions
- `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java` — 6-field record gaining 3 fields
- `api/src/main/java/io/casehub/ras/api/FeedbackTuningStrategy.java` — 2-method interface gaining classifyDrift()
- `api/src/main/java/io/casehub/ras/api/MissedDetectionRecord.java` — 7-field record gaining ganglionIds
- `api/src/main/java/io/casehub/ras/api/GanglionOutcomeStatistics.java` — 5-field record gaining missedCount + recall()
- `api/src/main/java/io/casehub/ras/api/OutcomeLedger.java` — ganglionStatistics() returns extended stats
- `runtime/src/main/java/io/casehub/ras/runtime/FeedbackUpdateJob.java:82-117` — processTenant() gains drift block
- `runtime/src/main/java/io/casehub/ras/runtime/FeedbackMetrics.java` — gains drift state-gauge
- `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionRecorder.java` — gains cross-reference
- `runtime/src/main/java/io/casehub/ras/runtime/MissedDetectionResource.java` — gains ganglionIds + advisory
- `runtime/src/main/java/io/casehub/ras/runtime/DefaultTuningStrategy.java:25` — hardcoded 0.5 → config threshold
- `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedger.java:68-83` — recordMissed gains ganglion_ids
- `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedger.java:163-199` — ganglionStatistics gains missed query
- `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java:771-788` — parseFeedbackConfig gains new fields
- GitHub #62 — DriftDirection classification
- GitHub #63 — per-ganglion missed detection signals
- GitHub #64 — trigger-history cross-reference
