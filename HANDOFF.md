# HANDOFF — casehub-ras

**Date:** 2026-07-06
**Issues:** #7 (closed)

## What was done

Persistent DroolsSessionStore implementation (#7). Redesigned DroolsSessionStore SPI from
get()/put()/remove() with four strings to Map-aligned `computeIfAbsent(DroolsSessionKey,
KieBase, config, generation)` with generation-based lazy invalidation. New `drools-reliability/`
module provides ReliableDroolsSessionStore backed by H2MVStore — sessions survive JVM restarts.
Design-reviewed (3 rounds, 15 issues, all resolved). Four drools-reliability gotchas captured
in the garden.

## Key decisions

- `removeAll()` eliminated from SPI — contradicts hot reload spec's per-key serialization model.
  Replaced with generation parameter on `computeIfAbsent` for lazy invalidation.
- Separate module (`drools-reliability/`) rather than optional deps in ras-drools — matches
  persistence-memory/persistence-jpa pattern.
- CDI Tier 2 (`@ApplicationScoped`) beats InMemory's Tier 1 (`@DefaultBean`) by classpath presence.
- Experimental/temporary — journal-based reliability (#29) will replace this.

## Cross-repo follow-up

- casehub-desiredstate#70 — drop runtime dep from ras-adapter (blocked by ras artifact publish)

## Immediate next step

Publish casehub-ras artifacts so desiredstate#70 can proceed.

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 6 | RAS integration with service lifecycle cases | M | Med | Blocked by ops#30 (still open) |
| 5 | Platform stream infrastructure | XL | High | Epic, needs design. Lives in casehub-platform |
