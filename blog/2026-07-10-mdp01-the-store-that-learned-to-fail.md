---
layout: post
title: "The Store That Learned to Fail"
date: 2026-07-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [drools, observability, error-handling, h2mvstore]
---

# CaseHub RAS — The Store That Learned to Fail

**Date:** 2026-07-10
**Type:** phase-update

---

## What I was trying to achieve: make ReliableDroolsSessionStore safe to run in production

The drools-reliability module was experimental — it persisted KieSession state to H2MVStore so sessions could survive JVM restarts, but it had no error handling, no metrics, and no health checks. If the storage backend failed, exceptions propagated untyped through the call stack. If the store shut down gracefully, `dispose()` deleted the persisted data it was supposed to preserve.

The whole layer is temporary. It'll be replaced when journal-based reliability or clustered session sharing ships. The hardening had to be pragmatic — enough observability to run in production, not enough investment to regret when it's replaced.

## What we believed going in: close() means nothing works

The natural test for "what happens when storage fails" is to close the store and call the method. We assumed `StorageManager.close()` would make everything fail — reads, writes, session creation.

It doesn't. H2MVStore's `MVMap.get()` serves from its page cache even after close. Only write operations hit `checkNotClosed()` and throw. The asymmetry is invisible until you write a test that expects a read failure and it silently passes.

## Three failure modes, three responses

The design review — which ran three rounds and raised 14 issues before approving — forced us to think carefully about which failures are fatal and which aren't.

**Storage reads fail** — can't determine whether a persisted session exists. Fatal. We wrap in `DroolsSessionStoreException` and let it propagate. The caller can't make safe decisions without knowing the state.

**Storage writes fail** — session was created successfully, it's in the hot cache, it works for this request. Just not durable. We log the error, increment a counter, and continue. The detection completes. The session works but won't survive a restart.

**Recovery fails** — corrupt session ID mapping, version mismatch, anything that prevents rebuilding from persisted state. We already handled this before hardening — log a warning, create a fresh session. The counter is new.

The review caught something I'd missed: `DroolsGanglion.detect()` calls `computeIfAbsent` *outside* its existing try-catch block. A `DroolsSessionStoreException` would propagate uncaught through the evaluator. We added a try-catch with defensive cleanup — if `remove()` also fails (because the same broken storage backend), the cleanup exception is added as suppressed. The original cause survives.

## The dispose() trap

The design review's most consequential finding: `@PreDestroy` was calling `dispose()` on every cached session. Per the garden entry we already had (GE-20260706-d02c71), `dispose()` on a reliable KieSession deletes its persisted data from H2MVStore. Every graceful restart was destroying exactly the data it was meant to preserve.

The fix: `destroy()` clears the hot cache and logs. That's it. No `dispose()`, no `StorageManager.close()` — the latter is a static singleton that breaks dev-mode restarts because the closed instance can't be reopened.

Sessions survive. Recovery works. The `restartSurvival` test was already passing because it used `clearHotCacheForTest()` (crash simulation), not `destroy()` (graceful shutdown). We'd been testing the crash path correctly and the graceful path wrong.

## What it is now

`ReliableDroolsSessionStore` emits seven Micrometer metrics — counters for session lifecycle events, a gauge for active cache size, a timer for `computeIfAbsent` with outcome tags. All optional via `Instance<MeterRegistry>` — the store works without Micrometer on the classpath.

A `@Readiness` health check probes the `StorageManager` and reports active session count. `SituationEvaluator.runDetection()` now wraps each ganglion call individually — one ganglion's storage failure doesn't kill the evaluation for others that use in-memory stores or ephemeral sessions.

The dependency strategy came from an existing protocol (PP-20260604-88f660): library JARs depend on `micrometer-core` and `microprofile-health-api`, not Quarkus extensions. The consuming app provides the extensions that wire up Prometheus scraping and `/q/health`.

The temporary store now fails in ways you can see, measure, and respond to. When it's replaced, the metrics names and health check contract will carry forward — the observability investment outlasts the implementation.
