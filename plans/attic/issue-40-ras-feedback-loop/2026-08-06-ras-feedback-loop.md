# RAS Feedback Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #40 -- RAS feedback loop -- case outcomes feeding back into detection tuning
**Issue group:** #40

**Goal:** Close the RAS sense-decide-act-learn loop: case outcomes feed back into detection via dismiss suppression, threshold drift adjustment, NaiveBayes prior recalibration, and detection quality metrics.

**Architecture:** Composable feedback pipeline with three layers -- ingestion (OutcomeRecorder via CaseOutcomeObserver), analysis (FeedbackAnalyzer), application (SuppressionStrategy + FeedbackTuningStrategy SPIs). FeedbackState component holds mutable feedback-derived state (threshold/prior overrides) with tenant scoping. Advisory mode (default) provides suppression + metrics only; tuning mode adds automatic threshold/prior adjustment.

**Tech Stack:** Java 21, Quarkus CDI, Micrometer, Hibernate ORM (JPA persistence), Flyway, JUnit 5

## Global Constraints

- `casehub-engine-api` already on runtime/ classpath -- no new dependency needed
- All feedback state is tenant-scoped: keyed by `(id, tenancyId)`
- `DefaultRasTriggerPolicy` is NOT modified -- stays a pure function
- Suppression is checked in `SituationEvaluator` BEFORE detection
- `FeedbackConfig.tuningEnabled` defaults to `false` (advisory mode)
- `retentionPeriod >= suppressionCooldown` invariant on `FeedbackConfig`
- NaiveBayes priors stored in log-space internally; `FeedbackState.applyPriorOverride()` converts raw -> log at the boundary
- `UNIQUE(case_id)` constraint on outcome records for at-least-once dedup
- Flyway migration version: V7
- YAML field name `ganglionId:` (not `id:`)

---

### Task 1: Core Types, SPIs, and SituationDefinition Changes

**Files:**
- Create: `api/src/main/java/io/casehub/ras/api/OutcomeClassification.java`
- Create: `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java`
- Create: `api/src/main/java/io/casehub/ras/api/OutcomeRecord.java`
- Create: `api/src/main/java/io/casehub/ras/api/OutcomeStatistics.java`
- Create: `api/src/main/java/io/casehub/ras/api/OutcomeLedger.java`
- Create: `api/src/main/java/io/casehub/ras/api/SuppressionStrategy.java`
- Create: `api/src/main/java/io/casehub/ras/api/FeedbackTuningStrategy.java`
- Create: `api/src/main/java/io/casehub/ras/api/AbstractOutcomeLedgerContractTest.java` (test-jar)
- Modify: `api/src/main/java/io/casehub/ras/api/SituationDefinition.java`
- Test: `api/src/test/java/io/casehub/ras/api/FeedbackConfigTest.java`
- Test: `api/src/test/java/io/casehub/ras/api/OutcomeStatisticsTest.java`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: `OutcomeClassification`, `FeedbackConfig`, `OutcomeRecord`, `OutcomeStatistics`, `OutcomeLedger`, `SuppressionStrategy`, `FeedbackTuningStrategy`. `SituationDefinition` gains `feedbackConfig()` accessor and `withChainMode(ChainMode)` method.

- [ ] **Step 1: Create OutcomeClassification enum**

```java
// api/src/main/java/io/casehub/ras/api/OutcomeClassification.java
package io.casehub.ras.api;

public enum OutcomeClassification {
    NOISE, CONFIRMED, NEUTRAL
}
```

- [ ] **Step 2: Write FeedbackConfig tests**

```java
// api/src/test/java/io/casehub/ras/api/FeedbackConfigTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class FeedbackConfigTest {

    @Test void classifyNoise() {
        var config = validConfig();
        assertEquals(OutcomeClassification.NOISE, config.classify("dismissed"));
    }

    @Test void classifyConfirmed() {
        var config = validConfig();
        assertEquals(OutcomeClassification.CONFIRMED, config.classify("escalated"));
    }

    @Test void classifyNeutral() {
        var config = validConfig();
        assertEquals(OutcomeClassification.NEUTRAL, config.classify("unknown-label"));
    }

    @Test void disjointLabelsRequired() {
        assertThrows(IllegalArgumentException.class, () ->
            new FeedbackConfig(Set.of("dismissed"), Set.of("dismissed"),
                Duration.ofHours(6), 0.1, Duration.ofDays(90), false));
    }

    @Test void cooldownMustBePositive() {
        assertThrows(IllegalArgumentException.class, () ->
            new FeedbackConfig(Set.of("dismissed"), Set.of("escalated"),
                Duration.ZERO, 0.1, Duration.ofDays(90), false));
    }

    @Test void learningRateBounds() {
        assertThrows(IllegalArgumentException.class, () ->
            new FeedbackConfig(Set.of("dismissed"), Set.of("escalated"),
                Duration.ofHours(6), 0.0, Duration.ofDays(90), false));
        assertThrows(IllegalArgumentException.class, () ->
            new FeedbackConfig(Set.of("dismissed"), Set.of("escalated"),
                Duration.ofHours(6), 1.1, Duration.ofDays(90), false));
    }

    @Test void retentionMustBeCooldownOrGreater() {
        assertThrows(IllegalArgumentException.class, () ->
            new FeedbackConfig(Set.of("dismissed"), Set.of("escalated"),
                Duration.ofHours(6), 0.1, Duration.ofHours(1), false));
    }

    @Test void tuningEnabledDefaults() {
        var config = validConfig();
        assertFalse(config.tuningEnabled());
    }

    private FeedbackConfig validConfig() {
        return new FeedbackConfig(Set.of("dismissed", "false-positive"),
            Set.of("escalated", "confirmed"), Duration.ofHours(6), 0.1,
            Duration.ofDays(90), false);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=FeedbackConfigTest`
