# Classpath YAML Bootstrap + Jackson YAML Alignment

**Issue:** casehubio/casehub-ras#70
**Branch:** issue-70-classpath-yaml-registrar

## Summary

Move `SituationYamlRegistrar` from engine to ras and migrate all ras YAML parsing from
SnakeYAML to Jackson. This decouples engine from ras-api (engine can drop its ras dependency)
and aligns ras on a single YAML parsing strategy consistent with the rest of the platform.

Four deliverables:
1. Jackson annotations on domain types in api/
2. ExpressionEvaluator Jackson deserializer in runtime/
3. Migrate YamlSituationDefinitionProvider from SnakeYAML to Jackson
4. New SituationYamlRegistrar implementing ClasspathYamlRegistrar

## 1. Jackson Annotations on Domain Types (api/)

### Already annotated
- `ChainMode` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for all 7 variants
- `TriggerAction` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for `create-case`, `notify-only`
- `TriggerMode` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for `fire-once`, `repeating`

### Need annotations

**GanglionDescriptor** — sealed interface, add:
```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = GanglionDescriptor.NaiveBayes.class, name = "naive-bayes"),
    @JsonSubTypes.Type(value = GanglionDescriptor.ExpressionRules.class, name = "expression-rules"),
    @JsonSubTypes.Type(value = GanglionDescriptor.SituationWatcher.class, name = "situation-watcher")
})
```

Consistent with ChainMode, TriggerAction, TriggerMode.

**SituationDefinition** — YAML keys differ from record parameter names in two places:
- `correlationKey` (YAML) → `correlationKeyExpression` (record): add `@JsonProperty("correlationKey")`
- `feedback` (YAML) → `feedbackConfig` (record): add `@JsonProperty("feedback")`

**FeedbackConfig** — record parameter names already match YAML keys (`noiseLabels`,
`confirmedLabels`, `suppressionCooldown`, `learningRate`, `retentionPeriod`, `tuningEnabled`,
`overSensitiveThreshold`, `underSensitiveThreshold`, `crossRefWindow`). Duration fields
handled by `JavaTimeModule`. No annotations needed.

**DetectionSignal** — enum used in GanglionDescriptor.ExpressionRules.Rule. YAML values
are uppercase (`DETECTED`, `WEAK`, `NOISE`, `ANTI`), matching enum constant names. No
annotations needed — Jackson handles this by default.

**SituationChangeEvent.ChangeType** — enum used as map key in
GanglionDescriptor.SituationWatcher.changeTypeMapping. Same uppercase convention. No
annotations needed.

### Nested types in GanglionDescriptor

- `NaiveBayes.Feature` — record with `ExpressionEvaluator expression`, `List<String> values`,
  `double[][] likelihoods`. `expression` field handled by the ExpressionEvaluator deserializer.
  `values` and `likelihoods` handled by Jackson natively.
- `NaiveBayes.SignalMapping` — plain record, all primitive/boxed fields. No annotations needed.
- `ExpressionRules.Rule` — has `ExpressionEvaluator when`, `DetectionSignal signal`,
  `double confidence`, `ExpressionEvaluator confidenceExpression`,
  `Map<String, ExpressionEvaluator> evidenceTemplates`. The `when` field is nullable
  (null for `otherwise` rules) — Jackson handles nullable record parameters. Add
  `@JsonProperty("otherwise")` handling via a `@JsonIgnore` or let Jackson skip unknown
  properties (FAIL_ON_UNKNOWN_PROPERTIES=false).

**otherwise handling:** The YAML for `otherwise` rules uses `otherwise: true` instead of `when`.
Jackson with FAIL_ON_UNKNOWN_PROPERTIES=false will ignore the `otherwise` key. The `when` field
will be null (absent from YAML), which is the correct representation for an otherwise rule.
Post-deserialization validation in `YamlSituationDefinitionProvider` enforces that `otherwise`
rules appear only as the last rule in the list, and that each rule has exactly one of `when`
or `otherwise`.

**SituationDefinition compact constructor:** The record's compact constructor validates
`eventTypes != null && !eventTypes.isEmpty()` and `chainMode != null`. Jackson populates
record parameters via the canonical constructor, so these validations fire naturally during
deserialization — invalid YAML produces `IllegalArgumentException` wrapped in
`MismatchedInputException`.

