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
**Rationale:** OVER_SENSITIVE and UNDER_SENSITIVE measure independent axes of the confusion matrix (FP rate vs FN rate). A detector CAN be both noisy AND miss real events — this is operationally distinct from either alone. BOTH_DRIFTING tells operators "reconfigure ganglia, not thresholds." Simple enum fits the gauge tag pattern (`ras.feedback.drift` with tag `direction=<value>`).
**Depends on:** D6 (recall data must exist for UNDER_SENSITIVE classification)
**Trade-offs:** 5 values instead of 4 adds a state operators must understand. But BOTH_DRIFTING maps to a clear action (ganglion reconfiguration) that is genuinely different from either single-direction drift.
**Sources:** TrendResult.TrendDirection pattern, DefaultTuningStrategy.java (noiseRate > 0.5 threshold), #60 D4 (no auto-tuning for recall), confusion matrix first-principles analysis
**Exploration:** deep-analysis
**Status:** captured

## D3: DriftDirection producer

**Choice:** FeedbackAnalyzer — new `classifyDrift(OutcomeStatistics)` method
**Alternatives:**
- FeedbackUpdateJob — classify directly in the job after getting stats. Simpler (no new method) but scatters analysis logic across two classes
**Rationale:** FeedbackAnalyzer is the single interpreter of outcome data. `analyze()` returns OutcomeStatistics, `ganglionAnalyze()` returns per-ganglion stats — classifying drift from those stats is a natural extension. FeedbackUpdateJob calls `classifyDrift()` and publishes via FeedbackMetrics, same delegation pattern as today.
**Trade-offs:** None significant — FeedbackAnalyzer is the right abstraction layer for interpretation.
**Sources:** FeedbackAnalyzer.java, FeedbackUpdateJob.java (existing delegation pattern)
**Exploration:** quick
**Status:** captured

## D4: DriftDirection consumer — observational only

**Choice:** FeedbackMetrics gauge tag only — `ras.feedback.drift` gauge with `direction` tag. Not fed into FeedbackTuningStrategy.
**Alternatives:**
- FeedbackTuningStrategy parameter — DriftDirection influences tuning decisions. E.g., suppress threshold raising when BOTH_DRIFTING (raising threshold worsens recall). Richer but couples drift classification to tuning behavior, and #60 D4 explicitly says no auto-tuning for recall.
**Rationale:** D4 from #60 established that recall is metric-only with no auto-tuning response. DriftDirection is an interpretive layer on top of existing gauges — its purpose is dashboard visibility and alerting, not automated response. Auto-tuning behavior (DefaultTuningStrategy raises threshold when noiseRate > 0.5) is unchanged. Suppressing auto-tuning when BOTH_DRIFTING is a future consideration that deserves its own analysis.
**Trade-offs:** When BOTH_DRIFTING, existing auto-tuning will raise threshold (making recall worse). Acceptable for this iteration — operators observe the BOTH_DRIFTING signal and intervene manually.
**Sources:** #60 D4 (no auto-tuning for recall), DefaultTuningStrategy.java, FeedbackMetrics.java
**Exploration:** quick
**Status:** captured

## D5: Drift classification thresholds

**Choice:** Configurable on FeedbackConfig — optional `overSensitiveThreshold` (default 0.5) and `underSensitiveThreshold` (default 0.5)
**Alternatives:**
- Hardcoded constants in FeedbackAnalyzer — simpler, fewer config knobs, can be made configurable later
- Fixed symmetric at 0.5 — dead simple but 0.5 recall may be too generous for critical situations (fraud detection wants recall > 0.9)
**Rationale:** Different situations have different quality requirements. A fraud detector needs high recall (threshold 0.9); a marketing signal detector tolerates lower recall (threshold 0.3). Per-situation configuration mirrors the existing per-situation FeedbackConfig pattern. Defaults at 0.5 match DefaultTuningStrategy's existing noiseRate > 0.5 threshold.
**Depends on:** D2 (enum classification needs threshold values)
**Trade-offs:** Two more optional fields on FeedbackConfig. Acceptable — FeedbackConfig already carries 6 fields, and these have sensible defaults.
**Sources:** FeedbackConfig.java, DefaultTuningStrategy.java (noiseRate > 0.5), YAML schema for feedback config
**Exploration:** quick
**Status:** captured

## D6: Per-ganglion missed detection signal

