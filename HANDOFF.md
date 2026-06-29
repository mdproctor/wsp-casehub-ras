# HANDOFF — casehub-ras

**Date:** 2026-06-29
**Branch closed:** issue-19-duplicate-trigger-contract-test → merged to main (18d43aa)
**Issues:** #19 (closed), #17 (closed)

## What was done

Duplicate trigger prevention for clustered deployments + SituationStore contract test.

- **tryClaimTrigger/resetTriggerClaim** — new SPI default methods on `SituationStore`.
  JPA: conditional JPQL UPDATE on `policy_triggered` column (V3 migration).
  InMemory: `ConcurrentHashMap.putIfAbsent`. Both stores override (not defaults).
- **Bifurcated claim path** in `SituationEvaluator.executeDecision()` — save-before-claim
  for new entities (creates the row), claim-before-save for existing (preserves `lastSignal`
  for correct expiry). Deferred entity removal — `policyTriggered=true` guards against
  retrying losers, cleaned up by expiry.
- **AbstractSituationStoreContractTest** — 12 shared behavioral tests in api/ test-jar.
  Both `InMemorySituationStoreTest` and `JpaSituationStoreTest` extend it.
- **InMemorySituationStore** gains `storeVersion` population (AtomicLong counter on save)
  so the evaluator's bifurcated path works consistently across both stores.

## Key decisions

- `policyTriggered` is store-level (not on `SituationContext`) — coordination mechanism,
  not domain state. Conditional UPDATE operates independently of find→modify→save cycle.
- Deferred removal: entity stays after trigger (design review R1-02 proved removing allows
  loser to re-create with fresh claim and fire duplicate).
- Claim-failure returns `true` (terminated) — the winner owns the cycle, loser cleans up
  in-memory resources (per-key lock, EventReorderBuffer).
- InMemory must override (not use defaults) — with deferred removal, default `tryClaimTrigger`
  (always true) lets every post-trigger event re-fire.

## What's next

- Deferred from earlier: #5 (platform stream infra), #6 (service lifecycle), #7 (DroolsSessionStore persistent).
