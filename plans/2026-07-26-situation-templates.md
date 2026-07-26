# Situation Templates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #42 — Situation templates — reusable parameterised situation definitions
**Issue group:** #42

**Goal:** Add YAML template support to `YamlSituationDefinitionProvider` so consumers
can define reusable situation patterns with typed parameters and instantiate them
with `fromTemplate:`.

**Architecture:** Templates are a YAML parse concern — internal to
`YamlSituationDefinitionProvider`. No API changes. The registry and all consumers see
fully resolved `SituationRegistration` objects. Internal records (`SituationTemplate`,
`ParameterDef`) are package-private for test access. Static utility methods handle
parameter substitution (recursive tree walk) and deep merge.

**Tech Stack:** Java 21, SnakeYAML, JUnit 5, AssertJ

## Global Constraints

- No changes to `api/` module — templates are a parse concern in `runtime/`
- Template-instantiated situations pass through the existing `parseSituation()` code path
- All new types are package-private (not public API)
- SnakeYAML handles typing — no explicit type declarations on parameters
- `situationId` and `eventTypes` are always consumer-provided identity fields

---

### Task 1: Template parsing, parameter substitution, and resolution

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: Existing `parseSituation(Map)`, `parseGanglia(Map)`, `requireString(Map, String)`
- Produces: `parseTemplates(Map) → Map<String, SituationTemplate>`, `resolveTemplate(Map, Map) → Map`, `substituteParams(Object, Map) → Object`, `checkUnresolved(Object, String) → void`, `deepMerge(Map, Map) → Map`

- [ ] **Step 1: Write failing test — basic template instantiation**

Add to `YamlSituationDefinitionProviderTest.java`:

```java
@Test
void templateInstantiationProducesCorrectDefinition() {
    var regs = provider("""
            templates:
              - id: streak-breach
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
                  triggerMode:
                    type: fire-once

            situations:
              - fromTemplate: streak-breach
                situationId: test-sit
                eventTypes: [test.event]
                parameters:
                  ganglionId: my-ganglion
                  caseNamespace: test-ns
                  caseName: my-case
            """).registrations();

    assertThat(regs).hasSize(1);
    var def = regs.get(0).definition();
    assertThat(def.situationId()).isEqualTo("test-sit");
    assertThat(def.eventTypes()).containsExactly("test.event");
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Streak.class);
    var streak = (ChainMode.Streak) def.chainMode();
    assertThat(streak.ganglionId()).isEqualTo("my-ganglion");
    assertThat(streak.requiredCount()).isEqualTo(3);
    assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(10));
    assertThat(def.triggerAction()).isInstanceOf(TriggerAction.CreateCase.class);
    var createCase = (TriggerAction.CreateCase) def.triggerAction();
    assertThat(createCase.config().caseNamespace()).isEqualTo("test-ns");
    assertThat(createCase.config().caseName()).isEqualTo("my-case");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#templateInstantiationProducesCorrectDefinition`
Expected: FAIL — template handling not yet implemented.

- [ ] **Step 3: Add internal types and core methods**

Add to `YamlSituationDefinitionProvider.java`:

1. Two nested records (after `ParseResult`):
```java
record SituationTemplate(
        String id,
        String description,
        Map<String, ParameterDef> parameters,
        Map<String, Object> definition,
        List<Map<String, Object>> ganglia
) {}

record ParameterDef(boolean required, Object defaultValue) {}
```

2. Two regex patterns (as static fields):
```java
private static final java.util.regex.Pattern WHOLE_PARAM_PATTERN =
        java.util.regex.Pattern.compile("^\\$\\{([^}]+)}$");
private static final java.util.regex.Pattern PARAM_PATTERN =
        java.util.regex.Pattern.compile("\\$\\{([^}]+)}");
```

3. `parseTemplates` method:
```java
@SuppressWarnings("unchecked")
static Map<String, SituationTemplate> parseTemplates(Map<String, Object> root) {
    List<Map<String, Object>> templates = (List<Map<String, Object>>) root.get("templates");
    if (templates == null) {return Map.of();}
    Map<String, SituationTemplate> result = new LinkedHashMap<>();
    for (Map<String, Object> t : templates) {
        String id = requireString(t, "id");
        String description = (String) t.get("description");
        Map<String, ParameterDef> params = parseParameterDefs(
                (Map<String, Object>) t.get("parameters"));
        Map<String, Object> definition = (Map<String, Object>) t.get("definition");
        if (definition == null) {
            throw new IllegalArgumentException("Template '" + id + "' has no definition");
        }
        List<Map<String, Object>> ganglia = (List<Map<String, Object>>) t.get("ganglia");
        result.put(id, new SituationTemplate(id, description, params, definition, ganglia));
    }
    return result;
}

@SuppressWarnings("unchecked")
private static Map<String, ParameterDef> parseParameterDefs(Map<String, Object> raw) {
    if (raw == null) {return Map.of();}
    Map<String, ParameterDef> result = new LinkedHashMap<>();
    for (var entry : raw.entrySet()) {
        Map<String, Object> defMap = (Map<String, Object>) entry.getValue();
        boolean required = Boolean.TRUE.equals(defMap.get("required"));
        Object defaultValue = defMap.get("default");
        result.put(entry.getKey(), new ParameterDef(required, defaultValue));
    }
    return result;
}
```

