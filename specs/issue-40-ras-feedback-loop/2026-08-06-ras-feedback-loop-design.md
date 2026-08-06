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
learningRate in (0.0, 1.0]. retentionPeriod > 0.

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
    int removeRecordsBefore(Instant cutoff);
}
```

`statistics()` is on the SPI so JPA pushes aggregation to SQL.
`lastNoiseDismissalTime()` is a dedicated query for fast suppression checks.

### FeedbackStrategy (SPI)

```java
interface FeedbackStrategy {
    boolean shouldSuppress(String situationId, String correlationKey, String tenancyId,
                           FeedbackConfig config, OutcomeLedger ledger);

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

        OutcomeClassification classification = config.classify(event.outcomeLabel());

        ledger.record(new OutcomeRecord(
            situationId,
            (String) event.caseFileSnapshot().get("correlationKey"),
            event.tenancyId(),
            event.outcomeLabel(),
            classification,
            event.closedAt(),
            event.caseId()
        ));
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
        if (stats.noiseRate() > 0.5) return DriftDirection.OVER_SENSITIVE;
        if (stats.noiseRate() < 0.1 && stats.confirmedCount() > 10)
            return DriftDirection.STABLE;
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

- **shouldSuppress**: queries `ledger.lastNoiseDismissalTime()`, returns true if
  within `config.suppressionCooldown()`
- **adjustThreshold**: if OVER_SENSITIVE, increases threshold by
  `learningRate * (noiseRate - 0.5)`. If UNDER_SENSITIVE, decreases similarly.
  Result clamped to [0.0, 1.0]. Bounded by learningRate to prevent the confidence
  zeroing gotcha (GE-20260714-439924 -- additive penalties at high frequency).
- **adjustPriors**: blends old priors toward empirical outcome distribution:
  `newPrior = (1 - learningRate) * oldPrior + learningRate * empiricalFreq`.
  Renormalized. Only when `totalOutcomes >= 5` (INSUFFICIENT_DATA guard).

## Mechanism Integration

### 1. Dismiss Suppression -- DefaultRasTriggerPolicy

Policy checks suppression BEFORE returning a trigger decision:

```java
if (feedback != null && feedbackStrategy.shouldSuppress(
        definition.situationId(), context.correlationKey(),
        context.tenancyId(), feedback, outcomeLedger)) {
    return new PolicyDecision(TriggerDecision.SUPPRESS,
        Map.of("suppressionReason", "noise_dismiss_cooldown"));
}
```

`DefaultRasTriggerPolicy` gains `Instance<FeedbackStrategy>` +
`Instance<OutcomeLedger>` constructor dependencies. Optional -- absent means
no suppression. Evaluator already handles SUPPRESS: fires
`SituationChangeEvent.ChangeType.SUPPRESSED`, removes situation, increments
`ras.situation.suppressed` metric.

### 2. Detection Quality Metrics -- FeedbackMetrics

Micrometer gauges, optional via `Instance<MeterRegistry>`:

- `ras.feedback.precision` -- gauge per (situationId, tenancyId)
- `ras.feedback.noise_rate` -- gauge per (situationId, tenancyId)
- `ras.feedback.drift` -- tagged gauge (direction)
- `ras.feedback.outcomes_total` -- counter per (situationId, tenancyId, classification)
- `ras.feedback.suppressions_total` -- counter per (situationId, tenancyId)

Updated periodically by FeedbackUpdateJob.

### 3. Threshold Drift -- SituationDefinitionRegistry overlay

`SituationDefinitionRegistry` gains an `effectiveThreshold` overlay:

```java
private final ConcurrentHashMap<String, Double> thresholdOverrides = new ConcurrentHashMap<>();

void applyThresholdOverride(String situationId, double adjustedThreshold) { ... }
Optional<Double> effectiveThreshold(String situationId) { ... }
```

`DefaultRasTriggerPolicy.evaluateThreshold()` checks the registry for an override
before using the definition's `minConfidence`. Original definition is never mutated.
Overrides recomputed from `OutcomeStatistics` on restart by FeedbackUpdateJob.

### 4. NaiveBayes Prior Updating

`NaiveBayesGanglion` gains a `volatile double[] feedbackAdjustedPriors` field +
`updatePriors(double[])` method. In `detect()`, new situation instances initialize
from adjusted priors (if present) instead of config priors:

```java
GanglionState loaded = stateStore.load(key)
    .orElseGet(() -> {
        double[] priors = feedbackAdjustedPriors != null
            ? feedbackAdjustedPriors
            : Arrays.copyOf(logPriors, logPriors.length);
        return new GanglionState(priors, OptionalLong.empty());
    });
```

`GanglionDescriptor.NaiveBayes` gains optional `outcomeGroundTruth: Map<String, String>`
mapping case outcome labels to NaiveBayes outcome names. FeedbackUpdateJob uses
this to convert OutcomeStatistics into empirical frequencies.

YAML:
```yaml
ganglia:
  - id: fraud-classifier
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
    // For each situation with feedbackConfig:
    // 1. FeedbackAnalyzer.analyze() -> FeedbackReport
    // 2. FeedbackStrategy.adjustThreshold() -> apply override to registry
    // 3. FeedbackStrategy.adjustPriors() -> update NaiveBayesGanglion priors
    // 4. FeedbackMetrics.update() -> refresh gauges
    // 5. OutcomeLedger.removeRecordsBefore() -> cleanup expired records
}
```

Suppression is NOT in the job -- checked in real-time by the trigger policy.

## Module Placement

| Component | Module | Pattern |
|-----------|--------|---------|
| `FeedbackConfig`, `OutcomeClassification`, `OutcomeRecord`, `OutcomeStatistics`, `FeedbackReport`, `DriftDirection`, `OutcomeLedger`, `FeedbackStrategy` | `api/` | Domain types + SPIs |
| `OutcomeRecorder`, `FeedbackAnalyzer`, `AutomaticFeedbackStrategy`, `FeedbackUpdateJob`, `FeedbackMetrics` | `runtime/` | Runtime, gains `casehub-engine-api` dep |
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
    case_id         UUID         NOT NULL
);

