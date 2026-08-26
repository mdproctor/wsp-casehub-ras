# Meta-Situations — Situations Observing Other Situations

**Issue:** casehubio/casehub-ras#41
**Date:** 2026-08-26
**Status:** Design

## Problem

Each situation in RAS is independent. Correlated situations are themselves a signal — "3 independent service-health situations active simultaneously" indicates system-wide degradation, not isolated failures. Currently there is no way to express this without writing a custom ganglion that queries `SituationSource`, which couples detection logic to the query API and bypasses the situation definition model.

Additionally, temporal absence patterns — "situation triggered but no resolution within an SLA window" — have no mechanism in the current architecture. These escalation patterns are a primary consumer of meta-situations.

## Architecture

A CDI observer bridge (`SituationChangeEventBridge`) converts every `SituationChangeEvent` into a `CloudEvent` and fires it back via CDI. The existing `RasEngine.onCloudEvent()` processes bridged events identically to any other CloudEvent. Meta-situations are regular `SituationDefinition`s whose `eventTypes` include bridged change types.

No new evaluation model. No separate engine. All existing infrastructure — stores, policies, chain modes, metrics, feedback, YAML definitions, replay — works unchanged.

### Data Flow

```
CloudEvent → RasEngine → SituationEvaluator → SituationChangeEvent
                ↑                                      ↓
                └────── SituationChangeEventBridge ─────┘
                        (wraps as CloudEvent)
```

Meta-situations can subscribe to bridged situation-change types AND raw CloudEvent types in the same definition, enabling cross-level correlation.

### Nesting

Unlimited depth. Meta-situations are situations — they fire `SituationChangeEvent`s, which get bridged, which can be observed by other meta-situations. Cycle detection at registration time prevents infinite loops (see Cycle Detection).

## Bridge Component

`SituationChangeEventBridge` — `@ApplicationScoped` CDI bean in `runtime/`.

### Observer

```java
void onSituationChange(@ObservesAsync SituationChangeEvent change)
```

### Bridged CloudEvent Shape

| Field | Value |
|-------|-------|
| `type` | Per-ChangeType: `ras.situation.triggered`, `ras.situation.resolved`, `ras.situation.discarded`, `ras.situation.suppressed`, `ras.situation.dismissed` |
| `source` | `URI.create("ras://bridge")` |
| `subject` | `change.situationId()` |
| `id` | Generated UUID |
| `time` | `Instant.now()` |

### Extensions

| Extension | Value | Purpose |
|-----------|-------|---------|
| `tenancyid` | `change.tenancyId()` | Tenant identity (standard RAS extension) |
| `situationid` | `change.situationId()` | Child situation identity |
| `correlationkey` | `change.correlationKey()` | Child correlation key |
| `changetype` | `change.changeType().name()` | Change type as string |

### Data Payload

JSON summary — lightweight, not the full `SituationContext`:

| Field | Type | Source |
|-------|------|--------|
| `situationId` | String | `change.situationId()` |
| `correlationKey` | String | `change.correlationKey()` |
| `tenancyId` | String | `change.tenancyId()` |
| `changeType` | String | `change.changeType().name()` |
| `firstSignal` | Instant | `change.context().firstSignal()` |
| `lastSignal` | Instant | `change.context().lastSignal()` |
| `triggerCount` | int | `change.context().triggerCount()` |
| `detectionCount` | int | `change.context().detections().size()` |

Ganglia needing richer child-situation data can query `SituationStore`.

### CDI Exception Isolation

The bridge wraps all logic in try-catch. The bridge is an `@ObservesAsync` observer — if it throws, it crashes the `NotifyOnly` trigger path (which calls `.join()` on the `CompletionStage`). The bridged CloudEvent is fired fire-and-forget (no `.join()`). See GE-20260730-d54a8f for the underlying CDI asymmetry.

### Thread Pool Isolation

Bridge-originated CloudEvents use CDI `NotificationOptions` with a separate managed executor, isolating meta-situation load from primary situation evaluation during degradation scenarios.

## Event Type Convention

Five bridged event types, matching `SituationChangeEvent.ChangeType` enum values:

| Bridged Type | Source ChangeType |
|-------------|-------------------|
| `ras.situation.triggered` | `TRIGGERED` |
| `ras.situation.resolved` | `RESOLVED` |
| `ras.situation.discarded` | `DISCARDED` |
| `ras.situation.suppressed` | `SUPPRESSED` |
| `ras.situation.dismissed` | `DISMISSED` |

