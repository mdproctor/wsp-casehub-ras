---
layout: post
title: "The claim that nobody could win"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [concurrency, occ, jpa, conditional-update, exactly-once]
---

I came into this branch with a clear problem: two JVMs processing events for the same situation both evaluate to CREATE_CASE, both fire `caseTrigger.fire()`, and you get a duplicate case. The #18 work had already put OCC on the CONTINUE_ACCUMULATING path — `@Version` plus `SituationConflictException` plus retry loop. CREATE_CASE was the gap: it called `store.remove()` (bulk JPQL DELETE, no version check) and bypassed everything.

The garden had a ready-made pattern — GE-20260512-e3e525 describes using a `policyTriggered` flag with a conditional UPDATE for exactly-once semantics. One JVM wins the `UPDATE ... WHERE policyTriggered = false`, the other gets 0 rows. Simple, battle-tested, and wrong for this case.

The adversarial design review caught it. The conditional UPDATE assumes the row exists — it was designed for M-of-N threshold completion where earlier events have already created the entity. But in OR mode with a single ganglion, the very first event evaluates to CREATE_CASE. No prior CONTINUE_ACCUMULATING save happened. No row exists. The UPDATE returns 0 rows, indistinguishable from "already claimed by another JVM." The most basic happy path — single event, single ganglion, fire a case — was silently broken.

The fix needed a bifurcated claim path. For new entities (storeVersion empty), save first to create the row, then claim. For existing entities (storeVersion present), claim first, then save. The second ordering matters because of a subtler problem the review also caught: save-before-claim updates `lastSignal` through `SituationContext.withDetection()`, and that refresh prevents the entity from ever expiring. Post-trigger events would keep refreshing `lastSignal` forever, defeating the expiry mechanism.

Claim-before-save for existing entities avoids this — when the claim fails, no save happens, `lastSignal` stays unchanged, and the entity expires on schedule.

The deferred removal was another review catch. I'd originally had the winner removing the entity after firing the trigger. But if the loser retries after the entity is gone, it re-creates a fresh one with `policyTriggered = false` and fires a duplicate — exactly the bug the whole mechanism was supposed to prevent. The entity now stays with `policyTriggered = true` as a guard, cleaned up by the existing expiry mechanisms.

The contract test (#17) was the other half of this branch. `InMemorySituationStore` and `JpaSituationStore` had independent test suites with ~10 overlapping tests. `AbstractSituationStoreContractTest` now defines the shared behavioral contract — 12 tests including the new claim methods. Both stores extend it, and the JPA module keeps its OCC-specific tests on top.

One consequence of the claim mechanism: `InMemorySituationStore` could no longer rely on the SPI defaults. With deferred removal, the entity stays after trigger — so the default `tryClaimTrigger` (always returns true) would let every post-trigger event re-fire. InMemory now overrides with a `ConcurrentHashMap`-based claim, and populates `storeVersion` on save so the evaluator's bifurcated path works consistently across both stores.

The design review ran four rounds and cost about $19. It was worth considerably more than that — the save-before-claim, bifurcated path, and deferred removal were all review catches that would have shipped as production bugs.
