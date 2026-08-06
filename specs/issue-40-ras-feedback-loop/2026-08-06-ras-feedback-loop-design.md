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
   noise rate) from outcome records within the configured retention window
3. **Application** -- Split SPIs: `SuppressionStrategy` (real-time suppress-or-fire
   per event) and `FeedbackTuningStrategy` (batch threshold/prior adjustment).
   Default implementations handle both; advisory mode provides suppression only.

## Key Design Decisions

- **Feedback config on SituationDefinition** -- per-situation opt-in via
  `FeedbackConfig` field. Absent = no feedback. Outcome label classification
  (noise/confirmed) is intrinsic to the situation, not operational overlay.
- **Split strategy SPIs** -- `SuppressionStrategy` and `FeedbackTuningStrategy`
  in api/. Separated by lifecycle: suppression is per-event on the hot path;
  tuning is batch every 5 minutes. Default implementations apply all mechanisms
  within bounds. Advisory mode (`tuningEnabled: false`, the default) provides
  suppression and metrics only — no threshold or prior adjustment.
- **Suppression in SituationEvaluator** -- suppression is a pre-detection concern,
  not a policy decision. Checked before running ganglia or policy evaluation.
  `DefaultRasTriggerPolicy` stays pure with zero dependencies.
- **FeedbackState component** -- mutable feedback-derived state (threshold
  overrides, prior adjustments) lives in a dedicated `FeedbackState` bean in
  runtime/, not on `SituationDefinitionRegistry` (immutable-snapshot discipline)
  or `NaiveBayesGanglion` (no mutable internal state). All overrides are
  tenant-scoped: keyed by `(id, tenancyId)`.
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
    Duration retentionPeriod,        // how long to keep outcome records
    boolean tuningEnabled            // false = advisory mode (suppression + metrics only)
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
tuningEnabled defaults to false (advisory mode).

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

### OutcomeLedger (SPI)

```java
interface OutcomeLedger {
    void record(OutcomeRecord record);
    OutcomeStatistics statistics(String situationId, String tenancyId, Instant since);
    Optional<Instant> lastNoiseDismissalTime(
        String situationId, String correlationKey, String tenancyId);
    Map<String, Long> countByLabel(String situationId, String tenancyId, Instant since);
    Set<String> distinctTenancies(String situationId);
    int removeRecordsBefore(String situationId, Instant cutoff);
}
```

`statistics()` is on the SPI so JPA pushes aggregation to SQL.
`lastNoiseDismissalTime()` is a dedicated query for fast suppression checks.
`countByLabel()` returns per-outcome-label counts (`GROUP BY outcome_label`) --
used by `FeedbackUpdateJob` for NaiveBayes prior adjustment via `outcomeGroundTruth`.
Separate from `statistics()` so the threshold-only path avoids per-label aggregation.
`distinctTenancies()` returns `SELECT DISTINCT tenancy_id WHERE situation_id = ?` --
used by `FeedbackUpdateJob` to discover the tenant iteration scope.
`removeRecordsBefore()` is per-situation so retention periods are honoured individually.

### SuppressionStrategy (SPI)

```java
interface SuppressionStrategy {
    boolean shouldSuppress(String situationId, String correlationKey, String tenancyId,
                           FeedbackConfig config, Optional<Instant> lastNoiseDismissalTime);
}
```

Per-event, on the hot path. Receives pre-queried data from the caller (the
evaluator queries the ledger), not the raw ledger. This makes the contract
explicit: suppression depends on the last noise dismissal time, nothing else.
Strategies are testable without mocking a data store.

### FeedbackTuningStrategy (SPI)

```java
interface FeedbackTuningStrategy {
    OptionalDouble adjustThreshold(OutcomeStatistics statistics, double currentThreshold,
                                    FeedbackConfig config);

    Optional<double[]> adjustPriors(double[] currentPriors, long[] outcomeCounts,
                                     FeedbackConfig config);
}
```

Batch, called by `FeedbackUpdateJob` every 5 minutes.

`adjustThreshold` receives `OutcomeStatistics` directly -- the strategy makes its
own interpretive decisions about drift classification.

