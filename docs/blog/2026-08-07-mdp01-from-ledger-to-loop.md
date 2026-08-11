---
layout: post
title: "From ledger to loop"
date: 2026-08-07
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [feedback-loop, outcome-ingestion, jpa, yaml-parsing, batch-processing]
series: issue-40-ras-feedback-loop
---

Continues from [Wiring the feedback path](2026-08-06-mdp02-wiring-the-feedback-path.md).

The previous session wired feedback into the existing runtime components — NaiveBayes priors, evaluator suppression, threshold overrides. This session built the layers that actually make it a loop: outcome ingestion, batch analysis, and JPA persistence. The feedback types existed; now they flow.

`OutcomeRecorder` implements `CaseOutcomeObserver` from `casehub-engine-api` — the SPI that fires when a case closes with an outcome. The recorder extracts `situationId` and `correlationKey` from the case file snapshot (put there by `DefaultCaseTrigger` at lines 107-109, which I hadn't noticed until I traced the data flow end-to-end). It looks up the situation's `FeedbackConfig`, classifies the outcome label, and records it to the ledger. The whole thing wraps in a try-catch because a failed recording should never prevent a case from closing.

The `FeedbackUpdateJob` is where the learning actually happens. Every five minutes it walks all situations with feedback config, queries per-tenant statistics from the ledger, and — if tuning is enabled — applies two kinds of adjustment. Threshold drift is straightforward: high noise rate pushes the `minConfidence` up via `FeedbackState`, which the evaluator consults on the next event. Prior recalibration is more involved. It maps case outcome label counts to NaiveBayes outcome indices through `outcomeGroundTruth`, blends the empirical distribution with the current priors using Laplace smoothing, and writes the result as log-space priors that new situation instances pick up.

The tenant isolation matters here. A deployment serving tenant A (fraud-heavy, confirming 70% of flagged transactions) and tenant B (compliance-heavy, dismissing 80% as noise) needs independent feedback loops. The job iterates `ledger.distinctTenancies()` per situation and applies adjustments per tenant. The priors for tenant A's NaiveBayes ganglion drift toward "fraud is common"; tenant B's drift toward "most flags are false positives." Same ganglion definition, different learned behaviour.

JPA persistence for the ledger hit a type-conversion gotcha. `SELECT MAX(closed_at)` via a Hibernate native query doesn't return `java.time.Instant` — the raw result type depends on the JDBC driver and Hibernate dialect, and pattern-matching against `Instant` and `java.sql.Timestamp` still missed it. The fix: use JPQL with the entity class for typed aggregate queries (Hibernate applies the field's type mapping), and reserve native SQL for operations JPQL can't express — `INSERT ... ON CONFLICT DO NOTHING` for the idempotent case-ID dedup, and bulk deletes for retention cleanup.

The YAML parsing rounds out the consumer surface. A `feedback:` section on any situation definition enables the loop; `outcomeGroundTruth:` on a NaiveBayes ganglion maps case labels to statistical outcomes. Both are optional — absent means no feedback, just detection as before. The advisory/tuning split means a deployer can start with suppression-only (`tuningEnabled: false`) to observe the noise rate before letting the system adjust its own parameters.

The design spec's nine tasks are complete. What's left is the branch close — review, squash, merge. The feedback loop is a closed system now: events arrive, ganglia detect, cases fire, outcomes return, and the next detection is slightly better calibrated than the last.
