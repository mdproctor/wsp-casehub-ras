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

### Already annotated (polymorphic dispatch)
- `ChainMode` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for all 7 variants
- `TriggerAction` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for `create-case`, `notify-only`
- `TriggerMode` — `@JsonTypeInfo(property="type")` + `@JsonSubTypes` for `fire-once`, `repeating`

### Additional annotations needed on already-annotated types

**ChainMode.And** — YAML key is `ganglia`, but record parameter is `requiredGanglia`:
```java
record And(@JsonProperty("ganglia") Set<String> requiredGanglia) implements ChainMode {
```

**ChainMode.Sequence** — YAML key is `ganglia`, but record parameter is `orderedGanglia`:
```java
record Sequence(@JsonProperty("ganglia") List<String> orderedGanglia) implements ChainMode {
```

Without these, Jackson looks for `requiredGanglia`/`orderedGanglia` keys, finds neither, and
the compact constructors throw on null/empty.

**TriggerAction.CreateCase** — The YAML format puts `caseNamespace`, `caseName`, `caseVersion`,
`baseCaseData` as flat sibling keys alongside `type`. But `CreateCase(CaseTriggerConfig config)`
wraps them in a single nested parameter. Jackson's polymorphic deserialization strips `type` and
tries to match remaining flat keys to `CreateCase` parameters — it finds no `config` key.

