# RAS Feedback Loop Design

**Issue:** casehubio/casehub-ras#40
**Date:** 2026-08-06
**Status:** Draft

## Problem

RAS is open-loop: detect -> trigger -> forget. Case outcomes never feed back into
detection. This means no suppression of repeat noise triggers, no threshold tuning
from outcome data, no validation of Bayesian priors against ground truth, and no
visibility into detection quality (precision/recall).

## Approach

Composable feedback pipeline with three layers:

1. **Ingestion** -- `OutcomeRecorder` receives case outcomes via the platform's
   `CaseOutcomeObserver` SPI, correlates to situations, stores in `OutcomeLedger`
2. **Analysis** -- `FeedbackAnalyzer` computes aggregate statistics (precision,
   noise rate, drift direction) from outcome records
3. **Application** -- `FeedbackStrategy` SPI decides what to do with analysis
   results. Strategy-based: default is automatic; alternative advisory strategy
   surfaces metrics without modifying detection parameters.

## Key Design Decisions

- **Feedback config on SituationDefinition** -- per-situation opt-in via
  `FeedbackConfig` field. Absent = no feedback. Outcome label classification
  (noise/confirmed) is intrinsic to the situation, not operational overlay.
- **Strategy-based application** -- `FeedbackStrategy` SPI in api/. Default
  `AutomaticFeedbackStrategy` applies all mechanisms within bounds. Alternative
  `AdvisoryFeedbackStrategy` applies suppression only, surfaces rest as metrics.
- **Direct engine-api dependency** -- `runtime/` depends on `casehub-engine-api`
  for `CaseOutcomeObserver`/`CaseOutcomeEvent`. Correct direction
  (integration -> foundation). No wrapper types.
- **Correlation via case file snapshot** -- `DefaultCaseTrigger.buildInputData()`
  already puts `situationId`, `correlationKey`, `tenancyId` into case data.
  `CaseOutcomeEvent.caseFileSnapshot()` carries these back. No separate
  case-ID-to-situation mapping needed.

## Core Types (api/)

### FeedbackConfig

New record, added as `@Nullable` field on `SituationDefinition`:

```java
record FeedbackConfig(
    Set<String> noiseLabels,         // case outcome labels meaning "this was noise"
    Set<String> confirmedLabels,     // case outcome labels meaning "this was real"
    Duration suppressionCooldown,    // per-instance cooldown after noise dismiss
    double learningRate,             // bounds prior/threshold adjustment (0.0-1.0)
    Duration retentionPeriod         // how long to keep outcome records
) {
    OutcomeClassification classify(String outcomeLabel) {
        if (noiseLabels.contains(outcomeLabel)) return OutcomeClassification.NOISE;
        if (confirmedLabels.contains(outcomeLabel)) return OutcomeClassification.CONFIRMED;
        return OutcomeClassification.NEUTRAL;
    }
}
```

Validation: `noiseLabels` and `confirmedLabels` must be disjoint. Cooldown > 0.
learningRate in (0.0, 1.0]. retentionPeriod > 0. retentionPeriod >= suppressionCooldown
(otherwise cleanup destroys records needed for active suppression checks).

### OutcomeClassification

```java
enum OutcomeClassification { NOISE, CONFIRMED, NEUTRAL }
```

### OutcomeRecord

```java
record OutcomeRecord(
    String situationId,
    String correlationKey,
    String tenancyId,
    String outcomeLabel,
    OutcomeClassification classification,
    Instant closedAt,
    UUID caseId
)
```

### OutcomeStatistics

Aggregate counts, computed by `OutcomeLedger.statistics()`:

```java
record OutcomeStatistics(
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

### FeedbackReport

Wraps statistics with derived analysis:

```java
record FeedbackReport(
    OutcomeStatistics statistics,
    DriftDirection drift,
    double precision,
    double noiseRate
) {}

