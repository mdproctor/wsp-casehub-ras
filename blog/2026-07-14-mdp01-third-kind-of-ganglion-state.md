---
layout: post
title: "The Third Kind of Ganglion State"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [persistence, spi, naive-bayes, ganglion]
series: issue-36-naivebayes-persistence
---

NaiveBayesGanglion had a gap that became obvious once I looked at it through the persistence lens: it holds running log-posteriors in a `ConcurrentHashMap` that evaporates on restart. For windowed situations this barely matters — the window expires anyway. For persistent situations, a correctly-detected anomaly can temporarily drop below threshold after restart because the accumulation restarts from priors.

The fix seemed simple until I thought about where the SPI should live. Ganglia fall into three state categories. Stateless ones like `JavaSwitchGanglion` need no store. Complex-state ones like `DroolsGanglion` need their own domain-specific store — `DroolsSessionStore` exists because KieSession recovery requires Drools-specific APIs. NaiveBayes state is just `double[]`. That's not accidentally NaiveBayes-specific — moving averages, EMA, voting tallies, Bayesian posteriors are all numeric arrays. Any ganglion complex enough to need non-numeric state is category three and needs its own store.

So we built `GanglionStateStore` as a generic SPI in `api/` for simple-state ganglia. The module placement follows a constraint, not a choice: `persistence-jpa/` must implement it and depends on `api/`, not `runtime/`. Putting the SPI in `runtime/` would invert the dependency direction. The in-memory `@DefaultBean` goes in `runtime/` because `api/` is pure Java with no Quarkus annotations. CDI activation follows the DroolsSessionStore pattern — add `persistence-jpa/` to your classpath and the JPA implementation wins automatically.

The design review surfaced three findings worth the cost. First, `persistence-jpa/` can't use `RasMetrics` from `runtime/` without creating a dependency cycle — so `JpaGanglionStateStore` gets its own module-local `GanglionStatePersistenceMetrics` bean, following the `DroolsReliabilityMetrics` pattern. Second, `removeForSituation` needs an index on `situation_id` because the unique constraint's leftmost column is `ganglion_id`. Third, `SituationExpiryJob` bulk-deletes expired situations without calling `closeGanglia()` — which means ganglion state rows become orphans. We added `removeOrphaned()` with a `NOT EXISTS` join to clean them up, and filed #39 because `ReliableDroolsSessionStore` has exactly the same gap.

The implementation is five commits: SPI types and contract test, in-memory store, NaiveBayesGanglion refactored to load/save/retry, JPA entity with Flyway V5, and the expiry job wiring. NaiveBayesGanglion now takes `GanglionStateStore` as a constructor parameter — a breaking change, but pre-release, so the only cost is updating producer methods. The retry loop in `detect()` handles optimistic locking conflicts from clustered deployments during Kafka partition rebalancing.

The three-category model — stateless, simple-state, complex-state — feels like it has legs beyond this specific fix. Any future numeric-accumulation ganglion slots into the same SPI without a new store type.
