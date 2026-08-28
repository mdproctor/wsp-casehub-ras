# Decisions — Issue #60: Recall Metric

## D1: Signal source

**Choice:** Operator annotation via REST as the first input mechanism; future signal sources (external ground truth, periodic audit) add additional integrations calling the same `OutcomeLedger.recordMissed()` contract
**Alternatives:**
- Operator annotation only (original) — hardcodes input channel, requires refactoring when any other source arrives; issue #58 already identifies three possible sources
- Dedicated ingestion SPI (MissedDetectionSource in api/) — unnecessary abstraction layer; `OutcomeLedger.recordMissed()` already serves as the pluggable contract point; input shape and validation differ per source, so a single ingestion SPI would be either too generic or too narrow
- External ground-truth system first — higher accuracy but requires integration contract before any metric is available
**Rationale:** Operator annotation is the most accessible signal source and requires no external system integration. The extension point for future sources is `OutcomeLedger.recordMissed()` — any adapter (REST endpoint, batch import, streaming integration) can call it directly. No separate ingestion SPI is needed because the existing platform SPIs (`OutcomeLedger`, `SuppressionStrategy`, `FeedbackTuningStrategy`) operate at the storage/strategy layer, not the input layer. The existing input path (`CaseOutcomeObserver` → `OutcomeRecorder` → `OutcomeLedger`) has no RAS-side ingestion SPI either — `CaseOutcomeObserver` is an engine SPI in `casehub-engine-api`.
**Trade-offs:** Recall metric quality depends on operator reporting discipline and is epistemologically limited — operators can only report misses they independently notice, so systematic blind spots (entire event patterns the system never processes) remain invisible. This is inherent to any single-source external signal, not unique to operator annotation. Adding future sources (ground truth, periodic audit) mitigates but never eliminates this bias.
**Sources:** Issue #60, issue #58 (three signal sources identified), feedback loop design spec (#40), CaseOutcomeObserver SPI pattern
**Exploration:** quick
**Status:** revised (R1-01: from operator-only to acknowledging future sources; extension point is OutcomeLedger.recordMissed(), not a separate ingestion SPI)

## D2: Signal granularity

**Choice:** Situation-level first — missed detection signal specifies situationId + correlationKey + tenancyId + eventTime
**Alternatives:**
- All three granularities from day one (situation + ganglion + event) — more API surface to stabilise
- Event-level first (submit CloudEvent for replay) — heaviest, requires replay infrastructure
**Rationale:** situationId and tenancyId are deployment-level identifiers operators already know. correlationKey is derived from domain attributes (e.g., account ID, device ID) via CloudEvent attribute mapping — in most deployments, the correlation key IS a domain identifier that operators independently know. When an operator reports "fraud was missed for account ACC-12345," ACC-12345 is the correlation key. The signal does NOT map to OutcomeRecord's tuple — missed detections have no caseId, outcomeLabel, or classification. The fields shared with OutcomeRecord (situationId, correlationKey, tenancyId) reflect common domain anchoring, not structural equivalence.
**Trade-offs:** Cannot compute per-ganglion recall until ganglion-level signals are added. Operators whose situations use compound or derived correlation keys (e.g., hash of multiple event attributes) need to understand the key format — the REST API should document the correlation key schema per situation definition.
**Sources:** OutcomeRecord.java, SituationDefinition.java, CloudEvent correlation key extraction
**Exploration:** quick
**Status:** revised (R1-04: removed false OutcomeLedger tuple mapping claim; clarified correlationKey is domain-level. R2-02: renamed "time window" to eventTime — the operator specifies when the missed event occurred, not a range)

## D3: Ingestion architecture

**Choice:** Extend `OutcomeLedger` with `recordMissed(MissedDetectionRecord)` — unified store, single query interface for computing both precision and recall
**Alternatives:**
- Parallel MissedDetectionLedger SPI + CDI event (original) — creates cross-store join problem for recall computation: TP from OutcomeLedger + FN from MissedDetectionLedger requires aligned time windows across two stores; more surface area (new SPI, event type, observer, persistence layer) for no architectural benefit
- Extend OutcomeRecord with source discriminator and nullable caseId — breaks OutcomeRecord's non-null contract (`Objects.requireNonNull(caseId)`); requires sentinel values; existing dedup on caseId (UNIQUE constraint, `seenCaseIds` set in InMemoryOutcomeLedger) doesn't apply to missed detections; forces every OutcomeRecord consumer to handle the nullable case
- REST endpoint only — correct as the first external API, but skips the internal SPI extension that makes the store queryable
**Rationale:** Precision and recall are two projections of the same confusion matrix. They should be computed from a single data source within a single time window. `OutcomeLedger` gains `recordMissed(MissedDetectionRecord)` for storage and `statistics()` returns an extended `OutcomeStatistics` including `missedCount`. `MissedDetectionRecord` is a separate record type with fields: `situationId`, `correlationKey`, `tenancyId`, `eventTime` (when the missed event occurred — operator-specified), `reportedBy` (operator identity for audit), `UUID reportId` (dedup key), `Instant recordedAt` (system-generated timestamp of when the report was filed). Stored in the same ledger, queryable through the same statistics interface. `OutcomeRecord` is unchanged: its non-null contract, dedup semantics, and all existing consumers are preserved.
**Depends on:** D2 (signal granularity determines MissedDetectionRecord fields)
**Sources:** OutcomeLedger.java, OutcomeRecord.java, QualityMetrics.java, InMemoryOutcomeLedger.java (caseId dedup), JpaOutcomeLedger.java (ON CONFLICT case_id)
**Exploration:** quick
**Status:** revised (R1-02: unified store eliminates cross-store join; separate MissedDetectionRecord preserves OutcomeRecord contract. R2-02: fields clarified — eventTime replaces reportedAt, recordedAt added for audit)

