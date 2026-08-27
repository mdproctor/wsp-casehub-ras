# Design Journal — issue-41-meta-situations

## 2026-08-27 — Design complete, implementation 5/9 tasks done

### Design phase
- 5 decisions captured: bridge architecture (D1), temporal absence as SituationDefinition.deadline (D2), subject=situationId (D3), per-ChangeType event types (D4), unlimited nesting with cycle detection (D5)
- Standard decision review revised D1-D3, surfaced D4-D5. Key revision: D2 moved from ChainMode.Deadline to SituationDefinition.deadline — ChainMode purity preserved for replay determinism
- Light spec review surfaced 12 findings (5 HIGH): cycle detection self-edges, CloudEventExpressionContext missing extensions, handledEventTypes NOISE pollution, SLA breach example broken, deadline replay gap. All addressed.
- Known gap documented: cancel-on-resolution mechanism for deadline situations

### Implementation progress
- **Batch 1 done:** SituationDefinition.deadline field (12th component), GanglionDescriptor.SituationWatcher sealed variant, SituationStore.findActiveBySituationId default method, CloudEventExpressionContext exposing all extensions
- **Batch 2 done:** SituationChangeEventBridge (CDI observer, fire-and-forget, try-catch isolation), SituationWatcherGanglion (changeType mapping, automatic evidence), registry constructGanglion branch
- **Batch 3 partial:** YAML parsing for situation-watcher + deadline done. Cycle detection next.
- Constructor ripple: adding deadline as 12th SituationDefinition field broke 11-arg calls in FeedbackUpdateJobTest, OutcomeRecorderTest, SituationEvaluatorFeedbackTest, SituationDefinitionRegistryTest — all fixed

### Remaining (4 tasks)
- Task 6: Cycle detection in SituationDefinitionRegistry (DFS, self-edge handling for FireOnce vs Repeating)
- Task 7: SituationEvaluator.executeTrigger extraction + triggerByDeadline entry point
- Task 8: DeadlineCheckJob + InMemory/JPA findActiveBySituationId implementations
- Task 9: SituationReplayRunner.drainAllDeadlines + integration tests