enum DriftDirection {
    OVER_SENSITIVE,     // noiseRate above threshold -- too many false triggers
    UNDER_SENSITIVE,    // reserved for future external signal (missed detections)
    STABLE,
    INSUFFICIENT_DATA
}
```

### OutcomeLedger (SPI)

```java
interface OutcomeLedger {
    void record(OutcomeRecord record);
    OutcomeStatistics statistics(String situationId, String tenancyId, Instant since);
    Optional<Instant> lastNoiseDismissalTime(
        String situationId, String correlationKey, String tenancyId);
    Set<String> distinctTenancies(String situationId);
    int removeRecordsBefore(String situationId, Instant cutoff);
}
```

`statistics()` is on the SPI so JPA pushes aggregation to SQL.
`lastNoiseDismissalTime()` is a dedicated query for fast suppression checks.
`distinctTenancies()` returns `SELECT DISTINCT tenancy_id WHERE situation_id = ?` —
used by `FeedbackUpdateJob` to discover the tenant iteration scope.
`removeRecordsBefore()` is per-situation so retention periods are honoured individually.

### FeedbackStrategy (SPI)

```java
interface FeedbackStrategy {
    boolean shouldSuppress(String situationId, String correlationKey, String tenancyId,
                           FeedbackConfig config, Optional<Instant> lastNoiseDismissal);

    OptionalDouble adjustThreshold(FeedbackReport report, double currentThreshold,
                                    FeedbackConfig config);

    Optional<double[]> adjustPriors(FeedbackReport report, double[] currentPriors,
                                     List<String> outcomes, FeedbackConfig config);
}
```

## Ingestion Layer (runtime/)

### OutcomeRecorder

Implements `CaseOutcomeObserver` from `casehub-engine-api`:

```java
@ApplicationScoped
class OutcomeRecorder implements CaseOutcomeObserver {
    @Inject OutcomeLedger ledger;
    @Inject SituationDefinitionRegistry registry;

    @Override
    public void onOutcome(CaseOutcomeEvent event) {
        String situationId = (String) event.caseFileSnapshot().get("situationId");
        if (situationId == null) return;

        FeedbackConfig config = registry.feedbackConfig(situationId);
        if (config == null) return;

        String correlationKey = (String) event.caseFileSnapshot().get("correlationKey");
        if (correlationKey == null) return;

        OutcomeClassification classification = config.classify(event.outcomeLabel());

        try {
            ledger.record(new OutcomeRecord(
                situationId,
                correlationKey,
                event.tenancyId(),
                event.outcomeLabel(),
                classification,
                event.closedAt(),
                event.caseId()
            ));
        } catch (Exception e) {
            LOG.warning("Failed to record outcome for situation '"
                + situationId + "': " + e.getMessage());
        }
    }
}
```

Non-RAS cases (no situationId) and feedback-disabled situations silently skipped.
Best-effort recording -- same philosophy as `SituationEventRecorder` from #43.

## Analysis Layer (runtime/)

### FeedbackAnalyzer

```java
@ApplicationScoped
class FeedbackAnalyzer {
    @Inject OutcomeLedger ledger;

    FeedbackReport analyze(String situationId, String tenancyId, FeedbackConfig config) {
        OutcomeStatistics stats = ledger.statistics(
            situationId, tenancyId,
            Instant.now().minus(config.retentionPeriod()));

        DriftDirection drift = computeDrift(stats);
        return new FeedbackReport(stats, drift, stats.precision(), stats.noiseRate());
    }

    private DriftDirection computeDrift(OutcomeStatistics stats) {
        if (stats.totalOutcomes() < 5) return DriftDirection.INSUFFICIENT_DATA;
        long decisive = stats.confirmedCount() + stats.noiseCount();
        if (decisive < 10) return DriftDirection.INSUFFICIENT_DATA;
        if (stats.noiseRate() > 0.5) return DriftDirection.OVER_SENSITIVE;
        return DriftDirection.STABLE;
    }
}
```

Drift thresholds (0.5 noise rate for OVER_SENSITIVE) are sensible defaults on
`AutomaticFeedbackStrategy`. Custom strategies can use different cutoffs.
UNDER_SENSITIVE requires an external signal (missed detections) that RAS cannot
compute from its own outcome data -- reserved in the enum for future use but
never produced by the default analyzer.

## Application Layer (runtime/)

### AutomaticFeedbackStrategy

`@DefaultBean` in runtime/:

- **shouldSuppress**: checks `lastNoiseDismissal` instant, returns true if
  within `config.suppressionCooldown()` of current time
- **adjustThreshold**: if OVER_SENSITIVE, increases threshold by
  `learningRate * (noiseRate - 0.5)`. If UNDER_SENSITIVE, decreases similarly.
  Result clamped to [0.0, 1.0]. Bounded by learningRate to prevent the confidence
  zeroing gotcha (GE-20260714-439924 -- additive penalties at high frequency).
- **adjustPriors**: blends old priors toward empirical outcome distribution:
  `newPrior = (1 - learningRate) * oldPrior + learningRate * empiricalFreq`.
  Renormalized. Only when `totalOutcomes >= 5` (INSUFFICIENT_DATA guard).

### AdvisoryFeedbackStrategy

Alternative `FeedbackStrategy` implementation in `runtime/`:

- **shouldSuppress**: same as `AutomaticFeedbackStrategy` -- suppression is
  a safe default that prevents known-noise repeat triggers
- **adjustThreshold**: returns `OptionalDouble.empty()` -- no automatic adjustment
- **adjustPriors**: returns `Optional.empty()` -- no automatic adjustment

Metrics are surfaced through the same `FeedbackMetrics` gauges regardless of
strategy. The advisory strategy lets operators observe detection quality drift
without automated parameter changes.

## Mechanism Integration

### 1. Dismiss Suppression -- DefaultRasTriggerPolicy

Policy checks suppression BEFORE returning a trigger decision:

```java
Optional<Instant> lastDismissal = outcomeLedger.get().lastNoiseDismissalTime(
    definition.situationId(), context.correlationKey(), context.tenancyId());
