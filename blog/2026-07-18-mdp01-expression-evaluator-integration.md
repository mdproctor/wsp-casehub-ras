---
layout: post
title: "ExpressionEvaluator integration — the YAML path gets real"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ras]
tags: [expressions, jq, mvel3, yaml, situation-definitions]
series: issue-46-expression-evaluator-integration
---

RAS situation definitions started declarative-only: you define event types, chain modes, trigger actions, and correlation windows in YAML. Anything beyond static configuration — extracting a correlation key from event data, filtering events by content, seeding case data from detection results — required dropping to Java.

The platform already had `ExpressionEvaluator` — JQ, MVEL3, and Lambda behind a common `compile() → eval()` contract. Wiring it into the YAML path was the obvious move. Three use cases made the cut for this round: correlation key extraction (`correlationKey: {expression, language}`), event filtering (`eventFilter: {expression, language}`), and dynamic case data (`dynamicCaseData: {key: {expression, language}}`). NaiveBayes feature extraction and evidence templates need YAML ganglion config to exist first — they're follow-on issues.

The central design question was where expression descriptors belong. `SituationDefinition` already carries operational config — `CaseTriggerConfig` with `baseCaseData`, `TriggerMode`, `eventBufferDelay`. It's a complete declarative configuration, not a pure declaration. Expressions fit there. The descriptors serialize with the definition, so dynamic registration via `SituationDefinitionRegistry` gets expression support for free.

Compilation lives in the registry, not the YAML provider. Any registration path — YAML, programmatic, dynamic — goes through `compileRegistration()` at registration time. String-based evaluators get compiled via `ExpressionEngineRegistry`. `LambdaExpression` (already compiled) passes through. Compiled results live on `SituationRegistration` in the volatile `RegistrySnapshot` — same atomic swap pattern the registry already uses for thread safety.

The JQ engine needed a bridge. `JQExpressionEngine` produces `CompiledExpression<JsonNode, List<JsonNode>>` regardless of context type — it doesn't support `Map` context. Rather than fixing the platform engine (separate issue), I built a `JqMapAdapter` in the RAS registry that converts `Map<String, Object>` → `JsonNode` via `ObjectMapper.valueToTree()` and unwraps `List<JsonNode>` results into scalar values. This works but surfaced a gotcha: `OffsetDateTime` and `Instant` values in the context map cause `valueToTree()` to fail silently without Jackson's JSR-310 module. The fix was converting temporal types to ISO-8601 strings in the context builders — which is also the right answer for JQ expressions that want to work with timestamps as strings.

The design review caught several things I'd missed: `ExpressionEvaluator` has no `expression()` accessor (needed `StringExpressionEvaluator` sub-interface in platform-api), `CaseTriggerConfig` was the wrong home for `dynamicCaseData` (moved to `SituationDefinition`), and the compilation flow needed explicit handling for JQ's `JsonNode`-only context. The review also identified that correlation metadata (`situationId`, `correlationKey`) must override dynamic expressions in the merge order — otherwise a deployer could accidentally corrupt system-generated values.

The three-tier model holds: Java DSL for full power, YAML for declarative definitions, YAML + expressions for the dynamic middle ground. Deployers choose the expression language per field — JQ for ops people, MVEL for Java developers.
