# Handoff — casehub-ras

## What Changed

Epic 2 (RAS Runtime — #2) designed, implemented, reviewed, and merged. Three-layer
decomposition: RasEngine (`@ObservesAsync CloudEvent`) → SituationEvaluator (striped
locking, chain mode dispatch) → CaseTrigger SPI (engine bridge). API gained
`correlationKey` (three-field identity), `TimestampedDetection` (Sequence ordering),
`CaseTrigger` SPI, `ChainMode.referencedGanglia()`. All downstream modules updated
(persistence-memory, ras-drools, testing). 162 tests across reactor. Design spec
reviewed twice with all findings resolved. Deferred items filed as #12–#16.

## Immediate Next Step

Pick next work from the backlog — #3 (JavaSwitchGanglion) or #5 (platform stream
infrastructure) are both unblocked. Run `/work` to begin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | JavaSwitchGanglion — built-in fast-path deterministic detection | M | Med | Ready — runtime exists |
| #5 | Platform stream infrastructure — CloudEvent producers | L | High | Separate repo |
| #6 | Service lifecycle management — long-lived case + RAS + child ganglion cases | M | High | Design issue |
| #7 | Persistent DroolsSessionStore (restart survival) | M | High | Blocked by #4 (done) |
| #9 | DroolsGanglion hot rule reload | M | High | Blocked by #4 (done) |
| #12 | Ganglion compact() invocation for persistent situations | S | Med | Deferred from #2 |
| #13 | YAML-driven SituationDefinitionProvider | S | Low | Deferred from #2 |
| #14 | JPA SituationStore — persistent situation storage | M | Med | Deferred from #2 |
| #15 | ANTI signal handling in DefaultRasTriggerPolicy | S | Med | Deferred from #2 |
| #16 | Per-situation event reordering buffer for pseudo clock mode | M | High | Deferred from #2 |

## References

- Epic 2 spec: `docs/superpowers/specs/2026-06-25-epic2-ras-runtime-design.md`
- Blog: `blog/2026-06-25-mdp01-the-runtime-that-found-its-own-clock.md`