Expected: FAIL -- FeedbackConfig class not found

- [ ] **Step 4: Create FeedbackConfig record**

```java
// api/src/main/java/io/casehub/ras/api/FeedbackConfig.java
package io.casehub.ras.api;

import java.time.Duration;
import java.util.Collections;
import java.util.Objects;
import java.util.Set;

public record FeedbackConfig(
        Set<String> noiseLabels,
        Set<String> confirmedLabels,
        Duration suppressionCooldown,
        double learningRate,
        Duration retentionPeriod,
        boolean tuningEnabled
) {
    public FeedbackConfig {
        Objects.requireNonNull(noiseLabels, "noiseLabels");
        Objects.requireNonNull(confirmedLabels, "confirmedLabels");
        Objects.requireNonNull(suppressionCooldown, "suppressionCooldown");
        Objects.requireNonNull(retentionPeriod, "retentionPeriod");
        noiseLabels = Set.copyOf(noiseLabels);
        confirmedLabels = Set.copyOf(confirmedLabels);
        if (!Collections.disjoint(noiseLabels, confirmedLabels)) {
            throw new IllegalArgumentException(
                "noiseLabels and confirmedLabels must be disjoint");
        }
        if (suppressionCooldown.isZero() || suppressionCooldown.isNegative()) {
            throw new IllegalArgumentException(
                "suppressionCooldown must be positive, got: " + suppressionCooldown);
        }
        if (learningRate <= 0.0 || learningRate > 1.0) {
            throw new IllegalArgumentException(
                "learningRate must be in (0.0, 1.0], got: " + learningRate);
        }
        if (retentionPeriod.isZero() || retentionPeriod.isNegative()) {
            throw new IllegalArgumentException(
                "retentionPeriod must be positive, got: " + retentionPeriod);
        }
        if (retentionPeriod.compareTo(suppressionCooldown) < 0) {
            throw new IllegalArgumentException(
                "retentionPeriod must be >= suppressionCooldown: "
                + retentionPeriod + " < " + suppressionCooldown);
        }
    }

    public OutcomeClassification classify(String outcomeLabel) {
        if (noiseLabels.contains(outcomeLabel)) return OutcomeClassification.NOISE;
        if (confirmedLabels.contains(outcomeLabel)) return OutcomeClassification.CONFIRMED;
        return OutcomeClassification.NEUTRAL;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=FeedbackConfigTest`
Expected: PASS

- [ ] **Step 6: Create OutcomeRecord, OutcomeStatistics, and write OutcomeStatistics tests**

```java
// api/src/main/java/io/casehub/ras/api/OutcomeRecord.java
package io.casehub.ras.api;

import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

public record OutcomeRecord(
        String situationId,
        String correlationKey,
        String tenancyId,
        String outcomeLabel,
        OutcomeClassification classification,
        Instant closedAt,
        UUID caseId
) {
    public OutcomeRecord {
        Objects.requireNonNull(situationId, "situationId");
        Objects.requireNonNull(correlationKey, "correlationKey");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(outcomeLabel, "outcomeLabel");
        Objects.requireNonNull(classification, "classification");
        Objects.requireNonNull(closedAt, "closedAt");
        Objects.requireNonNull(caseId, "caseId");
    }
}
```

```java
// api/src/main/java/io/casehub/ras/api/OutcomeStatistics.java
package io.casehub.ras.api;

import java.time.Instant;

public record OutcomeStatistics(
        String situationId,
        String tenancyId,
        long totalOutcomes,
        long noiseCount,
        long confirmedCount,
        long neutralCount,
        Instant windowStart
) {
    public double precision() {
        long decisive = confirmedCount + noiseCount;
        return decisive == 0 ? Double.NaN : (double) confirmedCount / decisive;
    }

    public double noiseRate() {
        return totalOutcomes == 0 ? Double.NaN : (double) noiseCount / totalOutcomes;
    }
}
```

```java
// api/src/test/java/io/casehub/ras/api/OutcomeStatisticsTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class OutcomeStatisticsTest {

    @Test void precisionWithMixedOutcomes() {
        var stats = new OutcomeStatistics("s1", "t1", 10, 3, 7, 0, Instant.now());
        assertEquals(0.7, stats.precision(), 0.001);
    }

    @Test void precisionNaNWhenNoDecisiveOutcomes() {
        var stats = new OutcomeStatistics("s1", "t1", 5, 0, 0, 5, Instant.now());
        assertTrue(Double.isNaN(stats.precision()));
    }

    @Test void noiseRateComputation() {
        var stats = new OutcomeStatistics("s1", "t1", 10, 6, 4, 0, Instant.now());
        assertEquals(0.6, stats.noiseRate(), 0.001);
    }

    @Test void noiseRateNaNWhenEmpty() {
        var stats = new OutcomeStatistics("s1", "t1", 0, 0, 0, 0, Instant.now());
        assertTrue(Double.isNaN(stats.noiseRate()));
    }
}
```

