# HANDOFF — casehub-ras

**Date:** 2026-07-18
**Issues:** #46 (closed), #47–#49 (filed)

## What was done

Implemented ExpressionEvaluator integration (#46) — expression-based correlation
key extraction, pre-ganglion event filtering, and dynamic case data in YAML situation
definitions. Three new fields on `SituationDefinition`, `EventFilter` interface,
`SituationRegistration` extended with compiled expressions. Registry-based compilation
at registration time via `ExpressionEngineRegistry`. JqMapAdapter bridges Map→JsonNode
for JQ engine. New `StringExpressionEvaluator` sub-interface in platform-api.
Also committed to platform repo (casehub-platform).

Design review ($14.94, 4 rounds, 18 issues) caught: missing `expression()` accessor,
wrong placement of `dynamicCaseData`, JQ context type incompatibility, merge order
vulnerability. Garden entry: Jackson `valueToTree()` + OffsetDateTime gotcha
(GE-20260718-expr01).

## Key decisions

- Expression descriptors on `SituationDefinition` (not `SituationRegistration`) — consistent with existing operational config pattern
- Registry compiles (not YAML provider) — centralised, works for all registration paths
- `CaseTriggerConfig` unchanged — `dynamicCaseData` moved to `SituationDefinition`
- JqMapAdapter in RAS (not platform engine fix) — bridges Map→JsonNode locally; #49 filed for native platform fix
- Merge order: static baseCaseData → dynamic expressions → correlation metadata → CaseInputContributors

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #47 | NaiveBayes feature extraction via expressions | M | Med | Needs YAML ganglion config |
| #48 | Evidence extraction templates | M | Med | Ganglion-level convenience |
| #49 | JQ engine native Map context support | S | Low | Removes JqMapAdapter workaround |
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #42 | Situation templates | M | Med | Reusable parameterised definitions |
| #43 | Passive observation mode | M | Med | Richer query API |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
