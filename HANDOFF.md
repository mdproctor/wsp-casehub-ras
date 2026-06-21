# Handoff — casehub-ras

## What Changed

Epic 4 (DroolsGanglion) complete and merged to main. `ras-drools/` module implements
`DroolsGanglion` — optional Drools CEP ganglion using classic kie-api (10.1.0). Configurable
session modes (LONG_LIVED/EPHEMERAL), pseudo clock driven by CloudEvent timestamps,
`DroolsObjectExtractor` SPI for fact extraction, `DroolsSessionStore` SPI for session lifecycle.
`Ganglion.close()` added to the SPI in `casehub-ras-api` — lifecycle callback for situation
termination, fixing a leak that affects all stateful ganglia.

## Immediate Next Step

Start Epic 2 (RAS Runtime — #2): RasEngine, CompositeEventCorrelator, CaseTriggerService.
Run `/work` to begin.

## What's Left

- `#10` — DroolsGanglion minor test coverage improvements (close() ephemeral test, null clock test, buildAll consistency) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | RAS Runtime — RasEngine, CompositeEventCorrelator, CaseTriggerService | L | High | Ready — #1 done |
| #3 | JavaSwitchGanglion — built-in fast-path deterministic detection | M | Med | Ready — #1 done |
| #5 | Platform stream infrastructure — CloudEvent producers | L | High | Separate repo |
| #6 | Service lifecycle management — long-lived case + RAS + child ganglion cases | M | High | Design issue |
| #7 | Persistent DroolsSessionStore (restart survival) | M | High | Blocked by #4 (done) |
| #8 | DroolsGanglion configurable result collection strategy | S | Med | Blocked by #4 (done) |
| #9 | DroolsGanglion hot rule reload | M | High | Blocked by #4 (done) |

## References

- DroolsGanglion spec: `docs/superpowers/specs/2026-06-21-epic4-drools-ganglion-design.md`
- Epic 1 spec: `docs/superpowers/specs/2026-06-18-epic1-core-ras-api-design.md`
- Blog: `blog/2026-06-21-mdp01-the-ganglion-that-got-reviewed-four-times.md`
