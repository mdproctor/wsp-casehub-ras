# HANDOFF — casehub-ras

**Date:** 2026-07-05
**Issues:** #27, #23 (closed as dup), #24, #25, #26

## What was done

API module refinement and ChainMode extensions across 5 issues on one branch.

- **SPI promotion (#27):** moved CorrelationKeyExtractor, DefaultCorrelationKeyExtractor, SituationDefinitionProvider, SituationRegistration from runtime to api. Domain adapters (e.g. desiredstate/ras-adapter) can now depend on casehub-ras-api alone.
- **Artifact naming (#24):** casehub-ras-jpa → casehub-ras-persistence-jpa, casehub-ras-memory → casehub-ras-persistence-memory. Fixes broken IoT webapp reference.
- **ChainMode.Streak (#25):** consecutive detection with ANTI reset. Single ganglion, NOISE ignored.
- **ChainMode.Rate (#26):** ratio-based sliding window over scoreable signals. Multi-ganglion, window must be full.
- **#23:** closed as duplicate of #27.

## Key decisions

- DefaultCorrelationKeyExtractor moved to api (not just the interface) — default behaviour is an API contract.
- Rate window must be full before triggering — prevents premature triggers on sparse data.
- Compaction constraint documented: Streak, Rate, Count, Sequence should use no-op compaction ganglia on persistent situations.

## Cross-repo follow-up

- casehub-desiredstate#70 — drop runtime dep from ras-adapter (blocked by #27 publish)
- casehub-parent#347 — platform-wide artifact naming audit

## Immediate next step

Publish casehub-ras artifacts so desiredstate#70 can proceed. Then pick from backlog.

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 7 | DroolsSessionStore persistent implementation | M | Med | Blocked by #4. Needs Drools 10 serialization investigation. |
| 6 | RAS integration with service lifecycle cases | M | Med | Blocked by ops#30 (still open). |
| 5 | Platform stream infrastructure | XL | High | Epic, needs design. Lives in casehub-platform. |
