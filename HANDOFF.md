# HANDOFF — casehub-ras

**Date:** 2026-07-17
**Issues:** #39 (closed), #38 (closed as dup), #40–#46 (filed)

## What was done

Implemented OrphanedResourceCleaner SPI (#39) — generic cleanup interface in `api/`
for situation-derived resources across heterogeneous storage backends. Consolidated
`GanglionStateStore.removeOrphaned()` into the new SPI. `ReliableDroolsSessionStore`
implements it with `SituationStore.find()` per key, shutdown guard, and per-key error
isolation. `SituationExpiryJob` refactored to `Instance<OrphanedResourceCleaner>` loop
with per-cleaner error isolation. Closed #38 as duplicate of #39. Filed six roadmap
issues (#40–#46) for RAS expansion beyond Drools hardening.

## Key decisions

- Generic SPI in `api/` (not on `DroolsSessionStore` — module dep prevents runtime/ calling ras-drools/)
- Consolidated `GanglionStateStore.removeOrphaned()` under same SPI (eliminates dual pattern)
- `cleanerType()` on SPI for tagged metrics (`ras.expiry.orphans_cleaned`)
- `volatile boolean closed` shutdown guard in `ReliableDroolsSessionStore` (GE-20260710-86e8d3)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #46 | ExpressionEvaluator integration | L | Med | Pluggable expressions in YAML situations; depends on platform#141 |
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #42 | Situation templates | M | Med | Reusable parameterised definitions |
| #43 | Passive observation mode | M | Med | Richer query API beyond activeSituations() |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning from outcome data; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
