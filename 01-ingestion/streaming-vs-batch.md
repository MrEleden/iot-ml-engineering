# Streaming vs Batch in the Ingestion Pipeline

## Why This Decision Matters

The ingestion layer has two distinct jobs: get data into Foundry (ingestion) and clean it for downstream use (processing). Streaming and batch are both available in Foundry, and they have fundamentally different cost, complexity, and latency profiles. Choosing wrong means either paying for real-time infrastructure you don't need, or missing acute failures that develop in minutes.

This document explains the choices _within_ the ingestion layer. For the broader scoring architecture (batch scoring + streaming scoring), see [ADR-002: Hybrid Batch + Streaming](../05-architecture/adr-002-batch-plus-streaming.md).

## The Hybrid Pattern

The ingestion pipeline uses streaming for getting data in and batch for processing it:

```
Streaming ingestion → Batch cleansing → Batch feature engineering → Batch scoring
                  ↘ Streaming scoring (acute anomalies only)
```

Each arrow represents a deliberate choice. Here's why.

### Streaming for Ingestion

IoT devices send data continuously. Polling 100K devices on a schedule would require a custom orchestrator, introduce latency proportional to the polling interval, and create thundering-herd load patterns. Streaming ingestion — via the Foundry streaming dataset connected to Event Hubs — avoids all of this:

- **Continuous delivery**: data arrives as devices send it, no polling delay.
- **Managed infrastructure**: Foundry handles consumer offsets, deserialization, and dead-letter routing. No custom consumers to operate.
- **Dual view**: the streaming dataset provides both a real-time streaming view (for acute anomaly detection) and a batch view via 5-minute materialization (for everything else).

Streaming ingestion is not a choice — it's the only viable approach at 14M readings/min from devices that push data asynchronously.

### Batch for Cleansing (Contract 2)

The cleansing transform — deduplication, range validation, timezone normalization, quality flagging — runs as a batch transform (`@transform_df`) on the materialized batch view of the streaming dataset, not as a streaming transform.

**Why not stream the cleansing step?**

1. **Deduplication requires state.** Deduplicating by `event_id` means checking every event against previously seen IDs. In a streaming context, this requires a stateful Flink operator with a large state backend. In batch, deduplication is a `dropDuplicates("event_id")` call over a bounded transaction — trivial.

2. **Device registry lookups.** Unknown devices are routed to a quarantine dataset. In streaming, this requires a side-input join against the device registry, which must be kept in sync with the streaming operator's state. In batch, it's a standard left anti join — simple and correct.

3. **Range validation is stateless and cheap.** It doesn't benefit from real-time execution. A 5–15 minute delay between landing and cleaning is well within the 15-minute freshness SLA for the clean dataset ([Contract 2](../05-architecture/data-contracts.md)).

4. **Foundry streaming transforms (Flink) have limited library support.** The cleansing logic uses PySpark functions that are well-tested in batch Transforms but would require different (and less mature) APIs in Flink.

The cleansing transform runs incrementally — it processes only the new transactions appended by the streaming dataset's batch materialization. See [Transform Patterns — Incremental Transforms](../04-palantir/transform-patterns.md) for the pattern.

### Batch for Feature Engineering and Scoring

Features are windowed aggregations (15-minute, 1-hour, 1-day windows) over clean data. These aggregations require looking back over time and across sensors — operations that are natural in batch but awkward and expensive in streaming.

Batch scoring runs these features through anomaly detection models (Isolation Forest, Autoencoder, statistical). These models expect complete feature vectors, not individual sensor readings. Scoring the full fleet hourly on pre-aggregated features is orders of magnitude cheaper than scoring 14M raw readings/min.

See [ADR-002](../05-architecture/adr-002-batch-plus-streaming.md) for the detailed cost analysis and decision rationale.

### Streaming for Acute Anomaly Detection

Some failures develop in seconds, not hours:

- Sudden compressor current spike (immediate overload risk)
- Rapid pressure drop (acute refrigerant leak)
- Compressor vibration exceeding 3× the 24-hour moving average (mechanical event)
- Temperature rise rate exceeding threshold (complete cooling loss)

These are detected by lightweight threshold rules running as Foundry streaming transforms on the streaming view of the ingestion dataset — _not_ by ML models. The streaming scoring path produces `CRITICAL` severity alerts directly.

This is the only place in the pipeline where streaming _processing_ (not just streaming ingestion) is used.

## When to Use Foundry Streaming Transforms vs Batch Transforms

| Criterion | Streaming Transform | Batch Transform |
|-----------|-------------------|-----------------|
| **Latency requirement** | Sub-minute detection needed | 15 min – 1 hour is acceptable |
| **Logic complexity** | Simple thresholds, single-sensor checks | Multi-sensor aggregations, joins, ML scoring |
| **State management** | Minimal (threshold + moving average) | Full DataFrame operations, device registry joins |
| **Foundry API maturity** | Flink-based, limited PySpark compatibility | Full PySpark, well-documented Transform idioms |
| **Debugging and testing** | Harder — streaming state is opaque | Easier — datasets are inspectable, transforms are deterministic |
| **Failure recovery** | Flink checkpointing, may replay events | Transaction rollback, idempotent reruns |
| **Use in our pipeline** | Acute anomaly threshold checks only | Cleansing, feature engineering, model scoring |
| **Training data access** | Not applicable | Range queries over historical partitions, non-incremental |

**Decision rule**: default to batch. Use streaming transforms only when the detection latency of the batch path (15 min–1 hour) would result in equipment damage or safety hazard.

## Cost and Complexity Tradeoffs

### Compute Cost