Per-situation filtering uses `eventFilter` expressions (e.g., `.situationid == "child-X"`), keeping the type namespace bounded and stable. Adding or removing situation definitions does not change the event type contract.

### CloudEventExpressionContext Update

`CloudEventExpressionContext.build()` must be updated to expose all CloudEvent extensions, not just `tenancyid`. The bridge adds `situationid`, `correlationkey`, and `changetype` as extensions — these must be accessible in eventFilter and correlationKey expressions. The update iterates `event.getExtensionNames()` and adds each extension to the expression context map. This is a general improvement, not meta-situation-specific — it also benefits any future CloudEvent extensions.

Expression access path: flat top-level keys (e.g., `.situationid`, `.correlationkey`), consistent with the existing `.tenancyid` access pattern.

### Default Correlation

`subject = situationId` means `DefaultCorrelationKeyExtractor` groups meta-situations by child situation identity. For tenant-level aggregation (the "3 services degraded in tenant X" pattern), use `correlationKeyExpression` to extract the `tenancyid` extension.

## GanglionDescriptor.SituationWatcher

New sealed interface variant in `api/`, alongside `NaiveBayes` and `ExpressionRules`. Enables YAML-configured situation-watching ganglia.

### Configuration

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `ganglionId` | String | Yes | Ganglion identifier |
| `changeTypeMapping` | Map<ChangeType, DetectionSignal> | Yes | Maps change types to detection signals |
| `evidenceTemplates` | Map<String, ExpressionEvaluator> | No | Expression-based evidence extraction |

### YAML

```yaml
ganglia:
  - ganglionId: service-health-watcher
    type: situation-watcher
    changeTypeMapping:
      triggered: DETECTED
      resolved: ANTI
      discarded: NOISE
```

### Runtime Construction

`SituationDefinitionRegistry.constructGanglion()` gets a new branch for `SituationWatcher` descriptors. Constructs a `SituationWatcherGanglion` in `runtime/` that:

1. Extracts `changetype` from the bridged CloudEvent's extensions
2. Maps to `DetectionSignal` via the configured mapping
3. Returns `DetectionResult` with the mapped signal and confidence 1.0
4. Events with unmapped change types are not routed to this ganglion (see handledEventTypes)

### Automatic Evidence

Every `SituationWatcherGanglion` detection includes:
- `childSituationId` — extracted from `situationid` extension
- `childCorrelationKey` — extracted from `correlationkey` extension
- `childChangeType` — extracted from `changetype` extension

Ganglion-level `evidenceTemplates` merge on top, following the existing merge order: auto → per-decision-path → ganglion-level.

### handledEventTypes

`SituationWatcherGanglion.handledEventTypes()` returns only the bridged event types for which `changeTypeMapping` has entries. A ganglion with `{triggered: DETECTED, resolved: ANTI}` returns `{"ras.situation.triggered", "ras.situation.resolved"}`. A ganglion with just `{triggered: DETECTED}` returns `{"ras.situation.triggered"}`.

This ensures correct routing: `SituationEvaluator.gangliaHandlingEventType()` only routes events to ganglia that actually map them, preventing NOISE pollution from unmapped events.

## Deadline — Temporal Absence

`SituationDefinition.deadline` — optional `Duration` field, alongside `correlationWindow`.

### Semantics

Deadline is a temporal lifecycle property, not a detection pattern. Every existing `ChainMode` variant is a pure function of `SituationContext.detections()` — given the same context, they always return the same result. This is what makes `SituationReplayRunner` deterministic. Placing deadline on the definition (not in a chain mode) preserves this invariant.

Any chain mode can have an optional deadline layered on top. After `firstSignal + deadline` elapses, if the situation hasn't already triggered, resolved, or been discarded, the `DeadlineCheckJob` forces a trigger regardless of chain mode satisfaction.

### Deadline + correlationWindow Interaction

When both are set, `correlationWindow` takes precedence. Before force-triggering, `triggerByDeadline()` checks `context.firstSignal() + definition.correlationWindow() < Instant.now()`. If the correlation window has expired, the situation is discarded instead of triggered — it should have been cleaned up on the next event arrival but no event arrived.