if (feedback != null && feedbackStrategy.get().shouldSuppress(
        definition.situationId(), context.correlationKey(),
        context.tenancyId(), feedback, lastDismissal)) {
    return new PolicyDecision(TriggerDecision.SUPPRESS,
        Map.of("suppressionReason", "noise_dismiss_cooldown"));
}
```

`DefaultRasTriggerPolicy` gains `Instance<FeedbackStrategy>` +
`Instance<OutcomeLedger>` constructor dependencies. Optional -- absent means
no suppression. Evaluator already handles SUPPRESS: fires
`SituationChangeEvent.ChangeType.SUPPRESSED`, removes situation, increments
`ras.engine.situations.suppressed` metric.

### 2. Detection Quality Metrics -- FeedbackMetrics

Micrometer gauges, optional via `Instance<MeterRegistry>`:

- `ras.feedback.precision` -- gauge per (situationId, tenancyId)
- `ras.feedback.noise_rate` -- gauge per (situationId, tenancyId)
- `ras.feedback.drift` -- tagged gauge (direction)
- `ras.feedback.outcomes_total` -- counter per (situationId, tenancyId, classification)
- `ras.feedback.suppressions_total` -- counter per (situationId, tenancyId)

Updated periodically by FeedbackUpdateJob.

Per-ganglion attribution (precision/noise rate per individual ganglion within a
situation) is deferred to #59 -- requires capturing the detection result breakdown
in the outcome record. Recall is deferred to #60 -- requires an external
missed-detection signal that RAS cannot compute from its own outcome data.

### 3. Threshold Drift -- SituationDefinitionRegistry overlay

`SituationDefinitionRegistry` gains an `effectiveThreshold` overlay keyed by
`(situationId, tenancyId)`:

```java
private record ThresholdKey(String situationId, String tenancyId) {}
private final ConcurrentHashMap<ThresholdKey, Double> thresholdOverrides = new ConcurrentHashMap<>();

void applyThresholdOverride(String situationId, String tenancyId, double adjustedThreshold) { ... }
Optional<Double> effectiveThreshold(String situationId, String tenancyId) { ... }
```

**Scope:** threshold drift applies only to `ChainMode.Threshold` in this iteration.
`ChainMode.Rate.minRate` is a structurally analogous tunable but is deferred to #57.

**Resolution:** `SituationEvaluator` resolves the effective threshold before passing
the definition to the trigger policy. When an override exists, the evaluator constructs
a new `SituationDefinition` with a modified `ChainMode.Threshold(ganglia, effectiveMinConfidence)`
and passes that to `triggerPolicy.evaluate()`. The policy interface is unchanged — it
sees a definition with the correct threshold already applied. Original definition is
never mutated. Overrides recomputed from `OutcomeStatistics` on restart by FeedbackUpdateJob.

### 4. NaiveBayes Prior Updating

`NaiveBayesGanglion` gains a per-`(situationId, tenancyId)` prior override map.
A single ganglion can serve multiple situations and tenants — different tenants
produce different outcome distributions, so prior adjustments must be per-tenant.
`updatePriors()` accepts raw probabilities and converts to log space before storing:

```java
private record SituationTenantKey(String situationId, String tenancyId) {}
private final ConcurrentHashMap<SituationTenantKey, double[]> feedbackAdjustedLogPriors =
    new ConcurrentHashMap<>();