## 2. ExpressionEvaluator Deserializer (runtime/)

New class: `ExpressionEvaluatorDeserializer extends StdDeserializer<ExpressionEvaluator>`

Handles the ras YAML expression format:
```yaml
correlationKey:
  expression: ".subject"
  language: jq
```

Logic:
1. Read node as tree (`JsonNode`)
2. If null → return null
3. If object → read `expression` (string) and `language` (string)
4. Create appropriate evaluator:
   - `"jq"` → `new JQExpressionEvaluator(expression)`
   - `"mvel"` → `new MvelExpressionEvaluator(expression)`
   - other → throw with descriptive error

Returns platform `StringExpressionEvaluator` implementations directly. The
`SituationDefinitionRegistry.compileExpression()` already handles these via pattern matching
on `StringExpressionEvaluator`.

### Shared ObjectMapper Factory

New utility: `RasYamlMapper` (package-private in runtime/)

```java
static ObjectMapper create() {
    var mapper = new ObjectMapper(new YAMLFactory());
    mapper.registerModule(new JavaTimeModule());
    var module = new SimpleModule();
    module.addDeserializer(ExpressionEvaluator.class, new ExpressionEvaluatorDeserializer());
    mapper.registerModule(module);
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    return mapper;
}
```

Shared between `YamlSituationDefinitionProvider` and `SituationYamlRegistrar`.

## 3. Migrate YamlSituationDefinitionProvider to Jackson (runtime/)

### What changes

Replace `new Yaml().load(inputStream)` with `objectMapper.readTree(inputStream)`.

**Eliminated methods** (~500 lines of manual parsing):
- `parseSituation()` → replaced by `objectMapper.treeToValue(node, SituationDefinition.class)`
- `parseChainMode()` → handled by ChainMode Jackson annotations
- `parseTriggerAction()` → handled by TriggerAction Jackson annotations
- `parseTriggerMode()` → handled by TriggerMode Jackson annotations
- `parseFeedbackConfig()` → handled by FeedbackConfig record + JavaTimeModule
- `parseGanglia()` + `parseNaiveBayesGanglion()` + `parseExpressionRulesGanglion()` +
  `parseSituationWatcherGanglion()` → replaced by
  `objectMapper.treeToValue(node, GanglionDescriptor.class)` with Jackson polymorphic dispatch
- `parseNaiveBayesFeature()` + `parseSignalMapping()` → handled by nested record deserialization
- `parseExpressionEntry()` + `parseExpressionEvaluator()` → handled by ExpressionEvaluatorDeserializer
- `parseEvidenceTemplates()` + `parseOutcomeEvidenceTemplates()` + `parseDynamicCaseData()` →
  handled by `Map<String, ExpressionEvaluator>` deserialization
- `requireString()`, `requireList()`, `requireNumber()` → replaced by Jackson's own
  validation (missing required fields throw `MismatchedInputException`)

### What stays

**Template system** — parameter substitution and deep-merge operate on the YAML tree before
typed deserialization. This logic migrates from `Map<String, Object>` operations to `JsonNode`
operations:

- `parseTemplates()` → reads template entries from `JsonNode` array, stores as
  `Map<String, TemplateDefinition>` where the definition body is a `JsonNode`
- `substituteParams()` → traverses `JsonNode` tree, replaces `${param}` text nodes
  with parameter values (whole-value substitution preserves types via `JsonNode` factory)
- `checkUnresolved()` → traverses `JsonNode` tree looking for remaining `${param}` placeholders
- `deepMerge()` → merges `ObjectNode` trees (Jackson provides `ObjectNode.setAll()`)
- `resolveTemplate()` → resolves template parameters and deep-merges consumer overrides,
  returns resolved `JsonNode`
- `buildResolvedParams()` → extracts parameter values from situation `JsonNode`

After template resolution, the resolved `JsonNode` is deserialized:
```java
objectMapper.treeToValue(resolvedNode, SituationDefinition.class)
```

**Built-in template loading** (`loadBuiltInTemplates()`) — same logic, Jackson tree API.

