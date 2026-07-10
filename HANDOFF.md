# HANDOFF — casehub-ras

**Date:** 2026-07-10
**Issues:** #28 (closed)

## What was done

DroolsSessionStore production hardening (#28). Added Micrometer metrics instrumentation
(6 counters, 1 gauge, 1 timer — optional via `Instance<MeterRegistry>`), `@Readiness`
health check probing H2MVStore, typed `DroolsSessionStoreException` SPI error contract,
and per-ganglion error isolation in `SituationEvaluator.runDetection()`. Fixed `@PreDestroy`
to preserve persisted data — `dispose()` was deleting H2MVStore state on every graceful restart.
Design-reviewed (3 rounds, 14 issues, all resolved). Garden entry GE-20260710-86e8d3 captured
H2MVStore read/write asymmetry after `close()`.

## Key decisions

- Fail-fast for storage reads (can't proceed without knowing state), log-and-continue for
  storage writes (session works, just not durable), silent recovery for corrupt sessions
  (existing behavior preserved with counter)
- Annotation-only library deps (`micrometer-core`, `microprofile-health-api`) per protocol
  PP-20260604-88f660 — consuming app provides Quarkus extensions
- No `StorageManager.close()` in `@PreDestroy` — static singleton breaks dev-mode restarts
- Per-ganglion isolation: one ganglion's failure doesn't kill evaluation for others

## Cross-repo follow-up

- casehubio/casehub-ops#47 — wire RAS health monitoring for managed applications (first consumer)
- casehubio/casehub-ops#48 — migrate AdaptiveTopologyManager to enriched SituationChangeEvent
- casehubio/casehub-desiredstate#71 — migrate DesiredStateSituationDefinitionProvider to TriggerAction API

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 32 | RAS runtime metrics instrumentation (SituationEvaluator, RasEngine) | M | Med | Filed during #28 design review |
| 33 | H2MVStore file-corruption recovery | S | Med | Filed during #28 design review |
| 31 | Ganglion-as-case — case-backed state persistence | M | High | Open question: still needed given DroolsSessionStore? |
| 29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| 30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| 5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
