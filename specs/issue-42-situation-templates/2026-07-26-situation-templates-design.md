# Situation Templates — Reusable Parameterised Situation Definitions

**Issue:** casehubio/casehub-ras#42
**Date:** 2026-07-26
**Status:** Design

## Problem

Consuming apps build YAML situation definitions from scratch. Common detection patterns
(SLA breach, anomaly spike, compliance drift, threshold crossing) are reimplemented per
app with minor variations. This increases adoption cost and introduces inconsistency —
each app makes slightly different chain mode choices for the same conceptual pattern.

Evidence from the codebase:
- `JpaRuntimeSituationDefinitionProvider` in iot duplicated the YAML parser from
  `YamlSituationDefinitionProvider` with schema divergences (`triggerConfig` vs
  `triggerAction`, legacy `requiredGanglia` field name, missing `streak`/`rate`
  chain modes). Templates don't solve this parser duplication — that requires
  extracting shared parsing logic independently (see casehubio/casehub-ras#55).
- The issue's own pattern list (SLA breach, anomaly spike, heartbeat missing,
  compliance drift, threshold crossing) represents real adoption friction: each
  consuming app currently hand-assembles the same chain mode configurations with
  minor variations in thresholds, ganglion IDs, and trigger config.

## Approach

YAML parameter substitution in `YamlSituationDefinitionProvider`. Templates are
situation definition patterns with `${param}` placeholders. Consumers instantiate
via `fromTemplate:` + `parameters:`. Resolution happens at parse time — the registry
and API layer never see templates, only fully resolved `SituationRegistration` objects.