`adjustPriors` receives pre-mapped per-NaiveBayes-outcome counts (computed by the
job from `countByLabel()` + `outcomeGroundTruth`), not raw statistics. The mapping
from case outcome labels to NaiveBayes outcomes is a ganglion concern, not a
strategy concern -- the strategy only decides how to blend priors toward empirical
frequencies.

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

Recording is synchronous on the engine's case-completion thread. A single
`INSERT ... ON CONFLICT DO NOTHING` is sub-millisecond for PostgreSQL; the
try/catch prevents error propagation. Async recording would add ordering,
error handling, and lifecycle complexity for no demonstrated benefit at
expected volumes.

ASSUMPTION: `CaseOutcomeObserver.onOutcome()` is invoked synchronously by the
engine during case completion. If the engine invokes it asynchronously (e.g.,
via an event bus), the latency concern does not apply.

## Analysis Layer (runtime/)

### FeedbackAnalyzer

```java
@ApplicationScoped
class FeedbackAnalyzer {
    @Inject OutcomeLedger ledger;

    OutcomeStatistics analyze(String situationId, String tenancyId, FeedbackConfig config) {
        return ledger.statistics(
            situationId, tenancyId,
            Instant.now().minus(config.retentionPeriod()));
    }
}
```

The analyzer applies the retention window from `FeedbackConfig` and returns raw
`OutcomeStatistics`. No interpretive classification (drift direction, sensitivity
thresholds) -- that responsibility belongs to the `FeedbackTuningStrategy`, which
decides what the statistics mean and how to act on them.

## Application Layer (runtime/)

### DefaultSuppressionStrategy

`@DefaultBean` in runtime/:

- **shouldSuppress**: returns true if `lastNoiseDismissalTime` is present and
  within `config.suppressionCooldown()` of current time

### DefaultTuningStrategy

`@DefaultBean` in runtime/:

- **adjustThreshold**: if `statistics.noiseRate() > 0.5` (over-sensitive),
  increases threshold by `learningRate * (noiseRate - 0.5)`. Result clamped
  to (0.0, 1.0] -- lower bound matches `ChainMode.Threshold`'s `minConfidence > 0.0`
  validation. Bounded by learningRate to prevent the confidence zeroing
  gotcha (GE-20260714-439924 -- additive penalties at high frequency).
  Returns `OptionalDouble.empty()` when `totalOutcomes < 10` (insufficient data).
- **adjustPriors**: receives `long[] outcomeCounts` (per-NaiveBayes-outcome counts,
  pre-mapped by the job from `countByLabel` + `outcomeGroundTruth`). Applies Laplace
  smoothing (add-one pseudocount per outcome) to prevent zero-frequency outcomes:
  `empiricalFreq[i] = (outcomeCounts[i] + 1) / (total + numOutcomes)`.
  Blends: `newPrior = (1 - learningRate) * oldPrior + learningRate * empiricalFreq`.
  Renormalized. Returns `Optional.empty()` when total counts < 5. This ensures no
  outcome can reach zero probability, which would produce `-Infinity` in log space
  and permanently disable that outcome class (same invariant as `NaiveBayesConfig`'s
  `priors[i] > 0.0` validation). `FeedbackState.applyPriorOverride()` also validates
  as defense-in-depth against custom strategy implementations that omit smoothing.

Advisory mode: set `tuningEnabled: false` in `FeedbackConfig` (the default).
`FeedbackUpdateJob` checks `config.tuningEnabled()` and skips threshold/prior
adjustment when disabled. Suppression and metrics are still active regardless.
`DefaultTuningStrategy` remains `@DefaultBean` — its availability is not the
activation mechanism. Tuning requires explicit per-situation opt-in via
`tuningEnabled: true`.

## Mechanism Integration

### 1. Dismiss Suppression -- SituationEvaluator

`SituationEvaluator` checks suppression BEFORE loading context or running
detection, saving wasted ganglion compute for suppressed events:

```java
FeedbackConfig feedbackConfig = definition.feedbackConfig();
if (feedbackConfig != null
        && suppressionStrategy.isResolvable()
        && outcomeLedger.isResolvable()) {
    Optional<Instant> lastDismissal = outcomeLedger.get().lastNoiseDismissalTime(
        situationId, correlationKey, tenancyId);
    if (suppressionStrategy.get().shouldSuppress(
            situationId, correlationKey, tenancyId, feedbackConfig, lastDismissal)) {
        metrics.feedbackSuppression(situationId, tenancyId);
        return true;
    }
}
```

