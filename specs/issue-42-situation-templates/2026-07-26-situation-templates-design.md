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
- `DesiredStateSituationDefinitionProvider` builds 3 programmatic definitions with
  near-identical structure, differing only in event types, ganglion IDs, chain mode
  params, and trigger config.
- `JpaRuntimeSituationDefinitionProvider` in iot duplicated the entire YAML parser
  from `YamlSituationDefinitionProvider` to get multi-resource loading + JPA overlay.

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

### Identity Fields

`situationId` and `eventTypes` are always consumer-provided. They are never part of
the template definition — they are what make each instance unique.

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
4. If value is a Map: recurse into each value.
5. Otherwise: leave as-is.

The distinction between whole-value substitution and substring interpolation is how
typed values flow through without coercion.

After substitution, any remaining `${...}` placeholder in the resolved tree is an
error — catches typos in template definitions.

### Validation

- Missing required parameter: `IllegalArgumentException` naming the template ID and
  parameter.
- Unknown parameter (consumer provides a key not declared in the template): warning
  log, ignored.

## Resolution Algorithm

When `YamlSituationDefinitionProvider` encounters a situation with `fromTemplate:`:

1. **Look up template.** Find by `id` in the parsed templates map.
   `IllegalArgumentException` if not found.
2. **Build parameter values.** Start with template defaults, overlay with consumer's
   `parameters:` map. Validate required params are present.
3. **Deep-copy and substitute.** Deep-copy the template's `definition:` Map tree.
   Walk the copy, replacing `${param}` placeholders with resolved values.
4. **Merge consumer overrides.** Any field the consumer provides alongside
   `fromTemplate:` (other than `fromTemplate`, `situationId`, `eventTypes`,
   `parameters`) is deep-merged into the resolved Map. Consumer fields win.
5. **Inject identity.** Set `situationId` and `eventTypes` from consumer values.
6. **Parse.** Pass the fully resolved Map to the existing `parseSituation()`.
   Identical code path to hand-written definitions from this point.

### Deep Merge Semantics

- Maps: recursive merge, consumer values win on key conflict.
- Lists: consumer replaces entirely (lists like `ganglia: [a, b]` are atomic).
- Scalars: consumer replaces.

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

When a situation references a template with bundled ganglia, the resolved ganglia are
added to the provider's `ganglionDescriptors()` return list. The existing registry
Phase 1 handles construction. Duplicate ganglion ID collision is caught by the
registry's existing check.

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
3. Consumer templates with the same `id` override built-in ones.

**Initial templates:**

| Template ID | Chain Mode | Parameters (required) | Parameters (defaulted) |
|---|---|---|---|
| `streak-breach` | Streak | ganglionId, caseNamespace, caseName | requiredCount=3, window=PT10M, caseVersion="1" |
| `threshold-crossing` | Threshold | ganglia, caseNamespace, caseName | minConfidence=0.8, window=PT5M, caseVersion="1" |
| `count-accumulation` | Count | ganglionId, caseNamespace, caseName | requiredCount=5, window=PT10M, caseVersion="1" |
| `rate-breach` | Rate | ganglia, caseNamespace, caseName | minRate=0.6, windowSize=10, window=PT30M, caseVersion="1" |

All default to `triggerMode: fire-once` and `triggerAction: create-case`.

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
| `YamlSituationDefinitionProvider` | Add `parseTemplates()`, template resolution, parameter substitution. Load built-in templates resource. Parse order: built-in templates → consumer templates → ganglia → situations. |
| `META-INF/ras-situation-templates.yaml` | New — the built-in template library (4 templates). |

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

**End-to-end:**
- Template-instantiated situation registered in `SituationDefinitionRegistry`.
- Detection works identically to a hand-written definition.
- Each built-in template instantiated and validated.

**Contract invariant:** For any template instantiation, there exists an equivalent
hand-written YAML situation definition that produces the identical
`SituationRegistration`. Tests verify this equivalence explicitly.
