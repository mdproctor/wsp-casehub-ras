# HANDOFF — casehub-ras

**Date:** 2026-07-13
**Issues:** #31 (closed — won't do), #36 (filed)

## What was done

First-principles investigation of #31 (ganglion-as-case). Evaluated every
stateful ganglion against the case-backed proposal. Conclusion: purpose-built
persistence wins everywhere — DroolsSessionStore for CEP, SituationStore for
accumulation. Case blackboard is a coordination medium, not a computation
buffer. Circular dependency (`engine → ras → engine`) is a hard blocker
regardless. Closed #31, filed #36 for the one real finding: NaiveBayesGanglion
log-posteriors are in-memory only, lost on restart.

Published 8 previously unpublished blog entries to casehub-notes (already
on personal-notes). All 10 RAS blog entries now at both destinations.

## Key decisions

- RAS owns its own persistence — cases coordinate via events, not by reaching into ganglion state
- Anything RAS needs from the platform belongs in `casehub-platform-api`
- NaiveBayes persistence gap is a small standalone fix (#36), not a case-engine integration

## What's left

- #36 — NaiveBayesGanglion: persist log-posteriors across restarts · S · Med

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #36 | NaiveBayesGanglion persistence | S | Med | Persist `double[]` alongside situation context |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
