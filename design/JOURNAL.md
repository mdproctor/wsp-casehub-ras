# Design Journal — issue-40-ras-feedback-loop

## 2026-08-06 — Design + foundation implementation

### Design decisions

- **Composable feedback pipeline** — three layers (ingestion, analysis, application)
  rather than monolithic or policy-centric approach. Chosen for testability and
  separation of real-time (suppression) vs batch (tuning) concerns.
- **Split SPIs** — `SuppressionStrategy` (per-event, hot path) and `FeedbackTuningStrategy`
  (batch, every 5 min). Emerged from structure review R1-05: single `FeedbackStrategy`
  mixed fundamentally different lifecycles.
- **FeedbackState component** — mutable feedback-derived state separated from
  `SituationDefinitionRegistry` (immutable snapshot discipline). Structure review R1-08
  caught the scope creep. Tenant-scoped via `(id, tenancyId)` keying.
- **Suppression in SituationEvaluator** — not in `DefaultRasTriggerPolicy`. Policy stays
  pure with zero dependencies. Structure review R1-03 identified the coupling risk.
- **Advisory mode default** — `tuningEnabled: false` on `FeedbackConfig`. Suppression and
  metrics always active; threshold/prior adjustment requires explicit opt-in. Cross-cutting
  review R1-05 refined the activation mechanism from CDI bean presence to config flag.
- **Log-space prior boundary** — `FeedbackState.applyPriorOverride()` converts raw priors
  to log-space at the boundary. Robustness review R1-02 caught the silent corruption path
  where raw priors would be injected into log-space ganglion state.

### Design review

4-dimension adversarial review (coherence, structure, robustness, cross-cutting).
50 issues raised, 46 verified, 4 accepted (container headings), 0 unresolved. $63.91.
Key architectural changes: SPI split, FeedbackState extraction, policy purity preservation,
tenant scoping, log-space boundary, zero-prior validation, retention >= cooldown invariant,
duplicate outcome dedup via UNIQUE(case_id).

### Implementation progress

Tasks 1-3 of 9 complete:
1. Core types, SPIs, SituationDefinition changes (api/) — 11 new/modified files
2. InMemoryOutcomeLedger (runtime/) — @DefaultBean, 12 contract tests
3. FeedbackState + DefaultSuppressionStrategy + DefaultTuningStrategy (runtime/) — 29 tests

Remaining: NaiveBayes/registry integration, evaluator integration, ingestion/batch,
YAML parsing, JPA persistence, docs.