`DefaultRasTriggerPolicy` is unchanged -- zero dependencies, pure function of
`(SituationContext, SituationDefinition)`. The `RasTriggerPolicy` SPI contract
(all decision inputs in parameters) is preserved.

`SituationEvaluator` gains `Instance<SuppressionStrategy>` +
`Instance<OutcomeLedger>` + `Instance<FeedbackState>` constructor dependencies.
All optional via `Instance<>` -- absent means no feedback.

### 2. Detection Quality Metrics -- FeedbackMetrics

Micrometer gauges, optional via `Instance<MeterRegistry>`:

- `ras.feedback.precision` -- gauge per (situationId, tenancyId)
- `ras.feedback.noise_rate` -- gauge per (situationId, tenancyId)
- `ras.feedback.outcomes_total` -- counter per (situationId, tenancyId, classification)
- `ras.feedback.suppressions_total` -- counter per (situationId, tenancyId)

Updated periodically by FeedbackUpdateJob.

**Recall is deferred.** Recall requires a false-negative signal -- situations that
should have been detected but weren't -- which RAS cannot compute from its own outcome
data. Deferred to #58.

### 3. Threshold Drift + Prior Overrides -- FeedbackState

New `@ApplicationScoped` component in `runtime/` centralising all
feedback-derived mutable state with tenant scoping:

```java
@ApplicationScoped
class FeedbackState {
    private final ConcurrentHashMap<StateKey, Double> thresholdOverrides = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<StateKey, double[]> priorOverrides = new ConcurrentHashMap<>();

    private record StateKey(String id, String tenancyId) {}

    OptionalDouble effectiveThreshold(String situationId, String tenancyId) { ... }
    Optional<double[]> adjustedLogPriors(String ganglionId, String tenancyId) { ... }
    double[] currentRawPriors(String ganglionId, String tenancyId, double[] basePriors) {
        return adjustedLogPriors(ganglionId, tenancyId)
            .map(logP -> normalizeLogToRaw(logP))
            .orElse(basePriors);
    }
    void applyThresholdOverride(String situationId, String tenancyId, double threshold) {
        if (Double.isNaN(threshold) || threshold <= 0.0 || threshold > 1.0) {
            LOG.warning("Rejecting feedback threshold for situation '" + situationId
                + "' tenant '" + tenancyId + "': " + threshold
                + " — must be in (0.0, 1.0]");
            return;
        }
        thresholdOverrides.put(new StateKey(situationId, tenancyId), threshold);
    }

    void applyPriorOverride(String ganglionId, String tenancyId, double[] rawPriors) {
        for (int i = 0; i < rawPriors.length; i++) {
            if (Double.isNaN(rawPriors[i]) || rawPriors[i] <= 0.0) {
                LOG.warning("Rejecting feedback priors for ganglion '" + ganglionId
                    + "' tenant '" + tenancyId + "': prior[" + i + "] = " + rawPriors[i]
                    + " — zero/negative priors make outcomes permanently impossible");
                return;
            }
        }
        priorOverrides.put(new StateKey(ganglionId, tenancyId),
            Arrays.stream(rawPriors).map(Math::log).toArray());
    }
}
```

**Threshold override application:** `SituationEvaluator` queries
`FeedbackState.effectiveThreshold()` and constructs an effective
`SituationDefinition` before calling `triggerPolicy.evaluate()`. The policy
sees the adjusted `minConfidence` as normal immutable input:

```java
SituationDefinition effectiveDef = definition;
if (feedbackState.isResolvable()
        && definition.chainMode() instanceof ChainMode.Threshold threshold) {
    OptionalDouble adjusted = feedbackState.get()
        .effectiveThreshold(situationId, tenancyId);
    if (adjusted.isPresent()) {
        effectiveDef = definition.withChainMode(
            new ChainMode.Threshold(threshold.ganglia(), adjusted.getAsDouble()));
    }
}
PolicyDecision policyDecision = triggerPolicy.evaluate(context, effectiveDef);
```

