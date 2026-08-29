# Meta-Situations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #41 — Meta-situations — situations observing other situations
**Issue group:** #41

**Goal:** Enable situations that observe other situations by bridging SituationChangeEvent to CloudEvent, adding a SituationWatcher ganglion descriptor, deadline-based temporal absence, and cycle detection.

**Architecture:** A CDI bridge observer wraps every SituationChangeEvent as a CloudEvent (type `ras.situation.<changeType>`) and fires it via CDI. The existing engine processes bridged events unchanged. Meta-situations are regular SituationDefinitions. A new `GanglionDescriptor.SituationWatcher` enables YAML-configured situation-watching ganglia. `SituationDefinition.deadline` adds temporal absence (force-trigger after timeout). Cycle detection at registration prevents infinite loops.

**Tech Stack:** Java 21, Quarkus (CDI, Scheduler), CloudEvents SDK, Jackson, JPA/Hibernate, Flyway, JUnit 5, AssertJ

## Global Constraints

- `deadline` is a `SituationDefinition` field, NOT stored in the database. No Flyway migration needed.
- Bridged event types: `ras.situation.triggered`, `ras.situation.resolved`, `ras.situation.discarded`, `ras.situation.suppressed`, `ras.situation.dismissed`
- YAML ganglion key is `ganglionId:` (not `id:`) — matches existing parser convention
- `CloudEventExpressionContext` must expose all extensions (not just `tenancyid`)
- Self-edges in cycle detection: excluded for `FireOnce`, included for `Repeating`
- `handledEventTypes()` for SituationWatcher returns only mapped change types
- Bridge fires CloudEvent fire-and-forget (no `.join()`) with try-catch for CDI exception isolation

---

## Batch 1: API Foundation

### Task 1: API types — SituationDefinition.deadline, GanglionDescriptor.SituationWatcher, SituationStore.findActiveBySituationId

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/SituationDefinition.java`
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java`
- Modify: `api/src/main/java/io/casehub/ras/api/SituationStore.java`
- Test: `api/src/test/java/io/casehub/ras/api/SituationDefinitionTest.java`
- Test: `api/src/test/java/io/casehub/ras/api/GanglionDescriptorTest.java`

**Interfaces:**
- Produces: `SituationDefinition.deadline()` — `@Nullable Duration`, validated positive when set
- Produces: `GanglionDescriptor.SituationWatcher` — sealed variant with `ganglionId`, `changeTypeMapping`, `handledEventTypes`, `evidenceTemplates`
- Produces: `SituationStore.findActiveBySituationId(String)` — default method returning `List.of()`

#### SituationDefinition.deadline

- [ ] **Step 1: Write test — deadline validated positive**

```java
@Test
void deadline_rejects_zero() {
    assertThatThrownBy(() -> new SituationDefinition(
            "sit", Set.of("e"), null, null,
            new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            null, null, null, Map.of(), null, Duration.ZERO))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("deadline must be positive");
}

@Test
void deadline_accepts_positive() {
    var def = new SituationDefinition(
            "sit", Set.of("e"), null, null,
            new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            null, null, null, Map.of(), null, Duration.ofMinutes(30));
    assertThat(def.deadline()).isEqualTo(Duration.ofMinutes(30));
}

@Test
void deadline_accepts_null() {
    var def = new SituationDefinition(
            "sit", Set.of("e"), null, null,
            new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            null);
    assertThat(def.deadline()).isNull();
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=SituationDefinitionTest`
Expected: compilation errors (no deadline field yet)

- [ ] **Step 3: Add deadline field to SituationDefinition**

Use `ide_replace_member` to update the record declaration. Add `Duration deadline` as the 12th component after `feedbackConfig`. Update the compact constructor to validate:

```java
if (deadline != null && (deadline.isZero() || deadline.isNegative())) {
    throw new IllegalArgumentException(
            "deadline must be positive when set, got: " + deadline);
}
```

Update both convenience constructors to pass `null` for deadline. Update `withChainMode()` to carry deadline through all 12 fields.

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=SituationDefinitionTest`
Expected: PASS

- [ ] **Step 5: Verify no compile errors across all modules**

Run: `mvn --batch-mode compile -DskipTests`

Fix any callers that use the canonical or convenience constructors. Common sites:
- `YamlSituationDefinitionProvider.parseSituation()` — add `null` for deadline (YAML parsing comes in Batch 3)
- `SituationEvaluator` — `definition.withChainMode()` calls
- Test files constructing SituationDefinition directly

#### GanglionDescriptor.SituationWatcher

- [ ] **Step 6: Write test — SituationWatcher record construction**

```java
@Test
void situationWatcher_handledEventTypes_derived_from_mapping() {
    var mapping = Map.of(
        SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED,
        SituationChangeEvent.ChangeType.RESOLVED, DetectionSignal.ANTI);
    var watcher = new GanglionDescriptor.SituationWatcher(
            "watcher-1", mapping, Map.of());
    assertThat(watcher.ganglionId()).isEqualTo("watcher-1");
    assertThat(watcher.handledEventTypes()).containsExactlyInAnyOrder(
            "ras.situation.triggered", "ras.situation.resolved");
}

@Test
void situationWatcher_single_mapping() {
    var mapping = Map.of(
        SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED);
    var watcher = new GanglionDescriptor.SituationWatcher(
            "watcher-2", mapping, Map.of());
    assertThat(watcher.handledEventTypes()).containsExactly("ras.situation.triggered");
}
```

- [ ] **Step 7: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=GanglionDescriptorTest`
Expected: compilation errors (no SituationWatcher variant)

- [ ] **Step 8: Add SituationWatcher variant to GanglionDescriptor**

Use `ide_insert_member` to add after the `ExpressionRules` record. The sealed interface must list it in permits (implicit for nested records).

```java
record SituationWatcher(
        String ganglionId,
        Map<SituationChangeEvent.ChangeType, DetectionSignal> changeTypeMapping,
        Map<String, ExpressionEvaluator> evidenceTemplates
) implements GanglionDescriptor {

    private static final String EVENT_TYPE_PREFIX = "ras.situation.";

    @Override
    public Set<String> handledEventTypes() {
        return changeTypeMapping.keySet().stream()
                .map(ct -> EVENT_TYPE_PREFIX + ct.name().toLowerCase())
                .collect(java.util.stream.Collectors.toUnmodifiableSet());
    }
}
```