### SituationDefinition Changes

- New field: `Duration deadline` (`@Nullable`)
- Validated: positive when set (same pattern as `correlationWindow`)
- YAML: `deadline: PT30M` (ISO-8601 Duration)
- `withChainMode()` copy method must carry `deadline` through (all 12 fields)
- Both convenience constructors pass `null` for `deadline`

### DeadlineCheckJob

`@Scheduled` bean in `runtime/`.

- Iterates deadline-enabled definitions from the registry (`registry.allSituationIds()` → filter by `definition.deadline() != null`)
- For each, queries `SituationStore.findActiveBySituationId(situationId)` for active instances
- Checks `context.firstSignal() + definition.deadline() < Instant.now()`
- When expired: calls `SituationEvaluator.triggerByDeadline(situationId, correlationKey, tenancyId)` — a new entry point that handles trigger mechanics without requiring a `CloudEvent` input
- Check interval: configurable via `ras.deadline.check-interval` (default `PT10S`)

### triggerByDeadline Implementation

To avoid duplicating the trigger sequence in `executeDecision()`, extract shared trigger mechanics into a private method (e.g., `executeTrigger(SituationContext, SituationDefinition, String, String, String, Instant)`) called by both `executeDecision()` and `triggerByDeadline()`. The shared method handles: claim → save → case trigger → change event → close ganglia, including the subtle ordering (claim-before-save vs save-before-claim based on `storeVersion`), error recovery (reset claim on failure), and metric emission.

### Dual-Path Evaluation

The event-driven path also checks `firstSignal + deadline < eventTime` on every evaluation for deterministic behaviour when events continue arriving. The scheduled job is a backstop for quiescent situations where no further events arrive.

### Replay Support

`SituationReplayRunner` needs a `drainAllDeadlines(Instant endOfReplayWindow)` method that checks all active situations for expired deadlines at a given replay timestamp. Without this, deadlines that expire after the last replayed event but before the replay window ends would be missed. `drainAllDeadlines()` is called after `drainAllBuffers()` at the end of replay.

### SituationStore Impact

New query method: `List<SituationContext> findActiveBySituationId(String situationId)` — returns all active situation instances for a given situation definition. The `DeadlineCheckJob` iterates deadline-enabled definitions from the registry, then queries the store by situationId. JPA implementation adds an indexed query on `situation_id` where `triggered_at IS NULL`. In-memory implementation filters the existing map.

## Cycle Detection

Mandatory registration-time validation in `SituationDefinitionRegistry`.

### Algorithm

Build a directed graph:
- **Nodes** = situation definitions
- **Edges** = "situation S₁ can fire → bridge produces change type T → situation S₂ subscribes to T"

Every registered situation can produce 5 bridged change types. An edge exists from S₁ to S₂ if S₂'s `eventTypes` contains any bridged type. The graph has at most N×5 edges — tractable for any realistic deployment.

### Self-Edge Handling

A meta-situation subscribing to `ras.situation.triggered` creates a self-edge (it can also produce that type when it triggers). Self-edges are handled by trigger mode:

- **FireOnce:** Self-edges are excluded from cycle detection. After triggering, the situation context is cleaned up (`store.remove()`, ganglia closed). If the meta-situation's own bridged trigger event arrives post-cleanup, it starts a fresh context that won't re-trigger from a single event (chain modes require sufficient detections).
- **Repeating:** Self-edges are included. A repeating meta-situation with `Count(ganglionId, 1)` would genuinely self-loop — each trigger produces a bridged event that re-triggers.

### Constructor Phase 3

Cycle detection also runs during `SituationDefinitionRegistry`'s constructor Phase 3 (initial YAML/CDI registration). The cycle graph is built incrementally as each registration is processed — each new definition is checked against the graph so far before being added.

### Dynamic Registration

`register()` checks cycles before adding. `deregister()` removes edges — no cycle check needed (removing nodes can only break cycles, not create them).

### EventFilter Conservatism

EventFilters are treated as transparent for cycle detection. A definition filtering on `.situationid == "child-X"` still creates edges from all situations. This may produce false-positive rejections but never false negatives. Conservative safety is correct for mandatory validation.

## Sequence Chain Mode Limitation