void updatePriors(String situationId, String tenancyId, double[] rawPriors) {
    feedbackAdjustedLogPriors.put(
        new SituationTenantKey(situationId, tenancyId),
        Arrays.stream(rawPriors).map(Math::log).toArray());
}
```

In `detect()`, new situation instances initialize from adjusted log priors (if
present for this situation+tenant) instead of config log priors:

```java
GanglionState loaded = stateStore.load(key)
    .orElseGet(() -> {
        var priorKey = new SituationTenantKey(context.situationId(), context.tenancyId());
        double[] priors = feedbackAdjustedLogPriors.getOrDefault(priorKey, logPriors);
        return new GanglionState(Arrays.copyOf(priors, priors.length), OptionalLong.empty());
    });
```

`SituationDefinitionRegistry` gains a `rawGanglion(String ganglionId)` method that
returns the unwrapped ganglion (before `EvidenceExtractingGanglion` decoration).
`FeedbackUpdateJob` uses this to reach the concrete `NaiveBayesGanglion` via
`instanceof` without fighting the decorator layer:

```java
Ganglion raw = registry.rawGanglion(ganglionId);
if (raw instanceof NaiveBayesGanglion nbg) {
    nbg.updatePriors(situationId, tenancyId, adjustedPriors);
}
```

`GanglionDescriptor.NaiveBayes` gains optional `outcomeGroundTruth: Map<String, String>`
mapping case outcome labels to NaiveBayes outcome names. FeedbackUpdateJob uses
this to convert OutcomeStatistics into empirical frequencies.

YAML:
```yaml
ganglia:
  - ganglionId: fraud-classifier
    type: naive-bayes
    outcomeGroundTruth:
      escalated: fraud
      dismissed: legitimate
```

### 5. FeedbackUpdateJob

Scheduled job, same pattern as SituationExpiryJob:

```java
@Scheduled(every = "${ras.feedback.update-interval:PT5M}")
void update() {
    for (String situationId : registry.allSituationIds()) {
        SituationRegistration reg = registry.findBySituationId(situationId).orElse(null);
        if (reg == null) continue;
        FeedbackConfig config = reg.definition().feedbackConfig();
        if (config == null) continue;

        for (String tenancyId : ledger.distinctTenancies(situationId)) {
            // 1. FeedbackAnalyzer.analyze(situationId, tenancyId, config) -> FeedbackReport
            // 2. FeedbackStrategy.adjustThreshold(report, ...) -> apply override to registry
            // 3. FeedbackStrategy.adjustPriors(report, ...) -> update NaiveBayesGanglion via rawGanglion()
            // 4. FeedbackMetrics.update(situationId, tenancyId, report) -> refresh gauges
        }
    }
}
```

`SituationDefinitionRegistry.allSituationIds()` exposes the existing
`RegistrySnapshot.situationIds()` set via a new public method.

Suppression is NOT in the job -- checked in real-time by the trigger policy.
Outcome record cleanup is NOT in the job -- owned by `SituationExpiryJob`, consistent
with its role as the single periodic cleanup owner (see §Changes to Existing Types).

## Module Placement

| Component | Module | Pattern |
|-----------|--------|---------|
| `FeedbackConfig`, `OutcomeClassification`, `OutcomeRecord`, `OutcomeStatistics`, `FeedbackReport`, `DriftDirection`, `OutcomeLedger`, `FeedbackStrategy` | `api/` | Domain types + SPIs |
| `OutcomeRecorder`, `FeedbackAnalyzer`, `AutomaticFeedbackStrategy`, `AdvisoryFeedbackStrategy`, `FeedbackUpdateJob`, `FeedbackMetrics` | `runtime/` | Runtime, gains `casehub-engine-api` dep |
| `JpaOutcomeLedger`, `OutcomeRecordEntity`, Flyway migration | `persistence-jpa/` | JPA persistence |
| `InMemoryOutcomeLedger` | `persistence-memory/` | `@Alternative @Priority(100)` |
| `AbstractOutcomeLedgerContractTest` | `api/` test-jar | Contract tests |

## Persistence

### JPA Entity -- OutcomeRecordEntity

```sql
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

`idx_outcome_suppression` is composite with `classification` so the
`lastNoiseDismissalTime()` query filters NOISE in the index scan.

