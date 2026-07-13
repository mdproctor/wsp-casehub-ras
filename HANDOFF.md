# HANDOFF — casehub-ras

**Date:** 2026-07-13
**Issues:** #34 (closed), #35 (closed), #33 (closed)

## What was done

Batched three issues on one branch: Jackson `@JsonTypeInfo`/`@JsonSubTypes` on
`TriggerAction`, `ChainMode`, `TriggerMode` for JSONB polymorphic serde (#34);
centralised `DroolsReliabilityMetrics` bean replacing inline `MeterRegistry` usage
in `ReliableDroolsSessionStore` (#35); H2MVStore file-corruption auto-recovery at
startup with probe-before-static-init pattern (#33). CLAUDE.md synced for all three
changes including `removeExpired`/`removeTriggeredBefore` return type fix from #32.

## Key decisions

- Jackson type discriminator names match YAML convention (`and`, `or`, `create-case`,
  `fire-once` etc.) — single source of truth across serialisation formats
- Corruption probe uses `static volatile boolean` gate — runs once per JVM to avoid
  file-lock conflict with the Drools `StorageManagerFactory` static holder singleton
- `jackson-annotations` is `provided` scope in api/ — consumers get it transitively
  from Quarkus, api/ doesn't force it

## Cross-repo follow-up

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 31 | Ganglion-as-case — case-backed state persistence | M | High | Open question: still needed given DroolsSessionStore? |
| 29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| 30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| 5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