CDI `fireAsync()` makes no ordering guarantees across independent calls. If child A triggers at T=1 and child B triggers at T=2, their bridged CloudEvents may arrive at `RasEngine` in any order (especially with the separate bridge executor pool).

`DefaultRasTriggerPolicy.evaluateSequence()` sorts detections by `eventTime` (`TimestampedDetection.eventTime()`), and the bridge stamps the CloudEvent `time` field with `Instant.now()`. For events with sufficient temporal separation, ordering is preserved. For near-simultaneous triggers (sub-millisecond), clock precision and thread scheduling could swap order.

Recommendation: prefer `And` over `Sequence` for meta-situations when strict ordering is not required. Document this as a known limitation.

## YAML Examples

### System-wide degradation (Count chain mode)

```yaml
ganglia:
  - ganglionId: service-trigger-watcher
    type: situation-watcher
    changeTypeMapping:
      triggered: DETECTED
      resolved: ANTI

situations:
  - situationId: system-wide-degradation
    eventTypes:
      - ras.situation.triggered
    eventFilter:
      expression: ".situationid | test(\"^service-health-\")"
      language: jq
    correlationKey:
      expression: .tenancyid
      language: jq
    chainMode:
      type: count
      ganglionId: service-trigger-watcher
      requiredCount: 3
    triggerAction:
      type: create-case
      caseNamespace: ops
      caseName: system-degradation
      caseVersion: "1.0"
```

### SLA breach (deadline temporal absence)

The SLA breach pattern uses Threshold with ANTI cancellation and a deadline backstop. The trigger-watcher adds confidence on child TRIGGERED events. The resolve-watcher subtracts confidence on child RESOLVED events (ANTI). The threshold is set high enough that events alone cannot satisfy it — only the deadline can force the trigger.

When a child resolves before the deadline, the ANTI detection reduces accumulated confidence. At deadline expiry, `DeadlineCheckJob` invokes `triggerByDeadline()` which fires regardless of chain mode state — but if the situation was already resolved/discarded by event-driven evaluation, no trigger occurs.

**Known gap:** There is currently no mechanism for a cancel event (child resolution) to proactively resolve/discard the pending meta-situation. The Threshold ANTI reduces confidence but does not trigger RESOLVE/DISCARD. A cancel-on-resolution mechanism (e.g., policy returning RESOLVE when accumulated confidence drops to zero or below) is a follow-on enhancement.

```yaml
ganglia:
  - ganglionId: trigger-watcher
    type: situation-watcher
    changeTypeMapping:
      triggered: DETECTED
  - ganglionId: resolve-watcher
    type: situation-watcher
    changeTypeMapping:
      resolved: ANTI

situations:
  - situationId: sla-breach
    eventTypes:
      - ras.situation.triggered
      - ras.situation.resolved
    eventFilter:
      expression: ".situationid == \"service-health\""
      language: jq
    correlationKey:
      expression: "(.tenancyid + \"/\" + .correlationkey)"
      language: jq
    chainMode:
      type: threshold
      ganglia:
        - trigger-watcher
        - resolve-watcher
      minConfidence: 999.0
    deadline: PT30M
    triggerAction:
      type: create-case
      caseNamespace: ops
      caseName: sla-breach
      caseVersion: "1.0"
```

### Mixed-level — degradation with root-cause correlation

```yaml
situations:
  - situationId: degradation-with-cause
    eventTypes:
      - ras.situation.triggered
      - io.casehub.service.error
    chainMode:
      type: and
      ganglia:
        - service-trigger-watcher
        - error-code-detector
    correlationKey:
      expression: .tenancyid
      language: jq
    triggerAction:
      type: create-case
      caseNamespace: ops
      caseName: degradation-root-cause
      caseVersion: "1.0"
```

## Module Placement

| Artifact | Module | Rationale |
|----------|--------|-----------|
| `GanglionDescriptor.SituationWatcher` | `api/` | Sealed interface variant |
| `SituationDefinition.deadline` field | `api/` | Definition record field |
| `SituationStore.findActiveBySituationId()` | `api/` | SPI query method for deadline job |
| `SituationChangeEventBridge` | `runtime/` | CDI observer + producer |
| `SituationWatcherGanglion` | `runtime/` | Concrete ganglion |
| `DeadlineCheckJob` | `runtime/` | `@Scheduled` bean |
| `SituationEvaluator.triggerByDeadline()` | `runtime/` | New evaluator entry point |
| `SituationEvaluator.executeTrigger()` | `runtime/` | Extracted shared trigger mechanics |
| `CloudEventExpressionContext` update | `runtime/` | Expose all extensions |
| Cycle detection in `register()` + constructor | `runtime/` | Registry validation |
| JPA deadline query | `persistence-jpa/` | Store implementation |
| InMemory deadline query | `persistence-memory/` | Store implementation |
| Flyway migration | `persistence-jpa/` | Add `deadline` column |

