# HANDOFF — casehub-ras

**Date:** 2026-06-30
**Branch closed:** issue-21-trigger-cleanup-long-lived-situations → merged to main
**Issues:** #21 (closed), #20 (closed)

## What was done

Trigger lifecycle model and situation query API.

- **TriggerDecision** gains `CREATE_CASE_AND_CONTINUE` and `RESOLVE` variants — five total.
- **TriggerMode** sealed interface (`FireOnce`, `Repeating(Duration cooldown)`) on `SituationDefinition`.
- **SituationStore SPI**: `save()` returns `Uni<SituationContext>` with storeVersion;
  `tryClaimTrigger` atomically stamps `lastTriggered`/`triggerCount` (store-managed fields).
  New `removeTriggeredBefore`, `findActive` methods.
- **Guard cleanup** in `SituationExpiryJob` — resolves #21 (stuck persistent policyTriggered entities).
- **SituationSource/ActiveSituation/SituationChangeEvent** query API in api/ — authoritative
  versions for #22's cross-repo migration from desiredstate-api.
- **DefaultSituationSource**, YAML `triggerMode` parsing, `RasEndpointRegistration` (QUERY in EndpointRegistry).

## Key decisions

- Trigger metadata (`lastTriggered`, `triggerCount`) is store-managed: stamped by `tryClaimTrigger`,
  preserved by `save()` (mapper excludes from `updateEntity`). Design review R2-02 caught that
  naive save would overwrite.
- `CREATE_CASE_AND_CONTINUE` uses save-first (no bifurcation) — detection always persisted
  before claiming. Loser returns false (continues accumulating).
- Guard period (default PT1M) prevents GC-pause duplicate race on entity removal.
- RAS reclassified as Foundation tier (casehubio/parent#327 filed).
- Situation types move from desiredstate-api to ras-api (#22 filed).

## What's next

- **#22** — Move ActiveSituation/SituationSource/SituationChangeEvent from desiredstate-api to
  ras-api + update ops-deployment imports. Cross-repo, blocked by parent#327.
- **#5** — Platform stream infrastructure (epic, separate design).
- **#6** — Service lifecycle management pattern (design).
- **#7** — DroolsSessionStore persistent implementation.
