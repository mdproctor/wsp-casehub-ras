# Handoff — casehub-ras

## What Changed

Epic 1 (Core RAS API) complete and merged to main. All API types, SPIs, InMemorySituationStore,
and test fixtures implemented. Design went through three review rounds — key outcomes:
CloudEvent replaces SensoryEvent (no wrapper), Model B routing (engine routes, ganglia evaluate),
engine-api removed from api/ classpath, sealed ChainMode hierarchy.

## Immediate Next Step

Start Epic 2 (RAS Runtime — #2): RasEngine, CompositeEventCorrelator, CaseTriggerService.
Run `/work` to begin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | RAS Runtime — RasEngine, CompositeEventCorrelator, CaseTriggerService | L | High | Blocked by #1 (done) |
| #3 | JavaSwitchGanglion — built-in fast-path deterministic detection | M | Med | Blocked by #1 (done) |
| #4 | DroolsGanglion — Drools CEP for temporal pattern detection | L | High | Blocked by #1 (done), #2 |
| #5 | Platform stream infrastructure — CloudEvent producers | L | High | Separate repo (platform#98 done) |
| #6 | Service lifecycle management — long-lived case + RAS + child ganglion cases | M | High | Design issue |

## References

- Design spec: `docs/superpowers/specs/2026-06-18-epic1-core-ras-api-design.md`
- Original spec: `docs/superpowers/specs/2026-06-12-casehub-ras-design.md`
- Blog: `blog/2026-06-20-mdp01-the-api-that-didnt-wrap.md`