Fix: add a `@JsonCreator` static factory that takes the flat properties and constructs
`CaseTriggerConfig` internally:
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
                caseNamespace, caseName, caseVersion, baseCaseData));
    }
}
```

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

**FeedbackConfig** — record parameter names already match YAML keys. Duration fields
handled by `JavaTimeModule`. No annotations needed, but the compact constructor must
apply defaults for three optional fields that Jackson cannot default on its own.

Jackson always uses the canonical (all-parameter) constructor for records. When optional
YAML keys are absent, Jackson passes `null` for object types and `0.0` for `double`:
- `crossRefWindow` absent → `null` → compact constructor null-coalesces to `Duration.ofHours(1)`
- `overSensitiveThreshold` absent → `0.0` → compact constructor replaces with `0.5`
- `underSensitiveThreshold` absent → `0.0` → compact constructor replaces with `0.5`

The defaults must be applied BEFORE validation in the compact constructor:
```java
public FeedbackConfig {
    crossRefWindow = crossRefWindow != null ? crossRefWindow : Duration.ofHours(1);
    overSensitiveThreshold = overSensitiveThreshold == 0.0 ? 0.5 : overSensitiveThreshold;
    underSensitiveThreshold = underSensitiveThreshold == 0.0 ? 0.5 : underSensitiveThreshold;

    Objects.requireNonNull(noiseLabels, "noiseLabels");
    // ... remaining validation unchanged
}
```

Using `0.0` as the sentinel is safe because the validation already requires `> 0.0` — a user
who explicitly writes `0.0` gets the default instead, but `0.0` was never valid anyway.

**DetectionSignal** — enum used in GanglionDescriptor.ExpressionRules.Rule. YAML values
are uppercase (`DETECTED`, `WEAK`, `NOISE`, `ANTI`), matching enum constant names.

**SituationChangeEvent.ChangeType** — enum used as map key in
GanglionDescriptor.SituationWatcher.changeTypeMapping. Same uppercase convention.

**Case-insensitive parsing:** The existing manual parser calls `.toUpperCase()` before
`valueOf()` for both enums, establishing a case-insensitive contract. To preserve this,
`RasYamlMapper` configures `MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS` (see §2).
No per-enum annotations needed.

### Nested types in GanglionDescriptor

- `NaiveBayes.Feature` — record with `ExpressionEvaluator expression`, `List<String> values`,
  `double[][] likelihoods`. The YAML format puts `expression` and `language` as flat siblings
  alongside `values` and `likelihoods` — Jackson's per-field deserializer cannot see sibling keys,
  so the ExpressionEvaluator deserializer alone cannot construct the evaluator.

  Fix: add a `@JsonCreator` static factory that reads all four keys:
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
  This bypasses the ExpressionEvaluator deserializer for Feature (which is correct — the factory
  handles expression construction directly). All other ExpressionEvaluator fields (correlationKey,
  eventFilter, etc.) are nested objects and use the deserializer normally.
- `NaiveBayes.SignalMapping` — plain record, all primitive/boxed fields. No annotations needed.
- `ExpressionRules.Rule` — has `ExpressionEvaluator when`, `DetectionSignal signal`,
  `double confidence`, `ExpressionEvaluator confidenceExpression`,
  `Map<String, ExpressionEvaluator> evidenceTemplates`. The `when` field is nullable
  (null for `otherwise` rules) — Jackson handles nullable record parameters.

**otherwise handling:** The YAML for `otherwise` rules uses `otherwise: true` instead of `when`.
Add `boolean otherwise` as a record parameter so Jackson captures the key as data instead
of ignoring it:
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

This preserves the mutual-exclusion validation from the manual parser. If `@JsonIgnoreProperties`
were used instead, Jackson would discard `otherwise` and post-deserialization code would see
`when == null` for BOTH legitimate otherwise rules AND rules that accidentally omit both keys —
losing the explicit-intent signal.

The existing YAML convention `otherwise:` (null value, no `true`) produces `false` for a
`boolean` record parameter. Standardise on `otherwise: true` — this is a deliberate format
change that makes intent explicit and works naturally with Jackson record deserialization.
Existing YAML files need a mechanical update: `otherwise:` → `otherwise: true`.

No `@JsonIgnoreProperties` or `FAIL_ON_UNKNOWN_PROPERTIES=false` needed — `otherwise` is
a real field, and all other types retain full unknown-property detection.

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
2. If `null` (Java null) or `node.isNull()` (YAML explicit `null` → `NullNode`) → return null.
   Both checks required: `readValueAsTree()` returns Java null for absent values,
   `NullNode` for YAML `correlationKey: null`.
3. If text → `new JQExpressionEvaluator(node.textValue())`. JQ is the default language,
   consistent with the engine's `ExpressionEvaluatorDeserializer` convention.
4. If object → read `expression` (required) and `language` (required):
   - Missing `expression` key → throw: `"Expression object missing required 'expression' field"`
   - Missing `language` key → throw: `"Expression object missing required 'language' field"`
   - `"jq"` → `new JQExpressionEvaluator(expression)`
   - `"mvel"` → `new MvelExpressionEvaluator(expression)`
   - other → throw: `"Unknown expression language '<lang>'. Expected: jq, mvel"`
5. Any other node type (number, array, boolean) → throw:
   `"Expected expression as string or {expression, language} object, got <nodeType>"`

Text-node handling (step 3) is a defensive fallback — all current ras YAML expression fields
except `NaiveBayes.Feature` use the object form. Feature is handled by its own `@JsonCreator`
factory (see §1), so this deserializer is not invoked for Feature at all.

Returns platform `StringExpressionEvaluator` implementations directly. The
`SituationDefinitionRegistry.compileExpression()` already handles these via pattern matching
on `StringExpressionEvaluator`.

### Shared ObjectMapper Factory

New utility: `RasYamlMapper` (package-private in runtime/)

```java
class RasYamlMapper {
    private static final ObjectMapper INSTANCE = build();

