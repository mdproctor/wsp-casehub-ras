---
layout: post
title: "The Store That Learned to Remember"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [drools, drools-reliability, persistence, spi-design, cdi]
series: issue-7-drools-session-store-persist
---

*Part of a series on [#7 — DroolsSessionStore: persistent implementation for restart survival](https://github.com/casehubio/casehub-ras/issues/7).*

# CaseHub RAS — The Store That Learned to Remember

**Date:** 2026-07-06
**Type:** phase-update

---

## What I was trying to achieve: persistent Drools session state across restarts

DroolsGanglion runs long-lived KieSession instances for CEP detection — sliding windows, temporal correlations, accumulated event history. When the JVM restarts, all of that state vanishes. If a rule says "alert when 3 failures occur within 1 hour" and two failures have already arrived, a restart silently resets the counter.

Issue #7 had been waiting on Epic 4 (the DroolsGanglion itself) to ship. That's done now. The blocker was gone.

## What I believed going in: serialization would be the hard problem

The original issue description said "KieSession cannot be reliably serialized in Drools 10" and pointed at a fact-replay model. I expected the bulk of the work to be figuring out how to capture and replay event history through our own journaling layer.

That turned out to be wrong. Drools already solved this with `drools-reliability` — an experimental module that intercepts every fact insert/delete on the ObjectStore, persists `StoredObject` entries to a pluggable `Storage<K,V>` backend, and replays them into a fresh session on recovery. The pseudo clock gets advanced to each event's timestamp during replay. It handles activation deduplication so rules that already fired don't re-fire.

The real question became: how to integrate it cleanly behind the existing `DroolsSessionStore` SPI so it can be ripped out and replaced later.

## The SPI that grew from Map semantics

The old DroolsSessionStore had the classic three-method pattern: `get()`, `put()`, `remove()`, each taking four strings (ganglionId, situationId, correlationKey, tenancyId). The caller controlled the lifecycle — get a session, use it, put it back.

That shape doesn't work when the store needs to control session creation. With drools-reliability, a session must be configured with `PersistedSessionOption` at birth. The caller can't just create one and hand it over.

I started thinking about what `Map` does. `computeIfAbsent` is the right semantic — return the existing value if present, create and store on miss. The store decides how to create. The caller provides the materials (KieBase, config). We promoted the four-string tuple to a `DroolsSessionKey` record and landed on:

```java
KieSession computeIfAbsent(DroolsSessionKey key,
                           KieBase kieBase,
                           KieSessionConfiguration config,
                           long generation);
```

The `generation` parameter was the design review's contribution, not mine. The original spec had a `removeAll(String ganglionId)` for hot reload invalidation. The reviewer caught that this directly contradicts the hot reload spec from two weeks ago — the spec had explicitly rejected bulk-remove because it races with in-flight `detect()` calls under SituationEvaluator's per-key serialization. Lazy generation comparison inside `computeIfAbsent` replaced it. Sessions are only disposed by the thread holding the per-key lock.

## Four gotchas from a module nobody documented

The implementation itself was straightforward once the SPI was right. `InMemoryDroolsSessionStore` got a `StampedSession` wrapper for generation tracking. `DroolsGanglion` lost its `sessionGenerations` map entirely — that's a store concern now. The ganglion kept only the volatile `reloadGeneration` field for JMM acquire-release ordering.

The new `ReliableDroolsSessionStore` lives in its own module (`drools-reliability/`) with plain `@ApplicationScoped` — beats InMemory's `@DefaultBean` by classpath presence. Two-layer cache: hot ConcurrentHashMap plus persistent H2MVStore session ID mapping.

Where it got interesting was the drools-reliability API itself. Four things the documentation doesn't tell you:

**Three factories, not one.** `StorageManagerFactory.get("h2mvstore")` is the documented entry point. But creating a persisted session also needs `ReliableGlobalResolverFactory.get("core")` and `SimpleReliableObjectStoreFactory.get("core")` or the KieService loader NPEs on a null tag. The pattern exists in Drools' own test utilities but nowhere else.

**dispose() deletes persistent data.** `ReliableStatefulKnowledgeSessionImpl.dispose()` calls `removeStoragesBySessionId()` internally. Standard `KieSession.dispose()` means "release resources" — nobody expects it to delete the persisted facts. For test restart simulation, you need to null the in-memory reference without calling dispose.

**FULL strategy is broken.** `PersistenceStrategy.FULL` is a valid enum value offered by the API. All its tests are disabled with serialization errors. Only `STORES_ONLY` works in 10.1.0.

**Session ID counter doesn't survive test restarts.** `TestableStorageManager.restart()` simulates the storage restart but doesn't refresh the session ID counter. New sessions get ID 0, which doesn't match the saved ID from before "restart". You need `ReliableRuntimeComponentFactoryImpl.refreshCounterUsingStorage()` — a call that happens automatically in a real JVM restart but not in the test harness.

All four went into the garden.

## Where this leaves us

DroolsSessionStore now has a clean separation: the SPI knows nothing about persistence strategy, the InMemory implementation is a volatile cache, and the Reliable implementation adds restart survival via H2MVStore. Swap it out by changing the classpath.

This is explicitly temporary. I have separate work underway on a journal-based reliability approach that doesn't depend on drools-reliability's experimental API. But for now, long-lived CEP sessions survive restarts, and the encapsulation means replacing the mechanism later won't touch any consumers.

The `desiredstate#70` dependency — dropping the runtime dep from ras-adapter now that the SPI types live in api — is still waiting on publishing the artifacts from last session's work. That's the immediate next step after this branch closes.
