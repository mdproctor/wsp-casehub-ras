# HANDOFF — casehub-ras

**Date:** 2026-07-07
**Issues:** #6 (closed)

## What was done

Service lifecycle RAS integration (#6). Extended casehub-ras to support signaling existing
cases via enriched CDI events, dynamic situation registration, and a generalized trigger output
model. TriggerAction sealed interface (CreateCase | NotifyOnly) replaces mandatory
CaseTriggerConfig. SituationChangeEvent enriched with SituationContext. Dynamic
register/deregister on SituationDefinitionRegistry via copy-on-write RegistrySnapshot.
SituationEvaluator dispatches on TriggerAction with different delivery semantics: NotifyOnly
awaits CDI event delivery with claim reset on failure; CreateCase remains fire-and-forget.
YAML triggerAction format with type discriminator. SituationStore.removeAllForSituation()
for persistent situation cleanup. Design-reviewed (5 rounds, 12 issues, all verified).
Per-task code review (8 tasks). Final whole-branch code review (5 findings, all fixed).

## Key decisions

- Approach B (enriched CDI events with consumer bridge) over Approach A (RAS signals cases
  directly). RAS stays decoupled from engine case model; bridge code is inherently
  domain-specific and belongs in the consuming app.
- TriggerDecision renamed: CREATE_CASE → TRIGGER, CREATE_CASE_AND_CONTINUE →
  TRIGGER_AND_CONTINUE. Names now describe situation lifecycle, not case lifecycle.
- Ganglion-as-case deferred to #31 — RAS already has purpose-built persistence
  (DroolsSessionStore, JpaSituationStore). Case blackboard is coordination medium, not
  computation buffer.
- NotifyOnly delivery awaits CDI event; CreateCase event is fire-and-forget. The case is the
  durable artifact for CreateCase; the event IS the sole output for NotifyOnly.

## Cross-repo follow-up

- casehubio/casehub-desiredstate#71 — migrate DesiredStateSituationDefinitionProvider to TriggerAction API
- casehubio/casehub-ops#47 — wire RAS health monitoring for managed applications (first consumer)
- casehubio/casehub-ops#48 — migrate AdaptiveTopologyManager to enriched SituationChangeEvent

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 31 | Ganglion-as-case — case-backed state persistence | M | High | Deferred from #6. Open question: is this still needed given DroolsSessionStore? |
| 28 | DroolsSessionStore production hardening | S | Med | Monitoring, graceful degradation |
| 29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| 30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| 5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
