## D1: Core architecture

**Choice:** Event-driven bridge — SituationChangeEvent → CloudEvent adapter
**Alternatives:**
- Poll-driven via SituationSource — introduces latency, breaks event-driven architecture, needs parallel evaluation model
- Native dual-input engine — massive duplication across every component, two definition types
**Rationale:** No changes to the existing evaluation model. Meta-situations are regular SituationDefinitions whose eventTypes include bridged change types. All existing infrastructure (stores, policies, chain modes, metrics, feedback, YAML, replay) works unchanged. New code: bridge observer (CDI async observer converting SituationChangeEvent → CloudEvent), cycle validator (directed-graph analysis at registration time), and GanglionDescriptor.SituationWatcher variant (YAML-configured ganglion for extracting situation-change data from bridged CloudEvents).
**Trade-offs:**
- Bridged CloudEvents carry situation-change summary data (situationId, correlationKey, tenancyId, changeType, firstSignal, lastSignal, triggerCount, detectionCount) — not the full SituationContext with all TimestampedDetections — keeping event payloads lightweight. Extensions carry situationid, correlationkey, changetype for expression-based extraction. Ganglia needing richer child-situation data can query SituationStore.
- The bridge creates an async CDI cascade (SituationChangeEvent → bridge → CloudEvent → RasEngine → SituationEvaluator). During degradation scenarios, meta-situation evaluation competes with primary situation processing for the CDI managed executor thread pool. Mitigated by configuring a separate executor for bridge-originated CloudEvents via CDI NotificationOptions, isolating meta-situation load from primary evaluation.
- Cycle detection at registration time: build a directed graph where nodes are situation definitions and edges represent "situation S₁ can trigger → produces bridged change type T → situation S₂ subscribes to T." Standard DFS cycle detection rejects definitions that create loops. EventFilters are treated conservatively (transparent) — may produce false-positive cycle rejections but never false negatives. This is mandatory validation, not an optional check.
**Sources:** RasEngine.java:29 (onCloudEvent routing), SituationDefinitionRegistry.java:120 (buildSnapshot/findByEventType), SituationEvaluator.java:295 (SituationChangeEvent firing), GE-20260730-d54a8f (CDI fireAsync exception propagation)
**Exploration:** quick
**Status:** revised — R2: tightened "zero new abstractions" claim to accurately reflect new code artifacts; specified bridged CloudEvent data shape (summary, not full context); added thread pool isolation trade-off; specified cycle detection algorithm

## D2: Temporal absence mechanism

**Choice:** `SituationDefinition.deadline: Duration` — optional field on the definition, checked by a scheduled DeadlineCheckJob
**Alternatives:**
- ChainMode.Deadline variant — breaks ChainMode's pure-function-of-detections model, non-deterministic in replay (wall-clock dependency), can't compose with other chain modes, violates referencedGanglia() contract (returns empty set), and DeadlineCheckJob invocation conflicts with SituationEvaluator's event-driven design
- Watchdog ganglion with GanglionStateStore — blurs ganglion/policy boundary, timer logic belongs in lifecycle not detection
- Synthetic heartbeat events — pollutes event stream, system-wide heartbeat rate not per-situation
**Rationale:** Deadline is a temporal lifecycle property, not a detection pattern. Every existing ChainMode variant (And, Or, Threshold, Sequence, Count, Streak, Rate) is a pure function of SituationContext.detections() — given the same context, they always return the same result. This is what makes SituationReplayRunner deterministic. A deadline based on `firstSignal + duration < Instant.now()` breaks this invariant. Placing deadline alongside correlationWindow on SituationDefinition maintains ChainMode purity and enables natural composition — any chain mode can have an optional deadline layered on top.
**Semantics:** After `firstSignal + deadline` elapses, if the situation hasn't already triggered or been resolved/discarded, the DeadlineCheckJob forces a trigger regardless of chain mode satisfaction. This serves the escalation use case: child-triggered event creates the meta-situation context, and if no resolution arrives within the SLA window, the deadline forces escalation.
**DeadlineCheckJob design:**
- Queries SituationStore for active situations where the definition has a non-null deadline
- For each, checks `context.firstSignal() + definition.deadline() < Instant.now()`
- When expired: calls a dedicated `SituationEvaluator.triggerByDeadline(situationId, correlationKey, tenancyId)` entry point that handles trigger mechanics (claim, case creation, change event, ganglion close) without requiring a CloudEvent input
- Check interval: configurable via `ras.deadline.check-interval` (default PT10S). For SLA monitoring with sub-minute deadlines, intervals of 5-15 seconds balance latency against database load. The event-driven path also checks `firstSignal + deadline < eventTime` for deterministic behaviour when events continue arriving.
- NOT analogous to SituationExpiryJob (which only calls store.removeExpired/removeTriggeredBefore and never evaluates policy). DeadlineCheckJob must evaluate and trigger — a fundamentally different operation.
**Depends on:** D1 (bridge architecture — deadline composes with bridged situation events)
**Sources:** ChainMode.java:20 (sealed interface — all variants are pure functions of detections), DefaultRasTriggerPolicy.java:22 (evaluate pattern — switch on ChainMode, returns boolean from detections only), SituationReplayRunner.java:31 (deterministic replay depends on ChainMode purity), SituationDefinition.java:13 (correlationWindow precedent for temporal definition fields)
**Exploration:** quick
**Status:** revised — R2: moved from ChainMode variant to SituationDefinition field. ChainMode.Deadline broke the pure-function-of-detections invariant, replay determinism, referencedGanglia() contract, and couldn't compose with other chain modes. Deadline as a definition field (like correlationWindow) preserves all existing invariants.

