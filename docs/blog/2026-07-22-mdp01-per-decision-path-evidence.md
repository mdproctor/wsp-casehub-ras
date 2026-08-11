---
layout: post
title: "Per-Decision-Path Evidence: When Cross-Cutting Is the Wrong Abstraction"
date: 2026-07-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [ganglion, evidence, expression-rules, naive-bayes, sealed-types]
series: issue-51-per-rule-evidence-templates
---

Ganglion-level evidence templates landed last session — extract fields from every CloudEvent regardless of which rule matched or which outcome won. The obvious next question: what about evidence that depends on the detection decision?

The tempting generalisation is per-signal evidence. Key it on `DetectionSignal` (DETECTED, WEAK, NOISE, ANTI) since every ganglion produces a signal. One cross-cutting `Map<DetectionSignal, Map<String, ExpressionEvaluator>>` on `GanglionDescriptor`, done.

I rejected it. Per-signal is too coarse for both variants.

For ExpressionRules, multiple rules can produce the same signal. Three rules all producing DETECTED for different conditions — high temperature, high pressure, high vibration — each need different evidence fields. Per-signal can't distinguish them.

For NaiveBayes, the signal mapping maps one target outcome to thresholds. All non-target outcomes produce NOISE or ANTI indistinguishably. "Normal won" and "malfunction won" both produce NOISE, but you want the error code when malfunction wins and the baseline reading when normal wins. Per-signal loses this.

The right abstraction: each variant's structural decision unit carries evidence templates. For ExpressionRules, that's the rule. For NaiveBayes, that's the outcome. Rules and outcomes are fundamentally different structures — no common type makes sense. Each ganglion evaluates its matched path's templates inside `detect()`, before the ganglion-level wrapper adds its cross-cutting templates on top.

Three-layer merge: automatic evidence (`matchedRuleIndex` / `posterior` + `features` + `winningOutcome`) → per-decision-path → ganglion-level. Each layer overwrites the previous on key clash.

The NaiveBayes side also gained `winningOutcome` as automatic evidence — the outcome with the highest posterior, which was implicit before. Useful diagnostic field regardless of per-outcome templates.

The implementation touches the full vertical: API types in `api/`, ganglion implementations in `runtime/`, YAML parsing, registry compilation with collision warnings, and E2E integration tests. All changes are breaking — pre-release, no convenience constructors. The sealed interface's exhaustive pattern match means adding a third descriptor variant in the future will surface every place that needs per-decision-path evidence handling as a compile error.
