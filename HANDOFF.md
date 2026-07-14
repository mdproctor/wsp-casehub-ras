# HANDOFF — casehub-ras

**Date:** 2026-07-14
**Issues:** #36 (closed)

## What was done

Implemented GanglionStateStore — pluggable persistence SPI for ganglion
computation state. SPI in `api/` (`GanglionStateStore`, `GanglionStateKey`,
`GanglionState`, `GanglionStateConflictException`), `InMemoryGanglionStateStore`
`@DefaultBean` in `runtime/`, `JpaGanglionStateStore` in `persistence-jpa/`
with entity, Flyway V5, and optimistic locking. NaiveBayesGanglion refactored
to use the store with load/save/retry pattern. `SituationExpiryJob` wired for
orphan cleanup via `removeOrphaned()`. Filed #39 for analogous DroolsSessionStore
orphan gap.

## Key decisions

- Generic SPI (not NaiveBayes-specific) — `double[]` reflects the domain of numeric accumulation ganglia
- SPI in `api/` (dependency direction constraint), in-memory in `runtime/` (`@DefaultBean` needs quarkus-arc)
- Module-local metrics in persistence-jpa/ (can't depend on runtime/ for RasMetrics)
- Retry owned by ganglion, not evaluator — ganglion state conflicts independent of situation conflicts

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #39 | DroolsSessionStore orphaned session cleanup | S | Med | Same removeOrphaned() pattern as GanglionStateStore |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
