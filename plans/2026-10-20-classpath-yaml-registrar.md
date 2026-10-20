# Classpath YAML Registrar + Jackson Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #70 — feat: accept SituationYamlRegistrar from engine — classpath YAML bootstrap for situation definitions
**Issue group:** #70

**Goal:** Move situation YAML bootstrap from engine to ras with full Jackson alignment, eliminating all SnakeYAML usage.

**Architecture:** Add Jackson annotations to ras-api domain types (ChainMode, TriggerAction, GanglionDescriptor, SituationDefinition, FeedbackConfig), create an ExpressionEvaluator deserializer and shared ObjectMapper in runtime, migrate YamlSituationDefinitionProvider from SnakeYAML to Jackson tree API, and add a new SituationYamlRegistrar implementing the platform bootstrap ClasspathYamlRegistrar SPI.

**Tech Stack:** Jackson (jackson-databind, jackson-dataformat-yaml, jackson-datatype-jsr310), Quarkus CDI, casehub-platform-bootstrap

## Global Constraints

- Java 21+, Quarkus 3.x
- No SnakeYAML direct usage — Jackson only
- `ExpressionEvaluator` fields use `{expression, language}` object format in YAML
- All sealed interface polymorphism via `@JsonTypeInfo(property="type")` + `@JsonSubTypes`
- Record deserialization uses canonical constructors — defaults handled in compact constructors
- No `FAIL_ON_UNKNOWN_PROPERTIES=false` — all unknown keys produce errors
- `ACCEPT_CASE_INSENSITIVE_ENUMS` enabled for backward compat with existing YAML

---

## Batch 1: Jackson Annotations on Domain Types (api/)

### Task 1: @JsonProperty on ChainMode.And, ChainMode.Sequence, and TriggerAction.CreateCase

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/ChainMode.java:34` (And record)
- Modify: `api/src/main/java/io/casehub/ras/api/ChainMode.java:65` (Sequence record)
- Modify: `api/src/main/java/io/casehub/ras/api/TriggerAction.java:14` (CreateCase record)
- Test: `api/src/test/java/io/casehub/ras/api/ChainModeJacksonTest.java` (new)

**Interfaces:**
- Consumes: Nothing — first task
- Produces: Jackson-deserializable `ChainMode.And`, `ChainMode.Sequence`, `TriggerAction.CreateCase` from YAML keys `ganglia` (And/Sequence) and flat `caseNamespace`/`caseName`/`caseVersion`/`baseCaseData` (CreateCase)

- [ ] **Step 1: Write failing test for ChainMode.And deserialization**

```java
package io.casehub.ras.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ChainModeJacksonTest {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

    @Test
    void andDeserializesGangliaKey() throws Exception {
        String yaml = """
                type: and
                ganglia: [g1, g2]
                """;
        ChainMode mode = YAML.readValue(yaml, ChainMode.class);
        assertThat(mode).isInstanceOf(ChainMode.And.class);
        assertThat(((ChainMode.And) mode).requiredGanglia()).containsExactlyInAnyOrder("g1", "g2");
    }

    @Test
    void sequenceDeserializesGangliaKey() throws Exception {
        String yaml = """
                type: sequence
                ganglia: [g1, g2, g3]
                """;
        ChainMode mode = YAML.readValue(yaml, ChainMode.class);
        assertThat(mode).isInstanceOf(ChainMode.Sequence.class);
        assertThat(((ChainMode.Sequence) mode).orderedGanglia()).containsExactly("g1", "g2", "g3");
    }
}
```

Add `jackson-dataformat-yaml` as test scope to api `pom.xml`:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=ChainModeJacksonTest -DfailIfNoTests=false`
Expected: FAIL — Jackson looks for `requiredGanglia`/`orderedGanglia` keys, not `ganglia`

- [ ] **Step 3: Add @JsonProperty to ChainMode.And and ChainMode.Sequence**

In `ChainMode.java`, add import and annotation:

```java
import com.fasterxml.jackson.annotation.JsonProperty;
```

Change `And` record (line 34):
```java
record And(@JsonProperty("ganglia") Set<String> requiredGanglia) implements ChainMode {
```

Change `Sequence` record (line 65):
```java
record Sequence(@JsonProperty("ganglia") List<String> orderedGanglia) implements ChainMode {
```

- [ ] **Step 4: Run test to verify And and Sequence pass**

Run: `mvn --batch-mode test -pl api -Dtest=ChainModeJacksonTest -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 5: Write failing test for TriggerAction.CreateCase**

Add to `ChainModeJacksonTest.java` (or create `TriggerActionJacksonTest.java`):

```java
@Test
void createCaseDeserializesFromFlatYaml() throws Exception {
    String yaml = """
            type: create-case
            caseNamespace: test
            caseName: test-case
            caseVersion: "1"
            """;
    TriggerAction action = YAML.readValue(yaml, TriggerAction.class);
    assertThat(action).isInstanceOf(TriggerAction.CreateCase.class);
    var cc = (TriggerAction.CreateCase) action;
    assertThat(cc.config().caseNamespace()).isEqualTo("test");
    assertThat(cc.config().caseName()).isEqualTo("test-case");
    assertThat(cc.config().caseVersion()).isEqualTo("1");
}