- [ ] **Step 9: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=GanglionDescriptorTest`
Expected: PASS

#### SituationStore.findActiveBySituationId

- [ ] **Step 10: Add default method to SituationStore**

Use `ide_insert_member` on `SituationStore`:

```java
default List<SituationContext> findActiveBySituationId(String situationId) {
    return List.of();
}
```

- [ ] **Step 11: Run full compile**

Run: `mvn --batch-mode compile -DskipTests`
Expected: compiles clean

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add api/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): API types — deadline field, SituationWatcher descriptor, findActiveBySituationId

Adds Duration deadline to SituationDefinition (nullable, validated positive).
Adds GanglionDescriptor.SituationWatcher sealed variant with changeTypeMapping.
Adds SituationStore.findActiveBySituationId default method.

Refs #41"
```

---

### Task 2: CloudEventExpressionContext — expose all extensions

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/CloudEventExpressionContext.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/CloudEventExpressionContextTest.java`

**Interfaces:**
- Consumes: `CloudEvent.getExtensionNames()` (CloudEvents SDK)
- Produces: Expression context map with all extensions as flat top-level keys

- [ ] **Step 1: Write test — all extensions exposed**

```java
@Test
void build_exposes_all_extensions() {
    CloudEvent event = CloudEventBuilder.v1()
            .withId("1")
            .withType("test")
            .withSource(URI.create("test://source"))
            .withExtension("tenancyid", "t1")
            .withExtension("situationid", "service-health")
            .withExtension("correlationkey", "server-1")
            .withExtension("changetype", "TRIGGERED")
            .build();
    Map<String, Object> ctx = CloudEventExpressionContext.build(event);
    assertThat(ctx.get("tenancyid")).isEqualTo("t1");
    assertThat(ctx.get("situationid")).isEqualTo("service-health");
    assertThat(ctx.get("correlationkey")).isEqualTo("server-1");
    assertThat(ctx.get("changetype")).isEqualTo("TRIGGERED");
}

