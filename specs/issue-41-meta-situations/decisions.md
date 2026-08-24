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

## D2: Temporal absence mechanism

**Choice:** ChainMode.Deadline — new sealed interface variant with scheduled DeadlineCheckJob
**Alternatives:**
- Watchdog ganglion with GanglionStateStore — blurs ganglion/policy boundary, timer logic belongs in policy not detection
- Synthetic heartbeat events — pollutes event stream, system-wide heartbeat rate not per-situation
**Rationale:** Deadline is a property of *when to trigger*, which is exactly what ChainMode models. A scheduled job (same pattern as SituationExpiryJob) checks for expired deadlines. Timer state lives in SituationContext (firstSignal + detections), so survives restarts without additional storage.
**Trade-offs:** New sealed interface variant requires YAML support, Jackson serde, and policy evaluation changes. Scheduled job introduces bounded latency (configurable check interval).
**Depends on:** D1 (bridge architecture — Deadline composes with bridged situation events)
**Sources:** ChainMode.java:20 (sealed interface), DefaultRasTriggerPolicy.java:23 (evaluate pattern), SituationExpiryJob (scheduled job pattern)
**Exploration:** quick
**Status:** captured
