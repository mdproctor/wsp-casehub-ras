---
layout: post
title: "The Collection Strategy That Started With the Wrong Ordering"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [drools, detection-signal, result-collection]
---

# CaseHub RAS — The Collection Strategy That Started With the Wrong Ordering

**Date:** 2026-06-23
**Type:** phase-update

---

## What I was trying to achieve: close two small issues before moving to the RAS Runtime

Epic 4 (DroolsGanglion) shipped last session. Two trailing issues remained — #10 (three minor test gaps from the final review) and #8 (configurable result collection strategy for when multiple DRL rules fire). Both were small enough to batch onto one branch.

## What I believed going in: the collection strategy was an internal Drools concern

The core question seemed straightforward: when `fireAllRules()` produces N `DetectionResult` emissions, how do you collapse them to one? I had four strategies in mind — highest confidence, first match, last wins, accumulate — and I assumed the design would live entirely inside `ras-drools/`. An enum, a channel change, a config field. Done.

## The signal ordering argument

The first design had ACCUMULATE's signal strength ordering as DETECTED > ANTI > WEAK > NOISE. The reasoning was that ANTI is "more informative" than WEAK — a definitive counter-signal vs. an uncertain detection. That reasoning was wrong.

The review caught it: ANTI is counter-evidence (Epic 1 §4.1 says exactly this — "reduces situation confidence"). Placing it above WEAK means a WEAK+ANTI co-occurrence reports ANTI, silently squashing a legitimate positive detection at the collection level. The trigger policy — which sees the full `SituationContext.detections` list — is where competing signals should be weighed. The collection strategy's job is mechanical reduction, not semantic judgment.

Corrected ordering: DETECTED > WEAK > ANTI > NOISE. Positive detections of any strength propagate upward.

But the more interesting observation came next: if signal strength matters enough to specify in a design spec, it isn't an implementation detail of a strategy in a different module. It's a property of the signal type itself. `DetectionSignal` should carry its own ordering.

The fix was to reorder the enum constants to ascending strength and add `isAtLeast()`:

```java
public enum DetectionSignal {
    NOISE, ANTI, WEAK, DETECTED;

    public boolean isAtLeast(DetectionSignal threshold) {
        return this.ordinal() >= threshold.ordinal();
    }
}
```

Declaration order is strength order — idiomatic Java. `isAtLeast()` gives trigger policies a readable comparison: `signal.isAtLeast(DetectionSignal.WEAK)`. What started as an internal Drools concern became an api/ change that benefits every future consumer.

## ACCUMULATE's confidence decoupling

The other design point worth noting: ACCUMULATE takes the max confidence across all results regardless of which signal it came from. Rule A might emit DETECTED(0.3), Rule B might emit WEAK(0.9). The accumulated result is DETECTED(0.9) — a combination no individual rule asserted.

This is defensible as "most optimistic collective assessment." In a detection system, false negatives cost more than false positives. The trigger policy is where conservatism belongs, not the collection layer.

## What it is now

`ResultCollectionStrategy` is a four-variant enum — HIGHEST_CONFIDENCE (default), FIRST_MATCH, LAST_WINS, ACCUMULATE. The channel accumulates all results as a list; the strategy resolves after `fireAllRules()`. `resolve()` always returns a valid `DetectionResult` — NOISE for empty, never null. The old null-check in `detect()` is gone.

The three #10 test gaps are closed: `close()` on ephemeral ganglion documented as a no-op, null-time events verified with a three-event sequence (real → null → real), and `buildAll(ExecutableModelProject.class)` consistency applied.

The branch is ready for merge. Epic 2 (RAS Runtime) is next — the engine that actually orchestrates ganglia, evaluates chain modes, and triggers case creation.