4. `substituteParams` method:
```java
@SuppressWarnings("unchecked")
static Object substituteParams(Object value, Map<String, Object> resolvedParams) {
    if (value instanceof String s) {
        java.util.regex.Matcher wholeMatcher = WHOLE_PARAM_PATTERN.matcher(s);
        if (wholeMatcher.matches()) {
            String name = wholeMatcher.group(1);
            return resolvedParams.containsKey(name) ? resolvedParams.get(name) : s;
        }
        return PARAM_PATTERN.matcher(s).replaceAll(mr -> {
            String name = mr.group(1);
            return resolvedParams.containsKey(name)
                    ? java.util.regex.Matcher.quoteReplacement(resolvedParams.get(name).toString())
                    : java.util.regex.Matcher.quoteReplacement(mr.group(0));
        });
    }
    if (value instanceof Map<?, ?> map) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (var entry : ((Map<String, Object>) map).entrySet()) {
            result.put(entry.getKey(), substituteParams(entry.getValue(), resolvedParams));
        }
        return result;
    }
    if (value instanceof List<?> list) {
        List<Object> result = new ArrayList<>(list.size());
        for (Object item : list) {
            result.add(substituteParams(item, resolvedParams));
        }
        return result;
    }
    return value;
}
```

5. `checkUnresolved` method:
```java
@SuppressWarnings("unchecked")
static void checkUnresolved(Object value, String templateId) {
    if (value instanceof String s && PARAM_PATTERN.matcher(s).find()) {
        throw new IllegalArgumentException(
                "Unresolved parameter in template '" + templateId + "': " + s);
    }
    if (value instanceof Map<?, ?> map) {
        for (var entry : ((Map<String, Object>) map).entrySet()) {
            if (PARAM_PATTERN.matcher(entry.getKey()).find()) {
                throw new IllegalArgumentException(
                        "Parameter placeholder in map key not supported in template '"
                        + templateId + "': " + entry.getKey());
            }
            checkUnresolved(entry.getValue(), templateId);
        }
    }
    if (value instanceof List<?> list) {
        for (Object item : list) {
            checkUnresolved(item, templateId);
        }
    }
}
```

6. `deepMerge` method:
```java
@SuppressWarnings("unchecked")
static Map<String, Object> deepMerge(Map<String, Object> base,
                                      Map<String, Object> overrides) {
    Map<String, Object> result = new LinkedHashMap<>(base);
    for (var entry : overrides.entrySet()) {
        Object baseValue = result.get(entry.getKey());
        Object overrideValue = entry.getValue();
        if (baseValue instanceof Map && overrideValue instanceof Map) {
            result.put(entry.getKey(), deepMerge(
                    (Map<String, Object>) baseValue,
                    (Map<String, Object>) overrideValue));
        } else {
            result.put(entry.getKey(), overrideValue);
        }
    }
    return result;
}
```

7. `resolveTemplate` method:
```java
@SuppressWarnings("unchecked")
static Map<String, Object> resolveTemplate(Map<String, Object> situationMap,
                                            Map<String, SituationTemplate> templates) {
    String templateId = (String) situationMap.get("fromTemplate");
    SituationTemplate template = templates.get(templateId);
    if (template == null) {
        throw new IllegalArgumentException("Unknown template: '" + templateId + "'");
    }

    String situationId = requireString(situationMap, "situationId");
    Object eventTypes = situationMap.get("eventTypes");

    Map<String, Object> resolvedParams = new LinkedHashMap<>();
    for (var entry : template.parameters().entrySet()) {
        if (entry.getValue().defaultValue() != null) {
            resolvedParams.put(entry.getKey(), entry.getValue().defaultValue());
        }
    }
    if (template.parameters().containsKey("situationId")) {
        resolvedParams.put("situationId", situationId);
    }
    if (template.parameters().containsKey("eventTypes")) {
        resolvedParams.put("eventTypes", eventTypes);
    }
    Map<String, Object> consumerParams = (Map<String, Object>) situationMap.get("parameters");
    if (consumerParams != null) {
        resolvedParams.putAll(consumerParams);
        for (String key : consumerParams.keySet()) {
            if (!template.parameters().containsKey(key)) {
                LOG.warning("Unknown parameter '" + key + "' for template '" + templateId + "'");
            }
        }
    }
    for (var entry : template.parameters().entrySet()) {
        if (entry.getValue().required() && !resolvedParams.containsKey(entry.getKey())) {
            throw new IllegalArgumentException(
                    "Missing required parameter '" + entry.getKey()
                    + "' for template '" + templateId + "'");
        }
    }

    Map<String, Object> resolved = (Map<String, Object>) substituteParams(
            new LinkedHashMap<>(template.definition()), resolvedParams);
    checkUnresolved(resolved, templateId);

    Map<String, Object> overrides = new LinkedHashMap<>();
    for (var entry : situationMap.entrySet()) {
        String key = entry.getKey();
        if (!"fromTemplate".equals(key) && !"situationId".equals(key)
            && !"eventTypes".equals(key) && !"parameters".equals(key)) {
            overrides.put(key, entry.getValue());
        }
    }
    if (!overrides.isEmpty()) {
        resolved = deepMerge(resolved, overrides);
    }

    resolved.put("situationId", situationId);
    resolved.put("eventTypes", eventTypes);
    return resolved;
}
```

