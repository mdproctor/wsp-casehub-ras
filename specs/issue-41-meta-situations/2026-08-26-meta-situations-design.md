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

Per-situation filtering uses `eventFilter` expressions (e.g., `extensions.situationid == 'child-X'`), keeping the type namespace bounded and stable. Adding or removing situation definitions does not change the event type contract.

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
  - id: service-health-watcher
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
4. Unmapped change types return `NOISE`

### Automatic Evidence

Every `SituationWatcherGanglion` detection includes:
- `childSituationId` — extracted from `situationid` extension
- `childCorrelationKey` — extracted from `correlationkey` extension
- `childChangeType` — extracted from `changetype` extension

Ganglion-level `evidenceTemplates` merge on top, following the existing merge order: auto → per-decision-path → ganglion-level.

### handledEventTypes

`SituationWatcherGanglion.handledEventTypes()` returns the 5 bridged event types (`ras.situation.triggered`, etc.). This is a capability declaration for startup validation — the registry validates that every ganglion referenced by a chain mode can handle at least one of the definition's `eventTypes`.

## Deadline — Temporal Absence

`SituationDefinition.deadline` — optional `Duration` field, alongside `correlationWindow`.

### Semantics

Deadline is a temporal lifecycle property, not a detection pattern. Every existing `ChainMode` variant is a pure function of `SituationContext.detections()` — given the same context, they always return the same result. This is what makes `SituationReplayRunner` deterministic. Placing deadline on the definition (not in a chain mode) preserves this invariant.

Any chain mode can have an optional deadline layered on top. After `firstSignal + deadline` elapses, if the situation hasn't already triggered, resolved, or been discarded, the `DeadlineCheckJob` forces a trigger regardless of chain mode satisfaction.

### SituationDefinition Changes

- New field: `Duration deadline` (`@Nullable`)
- Validated: positive when set (same pattern as `correlationWindow`)
- YAML: `deadline: PT30M` (ISO-8601 Duration)

### DeadlineCheckJob

`@Scheduled` bean in `runtime/`.

- Queries `SituationStore` for active situations where the definition has non-null `deadline`
- Checks `context.firstSignal() + definition.deadline() < Instant.now()`
- When expired: calls `SituationEvaluator.triggerByDeadline(situationId, correlationKey, tenancyId)` — a new entry point that handles trigger mechanics (claim, case creation, change event, ganglion close) without requiring a `CloudEvent` input
- Check interval: configurable via `ras.deadline.check-interval` (default `PT10S`)

### Dual-Path Evaluation

The event-driven path also checks `firstSignal + deadline < eventTime` on every evaluation for deterministic behaviour when events continue arriving. The scheduled job is a backstop for quiescent situations where no further events arrive.

### SituationStore Impact

New query method: `List<SituationContext> findWithDeadline(Instant cutoff)` (or similar). JPA implementation adds an indexed query joining on definitions with non-null deadline. In-memory implementation scans.

## Cycle Detection

Mandatory registration-time validation in `SituationDefinitionRegistry`.

### Algorithm

Build a directed graph:
- **Nodes** = situation definitions
- **Edges** = "situation S₁ can fire → bridge produces change type T → situation S₂ subscribes to T"

Every registered situation can produce 5 bridged change types. An edge exists from S₁ to S₂ if S₂'s `eventTypes` contains any bridged type. The graph has at most N×5 edges — tractable for any realistic deployment.

Standard DFS cycle detection on `register()`. If adding a definition would create a cycle, `register()` throws `IllegalArgumentException`. `deregister()` removes edges — no cycle check needed.

### EventFilter Conservatism

EventFilters are treated as transparent for cycle detection. A definition filtering on `situationid == 'child-X'` still creates edges from all situations. This may produce false-positive rejections but never false negatives. Conservative safety is correct for mandatory validation.

## YAML Examples

### System-wide degradation (Count chain mode)

```yaml
ganglia:
  - id: service-trigger-watcher
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

```yaml
ganglia:
  - id: trigger-watcher
    type: situation-watcher
    changeTypeMapping:
      triggered: DETECTED
  - id: resolve-watcher
    type: situation-watcher
    changeTypeMapping:
      resolved: DETECTED

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
      type: and
      requiredGanglia:
        - trigger-watcher
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
      requiredGanglia:
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
| `SituationStore.findWithDeadline()` | `api/` | SPI query method |
| `SituationChangeEventBridge` | `runtime/` | CDI observer + producer |
| `SituationWatcherGanglion` | `runtime/` | Concrete ganglion |
| `DeadlineCheckJob` | `runtime/` | `@Scheduled` bean |
| `SituationEvaluator.triggerByDeadline()` | `runtime/` | New evaluator entry point |
| Cycle detection in `register()` | `runtime/` | Registry validation |
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
| `SituationWatcherGanglionTest` | Unit | ChangeType → DetectionSignal mapping, unmapped types → NOISE, automatic evidence |
| `CycleDetectionTest` | Unit | Direct cycles, transitive cycles, DAG acceptance, filter conservatism, deregister idempotence |
| `DeadlineCheckJobTest` | Unit | Finds expired situations, calls triggerByDeadline, respects interval, skips triggered |
| `SituationEvaluator.triggerByDeadline()` | Unit | Trigger mechanics without CloudEvent — claim, case creation, change event, ganglion close |
| `MetaSituationIntegrationTest` | Integration | End-to-end: child CloudEvent → child trigger → bridge → meta-situation evaluates → meta-situation triggers |
| `DeadlineIntegrationTest` | Integration | Child triggers → meta-situation created → deadline expires → job forces trigger |
| `NestingIntegrationTest` | Integration | L1 meta-situation triggers → bridge → L2 meta-meta-situation evaluates |
| `AbstractGanglionContractTest` | Contract | SituationWatcherGanglion satisfies Ganglion contract |
| `YamlSituationDefinitionProvider` | Unit | Parses `situation-watcher` type, `deadline` field, bridged event types |
| `SituationReplayRunner` | Unit | Deadline doesn't break replay determinism |

## References

- `RasEngine.java:29` — `onCloudEvent()` routing via `findByEventType()`
- `SituationEvaluator.java:295` — `SituationChangeEvent` firing in `executeDecision()`
- `SituationDefinitionRegistry.java:120` — `buildSnapshot()` / `findByEventType()` routing
- `ChainMode.java:20` — sealed interface, all variants are pure functions of detections
- `DefaultRasTriggerPolicy.java:23` — policy evaluation pattern
- `SituationDefinition.java:13` — `correlationWindow` precedent for temporal fields
- `SituationReplayRunner.java:31` — deterministic replay depends on ChainMode purity
- `DefaultCorrelationKeyExtractor.java` — subject → correlationKey mapping
- `SituationChangeEvent.java:6` — record fields
- `GE-20260730-d54a8f` — CDI `fireAsync().join()` exception propagation asymmetry
- casehubio/casehub-ras#41 — issue
