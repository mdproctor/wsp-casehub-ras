# HANDOFF — casehub-ras

**Date:** 2026-08-07
**Issues:** #40 (closed)

## What was done

Closed #40 on branch `issue-40-ras-feedback-loop` (a1ca741). Implemented the
full RAS feedback loop — case outcomes feed back into detection tuning. Three-layer
composable pipeline: OutcomeRecorder (CaseOutcomeObserver ingestion), FeedbackAnalyzer
(batch statistics), FeedbackUpdateJob (scheduled threshold/prior adjustment). Advisory
mode (suppression + metrics) default; tuning mode adds automatic NaiveBayes prior
recalibration via outcomeGroundTruth mapping and ChainMode.Threshold drift. JPA
persistence via V7 Flyway migration. YAML parsing for `feedback:` section and
`outcomeGroundTruth:`. Code review caught @Scheduled overlap risk (fixed with
ConcurrentExecution.SKIP). Garden entries: branch-drift variant (GE-20260522-543863
revised), Hibernate native MAX timestamp gotcha (GE-20260807-66fe1b).

## Key decisions

- Suppression in SituationEvaluator, not DefaultRasTriggerPolicy (policy purity preserved)
- FeedbackState separated from SituationDefinitionRegistry (mutable vs immutable)
- Log-space prior boundary at applyPriorOverride() (raw-to-log conversion at API boundary)
- Advisory mode default (tuningEnabled: false) — suppression active, tuning opt-in
- JPQL for typed aggregate queries, native SQL only for ON CONFLICT and bulk DELETE

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #41 | Meta-situations | L | High | Situations observing other situations |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; builds on #40 feedback loop |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