**Validation** — Jackson's record deserialization provides structural validation
(missing required fields, type mismatches). Domain validation (e.g., "outcomes must have
at least 2 entries") moves to record compact constructors where it belongs, or stays as
post-deserialization checks in the provider.

### Domain validation that Jackson can't handle

Some validation in the current manual parsing is business rules, not structural:
- `outcomes.size() >= 2` for NaiveBayes — stays as post-deserialization check
- `rules must not be empty` for ExpressionRules — stays as post-deserialization check
- `otherwise must be last rule` — stays as post-deserialization check
- `confidence 0.0-1.0 range` — stays as post-deserialization check
- `handledEventTypes must not be empty` — can move to record compact constructor

These checks remain in `YamlSituationDefinitionProvider` as validation after deserialization,
or migrate to compact constructors on the relevant records/descriptors.

### SnakeYAML dependency removal

After migration, remove `org.yaml.snakeyaml` import from `YamlSituationDefinitionProvider`.
SnakeYAML is a transitive dependency of `jackson-dataformat-yaml` (Jackson uses it internally),
so no POM change needed — just no direct usage.

## 4. New SituationYamlRegistrar (runtime/)

New class: `SituationYamlRegistrar implements ClasspathYamlRegistrar`

```java
@ApplicationScoped
public class SituationYamlRegistrar implements ClasspathYamlRegistrar {

    private static final ObjectMapper yamlMapper = RasYamlMapper.create();

    @Inject SituationRegistrar situationRegistrar;

    @Override public String sectionKey() { return "situations"; }
    @Override public Set<String> classpathPaths() { return Set.of("situations"); }
    @Override public int priority() { return 300; }

    @Override
    public void register(String resourcePath, InputStream content) {
        SituationDefinition definition = yamlMapper.readValue(content, SituationDefinition.class);
        situationRegistrar.register(new SituationRegistration(definition));
    }
}
```

No `isResolvable()` check — the registrar is only on the classpath when ras is deployed,
so `SituationRegistrar` is always available.

### Dependency addition

Add to runtime `pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-bootstrap</artifactId>
</dependency>
```

`casehub-platform-bootstrap` is already in the parent BOM (`dependencyManagement`).

## Testing Strategy

### Unit tests

- **ExpressionEvaluatorDeserializer** — deserialize `{expression, language}` for jq, mvel;
  null handling; unknown language error
- **SituationDefinition Jackson round-trip** — serialize/deserialize through all ChainMode
  variants, TriggerAction variants, TriggerMode variants, FeedbackConfig, expression fields
- **GanglionDescriptor Jackson round-trip** — all three variants (NaiveBayes, ExpressionRules,
  SituationWatcher) with nested types
- **YamlSituationDefinitionProvider** — existing tests continue to pass with Jackson backend;
  template resolution tests continue to work
- **SituationYamlRegistrar** — unit test with mock SituationRegistrar, verify registration
  from individual YAML files

### Integration tests

- Quarkus dev-services test loading situations from `situations/` classpath directory via
  ClasspathBootstrap → SituationYamlRegistrar → SituationDefinitionRegistry

## Ordering

1. Jackson annotations on domain types (api/) — no dependencies
2. ExpressionEvaluatorDeserializer + RasYamlMapper (runtime/) — depends on step 1
3. Migrate YamlSituationDefinitionProvider (runtime/) — depends on step 2
4. SituationYamlRegistrar (runtime/) — depends on step 2, independent of step 3

Steps 3 and 4 can proceed in parallel after step 2.

## References

- `engine/runtime/.../bootstrap/SituationYamlRegistrar.java` — engine's current registrar (to be removed)
- `engine/api/.../ExpressionEvaluatorDeserializer.java` — engine's expression deserializer
- `ras/runtime/.../YamlSituationDefinitionProvider.java` — ras's current SnakeYAML provider
- `ras/runtime/.../SituationDefinitionRegistry.java` — ras's situation registry with dynamic registration
- `ras/api/.../SituationRegistrar.java` — registration SPI
- `ras/api/.../SituationDefinition.java` — situation definition record
- `ras/api/.../GanglionDescriptor.java` — ganglion descriptor sealed interface
- `platform/platform-api/.../ExpressionEvaluator.java` — platform expression marker interface
- `platform/platform-api/.../JQExpressionEvaluator.java` — platform JQ evaluator record
- `parent/casehub-platform-bootstrap/.../ClasspathYamlRegistrar.java` — bootstrap SPI
- `parent/casehub-platform-bootstrap/.../ClasspathBootstrap.java` — bootstrap scanner