8. Modify `parseSituations` signature and body — add template parameter and
   `fromTemplate:` detection:
```java
@SuppressWarnings("unchecked")
private static List<SituationRegistration> parseSituations(Map<String, Object> root,
                                                            Map<String, SituationTemplate> templates) {
    if (!root.containsKey("situations")) {return List.of();}
    List<Map<String, Object>> situations = (List<Map<String, Object>>) root.get("situations");
    List<SituationRegistration> result = new ArrayList<>(situations.size());
    for (Map<String, Object> sit : situations) {
        if (sit.containsKey("fromTemplate")) {
            String templateId = (String) sit.get("fromTemplate");
            String sitId = String.valueOf(sit.getOrDefault("situationId", "<missing>"));
            Map<String, Object> resolved;
            try {
                resolved = resolveTemplate(sit, templates);
            } catch (Exception e) {
                throw new IllegalArgumentException(
                        "Error resolving template '" + templateId
                        + "' for situation '" + sitId + "': " + e.getMessage(), e);
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
    return List.copyOf(result);
}
```

9. Modify `parseAll` to parse templates and pass to `parseSituations`:
```java
@SuppressWarnings("unchecked")
private static ParseResult parseAll(InputStream yaml) {
    Map<String, Object> root = new Yaml().load(yaml);
    if (root == null) {return new ParseResult(List.of(), List.of());}
    Map<String, SituationTemplate> templates = parseTemplates(root);
    List<GanglionDescriptor>    ganglia    = parseGanglia(root);
    List<SituationRegistration> situations = parseSituations(root, templates);
    return new ParseResult(situations, ganglia);
}
```

Add required import: `import java.util.regex.Matcher;`

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#templateInstantiationProducesCorrectDefinition`
Expected: PASS

- [ ] **Step 5: Write remaining template tests**

Add these tests to `YamlSituationDefinitionProviderTest.java`:

