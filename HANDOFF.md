# Handoff — casehub-ras

## What Changed

#8 (configurable result collection strategy) and #10 (test gaps) closed and merged.
`DetectionSignal` reordered to ascending strength (NOISE, ANTI, WEAK, DETECTED) with
`isAtLeast()` — api/ change, usable by trigger policies. `ResultCollectionStrategy` enum
added to `ras-drools/` with four strategies (HIGHEST_CONFIDENCE default, FIRST_MATCH,
LAST_WINS, ACCUMULATE). `DroolsGanglionConfig` gains the strategy as a config field.
Three test gaps from Epic 4 review closed.

## Immediate Next Step

Start Epic 2 (RAS Runtime — #2): RasEngine, CompositeEventCorrelator, CaseTriggerService.
Run `/work` to begin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | RAS Runtime — RasEngine, CompositeEventCorrelator, CaseTriggerService | L | High | Ready — #1 done |
| #3 | JavaSwitchGanglion — built-in fast-path deterministic detection | M | Med | Ready — #1 done |
| #5 | Platform stream infrastructure — CloudEvent producers | L | High | Separate repo |
| #6 | Service lifecycle management — long-lived case + RAS + child ganglion cases | M | High | Design issue |
| #7 | Persistent DroolsSessionStore (restart survival) | M | High | Blocked by #4 (done) |
| #9 | DroolsGanglion hot rule reload | M | High | Blocked by #4 (done) |

## References

- Result collection spec: `docs/superpowers/specs/2026-06-22-drools-result-collection-and-test-gaps.md`
- DroolsGanglion spec: `docs/superpowers/specs/2026-06-21-epic4-drools-ganglion-design.md`
- Epic 1 spec: `docs/superpowers/specs/2026-06-18-epic1-core-ras-api-design.md`
- Blog: `blog/2026-06-23-mdp01-the-collection-strategy-that-started-wrong.md`
