---
layout: post
title: "casehub-ras: The API That Didn't Wrap Anything"
date: 2026-06-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [api-design, cloudevents, sealed-interfaces]
---

The RAS design spec was written on June 12. Two days later, the parent repo's layering decisions landed — and they made a quiet but significant choice: `io.cloudevents.CloudEvent` is the platform's typed CDI event envelope. No wrapper type.

The original spec had `SensoryEvent`. A custom record with `streamType`, `tenancyId`, `payload` — domain-flavoured fields that made RAS feel like it owned its input type. But `SensoryEvent` was too close to the RAS domain for a type that platform stream modules would produce. The stream modules are foundation tier. They shouldn't know what a "sensory event" is. They fire CloudEvents.

I agreed with the decision immediately. The mapping between `SensoryEvent` and `CloudEvent` was 1:1 — `streamType` is `getType()`, `tenancyId` is `getExtension("tenancyid")`, `payload` is `getData()`. The wrapper bought nothing. Removing it also fixed the dependency direction: `casehub-ras-api` depends on `casehub-platform-api` (integration → foundation), and platform stream modules have zero dependency on RAS. The earlier design had it backwards.

With that settled, the interesting design question was routing. Who decides which situation a CloudEvent belongs to — the ganglion or the engine?

Model A says the ganglion examines the event and returns a `situationId` — domain knowledge inside the detection unit. But then the engine can't provide the right `SituationContext` until the ganglion has already decided which situation it's for. Chicken and egg.

Model B says the engine owns routing. `SituationDefinition` declares which event types activate which situations, which ganglia participate (via `ChainMode`), and how to correlate. The ganglion receives a `SituationContext` and evaluates — it doesn't choose. This meant `DetectionResult` doesn't need a `situationId` at all. The ganglion reports what it found; the engine knows which situation asked.

Model B won. It's the cleaner separation — and it makes multi-situation events straightforward. A temperature spike contributing to both "equipment-failure-risk" and "environmental-hazard" means two `SituationDefinition`s matched the event type, two calls to the ganglion with different contexts, two independent `DetectionResult`s. Single return type, no multi-result complexity.

The `ChainMode` sealed interface turned out well. Five variants — And, Or, Threshold, Sequence, Count — each carrying only its own configuration. `And` carries a `Set<String> requiredGanglia`. `Threshold` carries a `Set<String> ganglia` and a `double minConfidence` with no upper bound (a threshold of 2.0 means "I need the equivalent of two strong signals"). The compiler enforces exhaustive pattern matching. No default branch, no unrepresentable states.

Three review rounds caught real issues. The first found that `casehub-engine-api` had no business on the API module — `CaseTriggerConfig` deliberately uses string identifiers (`caseNamespace`, `caseName`, `caseVersion`) to avoid coupling every ganglion implementor to the engine's type graph. The second removed `Ganglion.correlationWindow()` entirely — the correlation window is a situation-level concern in `SituationDefinition`, not a ganglion-level one. The third was a wording fix.

The `compact()` default method on `Ganglion` came from a conversation about infinite windows. Some situations never expire — a service lifecycle monitor that watches equipment forever. The ganglion needs a way to keep its accumulated state manageable. `compact()` returns `Uni<SituationContext>` — the ganglion decides what to keep. A temperature monitor might keep only the last N readings. A Bayesian network might maintain running probabilities without raw events. The API doesn't prescribe what compaction means.

`SituationContext` handles out-of-order events — `min(firstSignal, eventTime)` and `max(lastSignal, eventTime)` instead of unconditional assignment. Event-time semantics, not wall-clock. `accumulatedEvidence` was removed after review — each `DetectionResult` carries its own evidence map, and `putAll` would silently overwrite keys when two ganglia use the same evidence key name. The forensic trail lives in the individual detections. When the runtime needs a merged view for case creation, it constructs one with whatever collision policy it chooses.

The `persistence-memory` module follows the platform convention: `@ApplicationScoped @Alternative @Priority(1)`, `ConcurrentHashMap`-backed, activates by classpath presence. The `SituationStore` SPI is narrow — find, save, remove, removeExpired. The runtime owns the correlation logic; the store just persists context.

56 tests across three modules. The API types, the in-memory store, and the test fixtures all compile and integrate. casehub-ras has its foundation — the types and SPIs that every subsequent epic builds on.