- [ ] **Step 7: Run OutcomeStatistics tests**

Run: `mvn --batch-mode test -pl api -Dtest=OutcomeStatisticsTest`
Expected: PASS

- [ ] **Step 8: Create OutcomeLedger SPI**

```java
// api/src/main/java/io/casehub/ras/api/OutcomeLedger.java
package io.casehub.ras.api;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

public interface OutcomeLedger {
    void record(OutcomeRecord record);
    OutcomeStatistics statistics(String situationId, String tenancyId, Instant since);
    Optional<Instant> lastNoiseDismissalTime(String situationId, String correlationKey,
                                              String tenancyId);
    Map<String, Long> countByLabel(String situationId, String tenancyId, Instant since);
    Set<String> distinctTenancies(String situationId);
    int removeRecordsBefore(String situationId, Instant cutoff);
}
```

- [ ] **Step 9: Create SuppressionStrategy and FeedbackTuningStrategy SPIs**

```java
// api/src/main/java/io/casehub/ras/api/SuppressionStrategy.java
package io.casehub.ras.api;

import java.time.Instant;
import java.util.Optional;

public interface SuppressionStrategy {
    boolean shouldSuppress(String situationId, String correlationKey, String tenancyId,
                           FeedbackConfig config, Optional<Instant> lastNoiseDismissalTime);
}
```

```java
// api/src/main/java/io/casehub/ras/api/FeedbackTuningStrategy.java
package io.casehub.ras.api;

import java.util.Optional;
import java.util.OptionalDouble;

public interface FeedbackTuningStrategy {
    OptionalDouble adjustThreshold(OutcomeStatistics statistics, double currentThreshold,
                                    FeedbackConfig config);
    Optional<double[]> adjustPriors(double[] currentPriors, long[] outcomeCounts,
                                     FeedbackConfig config);
}
```

- [ ] **Step 10: Modify SituationDefinition -- add feedbackConfig field and withChainMode**

Add `@Nullable FeedbackConfig feedbackConfig` as the 11th field. Add new canonical constructor, update compact constructor, add `withChainMode(ChainMode)`. Use `ide_replace_member` for the record body.

New record declaration:
```java
public record SituationDefinition(
        String situationId,
        Set<String> eventTypes,
        Duration correlationWindow,
        Duration eventBufferDelay,
        ChainMode chainMode,
        TriggerAction triggerAction,
        TriggerMode triggerMode,
        ExpressionEvaluator correlationKeyExpression,
        ExpressionEvaluator eventFilter,
        Map<String, ExpressionEvaluator> dynamicCaseData,
        FeedbackConfig feedbackConfig
) { ... }
```

Add convenience constructor (existing 10-arg defaults `feedbackConfig` to `null`).
Add 7-arg constructor (defaults both expression fields and feedbackConfig to `null`).
Add `withChainMode(ChainMode newChainMode)` method:
```java
public SituationDefinition withChainMode(ChainMode newChainMode) {
    return new SituationDefinition(situationId, eventTypes, correlationWindow,
        eventBufferDelay, newChainMode, triggerAction, triggerMode,
        correlationKeyExpression, eventFilter, dynamicCaseData, feedbackConfig);
}
```

Fix all existing callers that construct `SituationDefinition` with 10 args -- they now need an 11th `null` parameter. Use `ide_find_references` on the canonical constructor to find all call sites.

- [ ] **Step 11: Create AbstractOutcomeLedgerContractTest**

