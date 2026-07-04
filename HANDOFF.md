# HANDOFF — casehub-ras

**Date:** 2026-06-30
**Issues:** #22 (closed)

## What was done

Cross-repo migration of situation types from desiredstate-api to ras-api (#22).

- **casehub-desiredstate** (`issue-22-move-situation-types` branch, pushed): deleted ActiveSituation, SituationSource, SituationChangeEvent from api/, DefaultSituationSource stub from runtime/. Added ras-api dependency.
- **casehub-ops** (`issue-22-move-situation-types` branch, pushed): updated imports in 4 production + 4 test files. Expanded ActiveSituation (4→8 fields), SituationChangeEvent (1→4 fields), reactive SituationSource (List→Uni). Fixed pre-existing AgentCapability constructor breakage (capabilityVocabulary field added).
- **casehub-ops #6** updated: split into epic casehub-ops#29 with sub-issues #30 (service-as-case modelling) and #31 (operational efficiency). RAS #6 scoped to RAS-specific integration, blocked by ops#30.

## Key decisions

- ChannelDriftChecker migration (qhorus runtime→api Channel types) deferred — deeper than import swaps (Panache entities→records, CSV strings→typed collections). Needs its own issue.
- Service lifecycle epic moved from casehub-ras to casehub-ops — it's a Case/engine concern, not RAS.

## Immediate next step

The ops and desiredstate branches need merging. No remaining ras work — pick from backlog.

## Cross-repo issues filed (2026-07-05)

- **ras#27** — Promote SPI extension types (`CorrelationKeyExtractor`, `SituationDefinitionProvider`, `SituationRegistration`) from `io.casehub.ras.runtime` to `io.casehub.ras.api`. Filed during adversarial design review of casehub-desiredstate spec (constraint-evaluation-model). Non-breaking move. Scale: S, Complexity: Low.

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 27 | Promote SPI types to ras-api | S | Low | Filed from desiredstate ADR |
| 7 | DroolsSessionStore persistent implementation | M | Med | Unblocked |
| 6 | RAS integration with service lifecycle cases | M | Med | Blocked by ops#30 |
| 5 | Platform stream infrastructure | XL | High | Epic, needs design |
