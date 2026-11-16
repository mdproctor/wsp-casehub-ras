---
title: "Closing the Feedback Loop's Blind Spots"
date: 2026-08-30
author: mdp
entry_type: note
subtype: diary
series: issue-62-drift-recall-quality
projects:
  - casehubio/casehub-ras
tags:
  - feedback-loop
  - quality-metrics
  - drift-detection
  - recall
---

# Closing the Feedback Loop's Blind Spots

The RAS feedback loop (#40) computes precision and noise rate from case outcomes. The recall metric (#60) added the other half — missed detection signals from operators. But three things were deferred: classifying what drift *means*, making recall work per-ganglion, and checking whether a reported miss was actually detected.

Today I tackled all three.

## DriftDirection — Naming the State

The feedback loop publishes precision, recall, and noise rate as individual gauges. Operators stare at three numbers and interpret drift themselves. DriftDirection gives that interpretation a name.

The interesting design question was whether OVER_SENSITIVE and UNDER_SENSITIVE should be mutually exclusive. My first instinct was yes — pick the dominant direction, keep it simple. But I stopped to think about what each direction measures. Over-sensitive means too many false positives (high noise rate). Under-sensitive means too many false negatives (low recall). These are independent axes of the confusion matrix. A detector *can* be simultaneously noisy and miss real events.

That state — both bad at once — is operationally distinct. When noise is high, you raise thresholds. When recall is low, you lower them. When both are true, those actions conflict. The right response isn't a threshold knob — it's ganglion reconfiguration. The detection model itself is wrong.

So DriftDirection became a 5-value enum:

```java
public enum DriftDirection {
    OVER_SENSITIVE,
    UNDER_SENSITIVE,
    BOTH_DRIFTING,
    STABLE,
    INSUFFICIENT_DATA
}
```

BOTH_DRIFTING suppresses the existing auto-tuning. Without this guard, the system raises thresholds (because noise is high) while simultaneously signaling "reconfigure your ganglia" — actively making recall worse while telling operators not to touch thresholds.

Classification lives as a default method on `FeedbackTuningStrategy`, not on `FeedbackAnalyzer`. The analyzer fetches statistics. The strategy interprets them. That separation matters because a custom strategy should be able to define its own drift model — different thresholds, different combinations. The default method provides a standard classification; overriding it is the extension point.

## Per-Ganglion Recall

Missed detection reports are situation-level: "fire-risk-detector should have fired for ACC-12345." Per-ganglion recall needs to know *which* ganglion failed to fire. Most operators won't know this — they know the situation should have triggered, not which specific classifier inside it missed.

The solution: an optional `List<String> ganglionIds` on `MissedDetectionRecord`. Null or empty means situation-level miss (the common path). Non-empty means the operator knows which ganglion(s) should have contributed.

`GanglionOutcomeStatistics` gains `missedCount` and its own `recall()` method. But — and this was validated by the design review — `recall()` stays off the `QualityMetrics` interface. The #60 spec deferred this for a specific reason: for ganglia without explicit missed reports, `missedCount` stays 0, and a default `recall()` on the interface would return 1.0. Perfect recall, for a ganglion that might be completely broken — the only reason it shows 1.0 is that nobody filed a report against it. Misleading metric.

`GanglionOutcomeStatistics.recall()` handles this differently: it returns NaN when `missedCount == 0`, which suppresses the gauge entirely via the existing NaN convention. The gauge only appears after the first ganglion-level missed report is filed, which means it shows real data from the start.

## Trigger Cross-Reference

When an operator reports a missed detection, the system now checks `SituationQueryService.history()` for a matching trigger within a configurable time window. If found, the response includes `possiblyDetected: true` with an advisory message.

The response is advisory — the record is always stored. Operators are asserting domain knowledge the system can't verify. The cross-reference adds a second opinion, not a veto. A nice touch from the spec review: the advisory text is different for situation-level vs ganglion-level reports. A situation-level report with a matching trigger says "this may inflate the missed count." A ganglion-level report with a matching trigger says "other ganglia may have detected the event" — because the *situation* was caught, but the specific ganglion named in the report may legitimately have failed.

The cross-reference also tracks whether the result is conclusive. If the missed event falls outside the trigger history retention window (default 30 days, configurable), the response includes `crossRefConclusive: false` — so consumers know that "not detected" means "can't tell" rather than "genuinely missed."

## What This Opens Up

DriftDirection is published but purely observational. The next step is whether BOTH_DRIFTING should feed into a ganglion reconfiguration workflow — but that requires knowing *which* ganglia are miscalibrated, which circles back to per-ganglion quality metrics and the new per-ganglion recall data. The pieces are now in place for that analysis; the workflow itself is a separate design problem.
