# Passive Observation Query Service — Design Spec

**Issue:** casehubio/casehub-ras#43
**Date:** 2026-07-29
**Status:** Approved

## Problem

RAS loses information when situations terminate. `SituationEvaluator` removes the
`SituationContext` from `SituationStore` on TRIGGER, RESOLVE, DISCARD, and SUPPRESS,
then fires a transient `SituationChangeEvent` CDI event. The only surviving query
surface is `SituationSource.activeSituations()` — a flat list of what's accumulating
right now.

This limits RAS's usefulness as an observability layer. Consuming apps that want
dashboard-grade awareness (historical view, frequency, trending) or forensic
investigation (timeline of a specific situation instance) must build their own
infrastructure on top of raw `SituationStore` — which doesn't retain terminated
situations anyway.

## Approach

**Approach A (chosen):** Single flat SPI (`SituationQueryService`) in `api/` backed
by an append-only event log table. `SituationSource` stays as-is for live state queries.
The new SPI serves historical, frequency, trending, and health queries from the event log.

Rejected alternatives:
- **B — Absorb SituationSource:** Muddies the data source boundary (live store vs. event log).
  Implementation must straddle two data sources — leaky abstraction.
- **C — Split SPIs (SituationHistory + SituationAnalytics):** Premature ISP. One event log
  table doesn't justify two query interfaces. No evidence consumers want one without the other.

## Data Model

### Event Log Table: `ras_situation_event`

Append-only. Each row is a snapshot at the moment of a situation lifecycle transition.
Stores a summary projection — not the full `SituationContext` (which carries all
accumulated detections and can be large).

```sql
CREATE TABLE ras_situation_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    situation_id    VARCHAR(255) NOT NULL,
    correlation_key VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    change_type     VARCHAR(50) NOT NULL,
    event_time      TIMESTAMP WITH TIME ZONE NOT NULL,
    first_seen      TIMESTAMP WITH TIME ZONE NOT NULL,
    confidence      DOUBLE PRECISION NOT NULL,
    detection_count INT NOT NULL,
    trigger_count   INT NOT NULL,
    evidence        JSONB,
    metadata        JSONB
);

CREATE INDEX idx_situation_event_tenant_sit_time
    ON ras_situation_event (tenancy_id, situation_id, event_time);

CREATE INDEX idx_situation_event_tenant_time
    ON ras_situation_event (tenancy_id, event_time);

CREATE INDEX idx_situation_event_correlation
    ON ras_situation_event (tenancy_id, correlation_key, event_time);
```

Indexes cover three query patterns:
- Dashboard: frequency/trending for a situation within a tenant
- Dashboard: tenant-wide health queries
- Forensic: drill into a specific correlation key

### Domain Type: `SituationEvent` (api/)

```java
public record SituationEvent(
        String situationId,
        String correlationKey,
        String tenancyId,
        SituationChangeEvent.ChangeType changeType,
        Instant eventTime,
        Instant firstSeen,
        double confidence,
        int detectionCount,
        int triggerCount,
        Map<String, Object> evidence,
        Map<String, Object> metadata
) {
    public SituationEvent {
        Objects.requireNonNull(situationId, "situationId");
        Objects.requireNonNull(correlationKey, "correlationKey");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(changeType, "changeType");
        Objects.requireNonNull(eventTime, "eventTime");
        Objects.requireNonNull(firstSeen, "firstSeen");
        evidence = evidence != null ? Map.copyOf(evidence) : Map.of();
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

Confidence: max qualifying confidence from the detections at transition time (same
logic as `DefaultSituationSource.toActiveSituation()`). Evidence: from the detection
with max qualifying confidence. `firstSeen`: projected from `SituationContext.firstSignal()`
— the time the first detection arrived for this situation instance. Enables accumulation
duration computation (`eventTime - firstSeen`) and active-range queries.

## Event Capture

CDI observer on `SituationChangeEvent`. Zero changes to `SituationEvaluator`.

### `SituationEventRecorder` (persistence-jpa/)

```java
@ApplicationScoped
public class SituationEventRecorder {
    @Inject EntityManager em;