**Choice:** Optional `List<String> ganglionIds` on MissedDetectionRecord. Null/empty = situation-level miss (backwards compatible). Non-empty = operator specifies which ganglion(s) should have fired.
**Alternatives:**
- Separate MissedGanglionRecord type — cleaner type separation but doubles the ingestion surface (new SPI method, new table, new REST endpoint)
- Single optional ganglionId — one ganglionId per record, file multiple reports for multi-ganglion misses. Simpler record but more HTTP calls.
**Rationale:** Most operators report situation-level misses — they know the situation should have fired but don't know which specific ganglion failed. Optional ganglionIds allows advanced operators to provide granularity without burdening the common path. List (not single) because a multi-ganglion situation (e.g., AND chain mode) might have multiple ganglia that should have contributed. Backwards compatible: existing MissedDetectionRecord callers pass null/empty and behavior is unchanged.
**Depends on:** D7 (per-ganglion recall computation uses this data)
**Trade-offs:** Situation-level misses (the common case) don't contribute to per-ganglion recall — only reports with explicit ganglionIds do. Per-ganglion recall is a lower-fidelity metric than situation-level recall because fewer reports will carry ganglion attribution.
**Sources:** MissedDetectionRecord.java, GanglionOutcomeStatistics.java, issue #63 (per-ganglion recall requirement), issue #40 requirement 4
**Exploration:** quick
**Status:** captured

## D7: Recall promotion to QualityMetrics

**Choice:** Promote `recall()` from OutcomeStatistics to QualityMetrics interface as a default method. GanglionOutcomeStatistics gains `missedCount` field.
**Alternatives:**
- Keep recall() on OutcomeStatistics only — per-ganglion recall computed separately. Avoids false-1.0 for ganglia with no explicit missed signals, but creates asymmetry in the interface.
**Rationale:** With ganglion-level missed detection signals (D6), the false-1.0 problem identified in #60 D6 goes away for ganglia that have explicit missed reports. `recall()` as a default method on QualityMetrics: `confirmedCount / (confirmedCount + missedCount)`. When `missedCount == 0 && confirmedCount == 0`, returns NaN (no data). When `missedCount == 0 && confirmedCount > 0`, returns 1.0 — now semantically correct because the system HAS the ability to record ganglion-level misses; absence of missed reports means "none reported" rather than "measurement impossible."
**Depends on:** D6 (ganglion-level missed signals must exist for this to be semantically valid)
**Trade-offs:** Ganglia with no explicit missed signals will show recall=1.0 (or NaN if no confirmed outcomes either). This is technically correct ("no misses reported") but could be misleading for operators who haven't yet filed ganglion-level reports. The gauge tag should clarify: "recall reflects reported misses only."
**Sources:** QualityMetrics.java, OutcomeStatistics.java, GanglionOutcomeStatistics.java, #60 D6 (rationale for deferring), #60 spec §Why Not on QualityMetrics
**Exploration:** quick
**Status:** captured

## D8: Trigger-history cross-reference

**Choice:** Advisory warning in MissedDetectionRecorder. Add optional `Instance<SituationQueryService>` dependency. After validation passes, query `history(tenancyId, situationId, correlationKey, eventTime - window, eventTime + window)` for TRIGGERED events. Add `possiblyDetected` boolean and `Optional<Instant> lastTriggerTime` to RecordResult. Record is still stored regardless — the system informs but doesn't override operator judgment.
**Alternatives:**
- Soft rejection — don't store the record if a trigger was found. Operators must re-report if they believe the cross-ref is wrong. Overrides operator domain knowledge.
- No cross-reference — close #64 as wontfix. The epistemological argument from #60 D7 still holds. Misses the chance to surface an easy data quality improvement.
- Separate MissedDetectionValidator service — more classes but cleaner separation. Unnecessary given MissedDetectionRecorder already handles all validation.
- In the REST resource — scatters logic, keeps recorder simpler but breaks the pattern (recorder owns all validation)
**Rationale:** Operators are asserting domain facts the system can't verify (#60 D7). The cross-reference adds a second opinion, not a veto. SituationQueryService.history() provides the exact query needed — match by (tenancyId, situationId, correlationKey) within a time window around eventTime. Optional dependency via `Instance<>` maintains zero coupling when SituationQueryService is absent.
**Trade-offs:** Adds query latency to every missed detection report (one SituationQueryService query). Acceptable — missed detection reporting is a low-volume human-driven operation, not a hot path. Absence of a trigger record doesn't prove a miss (SituationExpiryJob may have cleaned it) — `possiblyDetected` should be surfaced with this caveat in the API docs.
**Sources:** SituationQueryService.java (history method), MissedDetectionRecorder.java, #60 D7 (deferred cross-reference), issue #64 (considerations section)
**Exploration:** quick
**Status:** captured