## Metrics

| Metric | Type | Tags | Purpose |
|--------|------|------|---------|
| `ras.bridge.events` | counter | `change_type`, `situation_id` | Bridged events emitted |
| `ras.bridge.errors` | counter | `situation_id` | Bridge observer failures (caught) |
| `ras.deadline.checked` | counter | `situation_id` | Situations checked by DeadlineCheckJob |
| `ras.deadline.triggered` | counter | `situation_id`, `tenancy_id` | Deadlines that forced a trigger |
| `ras.cycle.rejected` | counter | `situation_id` | Registrations rejected by cycle validator |

Existing metrics (`ras.event.received`, `ras.event.routed`, `ras.evaluation.*`) fire naturally for bridged events.

## Testing Strategy

| Test | Scope | Verifies |
|------|-------|----------|
| `SituationChangeEventBridgeTest` | Unit | CloudEvent shape, extensions, data serialization, exception isolation |
| `SituationWatcherGanglionTest` | Unit | ChangeType → DetectionSignal mapping, unmapped types not routed, automatic evidence |
| `CycleDetectionTest` | Unit | Direct cycles, transitive cycles, DAG acceptance, self-edge exclusion for FireOnce, self-edge inclusion for Repeating, filter conservatism, constructor-time detection |
| `DeadlineCheckJobTest` | Unit | Finds expired situations, calls triggerByDeadline, respects interval, skips triggered, correlationWindow precedence over deadline |
| `SituationEvaluator.triggerByDeadline()` | Unit | Shared trigger mechanics — claim, case creation, change event, ganglion close |
| `CloudEventExpressionContextTest` | Unit | All extensions exposed, not just tenancyid |
| `MetaSituationIntegrationTest` | Integration | End-to-end: child CloudEvent → child trigger → bridge → meta-situation evaluates → meta-situation triggers |
| `DeadlineIntegrationTest` | Integration | Child triggers → meta-situation created → deadline expires → job forces trigger |
| `NestingIntegrationTest` | Integration | L1 meta-situation triggers → bridge → L2 meta-meta-situation evaluates |
| `AbstractGanglionContractTest` | Contract | SituationWatcherGanglion satisfies Ganglion contract |
| `YamlSituationDefinitionProvider` | Unit | Parses `situation-watcher` type, `deadline` field, `ganglionId` key |
| `SituationReplayRunner` | Unit | `drainAllDeadlines(Instant)` fires expired deadlines at replay end |

## Known Gaps

1. **Cancel-on-resolution:** No mechanism for a cancel event to proactively resolve/discard a pending deadline situation. Threshold ANTI reduces confidence but does not trigger RESOLVE/DISCARD. Follow-on: extend `DefaultRasTriggerPolicy` to return RESOLVE when accumulated confidence drops to zero or below for deadline-enabled situations.

## References

- `RasEngine.java:29` — `onCloudEvent()` routing via `findByEventType()`
- `SituationEvaluator.java:295` — `SituationChangeEvent` firing in `executeDecision()`
- `SituationDefinitionRegistry.java:120` — `buildSnapshot()` / `findByEventType()` routing
- `ChainMode.java:20` — sealed interface, all variants are pure functions of detections
- `DefaultRasTriggerPolicy.java:23` — policy evaluation pattern
- `SituationDefinition.java:13` — `correlationWindow` precedent for temporal fields
- `SituationReplayRunner.java:31` — deterministic replay depends on ChainMode purity
- `CloudEventExpressionContext` — must be updated to expose all extensions (R1-02)
- `DefaultCorrelationKeyExtractor.java` — subject → correlationKey mapping
- `SituationChangeEvent.java:6` — record fields
- `GE-20260730-d54a8f` — CDI `fireAsync().join()` exception propagation asymmetry
- casehubio/casehub-ras#41 — issue
