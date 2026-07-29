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
        double confidence,
        int detectionCount,
        int triggerCount,
        Map<String, Object> evidence,
        Map<String, Object> metadata
) {}
```

Confidence: max qualifying confidence from the detections at transition time (same
logic as `DefaultSituationSource.toActiveSituation()`). Evidence: from the detection
with max qualifying confidence.

## Event Capture

CDI observer on `SituationChangeEvent`. Zero changes to `SituationEvaluator`.

### `SituationEventRecorder` (persistence-jpa/)

```java
@ApplicationScoped
public class SituationEventRecorder {
    @Inject EntityManager em;

    void onSituationChange(@ObservesAsync SituationChangeEvent event) {
        SituationEventEntity entity = project(event);
        em.persist(entity);
    }
}
```

- **`@ObservesAsync`** — matches existing `changeEvent.fireAsync()` calls. Never blocks
  the detection hot path.
- **Best-effort** — if the observer fails (DB down), detection/trigger processing is
  unaffected. Event history capture is not transactional with the situation lifecycle.
- **Own transaction** — Quarkus CDI async observers get a new request context.
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
                      Duration window, Duration baseline);

    TenantHealth health(String tenancyId, Duration window);

    default int removeEventsBefore(Instant cutoff) { return 0; }
}
```

Three `history()` overloads for progressive narrowing (tenant → situation → correlation
key). No nullable parameters — consistent with `SituationStore.find()` style.

`triggerCount()` filters to `TRIGGERED` change type specifically.

`trend()` normalizes counts by duration to compare rates. "10 triggers in 24h vs. 70 in
7d" → STABLE (same daily rate).

`health()` returns per-situation aggregate summaries within a single window. No trend
computation (that's a per-situation follow-up call via `trend()`).

`removeEventsBefore()` — default returns 0. JPA implementation does bulk DELETE.

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
thresholds are a follow-up if needed. `INSUFFICIENT_DATA` when baseline window has
zero events.

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
| (expiry job integration) | Calls `removeEventsBefore()` from `SituationExpiryJob` |
| `ras.event.recorded` counter | Tagged by `change_type`, via `RasMetrics` |
| `ras.event.cleanup.removed` counter | Via `RasMetrics` |

No changes to `SituationEvaluator`, `DefaultSituationSource`, or detection path.

## Retention and Cleanup

Event log cleanup integrates with the existing `SituationExpiryJob`.

- Config: `ras.event-history.retention` (Duration, default `P30D`)
- The expiry job computes `Instant.now().minus(retention)` and calls
  `SituationQueryService.removeEventsBefore()`
- JPA implementation: bulk `DELETE FROM SituationEventEntity WHERE eventTime < :cutoff`

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
