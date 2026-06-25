# Handoff — casehub-ras

## What Changed

S-scale deferrals from Epic 2 cleared as a batch (#13, #15, #12). YamlSituationDefinitionProvider
reads situation definitions from classpath YAML (configurable via `ras.situations.yaml`). ANTI
detections now subtract confidence in Threshold mode. SituationEvaluator calls compact() on
referenced ganglia for persistent situations after CONTINUE_ACCUMULATING. All three issues closed.
CLAUDE.md and Epic 2 spec updated to reflect new behaviour.

## Immediate Next Step

Pick next work from the backlog — #3 (JavaSwitchGanglion) or #5 (platform stream infrastructure)
are both unblocked. Run `/work` to begin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | JavaSwitchGanglion — built-in fast-path deterministic detection | M | Med | Ready — runtime exists |
| #5 | Platform stream infrastructure — CloudEvent producers | L | High | Separate repo |
| #6 | Service lifecycle management — long-lived case + RAS + child ganglion cases | M | High | Design issue |
| #7 | DroolsSessionStore: persistent implementation for restart survival | M | High | Blocked by #4 (done) |
| #9 | DroolsGanglion: hot rule reload without restart | M | High | Blocked by #4 (done) |
| #14 | JPA SituationStore — persistent situation storage | M | Med | Deferred from #2 |
| #16 | Per-situation event reordering buffer for pseudo clock mode | M | High | Deferred from #2 |

## References

- Blog: `blog/2026-06-25-mdp01-the-deferrals-that-took-an-afternoon.md`
- Epic 2 spec: `docs/superpowers/specs/2026-06-25-epic2-ras-runtime-design.md`
