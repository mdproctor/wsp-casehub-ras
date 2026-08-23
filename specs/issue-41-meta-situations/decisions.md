## D1: Core architecture

**Choice:** Event-driven bridge — SituationChangeEvent → CloudEvent adapter
**Alternatives:**
- Poll-driven via SituationSource — introduces latency, breaks event-driven architecture, needs parallel evaluation model
- Native dual-input engine — massive duplication across every component, two definition types
**Rationale:** Zero new abstractions. Meta-situations are regular SituationDefinitions whose eventTypes include bridged types. All existing infrastructure (stores, policies, chain modes, metrics, feedback, YAML, replay) works unchanged. The only new code is the bridge observer, cycle validator, GanglionDescriptor variant, and Deadline chain mode.
**Trade-offs:** Bridged CloudEvents carry situation data in a non-native shape — ganglia need to understand the bridged format. A GanglionDescriptor.SituationWatcher variant handles this for YAML-configured ganglia.
**Sources:** RasEngine.java:29 (onCloudEvent routing), SituationDefinitionRegistry.java:120 (buildSnapshot/findByEventType), SituationEvaluator.java:295 (SituationChangeEvent firing), GE-20260730-d54a8f (CDI fireAsync exception propagation)
**Exploration:** quick
**Status:** captured