| Path | What it processes | Volume | Compute profile |
|------|-------------------|--------|-----------------|
| Streaming ingestion | Raw device messages | 14M/min | Managed by Foundry (streaming dataset throughput tier) |
| Streaming scoring | Raw readings, threshold checks only | 14M/min but trivial per-message compute | Small Flink job, < 200 lines of logic |
| Batch cleansing | New landing zone transactions (incremental) | ~70M rows per 5-min transaction | Medium Spark profile, runs every 5–10 min |
| Batch feature aggregation | Clean data, windowed aggregations | 100K devices × features per window | Medium–Large Spark profile, runs hourly |
| Batch model scoring | Feature vectors | 100K rows per scoring run | Medium Spark profile, runs hourly |

Streaming scoring adds a small, fixed compute cost. The cost is justified by the safety requirement — acute failures must be caught in under 2 minutes. If this path didn't exist, the batch path would need to run every minute, which would be far more expensive.

### Complexity Cost

Maintaining two processing paradigms (batch PySpark + streaming Flink) increases the maintenance surface:

- **Two codebases**: batch transforms use `@transform_df` with PySpark; streaming transforms use Foundry's Flink API.
- **Two testing strategies**: batch transforms can be tested with deterministic input DataFrames; streaming transforms require simulated event sequences.
- **Two monitoring approaches**: batch transforms have transactional success/failure; streaming transforms have lag, throughput, and checkpoint metrics.

This complexity is managed by keeping the streaming path deliberately simple — threshold checks only, no ML models, < 200 lines of code. The batch path handles all complex logic.

### What Would Change This Decision

The current hybrid approach is correct for the current constraints. These changes would force a reassessment:

- **Acute failure prevalence increases**: if most failures become acute (not gradual), the batch path's hourly cadence becomes insufficient. We'd need to move feature engineering or model scoring into streaming — a significant architecture change.
- **Foundry streaming transforms mature significantly**: if Foundry's Flink support gains full PySpark API compatibility and mature ML model serving, the cost differential between batch and streaming processing shrinks.
- **Fleet grows beyond 500K devices**: at 5× scale, the batch cleansing transform processes ~350M rows per 5-min window. If this becomes too expensive per run, we might move cleansing to streaming to spread the cost continuously.
- **Freshness SLAs tighten below 5 minutes**: if downstream consumers need clean data in under 5 minutes (e.g., for operational dashboards), batch cleansing at 5-min intervals is no longer sufficient.

## Training Data Access: A Second Consumer Pattern

The discussion above focuses on operational consumers — transforms that process new data incrementally to produce features, scores, and alerts. Training pipelines are a fundamentally different consumer of the same ingestion data, and their access pattern is pure batch.

### What Training Pipelines Need

Training a seasonal anomaly detection model requires 3–12 months of historical sensor data. The training pipeline does not consume data incrementally as it arrives — it performs a bulk historical read over a date range, constructs a training dataset, and runs offline. Specific requirements:

- **Historical range queries**: read all data for a specific date range (e.g., `timestamp_ingested BETWEEN '2025-06-01' AND '2026-01-31'`) across a subset of sensors and devices. This is a full scan over historical partitions, not an incremental read of new transactions.
- **Point-in-time correctness**: the training dataset must reflect exactly the data that was available at a specific moment. If a late-arriving reading (device reconnection replay) was not available when the model would have run in production, it must not appear in the training data. Delta Lake time travel (`VERSION AS OF` / `TIMESTAMP AS OF`) provides this guarantee.
- **Reproducibility**: re-running the same training job with the same parameters must produce the same output. This requires that the underlying data is immutable for the queried version. Delta Lake's versioned snapshots and the append-only training archive both satisfy this — the same Delta version always returns the same rows.

### How Training Reads Coexist with Operational Writes

The landing zone is simultaneously being written to (every 5 minutes by the streaming dataset materialization) and read from (by training pipelines performing large historical scans). Delta Lake's multi-version concurrency control (MVCC) handles this without conflict:

- Writers create new Delta versions by appending Parquet files. They never modify existing files.
- Readers (training pipelines) read from a specific Delta version. New writes create new versions that the reader does not see — the reader's snapshot is stable for the duration of its query.
- No locking or coordination is needed between the streaming dataset writing new data and a training pipeline reading historical data.

This means training jobs can run at any time without impacting ingestion freshness SLAs, and ingestion writes do not invalidate an in-progress training data read.

### Where Training Data Lives

| Data age | Source | Format | Access pattern |
|----------|--------|--------|----------------|
| 0–90 days | Landing zone (hot tier) | Delta Lake | Range query with `timestamp_ingested` filter; time travel for point-in-time snapshots |
| 90 days – 3+ years | Training archive (cold tier) | Parquet, partitioned by `date(timestamp_utc)` and `device_id` | Direct partition reads; no time travel needed (data is immutable) |

Both sources share the same Contract 1 schema (`raw_sensor_readings`). A training pipeline that spans both tiers unions the two sources, with the archive providing the long tail and the landing zone providing the most recent window.

For details on the training pipeline implementation, see [Training Pipeline](../06-modeling/training-pipeline.md).

## Cross-References

- [ADR-002: Hybrid Batch + Streaming](../05-architecture/adr-002-batch-plus-streaming.md) — the architectural decision record for the hybrid scoring approach
- [Data Contracts](../05-architecture/data-contracts.md) — freshness SLAs for each pipeline stage
- [Streaming Ingestion](../04-palantir/streaming-ingestion.md) — Foundry streaming dataset configuration and materialization intervals
- [Transform Patterns](../04-palantir/transform-patterns.md) — batch transform idioms, incremental processing, windowed aggregations
- [Architecture](./architecture.md) — the physical data flow and component responsibilities