    static ObjectMapper instance() { return INSTANCE; }

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

Singleton — Jackson `ObjectMapper` is thread-safe after configuration. A shared instance avoids
duplicate type resolution caches and serializer/deserializer lookup tables.

`ACCEPT_CASE_INSENSITIVE_ENUMS` preserves the case-insensitive contract established by the
manual parser's `.toUpperCase()` calls on `DetectionSignal` and `ChangeType` values.

No `FAIL_ON_UNKNOWN_PROPERTIES=false` — the `otherwise` key is a real `boolean` field on
`ExpressionRules.Rule` (see §1), so no special handling is needed. All types retain full
unknown-property detection: a typo like `corelationKey` or `chainode` produces a
deserialization error instead of being silently ignored.

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
- `parseNaiveBayesFeature()` → handled by `Feature.fromYaml()` `@JsonCreator` (see §1)
- `parseSignalMapping()` → handled by record deserialization
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
`jackson-dataformat-yaml` is a transitive dependency of `casehub-platform` (already in the POM),
and SnakeYAML is a transitive dependency of `jackson-dataformat-yaml` (Jackson uses it internally).
No POM change needed — just no direct SnakeYAML usage.

## 3.1. Relationship: YamlSituationDefinitionProvider vs SituationYamlRegistrar

These two mechanisms are **complementary** — both active in the same deployment:

| | YamlSituationDefinitionProvider | SituationYamlRegistrar |
|---|---|---|
| **Source** | Monolithic `META-INF/ras-situations.yaml` | Individual YAML files from `situations/` classpath dir |
| **Lifecycle** | CDI construction (Phase 3 of `SituationDefinitionRegistry`) | `StartupEvent` via `ClasspathBootstrap` (after CDI construction) |
| **Templates** | Full template resolution (parameter substitution, deep merge) | No template support — files are standalone definitions |
| **Ganglia** | Provides both ganglia and situations | Situations only — must reference pre-existing ganglia |

**Duplicate situationId:** If a situation ID appears in both the monolithic YAML and an
individual file, `SituationDefinitionRegistry.register()` throws `IllegalStateException`.
This is a configuration error, not a race condition — the provider always completes first.

**Template support:** Individual YAML files go through `objectMapper.readValue(content,
SituationDefinition.class)` — no template resolution. This is intentional: individual files
are self-contained situation definitions meant for modular deployment, not template consumers.

## 4. New SituationYamlRegistrar (runtime/)

New class: `SituationYamlRegistrar implements ClasspathYamlRegistrar`

```java
@ApplicationScoped
public class SituationYamlRegistrar implements ClasspathYamlRegistrar {

    private static final ObjectMapper yamlMapper = RasYamlMapper.instance();

    @Inject SituationRegistrar situationRegistrar;

    @Override public String sectionKey() { return "situations"; }
    @Override public Set<String> classpathPaths() { return Set.of("situations"); }
    @Override public int priority() { return 300; }

    @Override
    public void register(String resourcePath, InputStream content) {
        try {
            SituationDefinition definition = yamlMapper.readValue(content, SituationDefinition.class);
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

Error handling mirrors the engine's `SituationYamlRegistrar` pattern: catch and log per-file
rather than propagating. `ClasspathBootstrap.scanAndRegister()` iterates discovered files
without catching exceptions from `register()` — an unhandled exception from one file would
abort the entire scan loop.

No `isResolvable()` check — the registrar is only on the classpath when ras is deployed,
so `SituationRegistrar` is always available.

**Ganglia constraint:** Situations registered via this path can only reference ganglia that
are already registered through `SituationDefinitionProvider.ganglionDescriptors()` (Phase 1)
or CDI `Ganglion` beans (Phase 2). The dynamic `register()` method validates ganglia existence
but does not add new ganglia. Individual YAML files are situation-only — they do not define
ganglia descriptors.

### Dependency additions

Add to runtime `pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-bootstrap</artifactId>
</dependency>
```

**Prerequisite:** `casehub-platform-bootstrap` is not yet in the parent BOM on main — it
exists on a worktree branch (slot 216). If the BOM update has not landed when this spec is
implemented, add an explicit version `${version.io.casehub}` to the dependency declaration.

`jackson-dataformat-yaml` (provides `YAMLFactory` used by `RasYamlMapper`) is already a
transitive dependency of `casehub-platform` — no explicit POM entry needed. After migration,
SnakeYAML direct usage is removed but it remains on the classpath as a transitive dependency
of `jackson-dataformat-yaml` (Jackson uses it internally).

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
