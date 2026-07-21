# HANDOFF — casehub-ras

**Date:** 2026-07-21
**Issues:** #48 (closed), #50 (closed), #51 (filed), #52 (filed)

## What was done

Closed #48 + #50 on branch `issue-48-evidence-extraction-expression-rules` (39a7ebf).
Added `EvidenceExtractingGanglion` wrapper — cross-cutting evidence template extraction
from CloudEvent via compiled expressions, merged into DetectionResult evidence. Added
`ExpressionRulesGanglion` — stateless, evaluates ordered boolean rules in YAML, first
match wins. `GanglionDescriptor` sealed interface gains `ExpressionRules` variant and
`evidenceTemplates()` default method on all variants. Shared `parseExpressionEntry()`
consolidates 5 callers. Design review ($15.93, 5 rounds, 10 issues). Filed #51
(per-rule evidence) and #52 (dynamic confidence) as follow-ons.

## Key decisions

- Evidence templates extract from CloudEvent only — DetectionResult portability invariant
- EvidenceExtractingGanglion as wrapper pattern, not built into each ganglion
- Confidence validated at YAML parse time (0.0–1.0), not deferred to runtime
- `matchedRuleIndex` automatic evidence in ExpressionRulesGanglion
- Key collision warning (not error) when templates shadow automatic evidence keys

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #51 | Per-rule evidence templates in expression-rules ganglion | S | Low | Extends ExpressionRules.Rule with evidence map |
| #52 | Dynamic confidence expressions | S | Med | Expression-computed confidence per rule |
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #42 | Situation templates | M | Med | Reusable parameterised definitions |
| #43 | Passive observation mode | M | Med | Richer query API |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
