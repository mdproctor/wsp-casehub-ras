# HANDOFF — casehub-ras

**Date:** 2026-06-28
**Branch closed:** issue-11-runtime-enhancements → merged to main (56e6b24)
**Issues:** #11 (closed, already done), #9 (closed), #16 (closed), #14 (closed)

## What was done

Runtime enhancements batch: three features shipped, one issue closed as already implemented.

- **DroolsGanglion hot rule reload (#9)** — `reload()` with lazy invalidation via
  generation counter. JMM acquire-release volatile ordering. Synchronized reload.
- **Event reordering buffer (#16)** — Watermark-based `EventReorderBuffer` in
  SituationEvaluator. `processEvent()` extraction with mid-batch termination signal.
  `EventBufferFlushJob`. YAML `eventBufferDelay` support.
- **JPA SituationStore (#14)** — New `persistence-jpa/` module. JSONB detections via
  `@JdbcTypeCode(SqlTypes.JSON)`. Flyway at `db/ras/migration/`. Persistence-memory
  package renamed, @Priority fixed.

## Key decisions

- Lazy invalidation over drain — persistent situations never converge with drain
- Acquire-release volatile read order — flag first, data second (JMM-correct)
- processEvent() returns termination signal — lock/buffer cleanup in evaluate(), not pipeline
- JSONB with @JdbcTypeCode — columnDefinition alone insufficient for parameter binding

## What's next

- No immediate follow-on. Filed: #17 (AbstractSituationStoreContractTest),
  #18 (clustered retry logic). Deferred from earlier: #5 (platform stream infra),
  #6 (service lifecycle), #7 (DroolsSessionStore persistent).
