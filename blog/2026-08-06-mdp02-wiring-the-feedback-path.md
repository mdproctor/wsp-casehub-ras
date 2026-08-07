---
layout: post
title: "Wiring the feedback path"
date: 2026-08-06
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [feedback-loop, naivebayes, suppression, threshold-drift, tenant-scoping]
series: issue-40-ras-feedback-loop
---

Continues from [Closing the loop](2026-08-06-mdp01-closing-the-loop.md).

The foundation types are in place. This session wired them into the existing runtime — the NaiveBayes ganglion, the registry, and the evaluator.

The NaiveBayes integration raised a question I hadn't thought through during the design: when should feedback-adjusted priors take effect? Not on every `detect()` call — the ganglion persists running log-posteriors in `GanglionStateStore`, and overwriting them mid-computation would corrupt the Bayesian update chain. The answer is new instances only. When `stateStore.load()` returns empty — no prior computation exists for this situation/correlation/tenant tuple — the ganglion queries `FeedbackState` for tenant-scoped adjusted priors instead of using the config defaults. Existing computations are untouched. The feedback loop adjusts the starting point for future situations, not the trajectory of current ones.

`outcomeGroundTruth` bridges two vocabularies that I'd been treating as one. Case outcome labels (dismissed, escalated, confirmed-fraud) are domain terms chosen by the case handler. NaiveBayes outcomes (fraud, legitimate) are statistical classes chosen by the ganglion designer. They're related but not the same — multiple case labels can map to one Bayes outcome, and some labels map to none. The mapping lives on `GanglionDescriptor.NaiveBayes` because that's where both vocabularies meet, and it's validated at startup so a typo like `escalated: froud` fails loudly rather than silently producing skewed priors.

The evaluator integration had a spec error that Claude caught while tracing the control flow. The design spec said suppression should `return true` from `processEvent()`. But `true` in that method means "terminated" — the `evaluate()` caller cleans up locks and event buffers for that situation instance. Suppression isn't termination. It's "skip this event, the situation is still active." The correct return is `false`. The kind of error that reads right in isolation but breaks when you follow the actual call chain.

The threshold override pattern keeps the policy pure. `DefaultRasTriggerPolicy` evaluates `(SituationContext, SituationDefinition)` — zero dependencies, zero mutable state. Instead of injecting `FeedbackState` into the policy, the evaluator constructs an effective `SituationDefinition` with the adjusted `minConfidence` via `withChainMode()`, and passes that to the policy. The policy sees an immutable definition with a different number. It doesn't know or care that the number came from feedback. Same pattern as expression-compiled correlation keys — the evaluator resolves, the downstream component sees the result.