CREATE INDEX idx_outcome_situation_tenant
    ON ras_outcome_record (situation_id, tenancy_id, closed_at);

CREATE INDEX idx_outcome_suppression
    ON ras_outcome_record (situation_id, correlation_key, tenancy_id,
                           classification, closed_at DESC);
```

`idx_outcome_suppression` is composite with `classification` so the
`lastNoiseDismissalTime()` query filters NOISE in the index scan.

### InMemoryOutcomeLedger

`ConcurrentHashMap<String, List<OutcomeRecord>>` keyed by
`situationId + "|" + tenancyId`. `@Alternative @Priority(100)`.

## Changes to Existing Types

| Type | Change |
|------|--------|
| `SituationDefinition` | New `@Nullable FeedbackConfig feedbackConfig` field. New 11-arg constructor. Existing constructors default to null. |
| `DefaultRasTriggerPolicy` | Constructor gains `Instance<FeedbackStrategy>` + `Instance<OutcomeLedger>`. Optional. |
| `NaiveBayesGanglion` | `volatile double[] feedbackAdjustedPriors` + `updatePriors()`. |
| `NaiveBayesConfig` | `@Nullable Map<String, String> outcomeGroundTruth`. |
| `GanglionDescriptor.NaiveBayes` | `outcomeGroundTruth` field. |
| `SituationDefinitionRegistry` | `thresholdOverrides` + `effectiveThreshold()` + `feedbackConfig()`. |
| `SituationExpiryJob` | Calls `OutcomeLedger.removeRecordsBefore()` via `Instance<OutcomeLedger>`. |
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
  - id: fraud-classifier
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
