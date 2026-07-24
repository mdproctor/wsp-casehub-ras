---
layout: post
title: "Retiring the reactive costume"
date: 2026-07-24
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [reactive, mutiny, virtual-threads, api-design]
series: issue-384-retire-reactive
---

RAS was wearing a reactive costume. Every one of its 226 Uni references fell into
exactly two categories: `Uni.createFrom().item(synchronousResult)` on the provider
side, `.await().indefinitely()` on the consumer side. Zero actual reactive
composition anywhere — no flatMap chains, no backpressure, no async pipelines.

I found this by tracing from the SPIs outward. `Ganglion.detect()` returns
`Uni<DetectionResult>`, but every implementation — `ExpressionRulesGanglion`,
`NaiveBayesGanglion`, `DroolsGanglion`, `JavaSwitchGanglion` — computes the
result synchronously and wraps it. The callers — `SituationEvaluator`,
`SituationExpiryJob` — immediately unwrap with `.await().indefinitely()`. Same
pattern across `SituationStore` (9 methods), `GanglionStateStore` (4 methods),
`RasTriggerPolicy`, `CaseTrigger`, `SituationSource`, `OrphanedResourceCleaner`.
Twenty-one methods across seven interfaces, all ceremony.

The question was whether RAS should have had dual-stack parity — blocking and
reactive variants side by side, like qhorus and platform. It shouldn't have. RAS
uses blocking JPA with `EntityManager` and `@Transactional`. It processes CDI
events synchronously. There was never a reactive I/O path to justify the Uni
signatures. The right design from day one was blocking returns.

The fix was straightforward: `Uni<T>` → `T`, `Uni<Void>` → `void`, across all
SPIs and every implementation. The one interesting edge is `DefaultCaseTrigger.fire()`,
which wraps `CaseHub.startCase()` — that returns `CompletionStage<UUID>` from
engine-api. For now it's `.toCompletableFuture().join()`. When parent#381 retires
the engine's `CompletionStage` returns, the `.join()` disappears and the method
becomes a direct call.

We changed 52 files in RAS, then fixed three consumer repos in the same pass —
ops, desiredstate, and iot. The consumer impact was minimal: `JavaSwitchGanglion`
subclasses override `evaluate()` which was always blocking, so they compiled
without changes. Only the repos implementing `RasTriggerPolicy`
(`IoTSuppressionTriggerPolicy` in iot) or calling SPI methods directly needed
updates. Mutiny is gone from every POM.

The broader point: RAS is now the first foundation-tier repo fully blocking under
ADR-0005. The remaining repos in parent#384 — platform, qhorus, neocortex, and
the rest — have genuine dual-stack to delete, not costume to remove.