Alternatives considered:
- **Java template API in api/**: type-safe but forces programmatic Java, doesn't help
  YAML-only consumers (the primary audience).
- **Deep-merge overlay (no parameters)**: simpler implementation but no parameter
  documentation, validation, or required-parameter enforcement.

## YAML Format

### Template Definition

New `templates:` section in the YAML file, parsed before `ganglia:` and `situations:`:

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
```

### Template Instantiation

Situations use `fromTemplate:` instead of inline field declarations:

```yaml
situations:
  - fromTemplate: streak-breach
    situationId: desiredstate.repeated-failure
    eventTypes: [node.faulted, node.recovered]
    parameters:
      ganglionId: node-fault
      caseNamespace: desiredstate
      caseName: replan
```

### Valid Template Fields

All fields supported by `parseSituation()` are valid in a template's `definition:`
section: `chainMode`, `correlationWindow`, `eventBufferDelay`, `triggerAction`
(including `baseCaseData`), `triggerMode`, `eventFilter`, `correlationKey`, and
`dynamicCaseData`. Any of these may contain `${param}` placeholders. The built-in
templates use a subset; consumers may use any combination.

### Identity Fields

`situationId` and `eventTypes` are always consumer-provided. They are never part of
the template definition — they are what make each instance unique.

Identity fields are implicitly available as parameter values during substitution: if
a template declares a parameter with the same name as an identity field (e.g.,
`eventTypes`), and the consumer does not provide it explicitly in `parameters:`, the
identity field's value is used. This eliminates forced duplication when a template's
bundled ganglia need `handledEventTypes` to match the situation's `eventTypes`.

### Consumer Overrides

Any field the consumer provides alongside `fromTemplate:` (other than `fromTemplate`,
`situationId`, `eventTypes`, `parameters`) is deep-merged into the resolved template.
This allows adding fields not in the template or replacing template defaults:

```yaml
situations:
  - fromTemplate: streak-breach
    situationId: my-situation
    eventTypes: [my.event]
    parameters:
      ganglionId: my-ganglion
      caseNamespace: ops
      caseName: escalate
    eventFilter:
      expression: ".data.severity == \"HIGH\""
      language: jq
    triggerMode:
      type: repeating
      cooldown: PT5M
```

Overriding `chainMode` on a ganglion-bundling template may orphan the bundled ganglia
if the new chain mode references different ganglion IDs. The orphaned ganglia are
constructed by the registry but never referenced by any situation. This is harmless
but suggests the template may not be the right fit for the consumer's needs.

## Parameter System

### Declaration

Each parameter has a name (the map key) and either `required: true` or `default: <value>`:

```yaml
parameters:
  ganglionId: {required: true}          # must be provided
  requiredCount: {default: 3}           # optional, defaults to int 3
  window: {default: PT10M}             # optional, defaults to string "PT10M"
  minRate: {default: 0.6}              # optional, defaults to double 0.6
```

No explicit type declarations. SnakeYAML handles typing on both sides:
- Template side: `${param}` is always a string placeholder in the parsed Map tree.
- Consumer side: `requiredCount: 3` produces Integer, `minRate: 0.6` produces Double.
- Substitution replaces the placeholder string with the typed value.

### Substitution Algorithm

Recursive tree walk of the template definition Map:

1. If value is a String and the **entire** string is `${paramName}`: replace with the
   typed parameter value (preserves int/double/list types).
2. If value is a String containing `${paramName}` as a **substring**
   (e.g., `"prefix-${id}-suffix"`): string interpolation, result is always a String.
3. If value is a List: recurse into each element.
4. If value is a Map: recurse into each value. Map keys are not substituted.
5. Otherwise: leave as-is.

The distinction between whole-value substitution and substring interpolation is how
typed values flow through without coercion.

After substitution, any remaining `${...}` placeholder in the resolved tree is an
error — catches typos in template definitions. This check scans both Map values and
Map keys: a `${param}` placeholder in a key position is always an error (key
substitution is not supported).

### Validation

- Missing required parameter: `IllegalArgumentException` naming the template ID and
  parameter.
- Unknown parameter (consumer provides a key not declared in the template): warning
  log, ignored.

## Resolution Algorithm

When `YamlSituationDefinitionProvider` encounters a situation with `fromTemplate:`:

1. **Look up template.** Find by `id` in the parsed templates map.
   `IllegalArgumentException` if not found.
2. **Build parameter values.** Start with template defaults, then inject identity
   fields (`situationId`, `eventTypes`) as parameter values for any declared
   template parameter with a matching name (identity fields override defaults
   but not explicit consumer parameters), then overlay with consumer's
   `parameters:` map. Validate required params are present.
3. **Deep-copy and substitute.** Deep-copy the template's `definition:` Map tree
   (and `ganglia:` section if present). Walk the copy, replacing `${param}`
   placeholders with resolved values.
4. **Merge consumer overrides.** Any field the consumer provides alongside
   `fromTemplate:` (other than `fromTemplate`, `situationId`, `eventTypes`,
   `parameters`) is deep-merged into the resolved Map. Consumer fields win.
5. **Inject identity.** Set `situationId` and `eventTypes` from consumer values.
6. **Parse.** Pass the fully resolved Map to the existing `parseSituation()`.
   Identical code path to hand-written definitions from this point. Any
   exception from `parseSituation()` is wrapped with template context (see
   §Error Wrapping).

Template definitions cannot reference other templates. `fromTemplate:` is only valid
in the `situations:` section. A `fromTemplate` key in a template's `definition:` is
not processed — it passes through to `parseSituation()` as an unknown key and is
ignored, likely producing a missing `chainMode` error.

### Error Wrapping

When `parseSituation()` throws after template resolution, the error is wrapped with
template context: template ID, situation ID, and which phase failed (substitution,
merge, or parsing). Example:

```
Error resolving template 'streak-breach' for situation 'my-sit':
  chainMode required for situation 'my-sit'
```

This distinguishes errors caused by template definition problems from consumer
override problems.

### Deep Merge Semantics

- Maps: recursive merge, consumer values win on key conflict.
- Lists: consumer replaces entirely (lists like `ganglia: [a, b]` are atomic).
- Scalars: consumer replaces.

When a consumer overrides a type-discriminated map (e.g., `triggerAction`) with a
different `type`, deep merge may leave fields from the original type variant in the
resolved map. This is safe: `parseTriggerAction` and `parseChainMode` read only the
fields required for the resolved type and ignore unknown keys. Parsers must not
validate that no extra keys are present.

## Ganglion Bundling

Templates can optionally include a `ganglia:` section — parameterised ganglion
descriptors instantiated alongside the situation:

```yaml
templates:
  - id: anomaly-classification
    parameters:
      ganglionId: {required: true}
      eventTypes: {required: true}
      features: {required: true}
      detectedThreshold: {default: 0.75}
      weakThreshold: {default: 0.30}
      caseNamespace: {required: true}
      caseName: {required: true}
    ganglia:
      - ganglionId: ${ganglionId}
        type: naive-bayes
        handledEventTypes: ${eventTypes}
        outcomes: [NORMAL, ANOMALY]
        priors: [0.9, 0.1]
        features: ${features}
        signalMapping:
          targetOutcome: ANOMALY
          detectedThreshold: ${detectedThreshold}
          weakThreshold: ${weakThreshold}
    definition:
      chainMode:
        type: or
        ganglia: [${ganglionId}]
      triggerAction:
        type: create-case
        caseNamespace: ${caseNamespace}
        caseName: ${caseName}
        caseVersion: "1"
```

Bundled ganglia are resolved **per-instantiation**, not per-template-definition. Each
`fromTemplate:` situation that references a template with `ganglia:` produces its own
resolved ganglia. Two instantiations of the same template with different `ganglionId`
parameters produce two different ganglia. Resolution happens during situation parsing
(step 5 of the loading flow), not during template registration.

The resolved ganglia are added to the provider's `ganglionDescriptors()` return list
alongside explicitly declared ganglia from the top-level `ganglia:` section. The
existing registry Phase 1 handles construction. Duplicate ganglion ID collision is
caught by the registry's existing check.

The `ganglia:` section is optional. Templates without it are situation-only patterns —
the consumer must ensure referenced ganglia exist elsewhere.

Implementation note: ganglion bundling is additive. Situation-only templates are
implemented first; ganglion bundling uses the same parameter substitution mechanism
applied to an additional section.

## Built-in Template Library

Ship as `META-INF/ras-situation-templates.yaml` in runtime/src/main/resources/.
Loaded automatically before the consumer's resource.

**Loading order:**
1. Built-in templates from `META-INF/ras-situation-templates.yaml`.
2. Consumer templates from consumer's YAML.
3. A consumer template with the same `id` as a built-in **replaces it entirely**.
   To modify a single default, copy the full template definition.

**Initial templates:**

All default to `triggerMode: fire-once` and `triggerAction: create-case`.

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

**Versioning:** Templates are classpath resources versioned by the Maven artifact
version. Adding optional parameters with defaults is backward-compatible. Removing or
renaming parameters is a breaking change.

## Module Placement

**No changes to api/.** Templates are a YAML parse concern. `SituationDefinition`,
`SituationRegistration`, `SituationDefinitionProvider`, `GanglionDescriptor`,
`SituationDefinitionRegistry` — all unchanged.

**All changes in runtime/:**

| File | Change |
|---|---|
| `YamlSituationDefinitionProvider` | Add `parseTemplates()`, template resolution, parameter substitution. Two-file loading (see below). |
| `META-INF/ras-situation-templates.yaml` | New — the built-in template library (4 templates). |

### Two-File Loading Flow

The constructor changes from loading a single resource to loading two:

1. **Load built-in templates.** `Thread.currentThread().getContextClassLoader()
   .getResourceAsStream("META-INF/ras-situation-templates.yaml")`. Hardcoded path,
   not configurable. If absent (e.g., test classpath), the template registry starts
   empty — this is not an error.
2. **Load consumer resource.** From `ras.situations.yaml` config property (existing
   behavior). The consumer file may contain `templates:`, `ganglia:`, and
   `situations:` sections.
3. **Merge templates.** Templates from step 1 populate a `Map<String, SituationTemplate>`
   by ID. Templates from step 2's `templates:` section are added to the same map —
   a consumer template with the same ID as a built-in replaces it entirely.
4. **Parse ganglia.** Consumer's `ganglia:` section parsed (existing behavior).
5. **Parse situations.** Consumer's `situations:` section parsed. Entries with
   `fromTemplate:` are resolved against the merged template map. Bundled ganglia
   from template instantiations are accumulated and added to the provider's
   `ganglionDescriptors()` return list alongside explicitly declared ganglia.

**Internal types** (package-private):

| Type | Purpose |
|---|---|
| `SituationTemplate` | Record: `id`, `description`, `parameters` (Map), `definition` (Map), optional `ganglia` (List) |
| `ParameterDef` | Record: `required` (boolean), `defaultValue` (Object, nullable) |

These are parse-time types, never exposed in the API.

## Testing Strategy

All tests in `YamlSituationDefinitionProviderTest`.

**Parameter substitution:**
- Whole-value substitution preserves types (int, double, string, list).
- Substring interpolation produces string.
- Nested substitution in Maps and Lists.
- Unresolved `${param}` after substitution is an error.

**Validation:**
- Missing required parameter → `IllegalArgumentException`.
- Unknown parameter → warning logged, no error.
- Template not found → `IllegalArgumentException`.
- Duplicate template ID within same file → last wins (same as ganglia).
- Consumer template overrides built-in template with same ID (consumer YAML
  is parsed after built-in, so last-parsed wins naturally).

**Template instantiation:**
- Simple template resolves to identical `SituationRegistration` as hand-written equivalent.
- Template with all defaults — consumer provides only required params.
- Template with consumer overrides — override fields replace template values.
- Deep merge: nested Map override replaces single key, not entire subtree.
- Deep merge: List override replaces entirely.

**Ganglion bundling:**
- Template with `ganglia:` → ganglion descriptors in `ganglionDescriptors()`.
- Bundled ganglion ID collision → registry catches duplicate.
- Template without `ganglia:` → no ganglion descriptors emitted.

**Identity field injection:**
- Identity field `eventTypes` auto-satisfies a `required: true` template parameter.
- Explicit consumer parameter overrides identity field value.
- Parameter with no matching identity field is unaffected.

**Two-file loading:**
- Built-in templates loaded before consumer templates.
- Consumer template with same ID replaces built-in entirely.
- Missing built-in resource → empty template registry (not an error).
- Consumer YAML with `templates:` + `ganglia:` + `situations:` all present.

**Error wrapping:**
- `parseSituation()` exception wrapped with template ID and situation ID.
- Substitution-phase error (unresolved placeholder) includes template context.
- `${param}` in a Map key position → error (key substitution not supported).

**End-to-end:**
- Template-instantiated situation registered in `SituationDefinitionRegistry`.
- Detection works identically to a hand-written definition.
- Each built-in template instantiated and validated.

**Contract invariant:** For any template instantiation, there exists an equivalent
hand-written YAML situation definition that produces the identical
`SituationRegistration`. Tests verify this equivalence explicitly.