`SituationDefinition` gains a `withChainMode(ChainMode)` method for this
construction. Original definition is never mutated.

**Scope:** threshold drift applies only to `ChainMode.Threshold` in this iteration.
`ChainMode.Rate.minRate` is a structurally analogous tunable but is deferred to #57.

**Why not on SituationDefinitionRegistry:** The registry uses a `volatile
RegistrySnapshot` pattern -- immutable snapshots, swap-on-write. Adding
`ConcurrentHashMap` mutable state that changes every 5 minutes violates this
discipline. The registry is a registration/lookup service; `FeedbackState`
holds runtime feedback state. Separate concerns, separate components.

**Tenant scoping:** Both maps key on `(id, tenancyId)`. If Tenant A has a
90% noise rate and Tenant B has 5%, their overrides are independent. No
cross-tenant contamination. Overrides recomputed from `OutcomeStatistics` on
restart by `FeedbackUpdateJob`.

### 4. NaiveBayes Prior Updating

`NaiveBayesGanglion` gains an optional `FeedbackState` constructor parameter
(passed through by `SituationDefinitionRegistry.constructNaiveBayes()`). In
`detect()`, new situation instances query `FeedbackState` for tenant-scoped
adjusted priors:

```java
GanglionState loaded = stateStore.load(key)
    .orElseGet(() -> {
        double[] priors = feedbackState != null
            ? feedbackState.adjustedLogPriors(config.ganglionId(), context.tenancyId())
                .orElse(Arrays.copyOf(logPriors, logPriors.length))
            : Arrays.copyOf(logPriors, logPriors.length);
        return new GanglionState(priors, OptionalLong.empty());
    });
```

No mutable internal state on the ganglion. The ganglion delegates per-instance
state to `GanglionStateStore` and queries `FeedbackState` for per-tenant
feedback-adjusted initial priors. This preserves the ganglion's existing state
discipline.