## D4: Tuning interaction

**Choice:** Metric only, no auto-tuning — surface recall as `ras.feedback.recall` gauge; operators observe and tune manually
**Alternatives:**
- Symmetric auto-tuning (low recall → lower thresholds) — dangerous with unreliable external signal
- Advisory with suppression override — lighter but still automated
**Rationale:** Auto-lowering thresholds based on operator reports is risky — the epistemological bias in operator reporting (D1) means the signal is unreliable. Lowering thresholds based on incomplete data could increase false positives dramatically. Operators observe the recall gauge and tune manually. `FeedbackTuningStrategy` is a no-op for recall.
**Trade-offs:** No automated response to under-sensitivity — purely observational
**Sources:** DefaultTuningStrategy.java, FeedbackAnalyzer.java
**Exploration:** quick
**Status:** revised (R1-03: removed phantom DriftDirection.UNDER_SENSITIVE reference — see D8)

## D5: Persistence model

**Choice:** No new persistence layer — `OutcomeLedger` persistence implementations (`JpaOutcomeLedger`, `InMemoryOutcomeLedger`) are extended to store `MissedDetectionRecord` alongside `OutcomeRecord`
**Alternatives:**
- Separate MissedDetectionLedger persistence (original) — superseded by D3 revision; would duplicate the persistence stack (SPI, InMemory @DefaultBean, JPA impl, Flyway migration) with no benefit
- Aggregate counters only — insufficient for audit trail (who reported what), deduplication (prevent same miss inflating FN count), and per-record retention cleanup
**Rationale:** D3's revision unifies missed detection storage into `OutcomeLedger`. The JPA migration extends the existing `ras_outcome_record` table (new nullable columns or companion rows with a source discriminator) rather than creating a parallel table. Individual missed detection records are needed for: (1) audit trail — which operator reported which miss, (2) deduplication — prevent the same miss being counted multiple times, (3) retention cleanup — `removeRecordsBefore()` applies uniformly.
**Depends on:** D3 (SPI design determines what the persistence layer stores)
**Sources:** JpaOutcomeLedger.java, InMemoryOutcomeLedger.java, OutcomeRecordEntity.java
**Exploration:** quick
**Status:** revised (R1-06, R1-09: D3 revision eliminates need for parallel persistence; justified individual records over aggregate counters)

## D6: Recall computation

**Choice:** `recall()` on `OutcomeStatistics` (not on `QualityMetrics`) as `confirmedCount / (double)(confirmedCount + missedCount)`, using the same `retentionPeriod` time window as precision. Surfaced as `ras.feedback.recall` gauge per (situationId, tenancyId). No F1 score gauge initially.
**Alternatives:**
- Separate retention window for recall — makes precision and recall non-comparable: the TP count in recall's numerator would differ from the TP count in precision's numerator because they span different time ranges
- F1 score gauge from day one — premature; operators can compute F1 externally from precision and recall gauges; adding F1 is a one-line change when needed
- Per-ganglion recall — impossible with situation-level missed detection signals (D2); ganglion-level signals would be needed, which is a future enrichment on D2
**Rationale:** Using the same `FeedbackConfig.retentionPeriod()` time window ensures TP counts are identical in precision and recall computations, making the two metrics directly comparable. `recall()` is a method on `OutcomeStatistics` directly — NOT a default method on `QualityMetrics`. `QualityMetrics` is implemented by both `OutcomeStatistics` (situation-level) and `GanglionOutcomeStatistics` (per-ganglion). Per-ganglion missed detection data does not exist (D2 is situation-level only), so a `recall()` default on the interface would return `confirmedCount / (confirmedCount + 0)` = 1.0 for any ganglion with confirmed outcomes — semantically wrong (implies perfect recall when actually no data exists). For `OutcomeStatistics`, `missedCount == 0` means "no misses reported" (metric accurate given available data); for `GanglionOutcomeStatistics`, it would mean "measurement impossible" — categorically different. `recall()` returns NaN when `confirmedCount + missedCount == 0` (no decisive data), consistent with `precision()`'s NaN convention. Computed from `OutcomeStatistics` returned by `FeedbackAnalyzer` → `OutcomeLedger.statistics()` — no new query path. `FeedbackMetrics.recordStatistics()` gains the recall gauge; NaN suppresses gauge registration via the existing convention in `FeedbackMetrics.setGauge()`. When ganglion-level signals are added in a future enrichment of D2, `recall()` can be promoted to the `QualityMetrics` interface.
**Trade-offs:** Per-ganglion recall deferred. No F1 gauge — operators compute externally. Same time window means a burst of missed detection reports within the window can temporarily deflate recall even if the underlying detection rate hasn't changed.
**Sources:** QualityMetrics.java, OutcomeStatistics.java, FeedbackMetrics.java, FeedbackAnalyzer.java, issue #40 requirement 4
**Exploration:** quick (surfaced by reviewer R1-05)
**Status:** revised (R2-01: recall() placed on OutcomeStatistics, not QualityMetrics — prevents misleading 1.0 recall on GanglionOutcomeStatistics)

