# Decisions — Issue #60: Recall Metric

## D1: Signal source

**Choice:** Operator annotation — human operator marks missed detections via API
**Alternatives:**
- External ground-truth system — higher accuracy but requires integration contract
- Both via unified SPI — over-engineers for the initial use case
**Rationale:** Lowest integration cost, simplest ingestion model. Recall accuracy depends on operator discipline but that's acceptable for an initial implementation.
**Trade-offs:** Recall metric quality depends entirely on operator reporting discipline
**Sources:** Issue #60, feedback loop design spec (#40)
**Exploration:** quick
**Status:** captured

## D2: Signal granularity

**Choice:** Situation-level first — operator specifies situationId + correlationKey + tenancyId + time window
**Alternatives:**
- All three granularities from day one (situation + ganglion + event) — more API surface to stabilise
- Event-level first (submit CloudEvent for replay) — heaviest, requires replay infrastructure
**Rationale:** Maps directly to existing OutcomeLedger tuple. Ganglion-level and event-level can be added later as enrichments on the same signal record.
**Trade-offs:** Cannot compute per-ganglion recall until ganglion-level signals are added
**Sources:** OutcomeLedger.java, OutcomeRecord.java
**Exploration:** quick
**Status:** captured

## D3: Ingestion architecture

**Choice:** New MissedDetectionLedger SPI in api/ + CDI event MissedDetectionEvent
**Alternatives:**
- Direct recordMissed() on OutcomeLedger — mixes case outcomes with external signals
- REST endpoint only — skips CDI eventing, doesn't fit SPI pattern
**Rationale:** Mirrors OutcomeLedger pattern. CDI observer records to ledger. External adapters fire CDI event. Consistent with existing architecture.
**Trade-offs:** New SPI + event type + observer — more surface area than extending OutcomeLedger
**Depends on:** D2 (signal shape determines ledger record fields)
**Sources:** OutcomeRecorder.java, CaseOutcomeEvent pattern
**Exploration:** quick
**Status:** captured

## D4: Tuning interaction

**Choice:** Metric only, no auto-tuning — surface recall gauge + UNDER_SENSITIVE drift direction
**Alternatives:**
- Symmetric auto-tuning (low recall → lower thresholds) — dangerous with unreliable signal
- Advisory with suppression override — lighter but still automated
**Rationale:** Auto-lowering thresholds based on operator reports is risky. Operators observe recall gauge and tune manually. DefaultTuningStrategy is a no-op for recall.
**Trade-offs:** No automated response to under-sensitivity — purely observational
**Sources:** DefaultTuningStrategy.java, FeedbackAnalyzer.java
**Exploration:** quick
**Status:** captured

## D5: Persistence model

**Choice:** Same pattern as OutcomeLedger — separate SPI, InMemory @DefaultBean, JPA impl with Flyway migration
**Alternatives:**
- Extend OutcomeLedger table with 'missed' boolean — simpler schema but semantically different
**Rationale:** Missed detections don't have case outcomes. Separate ledger keeps concerns clean. Consumers get persistence by adding persistence-jpa to classpath.
**Trade-offs:** New Flyway migration, new entity, new table — more schema surface
**Depends on:** D3 (SPI design determines what the persistence layer stores)
**Sources:** JpaOutcomeLedger.java, OutcomeLedger SPI, Flyway V1-V8 pattern
**Exploration:** quick
**Status:** captured
