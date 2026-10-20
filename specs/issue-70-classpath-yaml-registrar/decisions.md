## D1: YAML parser — Jackson everywhere

**Choice:** Jackson (YAMLFactory) for all YAML parsing in ras
**Alternatives:**
- SnakeYAML + existing manual parsing — already tested and handles all types, but creates a second parsing paradigm that diverges from engine and other modules
- Jackson with SnakeYAML fallback — most complex, two code paths to maintain
**Rationale:** Aligns ras with the rest of the platform. Jackson annotations on domain types (ChainMode, TriggerAction, TriggerMode) already exist. Eliminates ~500 lines of manual map parsing in YamlSituationDefinitionProvider.
**Trade-offs:** Template resolution (${param} substitution, deep-merge) must be reimplemented on JsonNode instead of Map<String, Object> — mechanically similar but touches different API.
**Sources:** Engine SituationYamlRegistrar (engine/runtime/.../bootstrap/SituationYamlRegistrar.java), existing YamlSituationDefinitionProvider (ras/runtime/.../YamlSituationDefinitionProvider.java)
**Exploration:** quick
**Status:** captured

## D2: ExpressionEvaluator deserialization — platform types, ras format

**Choice:** New Jackson StdDeserializer in ras runtime/ that reads {expression, language} and creates platform StringExpressionEvaluator impls (JQExpressionEvaluator, MvelExpressionEvaluator)
**Alternatives:**
- Reuse engine's ExpressionEvaluatorDeserializer — depends on engine types (ExpressionEngineRegistry, JQExpressionEvaluator from engine-api), defeats the purpose of decoupling
- Produce generic StringExpressionEvaluator with deferred compilation — adds a layer of indirection; platform JQ/MVEL evaluators are concrete and sufficient
**Rationale:** Ras already uses platform expression types. No engine dependency needed. The registry's compileExpression() already handles StringExpressionEvaluator via pattern matching.
**Trade-offs:** Engine's deserializer also supports bare-string format (defaults to JQ) and {lang: expr} shorthand — we only support the explicit {expression, language} format used by ras YAML.
**Sources:** Engine ExpressionEvaluatorDeserializer (engine/api/.../ExpressionEvaluatorDeserializer.java), platform StringExpressionEvaluator, JQExpressionEvaluator, MvelExpressionEvaluator
**Exploration:** quick
**Status:** captured
