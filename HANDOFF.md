# HANDOFF — casehub-ras

**Date:** 2026-07-20
**Issues:** #49 (closed), #47 (closed), #50 (filed — expression-rule ganglion)

## What was done

Closed #49 + #47 on branch `issue-49-jq-map-context-naivebayes-expr` (061f693).
Removed `JqMapAdapter` workaround — platform `JQExpressionEngine` now handles
Map context natively. Added `JqResultUnwrapper` for scalar result types.
Added `GanglionDescriptor` sealed interface in `api/` with `NaiveBayes` variant.
`SituationDefinitionProvider.ganglionDescriptors()` default method. YAML `ganglia:`
section in `ras-situations.yaml` with per-feature expression extraction.
`ExpressionFeatureExtractor` implements `NaiveBayesFeatureExtractor`. Three-phase
registry constructor. Design review ($12.47, 3 rounds, 13 issues). Platform
prereqs completed externally: `StringExpressionEvaluator` (48d82d6) and JQ Map
context fix (f66bba4). Platform issue filed: casehubio/platform#190 (JQ resultType).

## Key decisions

- GanglionDescriptor as sealed interface with type discriminator — extensible for expression-rules ganglion
- Per-feature expressions co-located with likelihood tables in YAML (not separate sections)
- Nullable MeterRegistry in ExpressionFeatureExtractor (avoids circular CDI dependency with RasMetrics)
- JqResultUnwrapper is an acknowledged workaround — platform#190 filed for native fix

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | Expression-rule ganglion (type: expression-rules) | M | Med | Discussed during brainstorming, deferred |
| #48 | Evidence extraction templates | M | Med | Ganglion-level convenience |
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #42 | Situation templates | M | Med | Reusable parameterised definitions |
| #43 | Passive observation mode | M | Med | Richer query API |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
