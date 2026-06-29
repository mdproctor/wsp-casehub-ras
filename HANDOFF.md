# HANDOFF — casehub-ras

**Date:** 2026-06-29
**Branch closed:** issue-18-clustered-retry-logic → merged to main (be92653)
**Issue:** #18 (closed)

## What was done

Clustered retry logic for concurrent JVM conflict handling.

- **OCC conflict detection** — `@Version` on `SituationEntity`, `OptionalLong storeVersion`
  on `SituationContext`, two-layer conflict detection in `JpaSituationStore` (application-level
  version comparison + Hibernate OLE/constraint violation wrapping as `SituationConflictException`).
- **Two-phase processEvent** — Phase 1 detects once (ganglia mutate internal state), Phase 2
  retries read-modify-write on conflict. Max retries configurable via
  `ras.evaluator.max-conflict-retries` (default 3).
- **Adversarial design review** caught a critical flaw: `@Version` alone doesn't prevent lost
  updates when save() transactions don't overlap. Fixed by propagating `storeVersion` through
  the domain model for application-level comparison before Hibernate's check.

## Key decisions

- `storeVersion` on domain record (not internal to JPA entity) — design review R1-02 proved
  the JPA-internal approach silently loses updates on non-overlapping transactions
- `find()` changed from TxType.SUPPORTS to REQUIRED — ensures fresh persistence context on
  each read for retry loop correctness (garden entry GE-20260629-7d2272)
- Detection never retried — ganglia mutate internal state (DroolsGanglion KieSession,
  NaiveBayesGanglion posteriors). DetectionResult portability invariant documented on Ganglion
- Duplicate case trigger (#19) deferred — separate concern from constraint violation retry

## What's next

- Filed: #19 (duplicate case trigger prevention under concurrent CREATE_CASE — depends on
  #18's @Version + SituationConflictException infrastructure).
- Deferred from earlier: #17 (AbstractSituationStoreContractTest), #5 (platform stream
  infra), #6 (service lifecycle), #7 (DroolsSessionStore persistent).
