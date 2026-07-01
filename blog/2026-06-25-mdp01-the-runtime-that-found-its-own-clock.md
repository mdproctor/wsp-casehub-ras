---
layout: post
title: "The Runtime That Found Its Own Clock"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [runtime, chain-modes, concurrency, identity-model]
series: issue-2-ras-runtime
---

# CaseHub RAS — The Runtime That Found Its Own Clock

**Date:** 2026-06-25
**Type:** phase-update

---

## What I was trying to build: the coordination layer

Epic 2 is the centre of the RAS — the part that ties everything together. CloudEvents arrive from platform stream modules. Ganglia detect patterns. But something needs to sit between those two halves: receive the events, find the right situations, dispatch to the right ganglia, accumulate the detections, evaluate whether the chain is satisfied, and create a case when it is.

I wanted three layers. RasEngine as the CDI observer — thin, just routing. SituationEvaluator as the pipeline per situation — the real work. CaseTrigger as the bridge to casehub-engine. Clean separation, each testable without CDI.

## The identity model that Epic 1 got wrong

Epic 1 proposed compositing the situation instance ID from the definition identifier plus a correlation key: `"equipment-failure-risk:machine-42"`. That was a design error I didn't catch until brainstorming Epic 2.

The problem is opacity. A composite string forces every consumer — the store, the session store, log output, metrics — to parse a convention they didn't author. If you want all instances of a definition, you prefix-scan. If the separator changes, everything breaks.

I replaced it with a three-field identity: `(situationId, correlationKey, tenancyId)`. `situationId` stays the definition-level ID. `correlationKey` defaults to `CloudEvent.getSubject()` — the entity the event concerns — or `"_singleton"` when subject is null. Breaking change across every module. The migration was mechanical, which is the point — it forced every call site to be explicit about all three fields.

## The timestamp problem I didn't see coming

The design review caught something I'd missed: `ChainMode.Sequence` is silently broken without per-detection timestamps.

Consider a sequence requiring `temp-spike` before `vibration-anomaly`. The CDI managed executor delivers events to `@ObservesAsync` with no ordering guarantee. If vibration arrives before temperature — because the executor thread pool scheduled it that way — the detections list records `[vibration, temp]`. The sequence check fails. The situation never triggers. No error, no warning.

The fix was `TimestampedDetection` — a wrapper that pairs the ganglion's `DetectionResult` with the source event's `Instant eventTime`. The ganglion produces the detection; the runtime adds the timestamp at the accumulation boundary. The Sequence evaluator sorts by `eventTime` before checking order.

I considered adding `eventTime` directly to `DetectionResult` instead. That would mean the ganglion constructs the result with a null timestamp that the runtime fills in later — a partial-construction smell. `TimestampedDetection` keeps the boundary explicit: the ganglion's output is complete, the runtime enriches it.

## The clock that wasn't mine

The spec said window expiry should use `Instant.now()` — wall-clock time. Claude implemented it that way. The first test failed.

The test used fixed timestamps from June 2026. `Instant.now()` returned the actual wall-clock time. The window expiry calculation compared `context.lastSignal` (a fixed test timestamp) against `Instant.now().minus(correlationWindow)` (real time minus five minutes). Whether the test passed depended on when you ran it.

Claude switched to event-relative time: compare `lastSignal` against `eventTime.minus(correlationWindow)`. Tests became deterministic. But it also turned out to be the better design — for batch processing or event replay, you want expiry relative to the event stream's timeline, not the processor's wall clock. The scheduled cleanup job (SituationExpiryJob) still uses `Instant.now()` for the safety-net sweep, which is correct there — it's cleaning store bloat, not making domain decisions.

## The chain mode evaluator

`DefaultRasTriggerPolicy` pattern-matches on the sealed `ChainMode` interface — five variants, exhaustive, no default branch. The compiler enforces completeness. If someone adds a sixth variant, this class won't compile until it handles it.

The signal threshold — `isAtLeast(DetectionSignal.WEAK)` — filters across all modes. NOISE and ANTI detections don't contribute to chain satisfaction. NOISE is nothing meaningful; ANTI is counter-evidence. Both are recorded in the context (the forensic trail is preserved), but neither advances the chain. Custom policies can handle ANTI differently through the `RasTriggerPolicy` SPI.

## What the branch delivered

The runtime module went from an empty `pom.xml` to ten production classes and 162 tests across the reactor. The full pipeline works: CloudEvent → RasEngine → SituationEvaluator → DefaultRasTriggerPolicy → CaseTrigger. Striped locking per situation instance prevents lost updates from concurrent events. The registry validates ganglion capabilities at startup — misconfigured definitions fail fast, not at the first event.

`CaseTrigger` landed as an SPI in the api/ module, not a concrete class in runtime/. The testing argument was decisive — without it, every pipeline test would need real CaseHub subclasses on the classpath. `MockCaseTrigger` records what was triggered; the pipeline is testable without engine infrastructure.

The three-field identity model and `TimestampedDetection` both changed the api/ surface. Every downstream module — persistence-memory, ras-drools, testing — absorbed the change. 42 files touched across the project. The kind of refactoring that's cheap now and expensive later.