```java
// api/src/test/java/io/casehub/ras/api/AbstractOutcomeLedgerContractTest.java
package io.casehub.ras.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

public abstract class AbstractOutcomeLedgerContractTest {

    protected abstract OutcomeLedger createLedger();
    private OutcomeLedger ledger;

    @BeforeEach void setUp() { ledger = createLedger(); }

    @Test void recordAndStatistics() {
        Instant since = Instant.now().minusSeconds(3600);
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "closed",
            OutcomeClassification.NEUTRAL, Instant.now(), UUID.randomUUID()));

        OutcomeStatistics stats = ledger.statistics("s1", "t1", since);
        assertEquals(3, stats.totalOutcomes());
        assertEquals(1, stats.noiseCount());
        assertEquals(1, stats.confirmedCount());
        assertEquals(1, stats.neutralCount());
    }

    @Test void lastNoiseDismissalTime() {
        Instant early = Instant.now().minusSeconds(3600);
        Instant late = Instant.now().minusSeconds(60);
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, early, UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, late, UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID()));

        Optional<Instant> result = ledger.lastNoiseDismissalTime("s1", "k1", "t1");
        assertTrue(result.isPresent());
        assertEquals(late, result.get());
    }

    @Test void lastNoiseDismissalTimeEmpty() {
        assertTrue(ledger.lastNoiseDismissalTime("s1", "k1", "t1").isEmpty());
    }

    @Test void multiTenantIsolation() {
        Instant since = Instant.now().minusSeconds(3600);
        ledger.record(outcome("s1", "k1", "tenantA", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "tenantB", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID()));

        assertEquals(1, ledger.statistics("s1", "tenantA", since).noiseCount());
        assertEquals(0, ledger.statistics("s1", "tenantA", since).confirmedCount());
        assertEquals(0, ledger.statistics("s1", "tenantB", since).noiseCount());
        assertEquals(1, ledger.statistics("s1", "tenantB", since).confirmedCount());
    }

    @Test void removeRecordsBefore() {
        Instant cutoff = Instant.now();
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, cutoff.minusSeconds(60), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "escalated",
            OutcomeClassification.CONFIRMED, cutoff.plusSeconds(60), UUID.randomUUID()));

        int removed = ledger.removeRecordsBefore("s1", cutoff);
        assertEquals(1, removed);
        assertEquals(1, ledger.statistics("s1", "t1", Instant.EPOCH).totalOutcomes());
    }

    @Test void duplicateCaseIdIgnored() {
        UUID caseId = UUID.randomUUID();
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), caseId));
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), caseId));

        assertEquals(1, ledger.statistics("s1", "t1", Instant.EPOCH).totalOutcomes());
    }

    @Test void countByLabel() {
        Instant since = Instant.now().minusSeconds(3600);
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "t1", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID()));

        Map<String, Long> counts = ledger.countByLabel("s1", "t1", since);
        assertEquals(2L, counts.get("dismissed"));
        assertEquals(1L, counts.get("escalated"));
    }

    @Test void distinctTenancies() {
        ledger.record(outcome("s1", "k1", "tenantA", "dismissed",
            OutcomeClassification.NOISE, Instant.now(), UUID.randomUUID()));
        ledger.record(outcome("s1", "k1", "tenantB", "escalated",
            OutcomeClassification.CONFIRMED, Instant.now(), UUID.randomUUID()));

        Set<String> tenancies = ledger.distinctTenancies("s1");
        assertEquals(Set.of("tenantA", "tenantB"), tenancies);
    }

    @Test void statisticsEmptyWhenNoRecords() {
        OutcomeStatistics stats = ledger.statistics("s1", "t1", Instant.EPOCH);
        assertEquals(0, stats.totalOutcomes());
        assertTrue(Double.isNaN(stats.precision()));
        assertTrue(Double.isNaN(stats.noiseRate()));
    }

    protected OutcomeRecord outcome(String situationId, String correlationKey,
            String tenancyId, String label, OutcomeClassification classification,
            Instant closedAt, UUID caseId) {
        return new OutcomeRecord(situationId, correlationKey, tenancyId,
            label, classification, closedAt, caseId);
    }
}
```

- [ ] **Step 12: Run all api/ tests**

Run: `mvn --batch-mode test -pl api`
Expected: PASS (contract test is abstract -- not run directly)

- [ ] **Step 13: Commit**

```
feat(#40): add feedback loop core types, SPIs, and SituationDefinition changes

Refs #40
```

---

### Task 2: InMemoryOutcomeLedger

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/InMemoryOutcomeLedger.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/InMemoryOutcomeLedgerTest.java`

**Interfaces:**
- Consumes: `OutcomeLedger`, `OutcomeRecord`, `OutcomeStatistics`, `OutcomeClassification` from Task 1
- Produces: `InMemoryOutcomeLedger` -- `@DefaultBean` implementation of `OutcomeLedger`

- [ ] **Step 1: Write test extending contract test**

```java
// runtime/src/test/java/io/casehub/ras/runtime/InMemoryOutcomeLedgerTest.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.AbstractOutcomeLedgerContractTest;
import io.casehub.ras.api.OutcomeLedger;