## D7: Missed detection validation

**Choice:** Validate on ingestion: (1) situation definition exists in `SituationDefinitionRegistry`, (2) `eventTime` within `retentionPeriod` of the situation's `FeedbackConfig`, (3) deduplication on composite key `(situationId, correlationKey, tenancyId, eventTime)` — duplicate reports for the same miss are idempotent
**Alternatives:**
- No validation — inflated FN count from duplicates and out-of-window reports; recall metric becomes unreliable
- Cross-reference `SituationQueryService.history()` to verify the event wasn't actually detected — conceptually correct but couples the ingestion path to the query SPI, adds query latency on every report, and the absence of a trigger record doesn't prove a miss (the event store may have been cleaned up by `SituationExpiryJob`)
- Full validation (situation exists + trigger didn't happen + temporal bounds + operator identity + rate limiting) — maximum correctness but heaviest; trigger-history check is the most valuable addition but has the caveats above
**Rationale:** Essential validations prevent data quality problems that directly corrupt the recall metric. Situation existence is a cheap `SituationDefinitionRegistry` lookup. Temporal bounds prevent operators from reporting misses outside the retention window (records would be cleaned up by `FeedbackUpdateJob` immediately anyway). Deduplication on the composite key prevents the same miss reported N times from deflating recall N-fold — the composite key captures "this specific situation should have fired for this correlation at this event time." Cross-referencing trigger history is deferred: operators reporting misses are asserting a fact about the real world based on domain knowledge the system doesn't have; the system should record the assertion, not second-guess it.
**Trade-offs:** Without trigger-history cross-reference, an operator can report a "miss" for something that was actually detected (operator error inflates FN, deflates recall). Acceptable for initial implementation — the epistemological limitation of operator reporting already bounds metric accuracy far more than operator errors do.
**Sources:** SituationDefinitionRegistry, SituationQueryService.java, FeedbackConfig.retentionPeriod(), OutcomeLedger dedup patterns
**Exploration:** quick (surfaced by reviewer R1-07)
**Status:** revised (R2-02: dedup key uses eventTime instead of reportedAt — event occurrence time, not report-filing time)

## D8: Drift direction classification

**Choice:** Deferred to implementation spec. `DriftDirection` does not exist in the codebase — neither as enum, class, interface, nor string literal in any Java source. Issue #60 and #58 reference it aspirationally. The recall metric is surfaced as a gauge (D6); drift classification is an interpretive layer that needs its own design.
**Alternatives:**
- Design DriftDirection in this decision set — would need to answer: enum vs sealed interface, producer (FeedbackAnalyzer vs FeedbackUpdateJob), consumer (FeedbackMetrics tag vs FeedbackTuningStrategy parameter), and how UNDER_SENSITIVE interacts with the existing noise-rate-driven threshold raising in DefaultTuningStrategy
- No DriftDirection — raw metrics (precision, recall, noise_rate) are sufficient for operator dashboards and alerting; classification adds abstraction without action since D4 says no auto-tuning for recall
**Rationale:** DriftDirection spans both precision (OVER_SENSITIVE when noise rate is high) and recall (UNDER_SENSITIVE when recall is low). Designing it requires decisions about the entire drift classification model — thresholds for when each direction applies, how they interact, whether they're mutually exclusive or can coexist. This is better addressed in the implementation spec where the full interaction model can be specified alongside the recall gauge implementation.
**Sources:** Issue #60, issue #58, DefaultTuningStrategy.java (noise rate > 0.5 threshold), rate-threshold spec (only other DriftDirection reference)
**Exploration:** quick (surfaced by reviewer R1-03 as implicit decision)
**Status:** captured
