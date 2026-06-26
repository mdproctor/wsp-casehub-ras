---
layout: post
title: "Two Ways to Know What You're Looking At"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [naive-bayes, ganglion, detection, api-design]
series: issue-3-java-switch-ganglion
---

*Part of a series on [#3 — JavaSwitchGanglion](https://github.com/casehubio/casehub-ras/issues/3). Previous: [The Deferrals That Took an Afternoon](2026-06-25-mdp01-the-deferrals-that-took-an-afternoon.md).*

Epic 3 adds two built-in Ganglion implementations to casehub-ras. Both are pure Java — zero external dependencies. The interesting part isn't what they do (detect signals from CloudEvents), it's how differently they approach the same interface.

## The switch statement as a first-class detection strategy

JavaSwitchGanglion is an abstract class in the api/ module. The developer subclasses it, overrides `evaluate()`, and writes Java pattern matching. That's it. `detect()` is final — it wraps `evaluate()` in a `Uni` and returns. The helper methods (`detected()`, `weak()`, `noise()`, `anti()`) auto-embed the ganglionId to eliminate a class of wiring mistake.

The placement in api/ was deliberate. A consumer building a simple threshold detector shouldn't need the full runtime on their classpath. Same rationale as `AbstractList` in java.util — a convenience adapter that lives where the SPI does.

The design decision that took the most thinking was whether to parameterise on a payload type `<T>`. I decided against it. A ganglion handling multiple event types with different payloads forces `<T>` to `Object` and you're back to casts. Headers-only detection makes `<T>` ugly (`Void`). Deserialization is application-specific — the one line of Jackson or Protobuf code belongs in the developer's `evaluate()`, not in a framework generalization that introduces format assumptions.

## Bayes in log space

NaiveBayesGanglion sits in runtime/ — a concrete class configured with prior probabilities, per-feature likelihood tables, and a feature extraction function. The Bayesian math is fixed; the developer provides the probabilistic model as data. Same pattern as DroolsGanglion (configured with DRL rules, not subclassed).

The implementation works in log-space throughout. Multiplying many small probabilities underflows to zero in linear space. In log space, `log(0.01) + log(0.05) + ...` stays finite. Log-sum-exp normalization converts back when you need actual posteriors. With the likelihood validation rejecting zero values at config time, the log-posteriors can never reach negative infinity — the math stays clean.

The state model is a `double[]` per situation instance, keyed by `(situationId, correlationKey, tenancyId)`. Fixed-size regardless of observation count. Each `detect()` call updates the array in-place and returns the current posterior as confidence.

## compact() as a real mechanism

The Threshold ChainMode interaction was the most architecturally significant finding during spec review. NaiveBayes returns running posteriors — each one subsumes all prior evidence. Without compaction, five observations with posteriors [0.2, 0.3, 0.4, 0.55, 0.72] sum to ~1.97 in a Threshold chain when the actual state is 0.72.

compact() was the fix. After each `CONTINUE_ACCUMULATING` for persistent situations, the evaluator calls compact(). NaiveBayesGanglion replaces all its detections with the last one — the one with the most complete posterior. Threshold then sees a single value: the current state.

One subtlety: the compact implementation uses processing order (last in the detection list), not eventTime ordering. The runtime's `@ObservesAsync` provides no temporal ordering guarantee — two events arriving close together may be serialized arbitrarily by the striped lock. The Bayesian update is commutative, so the internal state is correct regardless. But the detection with the latest eventTime isn't necessarily the one computed last. The last element in the list always is.

## ANTI as counter-evidence

The signal mapping includes an optional `antiThreshold`. When the target outcome's posterior drops below it, the ganglion emits ANTI with confidence `1.0 - targetPosterior`. In a Threshold chain, this subtracts from the accumulated sum — a NaiveBayes ganglion can now actively work against a situation, not just fail to support it.

## The NaN that passes all checks

The spec review surfaced a pre-existing gap in DetectionResult: `confidence < 0.0 || confidence > 1.0` lets NaN through because IEEE 754 NaN comparisons are always false. A ganglion producing NaN — through division errors, log underflow, anything — silently corrupts Threshold sums. The fix was one line (`Double.isNaN(confidence) ||`), but the same IEEE 754 quirk meant every numeric validation in the config types needed the same treatment. Priors, likelihoods, thresholds — all had bounds checks that NaN sailed past.

NaiveBayesGanglion is the first ganglion to implement compact() as more than a no-op. That's worth noticing — it validates the compact API's design intent. The mechanism was there since Epic 2; it took a stateful ganglion with running posteriors to demonstrate why it exists.
