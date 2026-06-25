---
layout: post
title: "The Deferrals That Took an Afternoon"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [yaml, trigger-policy, compaction]
---

Three issues sat in the casehub-ras backlog since Epic 2 closed: a YAML provider for situation definitions (#13), ANTI signal handling in the trigger policy (#15), and compact() invocation for persistent situations (#12). All S-scale, all deferred because the runtime had higher priorities. I picked them up as a batch on one branch.

## YAML that doesn't fight the domain model

The interesting design question with YamlSituationDefinitionProvider wasn't parsing — it was representing ChainMode. Five sealed variants (And, Or, Threshold, Sequence, Count) with different shapes. Jackson polymorphism would have been a fight. Instead I used SnakeYAML to parse to maps and constructed the domain objects manually — a `type` discriminator in the YAML, a switch expression in the provider. The whole thing reads clearly and doesn't need annotation gymnastics.

The provider returns an empty list when the classpath resource is absent, so it coexists with programmatic providers without ceremony. No special activation, no feature flags — if the YAML is there, the situations load.

## ANTI as counter-evidence

The trigger policy already filtered ANTI signals alongside NOISE via `isAtLeast(WEAK)`. The question was whether ANTI should actively work against the accumulated signal. In Threshold mode, subtraction is natural — ANTI detections reduce the confidence sum, which can pull a previously-satisfied threshold below its minimum. In the other modes (And, Or, Sequence, Count), the question is binary — did the ganglion fire? — so ANTI stays filtered there.

The implementation was a three-line change: widen the filter from `isAtLeast(WEAK)` to `isAtLeast(ANTI)`, then negate the confidence for ANTI signals. The tests were more interesting than the production code — verifying that ANTI from a non-participating ganglion is correctly ignored, that sufficient positive evidence overcomes ANTI, and that ANTI can pull a satisfied threshold back below minimum.

## Compaction as delegation

For compact() invocation, the cleanest approach was the simplest: call compact() on every referenced ganglion after `CONTINUE_ACCUMULATING` for persistent situations. No threshold, no timer — just invoke it every time. The ganglion's default implementation is a no-op, so the cost is negligible. The ganglion that actually needs compaction (DroolsGanglion, eventually) decides what to compact and when it matters. The evaluator just triggers; the domain logic stays in the plugin.

This batch made one thing clear: the deferred items from Epic 2 were genuinely small. The runtime's architecture held up — each change slotted into the existing seams without forcing anything open.