```java
@Test
void templateDefaultParameterValuesApplied() {
    var regs = provider("""
            templates:
              - id: with-defaults
                parameters:
                  ganglionId: {required: true}
                  count: {default: 5}
                  window: {default: PT30M}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: count
                    ganglionId: ${ganglionId}
                    requiredCount: ${count}
                  correlationWindow: ${window}
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: with-defaults
                situationId: test-defaults
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Count.class);
    assertThat(((ChainMode.Count) def.chainMode()).requiredCount()).isEqualTo(5);
    assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(30));
}

@Test
void templateWholeValueSubstitutionPreservesListType() {
    var regs = provider("""
            templates:
              - id: list-param
                parameters:
                  ganglia: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: threshold
                    ganglia: ${ganglia}
                    minConfidence: 0.8
                  correlationWindow: PT5M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: list-param
                situationId: test-list
                eventTypes: [test.event]
                parameters:
                  ganglia: [g1, g2]
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var threshold = (ChainMode.Threshold) regs.get(0).definition().chainMode();
    assertThat(threshold.ganglia()).containsExactlyInAnyOrder("g1", "g2");
}

@Test
void templateSubstringInterpolationProducesString() {
    var regs = provider("""
            templates:
              - id: substring
                parameters:
                  env: {required: true}
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: sla-${env}-breach
                    caseVersion: "1"
            situations:
              - fromTemplate: substring
                situationId: test-sub
                eventTypes: [test.event]
                parameters:
                  env: prod
                  ganglionId: g1
                  caseNamespace: ns
            """).registrations();

    var cc = (TriggerAction.CreateCase) regs.get(0).definition().triggerAction();
    assertThat(cc.config().caseName()).isEqualTo("sla-prod-breach");
}

@Test
void templateMissingRequiredParameterThrows() {
    assertThatIllegalArgumentException().isThrownBy(() -> provider("""
            templates:
              - id: needs-ganglion
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: needs-ganglion
                situationId: test-missing
                eventTypes: [test.event]
                parameters:
                  caseNamespace: ns
                  caseName: cn
            """))
            .withMessageContaining("ganglionId")
            .withMessageContaining("needs-ganglion");
}

@Test
void templateUnknownTemplateThrows() {
    assertThatIllegalArgumentException().isThrownBy(() -> provider("""
            situations:
              - fromTemplate: nonexistent
                situationId: test-unknown
                eventTypes: [test.event]
                parameters: {}
            """))
            .withMessageContaining("nonexistent");
}

@Test
void templateUnresolvedPlaceholderThrows() {
    assertThatIllegalArgumentException().isThrownBy(() -> provider("""
            templates:
              - id: typo
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglonId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: typo
                situationId: test-typo
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """))
            .withMessageContaining("${ganglonId}");
}

@Test
void templateConsumerOverridesTemplateField() {
    var regs = provider("""
            templates:
              - id: overridable
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
                  triggerMode:
                    type: fire-once
            situations:
              - fromTemplate: overridable
                situationId: test-override
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
                triggerMode:
                  type: repeating
                  cooldown: PT5M
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.triggerMode()).isInstanceOf(TriggerMode.Repeating.class);
    assertThat(((TriggerMode.Repeating) def.triggerMode()).cooldown())
            .isEqualTo(Duration.ofMinutes(5));
}

@Test
void templateDeepMergeOverridesSingleNestedKey() {
    var regs = provider("""
            templates:
              - id: merge-test
                parameters:
                  ganglionId: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: default-ns
                    caseName: default-case
                    caseVersion: "1"
            situations:
              - fromTemplate: merge-test
                situationId: test-merge
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                triggerAction:
                  caseName: overridden-case
            """).registrations();

    var cc = (TriggerAction.CreateCase) regs.get(0).definition().triggerAction();
    assertThat(cc.config().caseNamespace()).isEqualTo("default-ns");
    assertThat(cc.config().caseName()).isEqualTo("overridden-case");
}

@Test
void templateIdentityFieldInjectsIntoParameters() {
    var regs = provider("""
            templates:
              - id: identity-test
                parameters:
                  ganglionId: {required: true}
                  situationId: {required: true}
                  caseNamespace: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: case-for-${situationId}
                    caseVersion: "1"
            situations:
              - fromTemplate: identity-test
                situationId: my-sit-id
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
            """).registrations();

    var cc = (TriggerAction.CreateCase) regs.get(0).definition().triggerAction();
    assertThat(cc.config().caseName()).isEqualTo("case-for-my-sit-id");
}

@Test
void templateAndHandWrittenProduceIdenticalDefinition() {
    var fromTemplate = provider("""
            templates:
              - id: streak-pattern
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
                  triggerMode:
                    type: fire-once
            situations:
              - fromTemplate: streak-pattern
                situationId: equiv-sit
                eventTypes: [e1, e2]
                parameters:
                  ganglionId: my-g
                  caseNamespace: my-ns
                  caseName: my-case
            """).registrations().get(0).definition();

    var handWritten = provider("""
            situations:
              - situationId: equiv-sit
                eventTypes: [e1, e2]
                chainMode:
                  type: streak
                  ganglionId: my-g
                  requiredCount: 3
                correlationWindow: PT10M
                triggerAction:
                  type: create-case
                  caseNamespace: my-ns
                  caseName: my-case
                  caseVersion: "1"
                triggerMode:
                  type: fire-once
            """).registrations().get(0).definition();

    assertThat(fromTemplate).isEqualTo(handWritten);
}

@Test
void templateErrorWrapsWithTemplateContext() {
    assertThatIllegalArgumentException().isThrownBy(() -> provider("""
            templates:
              - id: bad-template
                parameters:
                  ganglionId: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
            situations:
              - fromTemplate: bad-template
                situationId: test-error
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
            """))
            .withMessageContaining("bad-template")
            .withMessageContaining("test-error");
}

@Test
void templateMixedWithHandWrittenSituations() {
    var regs = provider("""
            templates:
              - id: simple
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: simple
                situationId: templated-sit
                eventTypes: [t.event]
                parameters:
                  ganglionId: tg1
                  caseNamespace: tns
                  caseName: tcn
              - situationId: hand-written-sit
                eventTypes: [h.event]
                chainMode:
                  type: or
                  ganglia: [hg1]
                correlationWindow: PT5M
                triggerAction:
                  type: notify-only
            """).registrations();

    assertThat(regs).hasSize(2);
    assertThat(regs.get(0).definition().situationId()).isEqualTo("templated-sit");
    assertThat(regs.get(1).definition().situationId()).isEqualTo("hand-written-sit");
}

@Test
void templateConsumerOverrideAddsEventFilter() {
    var regs = provider("""
            templates:
              - id: filterable
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: filterable
                situationId: test-filter
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
                eventFilter:
                  expression: ".data.severity"
                  language: jq
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.eventFilter()).isNotNull();
    assertThat(def.eventFilter()).isInstanceOf(JQExpressionEvaluator.class);
}
```

- [ ] **Step 6: Run all template tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest`
Expected: ALL PASS

- [ ] **Step 7: Run full module tests to verify no regressions**

Run: `mvn --batch-mode test -pl runtime`
Expected: ALL PASS — existing tests unchanged.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java \
        runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java
git commit -m "feat(#42): template parsing, parameter substitution, and resolution"
```

---

### Task 2: Two-file loading + built-in template library

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Create: `runtime/src/main/resources/META-INF/ras-situation-templates.yaml`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: `parseTemplates(Map)` from Task 1
- Produces: Built-in templates loadable via classpath. `parseAll(InputStream, Map)` overload.

- [ ] **Step 1: Write failing test — built-in templates available without inline declaration**

```java
@Test
void builtInTemplateStreakBreachAvailableWithoutDeclaration() {
    var regs = provider("""
            situations:
              - fromTemplate: streak-breach
                situationId: test-builtin
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    assertThat(regs).hasSize(1);
    var def = regs.get(0).definition();
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Streak.class);
    assertThat(((ChainMode.Streak) def.chainMode()).requiredCount()).isEqualTo(3);
    assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(10));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#builtInTemplateStreakBreachAvailableWithoutDeclaration`
Expected: FAIL — "Unknown template: 'streak-breach'"

- [ ] **Step 3: Create built-in templates resource**

Create `runtime/src/main/resources/META-INF/ras-situation-templates.yaml`:

```yaml
templates:
  - id: streak-breach
    description: "Triggers when same ganglion fires N times consecutively"
    parameters:
      ganglionId: {required: true}
      requiredCount: {default: 3}
      window: {default: PT10M}
      caseNamespace: {required: true}
      caseName: {required: true}
      caseVersion: {default: "1"}
    definition:
      chainMode:
        type: streak
        ganglionId: ${ganglionId}
        requiredCount: ${requiredCount}
      correlationWindow: ${window}
      triggerAction:
        type: create-case
        caseNamespace: ${caseNamespace}
        caseName: ${caseName}
        caseVersion: ${caseVersion}
      triggerMode:
        type: fire-once

  - id: threshold-crossing
    description: "Triggers when ganglion confidence exceeds threshold"
    parameters:
      ganglia: {required: true}
      minConfidence: {default: 0.8}
      window: {default: PT5M}
      caseNamespace: {required: true}
      caseName: {required: true}
      caseVersion: {default: "1"}
    definition:
      chainMode:
        type: threshold
        ganglia: ${ganglia}
        minConfidence: ${minConfidence}
      correlationWindow: ${window}
      triggerAction:
        type: create-case
        caseNamespace: ${caseNamespace}
        caseName: ${caseName}
        caseVersion: ${caseVersion}
      triggerMode:
        type: fire-once

  - id: count-accumulation
    description: "Triggers when ganglion fires N times in a window"
    parameters:
      ganglionId: {required: true}
      requiredCount: {default: 5}
      window: {default: PT10M}
      caseNamespace: {required: true}
      caseName: {required: true}
      caseVersion: {default: "1"}
    definition:
      chainMode:
        type: count
        ganglionId: ${ganglionId}
        requiredCount: ${requiredCount}
      correlationWindow: ${window}
      triggerAction:
        type: create-case
        caseNamespace: ${caseNamespace}
        caseName: ${caseName}
        caseVersion: ${caseVersion}
      triggerMode:
        type: fire-once

  - id: rate-breach
    description: "Triggers when ganglion fire rate exceeds threshold in sliding window"
    parameters:
      ganglia: {required: true}
      minRate: {default: 0.6}
      windowSize: {default: 10}
      window: {default: PT30M}
      caseNamespace: {required: true}
      caseName: {required: true}
      caseVersion: {default: "1"}
    definition:
      chainMode:
        type: rate
        ganglia: ${ganglia}
        minRate: ${minRate}
        windowSize: ${windowSize}
      correlationWindow: ${window}
      triggerAction:
        type: create-case
        caseNamespace: ${caseNamespace}
        caseName: ${caseName}
        caseVersion: ${caseVersion}
      triggerMode:
        type: fire-once
```

- [ ] **Step 4: Implement two-file loading**

Modify `YamlSituationDefinitionProvider`:

1. Add overloaded `parseAll`:
```java
@SuppressWarnings("unchecked")
private static ParseResult parseAll(InputStream yaml,
                                     Map<String, SituationTemplate> builtInTemplates) {
    Map<String, Object> root = new Yaml().load(yaml);
    if (root == null) {return new ParseResult(List.of(), List.of());}
    Map<String, SituationTemplate> templates = new LinkedHashMap<>(builtInTemplates);
    templates.putAll(parseTemplates(root));
    List<GanglionDescriptor>    ganglia    = parseGanglia(root);
    List<SituationRegistration> situations = parseSituations(root, templates);
    return new ParseResult(situations, ganglia);
}
```

2. Update no-arg `parseAll` to delegate:
```java
@SuppressWarnings("unchecked")
private static ParseResult parseAll(InputStream yaml) {
    return parseAll(yaml, Map.of());
}
```

3. Add `loadBuiltInTemplates` method:
```java
private static Map<String, SituationTemplate> loadBuiltInTemplates() {
    InputStream is = Thread.currentThread().getContextClassLoader()
                           .getResourceAsStream("META-INF/ras-situation-templates.yaml");
    if (is == null) {return Map.of();}
    try (is) {
        Map<String, Object> root = new Yaml().load(is);
        return root != null ? parseTemplates(root) : Map.of();
    } catch (IOException e) {
        throw new UncheckedIOException("Failed to read built-in templates", e);
    }
}
```

4. Modify CDI constructor to use two-file loading:
```java
@Inject
YamlSituationDefinitionProvider(
        @ConfigProperty(name = "ras.situations.yaml",
                        defaultValue = "META-INF/ras-situations.yaml") String resourcePath) {
    Map<String, SituationTemplate> builtInTemplates = loadBuiltInTemplates();

    InputStream is = Thread.currentThread().getContextClassLoader()
                           .getResourceAsStream(resourcePath);
    if (is == null) {
        LOG.fine("No YAML situation definitions found at " + resourcePath);
        this.registrations       = List.of();
        this.ganglionDescriptors = List.of();
    } else {
        try (is) {
            var parsed = parseAll(is, builtInTemplates);
            this.registrations       = parsed.registrations();
            this.ganglionDescriptors = parsed.ganglionDescriptors();
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to read " + resourcePath, e);
        }
    }
}
```

5. Modify test constructor to also load built-in templates:
```java
YamlSituationDefinitionProvider(InputStream yaml) {
    var parsed = parseAll(yaml, loadBuiltInTemplates());
    this.registrations       = parsed.registrations();
    this.ganglionDescriptors = parsed.ganglionDescriptors();
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#builtInTemplateStreakBreachAvailableWithoutDeclaration`
Expected: PASS