## D3: Bridged CloudEvent shape and default correlation

**Choice:** subject = situationId — default correlation groups meta-situations by child situation identity
**Alternatives:**
- subject = tenancyId — default matches tenant-level aggregation, but violates CloudEvent spec semantics (subject should identify what changed, not the scope), creates redundancy with the tenancyid extension, and requires correlationKeyExpression for the most operationally critical use case (escalation)
- subject = composite situationId/correlationKey — maximally specific but almost always needs override
**Rationale:** The CloudEvent spec defines subject as "identifies the subject of the event in the context of the event producer." For a bridged SituationChangeEvent, the event producer is the RAS bridge and the subject is the situation that changed — not the tenant. Setting subject = situationId is semantically correct, avoids redundancy with the tenancyid extension (which already carries tenant identity), and provides the right default for escalation patterns (per-situation-instance tracking). Tenant-level aggregation (degradation, compliance use cases) requires `correlationKeyExpression: extensions.tenancyid` — a one-liner using the well-supported expression system.
**Trade-offs:** Degradation and compliance-drift use cases (tenant-level grouping) require correlationKeyExpression. Per-situation escalation works without configuration. The trade-off favours semantic correctness and the operationally critical use case over zero-config convenience for aggregation patterns.
**Depends on:** D1 (bridge architecture)
**Sources:** DefaultCorrelationKeyExtractor.java (subject → correlationKey mapping), CloudEvent spec (subject semantics), SituationChangeEvent.java (fields carried: tenancyId, situationId, correlationKey, changeType, context)
**Exploration:** quick
**Status:** revised — R2: changed from subject = tenancyId to subject = situationId for CloudEvent spec compliance, semantic correctness, and better default ergonomics for the escalation use case

## D4: Bridged event type naming convention

**Choice:** Per-ChangeType types — `ras.situation.triggered`, `ras.situation.resolved`, `ras.situation.discarded`, `ras.situation.suppressed`, `ras.situation.dismissed`
**Alternatives:**
- One type for all changes (`ras.situation.changed`) — simpler but makes selective subscription impossible; meta-situations can't observe only TRIGGERED events without eventFilter expressions; cycle validator can't distinguish change-type flows
- Per-situation types (`ras.situation.{situationId}.triggered`) — maximally specific but creates unbounded type namespace; cycle validator graph grows with every registered situation; adding/removing situations changes the event type contract
**Rationale:** Per-ChangeType types (5 total, matching SituationChangeEvent.ChangeType enum values) provide the right granularity. Meta-situations declare which transitions they observe via eventTypes (e.g., `[ras.situation.triggered]` to watch for child triggers). Selective subscription by change type covers the primary differentiation need — meta-situations rarely need to observe ALL change types. Per-situation filtering uses eventFilter expressions (`extensions.situationid == 'child-X'`), keeping the type namespace bounded and stable. Cycle validator operates on a finite graph of at most N×5 edges (N situations × 5 change types), tractable for any realistic deployment.
**Trade-offs:** A meta-situation that needs to observe only a specific child situation's triggers must combine eventTypes with an eventFilter. This is a one-liner expression, not a configuration burden.
**Depends on:** D1 (bridge architecture — type naming constrains eventType declarations and cycle detection)
**Sources:** SituationChangeEvent.java:14 (ChangeType enum — TRIGGERED, RESOLVED, DISCARDED, SUPPRESSED, DISMISSED), SituationDefinitionRegistry.java:171 (findByEventType routing)
**Exploration:** surfaced by review
**Status:** captured

## D5: Meta-situation nesting depth

**Choice:** Unlimited — meta-situations can observe other meta-situations to arbitrary depth
**Alternatives:**
- Depth-1 only (meta-situations can observe primary situations but not other meta-situations) — simpler cycle detection but prevents escalation hierarchies, cascading alerts, and multi-level aggregation patterns
- Configurable depth limit — complexity without clear benefit over cycle-validator-based safety
**Rationale:** D1's bridge design naturally enables unlimited nesting: bridged CloudEvents re-enter the generic RasEngine.onCloudEvent() pipeline, so any situation that declares a bridged event type in its eventTypes will observe it, regardless of whether the source is a primary or meta-situation. This is intentional, not accidental. Unlimited nesting enables powerful compositions — L1 meta-situation detects child triggers, L2 meta-meta-situation detects L1 escalation failures, etc. Safety is guaranteed by D1's mandatory cycle validator at registration time, which rejects definitions that would create infinite loops. The cycle graph is finite (bounded by registered definitions × bridged change types per D4) and traversable in O(V+E).
**Trade-offs:** Arbitrary depth increases the cycle validator's graph size and makes the system's behaviour harder to reason about for operators. Mitigated by: (1) cycle validator catches loops at registration, not at runtime; (2) per-ChangeType event types (D4) keep the graph tractable; (3) operational tooling (metrics, logging) tracks nesting depth per evaluation chain.
**Depends on:** D1 (bridge architecture), D4 (event type naming bounds the cycle graph)
**Sources:** RasEngine.java:29 (onCloudEvent routes all CloudEvents generically — no distinction between primary and bridged), SituationDefinitionRegistry.java:171 (findByEventType doesn't distinguish event origin)
**Exploration:** surfaced by review
**Status:** captured
