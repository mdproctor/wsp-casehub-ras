---
layout: post
title: "Closing the loop"
date: 2026-08-06
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [feedback-loop, detection-tuning, naivebayes, suppression, design]
series: issue-40-ras-feedback-loop
---

RAS has been open-loop since day one. It detects, triggers, forgets. A case gets created, someone investigates, dismisses it as noise — and RAS fires the exact same trigger next time. No suppression, no learning, no calibration against what actually happened.

I'd been circling this problem since writing the original issue. The four mechanisms are obvious enough: suppress dismissed noise, drift thresholds from outcome ratios, recalibrate NaiveBayes priors from ground truth, surface detection quality metrics. The interesting question was where to put the seams.

We started from the integration point. The platform already provides `CaseOutcomeEvent` and `CaseOutcomeObserver` in engine-api — I hadn't realised until we traced `DefaultCaseTrigger.fire()` through to the outcome lifecycle. The case file snapshot carries `situationId`, `correlationKey`, and `tenancyId` back from the case — same values RAS embedded at trigger time. No new correlation storage needed. The ingestion path writes itself.

The architecture settled on three layers: ingestion (outcome recording), analysis (statistics), application (what to do about it). Two SPIs at the application layer — `SuppressionStrategy` for per-event real-time decisions, `FeedbackTuningStrategy` for batch threshold and prior adjustment. The split came from the design review: a single `FeedbackStrategy` combined fundamentally different lifecycles. Suppression runs on every event evaluation; tuning runs every five minutes. Mixing them in one interface meant implementors had to think about both.

The design review changed three things that matter. First, suppression moved from `DefaultRasTriggerPolicy` into `SituationEvaluator`. The policy is a pure function of `(SituationContext, SituationDefinition)` with zero dependencies — injecting an `OutcomeLedger` there would contaminate every policy test. Suppression is a pre-detection concern: skip the ganglia entirely when a correlation key is within its cooldown. Second, all feedback-derived mutable state went into a dedicated `FeedbackState` component rather than on `SituationDefinitionRegistry`. The registry uses volatile immutable snapshots; feedback overrides change every five minutes. Different mutation disciplines, different components. Third, `NaiveBayesGanglion`'s adjusted priors must be in log-space. The ganglion operates in log-space internally — its constructor computes `logPriors = priors.map(Math::log)`. Injecting raw priors would silently corrupt every posterior calculation. `FeedbackState.applyPriorOverride()` converts at the boundary; the ganglion sees log-space priors or nothing.

Advisory mode is the default. `tuningEnabled: false` on `FeedbackConfig` gives suppression and metrics without automatic parameter adjustment. A deployment has to explicitly opt into self-tuning per situation definition — which is the right default for a system that modifies its own detection parameters.

Foundation implementation is in place: the types and SPIs in api/, `InMemoryOutcomeLedger` as the `@DefaultBean` fallback, `FeedbackState` with tenant-scoped overrides and input validation, both default strategies with Laplace-smoothed prior blending. Six tasks remain — NaiveBayes integration, evaluator wiring, the ingestion and batch pipeline, YAML parsing, JPA persistence, and docs.