- [ ] **Step 6: Write remaining two-file loading tests**

```java
@Test
void builtInTemplateThresholdCrossingAvailable() {
    var regs = provider("""
            situations:
              - fromTemplate: threshold-crossing
                situationId: test-threshold
                eventTypes: [test.event]
                parameters:
                  ganglia: [g1]
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Threshold.class);
    assertThat(((ChainMode.Threshold) def.chainMode()).minConfidence()).isEqualTo(0.8);
    assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(5));
}

@Test
void builtInTemplateCountAccumulationAvailable() {
    var regs = provider("""
            situations:
              - fromTemplate: count-accumulation
                situationId: test-count
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Count.class);
    assertThat(((ChainMode.Count) def.chainMode()).requiredCount()).isEqualTo(5);
}

@Test
void builtInTemplateRateBreachAvailable() {
    var regs = provider("""
            situations:
              - fromTemplate: rate-breach
                situationId: test-rate
                eventTypes: [test.event]
                parameters:
                  ganglia: [g1]
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var def = regs.get(0).definition();
    assertThat(def.chainMode()).isInstanceOf(ChainMode.Rate.class);
    var rate = (ChainMode.Rate) def.chainMode();
    assertThat(rate.minRate()).isEqualTo(0.6);
    assertThat(rate.windowSize()).isEqualTo(10);
    assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(30));
}

@Test
void consumerTemplateOverridesBuiltInWithSameId() {
    var regs = provider("""
            templates:
              - id: streak-breach
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 99
                  correlationWindow: PT99M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: streak-breach
                situationId: test-override
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """).registrations();

    var streak = (ChainMode.Streak) regs.get(0).definition().chainMode();
    assertThat(streak.requiredCount()).isEqualTo(99);
    assertThat(regs.get(0).definition().correlationWindow()).isEqualTo(Duration.ofMinutes(99));
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn --batch-mode test -pl runtime`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java \
        runtime/src/main/resources/META-INF/ras-situation-templates.yaml \
        runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java