    @Transactional
    void onSituationChange(@ObservesAsync SituationChangeEvent event) {
        try {
            SituationEventEntity entity = project(event);
            em.persist(entity);
        } catch (Exception e) {
            LOG.warning("Failed to record situation event: " + e.getMessage());
        }
    }
}
```

- **`@ObservesAsync`** — matches existing `changeEvent.fireAsync()` calls. Never blocks
  the detection hot path.
- **`@Transactional`** — required because Quarkus CDI async observers get a new request
  context but do NOT automatically start a JTA transaction.
- **Best-effort via try-catch** — the observer catches and logs all exceptions so it never
  propagates failures to the `fireAsync()` `CompletionStage`. This is critical: for
  `NotifyOnly` triggers, `SituationEvaluator.executeDecision()` calls
  `fireAsync(...).toCompletableFuture().join()`, which propagates observer exceptions.
  Without the catch, a DB failure in the recorder would fail and reset every NotifyOnly
  trigger. The `CreateCase` path is fire-and-forget (no `.join()`), but the recorder must
  be safe for both paths.
- **Projection** — extracts confidence, evidence, detection count, trigger count from
  `SituationContext` in the event.

### InMemory equivalent (persistence-memory/)

`InMemorySituationEventStore` — `@ObservesAsync` observer appending to a
`CopyOnWriteArrayList<SituationEvent>`. Backing store for the in-memory query service.

### No observer in runtime/

Event capture is a persistence concern. If neither persistence module is on the classpath,
no capture happens. Consistent with how `SituationStore` works.

## Query SPI

### `SituationQueryService` (api/)

```java
public interface SituationQueryService {

    List<SituationEvent> history(String tenancyId, Instant from, Instant to);

    List<SituationEvent> history(String tenancyId, String situationId,
                                 Instant from, Instant to);

    List<SituationEvent> history(String tenancyId, String situationId,
                                 String correlationKey, Instant from, Instant to);

    long triggerCount(String tenancyId, String situationId,
                      Instant from, Instant to);

    TrendResult trend(String tenancyId, String situationId,
                      Duration window, Duration baseline, Instant asOf);

    TenantHealth health(String tenancyId, Duration window, Instant asOf);
}
```

Three `history()` overloads for progressive narrowing (tenant → situation → correlation
key). `from` and `to` filter on `eventTime` — they select events whose lifecycle
transition occurred within `[from, to)`. The `firstSeen` field is informational in the
returned records; consumers use it for accumulation-duration analysis in application code.
No nullable parameters — consistent with `SituationStore.find()` style. Results are
ordered chronologically (ascending `eventTime`). Contract test verifies ordering.

`triggerCount()` filters to `TRIGGERED` change type specifically.

`trend()` counts TRIGGERED events only — same filter as `triggerCount()` — and compares
trigger rates between two non-overlapping periods anchored at `asOf`:
the **window** is `[asOf - window, asOf)` and the **baseline** is
`[asOf - window - baseline, asOf - window)`. These periods are disjoint — the baseline
ends where the window begins. Example: `trend("t1", "s1", Duration.ofDays(1),
Duration.ofDays(7), Instant.now())` compares the last 24h rate against the 7-day rate
ending 24h ago. The `asOf` parameter enables reproducible queries and deterministic
contract tests. Dashboard callers pass `Instant.now()`.

`health()` returns per-situation aggregate summaries within a single window anchored at
`asOf`: the window is `[asOf - window, asOf)`. Same reproducibility rationale as `trend()`
— dashboard callers pass `Instant.now()`, contract tests pin a fixed instant. No trend
computation (that's a per-situation follow-up call via `trend()`).

### Supporting Types (api/)

```java
public record TrendResult(
        long currentCount,
        long baselineCount,
        TrendDirection direction
) {
    public enum TrendDirection { RISING, FALLING, STABLE, INSUFFICIENT_DATA }
}
```

Direction computed by implementation via rate normalization. Current rate =
`currentCount / window.toMillis()`, baseline rate = `baselineCount / baseline.toMillis()`.
Ratio = `currentRate / baselineRate`. RISING if ratio > 1.2, FALLING if ratio < 0.8,
STABLE otherwise. These thresholds are hardcoded for the first cut — configurable
thresholds are a follow-up if needed. `INSUFFICIENT_DATA` when baseline period has
zero events. Both `currentCount` and `baselineCount` in the result are raw counts from
their respective non-overlapping periods (see `trend()` method documentation).

```java
public record TenantHealth(
        String tenancyId,
        Instant windowStart,
        Instant windowEnd,
        long totalEvents,
        List<SituationSummary> situations
) {}