@Test
void build_exposes_standard_fields() {
    CloudEvent event = CloudEventBuilder.v1()
            .withId("1")
            .withType("test.type")
            .withSource(URI.create("test://source"))
            .withSubject("sub")
            .build();
    Map<String, Object> ctx = CloudEventExpressionContext.build(event);
    assertThat(ctx.get("type")).isEqualTo("test.type");
    assertThat(ctx.get("subject")).isEqualTo("sub");
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=CloudEventExpressionContextTest`
Expected: FAIL — `situationid` is null

- [ ] **Step 3: Update CloudEventExpressionContext.build() to expose all extensions**

Use `ide_replace_member` on `build` method. Replace the hardcoded `tenancyid` line with a loop:

```java
static Map<String, Object> build(CloudEvent event) {
    Map<String, Object> ctx = new LinkedHashMap<>();
    ctx.put("type", event.getType());
    ctx.put("source", event.getSource() != null ? event.getSource().toString() : null);
    ctx.put("subject", event.getSubject());
    ctx.put("id", event.getId());
    ctx.put("time", event.getTime() != null ? event.getTime().toString() : null);
    for (String name : event.getExtensionNames()) {
        ctx.put(name, event.getExtension(name));
    }
    ctx.put("data", parseJsonData(event));
    return ctx;
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CloudEventExpressionContextTest`
Expected: PASS

- [ ] **Step 5: Run full test suite to verify no regressions**

Run: `mvn --batch-mode test -pl runtime`
Expected: all tests pass (existing `tenancyid` usage still works via the generic loop)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): CloudEventExpressionContext — expose all CloudEvent extensions

Replaces hardcoded tenancyid with generic loop over getExtensionNames().
Enables eventFilter and correlationKey expressions to reference bridge
extensions (situationid, correlationkey, changetype).

Refs #41"
```

---

## Batch 2: Bridge + SituationWatcherGanglion

### Task 3: SituationChangeEventBridge

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/SituationChangeEventBridge.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/SituationChangeEventBridgeTest.java`

**Interfaces:**
- Consumes: `SituationChangeEvent` (api/), `CloudEventBuilder` (CloudEvents SDK)
- Produces: Bridged `CloudEvent` with type `ras.situation.<changeType>`, extensions `situationid`, `correlationkey`, `changetype`, `tenancyid`

- [ ] **Step 1: Write test — bridged CloudEvent shape**

```java
@Test
void bridge_produces_correct_cloud_event_shape() {
    var context = SituationContext.initial("sit-1", "key-1", "tenant-1",
            Instant.parse("2026-01-01T00:00:00Z"));
    var change = new SituationChangeEvent("tenant-1", "sit-1", "key-1",
            SituationChangeEvent.ChangeType.TRIGGERED, context);

    var bridge = new SituationChangeEventBridge(collectingEvent, null);
    bridge.onSituationChange(change);

    assertThat(collectingEvent.fired).hasSize(1);
    CloudEvent bridged = collectingEvent.fired.get(0);
    assertThat(bridged.getType()).isEqualTo("ras.situation.triggered");
    assertThat(bridged.getSubject()).isEqualTo("sit-1");
    assertThat(bridged.getExtension("tenancyid")).isEqualTo("tenant-1");
    assertThat(bridged.getExtension("situationid")).isEqualTo("sit-1");
    assertThat(bridged.getExtension("correlationkey")).isEqualTo("key-1");
    assertThat(bridged.getExtension("changetype")).isEqualTo("TRIGGERED");
    assertThat(bridged.getSource().toString()).isEqualTo("ras://bridge");
}

@Test
void bridge_maps_all_change_types() {
    for (SituationChangeEvent.ChangeType ct : SituationChangeEvent.ChangeType.values()) {
        var context = SituationContext.initial("s", "k", "t", Instant.now());
        var change = new SituationChangeEvent("t", "s", "k", ct, context);
        var bridge = new SituationChangeEventBridge(collectingEvent, null);
        collectingEvent.fired.clear();
        bridge.onSituationChange(change);
        assertThat(collectingEvent.fired.get(0).getType())
                .isEqualTo("ras.situation." + ct.name().toLowerCase());
    }
}

@Test
void bridge_swallows_exceptions() {
    Event<CloudEvent> failingEvent = /* throws on fireAsync */;
    var bridge = new SituationChangeEventBridge(failingEvent, null);
    var context = SituationContext.initial("s", "k", "t", Instant.now());
    var change = new SituationChangeEvent("t", "s", "k",
            SituationChangeEvent.ChangeType.TRIGGERED, context);
    assertThatCode(() -> bridge.onSituationChange(change)).doesNotThrowAnyException();
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationChangeEventBridgeTest`
Expected: compilation errors (class doesn't exist)

- [ ] **Step 3: Implement SituationChangeEventBridge**

Create file via `ide_create_file`:

```java
package io.casehub.ras.runtime;

import io.casehub.ras.api.SituationChangeEvent;
import io.casehub.ras.api.SituationContext;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import java.net.URI;
import java.util.UUID;
import java.util.logging.Logger;

@ApplicationScoped
public class SituationChangeEventBridge {

    private static final Logger LOG = Logger.getLogger(SituationChangeEventBridge.class.getName());
    private static final URI BRIDGE_SOURCE = URI.create("ras://bridge");

    private final Event<CloudEvent> cloudEvent;
    private final RasMetrics metrics;

    @Inject
    public SituationChangeEventBridge(Event<CloudEvent> cloudEvent, RasMetrics metrics) {
        this.cloudEvent = cloudEvent;
        this.metrics = metrics;
    }

    void onSituationChange(@ObservesAsync SituationChangeEvent change) {
        try {
            String changeTypeLower = change.changeType().name().toLowerCase();
            SituationContext ctx = change.context();

            CloudEvent bridged = CloudEventBuilder.v1()
                    .withId(UUID.randomUUID().toString())
                    .withType("ras.situation." + changeTypeLower)
                    .withSource(BRIDGE_SOURCE)
                    .withSubject(change.situationId())
                    .withTime(java.time.OffsetDateTime.now())
                    .withExtension("tenancyid", change.tenancyId())
                    .withExtension("situationid", change.situationId())
                    .withExtension("correlationkey", change.correlationKey())
                    .withExtension("changetype", change.changeType().name())
                    .withDataContentType("application/json")
                    .withData(serializeSummary(change, ctx))
                    .build();

            cloudEvent.fireAsync(bridged);
            if (metrics != null) {
                metrics.bridgeEventEmitted(change.situationId(), changeTypeLower);
            }
        } catch (Exception ex) {
            LOG.warning("SituationChangeEventBridge failed for situation '"
                        + change.situationId() + "': " + ex.getMessage());
            if (metrics != null) {
                metrics.bridgeEventFailed(change.situationId());
            }
        }
    }

    private byte[] serializeSummary(SituationChangeEvent change, SituationContext ctx) {
        String json = String.format(
                "{\"situationId\":\"%s\",\"correlationKey\":\"%s\",\"tenancyId\":\"%s\","
                + "\"changeType\":\"%s\",\"firstSignal\":\"%s\",\"lastSignal\":\"%s\","
                + "\"triggerCount\":%d,\"detectionCount\":%d}",
                change.situationId(), change.correlationKey(), change.tenancyId(),
                change.changeType().name(),
                ctx.firstSignal(), ctx.lastSignal(),
                ctx.triggerCount(), ctx.detections().size());
        return json.getBytes(java.nio.charset.StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 4: Add bridge metrics to RasMetrics**

Use `ide_insert_member` on `RasMetrics` to add `bridgeEventEmitted(String, String)` and `bridgeEventFailed(String)` methods.

- [ ] **Step 5: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationChangeEventBridgeTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): SituationChangeEventBridge — CDI observer bridging to CloudEvent

Observes @ObservesAsync SituationChangeEvent, wraps as CloudEvent with
type ras.situation.<changeType>, extensions for situationid/correlationkey/
changetype, JSON summary data. Fire-and-forget with try-catch isolation.

Refs #41"
```

---

### Task 4: SituationWatcherGanglion + registry construction

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/SituationWatcherGanglion.java`
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/SituationWatcherGanglionTest.java`

**Interfaces:**
- Consumes: `GanglionDescriptor.SituationWatcher` (Task 1), `CloudEvent` extensions
- Produces: `SituationWatcherGanglion implements Ganglion` — maps bridged CloudEvent changeType → DetectionSignal

- [ ] **Step 1: Write test — changeType mapping**

```java
@Test
void detect_maps_triggered_to_detected() {
    var mapping = Map.of(SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED);
    var ganglion = new SituationWatcherGanglion("watcher", mapping);
    CloudEvent event = bridgedEvent("sit-1", "key-1", "tenant-1", "TRIGGERED");
    var ctx = SituationContext.initial("meta", "key", "t", Instant.now());
    DetectionResult result = ganglion.detect(event, ctx);
    assertThat(result.signal()).isEqualTo(DetectionSignal.DETECTED);
    assertThat(result.confidence()).isEqualTo(1.0);
    assertThat(result.ganglionId()).isEqualTo("watcher");
}

@Test
void detect_maps_resolved_to_anti() {
    var mapping = Map.of(
            SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED,
            SituationChangeEvent.ChangeType.RESOLVED, DetectionSignal.ANTI);
    var ganglion = new SituationWatcherGanglion("watcher", mapping);
    CloudEvent event = bridgedEvent("sit-1", "key-1", "tenant-1", "RESOLVED");
    var ctx = SituationContext.initial("meta", "key", "t", Instant.now());
    DetectionResult result = ganglion.detect(event, ctx);
    assertThat(result.signal()).isEqualTo(DetectionSignal.ANTI);
}

@Test
void detect_includes_automatic_evidence() {
    var mapping = Map.of(SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED);
    var ganglion = new SituationWatcherGanglion("watcher", mapping);
    CloudEvent event = bridgedEvent("child-sit", "child-key", "tenant-1", "TRIGGERED");
    var ctx = SituationContext.initial("meta", "key", "t", Instant.now());
    DetectionResult result = ganglion.detect(event, ctx);
    assertThat(result.evidence()).containsEntry("childSituationId", "child-sit");
    assertThat(result.evidence()).containsEntry("childCorrelationKey", "child-key");
    assertThat(result.evidence()).containsEntry("childChangeType", "TRIGGERED");
}

@Test
void handledEventTypes_derived_from_mapping() {
    var mapping = Map.of(SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED);
    var ganglion = new SituationWatcherGanglion("watcher", mapping);
    assertThat(ganglion.handledEventTypes()).containsExactly("ras.situation.triggered");
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationWatcherGanglionTest`
Expected: compilation errors

- [ ] **Step 3: Implement SituationWatcherGanglion**

Create file via `ide_create_file`:

```java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import io.cloudevents.CloudEvent;

import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class SituationWatcherGanglion implements Ganglion {

    private static final String EVENT_TYPE_PREFIX = "ras.situation.";

    private final String ganglionId;
    private final Map<SituationChangeEvent.ChangeType, DetectionSignal> changeTypeMapping;
    private final Set<String> handledTypes;

    public SituationWatcherGanglion(String ganglionId,
                                     Map<SituationChangeEvent.ChangeType, DetectionSignal> changeTypeMapping) {
        this.ganglionId = ganglionId;
        this.changeTypeMapping = Map.copyOf(changeTypeMapping);
        this.handledTypes = changeTypeMapping.keySet().stream()
                .map(ct -> EVENT_TYPE_PREFIX + ct.name().toLowerCase())
                .collect(Collectors.toUnmodifiableSet());
    }

    @Override
    public String ganglionId() { return ganglionId; }

    @Override
    public Set<String> handledEventTypes() { return handledTypes; }

    @Override
    public DetectionResult detect(CloudEvent event, SituationContext context) {
        Object changeTypeExt = event.getExtension("changetype");
        if (changeTypeExt == null) {
            return new DetectionResult(ganglionId, 0.0, DetectionSignal.NOISE, Map.of());
        }

        SituationChangeEvent.ChangeType changeType;
        try {
            changeType = SituationChangeEvent.ChangeType.valueOf(changeTypeExt.toString());
        } catch (IllegalArgumentException e) {
            return new DetectionResult(ganglionId, 0.0, DetectionSignal.NOISE, Map.of());
        }

        DetectionSignal signal = changeTypeMapping.getOrDefault(changeType, DetectionSignal.NOISE);
        Map<String, Object> evidence = Map.of(
                "childSituationId", String.valueOf(event.getExtension("situationid")),
                "childCorrelationKey", String.valueOf(event.getExtension("correlationkey")),
                "childChangeType", changeTypeExt.toString());

        return new DetectionResult(ganglionId, 1.0, signal, evidence);
    }
}
```

- [ ] **Step 4: Add SituationWatcher construction to SituationDefinitionRegistry**

Use `ide_insert_member` to add a `constructSituationWatcher` method after `constructExpressionRules`:

```java
private Ganglion constructSituationWatcher(GanglionDescriptor.SituationWatcher sw) {
    return new SituationWatcherGanglion(sw.ganglionId(), sw.changeTypeMapping());
}
```

Update `constructGanglion` to add a case for `SituationWatcher`:

```java
case GanglionDescriptor.SituationWatcher sw -> constructSituationWatcher(sw);
```

If `evidenceTemplates` are present, wrap with `EvidenceExtractingGanglion` (same pattern as NaiveBayes and ExpressionRules).

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationWatcherGanglionTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): SituationWatcherGanglion — maps bridged changeType to DetectionSignal

Concrete ganglion for GanglionDescriptor.SituationWatcher. Extracts
changetype from CloudEvent extensions, maps via changeTypeMapping,
includes automatic evidence (childSituationId, childCorrelationKey,
childChangeType). handledEventTypes derived from mapping keys.

Refs #41"
```

---

## Batch 3: YAML Parsing + Cycle Detection

### Task 5: YAML parsing — situation-watcher ganglion type + deadline field

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: `GanglionDescriptor.SituationWatcher` (Task 1), `SituationDefinition.deadline` (Task 1)
- Produces: YAML parsing for `type: situation-watcher` ganglia and `deadline:` field on situations

- [ ] **Step 1: Write test — parse situation-watcher ganglion**

```java
@Test
void parses_situation_watcher_ganglion() {
    String yaml = """
        ganglia:
          - ganglionId: sw-1
            type: situation-watcher
            changeTypeMapping:
              triggered: DETECTED
              resolved: ANTI
        situations:
          - situationId: meta-1
            eventTypes: [ras.situation.triggered]
            chainMode:
              type: count
              ganglionId: sw-1
              requiredCount: 3
            triggerAction:
              type: notify-only
        """;
    var provider = new YamlSituationDefinitionProvider(
            new ByteArrayInputStream(yaml.getBytes()));
    assertThat(provider.ganglionDescriptors()).hasSize(1);
    var desc = provider.ganglionDescriptors().get(0);
    assertThat(desc).isInstanceOf(GanglionDescriptor.SituationWatcher.class);
    var sw = (GanglionDescriptor.SituationWatcher) desc;
    assertThat(sw.ganglionId()).isEqualTo("sw-1");
    assertThat(sw.changeTypeMapping()).containsEntry(
            SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED);
    assertThat(sw.changeTypeMapping()).containsEntry(
            SituationChangeEvent.ChangeType.RESOLVED, DetectionSignal.ANTI);
}
```

- [ ] **Step 2: Write test — parse deadline field**

```java
@Test
void parses_deadline_field() {
    String yaml = """
        ganglia:
          - ganglionId: g1
            type: situation-watcher
            changeTypeMapping:
              triggered: DETECTED
        situations:
          - situationId: sla-1
            eventTypes: [ras.situation.triggered]
            chainMode:
              type: count
              ganglionId: g1
              requiredCount: 1
            deadline: PT30M
            triggerAction:
              type: notify-only
        """;
    var provider = new YamlSituationDefinitionProvider(
            new ByteArrayInputStream(yaml.getBytes()));
    assertThat(provider.registrations()).hasSize(1);
    assertThat(provider.registrations().get(0).definition().deadline())
            .isEqualTo(Duration.ofMinutes(30));
}

@Test
void deadline_absent_defaults_to_null() {
    String yaml = """
        situations:
          - situationId: normal-1
            eventTypes: [some.event]
            chainMode:
              type: count
              ganglionId: g1
              requiredCount: 1
            triggerAction:
              type: notify-only
        """;
    var provider = new YamlSituationDefinitionProvider(
            new ByteArrayInputStream(yaml.getBytes()));
    assertThat(provider.registrations().get(0).definition().deadline()).isNull();
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: FAIL

- [ ] **Step 4: Implement situation-watcher YAML parsing**

Use `ide_insert_member` to add `parseSituationWatcherGanglion` method in `YamlSituationDefinitionProvider` after `parseExpressionRulesGanglion`:

```java
private static GanglionDescriptor.SituationWatcher parseSituationWatcherGanglion(
        Map<String, Object> map) {
    String ganglionId = requireString(map, "ganglionId");
    @SuppressWarnings("unchecked")
    Map<String, Object> mappingRaw = (Map<String, Object>) map.get("changeTypeMapping");
    if (mappingRaw == null || mappingRaw.isEmpty()) {
        throw new IllegalArgumentException(
                "situation-watcher ganglion '" + ganglionId + "' requires non-empty changeTypeMapping");
    }
    Map<SituationChangeEvent.ChangeType, DetectionSignal> mapping = new LinkedHashMap<>();
    for (var entry : mappingRaw.entrySet()) {
        var changeType = SituationChangeEvent.ChangeType.valueOf(entry.getKey().toUpperCase());
        var signal = DetectionSignal.valueOf(entry.getValue().toString().toUpperCase());
        mapping.put(changeType, signal);
    }
    Map<String, ExpressionEvaluator> evidenceTemplates = parseEvidenceTemplates(map);
    return new GanglionDescriptor.SituationWatcher(ganglionId, mapping, evidenceTemplates);
}
```

Update `parseGanglia` to add a case for `"situation-watcher"`:

```java
case "situation-watcher" -> descriptors.add(parseSituationWatcherGanglion(map));
```

- [ ] **Step 5: Implement deadline YAML parsing**

In `parseSituation`, after parsing `feedbackConfig`, add deadline parsing:

```java
Duration deadline = null;
Object deadlineRaw = map.get("deadline");
if (deadlineRaw != null) {
    deadline = Duration.parse(deadlineRaw.toString());
}
```

Pass `deadline` to the `SituationDefinition` constructor.

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): YAML parsing — situation-watcher ganglion type + deadline field

Parses type: situation-watcher in ganglia section with changeTypeMapping.
Parses optional deadline: ISO-8601 Duration on situation definitions.

Refs #41"
```

---

### Task 6: Cycle detection in SituationDefinitionRegistry

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationDefinitionRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/SituationDefinitionRegistryTest.java`

**Interfaces:**
- Consumes: `SituationRegistration.definition().eventTypes()`, `TriggerMode` (FireOnce/Repeating)
- Produces: `IllegalArgumentException` on cycle detection at registration time

- [ ] **Step 1: Write tests — cycle detection**

```java
@Test
void register_rejects_direct_cycle_with_repeating() {
    // A observes ras.situation.triggered, A is Repeating with Count(g,1)
    // A triggers → bridge → A re-triggers → infinite loop
    var defA = new SituationDefinition("A", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            new TriggerMode.Repeating(Duration.ofMinutes(1)));
    var regA = new SituationRegistration(defA, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    var registry = SituationDefinitionRegistry.forTesting(
            List.of(() -> List.of(regA)), List.of(mockGanglion("g")));
    // Constructor should throw — cycle detected
    // (adjust test to use dynamic register if constructor doesn't validate)
}

@Test
void register_allows_self_edge_with_fire_once() {
    var defA = new SituationDefinition("A", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(), null); // FireOnce default
    var regA = new SituationRegistration(defA, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    assertThatCode(() -> SituationDefinitionRegistry.forTesting(
            List.of(() -> List.of(regA)), List.of(mockGanglion("g"))))
        .doesNotThrowAnyException();
}

@Test
void register_rejects_transitive_cycle() {
    // A→B→A: A observes ras.situation.triggered (from B),
    // B observes ras.situation.triggered (from A)
    var defA = new SituationDefinition("A", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            new TriggerMode.Repeating(Duration.ofMinutes(1)));
    var defB = new SituationDefinition("B", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(),
            new TriggerMode.Repeating(Duration.ofMinutes(1)));
    var regA = new SituationRegistration(defA, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    var regB = new SituationRegistration(defB, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    assertThatThrownBy(() -> SituationDefinitionRegistry.forTesting(
            List.of(() -> List.of(regA, regB)), List.of(mockGanglion("g"))))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void register_allows_dag() {
    // A observes ras.situation.triggered, B observes raw CloudEvents only
    // No cycle: A→(nothing that feeds A)
    var defA = new SituationDefinition("A", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(), null);
    var defB = new SituationDefinition("B", Set.of("some.raw.event"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(), null);
    var regA = new SituationRegistration(defA, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    var regB = new SituationRegistration(defB, DefaultCorrelationKeyExtractor.INSTANCE, null, null);
    assertThatCode(() -> SituationDefinitionRegistry.forTesting(
            List.of(() -> List.of(regA, regB)), List.of(mockGanglion("g"))))
        .doesNotThrowAnyException();
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationDefinitionRegistryTest`
Expected: FAIL (no cycle detection yet)

- [ ] **Step 3: Implement cycle detection**

Use `ide_insert_member` to add cycle detection methods to `SituationDefinitionRegistry`:

```java
private static final Set<String> BRIDGED_EVENT_TYPES = Set.of(
        "ras.situation.triggered", "ras.situation.resolved",
        "ras.situation.discarded", "ras.situation.suppressed",
        "ras.situation.dismissed");

private void validateNoCycles(List<SituationRegistration> registrations) {
    Map<String, SituationRegistration> byId = new LinkedHashMap<>();
    for (var reg : registrations) {
        byId.put(reg.definition().situationId(), reg);
    }

    // Build adjacency: S1 → S2 if S2 subscribes to any bridged type
    // (and S1 can produce bridged types, which all situations can)
    Map<String, Set<String>> adjacency = new LinkedHashMap<>();
    for (var reg : registrations) {
        adjacency.put(reg.definition().situationId(), new LinkedHashSet<>());
    }

    for (var target : registrations) {
        boolean subscribesBridged = target.definition().eventTypes().stream()
                .anyMatch(BRIDGED_EVENT_TYPES::contains);
        if (!subscribesBridged) continue;

        for (var source : registrations) {
            String sourceId = source.definition().situationId();
            String targetId = target.definition().situationId();

            if (sourceId.equals(targetId)) {
                // Self-edge: only include for Repeating trigger mode
                if (target.definition().triggerMode() instanceof TriggerMode.Repeating) {
                    adjacency.get(sourceId).add(targetId);
                }
                continue;
            }
            adjacency.get(sourceId).add(targetId);
        }
    }

    // DFS cycle detection
    Set<String> visited = new LinkedHashSet<>();
    Set<String> onStack = new LinkedHashSet<>();
    for (String node : adjacency.keySet()) {
        if (hasCycleDfs(node, adjacency, visited, onStack)) {
            throw new IllegalArgumentException(
                    "Cycle detected in meta-situation graph involving: " + onStack);
        }
    }
}

private boolean hasCycleDfs(String node, Map<String, Set<String>> adj,
                             Set<String> visited, Set<String> onStack) {
    if (onStack.contains(node)) return true;
    if (visited.contains(node)) return false;
    visited.add(node);
    onStack.add(node);
    for (String neighbor : adj.getOrDefault(node, Set.of())) {
        if (hasCycleDfs(neighbor, adj, visited, onStack)) return true;
    }
    onStack.remove(node);
    return false;
}
```

Call `validateNoCycles()` at the end of the constructor (Phase 3), and in `register()` before adding to the snapshot.

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationDefinitionRegistryTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): cycle detection — registration-time DFS validation

Validates meta-situation observation graph at registration time (both
constructor Phase 3 and dynamic register()). Self-edges excluded for
FireOnce, included for Repeating. EventFilters treated as transparent
(conservative — false-positive rejections possible, false negatives not).

Refs #41"
```

---

## Batch 4: Deadline Execution

### Task 7: SituationEvaluator — trigger extraction + triggerByDeadline

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationEvaluator.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/SituationEvaluatorTest.java`

**Interfaces:**
- Consumes: `SituationStore`, `CaseTrigger`, `RasTriggerPolicy`, `SituationDefinition.deadline()`
- Produces: `triggerByDeadline(String situationId, String correlationKey, String tenancyId)` — public entry point for DeadlineCheckJob

- [ ] **Step 1: Write test — triggerByDeadline fires case and change event**

```java
@Test
void triggerByDeadline_fires_case_and_change_event() {
    var def = new SituationDefinition("sit", Set.of("e"), null, null,
            new ChainMode.Count("g", 1),
            new TriggerAction.CreateCase(new CaseTriggerConfig("ns", "name", "1.0", Map.of())),
            null, null, null, Map.of(), null, Duration.ofMinutes(30));
    // Register definition, save a context with firstSignal in the past
    registry.register(new SituationRegistration(def,
            DefaultCorrelationKeyExtractor.INSTANCE, null, null));
    var context = SituationContext.initial("sit", "key", "tenant",
            Instant.now().minus(Duration.ofHours(1)));
    store.save(context);

    evaluator.triggerByDeadline("sit", "key", "tenant");

    assertThat(firedChanges).hasSize(1);
    assertThat(firedChanges.get(0).changeType())
            .isEqualTo(SituationChangeEvent.ChangeType.TRIGGERED);
    assertThat(caseTrigger.fired()).isTrue();
}

@Test
void triggerByDeadline_discards_when_correlationWindow_expired() {
    var def = new SituationDefinition("sit", Set.of("e"),
            Duration.ofMinutes(5), null,
            new ChainMode.Count("g", 1),
            new TriggerAction.CreateCase(new CaseTriggerConfig("ns", "name", "1.0", Map.of())),
            null, null, null, Map.of(), null, Duration.ofMinutes(30));
    registry.register(new SituationRegistration(def,
            DefaultCorrelationKeyExtractor.INSTANCE, null, null));
    var context = SituationContext.initial("sit", "key", "tenant",
            Instant.now().minus(Duration.ofHours(1)));
    store.save(context);

    evaluator.triggerByDeadline("sit", "key", "tenant");

    // Should discard, not trigger — correlationWindow (5m) expired long ago
    assertThat(caseTrigger.fired()).isFalse();
    assertThat(store.find("sit", "key", "tenant")).isEmpty();
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationEvaluatorTest#triggerByDeadline*`
Expected: compilation errors

- [ ] **Step 3: Extract shared trigger mechanics into executeTrigger()**

Use `ide_insert_member` to add a private `executeTrigger` method. Refactor the TRIGGER case in `executeDecision()` to call it. The method handles: claim → save → case trigger/notify → change event → close ganglia, including storeVersion-based bifurcation and error recovery.

- [ ] **Step 4: Implement triggerByDeadline**

Use `ide_insert_member` to add:

```java
public void triggerByDeadline(String situationId, String correlationKey, String tenancyId) {
    Optional<SituationRegistration> regOpt = registry.findBySituationId(situationId);
    if (regOpt.isEmpty()) return;
    SituationDefinition definition = regOpt.get().definition();

    Optional<SituationContext> ctxOpt = store.find(situationId, correlationKey, tenancyId);
    if (ctxOpt.isEmpty()) return;
    SituationContext context = ctxOpt.get();

    // correlationWindow precedence: if expired, discard instead of triggering
    if (definition.correlationWindow() != null
            && context.firstSignal().plus(definition.correlationWindow()).isBefore(Instant.now())) {
        closeGanglia(definition, situationId, correlationKey, tenancyId);
        store.remove(situationId, correlationKey, tenancyId);
        changeEvent.fireAsync(new SituationChangeEvent(
                tenancyId, situationId, correlationKey,
                SituationChangeEvent.ChangeType.DISCARDED, context));
        return;
    }

    executeTrigger(context, definition, situationId, correlationKey, tenancyId, Instant.now());
}
```

- [ ] **Step 5: Add deadline check to event-driven path**

In `processEvent()`, after the `PolicyDecision` evaluation, add a deadline check for deterministic evaluation when events continue arriving:

```java
if (definition.deadline() != null
        && context.firstSignal().plus(definition.deadline()).isBefore(eventTime)
        && policyDecision.decision() == TriggerDecision.CONTINUE_ACCUMULATING) {
    executeTrigger(context, definition, situationId, correlationKey, tenancyId, eventTime);
    metrics.stopProcessTimer(timer, situationId, tenancyId);
    return true;
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationEvaluatorTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): triggerByDeadline + executeTrigger extraction

Extracts shared trigger mechanics from executeDecision into executeTrigger.
Adds triggerByDeadline public entry point for DeadlineCheckJob. Checks
correlationWindow precedence (discard if expired). Event-driven path checks
deadline for deterministic behaviour when events continue arriving.

Refs #41"
```

---

### Task 8: DeadlineCheckJob + SituationStore implementations

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/DeadlineCheckJob.java`
- Create: `runtime/src/test/java/io/casehub/ras/runtime/DeadlineCheckJobTest.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ras/persistence/memory/InMemorySituationStore.java`
- Modify: `persistence-jpa/src/main/java/io/casehub/ras/persistence/jpa/JpaSituationStore.java`

**Interfaces:**
- Consumes: `SituationDefinitionRegistry.allSituationIds()`, `SituationStore.findActiveBySituationId()`, `SituationEvaluator.triggerByDeadline()`
- Produces: `@Scheduled` job that checks and triggers expired deadlines

- [ ] **Step 1: Write test — DeadlineCheckJob triggers expired situations**

```java
@Test
void check_triggers_expired_deadline() {
    var def = new SituationDefinition("sit", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(), null,
            null, null, Map.of(), null, Duration.ofMinutes(5));
    // Context with firstSignal 10 minutes ago (past deadline)
    var context = SituationContext.initial("sit", "key", "t",
            Instant.now().minus(Duration.ofMinutes(10)));
    store.save(context);
    registry.register(new SituationRegistration(def,
            DefaultCorrelationKeyExtractor.INSTANCE, null, null));

    job.check();

    assertThat(evaluator.triggerByDeadlineCalls()).containsExactly("sit/key/t");
}

@Test
void check_skips_non_expired_deadline() {
    var def = new SituationDefinition("sit", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 1),
            new TriggerAction.NotifyOnly(), null,
            null, null, Map.of(), null, Duration.ofMinutes(30));
    var context = SituationContext.initial("sit", "key", "t",
            Instant.now().minus(Duration.ofMinutes(5)));
    store.save(context);
    registry.register(new SituationRegistration(def,
            DefaultCorrelationKeyExtractor.INSTANCE, null, null));

    job.check();

    assertThat(evaluator.triggerByDeadlineCalls()).isEmpty();
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=DeadlineCheckJobTest`
Expected: compilation errors

- [ ] **Step 3: Implement DeadlineCheckJob**

Create via `ide_create_file`:

```java
package io.casehub.ras.runtime;

import io.casehub.ras.api.SituationContext;
import io.casehub.ras.api.SituationDefinition;
import io.casehub.ras.api.SituationStore;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.List;
import java.util.logging.Logger;

@ApplicationScoped
public class DeadlineCheckJob {

    private static final Logger LOG = Logger.getLogger(DeadlineCheckJob.class.getName());

    private final SituationStore store;
    private final SituationDefinitionRegistry registry;
    private final SituationEvaluator evaluator;
    private final RasMetrics metrics;

    @Inject
    public DeadlineCheckJob(SituationStore store, SituationDefinitionRegistry registry,
                            SituationEvaluator evaluator, RasMetrics metrics) {
        this.store = store;
        this.registry = registry;
        this.evaluator = evaluator;
        this.metrics = metrics;
    }

    @Scheduled(every = "${ras.deadline.check-interval:PT10S}")
    void check() {
        Instant now = Instant.now();
        for (String situationId : registry.allSituationIds()) {
            SituationDefinition def = registry.definition(situationId);
            if (def == null || def.deadline() == null) continue;

            List<SituationContext> active = store.findActiveBySituationId(situationId);
            for (SituationContext ctx : active) {
                metrics.deadlineChecked(situationId);
                if (ctx.firstSignal().plus(def.deadline()).isBefore(now)) {
                    try {
                        evaluator.triggerByDeadline(
                                situationId, ctx.correlationKey(), ctx.tenancyId());
                        metrics.deadlineTriggered(situationId, ctx.tenancyId());
                    } catch (Exception ex) {
                        LOG.warning("Deadline trigger failed for situation '"
                                    + situationId + "': " + ex.getMessage());
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 4: Implement InMemorySituationStore.findActiveBySituationId**

Use `ide_insert_member` on `InMemorySituationStore`:

```java
@Override
public List<SituationContext> findActiveBySituationId(String situationId) {
    return store.entrySet().stream()
            .filter(e -> e.getKey().situationId().equals(situationId))
            .map(Map.Entry::getValue)
            .toList();
}
```

- [ ] **Step 5: Implement JpaSituationStore.findActiveBySituationId**

Use `ide_insert_member` on `JpaSituationStore`:

```java
@Override
public List<SituationContext> findActiveBySituationId(String situationId) {
    var entities = em.createQuery(
            "SELECT s FROM SituationEntity s WHERE s.situationId = :sid AND s.triggeredAt IS NULL",
            SituationEntity.class)
        .setParameter("sid", situationId)
        .getResultList();
    return entities.stream().map(mapper::toDomain).toList();
}
```

- [ ] **Step 6: Add deadline metrics to RasMetrics**

Use `ide_insert_member` to add `deadlineChecked(String)` and `deadlineTriggered(String, String)`.

- [ ] **Step 7: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=DeadlineCheckJobTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/ persistence-memory/ persistence-jpa/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): DeadlineCheckJob + findActiveBySituationId implementations

Scheduled job checks active situations for expired deadlines. Iterates
deadline-enabled definitions from registry, queries store by situationId.
InMemory and JPA implementations of findActiveBySituationId.

Refs #41"
```

---

## Batch 5: Replay + Integration Tests

### Task 9: SituationReplayRunner.drainAllDeadlines

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/SituationReplayRunner.java`
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/SituationReplayRunnerTest.java` (or create if absent)

**Interfaces:**
- Consumes: `SituationEvaluator.triggerByDeadline()`, `SituationStore`, `SituationDefinitionRegistry`
- Produces: Deadline drain at end of replay for deterministic deadline evaluation

- [ ] **Step 1: Write test — replay drains deadlines at end**

```java
@Test
void replay_drains_expired_deadlines() {
    // Define a meta-situation with deadline PT5M
    // Feed events that create a context at T=0 but don't satisfy chain mode
    // Replay window ends at T=10M (past deadline)
    // Verify deadline triggers during drain
    var def = new SituationDefinition("meta", Set.of("ras.situation.triggered"),
            null, null, new ChainMode.Count("g", 2), // requires 2, only 1 arrives
            new TriggerAction.NotifyOnly(), null,
            null, null, Map.of(), null, Duration.ofMinutes(5));
    var provider = new TestProvider(List.of(
            new SituationRegistration(def, DefaultCorrelationKeyExtractor.INSTANCE, null, null)));

    Instant t0 = Instant.parse("2026-01-01T00:00:00Z");
    Instant t10m = t0.plus(Duration.ofMinutes(10));
    CloudEvent event = buildBridgedEvent("child", "key", "tenant", "TRIGGERED", t0);

    ReplayResult result = SituationReplayRunner.builder()
            .withProvider(provider)
            .withGanglia(List.of(new SituationWatcherGanglion("g",
                    Map.of(SituationChangeEvent.ChangeType.TRIGGERED, DetectionSignal.DETECTED))))
            .withEvents(List.of(event))
            .withReplayEnd(t10m)
            .build()
            .run();

    assertThat(result.triggersFor("meta")).hasSize(1);
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationReplayRunnerTest`
Expected: FAIL (no deadline drain)

- [ ] **Step 3: Add drainAllDeadlines to SituationReplayRunner**

After `evaluator.drainAllBuffers()` in the `run()` method, add:

```java
drainAllDeadlines(replayEnd != null ? replayEnd : lastEventTime);
```

Implement `drainAllDeadlines(Instant cutoff)`:

```java
private void drainAllDeadlines(Instant cutoff) {
    for (String situationId : registry.allSituationIds()) {
        SituationDefinition def = registry.definition(situationId);
        if (def == null || def.deadline() == null) continue;
        for (SituationContext ctx : collectingSituationStore.findActiveBySituationId(situationId)) {
            if (ctx.firstSignal().plus(def.deadline()).isBefore(cutoff)) {
                evaluator.triggerByDeadline(situationId, ctx.correlationKey(), ctx.tenancyId());
            }
        }
    }
}
```

Add `withReplayEnd(Instant)` to the Builder, and `replayEnd` field.

Implement `findActiveBySituationId` on `CollectingSituationStore` to delegate.

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationReplayRunnerTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): SituationReplayRunner — drainAllDeadlines for deterministic replay

Adds drainAllDeadlines(Instant) called after drainAllBuffers() at end of
replay. Checks all active situations for expired deadlines at the replay
end timestamp. Adds withReplayEnd(Instant) to Builder.

Refs #41"
```

---

### Task 10: Integration tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/ras/runtime/MetaSituationIntegrationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-9
- Produces: End-to-end verification of meta-situation flow

- [ ] **Step 1: Write MetaSituationIntegrationTest — full bridge flow**

```java
@Test
void child_trigger_bridges_and_evaluates_meta_situation() {
    // Setup: child situation "service-health" + meta-situation "system-degradation"
    // Meta subscribes to ras.situation.triggered, filters on situationid
    // Uses Count(3) chain mode
    // Fire 3 child CloudEvents → 3 child triggers → bridge → meta evaluates → meta triggers
    // Assert meta-situation triggered with correct evidence
}

@Test
void deadline_forces_trigger_when_chain_mode_unsatisfied() {
    // Setup: meta-situation with Count(3) and deadline PT5M
    // Fire 1 child CloudEvent → 1 child trigger → meta accumulates (count=1, needs 3)
    // Advance clock past deadline
    // Run DeadlineCheckJob
    // Assert meta-situation triggered by deadline
}

@Test
void nesting_l2_meta_observes_l1_meta() {
    // Setup: child "service-health" + L1 meta "degradation" + L2 meta "escalation"
    // L2 observes ras.situation.triggered, filters on situationid="degradation"
    // Fire enough child events to trigger L1 meta → bridge → L2 meta evaluates
    // Assert L2 meta triggered
}
```

- [ ] **Step 2: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=MetaSituationIntegrationTest`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat(#41): meta-situation integration tests

End-to-end tests: bridge flow, deadline force-trigger, L2 nesting.

Closes #41"
```

---

## References

- [2026-08-26-meta-situations-design.md] — design spec this plan implements
- `RasEngine.java:29` — onCloudEvent routing
- `SituationEvaluator.java:248` — executeDecision trigger mechanics
- `SituationDefinitionRegistry.java:54` — constructor Phase 3
- `CloudEventExpressionContext.java:17` — build() method
- `YamlSituationDefinitionProvider.java:182` — parseGanglia pattern
- `SituationExpiryJob.java:64` — scheduled job pattern
- `SituationReplayRunner.java:131` — drainAllBuffers() call site
- `GanglionDescriptor.java:9` — sealed interface permits
- `SituationStore.java:30` — findActive(tenancyId) pattern
- `ChainMode.java:20` — sealed interface (pure functions)
- `GE-20260730-d54a8f` — CDI fireAsync exception propagation
- GitHub #41 — Meta-situations issue