`GanglionDescriptor.NaiveBayes` gains optional `outcomeGroundTruth: Map<String, String>`
mapping case outcome labels to NaiveBayes outcome names. FeedbackUpdateJob uses
this to convert OutcomeStatistics into empirical frequencies. Validated at
construction time: every value must exist in the ganglion's `outcomes` list (same
pattern as `outcomeEvidenceTemplates` key validation in `NaiveBayesConfig`). A typo
like `escalated: froud` instead of `escalated: fraud` fails loudly at startup rather
than silently producing skewed priors.

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
            Instant windowStart = Instant.now().minus(config.retentionPeriod());
            OutcomeStatistics stats = analyzer.analyze(situationId, tenancyId, config);

            if (config.tuningEnabled()) {
                // Threshold adjustment (only for ChainMode.Threshold situations)
                if (reg.definition().chainMode() instanceof ChainMode.Threshold threshold) {
                    double currentThreshold = feedbackState.effectiveThreshold(situationId, tenancyId)
                        .orElse(threshold.minConfidence());
                    tuningStrategy.adjustThreshold(stats, currentThreshold, config)
                        .ifPresent(t -> feedbackState.applyThresholdOverride(situationId, tenancyId, t));
                }

                // Prior adjustment (per NaiveBayes ganglion in situation)
                for (String ganglionId : reg.definition().chainMode().referencedGanglia()) {
                    GanglionDescriptor desc = registry.ganglionDescriptor(ganglionId);
                    if (!(desc instanceof GanglionDescriptor.NaiveBayes nb)) continue;
                    if (nb.outcomeGroundTruth() == null) continue;

                    // Map per-label counts to per-outcome counts via outcomeGroundTruth
                    Map<String, Long> labelCounts =
                        ledger.countByLabel(situationId, tenancyId, windowStart);
                    long[] outcomeCounts = new long[nb.outcomes().size()];
                    for (var entry : labelCounts.entrySet()) {
                        String outcomeName = nb.outcomeGroundTruth().get(entry.getKey());
                        if (outcomeName != null) {
                            outcomeCounts[nb.outcomes().indexOf(outcomeName)] += entry.getValue();
                        }
                    }

                    double[] currentPriors = feedbackState.currentRawPriors(
                        ganglionId, tenancyId, nb.priors());
                    tuningStrategy.adjustPriors(currentPriors, outcomeCounts, config)
                        .ifPresent(p -> feedbackState.applyPriorOverride(ganglionId, tenancyId, p));
                }
            }

            feedbackMetrics.update(situationId, tenancyId, stats);
        }

        // 4. OutcomeLedger.removeRecordsBefore(situationId, retentionCutoff) -> cleanup
    }
}
```

`SituationDefinitionRegistry.allSituationIds()` exposes the existing
`RegistrySnapshot.situationIds()` set via a new public method.

Suppression is NOT in the job -- checked in real-time by `SituationEvaluator`.
Cleanup is owned solely by this job -- outcome records are feedback data and
their retention period is defined by `FeedbackConfig`. `SituationExpiryJob` is
not modified.

## Module Placement

| Component | Module | Pattern |
|-----------|--------|---------|
| `FeedbackConfig`, `OutcomeClassification`, `OutcomeRecord`, `OutcomeStatistics`, `OutcomeLedger`, `SuppressionStrategy`, `FeedbackTuningStrategy` | `api/` | Domain types + SPIs |
| `OutcomeRecorder`, `FeedbackAnalyzer`, `DefaultSuppressionStrategy`, `DefaultTuningStrategy`, `FeedbackUpdateJob`, `FeedbackMetrics`, `FeedbackState` | `runtime/` | Runtime, gains `casehub-engine-api` dep |
| `JpaOutcomeLedger`, `OutcomeRecordEntity`, Flyway migration | `persistence-jpa/` | JPA persistence |
| `InMemoryOutcomeLedger` | `runtime/` | `@DefaultBean` (same pattern as `InMemoryGanglionStateStore`) |
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
idempotent writes -- at-least-once delivery of case outcome events is expected.

### InMemoryOutcomeLedger

`ConcurrentHashMap<String, List<OutcomeRecord>>` keyed by
`situationId + "|" + tenancyId`. `@DefaultBean` in `runtime/` (same pattern as
`InMemoryGanglionStateStore` -- yields to JPA when `persistence-jpa/` is on
the classpath, provides a fallback when no persistence module is deployed).

## Changes to Existing Types

| Type | Change |
|------|--------|
| `SituationDefinition` | New `@Nullable FeedbackConfig feedbackConfig` field. New 11-arg constructor. Existing constructors default to null. New `withChainMode(ChainMode)` method for threshold override construction. |
| `SituationEvaluator` | Constructor gains `Instance<SuppressionStrategy>` + `Instance<OutcomeLedger>` + `Instance<FeedbackState>`. Pre-detection suppression check. Effective threshold construction before policy evaluation. |
| `NaiveBayesGanglion` | Constructor gains optional `FeedbackState` parameter. Queries for tenant-scoped priors on new instance creation. No mutable internal state added. |
| `NaiveBayesConfig` | `@Nullable Map<String, String> outcomeGroundTruth`. |
| `GanglionDescriptor.NaiveBayes` | `outcomeGroundTruth` field. |
| `SituationDefinitionRegistry` | Constructor gains `Instance<FeedbackState>` (passed through to ganglion construction). New `feedbackConfig()`, `allSituationIds()`, and `ganglionDescriptor(String)` methods. Phase 1 gains a parallel `descriptorsById` map populated alongside `gangliaById` during `constructGanglion()`. CDI ganglia (Phase 2) have no descriptor — `ganglionDescriptor()` returns null for them; callers guard with `instanceof GanglionDescriptor.NaiveBayes`. |
| `YamlSituationDefinitionProvider` | Parses `feedback:` + `outcomeGroundTruth:`. |

Note: `DefaultRasTriggerPolicy` and `SituationExpiryJob` are NOT modified.

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
      tuningEnabled: true

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
- FeedbackAnalyzer: statistics windowing
- DefaultSuppressionStrategy: cooldown-based suppression with pre-queried last dismissal time
- DefaultTuningStrategy: threshold adjustment, prior updating bounds
- FeedbackUpdateJob: end-to-end periodic update cycle with tenant isolation
- FeedbackState: tenant-scoped threshold and prior overrides, concurrent access
- SituationEvaluator: pre-detection suppression, effective threshold construction
- NaiveBayesGanglion: tenant-scoped adjusted priors from FeedbackState

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
