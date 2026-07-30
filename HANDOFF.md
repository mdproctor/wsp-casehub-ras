# HANDOFF — casehub-ras

**Date:** 2026-07-30
**Issues:** #43 (closed)

## What was done

Closed #43 on branch `issue-43-passive-observation-mode` (cb00121). Added
SituationQueryService SPI backed by an append-only `ras_situation_event` table
capturing situation lifecycle transitions. Three history() overloads (tenant,
situation, correlation key), triggerCount(), trend() with rate normalization,
health() with per-tenant aggregation. SituationEventRetention SPI for TTL
cleanup via SituationExpiryJob (configurable, default P30D). CDI @ObservesAsync
recorder in persistence-jpa/ with @Transactional + try-catch for NotifyOnly
trigger safety. InMemory and JPA implementations both passing 33 contract tests.
Design review (4 rounds, 17 issues, $12.98). Garden entry GE-20260730-d54a8f
(fireAsync().join() exception propagation gotcha).

## Key decisions

- SituationSource stays for live state; new SPI for historical (separate data sources)
- Terminal transitions only — no inception events (SituationEvaluator unchanged)
- SituationEventRetention separate from SituationQueryService (design review R1-04)
- firstSeen column enables accumulation duration computation (design review R1-03)
- TrendResult.compute() static factory shared across implementations (avoids cross-module dep)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