git commit -m "feat(#42): two-file loading + built-in template library (4 templates)"
```

---

### Task 3: Ganglion bundling

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`
- Test: `runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: `SituationTemplate.ganglia()`, `substituteParams()`, `parseGanglia()`
- Produces: Bundled `GanglionDescriptor` objects in `ganglionDescriptors()` return list

- [ ] **Step 1: Write failing test — bundled ganglion descriptor returned**

```java
@Test
void templateBundledGanglionReturned() {
    var provider = provider("""
            templates:
              - id: with-ganglion
                parameters:
                  ganglionId: {required: true}
                  eventTypes: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                ganglia:
                  - ganglionId: ${ganglionId}
                    type: expression-rules
                    handledEventTypes: ${eventTypes}
                    rules:
                      - when:
                          expression: "true"
                          language: mvel
                        signal: DETECTED
                        confidence: 0.9
                definition:
                  chainMode:
                    type: or
                    ganglia: [${ganglionId}]
                  correlationWindow: PT5M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: with-ganglion
                situationId: test-bundled
                eventTypes: [sensor.reading]
                parameters:
                  ganglionId: bundled-g
                  eventTypes: [sensor.reading]
                  caseNamespace: ns
                  caseName: cn
            """);

    assertThat(provider.registrations()).hasSize(1);
    assertThat(provider.ganglionDescriptors()).hasSize(1);
    var descriptor = provider.ganglionDescriptors().get(0);
    assertThat(descriptor).isInstanceOf(io.casehub.ras.api.GanglionDescriptor.ExpressionRules.class);
    assertThat(descriptor.ganglionId()).isEqualTo("bundled-g");
    assertThat(descriptor.handledEventTypes()).containsExactly("sensor.reading");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#templateBundledGanglionReturned`
Expected: FAIL — bundled ganglia not yet implemented.

- [ ] **Step 3: Implement ganglion bundling**

1. Add internal record for situation parse results:
```java
private record SituationParseResult(
        List<SituationRegistration> registrations,
        List<GanglionDescriptor> bundledGanglia
) {}
```

2. Change `parseSituations` to return `SituationParseResult` and accumulate
   bundled ganglia from resolved templates:
```java
@SuppressWarnings("unchecked")
private static SituationParseResult parseSituations(Map<String, Object> root,
                                                     Map<String, SituationTemplate> templates) {
    if (!root.containsKey("situations")) {
        return new SituationParseResult(List.of(), List.of());
    }
    List<Map<String, Object>> situations = (List<Map<String, Object>>) root.get("situations");
    List<SituationRegistration> result = new ArrayList<>(situations.size());
    List<GanglionDescriptor> bundledGanglia = new ArrayList<>();
    for (Map<String, Object> sit : situations) {
        if (sit.containsKey("fromTemplate")) {
            String templateId = (String) sit.get("fromTemplate");
            String sitId = String.valueOf(sit.getOrDefault("situationId", "<missing>"));
            Map<String, Object> resolved;
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
                Map<String, Object> gangliaRoot = Map.of("ganglia",
                        substituteParams(deepCopyList(template.ganglia()), resolvedParams));
                checkUnresolved(gangliaRoot, templateId);
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

3. Extract `buildResolvedParams` from `resolveTemplate` to avoid duplication:
```java
@SuppressWarnings("unchecked")
private static Map<String, Object> buildResolvedParams(Map<String, Object> situationMap,
                                                        SituationTemplate template) {
    String situationId = requireString(situationMap, "situationId");
    Object eventTypes = situationMap.get("eventTypes");

    Map<String, Object> resolvedParams = new LinkedHashMap<>();
    for (var entry : template.parameters().entrySet()) {
        if (entry.getValue().defaultValue() != null) {
            resolvedParams.put(entry.getKey(), entry.getValue().defaultValue());
        }
    }
    if (template.parameters().containsKey("situationId")) {
        resolvedParams.put("situationId", situationId);
    }
    if (template.parameters().containsKey("eventTypes")) {
        resolvedParams.put("eventTypes", eventTypes);
    }
    Map<String, Object> consumerParams = (Map<String, Object>) situationMap.get("parameters");
    if (consumerParams != null) {
        resolvedParams.putAll(consumerParams);
    }
    return resolvedParams;
}
```

4. Update `resolveTemplate` to use `buildResolvedParams`:
```java
static Map<String, Object> resolveTemplate(Map<String, Object> situationMap,
                                            Map<String, SituationTemplate> templates) {
    String templateId = (String) situationMap.get("fromTemplate");
    SituationTemplate template = templates.get(templateId);
    if (template == null) {
        throw new IllegalArgumentException("Unknown template: '" + templateId + "'");
    }

    Map<String, Object> resolvedParams = buildResolvedParams(situationMap, template);

    // Validate
    Map<String, Object> consumerParams = (Map<String, Object>) situationMap.get("parameters");
    if (consumerParams != null) {
        for (String key : consumerParams.keySet()) {
            if (!template.parameters().containsKey(key)) {
                LOG.warning("Unknown parameter '" + key + "' for template '" + templateId + "'");
            }
        }
    }
    for (var entry : template.parameters().entrySet()) {
        if (entry.getValue().required() && !resolvedParams.containsKey(entry.getKey())) {
            throw new IllegalArgumentException(
                    "Missing required parameter '" + entry.getKey()
                    + "' for template '" + templateId + "'");
        }
    }

    Map<String, Object> resolved = (Map<String, Object>) substituteParams(
            new LinkedHashMap<>(template.definition()), resolvedParams);
    checkUnresolved(resolved, templateId);

    // ... rest unchanged (overrides, identity injection)
}
```

5. Add `deepCopyList` helper:
```java
@SuppressWarnings("unchecked")
private static List<Map<String, Object>> deepCopyList(List<Map<String, Object>> list) {
    List<Map<String, Object>> copy = new ArrayList<>(list.size());
    for (Map<String, Object> item : list) {
        copy.add(new LinkedHashMap<>(item));
    }
    return copy;
}
```

6. Update both `parseAll` overloads — change from `parseSituations` returning
   `List<SituationRegistration>` to `SituationParseResult`:
```java
private static ParseResult parseAll(InputStream yaml,
                                     Map<String, SituationTemplate> builtInTemplates) {
    Map<String, Object> root = new Yaml().load(yaml);
    if (root == null) {return new ParseResult(List.of(), List.of());}
    Map<String, SituationTemplate> templates = new LinkedHashMap<>(builtInTemplates);
    templates.putAll(parseTemplates(root));
    List<GanglionDescriptor>    ganglia    = parseGanglia(root);
    SituationParseResult sitResult = parseSituations(root, templates);
    List<GanglionDescriptor> allGanglia = new ArrayList<>(ganglia);
    allGanglia.addAll(sitResult.bundledGanglia());
    return new ParseResult(sitResult.registrations(), List.copyOf(allGanglia));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#templateBundledGanglionReturned`
Expected: PASS

- [ ] **Step 5: Write additional ganglion bundling tests**

```java
@Test
void templateWithoutGangliaSectionEmitsNoDescriptors() {
    var provider = provider("""
            templates:
              - id: no-ganglia
                parameters:
                  ganglionId: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                definition:
                  chainMode:
                    type: streak
                    ganglionId: ${ganglionId}
                    requiredCount: 3
                  correlationWindow: PT10M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: no-ganglia
                situationId: test-no-ganglia
                eventTypes: [test.event]
                parameters:
                  ganglionId: g1
                  caseNamespace: ns
                  caseName: cn
            """);

    assertThat(provider.registrations()).hasSize(1);
    assertThat(provider.ganglionDescriptors()).isEmpty();
}

@Test
void templateBundledGanglionUsesIdentityFieldEventTypes() {
    var provider = provider("""
            templates:
              - id: identity-ganglia
                parameters:
                  ganglionId: {required: true}
                  eventTypes: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                ganglia:
                  - ganglionId: ${ganglionId}
                    type: expression-rules
                    handledEventTypes: ${eventTypes}
                    rules:
                      - otherwise: true
                        signal: NOISE
                        confidence: 0.1
                definition:
                  chainMode:
                    type: or
                    ganglia: [${ganglionId}]
                  correlationWindow: PT5M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: identity-ganglia
                situationId: test-identity-g
                eventTypes: [my.custom.event]
                parameters:
                  ganglionId: id-g
                  eventTypes: [my.custom.event]
                  caseNamespace: ns
                  caseName: cn
            """);

    assertThat(provider.ganglionDescriptors()).hasSize(1);
    assertThat(provider.ganglionDescriptors().get(0).handledEventTypes())
            .containsExactly("my.custom.event");
}

@Test
void templateBundledGangliaCoexistWithTopLevelGanglia() {
    var provider = provider("""
            ganglia:
              - ganglionId: top-level-g
                type: expression-rules
                handledEventTypes: [top.event]
                rules:
                  - otherwise: true
                    signal: NOISE
                    confidence: 0.1
            templates:
              - id: with-ganglia
                parameters:
                  ganglionId: {required: true}
                  eventTypes: {required: true}
                  caseNamespace: {required: true}
                  caseName: {required: true}
                ganglia:
                  - ganglionId: ${ganglionId}
                    type: expression-rules
                    handledEventTypes: ${eventTypes}
                    rules:
                      - otherwise: true
                        signal: NOISE
                        confidence: 0.1
                definition:
                  chainMode:
                    type: or
                    ganglia: [${ganglionId}]
                  correlationWindow: PT5M
                  triggerAction:
                    type: create-case
                    caseNamespace: ${caseNamespace}
                    caseName: ${caseName}
                    caseVersion: "1"
            situations:
              - fromTemplate: with-ganglia
                situationId: test-coexist
                eventTypes: [bundled.event]
                parameters:
                  ganglionId: bundled-g
                  eventTypes: [bundled.event]
                  caseNamespace: ns
                  caseName: cn
            """);

    assertThat(provider.ganglionDescriptors()).hasSize(2);
    assertThat(provider.ganglionDescriptors().stream()
                       .map(io.casehub.ras.api.GanglionDescriptor::ganglionId))
            .containsExactlyInAnyOrder("top-level-g", "bundled-g");
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl runtime`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java \
        runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java
git commit -m "feat(#42): ganglion bundling in situation templates"
```

---

### Task 4: End-to-end registry integration

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ras/runtime/SituationDefinitionRegistryTest.java`
  (or `YamlSituationDefinitionProviderTest.java` — use whichever has the existing e2e pattern)

**Interfaces:**
- Consumes: `YamlSituationDefinitionProvider`, `SituationDefinitionRegistry`, `SituationEvaluator`
- Produces: Verified end-to-end template→detection path

- [ ] **Step 1: Write end-to-end test — template-instantiated situation detects through registry**

Add to `YamlSituationDefinitionProviderTest.java` (following the existing
`endToEndYamlNaiveBayesGanglionDetectsAndTriggers` pattern):

```java
@Test
void endToEndTemplateInstantiatedSituationRegistersAndDetects() {
    var provider = provider("""
            ganglia:
              - ganglionId: e2e-template-g
                type: expression-rules
                handledEventTypes: [test.template.e2e]
                rules:
                  - when:
                      expression: ".data.severity == \\"HIGH\\""
                      language: jq
                    signal: DETECTED
                    confidence: 0.9
                  - otherwise: true
                    signal: NOISE
                    confidence: 0.1
            situations:
              - fromTemplate: streak-breach
                situationId: e2e-template-sit
                eventTypes: [test.template.e2e]
                parameters:
                  ganglionId: e2e-template-g
                  caseNamespace: e2e
                  caseName: template-test
            """);

    var registry = new SituationDefinitionRegistry(
            List.of(provider), List.of());

    assertThat(registry.definitionCount()).isEqualTo(1);
    var regs = registry.findByEventType("test.template.e2e");
    assertThat(regs).hasSize(1);
    assertThat(regs.get(0).definition().situationId()).isEqualTo("e2e-template-sit");

    var ganglion = registry.ganglion("e2e-template-g");
    assertThat(ganglion).isNotNull();
    assertThat(ganglion.ganglionId()).isEqualTo("e2e-template-g");
}
```

- [ ] **Step 2: Run test**

Run: `mvn --batch-mode test -pl runtime -Dtest=YamlSituationDefinitionProviderTest#endToEndTemplateInstantiatedSituationRegistersAndDetects`
Expected: PASS (uses built-in `streak-breach` template + explicit ganglion)

- [ ] **Step 3: Run full project build**

Run: `mvn --batch-mode install`
Expected: ALL PASS across all modules — templates are internal to runtime/,
no API changes, no cross-module impact.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ras/runtime/YamlSituationDefinitionProviderTest.java
git commit -m "test(#42): end-to-end template→registry integration test"
```

---

### Task 5: CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- None — documentation only

- [ ] **Step 1: Update CLAUDE.md YAML Situations section**

Add template documentation to the existing "YAML Situation Definitions"
section — template format, built-in templates, `fromTemplate:` usage.

Add to the `## YAML Ganglia` subsection's sibling level:

```markdown
### YAML Situation Templates (runtime/)

`YamlSituationDefinitionProvider` supports reusable situation templates with typed
parameters. Templates defined in `templates:` section (or built-in from
`META-INF/ras-situation-templates.yaml`). Consumers instantiate via `fromTemplate:`
+ `parameters:`. Resolution happens at parse time — registry sees fully resolved
`SituationRegistration` objects.

Built-in templates: `streak-breach`, `threshold-crossing`, `count-accumulation`,
`rate-breach`. Consumer templates override built-in with same `id`. Templates can
optionally bundle `ganglia:` section for parameterised ganglion descriptors.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#42): add situation templates to CLAUDE.md"
```
