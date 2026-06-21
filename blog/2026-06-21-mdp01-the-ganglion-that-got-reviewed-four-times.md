---
layout: post
title: "casehub-ras: The Ganglion That Got Reviewed Four Times"
date: 2026-06-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [drools, cep, api-design, ganglion]
series: issue-4-drools-ganglion
---

*Part of a series on [#4 — DroolsGanglion](https://github.com/casehubio/casehub-ras/issues/4). Previous: [The API That Didn't Wrap Anything](2026-06-20-mdp01-the-api-that-didnt-wrap.md).*

The first ganglion implementation landed. `DroolsGanglion` sits in `ras-drools/` — an optional module that activates by classpath presence and brings Drools CEP to the detection layer: sliding time windows, temporal operators, event correlation. The classic kie-api, not Rule Units.

The design question from the original spec was "stateful KieSession per situation, or shared session?" The answer turned out to be both — configurable per ganglion instance. `LONG_LIVED` mode keeps a session alive across `detect()` calls; `EPHEMERAL` creates and disposes one per call. Different situations have different statefulness needs, and forcing one model would have been wrong for half the use cases.

The interesting part was the review cycle. The spec went through four passes, each surfacing issues the previous one missed. The first caught the Drools 10 API coordinates — `drools-engine` doesn't exist at 10.1.0. I'd listed it as a dependency because the documentation references it, but the artifact was only introduced post-10.1.0 as a consolidation. The correct dependency is `drools-model-codegen`, which transitively pulls in the entire compilation and runtime chain. `KieHelper` is gone too — removed from Drools 10 entirely, no deprecation notice, just absent.

The second pass found the real architectural gaps. `KieSession` isn't thread-safe, so the spec needed an explicit concurrency contract: the runtime serializes all operations per situation key. Pseudo clock can't go backwards, so out-of-order events need monotonic dispatch — a precondition on the caller, not the ganglion. And `EPHEMERAL` mode is genuinely incompatible with temporal operators — a single-event session can't evaluate `this after[0s, 10m] $t1`. We scoped it explicitly to non-temporal rules.

The third pass caught a data corruption bug that would have hit the primary use case. `DroolsSessionStore` was keyed by `(situationId, tenancyId)`, but multiple ganglia can share a store while processing the same situation under `ChainMode.And`. Without `ganglionId` in the key, one ganglion's session overwrites another's. Four of five chain modes reference multiple ganglia — this wasn't an edge case. The same pass caught that `close()` needed to return `Uni<Void>` per the platform's reactive SPI protocol, not void.

The fourth pass found that inserting the raw `CloudEvent` as a fact in `LONG_LIVED` mode creates unbounded memory growth. `CloudEvent` is an external interface — no `@role(event)`, no `@expires`. Regular facts in STREAM mode never auto-expire. The fix: retract the CloudEvent after `fireAllRules()`. It's per-call metadata for rule evaluation, not a temporal fact for cross-call correlation. Extracted domain objects handle their own lifecycle via `@expires` declarations in DRL.

The lifecycle gap was the most consequential finding. When the engine terminates a situation (case created or discarded), it calls `SituationStore.remove()`. But no method existed on `Ganglion` to notify it — meaning `DroolsSessionStore` would hold orphaned `KieSession` objects forever. This isn't DroolsGanglion-specific; any stateful ganglion (Bayesian, LLM with cached context) would leak the same way. We added `close()` as a default method on the `Ganglion` SPI — a breaking change to Epic 1's output, but the migration is mechanical and the leak would have been real from day one.

The detect() flow ended up with eleven steps: session get/create, channel register, clock advance with ordering guard, CloudEvent insert, extractor facts, fire rules, retract CloudEvent, collect result, unregister channel, session store/dispose, return. Error handling disposes corrupted sessions — Drools has no transactional rollback, so the only safe option after a failed `fireAllRules()` is to discard the session and start fresh on the next call.

What makes the module work as an optional building block is what it doesn't own. `DroolsGanglion` is a plain class, not a CDI bean. Consumers produce instances via `@Produces` methods — one ganglion or many, one global or many scoped. The object extractor SPI lets consumers decompose CloudEvents into domain facts without touching ganglion internals. The session store SPI gives a path to evolve persistence without changing the detection logic. The pieces compose without coupling.