class InMemoryOutcomeLedgerTest extends AbstractOutcomeLedgerContractTest {
    @Override
    protected OutcomeLedger createLedger() {
        return new InMemoryOutcomeLedger();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=InMemoryOutcomeLedgerTest`
Expected: FAIL -- InMemoryOutcomeLedger not found

- [ ] **Step 3: Implement InMemoryOutcomeLedger**

```java
// runtime/src/main/java/io/casehub/ras/runtime/InMemoryOutcomeLedger.java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@ApplicationScoped
@DefaultBean
public class InMemoryOutcomeLedger implements OutcomeLedger {

    private final ConcurrentHashMap<String, List<OutcomeRecord>> store =
        new ConcurrentHashMap<>();
    private final Set<UUID> seenCaseIds = ConcurrentHashMap.newKeySet();

    private static String key(String situationId, String tenancyId) {
        return situationId + "|" + tenancyId;
    }

    @Override
    public void record(OutcomeRecord record) {
        if (!seenCaseIds.add(record.caseId())) return;
        store.computeIfAbsent(key(record.situationId(), record.tenancyId()),
            k -> Collections.synchronizedList(new ArrayList<>())).add(record);
    }

    @Override
    public OutcomeStatistics statistics(String situationId, String tenancyId,
                                         Instant since) {
        List<OutcomeRecord> records = store.getOrDefault(
            key(situationId, tenancyId), List.of());
        long noise = 0, confirmed = 0, neutral = 0;
        synchronized (records) {
            for (OutcomeRecord r : records) {
                if (!r.closedAt().isBefore(since)) {
                    switch (r.classification()) {
                        case NOISE -> noise++;
                        case CONFIRMED -> confirmed++;
                        case NEUTRAL -> neutral++;
                    }
                }
            }
        }
        return new OutcomeStatistics(situationId, tenancyId,
            noise + confirmed + neutral, noise, confirmed, neutral, since);
    }

    @Override
    public Optional<Instant> lastNoiseDismissalTime(String situationId,
            String correlationKey, String tenancyId) {
        List<OutcomeRecord> records = store.getOrDefault(
            key(situationId, tenancyId), List.of());
        synchronized (records) {
            return records.stream()
                .filter(r -> r.correlationKey().equals(correlationKey)
                    && r.classification() == OutcomeClassification.NOISE)
                .map(OutcomeRecord::closedAt)
                .max(Instant::compareTo);
        }
    }

    @Override
    public Map<String, Long> countByLabel(String situationId, String tenancyId,
                                           Instant since) {
        List<OutcomeRecord> records = store.getOrDefault(
            key(situationId, tenancyId), List.of());
        synchronized (records) {
            return records.stream()
                .filter(r -> !r.closedAt().isBefore(since))
                .collect(Collectors.groupingBy(
                    OutcomeRecord::outcomeLabel, Collectors.counting()));
        }
    }

    @Override
    public Set<String> distinctTenancies(String situationId) {
        return store.keySet().stream()
            .filter(k -> k.startsWith(situationId + "|"))
            .map(k -> k.substring(k.indexOf('|') + 1))
            .collect(Collectors.toSet());
    }

    @Override
    public int removeRecordsBefore(String situationId, Instant cutoff) {
        int removed = 0;
        for (var entry : store.entrySet()) {
            if (entry.getKey().startsWith(situationId + "|")) {
                List<OutcomeRecord> records = entry.getValue();
                synchronized (records) {
                    int before = records.size();
                    records.removeIf(r -> r.closedAt().isBefore(cutoff));
                    removed += before - records.size();
                    records.stream()
                        .filter(r -> r.closedAt().isBefore(cutoff))
                        .forEach(r -> seenCaseIds.remove(r.caseId()));
                }
            }
        }
        return removed;
    }
}
```

- [ ] **Step 4: Run contract tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=InMemoryOutcomeLedgerTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#40): add InMemoryOutcomeLedger with @DefaultBean

Refs #40
```

---

### Task 3: FeedbackState and Default Strategies

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackState.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/DefaultSuppressionStrategy.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/DefaultTuningStrategy.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackStateTest.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/DefaultSuppressionStrategyTest.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/DefaultTuningStrategyTest.java`

**Interfaces:**
- Consumes: `SuppressionStrategy`, `FeedbackTuningStrategy`, `FeedbackConfig`, `OutcomeStatistics` from Task 1
- Produces: `FeedbackState` (effectiveThreshold, adjustedLogPriors, applyThresholdOverride, applyPriorOverride, currentRawPriors), `DefaultSuppressionStrategy`, `DefaultTuningStrategy`

- [ ] **Step 1: Write FeedbackState tests**

Tests: threshold override storage/retrieval per tenant, prior override validation (rejects NaN, zero, negative), log-space conversion, currentRawPriors fallback to base, concurrent access safety.

- [ ] **Step 2: Implement FeedbackState**

`@ApplicationScoped` with two `ConcurrentHashMap<StateKey, ?>` maps keyed by `record StateKey(String id, String tenancyId)`. `applyThresholdOverride` validates `(0.0, 1.0]`. `applyPriorOverride` validates no NaN/zero/negative, then converts to log-space via `Math.log()`.

- [ ] **Step 3: Run FeedbackState tests -- PASS**

- [ ] **Step 4: Write DefaultSuppressionStrategy tests**

Tests: returns true when lastDismissal within cooldown, false when outside cooldown, false when empty, false when config has no noise labels.

- [ ] **Step 5: Implement DefaultSuppressionStrategy**

`@DefaultBean`. `shouldSuppress` returns `lastNoiseDismissalTime.isPresent() && lastNoiseDismissalTime.get().plus(config.suppressionCooldown()).isAfter(Instant.now())`.

- [ ] **Step 6: Run DefaultSuppressionStrategy tests -- PASS**

- [ ] **Step 7: Write DefaultTuningStrategy tests**

Tests: threshold increases when noiseRate > 0.5, no change when insufficient data (< 10 outcomes), result clamped to (0.0, 1.0], prior blending with Laplace smoothing, prior returns empty when < 5 total counts, no zero priors in output.

- [ ] **Step 8: Implement DefaultTuningStrategy**

`@DefaultBean`. `adjustThreshold`: returns empty when `totalOutcomes < 10`, else `currentThreshold + learningRate * (noiseRate - 0.5)` clamped to `(0.0, 1.0]`. `adjustPriors`: returns empty when total < 5, else Laplace-smoothed blend `(1-lr)*old + lr*empirical`, renormalized.

- [ ] **Step 9: Run all strategy tests -- PASS**

- [ ] **Step 10: Commit**

```
feat(#40): add FeedbackState and default suppression/tuning strategies

Refs #40
```

---

### Task 4: NaiveBayes + GanglionDescriptor + Registry Integration

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java` (NaiveBayes gains outcomeGroundTruth)
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/NaiveBayesConfig.java` (outcomeGroundTruth)
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationDefinitionRegistry.java` (feedbackConfig, allSituationIds, ganglionDescriptor, descriptorsById, FeedbackState passthrough)
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/NaiveBayesGanglion.java` (FeedbackState parameter)
- Test: `runtime/src/test/java/io/casehub/ras/runtime/NaiveBayesGanglionFeedbackTest.java`

**Interfaces:**
- Consumes: `FeedbackState` from Task 3, existing `GanglionDescriptor.NaiveBayes`, `NaiveBayesConfig`, `SituationDefinitionRegistry`, `NaiveBayesGanglion`
- Produces: `GanglionDescriptor.NaiveBayes.outcomeGroundTruth()`, `NaiveBayesConfig.outcomeGroundTruth()`, `SituationDefinitionRegistry.feedbackConfig(String)`, `SituationDefinitionRegistry.allSituationIds()`, `SituationDefinitionRegistry.ganglionDescriptor(String)`, `NaiveBayesGanglion` with FeedbackState-adjusted initial priors

- [ ] **Step 1: Add outcomeGroundTruth to GanglionDescriptor.NaiveBayes**

New field `@Nullable Map<String, String> outcomeGroundTruth` on the record. Existing constructor callers pass `null`. Add validation: if present, every value must exist in `outcomes` list.

- [ ] **Step 2: Add outcomeGroundTruth to NaiveBayesConfig**

New `@Nullable Map<String, String> outcomeGroundTruth` field. Existing constructor callers pass `null`. Same validation.

- [ ] **Step 3: Add SituationDefinitionRegistry methods**

`feedbackConfig(String situationId)` -- queries snapshot for the situation definition and returns its feedbackConfig.
`allSituationIds()` -- returns `snapshot.situationIds()`.
`ganglionDescriptor(String ganglionId)` -- returns from new `descriptorsById` map (populated during Phase 1 alongside `gangliaById`).
Phase 1 loop retains descriptors: `descriptorsById.put(descriptor.ganglionId(), descriptor)`.
Constructor gains `Instance<FeedbackState>` parameter for passthrough to ganglion construction.

- [ ] **Step 4: Write NaiveBayesGanglion feedback test**

Test: when FeedbackState has adjusted priors for a tenant, new situation instances use those priors instead of config priors. When FeedbackState has no priors, config priors are used.

- [ ] **Step 5: Modify NaiveBayesGanglion**

4-arg constructor adds `@Nullable FeedbackState feedbackState`. In `detect()`, where state is initialised for new situations:
```java
.orElseGet(() -> {
    double[] priors = feedbackState != null
        ? feedbackState.adjustedLogPriors(config.ganglionId(), context.tenancyId())
            .orElse(Arrays.copyOf(logPriors, logPriors.length))
        : Arrays.copyOf(logPriors, logPriors.length);
    return new GanglionState(priors, OptionalLong.empty());
});
```

- [ ] **Step 6: Run tests -- PASS**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#40): add outcomeGroundTruth, registry accessors, and NaiveBayes FeedbackState integration

Refs #40
```

---

### Task 5: SituationEvaluator Integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationEvaluator.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/RasMetrics.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/SituationEvaluatorFeedbackTest.java`

**Interfaces:**
- Consumes: `SuppressionStrategy`, `OutcomeLedger`, `FeedbackState`, `FeedbackConfig` from Tasks 1-3. `SituationDefinition.feedbackConfig()`, `SituationDefinition.withChainMode()` from Task 1.
- Produces: Pre-detection suppression check. Effective threshold construction before policy evaluation.

- [ ] **Step 1: Write evaluator feedback tests**

Tests: (a) event suppressed when within cooldown -- returns early, no ganglia invoked. (b) event not suppressed when outside cooldown. (c) threshold override applied -- policy sees adjusted minConfidence. (d) no feedback when dependencies absent (Instance empty). (e) no feedback when feedbackConfig is null.

- [ ] **Step 2: Add Instance dependencies to SituationEvaluator constructor**

`Instance<SuppressionStrategy> suppressionStrategy`, `Instance<OutcomeLedger> outcomeLedger`, `Instance<FeedbackState> feedbackState`. All optional via `isResolvable()`.

- [ ] **Step 3: Add suppression check to processEvent()**

Before `loadContext()` call:
```java
FeedbackConfig feedbackConfig = definition.feedbackConfig();
if (feedbackConfig != null
        && suppressionStrategy.isResolvable()
        && outcomeLedger.isResolvable()) {
    Optional<Instant> lastDismissal = outcomeLedger.get()
        .lastNoiseDismissalTime(definition.situationId(), correlationKey, tenancyId);
    if (suppressionStrategy.get().shouldSuppress(
            definition.situationId(), correlationKey, tenancyId,
            feedbackConfig, lastDismissal)) {
        metrics.feedbackSuppression(definition.situationId(), tenancyId);
        return true;
    }
}
```

- [ ] **Step 4: Add effective threshold construction before policy evaluation**

After detection, before `triggerPolicy.evaluate()`:
```java
SituationDefinition effectiveDef = definition;
if (feedbackState.isResolvable()
        && definition.chainMode() instanceof ChainMode.Threshold threshold) {
    OptionalDouble adjusted = feedbackState.get()
        .effectiveThreshold(definition.situationId(), tenancyId);
    if (adjusted.isPresent()) {
        effectiveDef = definition.withChainMode(
            new ChainMode.Threshold(threshold.ganglia(), adjusted.getAsDouble()));
    }
}
PolicyDecision policyDecision = triggerPolicy.evaluate(context, effectiveDef);
```

- [ ] **Step 5: Add feedbackSuppression metric to RasMetrics**

`feedbackSuppression(String situationId, String tenancyId)` -- counter `ras.feedback.suppressions_total`.

- [ ] **Step 6: Run tests -- PASS**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#40): add pre-detection suppression and effective threshold to SituationEvaluator

Refs #40
```

---

### Task 6: OutcomeRecorder, FeedbackAnalyzer, FeedbackUpdateJob, FeedbackMetrics

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/OutcomeRecorder.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackAnalyzer.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackUpdateJob.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/FeedbackMetrics.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/OutcomeRecorderTest.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackAnalyzerTest.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/FeedbackUpdateJobTest.java`

**Interfaces:**
- Consumes: `OutcomeLedger`, `FeedbackConfig`, `OutcomeStatistics`, `FeedbackTuningStrategy`, `FeedbackState`, `SituationDefinitionRegistry` from Tasks 1-4. `CaseOutcomeObserver`/`CaseOutcomeEvent` from `casehub-engine-api`.
- Produces: `OutcomeRecorder` (CaseOutcomeObserver impl), `FeedbackAnalyzer`, `FeedbackUpdateJob` (@Scheduled), `FeedbackMetrics`

- [ ] **Step 1: Write OutcomeRecorder tests**

Tests: (a) records outcome with correct classification. (b) skips non-RAS case (no situationId in snapshot). (c) skips when no feedbackConfig. (d) skips when correlationKey null. (e) swallows exceptions from ledger.

- [ ] **Step 2: Implement OutcomeRecorder**

`@ApplicationScoped implements CaseOutcomeObserver`. Extracts `situationId`, `correlationKey` from `caseFileSnapshot()`. Looks up `FeedbackConfig` via `registry.feedbackConfig()`. Classifies via `config.classify()`. Wraps `ledger.record()` in try/catch.

- [ ] **Step 3: Write FeedbackAnalyzer tests**

Tests: (a) returns statistics within retention window. (b) empty when no records.

- [ ] **Step 4: Implement FeedbackAnalyzer**

`@ApplicationScoped`. Single method `analyze(situationId, tenancyId, config)` that queries `ledger.statistics()` with `Instant.now().minus(config.retentionPeriod())`.

- [ ] **Step 5: Write FeedbackUpdateJob tests**

Tests: (a) iterates all situations with feedbackConfig. (b) applies threshold override when tuningEnabled. (c) skips threshold adjustment when tuningEnabled=false. (d) maps outcome labels to NaiveBayes outcomes via outcomeGroundTruth. (e) applies prior override for NaiveBayes ganglia. (f) calls cleanup per-situation. (g) tenant isolation -- different tenants get independent overrides.

- [ ] **Step 6: Implement FeedbackUpdateJob**

`@ApplicationScoped` with `@Scheduled(every = "${ras.feedback.update-interval:PT5M}")`. Iterates `registry.allSituationIds()`, checks for feedbackConfig, iterates `ledger.distinctTenancies()`, calls analyzer, applies strategies via FeedbackState, updates metrics, runs cleanup.

- [ ] **Step 7: Implement FeedbackMetrics**

Micrometer gauges for `ras.feedback.precision`, `ras.feedback.noise_rate`, `ras.feedback.outcomes_total`. Optional via `Instance<MeterRegistry>`.

- [ ] **Step 8: Run all tests -- PASS**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS

- [ ] **Step 9: Commit**

```
feat(#40): add OutcomeRecorder, FeedbackAnalyzer, FeedbackUpdateJob, and FeedbackMetrics

Refs #40
```

---

### Task 7: YAML Parsing

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java`
- Create: `runtime/src/test/resources/META-INF/ras-situations-feedback.yaml` (test resource)

**Interfaces:**
- Consumes: `FeedbackConfig`, `GanglionDescriptor.NaiveBayes.outcomeGroundTruth` from Tasks 1, 4
- Produces: YAML parsing for `feedback:` section on situations and `outcomeGroundTruth:` on NaiveBayes ganglia

- [ ] **Step 1: Create test YAML resource**

```yaml
# runtime/src/test/resources/META-INF/ras-situations-feedback.yaml
ganglia:
  - ganglionId: feedback-nb
    type: naive-bayes
    outcomes: [fraud, legitimate]
    priors: [0.1, 0.9]
    outcomeGroundTruth:
      escalated: fraud
      dismissed: legitimate
    features:
      amount:
        values: [high, low]
        likelihoods:
          - [0.8, 0.2]
          - [0.3, 0.7]
    signalMapping:
      targetOutcome: fraud
      detectedThreshold: 0.7
      weakThreshold: 0.4

situations:
  - situationId: feedback-test
    eventTypes: [payment.flagged]
    chainMode:
      type: threshold
      ganglia: [feedback-nb]
      minConfidence: 0.7
    triggerAction:
      type: create-case
      caseNamespace: compliance
      caseName: fraud
      caseVersion: "1.0"
    feedback:
      noiseLabels: [dismissed, false-positive]
      confirmedLabels: [escalated]
      suppressionCooldown: PT6H
      learningRate: 0.1
      retentionPeriod: P90D
      tuningEnabled: true
```

- [ ] **Step 2: Write YAML parsing tests**

Tests: (a) feedback section parsed into FeedbackConfig on SituationDefinition. (b) absent feedback section results in null feedbackConfig. (c) outcomeGroundTruth parsed on NaiveBayes descriptor. (d) invalid outcomeGroundTruth value (not in outcomes list) fails loudly. (e) tuningEnabled defaults to false when absent.

- [ ] **Step 3: Add feedback parsing to YamlSituationDefinitionProvider**

In `parseSituation()`, parse optional `feedback:` map:
```java
Map<String, Object> feedbackMap = (Map<String, Object>) situationMap.get("feedback");
FeedbackConfig feedbackConfig = feedbackMap != null ? parseFeedbackConfig(feedbackMap) : null;
```

`parseFeedbackConfig` extracts noiseLabels, confirmedLabels, suppressionCooldown (ISO-8601), learningRate, retentionPeriod (ISO-8601), tuningEnabled (default false).

In `parseNaiveBayes()`, parse optional `outcomeGroundTruth:` map and pass to constructor.

- [ ] **Step 4: Run tests -- PASS**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#40): add YAML parsing for feedback config and outcomeGroundTruth

Refs #40
```

---

### Task 8: JPA Persistence

**Files:**
- Create: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/OutcomeRecordEntity.java`
- Create: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedger.java`
- Create: `persistence-jpa/src/main/resources/db/ras/migration/V7__create_ras_outcome_record.sql`
- Test: `persistence-jpa/src/test/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedgerTest.java`

**Interfaces:**
- Consumes: `OutcomeLedger`, `OutcomeRecord`, `OutcomeStatistics`, `OutcomeClassification` from Task 1
- Produces: `JpaOutcomeLedger` -- `@ApplicationScoped` JPA-backed implementation

- [ ] **Step 1: Create Flyway migration**

```sql
-- persistence-jpa/src/main/resources/db/ras/migration/V7__create_ras_outcome_record.sql
CREATE TABLE ras_outcome_record (
    id              BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    situation_id    VARCHAR(255) NOT NULL,
    correlation_key VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    outcome_label   VARCHAR(255) NOT NULL,
    classification  VARCHAR(20)  NOT NULL,
    closed_at       TIMESTAMP    NOT NULL,
    case_id         UUID         NOT NULL,
    CONSTRAINT uq_outcome_case_id UNIQUE (case_id)
);

CREATE INDEX idx_outcome_situation_tenant
    ON ras_outcome_record (situation_id, tenancy_id, closed_at);

CREATE INDEX idx_outcome_suppression
    ON ras_outcome_record (situation_id, correlation_key, tenancy_id,
                           classification, closed_at DESC);
```

- [ ] **Step 2: Create OutcomeRecordEntity**

JPA entity mapping to `ras_outcome_record`. Fields match the table. `classification` stored as `VARCHAR` via `@Enumerated(EnumType.STRING)`.

- [ ] **Step 3: Write test extending contract test**

```java
// persistence-jpa/src/test/java/io/casehub/ras/persistence/jpa/JpaOutcomeLedgerTest.java
@QuarkusTest
class JpaOutcomeLedgerTest extends AbstractOutcomeLedgerContractTest {
    @Inject JpaOutcomeLedger ledger;

    @Override
    protected OutcomeLedger createLedger() { return ledger; }

    @BeforeEach
    @Transactional
    void cleanDb() { /* clear ras_outcome_record table */ }
}
```

- [ ] **Step 4: Implement JpaOutcomeLedger**

`@ApplicationScoped`. Uses `EntityManager` for queries.
- `record()`: `INSERT ... ON CONFLICT (case_id) DO NOTHING` via native query
- `statistics()`: `SELECT COUNT(*), SUM(CASE WHEN classification='NOISE' ...)` grouped aggregate
- `lastNoiseDismissalTime()`: `SELECT MAX(closed_at) WHERE classification='NOISE' AND ...`
- `countByLabel()`: `SELECT outcome_label, COUNT(*) ... GROUP BY outcome_label`
- `distinctTenancies()`: `SELECT DISTINCT tenancy_id WHERE situation_id = ?`
- `removeRecordsBefore()`: `DELETE FROM ... WHERE situation_id = ? AND closed_at < ?`

- [ ] **Step 5: Run contract tests -- PASS**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=JpaOutcomeLedgerTest`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#40): add JpaOutcomeLedger with V7 Flyway migration

Refs #40
```

---

### Task 9: Full Build Verification

**Files:** None (verification only)

- [ ] **Step 1: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS -- all modules compile and all tests pass

- [ ] **Step 2: Update CLAUDE.md**

Add `OutcomeLedger` SPI, `SuppressionStrategy` SPI, `FeedbackTuningStrategy` SPI, `FeedbackState`, `FeedbackConfig`, `FeedbackUpdateJob`, `OutcomeRecorder` to the module documentation and Core SPIs sections.

- [ ] **Step 3: Update consumer and contributor guides**

Update `docs/guides/consumer-guide.md` with feedback configuration YAML schema.
Update `docs/guides/contributor-guide.md` with FeedbackState, strategy SPIs, and outcome ingestion architecture.

- [ ] **Step 4: Commit**

```
docs(#40): update CLAUDE.md and guides for feedback loop

Refs #40
```