`JpaOutcomeLedger.record()` uses `INSERT ... ON CONFLICT (case_id) DO NOTHING` for
idempotent writes — at-least-once delivery of case outcome events is expected.

### InMemoryOutcomeLedger

`ConcurrentHashMap<String, List<OutcomeRecord>>` keyed by
`situationId + "|" + tenancyId`. `@Alternative @Priority(100)`.

## Changes to Existing Types

| Type | Change |
|------|--------|
| `SituationDefinition` | New `@Nullable FeedbackConfig feedbackConfig` field. New 11-arg constructor. Existing constructors default to null. |
| `DefaultRasTriggerPolicy` | Constructor gains `Instance<FeedbackStrategy>` + `Instance<OutcomeLedger>`. Optional. |
| `NaiveBayesGanglion` | `ConcurrentHashMap<SituationTenantKey, double[]> feedbackAdjustedLogPriors` + `updatePriors(situationId, tenancyId, rawPriors)`. Per-tenant prior overrides. |
| `NaiveBayesConfig` | `@Nullable Map<String, String> outcomeGroundTruth`. |
| `GanglionDescriptor.NaiveBayes` | `outcomeGroundTruth` field. |
| `SituationDefinitionRegistry` | `thresholdOverrides` (keyed by `(situationId, tenancyId)`) + `effectiveThreshold()` + `feedbackConfig()` + `allSituationIds()` + `rawGanglion()`. |
| `SituationEvaluator` | Resolves effective threshold from registry before calling `triggerPolicy.evaluate()`. |
| `SituationExpiryJob` | Calls `OutcomeLedger.removeRecordsBefore()` via `Instance<OutcomeLedger>`. Iterates situations with `FeedbackConfig`, cleans per retention period. |
| `YamlSituationDefinitionProvider` | Parses `feedback:` + `outcomeGroundTruth:`. |

## YAML Schema

```yaml
situations:
  - situationId: fraud-detection
    eventTypes: [payment.flagged]
    chainMode:
      type: threshold
      ganglia: [fraud-classifier, rule-engine]
      minConfidence: 0.7
    triggerAction:
      type: create-case
      caseNamespace: compliance
      caseName: fraud-investigation
      caseVersion: "1.0"
    feedback:
      noiseLabels: [dismissed, false-positive]
      confirmedLabels: [escalated, confirmed-fraud]
      suppressionCooldown: PT6H
      learningRate: 0.1
      retentionPeriod: P90D

ganglia:
  - ganglionId: fraud-classifier
    type: naive-bayes
    outcomes: [fraud, legitimate]
    priors: [0.1, 0.9]
    outcomeGroundTruth:
      escalated: fraud
      confirmed-fraud: fraud
      dismissed: legitimate
      false-positive: legitimate
    # ... features, signalMapping ...
```

## Testing

Contract tests in `api/` test-jar:
- `AbstractOutcomeLedgerContractTest` -- record/statistics/suppression/cleanup/multi-tenant

Integration tests in `runtime/`:
- OutcomeRecorder: outcome -> classification -> storage
- FeedbackAnalyzer: statistics -> drift detection
- AutomaticFeedbackStrategy: suppression, threshold adjustment, prior updating bounds
- FeedbackUpdateJob: end-to-end periodic update cycle
- DefaultRasTriggerPolicy: suppression integration (SUPPRESS decision when within cooldown)
- NaiveBayesGanglion: adjusted priors applied to new situations

## Dependencies

```
runtime/ -> casehub-engine-api (new, for CaseOutcomeObserver/CaseOutcomeEvent)
api/     -> casehub-platform-api (unchanged)
```

## Relationship to parent#365

parent#365 covers engine-layer learning loops (trust scoring, CBR, behavioral
signals). This issue covers detection-layer learning -- what happens before a
case exists. Complementary: engine loops improve case handling, RAS loops improve
case creation. No overlap in scope or types.

## Garden Context

- GE-20260714-439924: Additive confidence penalties at high frequency zero
  confidence. Mitigated by learningRate bound on threshold/prior adjustments --
  maximum adjustment per cycle is `learningRate * delta`, not unbounded.
- GE-20260719-59b809: Feedback timestamp filtering gotcha. `lastNoiseDismissalTime()`
  filters on `closed_at` (when the case was dismissed), which is the correct
  semantics for suppression cooldown -- cooldown starts when the dismiss happened.
