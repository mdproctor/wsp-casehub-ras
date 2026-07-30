---
layout: post
title: "RAS Learns to Remember"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [situation-query, event-log, observability, design-review]
---

# RAS Learns to Remember

Until now, RAS lost everything the moment a situation terminated. A temperature anomaly accumulates across five detections, triggers a case, and vanishes — the `SituationStore` entity is removed, the context garbage collected, and the only surviving record is the case itself. For `NotifyOnly` situations (no case created), even that doesn't exist. The CDI event fires, any observer does its thing, and the situation might as well never have happened.

This is fine for a trigger engine. It's not fine for an observability layer — which is what RAS is supposed to be.

## The event log

The fix is an append-only event log that captures situation lifecycle transitions. Every time `SituationEvaluator` fires a `SituationChangeEvent` — TRIGGERED, RESOLVED, DISCARDED, SUPPRESSED — a CDI observer projects the key facts into a `ras_situation_event` row: confidence at transition, detection count, evidence, the `firstSeen` timestamp from when the situation started accumulating.

The observer uses `@ObservesAsync`, matching the existing `changeEvent.fireAsync()` calls. It never touches the detection hot path. If the database is down, the observer logs a warning and moves on — event history is best-effort, not transactional with the situation lifecycle.

That last point turned out to be more important than I expected. The design review caught a nasty asymmetry: for `NotifyOnly` triggers, the evaluator calls `fireAsync(...).toCompletableFuture().join()`, which means observer exceptions propagate back to the caller. Without a try-catch wrapper, a database failure in the event recorder would fail and reset every NotifyOnly trigger. The `CreateCase` path doesn't `.join()`, so the same failure would be silently swallowed there. Same observer, same event type, different failure mode depending on which trigger path fired it.

## The query surface

`SituationQueryService` is a new SPI in `api/` with four capabilities: historical timeline queries (`history()` with progressive narrowing — tenant, situation, correlation key), trigger frequency counts, trend analysis with rate normalization across different window sizes, and per-tenant health summaries. A separate `SituationEventRetention` interface handles TTL cleanup, called by the existing `SituationExpiryJob`.

The trend computation normalizes by window duration — comparing the last 24 hours against the previous 7 days compares rates, not raw counts. Three triggers in a day versus twenty-one in a week is STABLE, not FALLING. The thresholds (>1.2 for RISING, <0.8 for FALLING) are hardcoded for now, living as a static factory on `TrendResult.compute()`.

`SituationSource` stays exactly as it is — live state queries from `SituationStore` for what's accumulating right now. The new SPI serves everything else from the event log. Different data sources, different query patterns, separate interfaces. I considered absorbing `SituationSource` into the new SPI but it muddies the data boundary — the implementation would need to straddle two stores for what's conceptually one interface.

## The design review surfaced real issues

The adversarial review ran four rounds and raised seventeen issues. The substantive ones changed the design: `firstSeen` was missing from the original data model (you couldn't compute how long a situation accumulated before triggering), `@Transactional` was absent from the observer (Quarkus async observers don't get automatic JTA transactions), `removeEventsBefore()` was polluting the query SPI instead of living on a separate retention interface, and `trend()` had ambiguous temporal semantics — the reviewer pushed until the window/baseline periods were explicitly disjoint and anchored at an `asOf` parameter for reproducible testing.

The implementation followed the reviewed spec closely. Contract test in `api/` test-jar with 33 scenarios, InMemory and JPA implementations both extending it. The recorder has its own test covering confidence extraction, evidence mapping, and metadata passthrough.

The open question is whether terminal transitions are enough. The event log captures when situations conclude, but not when they start accumulating. A situation that started at T0 and triggered at T3 is invisible to `history(T0, T2)` — it hadn't terminated yet. `firstSeen` lets consumers compute accumulation duration from the terminal event, but truly active-range queries would need an inception event at the first `CONTINUE_ACCUMULATING`. That's a change to `SituationEvaluator`, which I deliberately left out of scope — but it's the obvious follow-up if consumers need "what was active between T1 and T2" for situations that hadn't triggered yet.
