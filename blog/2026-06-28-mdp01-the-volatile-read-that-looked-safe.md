---
layout: post
title: "The volatile read that looked safe"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [jmm, drools, jpa, concurrency]
series: issue-11-runtime-enhancements
---

*Part of a series on [#11 — runtime enhancements](https://github.com/casehubio/casehub-ras/issues/11). Previous: [The deferrals that took an afternoon](2026-06-25-mdp01-the-deferrals-that-took-an-afternoon.md).*

This branch closed four issues — three with code, one that turned out to already be done. The interesting one was DroolsGanglion hot reload, where a straightforward-looking concurrency design hid a JMM ordering bug that would have been invisible on every x86 machine we'd ever test on.

## The reload mechanism

DroolsGanglion holds a `KieBase` — compiled Drools rules. Hot reload means swapping it at runtime. The obvious approach: make it `volatile`, compile new rules, swap. New sessions use the new KieBase; existing long-lived sessions drain naturally.

Drain doesn't work for persistent situations. A persistent situation (no correlation window) accumulates events indefinitely. If the rules have a bug, drain means the situation processes every future event under the buggy rules with no self-correcting mechanism. The session never closes — it just keeps accumulating wrong.

The fix is lazy invalidation. DroolsGanglion tracks a generation counter alongside the volatile KieBase. When `reload()` swaps the KieBase, it increments the generation. On the next `detect()` call, the evaluator checks whether the session's recorded generation matches the current one. If stale, it disposes the session and creates a new one from the current KieBase — right there, inside the existing per-key lock. No race, no SPI expansion, no operational "pause ingestion" procedures.

## The read that looked safe

The generation counter introduced two volatile fields: `kieBase` (data) and `reloadGeneration` (flag). The reload writes kieBase first, then increments the flag. The natural-looking read in `detect()`:

```java
KieBase currentBase = this.kieBase;       // read data
long currentGen = this.reloadGeneration;  // read flag
```

This is wrong. The JMM permits a valid synchronization order where the reader sees the old kieBase and the new generation — the reader's data read happens before the writer's data write in the synchronization order, while the flag read correctly sees the flag write. The result: a session created from the old KieBase, recorded at the new generation, never invalidated.

On x86 this can't happen. Total Store Order means loads are never reordered relative to other loads. But the JMM specification allows it, and ARM hardware could expose it. The fix is the standard acquire-release pattern — read the flag first:

```java
long currentGen = this.reloadGeneration;  // flag first (acquire)
KieBase currentBase = this.kieBase;       // data second
```

Now the happens-before chain works: `write(kieBase)` hb `write(reloadGeneration)` hb `read(reloadGeneration)` hb `read(kieBase)`. Two lines swapped, zero cost, JMM-correct on all architectures.

## The buffer that exposed a lock bug

The event reordering buffer (#16) introduced batch event processing — multiple events released from the buffer in timestamp order, processed in a loop. This surfaced a pre-existing issue: `locks.remove(key)` inside the `CREATE_CASE` branch. With single events, the lock removal was harmless — the synchronized block exits immediately. With a batch, the loop continues after the lock is gone. A concurrent `evaluate()` creates a new lock for the same key and enters its own synchronized block.

The fix: `processEvent()` returns a boolean termination signal. The loop breaks on `CREATE_CASE` or `DISCARD`. Lock and buffer cleanup move from inside the pipeline to the caller — `evaluate()` manages lifecycle, `processEvent()` handles domain logic.

## JPA store

The persistence-jpa module (#14) is the most conventional piece — a single entity with JSONB detections, blocking JPA wrapped in Uni for the reactive SPI. The one gotcha worth remembering: `columnDefinition = "jsonb"` on a String field only controls DDL. Hibernate still calls `setString()` for parameter binding, and PostgreSQL rejects VARCHAR parameters on JSONB columns. `@JdbcTypeCode(SqlTypes.JSON)` is required — it tells Hibernate to bind via PGobject instead.
