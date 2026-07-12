# HANDOFF — casehub-ras

**Date:** 2026-07-13
**Issues:** #32 (closed)

## What was done

RAS runtime metrics instrumentation (#32). Centralised `RasMetrics` `@ApplicationScoped`
bean with 22 metrics (18 counters, 2 timers, 2 gauges) across `RasEngine`,
`SituationEvaluator`, and `SituationExpiryJob`. `SituationStore` API change:
`removeExpired`/`removeTriggeredBefore` return `Uni<Integer>`. Design-reviewed
(2 rounds, 14 issues resolved — NotifyOnly coverage, removeExpired contract,
CDI cycle avoidance, timer placement). Garden entry GE-20260712-70d60c captured
`ide_edit_member` class-vs-constructor ambiguity gotcha.

## Key decisions

- Centralised `RasMetrics` bean (not inline per class) — single source of truth for
  metric names, tags, null-guarding. Inline pattern stays for `drools-reliability`
  (separate optional module)
- Opaque timer handles (`Object`) — callers don't reference `Timer.Sample`, preventing
  classloading when Micrometer absent
- `trigger_action` tag (`create_case`/`notify_only`) on fire_time/fired/failed — covers
  both TriggerAction paths without parallel metric names (R2-01 finding)
- `removeExpired` stays abstract, `removeTriggeredBefore` stays default (R2-02 finding)

## Cross-repo follow-up

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 33 | H2MVStore file-corruption recovery | S | Med | Filed during #28 design review |
| 31 | Ganglion-as-case — case-backed state persistence | M | High | Open question: still needed given DroolsSessionStore? |
| 29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| 30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| 5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
