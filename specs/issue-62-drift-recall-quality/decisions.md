# Decisions — Feedback Quality Enrichment (#62, #63, #64)

## D1: Spec scope

**Choice:** One unified spec covering all three issues (#62 DriftDirection, #63 per-ganglion missed detection, #64 trigger-history cross-reference)
**Alternatives:**
- Three separate specs — more modular but risks inconsistency at boundaries (e.g., MissedDetectionRecord gains ganglionIds in #63 — does the cross-ref in #64 need it?)
**Rationale:** All three issues are tightly coupled through MissedDetectionRecord and QualityMetrics. Decisions in #62 (drift classification using recall) depend on #63 (per-ganglion recall computation) and interact with #64 (cross-reference affects data quality that drift classification interprets). One spec avoids contradictions.
**Trade-offs:** Larger spec, but the coupling justifies it — splitting would require cross-references between specs that duplicate decisions.
**Sources:** Issue #62, #63, #64 (all spawned from #60 deferred items)
**Exploration:** quick
**Status:** captured

## D2: DriftDirection type

**Choice:** 5-value enum: `OVER_SENSITIVE`, `UNDER_SENSITIVE`, `BOTH_DRIFTING`, `STABLE`, `INSUFFICIENT_DATA`
**Alternatives:**
- 4-value enum (mutually exclusive, no BOTH_DRIFTING) — simpler but hides the "miscalibrated" state where both noise rate and recall are bad
- Sealed interface with per-variant evidence — heavier; the evidence (noiseRate, recall) is already available in OutcomeStatistics
- Two independent boolean axes (record type) — more informationally complete but breaks the TrendDirection gauge tag pattern and is harder for operators to consume
- Two separate gauge names (R1-07) — `ras.feedback.drift.precision` and `ras.feedback.drift.recall` as separate per-metric classifications. More alerting-composable (single label match per axis) but loses the combined BOTH_DRIFTING synthesis, which is the primary value of drift classification. Per-axis alerting is already possible via the existing `ras.feedback.precision` and `ras.feedback.recall` gauges — DriftDirection adds the interpretive overlay.
**Rationale:** OVER_SENSITIVE and UNDER_SENSITIVE measure independent axes of the confusion matrix (FP rate vs FN rate). A detector CAN be both noisy AND miss real events — this is operationally distinct from either alone. BOTH_DRIFTING tells operators "reconfigure ganglia, not thresholds." Simple enum fits the gauge tag pattern (`ras.feedback.drift` with tag `direction=<value>`). Per-ganglion drift direction explicitly deferred — situation-level is sufficient for this iteration.
**Trade-offs:** 5 values instead of 4 adds a state operators must understand. Alerting on "all high-noise" requires `direction=~"OVER_SENSITIVE|BOTH_DRIFTING"` — two-label match rather than single. Acceptable because the combined signal is the primary value; per-axis alerting uses existing gauges directly.
**Sources:** TrendResult.TrendDirection pattern, DefaultTuningStrategy.java (noiseRate > 0.5 threshold), #60 D4 (no auto-tuning for recall), confusion matrix first-principles analysis
**Exploration:** deep-analysis
**Status:** revised (R1-02: removed incorrect dependency on D6 — situation-level recall already exists from #60. R1-07: addressed two-gauge alternative. R1-10: explicitly deferred per-ganglion drift direction.)

## D3: DriftDirection producer

**Choice:** FeedbackAnalyzer — new `classifyDrift(OutcomeStatistics, FeedbackConfig)` method
**Alternatives:**
- FeedbackUpdateJob — classify directly in the job after getting stats. Simpler (no new method) but scatters analysis logic across two classes
**Rationale:** FeedbackAnalyzer is the single interpreter of outcome data. `analyze()` returns OutcomeStatistics, `ganglionAnalyze()` returns per-ganglion stats — classifying drift from those stats is a natural extension. FeedbackUpdateJob calls `classifyDrift()` and publishes via FeedbackMetrics, same delegation pattern as today. FeedbackConfig parameter required for per-situation thresholds (D5).
**Trade-offs:** None significant — FeedbackAnalyzer is the right abstraction layer for interpretation.
**Sources:** FeedbackAnalyzer.java, FeedbackUpdateJob.java (existing delegation pattern)
**Exploration:** quick
**Status:** revised (R1-03: method signature updated to include FeedbackConfig parameter for per-situation configurable thresholds from D5)

## D4: DriftDirection consumer — observational with BOTH_DRIFTING guard

**Choice:** FeedbackMetrics gauge tag (`ras.feedback.drift` with `direction` tag). Additionally, when DriftDirection is BOTH_DRIFTING, FeedbackUpdateJob suppresses threshold auto-adjustment for that (situationId, tenancyId) pair. DriftDirection is NOT passed to FeedbackTuningStrategy — the guard is in the job's control flow.
**Alternatives:**
- Purely observational (original) — BOTH_DRIFTING is a signal but auto-tuning actively worsens recall by raising thresholds. Operators working on ganglion reconfiguration would fight the auto-tuner.
- FeedbackTuningStrategy parameter — DriftDirection influences all tuning decisions. Richer but couples classification to strategy, and goes beyond the guard needed.
**Rationale:** R1-04 identified an active contradiction: when BOTH_DRIFTING and tuningEnabled=true, auto-tuning raises thresholds (fixing noise) while simultaneously worsening recall — and the system signals "reconfigure ganglia, not thresholds." The guard is minimal: `if (drift == BOTH_DRIFTING) skip threshold adjustment`. Auto-tuning for OVER_SENSITIVE-only (the common case) is unchanged. Prior adjustment is also skipped during BOTH_DRIFTING — both tuning paths are inappropriate when the fundamental detection model is miscalibrated.
**Depends on:** D2 (BOTH_DRIFTING variant must exist), D3 (classifyDrift must produce the value)
**Trade-offs:** Operators with tuningEnabled=true lose auto-correction when BOTH_DRIFTING. Acceptable — the auto-correction was actively harmful in that state. Operators must manually intervene, which is the correct action for miscalibrated ganglia.
**Sources:** #60 D4 (no auto-tuning for recall), DefaultTuningStrategy.java, FeedbackUpdateJob.java (processTenant method), R1-04 review finding
**Exploration:** quick
**Status:** revised (R1-04: added BOTH_DRIFTING guard to suppress threshold/prior auto-adjustment — resolves active contradiction where auto-tuning worsens the condition the system is signaling)

## D5: Drift classification thresholds and logic

**Choice:** Configurable on FeedbackConfig — optional `overSensitiveThreshold` (default 0.5) and `underSensitiveThreshold` (default 0.5). Classification logic in FeedbackAnalyzer.classifyDrift():

```
1. If totalOutcomes < MIN_DRIFT_OUTCOMES (10) → INSUFFICIENT_DATA
2. overSensitive = !NaN(noiseRate) && noiseRate > overSensitiveThreshold
3. underSensitive = !NaN(recall) && recall < underSensitiveThreshold
   — recall NaN (no confirmed + no missed) → underSensitive = false (no data, not under-sensitive)
4. If overSensitive && underSensitive → BOTH_DRIFTING
5. If overSensitive → OVER_SENSITIVE
6. If underSensitive → UNDER_SENSITIVE
7. Else → STABLE
```

MIN_DRIFT_OUTCOMES applies to the total outcome count, not per-axis. This aligns with DefaultTuningStrategy's MIN_OUTCOMES_THRESHOLD = 10. Recall-based classification additionally requires `(confirmedCount + missedCount) >= MIN_RECALL_SAMPLES (3)` — without this minimum, a single missed report with one confirmed outcome gives recall = 0.5, triggering UNDER_SENSITIVE from trivial data.

**Alternatives:**
- Hardcoded constants in FeedbackAnalyzer — simpler, fewer config knobs, can be made configurable later
- Fixed symmetric at 0.5 — dead simple but 0.5 recall may be too generous for critical situations (fraud detection wants recall > 0.9)
**Rationale:** Different situations have different quality requirements. A fraud detector needs high recall (threshold 0.9); a marketing signal detector tolerates lower recall (threshold 0.3). Per-situation configuration mirrors the existing per-situation FeedbackConfig pattern. Defaults at 0.5 match DefaultTuningStrategy's existing noiseRate > 0.5 threshold. Semantics: `overSensitiveThreshold` is a noise rate CEILING (above = over-sensitive), `underSensitiveThreshold` is a recall FLOOR (below = under-sensitive).
**Depends on:** D2 (enum classification needs threshold values)
**Trade-offs:** Two more optional fields on FeedbackConfig plus two constants. Acceptable — FeedbackConfig already carries 6 fields, and these have sensible defaults.
**Sources:** FeedbackConfig.java, DefaultTuningStrategy.java (noiseRate > 0.5, MIN_OUTCOMES_THRESHOLD = 10), YAML schema for feedback config
**Exploration:** quick
**Status:** revised (R1-05: specified full classification logic including NaN handling, minimum sample sizes, and threshold semantics. R1-09: added MIN_RECALL_SAMPLES guard for recall-based classification.)

## D6: Per-ganglion missed detection signal

**Choice:** Optional `List<String> ganglionIds` on MissedDetectionRecord. Null/empty = situation-level miss (backwards compatible). Non-empty = operator specifies which ganglion(s) should have fired.
**Alternatives:**
- Separate MissedGanglionRecord type — cleaner type separation but doubles the ingestion surface (new SPI method, new table, new REST endpoint)
- Single optional ganglionId — one ganglionId per record, file multiple reports for multi-ganglion misses. Simpler record but more HTTP calls.
**Rationale:** Most operators report situation-level misses — they know the situation should have fired but don't know which specific ganglion failed. Optional ganglionIds allows advanced operators to provide granularity without burdening the common path. List (not single) because a multi-ganglion situation (e.g., AND chain mode) might have multiple ganglia that should have contributed. Backwards compatible: existing MissedDetectionRecord callers pass null/empty and behavior is unchanged.
**Depends on:** D7 (per-ganglion recall computation uses this data)
**Trade-offs:** Situation-level misses (the common case) don't contribute to per-ganglion recall — only reports with explicit ganglionIds do. Per-ganglion recall is a lower-fidelity metric than situation-level recall because fewer reports will carry ganglion attribution. Dedup key remains the situation-level composite `(situationId, correlationKey, tenancyId, eventTime)` — a ganglion-enriched report after a situation-level report for the same event is rejected as duplicate. Late enrichment is not supported; operators who want ganglion attribution must include it in their initial report.
**Sources:** MissedDetectionRecord.java, GanglionOutcomeStatistics.java, issue #63 (per-ganglion recall requirement), issue #40 requirement 4
**Exploration:** quick
**Status:** revised (R1-06: documented dedup limitation — first reporter's ganglion attribution is permanent, late enrichment not supported)

## D7: Per-ganglion recall — separate method, NOT on QualityMetrics

**Choice:** GanglionOutcomeStatistics gains `missedCount` field and its own `recall()` method. `recall()` is NOT promoted to QualityMetrics. OutcomeStatistics keeps its existing `recall()` unchanged.
**Alternatives:**
- Promote recall() to QualityMetrics as a default method — creates false-1.0 for ganglia with no explicit missed detection reports (most ganglia), recreating the exact problem #60 D6 was designed to prevent
**Rationale:** R1-01 correctly identified that the false-1.0 problem persists: most operators will file situation-level missed reports (no ganglionIds), leaving per-ganglion missedCount at 0. A `recall()` default on QualityMetrics would return `confirmedCount / (confirmedCount + 0) = 1.0` — "semantically wrong (implies perfect recall when actually no data exists)" as #60 D6 stated. Having the ability to record ganglion-level misses does not mean operators will exercise it. Per-ganglion recall can be promoted to QualityMetrics in the future once adoption evidence shows sufficient ganglion-level reporting (e.g., a configurable threshold percentage of situations with ganglion-level missed reports).
**Depends on:** D6 (ganglion-level missed signals provide the data for this computation)
**Trade-offs:** Asymmetry: OutcomeStatistics.recall() and GanglionOutcomeStatistics.recall() are structurally identical methods but not shared via the interface. Acceptable — the asymmetry reflects a real semantic difference: situation-level missedCount=0 means "no misses reported" (accurate given available data), per-ganglion missedCount=0 means "measurement not available" (no ganglion-level reports filed). Different semantics, different methods.
**Sources:** QualityMetrics.java, OutcomeStatistics.java, GanglionOutcomeStatistics.java, #60 D6 (rationale for deferring), #60 spec §Why Not on QualityMetrics, R1-01 review finding
**Exploration:** quick
**Status:** revised (R1-01: reverted promotion — false-1.0 problem persists for ganglia without explicit missed reports. recall() stays off QualityMetrics, GanglionOutcomeStatistics gets its own method.)

## D8: Trigger-history cross-reference

**Choice:** Advisory warning in MissedDetectionRecorder. Add optional `Instance<SituationQueryService>` dependency. After validation passes, query `history(tenancyId, situationId, correlationKey, eventTime - crossRefWindow, eventTime + crossRefWindow)` for TRIGGERED events. Cross-reference window: configurable `Duration crossRefWindow` on FeedbackConfig (default `PT1H` — one hour before/after the reported eventTime). Add `possiblyDetected` boolean and `Optional<Instant> lastTriggerTime` to RecordResult. Record is still stored regardless — the system informs but doesn't override operator judgment.
**Alternatives:**
- Soft rejection — don't store the record if a trigger was found. Operators must re-report if they believe the cross-ref is wrong. Overrides operator domain knowledge.
- No cross-reference — close #64 as wontfix. The epistemological argument from #60 D7 still holds. Misses the chance to surface an easy data quality improvement.
- Separate MissedDetectionValidator service — more classes but cleaner separation. Unnecessary given MissedDetectionRecorder already handles all validation.
- In the REST resource — scatters logic, keeps recorder simpler but breaks the pattern (recorder owns all validation)
**Rationale:** Operators are asserting domain facts the system can't verify (#60 D7). The cross-reference adds a second opinion, not a veto. SituationQueryService.history() provides the exact query needed — match by (tenancyId, situationId, correlationKey) within a time window around eventTime. Optional dependency via `Instance<>` maintains zero coupling when SituationQueryService is absent. Cross-reference window defaults to 1 hour — wide enough for clock skew and event processing delays, narrow enough to be meaningful.
**Trade-offs:** Adds query latency to every missed detection report (one SituationQueryService query). Acceptable — missed detection reporting is a low-volume human-driven operation, not a hot path. Retention mismatch: if `ras.event-history.retention` (default P30D) is shorter than `FeedbackConfig.retentionPeriod`, the cross-reference returns "not detected" for events whose trigger history has been cleaned up. The `possiblyDetected` response must include a caveat: "absence of trigger history does not confirm a miss — events older than the event history retention period cannot be cross-referenced."
**Sources:** SituationQueryService.java (history method), MissedDetectionRecorder.java, #60 D7 (deferred cross-reference), issue #64 (considerations section), R1-08 review finding
**Exploration:** quick
**Status:** revised (R1-08: defined cross-reference window as configurable Duration on FeedbackConfig with PT1H default. Documented retention mismatch caveat for possiblyDetected response.)