public record SituationSummary(
        String situationId,
        long eventCount,
        long triggerCount,
        Instant lastEvent
) {}
```

## Module Layout

### api/

| Addition | Purpose |
|----------|---------|
| `SituationQueryService` | SPI interface |
| `SituationEvent` | Event log record |
| `SituationEventRetention` | Retention cleanup interface |
| `TrendResult` + `TrendDirection` | Trend query result |
| `TenantHealth` + `SituationSummary` | Health query result |
| `AbstractSituationQueryServiceContractTest` | Contract test in test-jar |

### persistence-jpa/

| Addition | Purpose |
|----------|---------|
| `SituationEventEntity` | JPA entity |
| `SituationEventRecorder` | CDI `@ObservesAsync` observer |
| `JpaSituationQueryService` | `@ApplicationScoped`, JPQL/native SQL |
| `V6__create_ras_situation_event.sql` | Flyway migration |
| `JpaSituationQueryServiceTest` | Extends contract test |

### persistence-memory/

| Addition | Purpose |
|----------|---------|
| `InMemorySituationEventStore` | `@ObservesAsync`, `CopyOnWriteArrayList` |
| `InMemorySituationQueryService` | `@Alternative @Priority(100)` |
| `InMemorySituationQueryServiceTest` | Extends contract test |

### runtime/

| Addition | Purpose |
|----------|---------|
| (expiry job integration) | `Instance<SituationEventRetention>` in `SituationExpiryJob` |
| `ras.event_log.recorded` counter | Tagged by `change_type`, via `RasMetrics` |
| `ras.expiry.event_log_cleaned` counter | Via `RasMetrics` |

No changes to `SituationEvaluator`, `DefaultSituationSource`, or detection path.

## Retention and Cleanup

Event log cleanup integrates with the existing `SituationExpiryJob` via a dedicated
retention interface — cleanup does not live on the query SPI.

### `SituationEventRetention` (api/)

```java
public interface SituationEventRetention {
    int removeEventsBefore(Instant cutoff);
}
```

The JPA implementation (`JpaSituationQueryService` or a separate bean) implements this
with a bulk `DELETE FROM SituationEventEntity WHERE eventTime < :cutoff`. The in-memory
implementation truncates its backing list.

### Expiry job integration

- Config: `ras.event-history.retention` (Duration, default `P30D`)
- `SituationExpiryJob` injects `Instance<SituationEventRetention>` (optional dependency —
  same pattern as existing `Instance<OrphanedResourceCleaner>`). If no implementation is
  on the classpath, cleanup is skipped.
- The expiry job computes `Instant.now().minus(retention)` and calls
  `removeEventsBefore()` on any resolvable instance.

No separate scheduled job — the expiry job already handles all RAS housekeeping.

## Testing

- **Contract test** (`AbstractSituationQueryServiceContractTest`): covers all SPI methods.
  Empty log, single event, multi-situation filtering, correlation key narrowing, time range
  boundaries, trigger-only counting, trend direction computation (RISING/FALLING/STABLE/
  INSUFFICIENT_DATA), trend rate normalization across different window sizes, health
  aggregation, cleanup.

- **Recorder test** (`SituationEventRecorderTest`): verifies observer projection —
  confidence extraction, evidence mapping, metadata passthrough, detection count accuracy.

- **Integration test**: end-to-end — CloudEvent → RasEngine → trigger → verify event log
  captured the transition correctly.

## Known Limitations

**Time-range queries for still-accumulating situations.** The event log captures terminal
transitions only. `history()` filters on `eventTime` — it returns events whose lifecycle
transition occurred within the requested range. A situation that started at T0,
accumulated through [T1, T2], and triggered at T3 > T2 will not appear in
`history(tenancyId, T1, T2)` because its event is at T3. The `firstSeen` field in each
returned `SituationEvent` record enables consumers to compute accumulation duration and
perform active-range analysis in application code, but does not change what `history()`
returns. Situations still accumulating at query time have no event record; they remain
visible via `SituationSource.activeSituations()`. Capturing inception events (the first
`CONTINUE_ACCUMULATING`) would close the gap for still-accumulating situations but
requires changes to `SituationEvaluator`, which is out of scope for this spec.

**DISMISSED change type.** `SituationChangeEvent.ChangeType.DISMISSED` exists in the enum
but is not currently fired by `SituationEvaluator`. It is reserved for a future dismiss
API (e.g., external user/system dismissal of an accumulating situation). The event log
and `change_type` column will capture DISMISSED events when that path is introduced.

## Not In Scope

- REST endpoint for the query service — the SPI is the deliverable. Consuming apps inject
  via CDI. REST surface is a follow-up if external dashboards need it.
- Materialized views / rollup tables — premature optimization. Add when aggregate query
  performance demands it.
- Streaming/push-based updates — query service is pull-based. Real-time consumers use
  `SituationChangeEvent` via CDI.
- Changes to `SituationSource` or `ActiveSituation` — unchanged for live state queries.
- Pagination on `history()` — follows existing `activeSituations()` pattern. Consumers
  constrain via time range.
- Inception event capture — tracking the start of accumulation in the event log would
  enable full active-range queries for still-accumulating situations, but requires
  changes to `SituationEvaluator` (out of scope).