@Test
void createCaseDeserializesWithBaseCaseData() throws Exception {
    String yaml = """
            type: create-case
            caseNamespace: ns
            caseName: name
            caseVersion: "2"
            baseCaseData:
              key1: value1
            """;
    TriggerAction action = YAML.readValue(yaml, TriggerAction.class);
    var cc = (TriggerAction.CreateCase) action;
    assertThat(cc.config().baseCaseData()).containsEntry("key1", "value1");
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=ChainModeJacksonTest -DfailIfNoTests=false`
Expected: FAIL — Jackson can't match flat keys to `CreateCase(CaseTriggerConfig config)`

- [ ] **Step 7: Add @JsonCreator factory to TriggerAction.CreateCase**

In `TriggerAction.java`, add imports and factory method:

```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.Map;
```

Replace the `CreateCase` record body:
```java
record CreateCase(CaseTriggerConfig config) implements TriggerAction {
    public CreateCase {
        Objects.requireNonNull(config, "config");
    }

    @JsonCreator
    public static CreateCase fromYaml(
            @JsonProperty("caseNamespace") String caseNamespace,
            @JsonProperty("caseName") String caseName,
            @JsonProperty("caseVersion") String caseVersion,
            @JsonProperty("baseCaseData") Map<String, Object> baseCaseData) {
        return new CreateCase(new CaseTriggerConfig(
                caseNamespace, caseName, caseVersion,
                baseCaseData != null ? baseCaseData : Map.of()));
    }
}
```

- [ ] **Step 8: Run all tests to verify pass**

Run: `mvn --batch-mode test -pl api -Dtest=ChainModeJacksonTest -DfailIfNoTests=false`
Expected: PASS (all And, Sequence, CreateCase tests)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/ras/api/ChainMode.java api/src/main/java/io/casehub/ras/api/TriggerAction.java api/src/test/java/io/casehub/ras/api/ChainModeJacksonTest.java api/pom.xml
git commit -m "feat: add @JsonProperty/@JsonCreator for ChainMode.And, Sequence, TriggerAction.CreateCase

Jackson YAML deserialization requires @JsonProperty when record parameter
names differ from YAML keys (ganglia vs requiredGanglia/orderedGanglia).
CreateCase uses @JsonCreator factory to unwrap flat YAML keys into nested
CaseTriggerConfig.

Refs casehubio/casehub-ras#70

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: @JsonTypeInfo on GanglionDescriptor, @JsonProperty on SituationDefinition, FeedbackConfig defaults

**Files:**
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java:9` (add Jackson annotations)
- Modify: `api/src/main/java/io/casehub/ras/api/SituationDefinition.java:10-12` (add @JsonProperty)
- Modify: `api/src/main/java/io/casehub/ras/api/FeedbackConfig.java:27-68` (defaults in compact constructor)
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java:38-42` (Feature @JsonCreator)
- Modify: `api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java:59-65` (Rule otherwise field)
- Test: `api/src/test/java/io/casehub/ras/api/GanglionDescriptorJacksonTest.java` (new)
- Test: `api/src/test/java/io/casehub/ras/api/SituationDefinitionJacksonTest.java` (new)

**Interfaces:**
- Consumes: Task 1's Jackson-deserializable ChainMode and TriggerAction
- Produces: Jackson-deserializable `GanglionDescriptor` (all 3 variants), `SituationDefinition`, `FeedbackConfig` with optional-field defaults. `NaiveBayes.Feature.fromYaml()` factory. `ExpressionRules.Rule` with `boolean otherwise` field.

- [ ] **Step 1: Write failing test for GanglionDescriptor polymorphic deserialization**

```java
package io.casehub.ras.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class GanglionDescriptorJacksonTest {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

    @Test
    void expressionRulesDeserializes() throws Exception {
        String yaml = """
                type: expression-rules
                ganglionId: test-rules
                handledEventTypes: [event.a]
                rules:
                  - when:
                      expression: ".data.level == \\"HIGH\\""
                      language: jq
                    signal: DETECTED
                    confidence: 0.9
                  - otherwise: true
                    signal: NOISE
                    confidence: 0.1
                """;
        GanglionDescriptor desc = YAML.readValue(yaml, GanglionDescriptor.class);
        assertThat(desc).isInstanceOf(GanglionDescriptor.ExpressionRules.class);
        var er = (GanglionDescriptor.ExpressionRules) desc;
        assertThat(er.ganglionId()).isEqualTo("test-rules");
        assertThat(er.rules()).hasSize(2);
        assertThat(er.rules().get(0).when()).isNotNull();
        assertThat(er.rules().get(0).otherwise()).isFalse();
        assertThat(er.rules().get(1).when()).isNull();
        assertThat(er.rules().get(1).otherwise()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=GanglionDescriptorJacksonTest -DfailIfNoTests=false`
Expected: FAIL — `GanglionDescriptor` has no `@JsonTypeInfo`

- [ ] **Step 3: Add @JsonTypeInfo and @JsonSubTypes to GanglionDescriptor**

In `GanglionDescriptor.java`, add imports and annotations:

```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
        @JsonSubTypes.Type(value = GanglionDescriptor.NaiveBayes.class, name = "naive-bayes"),
        @JsonSubTypes.Type(value = GanglionDescriptor.ExpressionRules.class, name = "expression-rules"),
        @JsonSubTypes.Type(value = GanglionDescriptor.SituationWatcher.class, name = "situation-watcher")
})
public sealed interface GanglionDescriptor {
```

- [ ] **Step 4: Add `boolean otherwise` to ExpressionRules.Rule**

Change the `Rule` record in `GanglionDescriptor.java`:

```java
public record Rule(
        ExpressionEvaluator when,
        boolean otherwise,
        DetectionSignal signal,
        double confidence,
        ExpressionEvaluator confidenceExpression,
        Map<String, ExpressionEvaluator> evidenceTemplates
) {
    public Rule {
        if (when != null && otherwise) {
            throw new IllegalArgumentException(
                    "Rule cannot have both 'when' and 'otherwise'");
        }
        if (when == null && !otherwise) {
            throw new IllegalArgumentException(
                    "Rule must have either 'when' or 'otherwise'");
        }
    }
}
```

- [ ] **Step 5: Fix all callers of the Rule constructor**

The Rule record now has 6 parameters (added `boolean otherwise`). All existing callers pass 5 args. Find and fix:

In `YamlSituationDefinitionProvider.java` (line ~317):
```java
rules.add(new GanglionDescriptor.ExpressionRules.Rule(when, !hasWhen, signal, confidence, confidenceExpr, parseEvidenceTemplates(ruleMap)));
```

Note: `!hasWhen` maps to `otherwise` — when `hasWhen` is false, `otherwise` is true.

Search for other callers with `ide_find_references` on the Rule constructor and fix each one.

- [ ] **Step 6: Add @JsonCreator to NaiveBayes.Feature**

In `GanglionDescriptor.java`, add the factory method to the `Feature` record:

```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.casehub.platform.api.expression.JQExpressionEvaluator;
import io.casehub.platform.api.expression.MvelExpressionEvaluator;
```

```java
public record Feature(
        ExpressionEvaluator expression,
        List<String> values,
        double[][] likelihoods
) {
    @JsonCreator
    public static Feature fromYaml(
            @JsonProperty("expression") String expressionText,
            @JsonProperty("language") String language,
            @JsonProperty("values") List<String> values,
            @JsonProperty("likelihoods") double[][] likelihoods) {
        Objects.requireNonNull(expressionText, "Feature missing required 'expression' field");
        Objects.requireNonNull(language, "Feature missing required 'language' field");
        ExpressionEvaluator evaluator = switch (language) {
            case "jq" -> new JQExpressionEvaluator(expressionText);
            case "mvel" -> new MvelExpressionEvaluator(expressionText);
            default -> throw new IllegalArgumentException(
                    "Unknown expression language: " + language);
        };
        return new Feature(evaluator, values, likelihoods);
    }
}
```

- [ ] **Step 7: Run GanglionDescriptor tests**

Run: `mvn --batch-mode test -pl api -Dtest=GanglionDescriptorJacksonTest -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 8: Add @JsonProperty to SituationDefinition**

In `SituationDefinition.java`, add import and annotations on the record parameters:

```java
import com.fasterxml.jackson.annotation.JsonProperty;
```

Change the canonical constructor parameters:
```java
public record SituationDefinition(
        String situationId,
        Set<String> eventTypes,
        Duration correlationWindow,
        Duration eventBufferDelay,
        ChainMode chainMode,
        TriggerAction triggerAction,
        TriggerMode triggerMode,
        @JsonProperty("correlationKey") ExpressionEvaluator correlationKeyExpression,
        ExpressionEvaluator eventFilter,
        Map<String, ExpressionEvaluator> dynamicCaseData,
        @JsonProperty("feedback") FeedbackConfig feedbackConfig,
        Duration deadline
) {
```

- [ ] **Step 9: Update FeedbackConfig compact constructor for Jackson defaults**

In `FeedbackConfig.java`, update the compact constructor to null-coalesce optional fields BEFORE validation:

```java
public FeedbackConfig {
    crossRefWindow = crossRefWindow != null ? crossRefWindow : Duration.ofHours(1);
    overSensitiveThreshold = overSensitiveThreshold == 0.0 ? 0.5 : overSensitiveThreshold;
    underSensitiveThreshold = underSensitiveThreshold == 0.0 ? 0.5 : underSensitiveThreshold;

    Objects.requireNonNull(noiseLabels, "noiseLabels");
    Objects.requireNonNull(confirmedLabels, "confirmedLabels");
    Objects.requireNonNull(suppressionCooldown, "suppressionCooldown");
    Objects.requireNonNull(retentionPeriod, "retentionPeriod");
    Objects.requireNonNull(crossRefWindow, "crossRefWindow");
    noiseLabels = Set.copyOf(noiseLabels);
    confirmedLabels = Set.copyOf(confirmedLabels);
    if (!Collections.disjoint(noiseLabels, confirmedLabels)) {
        throw new IllegalArgumentException(
                "noiseLabels and confirmedLabels must be disjoint");
    }
    if (suppressionCooldown.isZero() || suppressionCooldown.isNegative()) {
        throw new IllegalArgumentException(
                "suppressionCooldown must be positive, got: " + suppressionCooldown);
    }
    if (learningRate <= 0.0 || learningRate > 1.0) {
        throw new IllegalArgumentException(
                "learningRate must be in (0.0, 1.0], got: " + learningRate);
    }
    if (retentionPeriod.isZero() || retentionPeriod.isNegative()) {
        throw new IllegalArgumentException(
                "retentionPeriod must be positive, got: " + retentionPeriod);
    }
    if (retentionPeriod.compareTo(suppressionCooldown) < 0) {
        throw new IllegalArgumentException(
                "retentionPeriod must be >= suppressionCooldown: "
                        + retentionPeriod + " < " + suppressionCooldown);
    }
    if (overSensitiveThreshold <= 0.0 || overSensitiveThreshold > 1.0) {
        throw new IllegalArgumentException(
                "overSensitiveThreshold must be in (0.0, 1.0], got: " + overSensitiveThreshold);
    }
    if (underSensitiveThreshold <= 0.0 || underSensitiveThreshold > 1.0) {
        throw new IllegalArgumentException(
                "underSensitiveThreshold must be in (0.0, 1.0], got: " + underSensitiveThreshold);
    }
    if (crossRefWindow.isZero() || crossRefWindow.isNegative()) {
        throw new IllegalArgumentException(
                "crossRefWindow must be positive, got: " + crossRefWindow);
    }
}
```

- [ ] **Step 10: Write SituationDefinition Jackson test**

```java
package io.casehub.ras.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SituationDefinitionJacksonTest {

    private static final ObjectMapper YAML;
    static {
        YAML = new ObjectMapper(new YAMLFactory());
        YAML.registerModule(new JavaTimeModule());
    }

    @Test
    void minimalSituationDeserializes() throws Exception {
        String yaml = """
                situationId: test-sit
                eventTypes: [event.a]
                chainMode:
                  type: or
                  ganglia: [g1]
                triggerAction:
                  type: create-case
                  caseNamespace: ns
                  caseName: name
                  caseVersion: "1"
                """;
        SituationDefinition def = YAML.readValue(yaml, SituationDefinition.class);
        assertThat(def.situationId()).isEqualTo("test-sit");
        assertThat(def.chainMode()).isInstanceOf(ChainMode.Or.class);
        assertThat(def.triggerAction()).isInstanceOf(TriggerAction.CreateCase.class);
    }

    @Test
    void feedbackFieldMapsToFeedbackConfig() throws Exception {
        String yaml = """
                situationId: test-sit
                eventTypes: [event.a]
                chainMode:
                  type: or
                  ganglia: [g1]
                triggerAction:
                  type: notify-only
                feedback:
                  noiseLabels: [false-alarm]
                  confirmedLabels: [real-issue]
                  suppressionCooldown: PT1H
                  learningRate: 0.5
                  retentionPeriod: P7D
                """;
        SituationDefinition def = YAML.readValue(yaml, SituationDefinition.class);
        assertThat(def.feedbackConfig()).isNotNull();
        assertThat(def.feedbackConfig().noiseLabels()).containsExactly("false-alarm");
        assertThat(def.feedbackConfig().crossRefWindow()).hasHours(1);
    }
}
```

- [ ] **Step 11: Run all api tests**

Run: `mvn --batch-mode test -pl api`
Expected: PASS (all existing + new tests)

- [ ] **Step 12: Commit**

```bash
git add api/
git commit -m "feat: Jackson annotations on GanglionDescriptor, SituationDefinition, FeedbackConfig

- GanglionDescriptor: @JsonTypeInfo + @JsonSubTypes for naive-bayes,
  expression-rules, situation-watcher
- ExpressionRules.Rule: add boolean otherwise field with mutual-exclusion
  validation in compact constructor
- NaiveBayes.Feature: @JsonCreator factory for flat expression/language keys
- SituationDefinition: @JsonProperty for correlationKey→correlationKeyExpression,
  feedback→feedbackConfig
- FeedbackConfig: null-coalesce optional fields before validation

Refs casehubio/casehub-ras#70

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: ExpressionEvaluator Deserializer + RasYamlMapper (runtime/)

### Task 3: ExpressionEvaluatorDeserializer and RasYamlMapper

**Files:**
- Create: `runtime/src/main/java/io/casehub/ras/runtime/ExpressionEvaluatorDeserializer.java`
- Create: `runtime/src/main/java/io/casehub/ras/runtime/RasYamlMapper.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/ExpressionEvaluatorDeserializerTest.java` (new)

**Interfaces:**
- Consumes: Task 1-2's Jackson-annotated domain types
- Produces: `ExpressionEvaluatorDeserializer` class (package-private). `RasYamlMapper.instance()` returning a configured `ObjectMapper` with YAMLFactory, JavaTimeModule, ExpressionEvaluator deserializer, and `ACCEPT_CASE_INSENSITIVE_ENUMS`.

- [ ] **Step 1: Write failing test for ExpressionEvaluator deserialization**

```java
package io.casehub.ras.runtime;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import io.casehub.platform.api.expression.JQExpressionEvaluator;
import io.casehub.platform.api.expression.MvelExpressionEvaluator;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ExpressionEvaluatorDeserializerTest {

    private static final ObjectMapper YAML;
    static {
        YAML = new ObjectMapper(new YAMLFactory());
        var module = new SimpleModule();
        module.addDeserializer(ExpressionEvaluator.class, new ExpressionEvaluatorDeserializer());
        YAML.registerModule(module);
    }

    record Wrapper(ExpressionEvaluator expr) {}

    @Test
    void deserializesJqExpression() throws Exception {
        String yaml = """
                expr:
                  expression: ".data.value"
                  language: jq
                """;
        Wrapper w = YAML.readValue(yaml, Wrapper.class);
        assertThat(w.expr()).isInstanceOf(JQExpressionEvaluator.class);
        assertThat(w.expr().expression()).isEqualTo(".data.value");
    }

    @Test
    void deserializesMvelExpression() throws Exception {
        String yaml = """
                expr:
                  expression: "value > 10"
                  language: mvel
                """;
        Wrapper w = YAML.readValue(yaml, Wrapper.class);
        assertThat(w.expr()).isInstanceOf(MvelExpressionEvaluator.class);
        assertThat(w.expr().expression()).isEqualTo("value > 10");
    }

    @Test
    void bareStringDefaultsToJq() throws Exception {
        String yaml = """
                expr: ".data.severity"
                """;
        Wrapper w = YAML.readValue(yaml, Wrapper.class);
        assertThat(w.expr()).isInstanceOf(JQExpressionEvaluator.class);
        assertThat(w.expr().expression()).isEqualTo(".data.severity");
    }

    @Test
    void nullReturnsNull() throws Exception {
        String yaml = """
                expr: null
                """;
        Wrapper w = YAML.readValue(yaml, Wrapper.class);
        assertThat(w.expr()).isNull();
    }

    @Test
    void unknownLanguageThrows() {
        String yaml = """
                expr:
                  expression: "x"
                  language: groovy
                """;
        assertThatThrownBy(() -> YAML.readValue(yaml, Wrapper.class))
                .hasMessageContaining("Unknown expression language");
    }

    @Test
    void missingExpressionFieldThrows() {
        String yaml = """
                expr:
                  language: jq
                """;
        assertThatThrownBy(() -> YAML.readValue(yaml, Wrapper.class))
                .hasMessageContaining("expression");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=ExpressionEvaluatorDeserializerTest -DfailIfNoTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement ExpressionEvaluatorDeserializer**

```java
package io.casehub.ras.runtime;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.deser.std.StdDeserializer;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import io.casehub.platform.api.expression.JQExpressionEvaluator;
import io.casehub.platform.api.expression.MvelExpressionEvaluator;

import java.io.IOException;

class ExpressionEvaluatorDeserializer extends StdDeserializer<ExpressionEvaluator> {

    ExpressionEvaluatorDeserializer() {
        super(ExpressionEvaluator.class);
    }

    @Override
    public ExpressionEvaluator deserialize(JsonParser p, DeserializationContext ctxt)
            throws IOException {
        JsonNode node = p.readValueAsTree();
        if (node == null || node.isNull()) {
            return null;
        }
        if (node.isTextual()) {
            return new JQExpressionEvaluator(node.textValue());
        }
        if (node.isObject()) {
            JsonNode exprNode = node.get("expression");
            if (exprNode == null || !exprNode.isTextual()) {
                throw ctxt.weirdStringException(
                        node.toString(), ExpressionEvaluator.class,
                        "Expression object missing required 'expression' field");
            }
            JsonNode langNode = node.get("language");
            if (langNode == null || !langNode.isTextual()) {
                throw ctxt.weirdStringException(
                        node.toString(), ExpressionEvaluator.class,
                        "Expression object missing required 'language' field");
            }
            String expression = exprNode.textValue();
            String language = langNode.textValue();
            return switch (language) {
                case "jq" -> new JQExpressionEvaluator(expression);
                case "mvel" -> new MvelExpressionEvaluator(expression);
                default -> throw ctxt.weirdStringException(
                        language, ExpressionEvaluator.class,
                        "Unknown expression language '" + language + "'. Expected: jq, mvel");
            };
        }
        throw ctxt.weirdStringException(
                node.toString(), ExpressionEvaluator.class,
                "Expected expression as string or {expression, language} object, got " + node.getNodeType());
    }

    @Override
    public ExpressionEvaluator getNullValue(DeserializationContext ctxt) {
        return null;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=ExpressionEvaluatorDeserializerTest -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 5: Implement RasYamlMapper**

```java
package io.casehub.ras.runtime;

import com.fasterxml.jackson.databind.MapperFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.platform.api.expression.ExpressionEvaluator;

class RasYamlMapper {

    private static final ObjectMapper INSTANCE = build();

    static ObjectMapper instance() {
        return INSTANCE;
    }

    private static ObjectMapper build() {
        var mapper = new ObjectMapper(new YAMLFactory());
        mapper.registerModule(new JavaTimeModule());
        var module = new SimpleModule();
        module.addDeserializer(ExpressionEvaluator.class, new ExpressionEvaluatorDeserializer());
        mapper.registerModule(module);
        mapper.enable(MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS);
        return mapper;
    }
}
```

- [ ] **Step 6: Write integration test using RasYamlMapper with full SituationDefinition**

Add to `ExpressionEvaluatorDeserializerTest.java`:

```java
@Test
void rasYamlMapperDeserializesFullSituation() throws Exception {
    String yaml = """
            situationId: integration-test
            eventTypes: [event.a, event.b]
            correlationWindow: PT1H
            chainMode:
              type: threshold
              ganglia: [g1, g2]
              minConfidence: 0.5
            triggerAction:
              type: create-case
              caseNamespace: test
              caseName: case
              caseVersion: "1"
            correlationKey:
              expression: ".subject"
              language: jq
            """;
    var def = RasYamlMapper.instance().readValue(yaml,
            io.casehub.ras.api.SituationDefinition.class);
    assertThat(def.situationId()).isEqualTo("integration-test");
    assertThat(def.correlationKeyExpression()).isNotNull();
    assertThat(def.correlationKeyExpression().expression()).isEqualTo(".subject");
}
```

- [ ] **Step 7: Run all runtime tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=ExpressionEvaluatorDeserializerTest -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ras/runtime/ExpressionEvaluatorDeserializer.java runtime/src/main/java/io/casehub/ras/runtime/RasYamlMapper.java runtime/src/test/java/io/casehub/ras/runtime/ExpressionEvaluatorDeserializerTest.java
git commit -m "feat: ExpressionEvaluatorDeserializer + RasYamlMapper for Jackson YAML parsing

Custom Jackson deserializer handles {expression, language} format and
bare string (defaults to JQ). RasYamlMapper singleton configures
YAMLFactory + JavaTimeModule + case-insensitive enums.

Refs casehubio/casehub-ras#70

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 3: Migrate YamlSituationDefinitionProvider to Jackson (runtime/)

### Task 4: Replace SnakeYAML with Jackson in YamlSituationDefinitionProvider

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java` (major rewrite)
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java` (existing — must still pass)

**Interfaces:**
- Consumes: Task 3's `RasYamlMapper.instance()`, Task 1-2's Jackson-annotated domain types
- Produces: Same `SituationDefinitionProvider` contract — `registrations()` and `ganglionDescriptors()` return identical results. No public API changes.

- [ ] **Step 1: Run existing tests as baseline**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: PASS (baseline — all existing tests pass with SnakeYAML)

- [ ] **Step 2: Replace SnakeYAML import and parseAll entry point**

In `YamlSituationDefinitionProvider.java`:

Remove:
```java
import org.yaml.snakeyaml.Yaml;
```

Add:
```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.databind.node.TextNode;
```

Change the `parseAll` methods to use Jackson tree API:

```java
private static final ObjectMapper YAML_MAPPER = RasYamlMapper.instance();

@SuppressWarnings("unchecked")
private static ParseResult parseAll(InputStream yaml,
                                    Map<String, SituationTemplate> builtInTemplates) {
    JsonNode root;
    try {
        root = YAML_MAPPER.readTree(yaml);
    } catch (java.io.IOException e) {
        throw new java.io.UncheckedIOException("Failed to parse YAML", e);
    }
    if (root == null || root.isNull() || root.isEmpty()) {
        return new ParseResult(List.of(), List.of());
    }
    Map<String, SituationTemplate> templates = new LinkedHashMap<>(builtInTemplates);
    templates.putAll(parseTemplates(root));
    List<GanglionDescriptor> ganglia = parseGanglia(root);
    SituationParseResult sitResult = parseSituations(root, templates);
    List<GanglionDescriptor> allGanglia = new ArrayList<>(ganglia);
    allGanglia.addAll(sitResult.bundledGanglia());
    return new ParseResult(sitResult.registrations(), List.copyOf(allGanglia));
}
```

- [ ] **Step 3: Replace parseSituations with Jackson tree + treeToValue**

```java
private static SituationParseResult parseSituations(JsonNode root,
                                                     Map<String, SituationTemplate> templates) {
    JsonNode situationsNode = root.get("situations");
    if (situationsNode == null || !situationsNode.isArray()) {
        return new SituationParseResult(List.of(), List.of());
    }
    List<SituationRegistration> result = new ArrayList<>();
    List<GanglionDescriptor> bundledGanglia = new ArrayList<>();
    for (JsonNode sit : situationsNode) {
        if (sit.has("fromTemplate")) {
            String templateId = sit.get("fromTemplate").asText();
            String sitId = sit.has("situationId") ? sit.get("situationId").asText() : "<missing>";
            JsonNode resolved;
            try {
                resolved = resolveTemplate(sit, templates);
            } catch (Exception e) {
                throw new IllegalArgumentException(
                        "Error resolving template '" + templateId
                        + "' for situation '" + sitId + "': " + e.getMessage(), e);
            }
            SituationTemplate template = templates.get(templateId);
            if (template.ganglia() != null && !template.ganglia().isEmpty()) {
                Map<String, Object> resolvedParams = buildResolvedParams(sit, template);
                ArrayNode resolvedGangliaArray = YAML_MAPPER.createArrayNode();
                for (JsonNode g : template.ganglia()) {
                    resolvedGangliaArray.add(substituteParams(g.deepCopy(), resolvedParams));
                }
                checkUnresolved(resolvedGangliaArray, templateId);
                ObjectNode gangliaRoot = YAML_MAPPER.createObjectNode();
                gangliaRoot.set("ganglia", resolvedGangliaArray);
                bundledGanglia.addAll(parseGanglia(gangliaRoot));
            }
            try {
                result.add(parseSituation(resolved));
            } catch (Exception e) {
                throw new IllegalArgumentException(
                        "Error parsing resolved template '" + templateId
                        + "' for situation '" + sitId + "': " + e.getMessage(), e);
            }
        } else {
            result.add(parseSituation(sit));
        }
    }
    return new SituationParseResult(List.copyOf(result), List.copyOf(bundledGanglia));
}
```

- [ ] **Step 4: Replace parseSituation and parseGanglia with treeToValue**

```java
private static SituationRegistration parseSituation(JsonNode node) {
    try {
        SituationDefinition def = YAML_MAPPER.treeToValue(node, SituationDefinition.class);
        return new SituationRegistration(def);
    } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
        String sitId = node.has("situationId") ? node.get("situationId").asText() : "<unknown>";
        throw new IllegalArgumentException(
                "Failed to deserialize situation '" + sitId + "': " + e.getMessage(), e);
    }
}

private static List<GanglionDescriptor> parseGanglia(JsonNode root) {
    JsonNode gangliaNode = root.get("ganglia");
    if (gangliaNode == null || !gangliaNode.isArray()) {
        return List.of();
    }
    List<GanglionDescriptor> result = new ArrayList<>(gangliaNode.size());
    for (JsonNode g : gangliaNode) {
        try {
            result.add(YAML_MAPPER.treeToValue(g, GanglionDescriptor.class));
        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
            String gId = g.has("ganglionId") ? g.get("ganglionId").asText() : "<unknown>";
            throw new IllegalArgumentException(
                    "Failed to deserialize ganglion '" + gId + "': " + e.getMessage(), e);
        }
    }
    return List.copyOf(result);
}
```

- [ ] **Step 5: Migrate template system to JsonNode operations**

Replace `substituteParams`, `checkUnresolved`, `deepMerge`, `resolveTemplate`, `buildResolvedParams`, `parseTemplates`, `parseParameterDefs` — all must operate on `JsonNode` instead of `Map<String, Object>`.

The `SituationTemplate` record changes:
```java
record SituationTemplate(
        String id,
        String description,
        Map<String, ParameterDef> parameters,
        JsonNode definition,
        ArrayNode ganglia
) {}
```

This is the most substantial code change — rewrite each method to traverse `JsonNode` trees using Jackson's `ObjectNode`/`ArrayNode`/`TextNode` APIs. The algorithm is the same, just operating on a different tree representation.

Key mappings:
- `map.get(key)` → `node.get(key)`
- `map.put(key, value)` → `((ObjectNode)node).set(key, valueNode)`
- `map.containsKey(key)` → `node.has(key)`
- `list.get(i)` → `arrayNode.get(i)`
- `value instanceof String s` → `node.isTextual()` + `node.textValue()`
- `value instanceof Map` → `node.isObject()`
- `value instanceof List` → `node.isArray()`
- Deep copy: `node.deepCopy()`

- [ ] **Step 6: Delete all eliminated manual parse methods**

Remove the following methods entirely:
- `parseExpressionEntry()`
- `parseExpressionEvaluator()`
- `parseEvidenceTemplates()`
- `parseOutcomeEvidenceTemplates()`
- `parseDynamicCaseData()`
- `parseFeedbackConfig()`
- `parseTriggerMode()`
- `parseTriggerAction()`
- `parseChainMode()`
- `parseNaiveBayesGanglion()`
- `parseExpressionRulesGanglion()`
- `parseSituationWatcherGanglion()`
- `parseNaiveBayesFeature()`
- `parseSignalMapping()`
- `requireString(Map, String)`
- `requireList()`
- `requireNumber()`

Keep `parseTemplates()`, `substituteParams()`, `checkUnresolved()`, `deepMerge()`, `resolveTemplate()`, `buildResolvedParams()`, `parseParameterDefs()` — but rewritten for `JsonNode`.

- [ ] **Step 7: Add post-deserialization validation**

After `treeToValue()` calls for `GanglionDescriptor`, add business rule validation:

```java
private static void validateGanglionDescriptor(GanglionDescriptor desc) {
    switch (desc) {
        case GanglionDescriptor.NaiveBayes nb -> {
            if (nb.outcomes() == null || nb.outcomes().size() < 2) {
                throw new IllegalArgumentException(
                        "outcomes must have at least 2 entries for ganglion '" + nb.ganglionId() + "'");
            }
            if (nb.handledEventTypes() == null || nb.handledEventTypes().isEmpty()) {
                throw new IllegalArgumentException(
                        "handledEventTypes must not be empty for ganglion '" + nb.ganglionId() + "'");
            }
        }
        case GanglionDescriptor.ExpressionRules er -> {
            if (er.rules() == null || er.rules().isEmpty()) {
                throw new IllegalArgumentException(
                        "rules must not be empty for ganglion '" + er.ganglionId() + "'");
            }
            for (int i = 0; i < er.rules().size(); i++) {
                var rule = er.rules().get(i);
                if (rule.otherwise() && i != er.rules().size() - 1) {
                    throw new IllegalArgumentException(
                            "'otherwise' must be the last rule in ganglion '" + er.ganglionId() + "'");
                }
                if (rule.confidence() < 0.0 || rule.confidence() > 1.0) {
                    throw new IllegalArgumentException(
                            "confidence must be 0.0-1.0 in rule " + i
                            + " of ganglion '" + er.ganglionId() + "', got: " + rule.confidence());
                }
            }
        }
        case GanglionDescriptor.SituationWatcher sw -> {
            if (sw.changeTypeMapping() == null || sw.changeTypeMapping().isEmpty()) {
                throw new IllegalArgumentException(
                        "changeTypeMapping must not be empty for ganglion '" + sw.ganglionId() + "'");
            }
        }
    }
}
```

Call after each `treeToValue(g, GanglionDescriptor.class)`.

- [ ] **Step 8: Update loadBuiltInTemplates to use Jackson**

```java
private static Map<String, SituationTemplate> loadBuiltInTemplates() {
    InputStream is = Thread.currentThread().getContextClassLoader()
                           .getResourceAsStream("META-INF/ras-situation-templates.yaml");
    if (is == null) { return Map.of(); }
    try (is) {
        JsonNode root = YAML_MAPPER.readTree(is);
        return root != null ? parseTemplates(root) : Map.of();
    } catch (IOException e) {
        throw new UncheckedIOException("Failed to read built-in templates", e);
    }
}
```

- [ ] **Step 9: Run all existing tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: PASS — all existing tests pass with Jackson backend

If tests fail, debug by comparing the Jackson deserialization output with what the manual parser produced. Common issues: enum case sensitivity (should be handled by `ACCEPT_CASE_INSENSITIVE_ENUMS`), Duration format, missing `@JsonProperty` annotations.

- [ ] **Step 10: Run full runtime test suite**

Run: `mvn --batch-mode test -pl runtime`
Expected: PASS — no regressions

- [ ] **Step 11: Commit**

```bash
git add runtime/
git commit -m "feat: migrate YamlSituationDefinitionProvider from SnakeYAML to Jackson

Replace manual Map<String, Object> parsing with Jackson treeToValue()
for situations and ganglia. Template resolution migrated to JsonNode
tree operations. ~500 lines of manual parsing eliminated. Existing
tests pass unchanged.

Refs casehubio/casehub-ras#70

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 4: SituationYamlRegistrar (runtime/)

### Task 5: SituationYamlRegistrar implementing ClasspathYamlRegistrar

**Files:**
- Modify: `runtime/pom.xml` (add casehub-platform-bootstrap dependency)
- Create: `runtime/src/main/java/io/casehub/ras/runtime/SituationYamlRegistrar.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/SituationYamlRegistrarTest.java` (new)

**Interfaces:**
- Consumes: Task 3's `RasYamlMapper.instance()`, `SituationRegistrar` (from api/)
- Produces: `SituationYamlRegistrar` CDI bean implementing `ClasspathYamlRegistrar`. Registered automatically by `ClasspathBootstrap` at startup.

- [ ] **Step 1: Add casehub-platform-bootstrap dependency**

In `runtime/pom.xml`, add after existing dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-bootstrap</artifactId>
</dependency>
```

- [ ] **Step 2: Write failing test for SituationYamlRegistrar**

```java
package io.casehub.ras.runtime;

import io.casehub.ras.api.SituationRegistrar;
import io.casehub.ras.api.SituationRegistration;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class SituationYamlRegistrarTest {

    @Test
    void registersParsedSituationDefinition() {
        String yaml = """
                situationId: bootstrap-test
                eventTypes: [event.type.a]
                chainMode:
                  type: or
                  ganglia: [g1]
                triggerAction:
                  type: notify-only
                """;
        var captured = new ArrayList<SituationRegistration>();
        var registrar = new SituationYamlRegistrar();
        registrar.situationRegistrar = new CapturingSituationRegistrar(captured);

        registrar.register("situations/test.yaml",
                new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0).definition().situationId()).isEqualTo("bootstrap-test");
    }

    @Test
    void badYamlLogsErrorDoesNotThrow() {
        String yaml = "not: valid: yaml: [";
        var registrar = new SituationYamlRegistrar();
        registrar.situationRegistrar = new CapturingSituationRegistrar(new ArrayList<>());

        registrar.register("situations/bad.yaml",
                new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
    }

    @Test
    void sectionKeyIsSituations() {
        var registrar = new SituationYamlRegistrar();
        assertThat(registrar.sectionKey()).isEqualTo("situations");
    }

    @Test
    void classpathPathsContainsSituations() {
        var registrar = new SituationYamlRegistrar();
        assertThat(registrar.classpathPaths()).containsExactly("situations");
    }

    @Test
    void priorityIs300() {
        var registrar = new SituationYamlRegistrar();
        assertThat(registrar.priority()).isEqualTo(300);
    }

    private record CapturingSituationRegistrar(
            List<SituationRegistration> captured
    ) implements SituationRegistrar {
        @Override
        public void register(SituationRegistration registration) {
            captured.add(registration);
        }

        @Override
        public void deregister(String situationId) {}

        @Override
        public boolean exists(String situationId) {
            return captured.stream().anyMatch(
                    r -> r.definition().situationId().equals(situationId));
        }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationYamlRegistrarTest -DfailIfNoTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement SituationYamlRegistrar**

```java
package io.casehub.ras.runtime;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.platform.bootstrap.spi.ClasspathYamlRegistrar;
import io.casehub.ras.api.SituationDefinition;
import io.casehub.ras.api.SituationRegistrar;
import io.casehub.ras.api.SituationRegistration;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.util.Set;

@ApplicationScoped
public class SituationYamlRegistrar implements ClasspathYamlRegistrar {

    private static final Logger LOG = Logger.getLogger(SituationYamlRegistrar.class);
    private static final ObjectMapper YAML_MAPPER = RasYamlMapper.instance();

    @Inject
    SituationRegistrar situationRegistrar;

    @Override
    public String sectionKey() {
        return "situations";
    }

    @Override
    public Set<String> classpathPaths() {
        return Set.of("situations");
    }

    @Override
    public int priority() {
        return 300;
    }

    @Override
    public void register(String resourcePath, InputStream content) {
        try {
            SituationDefinition definition = YAML_MAPPER.readValue(content, SituationDefinition.class);
            situationRegistrar.register(new SituationRegistration(definition));
            LOG.debugf("Registered situation definition from %s: %s",
                    resourcePath, definition.situationId());
        } catch (IOException e) {
            LOG.errorf(e, "Failed to load situation definition from %s", resourcePath);
        } catch (Exception e) {
            LOG.errorf(e, "Failed to register situation definition from %s", resourcePath);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=SituationYamlRegistrarTest -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and tests pass

- [ ] **Step 7: Commit**

```bash
git add runtime/pom.xml runtime/src/main/java/io/casehub/ras/runtime/SituationYamlRegistrar.java runtime/src/test/java/io/casehub/ras/runtime/SituationYamlRegistrarTest.java
git commit -m "feat: SituationYamlRegistrar — classpath YAML bootstrap for situation definitions

Implements ClasspathYamlRegistrar from casehub-platform-bootstrap.
Scans situations/ classpath directories, deserializes individual YAML
files into SituationDefinition via Jackson, registers via SituationRegistrar.
Error handling: catch-and-log per file, never aborts the scan.

Refs casehubio/casehub-ras#70

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-10-20-classpath-yaml-registrar-design.md] — design spec this plan implements
- `engine/runtime/.../bootstrap/SituationYamlRegistrar.java` — engine's current registrar (to be removed)
- `ras/api/.../ChainMode.java:34,65` — And/Sequence records needing @JsonProperty
- `ras/api/.../TriggerAction.java:14` — CreateCase record needing @JsonCreator
- `ras/api/.../GanglionDescriptor.java:9` — sealed interface needing @JsonTypeInfo
- `ras/api/.../SituationDefinition.java:10` — record needing @JsonProperty
- `ras/api/.../FeedbackConfig.java:27` — compact constructor needing defaults
- `ras/runtime/.../YamlSituationDefinitionProvider.java` — SnakeYAML→Jackson migration
- `platform/platform-api/.../ExpressionEvaluator.java` — platform expression interface
- `parent/casehub-platform-bootstrap/.../ClasspathYamlRegistrar.java` — bootstrap SPI
- GitHub #70 — focal issue
